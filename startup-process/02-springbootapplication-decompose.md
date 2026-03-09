# @SpringBootApplication 3개 어노테이션 분해 — 메타 어노테이션의 실체

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@SpringBootApplication`을 구성하는 3개 어노테이션은 각각 무엇을 하는가?
- `@SpringBootConfiguration`이 `@Configuration`과 다른 점은 무엇인가?
- `@EnableAutoConfiguration`을 제거하면 무슨 일이 벌어지는가?
- `@ComponentScan`의 기본 스캔 범위는 어떻게 결정되는가?
- `@SpringBootApplication` 하나를 3개로 풀어쓸 때 완전히 동일한 결과를 내려면 어떻게 해야 하는가?
- `exclude`와 `excludeName`으로 특정 Auto-configuration을 끄는 원리는?

---

## 🔍 왜 이게 존재하는가

### 문제: 매번 3개 어노테이션을 모두 붙여야 한다면?

```java
// @SpringBootApplication 없이 동일한 효과를 내려면
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(basePackages = "com.example")
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

```
문제:
  - 프로젝트마다 3개를 항상 같이 붙여야 함
  - basePackages를 잊으면 스캔이 의도대로 안 될 수 있음
  - 각 어노테이션의 속성을 여러 곳에서 관리

해결:
  @SpringBootApplication = 세 어노테이션의 합성 메타 어노테이션
  → 단일 진입점, 기본값으로 대부분의 경우 커버
  → exclude 등 자주 쓰는 속성을 직접 노출
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @SpringBootApplication은 반드시 최상위 패키지에 있어야 한다

```
❌ 잘못된 이해:
  "어디에 놓든 상관없다"
  또는 반대로
  "최상위 패키지(com 또는 com.example)에 놓아야 한다"

✅ 실제 규칙:
  @ComponentScan은 @SpringBootApplication이 위치한 클래스의 패키지부터 스캔
  → com.example.App에 있으면 com.example.** 전체 스캔
  → com.example.config.App에 있으면 com.example.config.** 만 스캔
     → com.example.service, com.example.repository 등 스캔 안 됨!

권장:
  루트 애플리케이션 클래스는 com.example 패키지에 직접 배치
  하위 패키지는 모두 자동 스캔됨
```

### Before: exclude로 Auto-configuration을 끄면 해당 기능이 완전히 제거된다

```java
// ❌ 잘못된 이해
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
// "이제 DataSource 관련 Bean이 아예 없다"

// ✅ 실제:
// DataSourceAutoConfiguration만 제외됨
// 사용자가 직접 DataSource Bean을 등록하면 정상 사용 가능
// → "자동"으로 만들지 않겠다는 뜻, "영원히 없애겠다"가 아님
```

---

## ✨ 올바른 이해와 사용

### After: @SpringBootApplication = 3개 어노테이션의 합성

```java
// @SpringBootApplication 소스 (핵심만)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration          // ← ①
@EnableAutoConfiguration          // ← ②
@ComponentScan(excludeFilters = { // ← ③
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
})
public @interface SpringBootApplication {

    // @EnableAutoConfiguration의 exclude 속성을 직접 노출
    @AliasFor(annotation = EnableAutoConfiguration.class)
    Class<?>[] exclude() default {};

    @AliasFor(annotation = EnableAutoConfiguration.class)
    String[] excludeName() default {};

    // @ComponentScan의 basePackages 속성을 직접 노출
    @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
    String[] scanBasePackages() default {};

    @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
    Class<?>[] scanBasePackageClasses() default {};
}
```

---

## 🔬 내부 동작 원리

### 1. @SpringBootConfiguration — 테스트 컨텍스트 감지용 마커

```java
// @SpringBootConfiguration 소스
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration                    // ← 결국 @Configuration
@Indexed
public @interface SpringBootConfiguration {
    @AliasFor(annotation = Configuration.class)
    boolean proxyBeanMethods() default true;
}
```

```
@SpringBootConfiguration vs @Configuration 차이:

