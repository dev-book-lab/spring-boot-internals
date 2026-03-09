# @ConfigurationProperties 바인딩 메커니즘 — Binder가 YAML을 Java 객체로 변환하는 경로

---

## 🎯 핵심 질문

- `ConfigurationPropertiesBindingPostProcessor`는 언제 실행되는가?
- `Binder` → `BindHandler` → `BindConverter` 체인은 어떻게 구성되는가?
- 중첩 객체, List, Map 바인딩은 어떻게 처리되는가?
- `@Validated`로 JSR-303 검증이 바인딩과 어떻게 연동되는가?
- 바인딩 실패 시 오류 메시지에 파일 위치가 나오는 이유는?

---

## 🔍 왜 이게 존재하는가

```java
// @Value의 한계
@Value("${spring.datasource.url}")         String url;
@Value("${spring.datasource.username}")    String username;
@Value("${spring.datasource.password}")    String password;
@Value("${spring.datasource.hikari.maximum-pool-size:10}") int poolSize;
// → 키 분산, 리팩터링 어려움, 타입 변환 직접 처리

// @ConfigurationProperties로 해결
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties {
    private String url;
    private String username;
    private String password;
    private Hikari hikari = new Hikari();

    public static class Hikari {
        private int maximumPoolSize = 10;
        // getters, setters
    }
}
// → 타입 안전, 계층 구조, 기본값, IDE 자동완성
```

---

## 🔬 내부 동작 원리

### 1. 처리 시점 — BeanPostProcessor

```java
// ConfigurationPropertiesBindingPostProcessor
// BeanPostProcessor 구현 → Bean 생성 직후 바인딩 실행

@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) {
    // @ConfigurationProperties 어노테이션이 있는 Bean만 처리
    bind(ConfigurationPropertiesBean.get(this.applicationContext, bean, beanName));
    return bean;
}

// 실행 시점:
// finishBeanFactoryInitialization() → getBean() → doCreateBean()
//   → initializeBean() → applyBeanPostProcessorsBeforeInitialization()
//     → ConfigurationPropertiesBindingPostProcessor.postProcessBeforeInitialization()
//       → @PostConstruct 실행 전에 바인딩 완료
```

### 2. Binder — 핵심 바인딩 엔진

```java
// Binder.java — 바인딩 진입점
public class Binder {

    private final Iterable<ConfigurationPropertySource> sources;
    private final PlaceholdersResolver placeholdersResolver;
    private final ConversionService conversionService;
    private final BindHandler defaultBindHandler;

    // 바인딩 시작
    public <T> BindResult<T> bind(String name, Bindable<T> target) {
        return bind(name, target, this.defaultBindHandler);
    }

    public <T> BindResult<T> bind(String name, Bindable<T> target, BindHandler handler) {
        ConfigurationPropertyName configurationPropertyName =
            ConfigurationPropertyName.of(name);  // "spring.datasource" → 정규화

        return bind(configurationPropertyName, target, handler, null, false, false);
    }

    // 실제 바인딩
    private <T> Object bindObject(ConfigurationPropertyName name, Bindable<T> target,
                                   BindHandler handler, Context context, ...) {

        // 1. 직접 값 조회 (스칼라 타입)
        ConfigurationProperty property = findProperty(name, context);
        if (property != null) {
            return bindProperty(target, context, property);
        }

        // 2. 중첩 객체/컬렉션 바인딩
        if (isUnbindableBean(name, target, context)) {
            return null;
        }
        Object bindResult = bindBean(name, target, handler, context, ...);
        if (bindResult != null) {
            return bindResult;
        }

        // 3. 집계 타입 (Map, Collection)
        AggregateBinder<?> aggrBinder = getAggregateBinder(target, context);
        if (aggrBinder != null) {
            return bindAggregate(name, target, handler, context, aggrBinder);
        }
        return null;
    }
}
```

### 3. ConfigurationPropertySource — 프로퍼티 소스 어댑터

```java
// Environment의 PropertySource를 Binder가 이해하는 형식으로 변환
// 핵심: Relaxed Binding 지원

// SpringConfigurationPropertySource → PropertySource 어댑터
// 키 정규화: "spring.datasource.url", "SPRING_DATASOURCE_URL", "spring.datasource-url"
//            모두 동일 키로 처리

// PropertySourcesDeducer가 Environment의 모든 PropertySource를
// ConfigurationPropertySource 목록으로 변환
```

### 4. 중첩 객체 바인딩 — JavaBeanBinder

