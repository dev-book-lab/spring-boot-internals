# Embedded Server 커스터마이징 — WebServerFactoryCustomizer와 스레드 풀 튜닝

---

## 🎯 핵심 질문

- `WebServerFactoryCustomizer<T>`를 구현해 서버를 커스터마이징하는 방법은?
- 서버별 전용 커스터마이저(`TomcatConnectorCustomizer`, `UndertowBuilderCustomizer`)의 역할은?
- 스레드 풀 크기는 어떻게 계산하고 튜닝하는가?
- 요청 큐 크기(`accept-count`)와 동시 연결 수(`max-connections`)의 차이는?
- 커스터마이저 적용 순서(`@Order`)는 어떻게 제어하는가?

---

## 🔬 내부 동작 원리

### 1. 커스터마이징 계층

```
커스터마이징 방법 3가지 (넓은 것 → 좁은 것):

1. application.yml — server.* 프로퍼티
   → ServerProperties → ConfigurableServletWebServerFactory 적용
   → 가장 간단, 대부분 커버

2. WebServerFactoryCustomizer<T> Bean
   → BeanPostProcessor로 Factory 초기화 시 자동 적용
   → 프로퍼티로 설정 불가능한 세부 조정에 사용

3. 직접 Factory Bean 등록
   → Auto-configuration 스킵
   → 완전 제어, 복잡성 높음
```

### 2. WebServerFactoryCustomizer 구현 패턴

```java
// 패턴 1: 서버 중립적 커스터마이저
// ConfigurableServletWebServerFactory → Tomcat/Jetty/Undertow 공통 설정
@Component
@Order(Ordered.LOWEST_PRECEDENCE)  // 마지막에 적용
public class CommonServerCustomizer
        implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {

    @Override
    public void customize(ConfigurableServletWebServerFactory factory) {
        factory.setPort(8080);
        factory.setContextPath("/app");
        factory.setDisplayName("My Application");

        // 오류 페이지 커스터마이징
        factory.addErrorPages(
            new ErrorPage(HttpStatus.NOT_FOUND, "/error/404"),
            new ErrorPage(HttpStatus.INTERNAL_SERVER_ERROR, "/error/500")
        );

        // 세션 설정
        factory.setSession(
            Session.of(Duration.ofMinutes(30), true, null, null));
    }
}

// 패턴 2: Tomcat 전용 커스터마이저
@Component
public class TomcatServerCustomizer
        implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        // Tomcat 전용 설정만 가능
        factory.addConnectorCustomizers(this::customizeConnector);
        factory.addContextCustomizers(this::customizeContext);
        factory.addEngineValves(new AccessLogValve());
    }

    private void customizeConnector(Connector connector) {
        // NIO2 프로토콜로 변경 (AIO 지원)
        connector.setProperty("protocol", "org.apache.coyote.http11.Http11Nio2Protocol");
        // Keep-Alive 설정
        connector.setProperty("keepAliveTimeout", "20000");
        connector.setProperty("maxKeepAliveRequests", "100");
    }

    private void customizeContext(Context context) {
        context.setSessionTimeout(30);
        context.setUseHttpOnly(true);
    }
}
```

### 3. 스레드 풀 튜닝 — Tomcat

```java
// TomcatConnectorCustomizer로 세밀한 스레드 설정
@Component
public class TomcatThreadPoolCustomizer
        implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.addConnectorCustomizers(connector -> {
            // AbstractProtocol을 통해 스레드 풀 직접 접근
            ProtocolHandler handler = connector.getProtocolHandler();
            if (handler instanceof AbstractProtocol<?> protocol) {
                // 최대 스레드 수
                protocol.setMaxThreads(300);
                // 최소 유휴 스레드
                protocol.setMinSpareThreads(25);
                // 연결 타임아웃
                protocol.setConnectionTimeout(30_000);  // 30초
                // Keep-alive 연결 수
                protocol.setMaxConnections(10_000);
            }
        });
    }
}
```

```yaml
# application.yml로 동일 설정 (권장)
server:
  tomcat:
    threads:
      max: 300          # 최대 스레드 (기본 200)
      min-spare: 25     # 최소 유휴 스레드 (기본 10)
    accept-count: 200   # 모든 스레드 사용 중일 때 큐 대기 수
    max-connections: 10000  # 최대 동시 TCP 연결
    connection-timeout: 30s
```

### 4. 스레드 수 계산 방법

