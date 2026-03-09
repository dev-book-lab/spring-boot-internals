# Metrics 수집 — Micrometer 통합과 Prometheus 연동 데이터 흐름

---

## 🎯 핵심 질문

- `MeterRegistry` 계층 구조에서 `CompositeMeterRegistry`의 역할은?
- `Counter`, `Gauge`, `Timer`, `DistributionSummary`의 차이와 사용 기준은?
- `@Timed`는 내부적으로 어떻게 동작하는가?
- Prometheus 연동 시 `/actuator/prometheus`의 데이터 흐름은?
- 태그(Tag)는 어떻게 Prometheus 레이블로 변환되는가?
- `MeterBinder`를 통한 JVM 메트릭 자동 수집 원리는?

---

## 🔍 왜 이게 존재하는가

```
Micrometer = 메트릭 수집 라이브러리의 SLF4J

SLF4J: 로깅 라이브러리 추상화 (Logback, Log4j 교체 가능)
Micrometer: 메트릭 백엔드 추상화 (Prometheus, Datadog, CloudWatch 교체 가능)

MeterRegistry (추상)
  ├── PrometheusMeterRegistry  → /actuator/prometheus
  ├── CloudWatchMeterRegistry  → AWS CloudWatch
  ├── DatadogMeterRegistry     → Datadog
  ├── StatsdMeterRegistry      → StatsD
  └── SimpleMeterRegistry      → 인메모리 (테스트용)

CompositeMeterRegistry → 여러 Registry에 동시 기록
```

---

## 🔬 내부 동작 원리

### 1. MeterRegistry 계층과 Auto-configuration

```java
// MeterRegistryAutoConfiguration → CompositeMeterRegistry 등록
@AutoConfiguration
@ConditionalOnClass(MeterRegistry.class)
public class MeterRegistryAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(MeterRegistry.class)
    public CompositeMeterRegistry compositeMeterRegistry(
            MeterRegistryCustomizer<?>... customizers,
            Collection<MeterBinder> binders) {

        CompositeMeterRegistry composite = new CompositeMeterRegistry();

        // MeterRegistryCustomizer 적용 (공통 태그 등)
        applyCustomizers(composite, customizers);

        // MeterBinder 등록 (JVM 메트릭 등)
        binders.forEach(binder -> binder.bindTo(composite));

        return composite;
    }
}

// Prometheus가 classpath에 있으면 PrometheusMeterRegistry 추가
@AutoConfiguration
@ConditionalOnClass(PrometheusMeterRegistry.class)
public class PrometheusMetricsExportAutoConfiguration {

    @Bean
    public PrometheusMeterRegistry prometheusMeterRegistry(
            PrometheusProperties properties,
            CollectorRegistry collectorRegistry) {
        return new PrometheusMeterRegistry(
            new PrometheusConfig() { ... },
            collectorRegistry,
            Clock.SYSTEM);
    }
}
// → CompositeMeterRegistry에 PrometheusMeterRegistry 자동 추가
// → 측정값이 Prometheus 형식으로도 동시 기록됨
```

### 2. Meter 타입 4가지

```java
// ① Counter — 단조 증가 카운터 (요청 수, 오류 수)
Counter requestCounter = Counter.builder("api.requests")
    .tag("endpoint", "/users")
    .tag("method", "GET")
    .description("Total API requests")
    .register(meterRegistry);

requestCounter.increment();         // +1
requestCounter.increment(3.0);      // +3

// ② Gauge — 현재 상태값 (큐 크기, 활성 연결 수)
// 측정 시점에 람다를 호출해서 현재 값을 읽음
Gauge.builder("queue.size", messageQueue, Queue::size)
    .tag("queue", "orderQueue")
    .register(meterRegistry);
// → 조회 시 messageQueue.size() 호출 → 현재 값 반환
// → 강한 참조 주의: Gauge는 대상 객체를 약한 참조로 유지
//   대상 객체가 GC되면 Gauge도 사라짐

// ③ Timer — 시간 측정 + 히스토그램 (응답 시간, 처리 시간)
Timer timer = Timer.builder("api.response.time")
    .tag("endpoint", "/orders")
    .description("API response time")
    .publishPercentiles(0.5, 0.95, 0.99)  // p50, p95, p99
    .publishPercentileHistogram()           // Prometheus 히스토그램 버킷
    .sla(Duration.ofMillis(100), Duration.ofMillis(500))  // SLA 버킷
    .register(meterRegistry);

// 사용법
timer.record(() -> {
    // 측정할 작업
    processOrder();
});

// 또는 수동 기록
long start = System.nanoTime();
processOrder();
timer.record(System.nanoTime() - start, TimeUnit.NANOSECONDS);

// ④ DistributionSummary — 값 분포 (페이로드 크기, 응답 크기)
DistributionSummary summary = DistributionSummary.builder("http.response.size")
    .baseUnit("bytes")
    .publishPercentiles(0.5, 0.95)
    .register(meterRegistry);

summary.record(response.getContentLength());
```

### 3. @Timed — AOP 기반 자동 측정

