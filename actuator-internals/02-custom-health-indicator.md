# Custom Health Indicator 작성 — 상태 집계 알고리즘과 CompositeHealthContributor

---

## 🎯 핵심 질문

- `HealthIndicator`와 `HealthContributor`의 차이는?
- `CompositeHealthContributor`로 여러 하위 시스템을 묶는 패턴은?
- `/actuator/health`의 전체 상태는 어떤 알고리즘으로 집계되는가?
- `Status.UP/DOWN/UNKNOWN/OUT_OF_SERVICE`의 우선순위 순서는?
- `ReactiveHealthIndicator`는 언제 사용하는가?
- `show-details` 설정에 따라 응답 구조가 어떻게 바뀌는가?

---

## 🔍 왜 이게 존재하는가

```
운영 환경 모니터링 필요:
  로드밸런서: "이 인스턴스가 트래픽 받을 준비됐나?" → /actuator/health
  Kubernetes liveness probe: "프로세스 살아있나?" → /actuator/health/liveness
  Kubernetes readiness probe: "요청 처리 가능한가?" → /actuator/health/readiness

Spring Boot 기본 제공 HealthIndicator:
  DataSourceHealthIndicator  → DB 연결 확인
  RedisHealthIndicator       → Redis ping
  DiskSpaceHealthIndicator   → 디스크 여유 공간
  MongoHealthIndicator       → MongoDB 연결
  ...

커스텀 필요 케이스:
  외부 API 헬스 체크
  특정 비즈니스 규칙 기반 상태 (큐 크기, 캐시 히트율 등)
  다중 서브시스템 집계
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: HealthIndicator에서 예외를 직접 던진다

```java
// ❌ 잘못된 구현
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        restTemplate.getForEntity(apiUrl, String.class);  // 예외 발생 시 전파
        return Health.up().build();
    }
}
// 예외가 전파되면 /actuator/health 자체가 500 오류
// → 전체 헬스 체크 실패

// ✅ 올바른 구현 — try-catch로 항상 Health 반환
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        try {
            restTemplate.getForEntity(apiUrl, String.class);
            return Health.up().withDetail("url", apiUrl).build();
        } catch (Exception ex) {
            return Health.down(ex).withDetail("url", apiUrl).build();
        }
    }
}
```

### Before: 모든 서브시스템을 하나의 HealthIndicator에 넣는다

```java
// ❌ 하나의 Indicator에 모든 체크를 집어넣음
@Component
public class AllSystemsHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean dbOk = checkDb();
        boolean cacheOk = checkCache();
        boolean apiOk = checkExternalApi();
        // 어느 시스템이 문제인지 파악 불가
        return (dbOk && cacheOk && apiOk) ? Health.up().build() : Health.down().build();
    }
}

// ✅ CompositeHealthContributor로 분리 → 어느 서브시스템이 DOWN인지 명확
```

---

## ✨ 올바른 이해와 사용

### After: /actuator/health 응답 구조

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {"database": "H2", "validationQuery": "isValid()"}
    },
    "externalApi": {
      "status": "DOWN",
      "details": {"url": "https://api.example.com", "error": "Connection refused"}
    },
    "diskSpace": {
      "status": "UP",
      "details": {"total": 499963174912, "free": 389083136000}
    }
  }
}
// externalApi가 DOWN → 전체 status = DOWN → HTTP 503
```

---

## 🔬 내부 동작 원리

### 1. HealthIndicator 인터페이스 계층

```java
// HealthContributor — 최상위 마커 인터페이스
public interface HealthContributor { }

// HealthIndicator — 단일 Health 반환
@FunctionalInterface
public interface HealthIndicator extends HealthContributor {
    Health health();
}

// CompositeHealthContributor — 여러 HealthContributor 집계
public interface CompositeHealthContributor
        extends HealthContributor,
                Iterable<NamedContributor<HealthContributor>> {
    HealthContributor getContributor(String name);
}

// ReactiveHealthIndicator — WebFlux 환경용
@FunctionalInterface
public interface ReactiveHealthIndicator extends HealthContributor {
    Mono<Health> health();
}
```

### 2. Health 객체 구성