```
CPU 집중 작업:
  최적 스레드 수 ≈ CPU 코어 수 + 1

I/O 집중 작업 (대부분의 웹 앱):
  최적 스레드 수 = CPU 코어 수 × (1 + Wait Time / CPU Time)

  예: CPU 4코어, 요청 처리 시 80% I/O 대기
  = 4 × (1 + 0.8/0.2) = 4 × 5 = 20 스레드

  하지만 실제로는:
  - 스레드당 ~1MB 스택 메모리
  - 200 스레드 → 200MB 최소 메모리
  - 컨텍스트 스위칭 오버헤드

운영 환경 권장:
  1. 현재 Tomcat 기본값(200)으로 시작
  2. 모니터링: tomcat.threads.busy, tomcat.threads.config.max
  3. busy ≈ max → 병목 → max 증가 고려
  4. busy << max → 과도한 스레드 → 감소 고려

  공식: Little's Law
  L = λ × W  (L=동시 요청, λ=초당 요청, W=평균 처리 시간)
  예: 초당 100 요청, 평균 처리 200ms
  → 필요 스레드 ≈ 100 × 0.2 = 20 스레드
```

### 5. accept-count vs max-connections 차이

```
요청 처리 경로:

  새 연결 → OS 네트워크 스택 → accept queue (OS 레벨)
    → Tomcat acceptor → 연결 수락
    → NIO Selector 등록 (최대 max-connections 8192개)
    → 요청 준비됨 → Thread Pool에서 스레드 할당
    → 스레드 모두 사용 중 → accept-count 큐 대기 (기본 100)
    → 큐도 가득 찬 → 연결 거절 (Connection Refused)

max-connections=8192:
  동시에 유지할 수 있는 TCP 연결 수
  NIO Selector에 등록된 소켓 수
  → 커넥션 풀 클라이언트에서 유지하는 연결 포함

accept-count=100:
  모든 스레드가 바쁠 때 큐에서 대기하는 추가 요청 수
  → TCP listen backlog와 유사
  → 이 값보다 많은 요청 → 즉시 Connection Refused
```

### 6. Undertow 커스터마이징

```java
@Component
public class UndertowCustomizer
        implements WebServerFactoryCustomizer<UndertowServletWebServerFactory> {

    @Override
    public void customize(UndertowServletWebServerFactory factory) {
        factory.addBuilderCustomizers(builder -> {
            // IO 스레드 (Non-blocking, CPU 집중)
            builder.setIoThreads(4);  // 기본: CPU * 2

            // Worker 스레드 (블로킹 작업)
            builder.setWorkerThreads(32);  // 기본: CPU * 8

            // 버퍼 크기 (Direct Buffer 사용)
            builder.setBufferSize(1024);
            builder.setDirectBuffers(true);

            // HTTP 옵션
            builder.setServerOption(
                UndertowOptions.ENABLE_HTTP2, true);
            builder.setServerOption(
                UndertowOptions.MAX_ENTITY_SIZE, 10 * 1024 * 1024L);  // 10MB
        });

        // 액세스 로그
        factory.setAccessLogEnabled(true);
        factory.setAccessLogPattern(
            "%h %l %u %t \"%r\" %s %b %D ms");
    }
}
```

### 7. Jetty 커스터마이징

```java
@Component
public class JettyCustomizer
        implements WebServerFactoryCustomizer<JettyServletWebServerFactory> {

    @Override
    public void customize(JettyServletWebServerFactory factory) {
        factory.addServerCustomizers(server -> {
            // QueuedThreadPool 직접 설정
            QueuedThreadPool threadPool = new QueuedThreadPool();
            threadPool.setMaxThreads(200);
            threadPool.setMinThreads(8);
            threadPool.setIdleTimeout(60_000);
            threadPool.setName("jetty-worker");
            server.setThreadPool(threadPool);

            // HTTP 커넥터 설정
            ServerConnector connector = new ServerConnector(server);
            connector.setPort(8080);
            connector.setIdleTimeout(30_000);
            server.setConnectors(new Connector[]{connector});
        });
    }
}
```

### 8. 커스터마이저 우선순위

```java
// @Order로 적용 순서 제어
// 낮은 숫자 = 먼저 실행 (나중 실행된 것이 덮어씀)

@Component
@Order(1)  // 먼저 실행 — 기본 설정
public class BaseServerCustomizer
        implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {
    @Override
    public void customize(ConfigurableServletWebServerFactory factory) {
        factory.setPort(8080);
    }
}

@Component
@Order(2)  // 나중 실행 — 환경별 오버라이드
@Profile("prod")
public class ProdServerCustomizer
        implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {
    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.addConnectorCustomizers(connector -> {
            connector.setProperty("maxThreads", "400");
        });
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 스레드 풀 메트릭 모니터링

```yaml
# Actuator + Prometheus로 스레드 모니터링
management:
  endpoints:
    web:
      exposure:
        include: metrics, prometheus
```

```bash
# Tomcat 스레드 메트릭 조회
curl http://localhost:8080/actuator/metrics/tomcat.threads.busy
curl http://localhost:8080/actuator/metrics/tomcat.threads.config.max
curl http://localhost:8080/actuator/metrics/tomcat.connections.current

# busy / config.max 비율이 80% 이상이면 스레드 풀 포화 경고
```

### 실험 2: 부하 테스트로 스레드 설정 검증

```bash
# Apache Bench로 부하 테스트
ab -n 10000 -c 200 http://localhost:8080/api/endpoint

