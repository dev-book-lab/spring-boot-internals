# Custom Actuator Endpoint 작성 — @ReadOperation부터 파라미터 바인딩까지

---

## 🎯 핵심 질문

- `@ReadOperation`, `@WriteOperation`, `@DeleteOperation`이 HTTP 메서드로 변환되는 규칙은?
- `@Selector`와 `@Selector(match = ALL_REMAINING)`의 차이는?
- `@Nullable`로 선택적 파라미터를 처리하는 방법은?
- 반환 타입이 `void`, `Map`, `@Nullable Object`일 때 HTTP 응답이 어떻게 다른가?
- `@FilteredEndpoint`로 Security 통합하는 방법은?
- Endpoint를 비활성화하거나 조건부로 활성화하는 방법은?

---

## 🔬 내부 동작 원리

### 1. Operation 어노테이션 → HTTP 매핑 규칙

```java
// @ReadOperation → HTTP GET
// @WriteOperation → HTTP POST (RequestBody가 있을 때)
//                   HTTP POST (RequestBody 없을 때도 POST)
// @DeleteOperation → HTTP DELETE

// 파라미터 처리 규칙:
// @Selector → PathVariable
// @Nullable 파라미터 → 선택적 QueryParam 또는 RequestBody 필드
// 나머지 → GET이면 QueryParam, POST이면 RequestBody

@Endpoint(id = "cache")
public class CacheEndpoint {

    private final CacheManager cacheManager;

    // GET /actuator/cache — 모든 캐시 목록
    @ReadOperation
    public Map<String, Object> caches() {
        Map<String, Object> result = new LinkedHashMap<>();
        cacheManager.getCacheNames().forEach(name -> {
            Cache cache = cacheManager.getCache(name);
            result.put(name, Map.of("size", getCacheSize(cache)));
        });
        return result;
    }

    // GET /actuator/cache/{cacheName} — 특정 캐시 정보
    @ReadOperation
    public Map<String, Object> cache(@Selector String cacheName) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache == null) {
            return null;  // null 반환 → 404 Not Found
        }
        return Map.of(
            "name", cacheName,
            "size", getCacheSize(cache),
            "nativeCache", cache.getNativeCache().getClass().getSimpleName()
        );
    }

    // DELETE /actuator/cache/{cacheName} — 특정 캐시 비우기
    @DeleteOperation
    public void evictCache(@Selector String cacheName) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.clear();
        }
    }

    // POST /actuator/cache — 전체 캐시 비우기 (WriteOperation)
    @WriteOperation
    public void evictAll() {
        cacheManager.getCacheNames()
            .forEach(name -> {
                Cache cache = cacheManager.getCache(name);
                if (cache != null) cache.clear();
            });
    }
}
```

### 2. @Selector 고급 사용

```java
@Endpoint(id = "features")
public class FeatureFlagEndpoint {

    private final FeatureFlags flags;

    // GET /actuator/features/flags/my-feature
    // @Selector → PathVariable 단일 세그먼트
    @ReadOperation
    public FeatureStatus getFeature(@Selector String featureName) {
        return flags.getStatus(featureName);
    }

    // GET /actuator/features/config/app/module/setting
    // ALL_REMAINING → 슬래시 포함 나머지 경로 전체 캡처
    @ReadOperation
    public Object getNestedConfig(
            @Selector(match = Selector.Match.ALL_REMAINING) String path) {
        // path = "app/module/setting"
        return flags.getByPath(path);
    }

    // GET /actuator/features?group=payment&active=true
    // @Nullable → 선택적 QueryParam
    @ReadOperation
    public List<String> listFeatures(
            @Nullable String group,
            @Nullable Boolean active) {
        return flags.list(group, active);
    }
}
```

### 3. 반환 타입과 HTTP 상태 코드

```java
// 반환 타입 → HTTP 응답 매핑:

// ① 값 반환 → 200 OK + JSON body
@ReadOperation
public Map<String, Object> data() {
    return Map.of("key", "value");
}
// → 200 OK {"key":"value"}

// ② null 반환 → 404 Not Found
@ReadOperation
@Nullable
public Object maybeData(@Selector String id) {
    return repository.findById(id).orElse(null);
}
// → 존재하면 200 OK, 없으면 404 Not Found

// ③ void 반환 → 204 No Content
@WriteOperation
public void update(String key, String value) {
    store.put(key, value);
}
// → 204 No Content

// ④ WebEndpointResponse — HTTP 상태 코드 직접 제어
@ReadOperation
public WebEndpointResponse<Map<String, Object>> controlled(@Selector String id) {
    if (!isValid(id)) {
        return new WebEndpointResponse<>(Map.of("error", "Invalid ID"), 400);
    }
    return new WebEndpointResponse<>(getData(id), 200);
}
```

