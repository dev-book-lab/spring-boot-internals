# @EnableAutoConfiguration 동작 원리 — AutoConfigurationImportSelector 완전 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@EnableAutoConfiguration`이 붙으면 스프링이 어디서 Auto-configuration 후보 목록을 가져오는가?
- `AutoConfigurationImportSelector`는 어떤 시점에 어떻게 호출되는가?
- `ImportCandidates`는 무엇이고 어떻게 로딩되는가?
- 수백 개의 Auto-configuration 후보 중 실제로 적용되는 것이 결정되는 과정은?
- `spring-boot-autoconfigure-processor`가 시작 성능에 어떻게 기여하는가?
- `@Conditional` 조건 평가가 지연되는(deferred) 이유는 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: 클래스패스에 무엇이 있는지에 따라 다른 Bean을 등록해야 한다

```java
// 만약 Auto-configuration이 없다면
@Configuration
public class ManualConfig {

    // HikariCP가 클래스패스에 있으면 DataSource 등록
    // → 항상 존재한다고 가정? 없으면 어떻게?
    @Bean
    @ConditionalOnClass(HikariDataSource.class)
    @ConditionalOnMissingBean(DataSource.class)
    public DataSource dataSource(DataSourceProperties props) {
        return new HikariDataSource(...);
    }

    // Jackson이 있으면 ObjectMapper 등록
    @Bean
    @ConditionalOnClass(ObjectMapper.class)
    @ConditionalOnMissingBean
    public ObjectMapper objectMapper() { ... }

    // ... 수백 개의 @Bean을 직접 관리
}
```

```
문제:
  - 사용하는 라이브러리마다 조건부 Bean 설정 직접 작성
  - 라이브러리 버전 업그레이드 시 설정도 수동 변경
  - 팀마다 다른 설정 → 일관성 없음

해결:
  @EnableAutoConfiguration
  → 각 라이브러리 제공자가 자신의 Auto-configuration을 패키지에 포함
  → Spring Boot가 클래스패스 분석으로 자동 적용/제외 결정
  → 사용자는 별도 설정 없이 의존성 추가만으로 기능 사용
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Auto-configuration은 항상 전체가 처리된다

```
❌ 잘못된 이해:
  "수백 개의 Auto-configuration 클래스가 모두 로딩되고 평가된다"

✅ 실제:
  후보 목록 수집 → 1차 필터링(AutoConfigurationMetadata) → 조건 평가 순서
  
  1차 필터링에서 대부분 탈락:
    spring-boot-autoconfigure-processor가 컴파일 타임에
    @ConditionalOnClass 조건을 미리 메타데이터로 추출
    → 클래스 로딩 없이 클래스명 존재 여부만 확인해 즉시 제외
    → 실제로 Class 로딩, BeanDefinition 파싱이 일어나는 것은
       1차 필터링을 통과한 소수

성능 영향:
  전체 수백 개 클래스를 다 로딩하면 시작 시간이 크게 늘어남
  → autoconfigure-processor 덕분에 실제 클래스 로딩은 최소화
```

### Before: @EnableAutoConfiguration은 한 번만 처리된다

```
❌ 잘못된 이해:
  "@EnableAutoConfiguration을 붙이면 그 시점에 처리된다"

✅ 실제:
  @EnableAutoConfiguration → @Import(AutoConfigurationImportSelector.class)
  → DeferredImportSelector 구현체
  → 일반 @Import와 다르게 모든 @Configuration 클래스 처리가 끝난 후 실행
  → 사용자 Bean 정의가 모두 등록된 후에 Auto-configuration 후보가 처리됨
  → @ConditionalOnMissingBean이 정확하게 동작하는 이유
```

---

## ✨ 올바른 이해와 사용

### After: 처리 흐름의 4단계

```
@EnableAutoConfiguration 처리 흐름:

단계 1: 후보 수집
  AutoConfigurationImportSelector.selectImports()
  → ImportCandidates.load() → META-INF/spring/...imports 파일 읽기
  → 수백 개의 클래스명 목록

단계 2: 1차 필터링 (클래스 로딩 없이)
  AutoConfigurationMetadata 로딩
  → META-INF/spring-autoconfigure-metadata.properties 파일
  → @ConditionalOnClass 조건을 클래스명 문자열로 미리 체크
  → 클래스패스에 없는 것 제거 (대부분 여기서 탈락)