기능적으로는 동일 (CGLIB Full Mode 적용)

추가된 목적:
  @SpringBootTest가 @SpringBootConfiguration을 탐색해
  테스트용 ApplicationContext 시작점을 찾는다

탐색 알고리즘:
  테스트 클래스 위치에서 상위 패키지로 올라가며
  @SpringBootConfiguration이 붙은 클래스를 찾음
  → 찾은 클래스를 SpringApplication의 primarySource로 사용

결론:
  @Configuration만 붙이면 @SpringBootTest가 시작점을 못 찾을 수 있음
  → Spring Boot 애플리케이션에서는 @SpringBootConfiguration 사용
```

```java
// SpringBootTestContextBootstrapper 내부 — 시작점 탐색
protected Class<?> getSpringBootTestAnnotatedClass(Class<?> testClass) {
    // 테스트 클래스에서 상위로 올라가며 @SpringBootConfiguration 탐색
    MergedAnnotations annotations = MergedAnnotations.from(testClass,
        SearchStrategy.TYPE_HIERARCHY);
    // ...
}

// SpringBootConfigurationFinder
Class<?> findFromPackage(String packageName) {
    // packageName부터 상위로 올라가며
    // @SpringBootConfiguration이 붙은 유일한 클래스를 반환
}
```

### 2. @EnableAutoConfiguration — Auto-configuration의 핵심 스위치

```java
// @EnableAutoConfiguration 소스
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage                    // ← 패키지 등록
@Import(AutoConfigurationImportSelector.class) // ← 핵심
public @interface EnableAutoConfiguration {
    Class<?>[] exclude() default {};
    String[] excludeName() default {};
}
```

```java
// @AutoConfigurationPackage — 패키지 등록의 역할
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)  // ← 패키지 등록
public @interface AutoConfigurationPackage {
    String[] basePackages() default {};
    Class<?>[] basePackageClasses() default {};
}

// AutoConfigurationPackages.Registrar
static class Registrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
                                         BeanDefinitionRegistry registry) {
        // @EnableAutoConfiguration이 붙은 클래스의 패키지를 등록
        register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0]));
    }
}
```

```
@AutoConfigurationPackage가 등록한 패키지의 용도:

JPA Auto-configuration에서 사용:
  → EntityScan의 기본 범위 = @SpringBootApplication이 있는 패키지
  → com.example.App → com.example.** 하위의 @Entity 클래스 자동 감지

Spring Data의 Repository 스캔:
  → @EnableJpaRepositories의 기본 범위

즉, @ComponentScan과 별도로
  JPA Entity, Spring Data Repository 스캔 범위를 결정하는 역할
```

### 3. @ComponentScan의 기본 동작과 두 가지 ExcludeFilter

```java
// @SpringBootApplication의 @ComponentScan excludeFilters

// ① TypeExcludeFilter — 테스트 전용 필터 위임
// 테스트 코드에서 TypeExcludeFilter를 구현한 Bean을 등록하면
// 실제 스캔 시 해당 필터가 적용됨 → 테스트 환경 커스텀 스캔 지원

// ② AutoConfigurationExcludeFilter — Auto-configuration 클래스 중복 스캔 방지
public class AutoConfigurationExcludeFilter implements TypeFilter {
    @Override
    public boolean match(MetadataReader metadataReader, ...) {
        // @Configuration이면서 Auto-configuration 후보이기도 한 클래스는 ComponentScan 제외
        // → @EnableAutoConfiguration이 따로 처리하므로 중복 등록 방지
        return isConfiguration(metadataReader) && isAutoConfiguration(metadataReader);
    }
}
```

```
basePackages 기본값 결정:

@ComponentScan(basePackages = "") 일 때 (기본값 = 빈 배열):
  ClassPathScanningCandidateComponentProvider가
  @ComponentScan이 붙은 클래스의 패키지를 기준으로 스캔

  com.example.App → com.example 패키지 + 하위 전체