```java
// Health — 상태 + 세부 정보
public final class Health {

    private final Status status;
    private final Map<String, Object> details;

    // Builder 패턴
    public static Builder up()             { return status(Status.UP); }
    public static Builder down()           { return status(Status.DOWN); }
    public static Builder down(Throwable ex) { ... }
    public static Builder outOfService()   { return status(Status.OUT_OF_SERVICE); }
    public static Builder unknown()        { return status(Status.UNKNOWN); }

    public static class Builder {
        public Builder withDetail(String key, Object value) { ... }
        public Builder withDetails(Map<String, ?> details) { ... }
        public Builder withException(Throwable ex) { ... }
        public Health build() { return new Health(this); }
    }
}

// 내장 Status 4가지
public class Status {
    public static final Status UNKNOWN        = new Status("UNKNOWN");
    public static final Status UP             = new Status("UP");
    public static final Status DOWN           = new Status("DOWN");
    public static final Status OUT_OF_SERVICE = new Status("OUT_OF_SERVICE");

    // 커스텀 Status도 가능
    public static final Status DEGRADED = new Status("DEGRADED");
}
```

### 3. 단순 HealthIndicator 구현

```java
// 외부 API 헬스 체크
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    private final RestTemplate restTemplate;
    private final String apiUrl;

    public ExternalApiHealthIndicator(
            RestTemplate restTemplate,
            @Value("${external.api.url}") String apiUrl) {
        this.restTemplate = restTemplate;
        this.apiUrl = apiUrl;
    }

    @Override
    public Health health() {
        try {
            ResponseEntity<String> response =
                restTemplate.getForEntity(apiUrl + "/health", String.class);

            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("url", apiUrl)
                    .withDetail("httpStatus", response.getStatusCode().value())
                    .build();
            }

            return Health.down()
                .withDetail("url", apiUrl)
                .withDetail("httpStatus", response.getStatusCode().value())
                .build();

        } catch (Exception ex) {
            return Health.down(ex)
                .withDetail("url", apiUrl)
                .withDetail("error", ex.getMessage())
                .build();
        }
    }
}
// Bean 이름 규칙:
// "externalApiHealthIndicator" → "HealthIndicator" 접미사 제거 → "externalApi" 컴포넌트
// → /actuator/health 응답에 components.externalApi 로 자동 포함
```

### 4. CompositeHealthContributor — 여러 하위 시스템 집계

```java
@Component("databases")  // Bean 이름 → components 키
public class DatabaseHealthContributor implements CompositeHealthContributor {

    private final Map<String, HealthContributor> contributors = new LinkedHashMap<>();

    public DatabaseHealthContributor(
            DataSource primaryDataSource,
            DataSource secondaryDataSource) {

        contributors.put("primary",
            (HealthIndicator) () -> checkDataSource(primaryDataSource, "primary"));
        contributors.put("secondary",
            (HealthIndicator) () -> checkDataSource(secondaryDataSource, "secondary"));
    }

    private Health checkDataSource(DataSource ds, String name) {
        try (Connection conn = ds.getConnection()) {
            boolean valid = conn.isValid(2);  // 2초 타임아웃
            return valid
                ? Health.up().withDetail("database", name).build()
                : Health.down().withDetail("database", name)
                               .withDetail("reason", "isValid() returned false").build();
        } catch (SQLException ex) {
            return Health.down(ex).withDetail("database", name).build();
        }
    }

    @Override
    public HealthContributor getContributor(String name) {
        return contributors.get(name);
    }

    @Override
    public Iterator<NamedContributor<HealthContributor>> iterator() {
        return contributors.entrySet().stream()
            .map(entry -> NamedContributor.of(entry.getKey(), entry.getValue()))
            .iterator();
    }
}
```

```json
// /actuator/health 응답 (show-details: always)
{
  "status": "DOWN",
  "components": {
    "databases": {
      "status": "DOWN",
      "components": {
        "primary":   {"status": "UP",   "details": {"database": "primary"}},
        "secondary": {"status": "DOWN", "details": {"database": "secondary", "error": "..."}}
      }
    },
    "diskSpace": {"status": "UP"},
    "ping":      {"status": "UP"}
  }
}
// secondary DOWN → databases DOWN → 전체 DOWN → HTTP 503
```

### 5. 상태 집계 알고리즘 — SimpleStatusAggregator