단계 3: exclude 적용
  사용자가 명시한 exclude/excludeName/spring.autoconfigure.exclude 제거

단계 4: 순서 결정 후 등록
  @AutoConfigureOrder, @AutoConfigureAfter, @AutoConfigureBefore
  → 위상 정렬로 등록 순서 결정
  → ConfigurationClassParser가 각 Auto-configuration 처리
```

---

## 🔬 내부 동작 원리

### 1. AutoConfigurationImportSelector — DeferredImportSelector

```java
// AutoConfigurationImportSelector.java
public class AutoConfigurationImportSelector
        implements DeferredImportSelector, BeanClassLoaderAware, ... {

    // DeferredImportSelector를 구현 → 일반 @Import보다 나중에 실행
    // → 모든 @Configuration 처리 완료 후 실행

    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        // getAutoConfigurationEntry()로 최종 목록 반환
        AutoConfigurationEntry entry = getAutoConfigurationEntry(annotationMetadata);
        return StringUtils.toStringArray(entry.getConfigurations());
    }

    protected AutoConfigurationEntry getAutoConfigurationEntry(
            AnnotationMetadata annotationMetadata) {

        if (!isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        }

        AnnotationAttributes attributes = getAttributes(annotationMetadata);

        // ① 후보 목록 수집
        List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);

        // ② 중복 제거
        configurations = removeDuplicates(configurations);

        // ③ exclude 적용
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);

        // ④ AutoConfigurationImportFilter로 1차 필터링
        configurations = getConfigurationClassFilter().filter(configurations);

        // ⑤ 이벤트 발행 (조건 평가 리포트용)
        fireAutoConfigurationImportEvents(configurations, exclusions);

        return new AutoConfigurationEntry(configurations, exclusions);
    }
}
```

### 2. getCandidateConfigurations() — 후보 목록 로딩

```java
// getCandidateConfigurations() — Boot 3.x
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
                                                    AnnotationAttributes attributes) {
    // ImportCandidates.load()로 후보 로딩
    List<String> configurations = ImportCandidates.load(
        AutoConfiguration.class, getBeanClassLoader()
    ).getCandidates();

    Assert.notEmpty(configurations,
        "No auto configuration classes found in "
        + "META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports");

    return configurations;
}

// ImportCandidates.load()
public static ImportCandidates load(Class<?> annotation, ClassLoader classLoader) {
    // META-INF/spring/{annotation-classname}.imports 파일 로딩
    // 예: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    String location = String.format(LOCATION, annotation.getName());
    // classLoader.getResources(location)으로 classpath 전체에서 파일 탐색
    // → 여러 JAR에 분산된 imports 파일을 모두 합침
    Enumeration<URL> urls = classLoader.getResources(location);
    // ...
}
```

```
Boot 3.x imports 파일 위치:
  spring-boot-autoconfigure-3.x.jar 내부:
  META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports

파일 내용 (일부):
  org.springframework.boot.autoconfigure.aop.AopAutoConfiguration
  org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration
  org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration
  org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
  org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
  org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
  ... (약 150개)

Boot 2.x spring.factories 방식:
  META-INF/spring.factories 파일
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
    org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
    ...
```

### 3. AutoConfigurationImportFilter — 1차 필터링 (성능 핵심)

```java
// AutoConfigurationImportFilter 체인 — 클래스 로딩 없이 필터링
public interface AutoConfigurationImportFilter {
    boolean[] match(String[] autoConfigurationClasses,
                    AutoConfigurationMetadata autoConfigurationMetadata);
}

// 구현체들:
// OnClassCondition    → @ConditionalOnClass / @ConditionalOnMissingClass
// OnBeanCondition     → @ConditionalOnBean / @ConditionalOnMissingBean (1차)
// OnWebApplicationCondition → @ConditionalOnWebApplication

// AutoConfigurationMetadata — 컴파일 타임 메타데이터
// spring-autoconfigure-metadata.properties (자동 생성)
// 내용 예시:
// org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.ConditionalOnClass=\
//   javax.sql.DataSource,org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType

// OnClassCondition.getOutcomes() — 클래스 로딩 없이 체크
private ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
                                        AutoConfigurationMetadata metadata) {
    ConditionOutcome[] outcomes = new ConditionOutcome[autoConfigurationClasses.length];

    for (int i = 0; i < outcomes.length; i++) {
        String className = autoConfigurationClasses[i];
        // 메타데이터에서 ConditionalOnClass 값 조회
        String candidates = metadata.get(className, "ConditionalOnClass");

        if (candidates != null) {
            // ClassNameFilter.MISSING 으로 클래스 존재 여부만 확인
            // → Class.forName() 하지 않음, ClassLoader.getResource() 사용
            outcomes[i] = getOutcome(candidates, ClassNameFilter.MISSING, this.beanClassLoader);
        }
    }
    return outcomes;
}
```

```
1차 필터링 효과 (예시):

전체 Auto-configuration 후보: 약 150개

클래스패스에 spring-data-jpa 없는 경우:
  JpaRepositoriesAutoConfiguration    → ConditionalOnClass: JpaRepository → 제외
  HibernateJpaAutoConfiguration       → ConditionalOnClass: LocalContainerEntityManagerFactoryBean → 제외
  JndiDataSourceAutoConfiguration     → ConditionalOnClass: JndiLocatorDelegate → 제외

클래스패스에 spring-security 없는 경우:
  SecurityAutoConfiguration           → ConditionalOnClass: DefaultAuthenticationEventPublisher → 제외
  SecurityFilterAutoConfiguration     → 제외
  ...

결과: 실제 @Conditional 평가(클래스 로딩 포함)는 필터링 후 남은 소수에만 적용
```

### 4. DeferredImportSelector가 중요한 이유

```java
// 일반 ImportSelector와 DeferredImportSelector 처리 시점 차이

// ConfigurationClassParser.parse() 흐름:
// 1단계: 모든 @Configuration 클래스 파싱 (사용자 정의 @Bean 등록)
// 2단계: DeferredImportSelector 실행 (Auto-configuration 처리)

// 왜 중요한가:
// @ConditionalOnMissingBean(DataSource.class) 시나리오

// 1단계 완료 후:
//   UserConfig.class → @Bean DataSource myDataSource() 등록됨

// 2단계에서 Auto-configuration 처리:
//   DataSourceAutoConfiguration → @ConditionalOnMissingBean(DataSource.class)
//   → 이미 DataSource Bean이 등록됨 → 조건 false → 스킵

// 만약 DeferredImportSelector가 아니었다면:
//   Auto-configuration과 사용자 Config 처리 순서가 섞임
//   → @ConditionalOnMissingBean 평가 시 사용자 DataSource가 아직 없을 수 있음
//   → 자동 DataSource도 등록 → 중복 Bean 충돌
```

```
DeferredImportSelector 처리 흐름:

1단계 파싱:
  @SpringBootApplication → @ComponentScan → UserService, UserRepository 등
  @SpringBootApplication → @SpringBootConfiguration → AppConfig → DataSource Bean 등록

2단계 (Deferred):
  AutoConfigurationImportSelector.selectImports()
  → 후보 수집 → 필터링 → 순서 정렬
  → 각 Auto-configuration 처리 시 @ConditionalOnMissingBean 평가
  → 이미 등록된 DataSource 감지 → DataSourceAutoConfiguration 스킵

결론:
  DeferredImportSelector가 사용자 설정이 Auto-configuration보다 항상 우선하도록 보장
```

### 5. Group으로 묶인 처리 — AutoConfigurationGroup

```java
// DeferredImportSelector.Group — 여러 selector를 묶어서 처리
public interface Group {
    void process(AnnotationMetadata metadata, DeferredImportSelector selector);
    Iterable<Entry> selectImports();
}

// AutoConfigurationImportSelector.AutoConfigurationGroup
private static class AutoConfigurationGroup implements DeferredImportSelector.Group {