### 4. 완성된 커스텀 Endpoint 예제 — FeatureFlag

```java
@Endpoint(id = "featureflags")
@Component
public class FeatureFlagEndpoint {

    private final Map<String, Boolean> flags = new ConcurrentHashMap<>();

    // GET /actuator/featureflags
    @ReadOperation
    public Map<String, Boolean> flags() {
        return Collections.unmodifiableMap(flags);
    }

    // GET /actuator/featureflags/{name}
    @ReadOperation
    @Nullable
    public Map<String, Object> flag(@Selector String name) {
        Boolean value = flags.get(name);
        if (value == null) return null;  // 404
        return Map.of("name", name, "enabled", value);
    }

    // POST /actuator/featureflags — body: {"name":"my-feature","enabled":true}
    @WriteOperation
    public void setFlag(String name, boolean enabled) {
        flags.put(name, enabled);
    }

    // DELETE /actuator/featureflags/{name}
    @DeleteOperation
    public void removeFlag(@Selector String name) {
        flags.remove(name);
    }
}
```

```yaml
# 활성화 설정
management:
  endpoints:
    web:
      exposure:
        include: featureflags,health,info

  endpoint:
    featureflags:
      enabled: true  # false로 비활성화 가능
      cache:
        time-to-live: 10s  # @ReadOperation 결과 캐싱
```

### 5. @WebEndpoint — HTTP 전용, 더 세밀한 제어

```java
// @WebEndpoint — produces, consumes 등 HTTP 세부 제어
@WebEndpoint(id = "report")
@Component
public class ReportEndpoint {

    // produces 지정 — Content-Type 명시
    @ReadOperation(produces = MediaType.TEXT_HTML_VALUE)
    public String htmlReport() {
        return "<html><body><h1>Report</h1></body></html>";
    }

    // 특정 경로 파라미터 조합
    @ReadOperation
    public byte[] downloadReport(
            @Selector String format,
            @Nullable String from,
            @Nullable String to) {
        // GET /actuator/report/pdf?from=2024-01&to=2024-12
        return reportService.generate(format, from, to);
    }
}
```

### 6. EndpointWebExtension — 기존 Endpoint 확장

```java
// 기존 HealthEndpoint의 HTTP 동작만 커스터마이징
@EndpointWebExtension(endpoint = HealthEndpoint.class)
@Component
public class CustomHealthEndpointWebExtension {

    private final HealthEndpoint delegate;
    private final HealthEndpointWebExtension original;

    @ReadOperation
    public WebEndpointResponse<HealthComponent> health(
            @Nullable ApiVersion apiVersion,
            @Nullable SecurityContext securityContext,
            ShowDetails showDetails,
            @Nullable @Selector(match = Selector.Match.ALL_REMAINING) String[] path) {

        // 원본 동작 + 커스텀 헤더 추가 등
        WebEndpointResponse<HealthComponent> response =
            original.health(apiVersion, securityContext, showDetails, path);

        // 커스텀 로직 (예: 특정 상태 숨기기)
        return response;
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 커스텀 Endpoint 동작 테스트

```java
@WebMvcTest(controllers = {})
@AutoConfigureWebMvc
@Import({FeatureFlagEndpoint.class, WebMvcEndpointManagementContextConfiguration.class})
class FeatureFlagEndpointTest {

    @Autowired
    MockMvc mockMvc;

