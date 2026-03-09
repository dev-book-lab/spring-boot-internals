# Actuator Security 설정 — EndpointRequest 매처와 운영 환경 노출 최소화

---

## 🎯 핵심 질문

- `EndpointRequest.toAnyEndpoint()`는 내부적으로 어떻게 경로를 매칭하는가?
- Spring Security와 Actuator를 통합하는 올바른 패턴은?
- 공개 Endpoint(health, info)와 보안 Endpoint(env, beans)를 분리하는 방법은?
- 운영 환경에서 최소한으로 노출해야 하는 Endpoint 목록은?
- `management.server.port` 분리 시 Security 설정이 달라지는 이유는?

---

## 🔬 내부 동작 원리

### 1. EndpointRequest — Actuator 전용 RequestMatcher

```java
// EndpointRequest — Actuator 경로를 타입 안전하게 참조
public final class EndpointRequest {

    // 모든 Actuator Endpoint 경로 매칭
    public static EndpointRequestMatcher toAnyEndpoint() {
        return new EndpointRequestMatcher(true);
    }

    // 특정 Endpoint 경로만 매칭
    public static EndpointRequestMatcher to(Object... endpoints) {
        return new EndpointRequestMatcher(Arrays.asList(endpoints), Collections.emptyList());
    }

    // 특정 Endpoint 제외
    public static EndpointRequestMatcher toAnyEndpoint()
            .excluding(HealthEndpoint.class, InfoEndpoint.class);
}

// EndpointRequestMatcher.matches() 동작:
//   /actuator/health → health Endpoint 경로인지 확인
//   management.endpoints.web.base-path 설정 반영
//   동적으로 현재 노출된 Endpoint 목록 조회 → 경로 매칭

// 일반 AntPathRequestMatcher와 차이:
//   AntPath: 경로 문자열 패턴 매칭 (정적)
//   EndpointRequest: Actuator Endpoint ID → 경로 동적 매핑
//   → base-path가 변경되어도 코드 수정 없이 동작
```

### 2. Spring Security 통합 — 완성 패턴

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    // ① 앱 Security + Actuator Security 통합
    @Bean
    @Order(1)  // Actuator Security가 앱 Security보다 먼저
    public SecurityFilterChain actuatorSecurityChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(auth -> auth
                // 공개 Endpoint (인증 없이 접근)
                .requestMatchers(EndpointRequest.to(HealthEndpoint.class,
                                                     InfoEndpoint.class))
                    .permitAll()
                // 읽기 전용 모니터링 Endpoint
                .requestMatchers(EndpointRequest.to(MetricsEndpoint.class,
                                                     PrometheusEndpoint.class))
                    .hasRole("MONITORING")
                // 민감 Endpoint — 관리자만
                .anyRequest()
                    .hasRole("ACTUATOR_ADMIN")
            )
            .httpBasic(Customizer.withDefaults())  // Basic Auth
            .csrf(csrf -> csrf.disable())  // Actuator는 CSRF 불필요
            .build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain appSecurityChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

### 3. 역할 기반 Endpoint 분류

```java
// ACTUATOR_ADMIN: 모든 Actuator 접근 (운영팀)
// MONITORING: 메트릭/헬스 조회만 (모니터링 시스템)
// 익명: health, info만

.authorizeHttpRequests(auth -> auth
    // 레벨 1: 공개 (외부 모니터링, 로드밸런서)
    .requestMatchers(EndpointRequest.to(
        HealthEndpoint.class,
        InfoEndpoint.class
    )).permitAll()

    // 레벨 2: 모니터링 시스템 (Prometheus, Grafana)
    .requestMatchers(EndpointRequest.to(
        MetricsEndpoint.class,
        "prometheus"  // ID 문자열로도 지정 가능
    )).hasRole("MONITORING")

    // 레벨 3: 운영 관리 (나머지 전체)
    .requestMatchers(EndpointRequest.toAnyEndpoint())
        .hasRole("ACTUATOR_ADMIN")
)
```