```java
// JavaBeanBinder — setter 기반 바인딩
class JavaBeanBinder implements DataObjectBinder {

    @Override
    public <T> T bind(ConfigurationPropertyName name, Bindable<T> target,
                       Context context, DataObjectPropertyBinder propertyBinder) {
        // ① 대상 클래스의 모든 setter 메서드 탐색
        BeanSupplier<T> beanSupplier = BeanSupplier.of(target);
        Bean<T> bean = Bean.get(target, hasKnownBindableProperties(name, context));

        // ② 각 프로퍼티명으로 하위 키 조회 및 바인딩
        boolean bound = false;
        for (Map.Entry<String, BeanProperty> entry : bean.getProperties().entrySet()) {
            bound |= bindProperty(name, context, propertyBinder, beanSupplier, entry);
        }
        return (bound ? beanSupplier.get() : null);
    }

    private boolean bindProperty(ConfigurationPropertyName name, Context context,
                                   DataObjectPropertyBinder propertyBinder,
                                   BeanSupplier<T> beanSupplier,
                                   Map.Entry<String, BeanProperty> entry) {
        // "spring.datasource" + "url" → "spring.datasource.url" 키로 조회
        String propertyName = entry.getKey();
        BeanProperty property = entry.getValue();

        Object bound = propertyBinder.bindProperty(propertyName,
            Bindable.of(property.getType()));

        if (bound != null) {
            // setter 호출: setUrl(value)
            property.setValue(beanSupplier, bound);
            return true;
        }
        return false;
    }
}
```

### 5. 컬렉션 바인딩 — CollectionBinder

```yaml
# YAML
my:
  servers:
    - host: server1
      port: 8080
    - host: server2
      port: 9090
  tags:
    - web
    - api
```

```java
// CollectionBinder — List 바인딩
// "my.servers[0].host" → "my.servers[1].host" 순서로 탐색
// 각 원소를 중첩 객체로 재귀 바인딩

@ConfigurationProperties(prefix = "my")
public class MyProperties {
    private List<Server> servers = new ArrayList<>();
    private List<String> tags = new ArrayList<>();

    public static class Server {
        private String host;
        private int port;
        // getters, setters
    }
}
```

```java
// MapBinder — Map 바인딩
// "my.features.feature-a=true" → Map<String, Boolean>

@ConfigurationProperties(prefix = "my")
public class MyProperties {
    private Map<String, Boolean> features = new HashMap<>();
    // my.features.feature-a=true → features.put("feature-a", true)
}
```

### 6. BindConverter — 타입 변환

```java
// BindConverter — ConversionService + TypeDescriptor 기반 변환
class BindConverter {

    Object convert(Object value, ResolvableType type, Annotation... annotations) {
        if (value == null) return null;

        // String → 대상 타입 변환
        // Spring ConversionService의 Converter 체인 활용

        // 내장 변환:
        //   String → Integer, Long, Double 등 기본 타입
        //   String → Duration  ("30s", "5m", "1h" → Duration.ofSeconds(30) 등)
        //   String → DataSize  ("512MB", "1GB" → DataSize.ofMegabytes(512) 등)
        //   String → Period    ("1d", "2w" → Period.ofDays(1) 등)
        //   String → Charset   ("UTF-8" → Charset.forName("UTF-8"))
        //   String → InetAddress, URI, URL
        //   String → Enum (대소문자 무시)
        //   String → Class     (클래스명 → Class.forName())
    }
}
```

```yaml
# Duration, DataSize 바인딩 예시
my:
  timeout: 30s        # → Duration.ofSeconds(30)
  max-timeout: 5m     # → Duration.ofMinutes(5)
  buffer-size: 512MB  # → DataSize.ofMegabytes(512)
  cache-ttl: 1h30m    # → Duration (ISO-8601 형식도 지원: PT1H30M)
```

### 7. @Validated — 바인딩 후 검증

```java
// @Validated를 @ConfigurationProperties 클래스에 붙이면
// 바인딩 완료 후 JSR-303(Bean Validation) 검증 실행

@ConfigurationProperties(prefix = "my.service")
@Validated
public class ServiceProperties {

    @NotBlank
    private String apiKey;

    @Min(1) @Max(100)
    private int threadCount = 10;

    @NotNull
    @Valid  // 중첩 객체도 검증
    private Database database = new Database();

    public static class Database {
        @NotBlank
        private String url;

        @Pattern(regexp = "\\d{4,5}")
        private String port;
        // getters, setters
    }
}
```

```java
// 검증 실패 시:
// ConfigurationPropertiesBindingPostProcessor →
//   ValidationBindHandler.onFinish() →
//     Validator.validate() →
//       ConstraintViolationException 발생
//         → 애플리케이션 시작 실패, 명확한 오류 메시지

// 오류 예시:
// Binding to target ServiceProperties failed:
//   Property: my.service.api-key
//   Value: ""
//   Reason: must not be blank
```