    @Test
    void readAllFlags() throws Exception {
        mockMvc.perform(get("/actuator/featureflags"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$").isMap());
    }

    @Test
    void setAndGetFlag() throws Exception {
        mockMvc.perform(post("/actuator/featureflags")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"name":"test-feature","enabled":true}"""))
            .andExpect(status().isNoContent());

        mockMvc.perform(get("/actuator/featureflags/test-feature"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.enabled").value(true));
    }

    @Test
    void unknownFlagReturns404() throws Exception {
        mockMvc.perform(get("/actuator/featureflags/unknown"))
            .andExpect(status().isNotFound());
    }
}
```

### 실험 2: Endpoint 캐싱

```yaml
# @ReadOperation 결과 캐싱 — 잦은 조회 최적화
management:
  endpoint:
    featureflags:
      cache:
        time-to-live: 30s  # 30초 동안 결과 캐싱
```

```java
// 캐시 무효화: WriteOperation/DeleteOperation 호출 시 자동 무효화
// → @WriteOperation 실행 후 캐시 초기화
```

---

## ⚙️ 설정 최적화 팁

```java
// Endpoint ID 규칙
// - 소문자, 숫자, 하이픈만 허용
// - Spring Boot 내장 이름과 충돌 금지
//   (health, info, metrics, env, beans, mappings, ...)

// 읽기 전용 Endpoint → @ReadOperation만 정의 + @Endpoint
// 쓰기/삭제 포함 → 보안 설정으로 WRITE/DELETE 접근 제한 필수

// 운영 환경: WriteOperation/DeleteOperation은 ACTUATOR_ADMIN 역할 제한
```

---

## 📌 핵심 정리

```
Operation → HTTP 매핑
  @ReadOperation   → GET
  @WriteOperation  → POST (파라미터 유무 무관)
  @DeleteOperation → DELETE

파라미터 → HTTP 변환
  @Selector                → PathVariable
  @Selector(ALL_REMAINING) → 슬래시 포함 경로
  @Nullable                → 선택적 (없어도 OK)
  나머지 GET              → QueryParam
  나머지 POST             → RequestBody 필드

반환 → HTTP 상태
  값 반환    → 200 OK
  null 반환  → 404 Not Found
  void 반환  → 204 No Content
  WebEndpointResponse → 직접 코드 지정

비활성화
  management.endpoint.{id}.enabled=false
  또는 @ConditionalOnProperty로 조건부 등록
```

---

## 🤔 생각해볼 문제

**Q1.** `@Endpoint`로 정의한 Endpoint의 `@WriteOperation`에 파라미터가 없으면 HTTP 어떻게 처리되는가?

**Q2.** `@Endpoint`와 `@WebEndpoint`를 같은 `id`로 정의했을 때 어떤 것이 HTTP에서 사용되는가?

**Q3.** `management.endpoint.{id}.cache.time-to-live`를 설정하면 `@WriteOperation`이 실행된 후 캐시가 즉시 무효화되는가, 아니면 TTL이 만료될 때까지 기다려야 하는가?

> 💡 **해설**
>
> **Q1.** 파라미터 없는 `@WriteOperation`은 `POST /actuator/{id}` 경로로 매핑된다. RequestBody가 없거나 빈 JSON `{}`을 보내면 된다. `curl -X POST http://localhost:8080/actuator/featureflags`처럼 body 없이 POST하면 처리된다. 파라미터가 없으므로 Body 파싱은 일어나지 않고 메서드가 바로 실행된다.
>
> **Q2.** `@WebEndpoint`(또는 `@EndpointWebExtension`)가 우선한다. `WebEndpointDiscoverer`는 같은 `id`를 가진 `@Endpoint`와 `@WebEndpoint`를 발견하면 `@WebEndpoint`의 Operation으로 대체한다. `@Endpoint`의 Operations는 JMX에서 사용되고, HTTP에서는 `@WebEndpoint`의 Operations가 사용된다. 이 패턴을 통해 HTTP와 JMX의 동작을 독립적으로 커스터마이징할 수 있다.
>
> **Q3.** `@WriteOperation` 또는 `@DeleteOperation` 실행 시 해당 Endpoint의 캐시가 즉시 무효화된다. `CachingOperationInvoker`가 write/delete operation 실행 전에 캐시를 clear하도록 구현되어 있다. 따라서 `@WriteOperation`으로 데이터를 변경한 후 바로 `@ReadOperation`을 호출하면 캐시된 이전 값이 아닌 최신 값을 반환한다.

---

<div align="center">

**[⬅️ 이전: Micrometer 통합](./03-micrometer-metrics.md)** | **[홈으로 🏠](../README.md)** | **[다음: JMX vs HTTP Exposure ➡️](./05-jmx-vs-http-exposure.md)**

</div>
