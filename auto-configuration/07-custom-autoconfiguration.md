# Custom Auto-configuration 작성 가이드 — 라이브러리에 자동 설정을 추가하는 전 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 라이브러리에 Auto-configuration을 추가하는 전체 절차는?
- `@AutoConfiguration`, `@ConditionalOnClass`, `@ConditionalOnMissingBean`을 어떻게 조합하는가?
- Auto-configuration 전용 프로퍼티를 `@ConfigurationProperties`로 안전하게 바인딩하는 방법은?
- 사용자가 기본 Bean을 교체하거나 비활성화할 수 있도록 설계하는 패턴은?
- `@AutoConfigureAfter`로 다른 Auto-configuration과 안전하게 연동하는 방법은?
- Auto-configuration을 테스트하는 올바른 방법(`ApplicationContextRunner`)은?

---

## 🔍 왜 이게 존재하는가

### 시나리오: 사내 공통 라이브러리에 Auto-configuration 추가

```java
// 현재 상황:
// - 사내 공통 라이브러리: my-common-lib
// - 모든 서비스에서 사용하는 AuditLogger, MetricsCollector 등 공통 Bean
// - 각 서비스마다 동일한 설정 반복 중

// 각 서비스의 반복 코드
@Configuration
public class CommonBeanConfig {
    @Bean
    public AuditLogger auditLogger(
            @Value("${audit.service-name}") String serviceName) {
        return new AuditLogger(serviceName);
    }

    @Bean
    public MetricsCollector metricsCollector(
            @Value("${metrics.endpoint}") String endpoint) {
        return new MetricsCollector(endpoint);
    }
}
```

```
Auto-configuration으로 해결:
  my-common-lib 내부에 Auto-configuration 포함
  → 의존성 추가만으로 AuditLogger, MetricsCollector 자동 등록
  → 설정이 필요하면 application.properties로 커스터마이징
  → Bean을 교체하고 싶으면 직접 @Bean 등록 (ConditionalOnMissingBean)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Auto-configuration 클래스를 @Component로 등록한다

```java
// ❌ 잘못된 방식
@Component  // @ComponentScan으로 등록 시도
@AutoConfiguration
public class MyLibAutoConfiguration { ... }
```

```
왜 안 되는가:
  @ComponentScan은 사용자 패키지를 스캔 → 라이브러리 패키지는 스캔 안 됨
  (AutoConfigurationExcludeFilter가 @ComponentScan에서 제외)

올바른 방법:
  META-INF/spring/...AutoConfiguration.imports 파일에 등록
  → classpath 어디에 있든 자동으로 로딩됨
```

### Before: @ConfigurationProperties 클래스도 @Component가 필요하다

```java
// ❌ 불필요한 @Component
@Component
@ConfigurationProperties(prefix = "my.lib")
public class MyLibProperties { ... }

// ✅ 올바른 방식 — @AutoConfiguration에서 @EnableConfigurationProperties로 등록
@AutoConfiguration
@EnableConfigurationProperties(MyLibProperties.class)
public class MyLibAutoConfiguration { ... }

// 또는 Spring Boot 2.2+ — @ConfigurationPropertiesScan도 가능하지만
// Auto-configuration 라이브러리에서는 명시적 @EnableConfigurationProperties 권장
```

---

## ✨ 올바른 이해와 사용

### After: Custom Auto-configuration 완성 구조

```
my-common-lib/
  src/main/java/
    com/example/lib/
      autoconfigure/
        MyLibAutoConfiguration.java      ← @AutoConfiguration 클래스
        MyLibProperties.java             ← @ConfigurationProperties
      core/
        AuditLogger.java                 ← 라이브러리 핵심 클래스
        MetricsCollector.java
  src/main/resources/
    META-INF/
      spring/
        org.springframework.boot.autoconfigure.AutoConfiguration.imports  ← 등록 파일
  src/test/java/
    com/example/lib/
      MyLibAutoConfigurationTest.java    ← ApplicationContextRunner 테스트
