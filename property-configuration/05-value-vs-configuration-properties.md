# @Value vs @ConfigurationProperties 비교 — 바인딩 시점과 타입 안전성 트레이드오프

---

## 🎯 핵심 질문

- `@Value`와 `@ConfigurationProperties`의 바인딩 시점 차이는?
- `@Value`에서 SpEL 표현식과 프로퍼티 플레이스홀더의 차이는?
- `@ConfigurationProperties`가 리팩터링에 유리한 구체적 이유는?
- `@Value`를 반드시 사용해야 하는 경우는?
- `@Value`가 `static` 필드에 작동하지 않는 이유는?

---

## 🔍 핵심 비교

```
                    @Value              @ConfigurationProperties
───────────────────────────────────────────────────────────────
바인딩 시점         Bean 생성 시        BeanPostProcessor (초기화 전)
Relaxed Binding    ❌ 없음             ✅ 있음
타입 변환          제한적              풍부 (Duration, DataSize...)
컬렉션 바인딩       불편               자연스러움
중첩 객체          불가               자연스러움
기본값 표현         ${key:default}      @DefaultValue 또는 필드 초기화
IDE 자동완성        키 문자열 → 없음    메타데이터로 지원
검증               없음               @Validated + JSR-303
테스트 용이성       낮음 (키 문자열)    높음 (타입 기반)
사용 위치           모든 Bean          @Component 계층 또는 직접 등록
```

---

## 🔬 내부 동작 원리

### 1. @Value 바인딩 메커니즘

```java
// @Value 처리: AutowiredAnnotationBeanPostProcessor
// → postProcessProperties() → InjectionMetadata.inject()
//   → AutowiredFieldElement.inject()
//     → beanFactory.resolveEmbeddedValue("${server.port:8080}")
//       → AbstractBeanFactory.resolveEmbeddedValue()
//         → EmbeddedValueResolver → PropertySourcesPropertyResolver
//           → Environment.getProperty("server.port") → "8080"
//             → TypeConverter.convertIfNecessary(String, int.class) → 8080

@Component
public class MyService {

    // ① 프로퍼티 플레이스홀더 — Environment 조회
    @Value("${server.port}")
    private int port;

    @Value("${server.host:localhost}")  // 기본값
    private String host;

    // ② SpEL 표현식 — #{} 로 계산식 가능
    @Value("#{${server.port} + 1}")
    private int nextPort;

    @Value("#{T(java.lang.Math).random() * 100}")
    private double randomValue;

    @Value("#{systemProperties['user.home']}")
    private String userHome;

    // ③ 리소스 주입
    @Value("classpath:templates/email.html")
    private Resource emailTemplate;
}
```

```java
// @Value 처리 순서 (BeanFactoryPostProcessor 이후):
// 1. Bean 인스턴스 생성 (생성자)
// 2. AutowiredAnnotationBeanPostProcessor.postProcessProperties()
//    → @Value("${key}") 필드에 값 주입
// 3. @PostConstruct 실행
//
// 문제: BeanFactoryPostProcessor 내부에서 @Value 사용 시
//   → BeanFactoryPostProcessor는 Bean 초기화보다 앞서 실행
//   → PropertySourcesPlaceholderConfigurer 미실행 가능
//   → @Value 미작동 → null 또는 "${key}" 문자열 그대로
```

### 2. @Value가 실패하는 경우들

```java
// ① static 필드 — 작동 안 함
@Component
public class MyConfig {
    @Value("${app.name}")
    private static String APP_NAME;  // null — static 필드는 인스턴스 초기화 전에 로딩
    // 해결: @PostConstruct에서 인스턴스 필드 → static으로 복사 (안티패턴)
    //       또는 @ConfigurationProperties 사용
}

// ② BeanFactoryPostProcessor에서 사용
@Component
public class MyBFPP implements BeanFactoryPostProcessor {
    @Value("${app.name}")
    private String appName;  // null — BFPP 처리 시점에 @Value 아직 처리 안 됨
    // 해결: postProcessBeanFactory() 내에서 environment.getProperty()로 직접 조회
}

// ③ 존재하지 않는 키 — 기본값 없으면 예외
@Value("${non.existent.key}")  // IllegalArgumentException: Could not resolve placeholder
@Value("${non.existent.key:}")  // 빈 문자열로 기본값 설정 (안전)
@Value("${non.existent.key:#{null}}")  // null로 기본값 설정

// ④ Relaxed Binding 없음 — 정확한 키 필요
// application.yml: my.max-pool-size=10
@Value("${my.maxPoolSize}")      // ❌ null (camelCase 매핑 없음)
@Value("${my.max-pool-size}")    // ✅ 10
```

