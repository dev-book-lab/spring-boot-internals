# Actuator Endpoint 구조 — @Endpoint에서 HTTP 경로까지 변환 과정

---

## 🎯 핵심 질문

- `@Endpoint`, `@WebEndpoint`, `@JmxEndpoint`의 차이는?
- `EndpointDiscoverer`가 Endpoint Bean을 발견하고 HTTP 경로로 변환하는 과정은?
- `@ReadOperation`, `@WriteOperation`이 HTTP 메서드로 매핑되는 원리는?
- `management.endpoints.web.exposure.include` 필터가 적용되는 정확한 시점은?
- Endpoint 응답은 어떻게 직렬화되는가?

---

## 🔍 왜 이게 존재하는가

```
Actuator의 설계 목표:
  하나의 Endpoint 정의 → HTTP와 JMX 양쪽으로 자동 노출

@Endpoint("health")
  → HTTP: GET /actuator/health
  → JMX:  org.springframework.boot:type=Endpoint,name=Health

@WebEndpoint("docs")   ← HTTP 전용
  → HTTP: GET /actuator/docs
  → JMX: 노출 안 됨

@JmxEndpoint("gc")    ← JMX 전용
  → HTTP: 노출 안 됨
  → JMX: ...Endpoint,name=Gc
```

---

## 🔬 내부 동작 원리

### 1. Endpoint 어노테이션 계층

```java
// @Endpoint — 기술 독립적 (HTTP + JMX 모두)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Endpoint {
    String id();                    // "health", "info", "metrics" 등
    boolean enableByDefault() default true;
}

// @WebEndpoint — HTTP 전용
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@EndpointExtension(filter = WebEndpointFilter.class)
public @interface WebEndpoint {
    String id();
}

// @JmxEndpoint — JMX 전용
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@EndpointExtension(filter = JmxEndpointFilter.class)
public @interface JmxEndpoint {
    String id();
}

// @EndpointWebExtension — 기존 @Endpoint에 Web 전용 기능 추가
// (예: HealthEndpoint의 HTTP 응답 커스터마이징)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@EndpointExtension(filter = WebEndpointFilter.class)
public @interface EndpointWebExtension {
    Class<?> endpoint();  // 확장 대상 Endpoint 클래스
}
```

### 2. EndpointDiscoverer — Endpoint Bean 탐색

```java
// WebEndpointDiscoverer (HTTP용) 처리 흐름
public class WebEndpointDiscoverer
        extends EndpointDiscoverer<ExposableWebEndpoint, WebOperation> {

    @Override
    protected ExposableWebEndpoint createEndpoint(Object endpointBean,
            EndpointId id, boolean enabledByDefault,
            Collection<WebOperation> operations) {
        return new DiscoveredWebEndpoint(this, endpointBean, id,
                this.requestMappingFactory, enabledByDefault, operations);
    }
}

// EndpointDiscoverer 핵심 로직
protected Collection<ExposableEndpoint<O>> discoverEndpoints() {
    // ① ApplicationContext에서 @Endpoint Bean 탐색
    Collection<EndpointBean> endpointBeans = discoverEndpointBeans();

    // ② @EndpointWebExtension Bean 탐색 (확장 있으면 합침)
    Map<EndpointId, EndpointExtensionBean> extensionBeans = discoverExtensionBeans();

    // ③ Operation 추출 (메서드 레벨 @ReadOperation 등 파싱)
    Collection<ExposableEndpoint<O>> endpoints = new ArrayList<>();
    for (EndpointBean endpointBean : endpointBeans) {
        // @ReadOperation, @WriteOperation, @DeleteOperation 메서드 탐색
        Collection<O> operations = createOperations(endpointBean, extensionBeans);
        if (isEndpointExposed(endpointBean)) {  // exposure 필터 적용
            endpoints.add(createEndpoint(endpointBean.getBean(),
                endpointBean.getId(), endpointBean.isEnabledByDefault(), operations));
        }
    }
    return endpoints;
}
```

### 3. Operation 메서드 → HTTP 매핑