# 결과 확인:
# - Non-2xx responses: 0이어야 함
# - 응답 시간 분포: p99 < 목표 SLA
# - Tomcat busy 스레드: max 근접 시 증가 고려
```

### 실험 3: 커스터마이저 적용 확인

```java
@Component
public class VerifyCustomizerApplied implements ApplicationListener<WebServerInitializedEvent> {
    @Override
    public void onApplicationEvent(WebServerInitializedEvent event) {
        WebServer server = event.getWebServer();
        if (server instanceof TomcatWebServer tomcatServer) {
            Connector connector = tomcatServer.getTomcat().getConnector();
            System.out.println("Max Threads: " +
                connector.getProtocolHandler().getMaxThreads());
        }
    }
}
```

---

## ⚙️ 설정 최적화 팁

```yaml
# 운영 환경 Tomcat 권장 설정
server:
  tomcat:
    threads:
      max: 200          # 기본값 — 모니터링 후 조정
      min-spare: 20     # 기본 10보다 크게 — 급격한 트래픽 증가 대응
    accept-count: 200   # 큐 크기 — SLA 기준으로 설정
    max-connections: 8192
    connection-timeout: 20s

    # 압축 (대역폭 절약)
    compression:
      enabled: true
      mime-types: text/html,text/xml,text/plain,text/css,application/json
      min-response-size: 1024  # 1KB 이상만 압축

    # 액세스 로그 (운영 필수)
    accesslog:
      enabled: true
      pattern: '%h %l %u %t "%r" %s %b %{X-Forwarded-For}i %D'
      rotate: true
      rename-on-rotate: true
      max-days: 30
```

---

## 📌 핵심 정리

```
커스터마이징 방법 선택
  간단한 설정: application.yml server.* 프로퍼티
  세밀한 제어: WebServerFactoryCustomizer<T> Bean
  완전 제어: Factory Bean 직접 등록 (Auto-configuration 스킵)

커스터마이저 적용 시점
  WebServerFactoryCustomizerBeanPostProcessor
  → Factory Bean 초기화 직후, getWebServer() 전

스레드 풀 튜닝 원칙
  Little's Law: 필요 스레드 = 처리율 × 평균 응답시간
  메모리 제약: 스레드당 ~1MB 스택
  모니터링 우선: busy/max 비율로 병목 감지

accept-count vs max-connections
  max-connections: NIO Selector 동시 연결 수 (기본 8192)
  accept-count: 스레드 포화 시 큐 대기 수 (기본 100)
  → 둘 다 초과 시 Connection Refused
```

---

## 🤔 생각해볼 문제

**Q1.** `WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>`와 `WebServerFactoryCustomizer<TomcatServletWebServerFactory>`를 모두 등록했을 때 Tomcat 서버에서는 어느 것이 먼저 실행되는가?

**Q2.** `server.tomcat.threads.max=500`으로 설정하면 항상 500개의 스레드가 생성되는가?

**Q3.** `accept-count=0`으로 설정하면 어떤 효과가 있는가?

> 💡 **해설**
>
> **Q1.** `@Order`가 없으면 등록 순서에 따라 결정되며, 일반적으로 `ConfigurableServletWebServerFactory` 타입의 Customizer가 더 넓은 타입이어서 먼저 등록되는 경향이 있다. `@Order`를 명시하는 것이 가장 안전하다. 두 Customizer가 모두 적용된다는 점이 중요하다 — `ConfigurableServletWebServerFactory`를 타겟으로 하는 것도 Tomcat Factory에 적용된다(Tomcat Factory가 이 인터페이스를 구현하므로). 특정 순서가 필요하면 `@Order(1)`, `@Order(2)` 명시를 권장한다.
>
> **Q2.** 아니다. `max=500`은 최대값을 지정하는 것이다. Tomcat은 `min-spare` 값만큼 최소 스레드를 유지하고, 요청이 증가하면 필요에 따라 스레드를 생성한다. 최대 500개까지 생성되지만 유휴 스레드는 일정 시간 후 반환된다. 실제 사용 중인 스레드 수는 Actuator의 `tomcat.threads.busy` 메트릭으로 확인할 수 있다.
>
> **Q3.** `accept-count=0`은 큐 대기를 허용하지 않는다. 모든 스레드가 바쁠 때 들어오는 추가 요청은 즉시 Connection Refused된다. 이 설정은 빠른 실패(fail-fast) 전략이다. 큐에서 대기하다 처리되는 것보다 즉시 거절해 클라이언트가 빠르게 다른 인스턴스를 시도하도록 유도할 때 사용한다. 로드밸런서 뒤에서 여러 인스턴스가 있는 환경에서 유효한 전략이다.

---

<div align="center">

**[⬅️ 이전: ServletWebServerFactory 동작 원리](./02-servlet-web-server-factory.md)** | **[홈으로 🏠](../README.md)** | **[다음: SSL/TLS 설정 ➡️](./04-ssl-tls-configuration.md)**

</div>