명시적 설정이 필요한 경우:
  @SpringBootApplication(scanBasePackages = "com.example,com.other")
  또는
  @SpringBootApplication(scanBasePackageClasses = {App.class, LegacyModule.class})
```

### 4. exclude 동작 원리

```java
// exclude/excludeName 처리 — AutoConfigurationImportSelector
protected Set<String> getExclusions(AnnotationMetadata metadata,
                                     AnnotationAttributes attributes) {
    Set<String> excluded = new LinkedHashSet<>();

    // @SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
    excluded.addAll(asList(attributes, "exclude"));

    // @SpringBootApplication(excludeName = "org.springframework.boot...")
    excluded.addAll(asList(attributes, "excludeName"));

    // spring.autoconfigure.exclude 프로퍼티
    excluded.addAll(getExcludeAutoConfigurationsProperty());

    return excluded;
}

// 제외 처리
private List<String> filter(List<String> configurations,
                              AutoConfigurationMetadata autoConfigurationMetadata) {
    // 후보 목록에서 excluded 제거
    configurations.removeAll(exclusions);
    // ...
}
```

```
exclude 세 가지 방법 비교:

1. 어노테이션 속성
   @SpringBootApplication(exclude = SecurityAutoConfiguration.class)
   → 컴파일 타임에 클래스 존재 확인 가능
   → 클래스가 classpath에 없으면 컴파일 오류

2. excludeName 속성
   @SpringBootApplication(excludeName = "org.springframework.boot.autoconfigure.security.SecurityAutoConfiguration")
   → 클래스가 classpath에 없어도 오류 없음
   → 선택적 의존성 제거에 유용

3. 프로퍼티
   spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.security.SecurityAutoConfiguration
   → 코드 변경 없이 환경별로 다르게 제어 가능
   → 운영/테스트 환경별 Auto-configuration 제거에 유용
```

### 5. 메타 어노테이션 합성 — @AliasFor 동작

```java
// @AliasFor로 속성을 상위 어노테이션에 위임
@SpringBootApplication(
    exclude = DataSourceAutoConfiguration.class,    // → @EnableAutoConfiguration.exclude
    scanBasePackages = "com.example"                // → @ComponentScan.basePackages
)
public class App { }

// 내부적으로:
// AnnotationUtils.getAnnotationAttributes()가
// @AliasFor 메타데이터를 분석해 실제 어노테이션에 값 전달
// → @EnableAutoConfiguration의 exclude에 DataSourceAutoConfiguration.class 전달
// → @ComponentScan의 basePackages에 "com.example" 전달
```

### 6. @SpringBootApplication 없이 완전히 동일하게 쓰기

```java
// @SpringBootApplication과 100% 동일한 효과
@SpringBootConfiguration
@EnableAutoConfiguration(exclude = DataSourceAutoConfiguration.class)
@ComponentScan(
    basePackages = "com.example",
    excludeFilters = {
        @ComponentScan.Filter(
            type = FilterType.CUSTOM,
            classes = TypeExcludeFilter.class),
        @ComponentScan.Filter(
            type = FilterType.CUSTOM,
            classes = AutoConfigurationExcludeFilter.class)
    }
)
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

```
AutoConfigurationExcludeFilter를 빠뜨리면:
  @Configuration + Auto-configuration 대상인 클래스가
  @ComponentScan으로 한 번, @EnableAutoConfiguration으로 한 번
  → 총 두 번 등록 시도 → BeanDefinitionOverrideException 가능
```

---

## 💻 실험으로 확인하기

### 실험 1: @EnableAutoConfiguration 제거 시 차이

```java
// @EnableAutoConfiguration 없이
@SpringBootConfiguration
@ComponentScan
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

```
결과:
  DataSource, JPA, Web MVC 등 Auto-configuration 전혀 적용 안 됨
  → @RestController가 있어도 DispatcherServlet 등록 안 됨
  → HTTP 요청 처리 불가
  → 사용자가 모든 Bean을 직접 등록해야 함