```java
// @ReadOperation → HTTP GET
// @WriteOperation → HTTP POST (body 있으면) 또는 POST (body 없으면)
// @DeleteOperation → HTTP DELETE

@Endpoint(id = "myendpoint")
public class MyEndpoint {

    @ReadOperation
    public MyData read() { ... }
    // → GET /actuator/myendpoint

    @ReadOperation
    public MyData readWithParam(@Selector String name) { ... }
    // → GET /actuator/myendpoint/{name}

    @WriteOperation
    public void write(@Nullable String data) { ... }
    // → POST /actuator/myendpoint  (body: {"data": "..."})

    @DeleteOperation
    public void delete(@Selector String name) { ... }
    // → DELETE /actuator/myendpoint/{name}
}
```

```java
// OperationInvoker 체인:
// WebMvcEndpointHandlerMapping → RequestMappingHandlerMapping 등록
//   → HTTP 요청 수신
//   → AbstractWebMvcEndpointHandlerMapping.handleRequest()
//   → EndpointInvocationContext 생성
//   → OperationInvoker.invoke(context)
//   → 실제 @ReadOperation 메서드 실행
//   → 반환값 → EndpointObjectMapper로 JSON 직렬화
//   → HTTP 응답

// 파라미터 바인딩:
// - @Selector → PathVariable
// - 나머지 → RequestParam (GET) 또는 RequestBody (POST)
// - @Nullable → optional 파라미터
```

### 4. Exposure 필터링 — 어느 시점에 적용되는가

```java
// management.endpoints.web.exposure.include=health,info,metrics
// → WebEndpointDiscoverer.isEndpointExposed()에서 체크

// EndpointDiscoverer.isEndpointExposed()
private boolean isEndpointExposed(EndpointBean endpointBean) {
    EndpointFilter<ExposableEndpoint<O>> filter = getFilter();
    if (filter == null) return true;

    // ExposureEndpointFilter — include/exclude 설정 기반 필터
    return filter.match(endpointBean.asEndpoint());
}

// ExposureEndpointFilter
public boolean match(ExposableEndpoint<?> endpoint) {
    EndpointId id = endpoint.getEndpointId();
    // include 목록에 있는가?
    if (this.include.contains("*") || this.include.contains(id.toLowerCaseString())) {
        // exclude 목록에 없는가?
        return !this.exclude.contains(id.toLowerCaseString());
    }
    return false;
}
```

```yaml
# Exposure 설정
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus  # 명시적 포함
        exclude: env,beans                        # 명시적 제외
      base-path: /actuator                        # 기본 경로
    jmx:
      exposure:
        include: "*"   # JMX는 기본 전체 노출
```

### 5. WebMvcEndpointHandlerMapping — HTTP 경로 등록

```java
// WebMvcEndpointHandlerMapping
// → AbstractHandlerMapping을 상속해 Spring MVC에 통합

// 초기화 시 모든 ExposableWebEndpoint를 HandlerMapping에 등록
@Override
protected void initHandlerMethods() {
    for (ExposableWebEndpoint endpoint : this.endpoints) {
        for (WebOperation operation : endpoint.getOperations()) {
            // WebOperationRequestPredicate → RequestMappingInfo 변환
            // /actuator/{endpoint-id}/{path} 패턴으로 등록
            registerMapping(endpoint, operation);
        }
    }
}
// → DispatcherServlet의 HandlerMapping 목록에 추가
// → 일반 @RequestMapping과 동일하게 요청 처리
```

### 6. Endpoint 응답 직렬화

```java
// EndpointObjectMapper — Actuator 전용 ObjectMapper
// 일반 앱 ObjectMapper와 분리 → 커스텀 직렬화 규칙 적용

// Health 응답 예시:
// {"status":"UP","components":{"db":{"status":"UP","details":{"database":"H2"}}}}

// 커스텀 직렬화:
// Health → HealthEndpointWebExtension이 상태에 따라 HTTP 상태 코드 결정
// UP → 200 OK
// DOWN → 503 Service Unavailable
// OUT_OF_SERVICE → 503
// UNKNOWN → 200

// management.endpoint.health.show-details=always → details 포함
// management.endpoint.health.show-details=when-authorized → 인증 시 포함
```

---

## 💻 실험으로 확인하기

### 실험 1: 등록된 Endpoint 목록 확인

