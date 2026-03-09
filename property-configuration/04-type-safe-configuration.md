# Type-safe Configuration Properties — record + @ConfigurationProperties + @Validated 패턴

---

## 🎯 핵심 질문

- `record`와 `@ConfigurationProperties`를 조합하면 어떻게 불변 설정 객체가 만들어지는가?
- `ValueObjectBinder`가 생성자 기반 바인딩을 처리하는 방법은?
- `@DefaultValue`는 어떻게 처리되는가?
- `@Validated`의 중첩 검증(`@Valid`)은 어떻게 동작하는가?
- 불변 설정 객체의 테스트 전략은?

---

## 🔍 왜 이게 존재하는가

```java
// 문제 1: setter 기반 — 런타임에 변경 가능
@ConfigurationProperties(prefix = "my")
public class MyProps {
    private String host;
    private int port;
    // setter 있음 → 어디서든 setHost() 호출 가능 → 의도치 않은 변경 위험
}

// 문제 2: null 안전성 없음 — 기본값 분산
@ConfigurationProperties(prefix = "my")
public class MyProps {
    private String host = "localhost";  // 기본값 코드에 분산
    private int port = 8080;
    // 기본값이 어디 있는지 한눈에 파악 어려움
}
```

```java
// 해결: record + @ConfigurationProperties (Boot 2.6+)
@ConfigurationProperties(prefix = "my")
public record MyProperties(
    @DefaultValue("localhost") String host,
    @DefaultValue("8080") int port,
    @NotNull @Valid Database database
) {
    public record Database(
        @NotBlank String url,
        @DefaultValue("10") int poolSize
    ) {}
}
// → 불변 (모든 필드 final)
// → 기본값 선언적으로 명시
// → 컴파일 타임 타입 안전성
// → 검증 통합
```

---

## 🔬 내부 동작 원리

### 1. ValueObjectBinder — 생성자 기반 바인딩

```java
// ValueObjectBinder.java
// record 또는 @ConstructorBinding 클래스를 처리

class ValueObjectBinder implements DataObjectBinder {

    @Override
    public <T> T bind(ConfigurationPropertyName name, Bindable<T> target,
                       Context context, DataObjectPropertyBinder propertyBinder) {

        // 대상 타입의 생성자 탐색
        ValueObjectConstructor constructor = findConstructor(target);
        if (constructor == null) return null;

        // 생성자 파라미터별 바인딩
        List<ConstructorParameter> parameters = constructor.getParameters();
        Object[] args = new Object[parameters.size()];

        for (int i = 0; i < parameters.size(); i++) {
            ConstructorParameter parameter = parameters.get(i);
            String parameterName = parameter.getName();  // "host", "port" 등

            // 프로퍼티 바인딩 시도
            Object value = propertyBinder.bindProperty(
                parameterName,
                Bindable.of(parameter.getType())
                    .withAnnotations(parameter.getAnnotations()));

            if (value == null) {
                // @DefaultValue 처리
                value = getDefaultValue(context, parameter);
            }
            args[i] = value;
        }

        // 생성자 호출
        return constructor.create(args);
    }

    private Object getDefaultValue(Context context, ConstructorParameter parameter) {
        DefaultValue annotation = parameter.getAnnotation(DefaultValue.class);
        if (annotation == null) return null;

        // @DefaultValue의 value()를 대상 타입으로 변환
        String[] defaults = annotation.value();
        if (defaults.length == 0) return null;  // 빈 @DefaultValue → null

        // String → 대상 타입 변환
        return context.getConverter().convert(
            (defaults.length == 1) ? defaults[0] : defaults,
            parameter.getType());
    }
}
```

### 2. 생성자 탐색 — record vs @ConstructorBinding

```java
// findConstructor() 로직

// Case 1: record → canonical constructor 자동 선택
public record MyProperties(String host, int port) { }
// → MyProperties(String, int) 생성자 자동 선택 (단일 생성자)

// Case 2: 일반 클래스 + 단일 생성자 (Boot 3.0+)
public class MyProperties {
    private final String host;
    private final int port;

    // 단일 생성자 → 자동으로 생성자 바인딩 사용 (Boot 3.0+)
    public MyProperties(String host, int port) {
        this.host = host;
        this.port = port;
    }
}

// Case 3: 여러 생성자 → @ConstructorBinding으로 명시 (Boot 2.x)
public class MyProperties {
    @ConstructorBinding  // 이 생성자를 바인딩용으로 사용
    public MyProperties(String host, @DefaultValue("8080") int port) { ... }

    public MyProperties() { ... }  // 기본 생성자도 있음
}

// Boot 3.0+: @ConstructorBinding은 여러 생성자가 있을 때만 필요
```

