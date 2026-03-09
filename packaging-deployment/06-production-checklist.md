# Production-ready Checklist — 운영 배포 전 필수 설정 가이드

---

## 🎯 핵심 질문

- Actuator 엔드포인트를 운영 환경에서 안전하게 노출하는 방법은?
- 로깅 레벨 전략과 운영 환경에서의 구조적 로그(JSON) 설정은?
- 컨테이너 환경에서의 JVM 메모리 튜닝 기준은?
- Graceful Shutdown이 동작하는 원리와 설정 방법은?
- Kubernetes 환경을 위한 Liveness/Readiness Probe 설계 기준은?

---

## 🔬 내부 동작 원리

### 1. Actuator 보안 설정

```yaml
# application-prod.yml
management:
  # 별도 포트로 분리 (외부 노출 차단)
  server:
    port: 8081   # 내부 네트워크에서만 접근
    # 8080(서비스) vs 8081(관리) 포트 분리
    # → 로드밸런서/인그레스는 8080만 노출
    # → 8081은 클러스터 내부 또는 VPN에서만 접근

  endpoints:
    web:
      exposure:
        # 운영에서 최소한만 노출
        include: "health,info,metrics,prometheus"
        # exclude: "*" 후 필요한 것만 include 추천

  endpoint:
    health:
      show-details: when-authorized  # 인증된 요청만 상세 정보
      show-components: when-authorized
      probes:
        enabled: true   # Liveness/Readiness 엔드포인트 활성화
    info:
      enabled: true
    shutdown:
      enabled: false    # 운영에서 절대 활성화 금지
    env:
      enabled: false    # 환경변수/설정 노출 금지
    beans:
      enabled: false    # Bean 목록 노출 금지
    mappings:
      enabled: false
```

```java
// Actuator 엔드포인트 보안 설정 (Spring Security)
@Configuration
@Order(1)  // 앱 Security 설정보다 우선
public class ActuatorSecurityConfig {

    @Bean
    public SecurityFilterChain actuatorSecurityChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(auth -> auth
                // health, info: 인증 없이 접근 (K8s probe용)
                .requestMatchers(
                    EndpointRequest.to(HealthEndpoint.class),
                    EndpointRequest.to(InfoEndpoint.class))
                .permitAll()
                // metrics, prometheus: 모니터링 시스템 접근
                .requestMatchers(EndpointRequest.to("metrics", "prometheus"))
                .hasRole("MONITORING")
                // 나머지: 관리자만
                .anyRequest().hasRole("ACTUATOR_ADMIN")
            );
        return http.build();
    }
}
```

### 2. 로깅 전략

```yaml
# application-prod.yml
logging:
  # 운영 환경 구조적 로그 (JSON)
  # → Elasticsearch, Splunk, CloudWatch 등 수집/검색 용이
  structured:
    format:
      console: ecs    # Elastic Common Schema JSON 형식
      # 또는 logstash, gelf 등

  # 로그 레벨 전략
  level:
    root: WARN                            # 기본: 경고 이상만
    com.example: INFO                     # 앱 패키지: INFO
    com.example.security: WARN            # 보안 관련: WARN (토큰 로그 방지)
    org.springframework.web: WARN         # Spring MVC: WARN
    org.springframework.security: WARN
    org.hibernate.SQL: WARN              # SQL 로그 비활성화 (운영)
    org.hibernate.type: WARN

  # 민감 정보 마스킹 (Spring Boot 3.4+)
  # application.yml에서 *** 로 마스킹되는 키 패턴
```

```java
// 구조적 로그 예시 (ECS JSON 형식)
// 출력:
// {
//   "@timestamp": "2026-03-09T10:15:30.123Z",
//   "log.level": "INFO",
//   "message": "User login successful",
//   "service.name": "myapp",
//   "service.version": "1.2.3",
//   "trace.id": "abc123",         ← 분산 추적 연동
//   "span.id": "def456",
//   "user.id": "12345"
// }

// MDC로 요청 컨텍스트 추가
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        try {
            // 요청별 추적 ID
            String requestId = UUID.randomUUID().toString();
            MDC.put("request.id", requestId);
            MDC.put("http.method", request.getMethod());
            MDC.put("url.path", request.getRequestURI());

            filterChain.doFilter(request, response);
        } finally {
            MDC.clear();  // 스레드 풀 재사용 시 누수 방지
        }
    }
}
```