```java
// @Timed 동작 조건:
// 1. spring-boot-starter-actuator + micrometer-core 의존성
// 2. TimedAspect Bean 등록 필요

@Configuration
public class TimingConfig {
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}

// 사용
@Service
public class OrderService {

    @Timed(value = "order.processing", extraTags = {"type", "standard"})
    public Order processOrder(OrderRequest request) {
        // 메서드 실행 시간 자동 측정
        // Timer name: order.processing
        // Tags: type=standard, class=OrderService, method=processOrder, exception=none
        return ...;
    }

    @Timed(value = "order.payment",
           percentiles = {0.5, 0.95},
           histogram = true)
    public PaymentResult processPayment(Payment payment) { ... }
}
```

```java
// @Timed 내부 동작 (TimedAspect)
@Around("@annotation(timed)")
public Object timedMethod(ProceedingJoinPoint pjp, Timed timed) throws Throwable {
    Timer.Sample sample = Timer.start(registry);
    String exceptionClass = "none";

    try {
        return pjp.proceed();
    } catch (Exception ex) {
        exceptionClass = ex.getClass().getSimpleName();
        throw ex;
    } finally {
        // 메서드 종료 후 Timer 기록
        sample.stop(
            Timer.builder(timed.value())
                .tags(timed.extraTags())
                .tag("exception", exceptionClass)
                .tag("class", pjp.getTarget().getClass().getSimpleName())
                .tag("method", pjp.getSignature().getName())
                .register(registry)
        );
    }
}
```

### 4. 공통 태그 — MeterRegistryCustomizer

```java
// 모든 Meter에 공통 태그 추가
@Bean
public MeterRegistryCustomizer<MeterRegistry> commonTags(
        @Value("${spring.application.name}") String appName) {
    return registry -> registry.config()
        .commonTags(
            "application", appName,
            "environment", System.getenv().getOrDefault("ENVIRONMENT", "local"),
            "region", System.getenv().getOrDefault("AWS_REGION", "unknown")
        );
}
// → 모든 메트릭에 application, environment, region 레이블 자동 추가
// → Prometheus에서 {application="order-service", environment="prod", ...}
```

### 5. MeterBinder — JVM 메트릭 자동 수집

```java
// Spring Boot가 자동 등록하는 MeterBinder들:
// JvmGcMetrics        → jvm.gc.pause, jvm.gc.memory.promoted 등
// JvmMemoryMetrics    → jvm.memory.used, jvm.memory.max 등
// JvmThreadMetrics    → jvm.threads.live, jvm.threads.daemon 등
// ProcessorMetrics    → system.cpu.usage, process.cpu.usage 등
// UptimeMetrics       → process.uptime, process.start.time
// LogbackMetrics      → logback.events (level별 로그 카운트)
// HikariCpMetrics     → hikaricp.connections.active 등 (HikariCP 있을 때)

// 커스텀 MeterBinder
@Component
public class QueueMetricsBinder implements MeterBinder {

    private final OrderQueue orderQueue;

    @Override
    public void bindTo(MeterRegistry registry) {
        Gauge.builder("order.queue.size", orderQueue, Queue::size)
            .description("Current order queue size")
            .register(registry);

        Gauge.builder("order.queue.capacity", orderQueue, Queue::remainingCapacity)
            .description("Remaining order queue capacity")
            .register(registry);
    }
}
```

### 6. Prometheus 연동 — 데이터 흐름

```
코드:
  requestCounter.increment()
  timer.record(100ms)

               ↓
MeterRegistry (in-memory 저장)
  counter[api.requests{endpoint="/users"}] = 42
  timer[api.response.time{endpoint="/users"}] = {count=42, sum=4200ms, p95=150ms}

               ↓ (GET /actuator/prometheus 요청 시)
PrometheusMeterRegistry.scrape()
  → Prometheus 텍스트 포맷으로 직렬화

               ↓
HTTP 응답:
  # HELP api_requests_total Total API requests
  # TYPE api_requests_total counter
  api_requests_total{application="order-service",endpoint="/users",method="GET"} 42.0
  
  # HELP api_response_time_seconds API response time
  # TYPE api_response_time_seconds summary
  api_response_time_seconds{endpoint="/users",quantile="0.95"} 0.15
  api_response_time_seconds_count{endpoint="/users"} 42
  api_response_time_seconds_sum{endpoint="/users"} 4.2
```

```yaml
# Prometheus 스크레이핑 설정 (prometheus.yml)
scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8080']
    scrape_interval: 15s
```

---

## 💻 실험으로 확인하기

### 실험 1: 메트릭 조회

```bash
# 메트릭 목록
curl http://localhost:8080/actuator/metrics | jq .names

# 특정 메트릭 값
curl http://localhost:8080/actuator/metrics/jvm.memory.used | jq .
# {"name":"jvm.memory.used","measurements":[{"statistic":"VALUE","value":1.2345E8}],...}

# 태그 필터
curl "http://localhost:8080/actuator/metrics/jvm.memory.used?tag=area:heap"
```

### 실험 2: Counter + Gauge + Timer 기본 예제