```bash
# GET /actuator → 모든 노출된 Endpoint 링크 반환
curl http://localhost:8080/actuator | jq .

# 응답:
# {
#   "_links": {
#     "health": {"href": "http://localhost:8080/actuator/health"},
#     "info": {"href": "http://localhost:8080/actuator/info"},
#     "metrics": {"href": "http://localhost:8080/actuator/metrics"}
#   }
# }
```

### 실험 2: Endpoint 탐색 로그 활성화

```yaml
logging:
  level:
    org.springframework.boot.actuate.endpoint: TRACE
# → Endpoint 탐색, Operation 등록 과정 상세 출력
```

### 실험 3: 전체 Endpoint 임시 활성화 (개발용)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"  # 모든 Endpoint 노출 (운영 금지)
  endpoint:
    health:
      show-details: always
    env:
      show-values: always  # 프로퍼티 값 노출 (민감정보 주의)
```

---

## 📌 핵심 정리

```
어노테이션 선택
  @Endpoint         HTTP + JMX 양쪽 (기술 독립)
  @WebEndpoint      HTTP 전용
  @JmxEndpoint      JMX 전용
  @EndpointWebExtension  기존 @Endpoint의 HTTP 동작 확장

HTTP 매핑
  @ReadOperation   → GET
  @WriteOperation  → POST
  @DeleteOperation → DELETE
  @Selector        → PathVariable ({name} 세그먼트)

처리 흐름
  @Endpoint Bean 탐색 (EndpointDiscoverer)
  → Exposure 필터 적용 (include/exclude 설정)
  → WebMvcEndpointHandlerMapping에 경로 등록
  → 요청 수신 → OperationInvoker → 메서드 실행 → JSON 직렬화

기본 Exposure
  HTTP: health, info만 (Boot 기본)
  JMX: 전체 (Boot 기본)
```

---

## 🤔 생각해볼 문제

**Q1.** `@Endpoint(id = "health")`와 `@EndpointWebExtension`을 동시에 정의했을 때, HTTP 요청은 어느 클래스의 메서드로 라우팅되는가?

**Q2.** `management.endpoints.web.exposure.include=*`로 전체 노출 시 `beans`, `env` Endpoint가 위험한 이유는 무엇인가?

**Q3.** Actuator Endpoint의 `@ReadOperation` 메서드가 `Mono<T>`를 반환하면 어떻게 처리되는가?

> 💡 **해설**
>
> **Q1.** `@EndpointWebExtension`의 `@ReadOperation` 메서드가 우선한다. `WebEndpointDiscoverer`는 같은 `id`를 가진 `@Endpoint`와 `@EndpointWebExtension`을 발견하면 Extension의 Operation으로 교체한다. 즉 `HealthEndpointWebExtension.health()`가 `HealthEndpoint.health()`를 대신해 HTTP 요청을 처리한다. JMX에서는 `@EndpointWebExtension`이 무시되고 원래 `@Endpoint`의 메서드가 사용된다.
>
> **Q2.** `beans` Endpoint는 애플리케이션에 등록된 모든 Bean의 이름, 타입, 의존 관계를 노출한다. 공격자가 이를 보면 애플리케이션 구조를 파악해 취약한 Bean을 탐색할 수 있다. `env` Endpoint는 모든 PropertySource 값을 노출하는데, DB 비밀번호, API 키, JWT Secret 등 민감한 환경변수가 포함될 수 있다. `show-values: always`로 설정하면 마스킹 없이 평문으로 노출된다.
>
> **Q3.** Spring Boot Actuator는 반응형 타입(`Mono<T>`, `Flux<T>`)을 지원한다. WebFlux 환경에서는 그대로 처리되고, Spring MVC 환경에서는 `ReactiveOperationInvokerAdvisor`가 `Mono`를 블로킹 방식으로 구독(`block()`)해 결과를 반환한다. Actuator의 `@ReadOperation`이 `Mono`를 반환하도록 작성하면 WebFlux와 MVC 양쪽에서 동작하지만, MVC에서의 `block()` 호출은 주의가 필요하다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Custom Health Indicator 작성 ➡️](./02-custom-health-indicator.md)**

</div>