### 4. 운영 환경 최소 노출 전략

```yaml
# 운영 환경 권장 설정
management:
  endpoints:
    web:
      exposure:
        include:
          - health       # 로드밸런서, Kubernetes probe
          - info         # 버전 정보 (빌드 정보 등)
          - prometheus   # Prometheus 스크레이핑
          # 필요 시 추가:
          # - metrics    # Spring Boot Admin 등 모니터링 도구
          # - loggers    # 운영 중 로그 레벨 동적 변경
        exclude:
          - env          # 환경변수 노출 금지
          - beans        # Bean 목록 노출 금지
          - heapdump     # 힙 덤프 (보안 위험)
          - threaddump   # 스레드 덤프
    jmx:
      exposure:
        include: ""  # JMX 비활성화
  endpoint:
    health:
      show-details: when-authorized  # 세부 정보는 인증 후
      show-components: always         # 컴포넌트 상태는 항상
  server:
    port: 8081           # 포트 분리
    address: 127.0.0.1   # 내부망 전용
```

### 5. 포트 분리 환경의 Security

```java
// management.server.port 분리 시:
// 두 개의 ApplicationContext:
//   1. 앱 컨텍스트 (포트 8080)
//   2. Management 컨텍스트 (포트 8081)

// Management 컨텍스트의 Security:
//   ManagementWebSecurityAutoConfiguration 자동 적용
//   → 별도 SecurityFilterChain Bean 필요
//   → OR: management.server.port의 보안은 방화벽으로 처리 (내부망만)

// 포트가 분리되어 내부망에서만 접근 가능하면
// Security 설정 없이 permitAll()도 실용적 선택

@Bean
public SecurityFilterChain managementSecurityChain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher(EndpointRequest.toAnyEndpoint())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(EndpointRequest.toAnyEndpoint()).permitAll()
            // 내부망 전용이므로 인증 없이 전체 허용
        )
        .csrf(csrf -> csrf.disable())
        .build();
}
```

### 6. health 세부 정보 보안

```yaml
# 상황별 show-details 설정
management:
  endpoint:
    health:
      # never: 상태 코드만 (UP/DOWN)
      # when-authorized: 인증된 사용자에게만 세부 정보
      # always: 항상 세부 정보 (내부망 전용일 때만)
      show-details: when-authorized
      roles:
        - ACTUATOR_ADMIN
        - MONITORING
      # show-components: always → 컴포넌트는 항상 표시 (세부 없이)
      show-components: always
```

---

## 💻 실험으로 확인하기

### 실험 1: Security 없이 Endpoint 접근 시도

```bash
# Security 미적용 시
curl http://localhost:8080/actuator/env  # 200 + 모든 환경변수 노출

# Security 적용 후
curl http://localhost:8080/actuator/env  # 401 Unauthorized
curl -u admin:secret http://localhost:8080/actuator/env  # 200 (ACTUATOR_ADMIN만)
```

### 실험 2: Endpoint 노출 최소화 확인

```bash
# include: health,info,prometheus 설정 후
curl http://localhost:8080/actuator | jq '._links | keys'
# → ["health","info","prometheus"] 만 표시
# env, beans, heapdump 없음
```

### 실험 3: EndpointRequest 동작 확인

```java
@Test
void endpointRequestMatchesActuatorPaths() {
    EndpointRequest.EndpointRequestMatcher matcher =
        EndpointRequest.to(HealthEndpoint.class);

    // "/actuator/health" → match
    // "/actuator/health/liveness" → match
    // "/api/users" → no match
    // base-path 변경 시 자동 반영
}
```

---

## ⚙️ 설정 최적화 팁