```

---

## 🔬 내부 동작 원리

### 1. @ConfigurationProperties — 프로퍼티 바인딩

```java
// MyLibProperties.java
@ConfigurationProperties(prefix = "my.lib")
public class MyLibProperties {

    private String serviceName = "default-service";
    private Audit audit = new Audit();
    private Metrics metrics = new Metrics();

    public static class Audit {
        private boolean enabled = true;
        private String logLevel = "INFO";
        private Duration retentionPeriod = Duration.ofDays(30);
        // getters, setters
    }

    public static class Metrics {
        private boolean enabled = true;
        private String endpoint = "http://localhost:9090";
        private Duration pushInterval = Duration.ofSeconds(60);
        // getters, setters
    }
    // getters, setters
}
```

```yaml
# 사용자 application.yml (오버라이드 예시)
my:
  lib:
    service-name: order-service
    audit:
      enabled: true
      log-level: DEBUG
      retention-period: 7d    # Duration 바인딩
    metrics:
      endpoint: http://metrics.internal:9090
      push-interval: 30s
```

### 2. MyLibAutoConfiguration — 완성 코드

```java
// MyLibAutoConfiguration.java
@AutoConfiguration(
    after = DataSourceAutoConfiguration.class  // DataSource 필요 시
)
@ConditionalOnClass(AuditLogger.class)          // 라이브러리 핵심 클래스 존재 확인
@EnableConfigurationProperties(MyLibProperties.class)
public class MyLibAutoConfiguration {

    // ─── AuditLogger 자동 등록 ───
    @Bean
    @ConditionalOnMissingBean(AuditLogger.class)  // 사용자 정의 시 스킵
    @ConditionalOnProperty(
        prefix = "my.lib.audit",
        name = "enabled",
        havingValue = "true",
        matchIfMissing = true   // 설정 없으면 기본 활성화
    )
    public AuditLogger auditLogger(MyLibProperties properties) {
        return new AuditLogger(
            properties.getServiceName(),
            properties.getAudit().getLogLevel()
        );
    }

    // ─── MetricsCollector — DataSource 있을 때만 ───
    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(DataSource.class)
    @ConditionalOnBean(DataSource.class)          // DataSource 존재 확인
    static class MetricsWithDataSourceConfiguration {

        @Bean
        @ConditionalOnMissingBean
        @ConditionalOnProperty(
            prefix = "my.lib.metrics",
            name = "enabled",
            havingValue = "true",
            matchIfMissing = true
        )
        public MetricsCollector metricsCollector(
                MyLibProperties properties,
                DataSource dataSource) {
            return new MetricsCollector(
                properties.getMetrics().getEndpoint(),
                dataSource
            );
        }
    }

    // ─── 조건부 없는 공통 Bean ───
    @Bean
    @ConditionalOnMissingBean
    public MyLibHealthIndicator myLibHealthIndicator(
            ObjectProvider<AuditLogger> auditLogger) {
        return new MyLibHealthIndicator(auditLogger.getIfAvailable());
    }
}
```

### 3. .imports 파일 등록

```
# src/main/resources/META-INF/spring/
# org.springframework.boot.autoconfigure.AutoConfiguration.imports

com.example.lib.autoconfigure.MyLibAutoConfiguration
```

```
파일 형식 규칙:
  한 줄 = 하나의 Auto-configuration 클래스 완전한 이름
  # 주석 가능
  빈 줄 무시
  여러 줄 추가 가능 (모듈별 Auto-configuration 클래스 여러 개)
```

### 4. 사용자 오버라이드 패턴 3가지

```java
// 패턴 1: 동일 타입 Bean 직접 등록 → @ConditionalOnMissingBean으로 자동 스킵
@Configuration
public class UserConfig {
    @Bean
    public AuditLogger customAuditLogger() {
        return new AuditLogger("my-custom-service", "WARN");
    }
    // → MyLibAutoConfiguration.auditLogger() @ConditionalOnMissingBean → 스킵됨
}

// 패턴 2: 프로퍼티로 비활성화
// application.yml:
// my.lib.audit.enabled: false
// → @ConditionalOnProperty 조건 false → auditLogger Bean 등록 안 됨