### 3. JVM 메모리 튜닝

```bash
# 컨테이너 환경 JVM 메모리 설정
java \
  # 컨테이너 메모리 제한 자동 인식 (Java 11+)
  -XX:MaxRAMPercentage=75.0 \
  # 컨테이너 메모리의 75% → Heap Max
  # 나머지 25% → Metaspace + 스레드 스택 + 다이렉트 메모리

  # GC 선택 (Spring Boot 앱 기본 권장)
  -XX:+UseG1GC \
  # G1: 대부분의 경우 최선의 선택
  # ZGC: 지연 시간 극히 민감한 경우 (-XX:+UseZGC)

  # GC 로그 (운영 문제 진단)
  -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=20m \

  # OOM 발생 시 힙 덤프
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/dumps/heap.hprof \

  # OOM 시 컨테이너 재시작 (K8s가 자동 재스케줄)
  -XX:+ExitOnOutOfMemoryError \

  -jar app.jar
```

```yaml
# 컨테이너 메모리 설정 가이드
# 컨테이너 제한 512MB 예시:
# MaxRAMPercentage=75% → Heap Max ≈ 384MB
# Metaspace: ~100MB (Spring Boot 앱 일반적)
# 스레드 스택: 스레드 수 × 512KB
# 다이렉트 메모리: Netty 사용 시 주의 필요
# → 총합이 512MB 넘지 않도록 조정
# → 안전 마진: 25~30% 여유 권장

# Paketo Buildpack 사용 시:
# BPL_JVM_THREAD_COUNT: 스레드 수 (메모리 계산 기반)
# BPL_JVM_HEAD_ROOM: 시스템 메모리 여유 비율 (기본 10%)
# → 자동으로 -Xmx 산정
```

### 4. Graceful Shutdown

```yaml
# Graceful Shutdown 설정
server:
  shutdown: graceful    # 기본값: immediate

spring:
  lifecycle:
    # 최대 대기 시간 (기본 30초)
    timeout-per-shutdown-phase: 30s
    # → 30초 내 처리 중인 요청 완료 대기
    # → 30초 초과 시 강제 종료
```

```java
// Graceful Shutdown 동작 원리

// 1. SIGTERM 수신 (K8s가 Pod 종료 시 전송)
// 2. Spring Boot SmartLifecycle.stop() 호출
// 3. 새 요청 수락 중단 (Tomcat: 커넥터 pause)
//    → 로드밸런서가 이 서버로 새 요청 보내지 않음
// 4. 처리 중인 요청 완료 대기 (timeout-per-shutdown-phase)
// 5. ApplicationContext.close() → @PreDestroy, Bean 소멸
// 6. 프로세스 종료

// K8s 배포 설정과 연동:
// terminationGracePeriodSeconds: 60  (Pod 종료 최대 대기)
// → Spring Boot timeout(30s) < K8s grace period(60s)
// → Spring이 완전히 종료된 후 K8s가 프로세스 강제 종료

// 주의: DB 연결, 메시지 컨슈머 등 @PreDestroy 처리 필요
@Component
public class KafkaConsumerLifecycle implements SmartLifecycle {

    private volatile boolean running = false;

    @Override
    public void stop(Runnable callback) {
        // 컨슈머 polling 중단
        this.consumer.wakeup();
        // 현재 처리 중인 메시지 완료 대기
        this.processingLatch.await(10, TimeUnit.SECONDS);
        this.running = false;
        callback.run();  // 완료 신호
    }

    @Override
    public int getPhase() {
        return Integer.MAX_VALUE - 100;  // 늦게 시작, 일찍 종료
    }
}
```

### 5. Kubernetes Health Probe 설계

```yaml
# application.yml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessstate:
      enabled: true   # /actuator/health/liveness
    readinessstate:
      enabled: true   # /actuator/health/readiness
```