```java
@RestController
@RequiredArgsConstructor
public class MetricsController {

    private final MeterRegistry registry;
    private final AtomicInteger activeRequests = new AtomicInteger(0);

    @PostConstruct
    void init() {
        // Gauge: activeRequests 현재 값 모니터링
        Gauge.builder("active.requests", activeRequests, AtomicInteger::get)
            .register(registry);
    }

    @GetMapping("/api/orders")
    public List<Order> getOrders() {
        activeRequests.incrementAndGet();
        try {
            return Timer.builder("orders.fetch.time")
                .register(registry)
                .recordCallable(orderService::findAll);
        } finally {
            activeRequests.decrementAndGet();
            registry.counter("orders.fetch.count").increment();
        }
    }
}
```

---

## ⚙️ 설정 최적화 팁

```yaml
management:
  metrics:
    export:
      prometheus:
        enabled: true
        step: 1m              # Prometheus 스크레이핑 간격 힌트
    tags:
      application: ${spring.application.name}  # 공통 태그
    distribution:
      percentiles-histogram:
        http.server.requests: true  # HTTP 요청 히스토그램 활성화
      slo:
        http.server.requests: 50ms, 100ms, 200ms, 500ms  # SLA 버킷
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
```

---

## 📌 핵심 정리

```
Micrometer 계층
  MeterRegistry (추상) → CompositeMeterRegistry → [Prometheus, Datadog, ...]
  모든 코드는 MeterRegistry에 기록 → 등록된 모든 백엔드에 자동 전달

Meter 타입
  Counter          단조 증가 (요청 수, 오류 수)
  Gauge            현재 상태 스냅샷 (큐 크기, 연결 수)
  Timer            시간 측정 + 통계 (응답시간 p50/p95/p99)
  DistributionSummary  값 분포 (응답 크기)

@Timed 동작
  TimedAspect (AOP) → Around advice → Timer.start/stop
  TimedAspect Bean 직접 등록 필요

Prometheus 흐름
  코드 기록 → MeterRegistry in-memory
  → GET /actuator/prometheus → scrape() → 텍스트 포맷 직렬화

공통 태그
  MeterRegistryCustomizer → registry.config().commonTags(...)
  → 모든 Meter에 자동 적용 (application, environment, region)
```

---

## 🤔 생각해볼 문제

**Q1.** `Counter`와 `Timer`는 둘 다 호출 횟수를 측정한다. 그렇다면 `Timer` 하나로 둘 다 대체할 수 있는가?

**Q2.** `Gauge`에서 대상 객체를 약한 참조로 유지하는 이유는 무엇이며, 이로 인해 어떤 버그가 발생할 수 있는가?

**Q3.** 태그(Tag) 카디널리티가 높으면 왜 메모리 문제가 발생하는가?

> 💡 **해설**
>
> **Q1.** 기술적으로 `Timer`는 `count()`와 `totalTime()`을 모두 제공하므로 횟수 측정도 가능하다. 하지만 목적이 다르다. `Counter`는 단순히 "몇 번 발생했는가"에 최적화되어 메모리 효율이 높다. `Timer`는 시간 측정을 위한 추가 데이터 구조(히스토그램 버킷, 퍼센타일 등)를 유지하므로 단순 카운팅에는 오버헤드가 크다. 응답 시간 측정이 필요하면 `Timer`, 단순 발생 횟수만 필요하면 `Counter`를 사용하는 것이 올바른 선택이다.
>
> **Q2.** `Gauge`가 대상 객체를 강한 참조로 유지하면 GC가 해당 객체를 수집하지 못해 메모리 누수가 발생한다. 예를 들어 `List`를 Gauge로 등록하고 나중에 해당 List를 교체하면 이전 List가 GC되지 않는다. 약한 참조 덕분에 대상 객체가 더 이상 사용되지 않으면 GC 대상이 되고 Gauge도 자동으로 제거된다. 버그 패턴: Gauge 등록 후 람다에서 로컬 변수를 캡처했는데 해당 객체가 GC되면 Gauge 값이 갑자기 사라진다. `StrongReferenceGaugeFunction`으로 강한 참조를 명시할 수 있다.
>
> **Q3.** `MeterRegistry`는 태그 값 조합마다 별도 `Meter` 인스턴스를 생성한다. 태그 값이 `userId=1234`, `userId=5678`처럼 무한히 많아지면 Meter 인스턴스도 무한히 생성된다. 예를 들어 `tag("userId", request.getUserId())`를 사용하면 사용자 수만큼 Timer 인스턴스가 생성되어 메모리 고갈이 발생한다. 태그는 낮은 카디널리티(HTTP 메서드, 엔드포인트 경로, 상태 코드 등 수십 가지 이내)로 제한해야 하며, 사용자 ID, 요청 ID 같은 고유값은 태그로 사용하면 안 된다.

---

<div align="center">

**[⬅️ 이전: Custom Health Indicator](./02-custom-health-indicator.md)** | **[홈으로 🏠](../README.md)** | **[다음: Custom Actuator Endpoint 작성 ➡️](./04-custom-endpoint.md)**

</div>