// 패턴 3: exclude로 전체 Auto-configuration 제거
// @SpringBootApplication(exclude = MyLibAutoConfiguration.class)
// → MyLibAutoConfiguration 전체 스킵
```

### 5. ApplicationContextRunner — Auto-configuration 테스트

```java
// MyLibAutoConfigurationTest.java
class MyLibAutoConfigurationTest {

    // ApplicationContextRunner: 실제 서버 시작 없이 컨텍스트만 생성하는 테스트 도우미
    private final ApplicationContextRunner contextRunner =
        new ApplicationContextRunner()
            .withConfiguration(
                AutoConfigurations.of(MyLibAutoConfiguration.class));

    @Test
    void auditLoggerIsAutoConfigured() {
        contextRunner
            .withClassPath(new FilteredClassPath(AuditLogger.class))  // 클래스패스 조작
            .run(context -> {
                assertThat(context).hasSingleBean(AuditLogger.class);
                AuditLogger logger = context.getBean(AuditLogger.class);
                assertThat(logger.getServiceName()).isEqualTo("default-service");
            });
    }

    @Test
    void auditLoggerUsesCustomProperties() {
        contextRunner
            .withPropertyValues(
                "my.lib.service-name=test-service",
                "my.lib.audit.log-level=DEBUG")
            .run(context -> {
                AuditLogger logger = context.getBean(AuditLogger.class);
                assertThat(logger.getServiceName()).isEqualTo("test-service");
                assertThat(logger.getLogLevel()).isEqualTo("DEBUG");
            });
    }

    @Test
    void auditLoggerBacksOffWhenUserDefinesOwn() {
        contextRunner
            .withUserConfiguration(UserAuditLoggerConfig.class)  // 사용자 Config 추가
            .run(context -> {
                assertThat(context).hasSingleBean(AuditLogger.class);
                // 사용자 정의 Bean이 있으므로 Auto-configuration Bean이 아님
                assertThat(context.getBean(AuditLogger.class))
                    .isInstanceOf(CustomAuditLogger.class);  // 사용자 타입
            });
    }

    @Test
    void auditLoggerIsDisabledWhenPropertyIsFalse() {
        contextRunner
            .withPropertyValues("my.lib.audit.enabled=false")
            .run(context -> {
                assertThat(context).doesNotHaveBean(AuditLogger.class);
            });
    }

    @Test
    void metricsCollectorRequiresDataSource() {
        contextRunner
            // DataSource 없이
            .run(context -> {
                assertThat(context).doesNotHaveBean(MetricsCollector.class);
            });
    }

    @Test
    void metricsCollectorIsCreatedWithDataSource() {
        contextRunner
            .withUserConfiguration(DataSourceConfig.class)  // DataSource 제공
            .run(context -> {
                assertThat(context).hasSingleBean(MetricsCollector.class);
            });
    }

    // 사용자 Config
    @Configuration
    static class UserAuditLoggerConfig {
        @Bean
        public AuditLogger auditLogger() {
            return new CustomAuditLogger();
        }
    }

    @Configuration
    static class DataSourceConfig {
        @Bean
        public DataSource dataSource() {
            return new EmbeddedDatabaseBuilder()
                .setType(EmbeddedDatabaseType.H2).build();
        }
    }
}
```

### 6. AutoConfigurations vs UserConfigurations

```java
// ApplicationContextRunner의 두 가지 설정 주입 방식

// AutoConfigurations.of() — DeferredImportSelector처럼 처리됨
// → 사용자 Config 이후에 처리
// → @ConditionalOnMissingBean이 사용자 Bean 감지 가능
contextRunner.withConfiguration(AutoConfigurations.of(MyLibAutoConfiguration.class));

// UserConfigurations.of() — 일반 @Configuration처럼 먼저 처리됨
contextRunner.withUserConfiguration(SomeConfig.class);
// 또는
contextRunner.withConfiguration(UserConfigurations.of(SomeConfig.class));