```yaml
# K8s Deployment 설정
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.2.3

          ports:
            - containerPort: 8080  # 서비스 포트
            - containerPort: 8081  # Actuator 포트

          # Liveness Probe: 앱이 살아있는가?
          # → 실패 시 컨테이너 재시작
          # → 복구 불가능한 상태(교착, OOM 등) 감지
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8081
            initialDelaySeconds: 60   # 기동 대기
            periodSeconds: 10
            failureThreshold: 3       # 3회 연속 실패 시 재시작

          # Readiness Probe: 트래픽을 받을 준비가 됐는가?
          # → 실패 시 서비스 엔드포인트에서 제외 (재시작 안 함)
          # → DB 연결 없음, 의존 서비스 다운 등
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8081
            initialDelaySeconds: 30
            periodSeconds: 5
            failureThreshold: 3

          # Startup Probe: 초기 기동 완료 확인
          # → 기동 중 Liveness 실패로 인한 조기 재시작 방지
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8081
            failureThreshold: 30    # 30 × 10s = 최대 300초 기동 허용
            periodSeconds: 10
```

```java
// 커스텀 Readiness HealthIndicator
// 의존 서비스 준비 상태를 Readiness에 반영
@Component
public class ExternalServiceReadinessIndicator
        implements HealthIndicator {

    private final ExternalServiceClient client;

    @Override
    public Health health() {
        try {
            boolean available = client.isAvailable();
            if (available) {
                return Health.up()
                    .withDetail("service", "external-api")
                    .build();
            }
            // DOWN → Readiness 실패 → 트래픽 중단
            return Health.down()
                .withDetail("reason", "External API unavailable")
                .build();
        } catch (Exception ex) {
            return Health.down(ex).build();
        }
    }
}

// 앱 자체 버그는 Liveness에만 반영
// 외부 의존성 문제는 Readiness에 반영
// → 재시작해도 해결 안 되는 문제 → Liveness 실패 금지
```

### 6. 운영 환경 필수 설정 요약

```yaml
# application-prod.yml 전체 권장 설정

server:
  shutdown: graceful
  tomcat:
    threads:
      max: 200
      min-spare: 10
    connection-timeout: 5s
    max-connections: 8192
    accept-count: 100

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 3000   # 3초 초과 시 예외 (대기 금지)
      idle-timeout: 600000       # 10분 유휴 시 반환
      max-lifetime: 1800000      # 30분 최대 수명
      validation-timeout: 1000
      keepalive-time: 30000      # 30초마다 keepalive 쿼리

  jpa:
    open-in-view: false          # OSIV 반드시 비활성화 (커넥션 낭비)
    properties:
      hibernate:
        format_sql: false
        generate_statistics: false  # 운영 성능 오버헤드

management:
  server:
    port: 8081
  endpoints:
    web:
      exposure:
        include: "health,info,metrics,prometheus"
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true

logging:
  structured:
    format:
      console: ecs
  level:
    root: WARN
    com.example: INFO
```

---

## 💻 실험으로 확인하기

### 실험 1: Graceful Shutdown 테스트

```bash
# 앱 실행 후 장시간 처리 요청 전송
curl -X POST http://localhost:8080/api/long-task &

# SIGTERM 전송 (K8s 종료 시뮬레이션)
kill -15 $(pgrep -f "java.*app.jar")

# 로그 확인:
# "Commencing graceful shutdown. Waiting for active requests to complete"
# (요청 처리 완료 후)
# "Graceful shutdown complete"
# → 처리 중 요청이 완료된 후 종료됨
```

### 실험 2: 메모리 설정 확인

```bash
# 컨테이너에서 JVM 메모리 자동 설정 확인
docker run --memory=512m myapp:latest \
  java -XX:MaxRAMPercentage=75.0 \
       -XshowSettings:vm \
       -jar app.jar

# 출력:
# VM settings:
#     Max. Heap Size (Estimated): 384.00M  (= 512M * 75%)
#     Ergonomics Machine Class: server
#     Using VM: OpenJDK 64-Bit Server VM
```

### 실험 3: Health Probe 상태 전환

```bash
# Readiness 상태 수동 전환 (배포 롤링 업데이트 시뮬레이션)
# ApplicationAvailability를 통해 상태 변경

# Readiness DOWN으로 전환
curl -X POST http://localhost:8081/actuator/health
# K8s가 이 서버를 로드밸런서에서 제외

# Liveness 상태 확인
curl http://localhost:8081/actuator/health/liveness
# {"status":"UP"}

# Readiness 상태 확인
curl http://localhost:8081/actuator/health/readiness
# {"status":"UP", "components": {...}}
```

---

## 📌 핵심 정리