    @Override
    public void process(AnnotationMetadata annotationMetadata,
                         DeferredImportSelector deferredImportSelector) {
        // getAutoConfigurationEntry() 호출 → 필터링된 목록 수집
        AutoConfigurationEntry entry =
            ((AutoConfigurationImportSelector) deferredImportSelector)
                .getAutoConfigurationEntry(annotationMetadata);

        this.autoConfigurationEntries.add(entry);
        // 클래스명 → AnnotationMetadata 매핑 저장
        for (String importClass : entry.getConfigurations()) {
            this.entries.putIfAbsent(importClass, annotationMetadata);
        }
    }

    @Override
    public Iterable<Entry> selectImports() {
        // 모든 후보를 모아서 순서 정렬 후 반환
        List<String> allImports = this.autoConfigurationEntries.stream()
            .flatMap(entry -> entry.getConfigurations().stream())
            .collect(Collectors.toList());

        // @AutoConfigureOrder, @AutoConfigureAfter, @AutoConfigureBefore 정렬
        return sortAutoConfigurations(processExclusions(allImports), ...)
            .stream()
            .map(importClass -> new Entry(this.entries.get(importClass), importClass))
            .collect(Collectors.toList());
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: --debug로 필터링 결과 확인

```bash
java -jar app.jar --debug 2>&1 | grep -A 3 "DataSource"
```

```
출력 예시:
DataSourceAutoConfiguration matched:
   - @ConditionalOnClass found required classes 'javax.sql.DataSource',
     'org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType'
   - @ConditionalOnMissingBean (types: javax.sql.DataSource) did not find any beans
```

### 실험 2: imports 파일 직접 확인

```bash
# spring-boot-autoconfigure jar 내부 확인
jar -tf ~/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/3.2.0/\
spring-boot-autoconfigure-3.2.0.jar | grep ".imports"

# 내용 확인
unzip -p ~/.m2/.../spring-boot-autoconfigure-3.2.0.jar \
  "META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports" | head -20
```

### 실험 3: 커스텀 Auto-configuration 등록

```java
// 1. 커스텀 Auto-configuration 클래스 작성
@AutoConfiguration
@ConditionalOnClass(MyLibrary.class)
@ConditionalOnMissingBean(MyLibraryService.class)
public class MyLibraryAutoConfiguration {
    @Bean
    public MyLibraryService myLibraryService() {
        return new MyLibraryService();
    }
}

// 2. META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 파일 생성
// com.example.library.MyLibraryAutoConfiguration

// 3. 결과: classpath에 MyLibrary.class가 있고
//    사용자가 MyLibraryService Bean을 등록하지 않았으면 자동 등록됨
```

### 실험 4: AutoConfigurationMetadata 파일 직접 보기

```bash
unzip -p ~/.m2/.../spring-boot-autoconfigure-3.2.0.jar \
  "META-INF/spring-autoconfigure-metadata.properties" | grep "DataSource" | head -10
```

```
출력 예시:
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.ConditionalOnClass=\
  javax.sql.DataSource,org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType

org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration.AutoConfigureAfter=\
  org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration

→ 이 데이터를 클래스 로딩 없이 읽어 1차 필터링에 사용
```

---

## ⚙️ 설정 최적화 팁

```yaml
# 특정 Auto-configuration 제외 (코드 변경 없이)
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
      - org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration

# Auto-configuration 조건 평가 디버그
logging:
  level:
    org.springframework.boot.autoconfigure: DEBUG
```

```java
// ConditionEvaluationReport로 프로그래밍 방식 확인
@Autowired
private ConditionEvaluationReport report;

public void printReport() {
    report.getConditionAndOutcomesBySource().forEach((source, outcomes) -> {
        System.out.println(source + ": " + outcomes.isFullMatch());
    });
}
```

---

## 🤔 트레이드오프

```
Auto-configuration 자동화:
  장점  의존성 추가만으로 동작, 일관된 설정, 버전 관리 편의
  단점  "마법" 느낌 → 디버깅 어려움, 예상치 못한 Bean 등록 가능

DeferredImportSelector 지연 처리:
  장점  사용자 Bean이 항상 Auto-configuration보다 우선
  단점  처리 순서가 명확하지 않으면 @ConditionalOnBean 의존 시 함정 가능
        Auto-configuration끼리의 순서는 @AutoConfigureAfter로 명시 필요

1차 필터링 (autoconfigure-processor):
  장점  클래스 로딩 없이 대부분을 제거 → 시작 성능 향상
  단점  컴파일 타임 메타데이터가 런타임 상태와 다를 수 있음
       (드문 케이스, 실제로는 거의 문제 없음)
```

---

## 📌 핵심 정리

```
@EnableAutoConfiguration = @Import(AutoConfigurationImportSelector)

AutoConfigurationImportSelector
  DeferredImportSelector 구현 → 사용자 @Configuration 모두 처리 후 실행
  → @ConditionalOnMissingBean의 정확한 동작 보장

후보 로딩 (Boot 3.x)
  ImportCandidates.load(AutoConfiguration.class)
  META-INF/spring/...AutoConfiguration.imports 파일 읽기
  JAR마다 분산 → classpath 전체에서 합산

1차 필터링 (성능 핵심)
  AutoConfigurationMetadata (spring-autoconfigure-metadata.properties)
  → 컴파일 타임에 생성된 @ConditionalOnClass 조건
  → 클래스 로딩 없이 문자열 비교로 즉시 제외
  → 대부분의 후보가 여기서 탈락

남은 후보만 @Conditional 실제 평가
  클래스 로딩 → BeanDefinition 파싱 → 조건 평가

최종 등록 순서
  @AutoConfigureOrder / @AutoConfigureAfter / @AutoConfigureBefore
  → 위상 정렬
```

---

## 🤔 생각해볼 문제

**Q1.** `@EnableAutoConfiguration`과 `@ComponentScan`을 모두 사용할 때, 사용자가 작성한 `@Configuration` 클래스와 Auto-configuration 클래스 중 어느 것이 먼저 처리되는가? 그 이유는?

**Q2.** 커스텀 Auto-configuration 클래스를 `@Component`로 등록하면 어떤 문제가 발생하는가?

**Q3.** Boot 2.x에서 Boot 3.x로 마이그레이션 시 `spring.factories`에 Auto-configuration을 등록한 커스텀 라이브러리는 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** 사용자가 작성한 `@Configuration` 클래스가 먼저 처리된다. `AutoConfigurationImportSelector`는 `DeferredImportSelector`를 구현하므로, `ConfigurationClassParser`가 모든 일반 `@Configuration` 클래스를 파싱한 후 마지막에 `DeferredImportSelector`들을 처리한다. 이 순서가 `@ConditionalOnMissingBean`이 정확하게 동작하는 핵심 이유다. 사용자 `DataSource` Bean이 먼저 등록되고 나서 Auto-configuration이 처리될 때 그 Bean의 존재를 감지할 수 있다.
>
> **Q2.** `@Component`로 등록하면 `@ComponentScan`이 해당 클래스를 일반 Bean으로 등록한다. 동시에 `@EnableAutoConfiguration`도 해당 클래스를 Auto-configuration 후보로 가져오면 동일 클래스가 두 번 등록 시도될 수 있다. `@SpringBootApplication`의 `@ComponentScan`에는 `AutoConfigurationExcludeFilter`가 포함되어 있어 Auto-configuration 후보 클래스를 ComponentScan 대상에서 자동으로 제외하지만, 직접 `@ComponentScan`을 쓰거나 다른 방식으로 등록하면 이 필터가 없을 수 있다. 또한 Auto-configuration의 `@AutoConfigureAfter` 순서 보장도 동작하지 않게 된다.
>
> **Q3.** Boot 3.x는 `spring.factories`에서 `EnableAutoConfiguration` 키를 읽는 코드 경로를 제거했다. 따라서 Boot 2.x 스타일의 커스텀 라이브러리는 Boot 3.x 애플리케이션에서 Auto-configuration이 적용되지 않는다. 마이그레이션하려면 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 파일을 새로 생성하고 Auto-configuration 클래스명을 등록해야 한다. Spring Boot 2.7.x는 두 방식을 모두 지원하므로 이 버전을 중간 단계로 활용하면 점진적 마이그레이션이 가능하다.

---

<div align="center">

**[⬅️ 이전: @SpringBootApplication 3개 어노테이션 분해](./02-springbootapplication-decompose.md)** | **[홈으로 🏠](../README.md)** | **[다음: spring.factories vs @AutoConfiguration ➡️](./04-spring-factories-vs-autoconfiguration.md)**

</div>