### 3. @ConfigurationProperties가 리팩터링에 유리한 이유

```java
// 프로퍼티 키가 코드 내 문자열로 분산되는 @Value
@Value("${my.service.api.endpoint}")       String endpoint;    // A.java
@Value("${my.service.api.endpoint}")       String endpoint;    // B.java
@Value("${my.service.api.endpoint}")       String endpoint;    // C.java
// → 키 변경 시 모든 @Value 문자열을 수동으로 찾아 변경
// → IDE "Find Usages"에서 String 리터럴은 참조 추적 어려움

// @ConfigurationProperties — 타입으로 집중
@ConfigurationProperties(prefix = "my.service.api")
public class ApiProperties {
    private String endpoint;
    // getters
}

// 사용 측
@Autowired ApiProperties api;
api.getEndpoint();  // → IDE 리팩터링으로 안전하게 변경
// 키 변경 시 ApiProperties 한 곳만 수정
```

### 4. 각각을 선택하는 기준

```java
// @Value 적합한 경우:

// ① 단순 단일 값, 해당 Bean에서만 사용
@Value("${spring.application.name}")
private String appName;  // 이 Bean에서만 필요한 단순 값

// ② SpEL 계산식이 필요한 경우
@Value("#{environment.getActiveProfiles().length > 0 ? 'multi' : 'single'}")
private String profileMode;

// ③ Spring @Bean 메서드 파라미터
@Bean
public DataSource dataSource(
        @Value("${db.url}") String url,
        @Value("${db.username}") String username) {
    // ...
}

// ④ 간단한 스크립트/테스트
@SpringBootTest(properties = "my.flag=true")
class SimpleTest {
    @Value("${my.flag}")
    boolean flag;
}
```

```java
// @ConfigurationProperties 적합한 경우:

// ① 관련 프로퍼티 그룹 (3개 이상)
@ConfigurationProperties(prefix = "my.datasource")
public class DataSourceProps {
    private String url, username, password;
    private int poolSize, timeoutMs;
}

// ② 컬렉션/중첩 객체
@ConfigurationProperties(prefix = "my.servers")
public class ServersProps {
    private List<ServerConfig> list = new ArrayList<>();
}

// ③ 여러 Bean에서 공유
// → 한 번 정의, 여러 곳에서 @Autowired

// ④ 검증이 필요한 경우
@ConfigurationProperties(prefix = "app.security")
@Validated
public class SecurityProps {
    @NotBlank private String jwtSecret;
    @Min(60) private int tokenExpiry;
}

// ⑤ 라이브러리/Auto-configuration 설정
// → 사용자가 application.yml로 커스터마이징 가능한 설정
```

### 5. @Value의 타입 변환 한계

```java
// @Value — String → 기본 타입만 자동 변환
@Value("${timeout.ms}")
private int timeoutMs;       // ✅ String → int

@Value("${timeout.seconds}")
private Duration timeout;    // ❌ String → Duration 변환 안 됨 (기본)
// 해결: @Value("${timeout.seconds:30}") + 수동 변환
//   private Duration timeout = Duration.ofSeconds(timeoutSecs);

// @ConfigurationProperties — 풍부한 타입 변환
@ConfigurationProperties(prefix = "my")
public class MyProps {
    private Duration timeout;         // "30s" → Duration.ofSeconds(30)
    private DataSize maxUpload;       // "10MB" → DataSize.ofMegabytes(10)
    private Period retentionPeriod;   // "30d" → Period.ofDays(30)
    private Charset encoding;         // "UTF-8" → Charset
    private InetAddress serverIp;     // "192.168.1.1" → InetAddress
}
```

---

## 💻 실험으로 확인하기

### 실험 1: @Value vs @ConfigurationProperties 바인딩 시점 차이