```java
// CompositeHealth 계산 — SimpleStatusAggregator 기본 구현
public class SimpleStatusAggregator implements StatusAggregator {

    // 기본 우선순위 (앞에 있을수록 높음)
    private static final List<String> DEFAULT_ORDER =
        Arrays.asList("DOWN", "OUT_OF_SERVICE", "UNKNOWN", "UP");

    @Override
    public Status getAggregateStatus(Set<Status> statuses) {
        return statuses.stream()
            .min(Comparator.comparingInt(this::getOrder))
            .orElse(Status.UNKNOWN);
    }

    private int getOrder(Status status) {
        int index = DEFAULT_ORDER.indexOf(status.getCode());
        return (index >= 0) ? index : DEFAULT_ORDER.size();
    }
}

// 집계 규칙:
// DOWN > OUT_OF_SERVICE > UNKNOWN > UP
// 컴포넌트 중 하나라도 DOWN → 전체 DOWN
// 모두 UP → 전체 UP
// DOWN 없고 OUT_OF_SERVICE 있음 → 전체 OUT_OF_SERVICE
```

```yaml
# HTTP 상태 코드 매핑 커스터마이징
management:
  endpoint:
    health:
      status:
        order: DOWN, OUT_OF_SERVICE, UNKNOWN, UP
        http-mapping:
          DOWN: 503
          OUT_OF_SERVICE: 503
          UNKNOWN: 200
          UP: 200
          DEGRADED: 207    # 커스텀 Status 매핑
```

### 6. HealthEndpoint 처리 경로

```java
// /actuator/health 요청 처리 흐름:
//
// HealthEndpointWebExtension.health()
//   → HealthEndpoint.health()
//     → HealthContributorRegistry에서 모든 Contributor 조회
//     → 각 HealthIndicator.health() 병렬 호출 (기본: 직렬)
//     → CompositeHealth 생성
//       → SimpleStatusAggregator로 전체 상태 집계
//     → show-details 설정에 따라 응답 필터링
//     → StatusAggregator 결과로 HTTP 상태 코드 결정
//       (DOWN → 503, UP → 200)

// 병렬 실행 설정
management:
  health:
    defaults:
      enabled: true
```

### 7. Liveness / Readiness Probe (Kubernetes)

```java
// Spring Boot 2.3+ — Kubernetes 전용 그룹
// /actuator/health/liveness  → 프로세스 살아있나? (장애 시 pod restart)
// /actuator/health/readiness → 트래픽 받을 수 있나? (장애 시 로드밸런서 제외)

@Component
public class MyLivenessIndicator implements HealthIndicator {
    @Override
    public Health health() {
        // 복구 불가능한 내부 오류 확인 (OOM, deadlock 등)
        // DOWN 반환 시 Kubernetes가 pod를 재시작
        return Health.up().build();
    }
}

// 수동으로 Readiness 상태 전환
@Autowired ApplicationEventPublisher publisher;

public void startMaintenance() {
    publisher.publishEvent(new AvailabilityChangeEvent<>(
        this, ReadinessState.REFUSING_TRAFFIC));
    // → /actuator/health/readiness → OUT_OF_SERVICE
    // → Kubernetes가 로드밸런서에서 pod 제외
}

public void endMaintenance() {
    publisher.publishEvent(new AvailabilityChangeEvent<>(
        this, ReadinessState.ACCEPTING_TRAFFIC));
}
```