### 3. @DefaultValue 고급 사용

```java
@ConfigurationProperties(prefix = "my")
public record MyProperties(
    // 단순 기본값
    @DefaultValue("localhost") String host,
    @DefaultValue("8080") int port,

    // List 기본값
    @DefaultValue({"admin", "user"}) List<String> defaultRoles,

    // Duration 기본값
    @DefaultValue("30s") Duration timeout,

    // 빈 @DefaultValue → 빈 컬렉션 기본값
    @DefaultValue List<String> tags,   // → Collections.emptyList()
    @DefaultValue Map<String, String> headers,  // → Collections.emptyMap()

    // 중첩 객체 — 모든 하위 프로퍼티가 기본값이면 기본 객체 생성
    @DefaultValue Database database
) {
    public record Database(
        @DefaultValue("jdbc:h2:mem:test") String url,
        @DefaultValue("10") int poolSize
    ) {}
}
```

### 4. @Validated 중첩 검증 패턴

```java
@ConfigurationProperties(prefix = "app")
@Validated
public record AppProperties(
    @NotBlank String name,

    @NotNull
    @Valid   // ← 중첩 객체 검증 활성화
    SecurityProperties security,

    @NotNull
    @Valid
    DatabaseProperties database
) {
    public record SecurityProperties(
        @NotBlank String jwtSecret,
        @Min(60) @Max(86400) long tokenExpirySeconds,
        @NotEmpty List<@NotBlank String> allowedOrigins
    ) {}

    public record DatabaseProperties(
        @NotBlank
        @Pattern(regexp = "jdbc:.+") String url,

        @Min(1) @Max(50) int poolSize,

        @NotNull Duration connectionTimeout
    ) {}
}
```

```yaml
# application.yml
app:
  name: my-service
  security:
    jwt-secret: ${JWT_SECRET}          # 환경변수 참조
    token-expiry-seconds: 3600
    allowed-origins:
      - https://frontend.example.com
  database:
    url: jdbc:postgresql://localhost:5432/mydb
    pool-size: 20
    connection-timeout: 3s
```

### 5. 불변 설정 객체의 테스트

```java
// @ConfigurationPropertiesTest — 슬라이스 테스트
@ConfigurationPropertiesTest  // 전체 컨텍스트 없이 Properties만 로딩
@EnableConfigurationProperties(AppProperties.class)
class AppPropertiesTest {

    @Autowired AppProperties appProperties;

    @Test
    void defaultValuesAreCorrect() {
        // @DefaultValue 검증
        assertThat(appProperties.host()).isEqualTo("localhost");
        assertThat(appProperties.port()).isEqualTo(8080);
    }

    @Test
    void propertiesAreImmutable() {
        // record는 setter 없음 → 컴파일 타임 보장
        // 런타임 불변성 테스트
        assertThatCode(() -> {
            // AppProperties는 record → setHost() 메서드 없음 → 컴파일 오류
        }).doesNotThrowAnyException();
    }
}

// 특정 프로퍼티 값으로 테스트
@SpringBootTest(properties = {
    "app.name=test-service",
    "app.security.jwt-secret=test-secret",
    "app.security.token-expiry-seconds=3600",
    "app.security.allowed-origins=https://test.example.com",
    "app.database.url=jdbc:h2:mem:test",
    "app.database.pool-size=5",
    "app.database.connection-timeout=1s"
})
class AppPropertiesIntegrationTest { ... }
```

### 6. 설정 메타데이터 생성

```java
// spring-boot-configuration-processor 추가 시
// record의 각 컴포넌트가 @ConfigurationProperties 메타데이터로 추출됨

// build.gradle:
// annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'

// 생성되는 META-INF/spring-configuration-metadata.json:
// {
//   "properties": [
//     {
//       "name": "app.name",
//       "type": "java.lang.String",
//       "description": "Application name."
//     },
//     {
//       "name": "app.security.jwt-secret",
//       "type": "java.lang.String"
//     },
//     {
//       "name": "app.database.pool-size",
//       "type": "java.lang.Integer",
//       "defaultValue": 10
//     }
//   ]
// }
// → IDE application.yml 편집 시 자동완성 제공
```

---

## 💻 실험으로 확인하기

### 실험 1: record 바인딩 확인

```java
@ConfigurationProperties(prefix = "server")
public record ServerConfig(
    @DefaultValue("localhost") String host,
    @DefaultValue("8080") int port,
    @DefaultValue("false") boolean ssl
) {}

// 설정 없이 기본값으로 동작:
// serverConfig.host() = "localhost"
// serverConfig.port() = 8080
// serverConfig.ssl() = false

// application.yml에서 일부만 오버라이드:
// server.host: production.example.com
// → serverConfig.host() = "production.example.com"
// → serverConfig.port() = 8080  (기본값 유지)
```