```java
@Configuration
public class TimingTest implements BeanFactoryPostProcessor {

    @Value("${app.name}")  // BeanFactoryPostProcessor에서는 null
    private String appName;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        System.out.println("BFPP - appName: " + appName);  // null!

        // 직접 Environment 조회 방식 (올바른 방법)
        String name = factory.resolveEmbeddedValue("${app.name:unknown}");
        System.out.println("BFPP - 직접 조회: " + name);  // 정상
    }
}
```

### 실험 2: @Value 기본값 패턴

```java
@Value("${optional.key:}")           // 빈 문자열
@Value("${optional.flag:false}")     // false
@Value("${optional.count:0}")        // 0
@Value("${optional.name:#{null}}")   // null (SpEL로 null 표현)
@Value("${optional.list:}")          // 빈 문자열 (List 변환 안 됨)
```

---

## 📌 핵심 정리

```
@Value 사용 기준
  단순 단일 값 (1~2개)
  SpEL 계산식 필요
  특정 Bean에서만 사용

@ConfigurationProperties 사용 기준
  관련 프로퍼티 그룹 (3개 이상)
  컬렉션/중첩 객체
  여러 Bean 공유
  검증 필요
  Duration, DataSize 등 풍부한 타입

@Value 함정
  static 필드 → 작동 안 함
  BeanFactoryPostProcessor → 작동 안 함
  Relaxed Binding 없음 → 정확한 키 필요
  Duration, DataSize → 자동 변환 없음
```

---

## 🤔 생각해볼 문제

**Q1.** `@Value("${app.name:default}")`에서 실제 값이 `"${app.name:default}"` 문자열 그대로 주입되는 경우는 어떤 상황인가?

**Q2.** `@Configuration` 클래스의 `@Bean` 메서드에서 `@Value`를 메서드 파라미터로 사용하는 것과 필드에 사용하는 것의 차이는?

**Q3.** 동일한 프로퍼티가 `@Value`와 `@ConfigurationProperties` 양쪽에서 사용될 때 두 값이 달라질 수 있는 경우는?

> 💡 **해설**
>
> **Q1.** `PropertySourcesPlaceholderConfigurer`(또는 `PropertyPlaceholderConfigurer`)가 Bean으로 등록되지 않으면 `${...}` 플레이스홀더가 치환되지 않고 문자열 그대로 남는다. `@SpringBootApplication`이 있는 Spring Boot 프로젝트에서는 자동으로 등록되지만, 순수 Spring 컨텍스트에서 `@PropertySource` + `@Value`를 사용할 때 `@EnableConfigurationProperties` 없이 설정하면 발생할 수 있다. XML 설정에서 `<context:property-placeholder>` 누락 시에도 같은 증상이 나타난다.
>
> **Q2.** `@Bean` 메서드 파라미터의 `@Value`는 해당 Bean 생성 시점에 평가되어 파라미터로 전달된다. 이는 메서드 호출마다 최신 Environment 값을 읽는다. 반면 필드에 `@Value`를 붙이면 `@Configuration` 클래스 Bean이 초기화될 때 한 번 주입된다. CGLIB 프록시된 `@Configuration`에서 필드 `@Value`는 프록시 인스턴스에 주입되므로 `@Bean` 메서드가 프록시를 통해 호출될 때 해당 필드에 접근한다. 실용적으로는 두 방식의 차이가 미미하지만, 파라미터 방식이 의도를 더 명확히 드러낸다.
>
> **Q3.** `@Value`는 Relaxed Binding이 없으므로 정확한 키로 조회하고, `@ConfigurationProperties`는 Relaxed Binding으로 정규화된 키로 조회한다. 예를 들어 환경변수 `MY_HOST=prod-server`가 있고 `application.yml`에 `my.host=localhost`가 있을 때, `@Value("${my.host}")` → `"localhost"` (환경변수는 `MY_HOST`라는 다른 키로 인식), `@ConfigurationProperties`의 `String host` 필드 → `"prod-server"` (환경변수 Relaxed Binding으로 매핑). 이 불일치가 같은 애플리케이션에서 두 값이 달라지는 원인이 된다.

---

<div align="center">

**[⬅️ 이전: Type-safe Configuration Properties](./04-type-safe-configuration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Profile 활성화 전략 ➡️](./06-profile-activation.md)**

</div>