---

## 💻 실험으로 확인하기

### 실험 1: 바인딩 디버깅

```yaml
logging:
  level:
    org.springframework.boot.context.properties: TRACE
# TRACE 출력으로 각 프로퍼티 바인딩 과정 추적 가능
```

### 실험 2: Duration 바인딩

```java
@ConfigurationProperties(prefix = "my")
public class MyProps {
    private Duration timeout = Duration.ofSeconds(30);  // 기본값
}
```

```yaml
my:
  timeout: 1m      # Duration.ofMinutes(1)
  timeout: PT1M30S # Duration.ofMinutes(1).plusSeconds(30)
  timeout: 60000   # Long으로 읽으면 ms로 해석 → @DurationUnit(ChronoUnit.MILLIS)
```

### 실험 3: @Validated 오류 메시지

```yaml
my:
  service:
    api-key: ""     # @NotBlank 위반
    thread-count: 0 # @Min(1) 위반
```

```
시작 실패 로그:
***************************
APPLICATION FAILED TO START
***************************
Description:
  Binding to target ServiceProperties failed:
    Property: my.service.api-key
    Value: ""
    Origin: class path resource [application.yml] - 3:14
    Reason: must not be blank

    Property: my.service.thread-count
    Value: 0
    Reason: must be greater than or equal to 1
```

---

## 📌 핵심 정리

```
처리 시점
  Bean 생성 후, @PostConstruct 전
  ConfigurationPropertiesBindingPostProcessor (BeanPostProcessor)

바인딩 체인
  Binder → ConfigurationPropertySource (Relaxed Binding 정규화)
  → JavaBeanBinder (중첩 객체, setter 기반)
  → CollectionBinder / MapBinder (컬렉션/맵)
  → BindConverter (타입 변환: Duration, DataSize 등)

검증 통합
  @Validated + JSR-303 어노테이션
  → 바인딩 완료 후 즉시 검증
  → 실패 시 시작 실패 + 명확한 오류 위치

오류 추적
  OriginTrackedMapPropertySource
  → 파일명 + 라인 번호 → 바인딩 실패 시 정확한 위치 출력
```

---

## 🤔 생각해볼 문제

**Q1.** `@ConfigurationProperties` 클래스에 `@Component`를 붙이는 방식과 `@EnableConfigurationProperties`로 등록하는 방식의 차이는 무엇인가?

**Q2.** `@ConfigurationProperties(prefix = "my")` 클래스에 setter가 없는 final 필드만 있으면 바인딩이 실패하는가?

**Q3.** `List<String>` 필드에 `application.properties`에서 `my.tags=a,b,c`처럼 콤마 구분 문자열로 바인딩하면 어떻게 처리되는가?

> 💡 **해설**
>
> **Q1.** `@Component`를 붙이면 `@ComponentScan`으로 Bean 등록되고 이후 `ConfigurationPropertiesBindingPostProcessor`가 바인딩한다. `@EnableConfigurationProperties(MyProps.class)`를 사용하면 `@ComponentScan` 없이도 Bean이 등록되어 라이브러리처럼 스캔 범위 밖에서도 사용할 수 있다. Auto-configuration 클래스에서는 `@EnableConfigurationProperties` 방식이 권장된다. `@Component`를 붙이면 `AutoConfigurationExcludeFilter`가 없는 `@ComponentScan`에서 예상치 못하게 스캔될 수 있다.
>
> **Q2.** 실패하지 않는다. Spring Boot 2.2+에서 생성자 기반 바인딩을 지원한다. 단일 생성자가 있고 모든 파라미터가 프로퍼티에 대응하면 `ValueObjectBinder`가 생성자를 통해 바인딩한다. `@ConstructorBinding`(Boot 2.x) 또는 Java `record`(Boot 2.6+)를 사용하면 더 명시적으로 불변 바인딩이 가능하다.
>
> **Q3.** `BindConverter`가 `String → List<String>` 변환 시 콤마 구분자로 자동 분리한다. `"a,b,c"` → `["a", "b", "c"]`. 단, YAML에서는 시퀀스(`- a\n- b`) 형식을 권장하며, 콤마 구분 문자열은 `.properties` 포맷에서 주로 사용한다. YAML에서 `my.tags: a,b,c`처럼 쓰면 문자열로 읽혀 동일하게 분리된다.

---

<div align="center">

**[⬅️ 이전: application.properties vs yml](./01-application-properties-yml.md)** | **[홈으로 🏠](../README.md)** | **[다음: Relaxed Binding ➡️](./03-relaxed-binding.md)**

</div>