```java
// 1. info Endpoint에 빌드/Git 정보 추가 (보안 고려)
// build.gradle: springBoot { buildInfo() }
// git.properties 플러그인 → git.commit.id 등 자동 추가
// → /actuator/info에 빌드 시간, git SHA 포함
// → 운영 환경에서 배포 버전 확인 용도로 유용

// 2. health groups — Kubernetes probe 분리
management:
  endpoint:
    health:
      probes:
        enabled: true
  # /actuator/health/liveness  → ACTUATOR_ADMIN 또는 공개
  # /actuator/health/readiness → ACTUATOR_ADMIN 또는 공개
  # Kubernetes probe는 인증 없이 접근해야 함 → permitAll
```

---

## 📌 핵심 정리

```
EndpointRequest
  타입 안전한 Actuator 경로 매처
  EndpointRequest.to(HealthEndpoint.class) → /actuator/health
  base-path 변경에 자동 대응 (경로 문자열 하드코딩 불필요)

Security 통합 패턴
  @Order(1) actuatorSecurityChain → EndpointRequest.toAnyEndpoint()
  공개: health, info → permitAll
  모니터링: metrics, prometheus → MONITORING 역할
  민감: 나머지 → ACTUATOR_ADMIN 역할

운영 환경 필수 설정
  include: health, info, prometheus 만
  exclude: env, beans, heapdump, threaddump
  jmx: 비활성화
  management.server.port 분리 + address=127.0.0.1

show-details
  never: 외부 공개 (상태만)
  when-authorized: 모니터링 인증 사용자
  always: 내부망 포트 분리 환경
```

---

## 🤔 생각해볼 문제

**Q1.** `EndpointRequest.toAnyEndpoint().excluding(HealthEndpoint.class)`에서 health 외의 모든 Endpoint는 어떤 Security 규칙이 적용되는가?

**Q2.** Kubernetes에서 liveness/readiness probe가 Spring Security에 막히지 않도록 하려면 어떻게 설정해야 하는가?

**Q3.** `management.endpoint.health.show-details=when-authorized`와 `management.endpoint.health.roles=MONITORING`을 함께 설정했을 때, `ACTUATOR_ADMIN` 역할을 가진 사용자는 세부 정보를 볼 수 있는가?

> 💡 **해설**
>
> **Q1.** `.excluding(HealthEndpoint.class)` 이후의 규칙이 적용된다. 예를 들어 `EndpointRequest.toAnyEndpoint().excluding(HealthEndpoint.class)`를 `.hasRole("ACTUATOR_ADMIN")`으로 매핑하면, health를 제외한 모든 Endpoint는 ACTUATOR_ADMIN 역할이 필요하다. health는 이 매처에서 제외됐으므로 별도 규칙에서 `.permitAll()`로 처리하면 된다. SecurityFilterChain에서 규칙은 위에서 아래로 첫 번째 매칭 규칙이 적용되므로 health를 먼저 배치하는 것이 중요하다.
>
> **Q2.** Kubernetes probe 경로(`/actuator/health/liveness`, `/actuator/health/readiness`)를 Security에서 `permitAll()`로 설정해야 한다. `EndpointRequest.to(HealthEndpoint.class)`는 `/actuator/health/**` 경로 전체를 매칭하므로 liveness/readiness도 포함된다. 또는 `management.server.port`를 분리하고 `address=127.0.0.1`로 설정하면 클러스터 내부 네트워크에서 Security 없이 접근하도록 구성할 수 있다.
>
> **Q3.** 볼 수 없다. `management.endpoint.health.roles`는 `show-details=when-authorized`일 때 세부 정보 접근을 허용할 역할 목록을 지정한다. `roles=MONITORING`으로 설정하면 MONITORING 역할만 세부 정보를 볼 수 있고, ACTUATOR_ADMIN은 별도로 지정하지 않으면 세부 정보가 숨겨진다. ACTUATOR_ADMIN도 세부 정보를 보려면 `roles: MONITORING, ACTUATOR_ADMIN`으로 모두 추가해야 한다.

---

<div align="center">

**[⬅️ 이전: JMX vs HTTP Exposure](./05-jmx-vs-http-exposure.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — Embedded Server Configuration ➡️](../embedded-server/01-tomcat-jetty-undertow.md)**

</div>