### 실험 2: @Validated 실패 메시지

```yaml
app:
  name: ""              # @NotBlank 위반
  security:
    jwt-secret: ""      # @NotBlank 위반
    token-expiry-seconds: 0  # @Min(60) 위반
```

```
시작 실패 오류:
  Property: app.name          Value: "" Reason: must not be blank
  Property: app.security.jwt-secret   Value: "" Reason: must not be blank
  Property: app.security.token-expiry-seconds  Value: 0  Reason: must be >= 60
```

---

## ⚙️ 설정 최적화 팁

```java
// Bean으로 사용 시 @EnableConfigurationProperties 또는 Auto-configuration에서 등록
@AutoConfiguration
@EnableConfigurationProperties(AppProperties.class)
public class AppAutoConfiguration {

    @Bean
    public AppService appService(AppProperties props) {
        // 불변 props → 안전하게 여러 Bean에서 참조 가능
        return new AppService(props.name(), props.database().url());
    }
}
```

---

## 📌 핵심 정리

```
생성자 기반 바인딩 (ValueObjectBinder)
  record → canonical constructor 자동 선택
  단일 생성자 클래스 → 자동 (Boot 3.0+)
  여러 생성자 → @ConstructorBinding 명시 필요

@DefaultValue
  단순 타입: @DefaultValue("30s") Duration timeout
  List:      @DefaultValue({"a", "b"}) List<String> roles
  빈 컬렉션: @DefaultValue List<String> tags → emptyList()
  중첩 객체: @DefaultValue → 모든 하위에 기본값 있을 때 기본 객체 생성

@Validated + record
  @Valid로 중첩 record도 검증
  실패 시 시작 실패 + 정확한 프로퍼티 경로 오류

불변성 보장
  record → setter 없음 → 런타임 변경 불가
  → 여러 Bean에서 안전하게 공유 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `@DefaultValue`가 있는 파라미터와 없는 파라미터 중 설정 값이 없을 때 각각 어떻게 처리되는가?

**Q2.** `@ConfigurationProperties` record에서 `@Validated`를 클래스 레벨에 붙이는 것과 각 파라미터에 직접 Bean Validation 어노테이션을 붙이는 것의 차이는 무엇인가?

**Q3.** 동일한 설정을 여러 서비스에서 공유할 때 record 기반 설정 클래스가 일반 클래스보다 유리한 이유는 무엇인가?

> 💡 **해설**
>
> **Q1.** `@DefaultValue`가 있으면 설정 값이 없을 때 어노테이션의 `value()`를 대상 타입으로 변환해 사용한다. `@DefaultValue`가 없고 설정 값도 없으면 원시 타입(`int`, `boolean` 등)은 `0`, `false` 등 기본값이 사용되고, 참조 타입은 `null`이 전달된다. `null`이 전달되면 record 생성자 파라미터가 `null`이 되므로 이후 `@NotNull` 검증이나 NPE에 취약하다. 따라서 필수 설정에는 `@NotNull`을, 선택적 설정에는 `@DefaultValue`를 붙이는 것이 권장된다.
>
> **Q2.** `@Validated`를 클래스 레벨에 붙여야만 `ConfigurationPropertiesBindingPostProcessor`가 바인딩 완료 후 `Validator.validate()`를 호출한다. 클래스 레벨 `@Validated`가 없으면 파라미터에 `@NotBlank` 등을 붙여도 검증이 실행되지 않는다. `@Valid`는 중첩 객체(record 내 record)에 붙여서 재귀 검증을 활성화한다. 즉 `@Validated`(클래스 레벨) = 검증 실행 스위치, `@Valid`(필드 레벨) = 중첩 재귀 검증 스위치다.
>
> **Q3.** record는 모든 필드가 `final`이므로 어떤 Bean이든 설정 객체를 주입받아도 값을 변경할 수 없다. 일반 setter 기반 클래스는 한 컴포넌트에서 setter를 호출하면 다른 컴포넌트가 참조하는 값도 변경되는 문제가 생긴다. 또한 record는 `equals()`, `hashCode()`, `toString()`이 자동 생성되어 테스트와 로깅이 편리하고, 구조적 동등성 비교가 가능하다.

---

<div align="center">

**[⬅️ 이전: Relaxed Binding](./03-relaxed-binding.md)** | **[홈으로 🏠](../README.md)** | **[다음: @Value vs @ConfigurationProperties ➡️](./05-value-vs-configuration-properties.md)**

</div>