```
Actuator 보안
  management.server.port: 8081  서비스 포트와 분리
  최소 노출: health, info, metrics, prometheus
  shutdown 엔드포인트: 절대 활성화 금지
  show-details: when-authorized  상세 정보 인증 후 노출

로깅
  운영: WARN 기본, 앱 패키지만 INFO
  구조적 로그(JSON): 로그 집중화 시스템 연동
  MDC로 요청 컨텍스트 추가
  민감 정보(토큰, 비밀번호) 로그 마스킹

JVM 메모리
  -XX:MaxRAMPercentage=75.0  컨테이너 제한 자동 인식
  -XX:+ExitOnOutOfMemoryError  OOM 시 재시작 (K8s 활용)
  G1GC 기본, ZGC는 지연 시간 극히 민감한 경우

Graceful Shutdown
  server.shutdown: graceful
  timeout-per-shutdown-phase: 30s
  K8s terminationGracePeriodSeconds > Spring timeout

K8s Health Probe
  /liveness  앱 생사 (실패 → 재시작)
  /readiness 트래픽 수신 준비 (실패 → 서비스 제외)
  외부 의존성 문제 → Readiness에만 반영 (Liveness 실패 금지)
```

---

## 🤔 생각해볼 문제

**Q1.** `server.shutdown=graceful` 설정 시 처리 중인 WebSocket 연결이나 SSE(Server-Sent Events) 스트림은 어떻게 처리되는가?

**Q2.** 외부 DB가 다운됐을 때 Liveness가 아닌 Readiness 프로브만 실패하도록 설계해야 하는 이유는?

**Q3.** K8s 롤링 업데이트 중 구버전 Pod가 아직 트래픽을 받는 동안 신버전 Pod의 Readiness가 UP이 되면 즉시 구버전 트래픽을 끊는다. 이 과정에서 발생할 수 있는 문제와 해결 방법은?

> 💡 **해설**
>
> **Q1.** `server.shutdown=graceful`은 HTTP 요청/응답 사이클을 기다리지만, WebSocket과 SSE는 장기 연결이므로 `timeout-per-shutdown-phase` 시간 내에 자연스럽게 종료되지 않을 수 있다. Spring의 기본 동작은 타임아웃이 되면 연결을 강제 종료한다. WebSocket의 경우 서버가 CLOSE 프레임을 전송해 클라이언트가 재연결을 시도할 수 있도록 처리해야 한다. 클라이언트 측에서 재연결 로직을 구현하고, 서버에서는 `@PreDestroy`나 `SmartLifecycle`에서 활성 WebSocket 세션에 종료 메시지를 전송하는 것이 좋다.
>
> **Q2.** DB 다운 시 Liveness가 실패하면 K8s가 컨테이너를 재시작한다. 그러나 재시작 후에도 DB가 여전히 다운 상태라면 재시작 → Liveness 실패 → 재시작의 무한 루프가 발생한다(CrashLoopBackOff). 이는 문제를 해결하지 못하고 오히려 빠른 재시작 시도로 DB에 부담을 준다. Readiness가 실패하면 K8s는 컨테이너를 서비스 엔드포인트에서만 제외하고 재시작하지 않는다. DB가 복구되면 자동으로 Readiness가 복구되어 트래픽이 다시 유입된다. "재시작해도 해결되지 않는 문제"는 Readiness에, "앱 자체의 복구 불가능한 상태"는 Liveness에 반영하는 것이 원칙이다.
>
> **Q3.** 구버전 Pod에서 진행 중이던 긴 DB 트랜잭션이나 외부 API 호출이 완료되기 전에 트래픽이 끊기면 사용자에게 오류가 반환될 수 있다. 해결 방법: 첫째, `preStop` 훅에 `sleep` 을 추가해 K8s가 로드밸런서에서 Pod를 제외한 후 기존 요청을 처리할 시간을 준다. 둘째, `terminationGracePeriodSeconds`를 충분히 크게 설정하고 Spring Graceful Shutdown 타임아웃보다 크게 설정한다. 셋째, 클라이언트 측 재시도 로직(지수 백오프)을 구현해 일시적 오류를 자동 복구한다. 넷째, `minReadySeconds`를 설정해 신버전 Pod가 안정적으로 동작하는 것을 확인한 후 구버전을 제거한다.

---

<div align="center">

**[⬅️ 이전: Cloud Native Buildpacks](./05-cloud-native-buildpacks.md)** | **[홈으로 🏠](../README.md)**

</div>