```yaml
# Kubernetes probe 활성화
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

### 8. ReactiveHealthIndicator — WebFlux 환경

```java
// WebFlux 환경에서 비동기 헬스 체크
@Component
public class ReactiveExternalApiHealthIndicator
        implements ReactiveHealthIndicator {

    private final WebClient webClient;

    public ReactiveExternalApiHealthIndicator(WebClient.Builder builder,
            @Value("${external.api.url}") String apiUrl) {
        this.webClient = builder.baseUrl(apiUrl).build();
    }

    @Override
    public Mono<Health> health() {
        return webClient.get()
            .uri("/health")
            .retrieve()
            .toBodilessEntity()
            .map(response -> Health.up()
                .withDetail("httpStatus", response.getStatusCode().value())
                .build())
            .onErrorResume(ex -> Mono.just(
                Health.down(ex)
                    .withDetail("error", ex.getMessage())
                    .build()))
            .timeout(Duration.ofSeconds(3),
                Mono.just(Health.down()
                    .withDetail("error", "timeout after 3s")
                    .build()));
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: HealthIndicator 자동 등록 확인

```yaml
# application.yml
management:
  endpoint:
    health:
      show-details: always
  endpoints:
    web:
      exposure:
        include: health
```

```bash
curl http://localhost:8080/actuator/health | jq .components
# → externalApiHealthIndicator → "externalApi" 키로 자동 포함됨
# → databases → "databases.primary", "databases.secondary" 계층 구조
```

### 실험 2: DOWN 상태 시뮬레이션

```java
@Component
public class AlwaysDownHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        return Health.down()
            .withDetail("reason", "Simulated DOWN for testing")
            .withDetail("timestamp", Instant.now().toString())
            .build();
    }
}
// → /actuator/health → 503 Service Unavailable
// → Kubernetes readiness probe 실패 → 로드밸런서 제외
```

### 실험 3: 커스텀 Status + HTTP 코드 매핑

```java
public static final Status DEGRADED = new Status("DEGRADED");

@Component
public class QueueHealthIndicator implements HealthIndicator {

    private final Queue<?> orderQueue;
    private static final int WARNING_THRESHOLD = 1000;
    private static final int CRITICAL_THRESHOLD = 5000;

    @Override
    public Health health() {
        int size = orderQueue.size();

        if (size >= CRITICAL_THRESHOLD) {
            return Health.down()
                .withDetail("queueSize", size)
                .withDetail("threshold", CRITICAL_THRESHOLD)
                .build();
        }
        if (size >= WARNING_THRESHOLD) {
            return Health.status(DEGRADED)
                .withDetail("queueSize", size)
                .withDetail("threshold", WARNING_THRESHOLD)
                .build();
        }
        return Health.up()
            .withDetail("queueSize", size)
            .build();
    }
}
```

```yaml
# DEGRADED 상태 HTTP 코드 설정
management:
  endpoint:
    health:
      status:
        order: DOWN, OUT_OF_SERVICE, DEGRADED, UNKNOWN, UP
        http-mapping:
          DEGRADED: 207   # Multi-Status (부분 장애 표현)
```

### 실험 4: HealthContributor 그룹 분리

```yaml
# 비핵심 시스템을 별도 그룹으로 분리
management:
  endpoint:
    health:
      group:
        core:
          include: db, diskSpace
          show-details: always
        external:
          include: externalApi, paymentGateway
          show-details: when-authorized
          status:
            http-mapping:
              DOWN: 200   # 외부 시스템 DOWN은 전체 서비스 장애 아님
```

```bash
curl /actuator/health/core      # 핵심 시스템만 집계
curl /actuator/health/external  # 외부 시스템만 집계
curl /actuator/health           # 전체 (그룹 포함)
```

---

## ⚙️ 설정 최적화 팁

```yaml
management:
  endpoint:
    health:
      show-details: when-authorized  # 인증된 사용자에게만 세부 정보
      show-components: always        # 컴포넌트 목록은 항상 표시 (세부 없이)
      roles: ACTUATOR_ADMIN, MONITORING
      group:
        liveness:
          include: livenessstate     # Kubernetes liveness
        readiness:
          include: readinessstate, db, diskSpace  # Kubernetes readiness
```

```java
// 헬스 체크 타임아웃 제한 — 느린 외부 시스템 영향 방지
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    private final RestTemplate restTemplate;

    public ExternalApiHealthIndicator() {
        // 타임아웃 설정
        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(2000);
        factory.setReadTimeout(3000);
        this.restTemplate = new RestTemplate(factory);
    }

    @Override
    public Health health() {
        // 최대 3초 내에 응답 없으면 DOWN
        try {
            restTemplate.getForEntity(apiUrl + "/health", String.class);
            return Health.up().build();
        } catch (ResourceAccessException ex) {
            // 타임아웃 포함
            return Health.down(ex).withDetail("error", "timeout or connection refused").build();
        }
    }
}
```

---

## 🤔 트레이드오프

```
HealthIndicator 세밀도:
  세밀하게 분리 (서브시스템별 개별 Indicator)
    장점  어느 시스템이 문제인지 즉시 파악
    단점  Indicator 수 증가, 관리 복잡성

  CompositeHealthContributor로 그룹화
    장점  계층 구조로 의미 있는 그룹핑
    단점  중간 집계 노드 추가 구현 필요

show-details 설정:
  always   → 모든 세부 정보 공개 (내부망 전용 환경)
  never    → 상태 코드만 (외부 공개)
  when-authorized → 균형 (권장)

Readiness probe에 외부 시스템 포함 여부:
  포함: 외부 API DOWN → pod 트래픽 차단 → 과도한 pod 제외 위험
  제외: 외부 API DOWN이어도 pod는 트래픽 받음 → 앱 레벨에서 처리
  → 비핵심 외부 시스템은 별도 health group으로 분리 권장
```

---

## 📌 핵심 정리

```
인터페이스 선택
  단일 체크       → HealthIndicator
  하위 시스템 그룹 → CompositeHealthContributor
  Reactive 환경   → ReactiveHealthIndicator

Bean 이름 → 컴포넌트 키 변환
  {name}HealthIndicator → "name"
  @Component("myGroup") → "myGroup"

집계 알고리즘 (SimpleStatusAggregator)
  DOWN > OUT_OF_SERVICE > UNKNOWN > UP
  하나라도 DOWN → 전체 DOWN → HTTP 503
  커스텀 Status + http-mapping으로 HTTP 코드 변경 가능

Kubernetes 프로브
  /actuator/health/liveness  → LivenessState (pod restart 제어)
  /actuator/health/readiness → ReadinessState (트래픽 제어)
  management.endpoint.health.probes.enabled: true

show-details 동작
  never           → {"status":"UP"}
  always          → 전체 세부 정보 포함
  when-authorized → Security 역할 기반
```

---

## 🤔 생각해볼 문제

**Q1.** `HealthIndicator.health()`가 예외를 던지면 어떻게 처리되는가?

**Q2.** `management.endpoint.health.show-details=when-authorized` 설정 시 "authorized"의 기준은 무엇인가?

**Q3.** `CompositeHealthContributor`의 일부 컴포넌트가 DOWN일 때 전체 Health가 DOWN이 아니도록 하려면 어떻게 해야 하는가?

> 💡 **해설**
>
> **Q1.** `HealthIndicator.health()` 실행 중 예외가 발생하면 `HealthEndpoint`가 예외를 포착해 자동으로 `Health.down(throwable)`으로 변환한다. 예외가 전파되지 않으므로 다른 HealthIndicator의 실행이 중단되지 않고, `/actuator/health` 자체는 응답을 계속 반환한다. 이 동작 덕분에 외부 시스템 연결 실패가 Actuator 자체 장애(500)로 이어지지 않는다. 다만 내부 `withException()`을 통해 예외 메시지가 details에 포함될 수 있으므로 `show-details` 설정에 주의해야 한다.
>
> **Q2.** `when-authorized`는 Spring Security가 설정된 경우 `management.endpoint.health.roles`에 지정된 역할을 가진 인증 사용자일 때 세부 정보를 반환한다. Spring Security가 없으면 항상 세부 정보를 숨긴다. `roles`를 명시하지 않으면 인증된 모든 사용자에게 세부 정보를 보여준다. 운영 환경에서는 `roles: ACTUATOR_ADMIN, MONITORING`처럼 역할을 명시해 세밀하게 제어하는 것이 권장된다.
>
> **Q3.** 가장 깔끔한 방법은 Health 그룹(`management.endpoint.health.group`)을 사용해 비핵심 시스템을 별도 그룹으로 분리하는 것이다. 그룹의 `status.http-mapping.DOWN=200`으로 설정하면 해당 그룹 내 DOWN이 HTTP 503을 반환하지 않는다. 또는 `CompositeHealthContributor` 구현 내부에서 자체 집계 로직을 작성해 일부 컴포넌트의 DOWN을 UNKNOWN이나 DEGRADED로 격하한 뒤 `StatusAggregator`를 커스터마이징할 수도 있다. `StatusAggregator` Bean을 직접 등록하면 집계 우선순위 전체를 재정의할 수 있다.

---

<div align="center">

**[⬅️ 이전: Actuator Endpoint 구조](./01-actuator-endpoint-structure.md)** | **[홈으로 🏠](../README.md)** | **[다음: Metrics 수집 — Micrometer 통합 ➡️](./03-micrometer-metrics.md)**

</div>