// 순서:
// UserConfiguration 먼저 처리 → AutoConfiguration 처리
// → 실제 Spring Boot의 DeferredImportSelector 동작과 동일하게 재현
```

---

## 💻 실험으로 확인하기

### 실험 1: 라이브러리 의존성 추가만으로 Bean 자동 등록 확인

```java
// 의존성 추가 후 (my-common-lib)
// 별도 설정 없이
@SpringBootApplication
public class ServiceApp { }

@RestController
class TestController {
    @Autowired
    AuditLogger auditLogger;  // 자동 주입됨

    @GetMapping("/test")
    String test() {
        auditLogger.log("Test request");
        return "OK";
    }
}
```

### 실험 2: @ConditionalOnProperty 비활성화

```yaml
# application.yml
my:
  lib:
    audit:
      enabled: false  # AuditLogger Bean 미등록
```

```java
// Bean 없으면 @Autowired 실패 → required=false로 처리
@Autowired(required = false)
AuditLogger auditLogger;

if (auditLogger != null) {
    auditLogger.log("...");
}
```

### 실험 3: 메타데이터 힌트 파일 생성

```java
// @ConfigurationProperties 메타데이터 자동 생성
// build.gradle에 추가:
// annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'

// → 컴파일 시 META-INF/spring-configuration-metadata.json 자동 생성
// → IDE에서 application.yml 자동완성 지원
```

```json
// 생성된 META-INF/spring-configuration-metadata.json 예시
{
  "groups": [
    {
      "name": "my.lib",
      "type": "com.example.lib.autoconfigure.MyLibProperties",
      "sourceType": "com.example.lib.autoconfigure.MyLibProperties"
    }
  ],
  "properties": [
    {
      "name": "my.lib.service-name",
      "type": "java.lang.String",
      "defaultValue": "default-service"
    },
    {
      "name": "my.lib.audit.enabled",
      "type": "java.lang.Boolean",
      "defaultValue": true
    }
  ]
}
```

---

## ⚙️ 설정 최적화 팁

```java
// 1. 라이브러리 프로퍼티를 독립 모듈로 분리
//    (autoconfigure 모듈 + core 모듈 + starter 모듈)
//    Spring Boot 공식 스타터 구조:
//    spring-boot-starter-data-jpa
//      → spring-boot-autoconfigure (JpaAutoConfiguration 포함)
//      → spring-data-jpa (실제 기능)

// 2. @AutoConfiguration + spring-boot-autoconfigure-processor
//    → META-INF/spring-autoconfigure-metadata.properties 자동 생성
//    → 1차 필터링 최적화

// 3. ObjectProvider 활용 — 선택적 의존성
@Bean
public MyService myService(
        ObjectProvider<DataSource> dataSource,          // 없어도 됨
        ObjectProvider<MeterRegistry> meterRegistry) {  // 없어도 됨
    return new MyService(
        dataSource.getIfAvailable(),
        meterRegistry.getIfAvailable());
}
```

---

## 🤔 트레이드오프

```
Auto-configuration 설계:
  너무 많은 Bean 자동 등록:
    장점  사용성 최대화
    단점  불필요한 Bean으로 시작 시간 증가, 의존성 충돌 가능

  @ConditionalOnProperty로 기능 분리:
    장점  사용자가 필요한 것만 활성화
    단점  프로퍼티 문서화 필요, 기본값 결정 어려움

matchIfMissing = true vs false:
  true   프로퍼티 없으면 기본 활성화 (사용자가 인식 못해도 동작)
  false  프로퍼티 없으면 비활성화 (명시적으로 활성화해야 동작)
  권장: 필수 기능 → true, 선택적 기능 → false

ApplicationContextRunner 테스트:
  장점  서버 시작 없이 빠른 테스트, classpath 조작 가능
  단점  실제 Spring Boot 환경과 완전히 동일하지 않음
       (배너, Actuator 등 일부 컴포넌트 미포함)
       → 통합 테스트(@SpringBootTest)와 병행 권장
```

---

## 📌 핵심 정리

```
Custom Auto-configuration 구현 4단계