내장 서버:
  WebApplicationType은 여전히 SERVLET으로 감지됨
  → 서버는 뜨지만 DispatcherServlet 없어서 404만 반환
```

### 실험 2: @ComponentScan 기본 범위 확인

```
프로젝트 구조:
  com.example.App                  ← @SpringBootApplication 위치
  com.example.service.UserService  ← @Service
  com.other.legacy.LegacyService   ← @Service (다른 패키지)

결과:
  UserService → Bean으로 등록됨 (com.example.** 하위)
  LegacyService → Bean으로 등록 안 됨 (com.example 범위 밖)

해결:
  @SpringBootApplication(scanBasePackages = {"com.example", "com.other"})
```

### 실험 3: --debug로 Auto-configuration 리포트 확인

```bash
java -jar app.jar --debug
```

```
출력 예시 (일부):
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches:     ← 적용된 Auto-configuration
-----------------
   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required class 'javax.sql.DataSource' (...)
      - @ConditionalOnMissingBean (types: javax.sql.DataSource) did not find any beans (...)

Negative matches:     ← 조건 불충족으로 제외된 것
-----------------
   ActiveMQAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'javax.jms.ConnectionFactory'

Exclusions:           ← exclude로 명시적 제거
-----------
   org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### 실험 4: proxyBeanMethods 비교

```java
// proxyBeanMethods = true (기본값, Full Mode)
@SpringBootApplication  // proxyBeanMethods=true
public class App {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // serviceB() 재호출 → 프록시가 가로채 캐시된 Bean 반환
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}

// proxyBeanMethods = false (Lite Mode — 성능 향상)
@SpringBootApplication(proxyBeanMethods = false)
public class App {
    @Bean
    public ServiceA serviceA(ServiceB serviceB) { // 파라미터로 주입받아야 동일 인스턴스 보장
        return new ServiceA(serviceB);
    }
    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
```

---

## ⚙️ 설정 최적화 팁

```java
// 1. 불필요한 Auto-configuration 제거로 시작 시간 단축
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,      // DB 없는 서비스
    SecurityAutoConfiguration.class,        // Security 직접 구성
    FlywayAutoConfiguration.class           // 마이그레이션 수동 관리
})

// 2. 멀티 모듈에서 스캔 범위 명시
@SpringBootApplication(scanBasePackageClasses = {
    App.class,           // com.example (메인 모듈)
    CommonConfig.class   // com.common (공통 모듈)
})

// 3. proxyBeanMethods=false — @Bean 메서드 간 직접 호출 없을 때
//    CGLIB 서브클래스 생성 생략 → 시작 시간 소폭 단축
@SpringBootApplication(proxyBeanMethods = false)
```

---

## 🤔 트레이드오프

```
@SpringBootApplication 편의성:
  장점  세 어노테이션을 하나로, 속성을 직접 노출
  단점  내부 구조를 모르면 exclude, scanBasePackages 동작을 예측하기 어려움

exclude vs 직접 Bean 등록:
  exclude    Auto-configuration 자체를 제거
             → Auto-configuration이 등록하는 모든 Bean 제거
  직접 등록  @ConditionalOnMissingBean이 감지해 Auto-configuration을 자동으로 skip
             → 사용자 정의 Bean이 우선, 필요한 것만 오버라이드

proxyBeanMethods:
  true   CGLIB 프록시 → @Bean 메서드 재호출 시 Singleton 보장, 성능 비용
  false  프록시 없음 → 파라미터 주입으로 의존성 받아야 함, 빠른 시작
         Spring Boot 내부 Auto-configuration 클래스 대부분이 false 사용
```

---

## 📌 핵심 정리