1. @ConfigurationProperties 작성
   @ConfigurationProperties(prefix = "my.lib")
   → 프로퍼티 바인딩, 기본값 설정

2. @AutoConfiguration 클래스 작성
   @AutoConfiguration + @ConditionalOnClass + @EnableConfigurationProperties
   @Bean + @ConditionalOnMissingBean + @ConditionalOnProperty

3. .imports 파일 등록
   META-INF/spring/...AutoConfiguration.imports
   → 클래스명 한 줄 추가

4. ApplicationContextRunner 테스트
   자동 등록 확인, 오버라이드 확인, 비활성화 확인

사용자 오버라이드 방법 3가지
  동일 타입 @Bean 등록 (@ConditionalOnMissingBean으로 자동 스킵)
  @ConditionalOnProperty false (프로퍼티로 비활성화)
  exclude = MyAutoConfiguration.class (전체 제거)

선택적 의존성
  ObjectProvider<T> 사용
  → 없어도 되는 Bean은 getIfAvailable() null 처리
```

---

## 🤔 생각해볼 문제

**Q1.** Auto-configuration 클래스에 `@ConditionalOnClass`가 없으면 어떤 문제가 발생할 수 있는가?

**Q2.** `ApplicationContextRunner.withConfiguration(AutoConfigurations.of(...))`와 `withUserConfiguration(...)` 중 어느 것이 먼저 처리되는가? 그 이유는?

**Q3.** 사내 공통 라이브러리의 Auto-configuration 클래스가 Spring Boot 버전 업그레이드 후 동작하지 않는다. 어떤 원인을 먼저 확인해야 하는가?

> 💡 **해설**
>
> **Q1.** `@ConditionalOnClass`가 없으면 라이브러리 핵심 클래스가 classpath에 없는 환경에서도 Auto-configuration 클래스가 로딩 시도된다. `@Bean` 메서드 반환 타입에 해당 클래스가 직접 등장하면 `NoClassDefFoundError`가 발생해 컨텍스트 시작이 실패한다. 또한 1차 필터링(`AutoConfigurationMetadata`)에서 제외되지 않으므로 불필요한 클래스 로딩 비용이 발생한다. 모든 Auto-configuration 클래스에는 반드시 라이브러리 핵심 클래스에 대한 `@ConditionalOnClass`를 클래스 레벨에 붙여야 한다.
>
> **Q2.** `withUserConfiguration`(UserConfigurations)이 먼저 처리되고, `withConfiguration(AutoConfigurations.of(...))`가 나중에 처리된다. `ApplicationContextRunner`는 실제 Spring Boot의 `DeferredImportSelector` 동작을 재현하도록 설계되어, `AutoConfigurations`는 DeferredImportSelector처럼 사용자 Configuration 처리 완료 후 실행된다. 이 순서 덕분에 테스트에서도 `@ConditionalOnMissingBean`이 사용자 Bean을 정확히 감지할 수 있다.
>
> **Q3.** 가장 먼저 확인할 사항은 Boot 3.x로 업그레이드 시 `spring.factories`에서 Auto-configuration을 등록했는지 여부다. Boot 3.x는 `spring.factories`의 `EnableAutoConfiguration` 키를 더 이상 읽지 않으므로, `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 파일이 없으면 Auto-configuration이 로딩되지 않는다. 두 번째로 확인할 것은 `@AutoConfiguration` 어노테이션 사용 여부다. Boot 3.x에서는 `@Configuration` 대신 `@AutoConfiguration`을 사용해야 `proxyBeanMethods=false` 기본값과 `before/after` 속성을 올바르게 활용할 수 있다. `--debug` 플래그로 조건 평가 리포트를 확인해 어느 조건에서 제외됐는지 파악하는 것도 중요하다.

---

<div align="center">

**[⬅️ 이전: Web MVC Auto-configuration](./06-web-mvc-autoconfiguration.md)** | **[홈으로 🏠](../README.md)** | **[다음: spring-boot-autoconfigure-processor 역할 ➡️](./08-autoconfigure-processor.md)**

</div>