```
@SpringBootApplication = 3개 어노테이션 합성

① @SpringBootConfiguration
   = @Configuration (CGLIB Full Mode)
   + @SpringBootTest가 시작점을 찾는 마커

② @EnableAutoConfiguration
   = @AutoConfigurationPackage (패키지 등록 → JPA Entity 스캔 범위)
   + @Import(AutoConfigurationImportSelector) (Auto-configuration 후보 로딩)

③ @ComponentScan
   기본 범위 = @SpringBootApplication 위치 패키지
   excludeFilters에 TypeExcludeFilter + AutoConfigurationExcludeFilter 포함

exclude 3가지 방법
   어노테이션 속성 (컴파일 타임 검사)
   excludeName 속성 (클래스 없어도 가능)
   spring.autoconfigure.exclude 프로퍼티 (코드 변경 없이)

@AliasFor
   어노테이션 속성을 메타 어노테이션에 위임하는 스프링 메커니즘
   → scanBasePackages → @ComponentScan.basePackages로 전달
```

---

## 🤔 생각해볼 문제

**Q1.** `@SpringBootApplication(scanBasePackages = "com.example")`와 기본값(`scanBasePackages` 미설정)이 동일한 패키지를 스캔하는 경우에도 차이가 있는가?

**Q2.** 다음 구조에서 `@SpringBootTest`는 어떤 클래스를 시작점으로 사용하는가?

```
com.example.App                 ← @SpringBootApplication
com.example.config.TestConfig   ← @TestConfiguration

// 테스트 클래스
com.example.service.UserServiceTest ← @SpringBootTest
```

**Q3.** `@EnableAutoConfiguration`을 제거하고 `DataSourceAutoConfiguration`을 직접 `@Import`하면 어떻게 동작하는가?

> 💡 **해설**
>
> **Q1.** 차이가 있다. `scanBasePackages`를 명시적으로 설정하면 `@ComponentScan`의 `basePackages` 속성이 해당 값으로 고정된다. 기본값(미설정)일 때는 `ClassPathScanningCandidateComponentProvider`가 런타임에 `@SpringBootApplication`이 붙은 클래스의 패키지를 동적으로 계산한다. 결과는 같더라도 명시 설정은 컴파일 타임에 값이 결정되고, 기본값은 런타임에 결정된다는 처리 시점 차이가 있다. 또한 명시 설정 시 클래스를 다른 패키지로 이동하더라도 스캔 범위가 바뀌지 않지만, 기본값은 클래스 위치에 따라 자동으로 변경된다.
>
> **Q2.** `com.example.App`이 시작점으로 사용된다. `@SpringBootTest`는 테스트 클래스(`com.example.service.UserServiceTest`)의 패키지부터 상위 패키지로 탐색하며 `@SpringBootConfiguration`(또는 `@SpringBootApplication`)이 붙은 클래스를 찾는다. `com.example.service` → `com.example` 순서로 탐색하며 `com.example.App`을 발견하면 이를 `primarySource`로 사용해 전체 애플리케이션 컨텍스트를 로드한다. `TestConfig`는 `@TestConfiguration`이므로 탐색 대상이 아니다.
>
> **Q3.** `DataSourceAutoConfiguration`만 단독으로 적용된다. `@Import`로 직접 지정하면 해당 설정 클래스는 로딩되고 `@Conditional` 조건을 만족하면 Bean이 등록된다. 하지만 `@EnableAutoConfiguration`이 없으므로 다른 Auto-configuration(`JpaAutoConfiguration`, `WebMvcAutoConfiguration` 등)은 전혀 로딩되지 않는다. 또한 `@AutoConfigurationPackage`도 없으므로 JPA Entity 스캔 범위가 등록되지 않아 `@Entity` 클래스를 인식하지 못할 수 있다. 개별 Auto-configuration을 선별해서 테스트할 때 유용하지만, 운영 코드에서 이 방식은 권장되지 않는다.

---

<div align="center">

**[⬅️ 이전: SpringApplication.run() 전체 과정](./01-spring-application-run.md)** | **[홈으로 🏠](../README.md)** | **[다음: @EnableAutoConfiguration 동작 원리 ➡️](./03-enable-auto-configuration.md)**

</div>
