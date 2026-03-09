# spring.factories vs @AutoConfiguration — Boot 3.x에서 무엇이 바뀌었는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `spring.factories`는 무엇이고 어떻게 동작하는가?
- Boot 3.x에서 왜 `spring.factories` 방식을 폐기했는가?
- `@AutoConfiguration` 어노테이션과 `.imports` 파일 방식이 다른 점은?
- `SpringFactoriesLoader`는 내부에서 어떻게 파일을 읽는가?
- Boot 2.7.x를 중간 단계로 사용하는 마이그레이션 전략은?
- `@AutoConfiguration`의 `before`, `after` 속성은 어떻게 동작하는가?

---

## 🔍 왜 이게 존재하는가

### spring.factories — 스프링의 SPI(Service Provider Interface) 메커니즘

```java
// Java 표준 SPI (java.util.ServiceLoader)
// META-INF/services/com.example.MyInterface 파일에 구현체 클래스명 나열
// → ServiceLoader.load(MyInterface.class)로 로딩

// Spring의 확장 SPI
// META-INF/spring.factories 파일에 인터페이스 → 구현체 목록 매핑
// → SpringFactoriesLoader.loadFactoryNames()로 로딩
```

```
spring.factories 파일 예시:
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    com.example.FooAutoConfiguration,\
    com.example.BarAutoConfiguration

  org.springframework.context.ApplicationListener=\
    com.example.MyApplicationListener

  org.springframework.boot.SpringApplicationRunListener=\
    com.example.MyRunListener

하나의 파일로 여러 종류의 확장점을 모두 등록하는 중앙 집중식 구조
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: spring.factories는 Auto-configuration 전용이다

```
❌ 잘못된 이해:
  "spring.factories = Auto-configuration 목록 파일"

✅ 실제:
  spring.factories는 Spring Framework 전반의 SPI 메커니즘
  다음 모든 확장점을 등록:
  
  Auto-configuration 관련:
    EnableAutoConfiguration       → Auto-configuration 클래스 목록
    AutoConfigurationImportFilter → 조건 필터
    
  Spring Boot 시작 과정:
    ApplicationContextInitializer → Context 초기화
    ApplicationListener           → 이벤트 리스너
    SpringApplicationRunListener  → 실행 단계 훅
    BootstrapRegistryInitializer  → 부트스트랩 컨텍스트
    
  Spring Framework 핵심:
    PropertySourceLoader          → 프로퍼티 파일 포맷 확장
    FailureAnalyzer               → 시작 실패 분석기
```

### Before: Boot 3.x에서 spring.factories는 완전히 제거됐다

```
❌ 잘못된 이해:
  "Boot 3.x에서 spring.factories 파일 자체가 없어졌다"

✅ 실제:
  spring.factories 파일 자체는 여전히 존재
  
  변경된 것:
    EnableAutoConfiguration 키만 spring.factories에서 제거
    → 별도 .imports 파일로 분리
  
  여전히 spring.factories로 등록하는 것들:
    ApplicationContextInitializer
    ApplicationListener
    SpringApplicationRunListener
    FailureAnalyzer
    PropertySourceLoader
    등 Auto-configuration 이외의 모든 확장점
```

---

## ✨ 올바른 이해와 사용

### After: Boot 2.x → 3.x 등록 방식 비교

```
Boot 2.x:
  파일: META-INF/spring.factories
  내용:
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
      com.example.MyAutoConfiguration

  어노테이션: @Configuration (또는 별도 어노테이션 없음)

---

Boot 3.x:
  파일: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
  내용:
    com.example.MyAutoConfiguration

  어노테이션: @AutoConfiguration (새로 도입)

---

Boot 2.7.x (과도기):
  두 방식 모두 지원
  → spring.factories의 EnableAutoConfiguration 키도 읽음
  → .imports 파일도 읽음
  → 마이그레이션 중간 단계로 활용 가능
```

---

## 🔬 내부 동작 원리

### 1. SpringFactoriesLoader — spring.factories 로딩

```java
// SpringFactoriesLoader.java
public final class SpringFactoriesLoader {

    // 파일 경로 상수
    public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

    // 팩토리 이름 목록 조회 (클래스명만)
    public static List<String> loadFactoryNames(Class<?> factoryType,
                                                  ClassLoader classLoader) {
        return loadSpringFactories(classLoader)
            .getOrDefault(factoryType.getName(), Collections.emptyList());
    }

    // 실제 로딩 — ClassLoader로 classpath 전체 탐색
    private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
        // 캐시 확인
        Map<String, List<String>> result = cache.get(classLoader);
        if (result != null) {
            return result;
        }

        result = new HashMap<>();
        // classpath의 모든 META-INF/spring.factories 파일 탐색
        Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);

        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);

            // 키=값 형태로 파싱 (값은 콤마로 구분된 클래스명 목록)
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                String[] factoryImplementationNames =
                    StringUtils.commaDelimitedListToStringArray((String) entry.getValue());

                for (String factoryImplementationName : factoryImplementationNames) {
                    result.computeIfAbsent(factoryTypeName, k -> new ArrayList<>())
                          .add(factoryImplementationName.trim());
                }
            }
        }

        // ClassLoader 기준으로 결과 캐시
        cache.put(classLoader, Collections.unmodifiableMap(result));
        return result;
    }
}
```

```
캐시 전략:
  ClassLoader를 키로 캐시
  → 동일 ClassLoader로 여러 번 조회해도 파일 I/O는 최초 1회만
  → 멀티 모듈에서 ClassLoader가 다르면 각각 캐시

classpath 전체 탐색:
  application JAR의 META-INF/spring.factories
  + spring-boot.jar의 META-INF/spring.factories
  + spring-boot-autoconfigure.jar의 META-INF/spring.factories
  + 각 라이브러리 JAR의 META-INF/spring.factories
  → 모두 합산하여 하나의 Map 생성
```

### 2. Boot 3.x의 ImportCandidates — .imports 파일 로딩

```java
// ImportCandidates.java (Boot 3.x)
public final class ImportCandidates implements Iterable<String> {

    // 파일 경로 패턴
    private static final String LOCATION = "META-INF/spring/%s.imports";

    public static ImportCandidates load(Class<?> annotation, ClassLoader classLoader) {
        // 파일명 = 어노테이션 클래스의 완전한 이름
        // 예: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
        String location = String.format(LOCATION, annotation.getName());

        Enumeration<URL> urls = classLoader.getResources(location);
        List<String> importCandidates = new ArrayList<>();

        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            // 파일 읽기 (한 줄 = 하나의 클래스명)
            try (BufferedReader reader = new BufferedReader(
                    new InputStreamReader(url.openStream()))) {
                reader.lines()
                    .map(String::trim)
                    .filter(line -> !line.isEmpty() && !line.startsWith("#")) // 빈 줄·주석 제외
                    .forEach(importCandidates::add);
            }
        }
        return new ImportCandidates(importCandidates);
    }
}
```

```
.imports 파일 형식:
  # 주석은 #으로 시작
  # 한 줄에 하나의 클래스명
  org.springframework.boot.autoconfigure.aop.AopAutoConfiguration
  org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration
  ...

spring.factories 형식과 비교:
  spring.factories:
    key=value1,\
    value2,\
    value3
    → 긴 목록은 백슬래시로 줄바꿈 → 가독성 나쁨, 파싱 복잡

  .imports:
    value1
    value2
    value3
    → 한 줄 한 클래스명 → 가독성 좋음, git diff 명확
```

### 3. @AutoConfiguration 어노테이션

```java
// @AutoConfiguration 소스 (Boot 3.x)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration(proxyBeanMethods = false)   // ← proxyBeanMethods=false 기본값
@AutoConfigureBefore                        // ← before 속성 통합
@AutoConfigureAfter                         // ← after 속성 통합
public @interface AutoConfiguration {

    @AliasFor(annotation = Configuration.class, attribute = "proxyBeanMethods")
    boolean proxyBeanMethods() default false;

    // @AutoConfigureBefore의 value와 동일
    @AliasFor(annotation = AutoConfigureBefore.class, attribute = "value")
    Class<?>[] before() default {};

    @AliasFor(annotation = AutoConfigureBefore.class, attribute = "name")
    String[] beforeName() default {};

    // @AutoConfigureAfter의 value와 동일
    @AliasFor(annotation = AutoConfigureAfter.class, attribute = "value")
    Class<?>[] after() default {};

    @AliasFor(annotation = AutoConfigureAfter.class, attribute = "name")
    String[] afterName() default {};
}
```

```
@AutoConfiguration의 주요 변화:

1. proxyBeanMethods = false 기본값
   Boot 2.x의 Auto-configuration 클래스들은 @Configuration만 사용
   → proxyBeanMethods 기본값 = true (CGLIB 프록시)
   → Auto-configuration은 대부분 @Bean 메서드 간 직접 호출 없음
   → 불필요한 CGLIB 프록시 생성
   
   Boot 3.x @AutoConfiguration:
   → proxyBeanMethods = false 기본값
   → CGLIB 프록시 생략 → 시작 성능 향상

2. before/after 속성 통합
   Boot 2.x:
     @Configuration
     @AutoConfigureAfter(DataSourceAutoConfiguration.class)
     @AutoConfigureBefore(JpaRepositoriesAutoConfiguration.class)
     class HibernateJpaAutoConfiguration { }
   
   Boot 3.x:
     @AutoConfiguration(
       after = DataSourceAutoConfiguration.class,
       before = JpaRepositoriesAutoConfiguration.class
     )
     class HibernateJpaAutoConfiguration { }
```

### 4. Boot 2.7.x 과도기 — 두 방식 동시 지원

```java
// Boot 2.7.x AutoConfigurationImportSelector
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
                                                    AnnotationAttributes attributes) {
    // Boot 2.x 방식: spring.factories 읽기
    List<String> configurations = new ArrayList<>(
        SpringFactoriesLoader.loadFactoryNames(EnableAutoConfiguration.class,
                                                getBeanClassLoader()));

    // Boot 3.x 방식: .imports 파일 읽기 (2.7에서 추가)
    ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader())
        .forEach(configurations::add);

    // 둘을 합산하여 반환
    return configurations;
}
```

```
마이그레이션 전략:

1단계 (Boot 2.7 대상):
  spring.factories의 EnableAutoConfiguration 유지
  + .imports 파일도 추가 (두 군데 모두 등록)

2단계 (Boot 3.x 전환 후):
  spring.factories에서 EnableAutoConfiguration 키 제거
  .imports 파일만 유지

주의:
  ApplicationContextInitializer, ApplicationListener 등은
  여전히 spring.factories 방식 사용
  → 이것들은 .imports로 이전하지 않음
```

### 5. 왜 spring.factories에서 분리했는가 — 설계 이유

```
분리 이유 1: 단일 책임
  spring.factories:
    Auto-configuration 외의 모든 확장점을 하나의 파일에 혼합
    → 파일이 거대해지고 목적이 불분명
  .imports:
    Auto-configuration 클래스 목록만 담음
    → 명확한 목적, 단일 책임

분리 이유 2: Git diff 가독성
  spring.factories:
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
      com.example.AAutoConfiguration,\
      com.example.BAutoConfiguration    ← 하나 추가/제거 시 줄 수 변화 불규칙
  .imports:
    com.example.AAutoConfiguration
    com.example.BAutoConfiguration      ← 하나 추가/제거 시 항상 한 줄 변경 → diff 명확

분리 이유 3: GraalVM Native Image
  spring.factories 파싱은 런타임에 Properties 객체 생성
  .imports는 단순 텍스트 라인 리딩
  → Native Image AOT 처리에서 .imports 방식이 더 분석하기 쉬움
  → 리플렉션 힌트 생성 자동화에 유리

분리 이유 4: 성능 (미미하지만)
  spring.factories는 모든 키에 대해 파싱
  .imports는 Auto-configuration 목록만 읽음
  → 파일 분리로 각 용도별 로딩 최적화 가능
```

---

## 💻 실험으로 확인하기

### 실험 1: spring.factories 현재 내용 확인

```bash
# Boot 3.x spring-boot-autoconfigure.jar의 spring.factories
unzip -p ~/.m2/.../spring-boot-autoconfigure-3.2.0.jar META-INF/spring.factories

# EnableAutoConfiguration 키 없음 확인
# 대신 다른 확장점들만 남아 있음
```

### 실험 2: .imports 파일 항목 수 세기

```bash
unzip -p ~/.m2/.../spring-boot-autoconfigure-3.2.0.jar \
  "META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports" \
  | wc -l
# 약 148줄 (Boot 3.2 기준)
```

### 실험 3: 커스텀 라이브러리 등록 — Boot 2.x vs 3.x

```
Boot 2.x 방식:
  src/main/resources/META-INF/spring.factories:
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
      com.mylib.MyAutoConfiguration

Boot 3.x 방식:
  src/main/resources/META-INF/spring/
    org.springframework.boot.autoconfigure.AutoConfiguration.imports:
      com.mylib.MyAutoConfiguration
```

```java
// @AutoConfiguration 사용 (Boot 3.x 권장)
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
@ConditionalOnClass(MyLibrary.class)
public class MyAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(DataSource dataSource) {
        return new MyService(dataSource);
    }
}
```

---

## ⚙️ 설정 최적화 팁

```java
// Auto-configuration 클래스 작성 시 권장 패턴 (Boot 3.x)

// 1. @AutoConfiguration 사용 (proxyBeanMethods=false 기본)
@AutoConfiguration(after = DataSourceAutoConfiguration.class)

// 2. 클래스 레벨 조건 — 1차 필터링에 활용됨
@ConditionalOnClass(MyLibrary.class)

// 3. Bean 레벨 조건 — 사용자 Bean 존재 시 스킵
@ConditionalOnMissingBean(MyService.class)

// 4. spring-boot-autoconfigure-processor 추가 → 컴파일 타임 메타데이터 생성
// pom.xml:
// <dependency>
//   <groupId>org.springframework.boot</groupId>
//   <artifactId>spring-boot-autoconfigure-processor</artifactId>
//   <optional>true</optional>
// </dependency>
```

---

## 🤔 트레이드오프

```
spring.factories (통합 파일):
  장점  하나의 파일로 모든 확장점 관리, 기존 Spring 생태계와의 호환
  단점  파일이 커짐, Auto-configuration 목록을 별도로 구분하기 어려움
       Git diff 가독성 낮음, Native Image 분석 복잡

.imports (분리 파일):
  장점  목적별 파일 분리, 한 줄 한 항목으로 Git diff 명확
       Native Image 처리 친화적
  단점  Boot 2.x 라이브러리와 호환성 추가 작업 필요

@AutoConfiguration(proxyBeanMethods=false):
  장점  CGLIB 프록시 생략 → 시작 성능 향상, 메모리 절약
  단점  @Bean 메서드 간 직접 호출로 Singleton 보장 안 됨
       → 필요 시 명시적 proxyBeanMethods=true 사용
```

---

## 📌 핵심 정리

```
spring.factories (Boot 2.x):
  META-INF/spring.factories
  하나의 파일에 모든 확장점 (EnableAutoConfiguration 포함)
  SpringFactoriesLoader.loadFactoryNames() 로딩

.imports (Boot 3.x):
  META-INF/spring/{AnnotationClassName}.imports
  Auto-configuration 전용 분리 파일
  ImportCandidates.load() 로딩

@AutoConfiguration (Boot 3.x):
  = @Configuration(proxyBeanMethods=false)
    + @AutoConfigureAfter + @AutoConfigureBefore 통합
  proxyBeanMethods=false → CGLIB 프록시 생략 → 성능 향상

Boot 2.7.x 과도기:
  두 방식 모두 지원 → 점진적 마이그레이션 가능

여전히 spring.factories로:
  ApplicationContextInitializer
  ApplicationListener
  SpringApplicationRunListener
  FailureAnalyzer 등 (Auto-configuration 외 모든 확장점)
```

---

## 🤔 생각해볼 문제

**Q1.** Boot 3.x 프로젝트에서 Boot 2.x 스타일로 `spring.factories`에 `EnableAutoConfiguration`을 등록한 외부 라이브러리를 사용하면 어떻게 되는가?

**Q2.** `@AutoConfiguration(proxyBeanMethods = false)`일 때 다음 코드에서 문제가 생기는가?

```java
@AutoConfiguration(proxyBeanMethods = false)
public class MyAutoConfiguration {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // serviceB() 직접 호출
    }
    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
```

**Q3.** `SpringFactoriesLoader`의 캐시 전략에서 `ClassLoader`를 키로 사용하는 이유는 무엇인가?

> 💡 **해설**
>
> **Q1.** Boot 3.x에서는 `spring.factories`의 `EnableAutoConfiguration` 키를 읽지 않는다. `AutoConfigurationImportSelector`가 `ImportCandidates.load(AutoConfiguration.class, ...)`만 호출하기 때문에 해당 라이브러리의 Auto-configuration은 적용되지 않는다. 따라서 라이브러리 제공자가 Boot 3.x 호환 버전을 배포하거나, 사용자가 직접 `META-INF/spring/...AutoConfiguration.imports` 파일을 프로젝트에 추가해 해당 Auto-configuration 클래스를 등록해야 한다.
>
> **Q2.** 문제가 생긴다. `proxyBeanMethods = false`이면 CGLIB 프록시가 씌워지지 않으므로 `serviceB()`를 직접 호출하면 프록시 인터셉터 없이 그냥 새 `ServiceB()` 인스턴스가 생성된다. `serviceA()`가 받는 `ServiceB`는 Spring Bean으로 등록된 인스턴스와 다른 별개의 객체가 된다. 해결하려면 `serviceB()`를 직접 호출하는 대신 파라미터로 주입받아야 한다: `public ServiceA serviceA(ServiceB serviceB) { return new ServiceA(serviceB); }`.
>
> **Q3.** 애플리케이션에서 여러 `ClassLoader`가 공존할 수 있기 때문이다. Spring MVC 환경에서 Root Context의 `ClassLoader`와 Servlet Context의 `ClassLoader`가 다를 수 있고, 멀티 모듈 환경에서도 각 모듈이 다른 `ClassLoader`를 사용할 수 있다. `ClassLoader`마다 접근 가능한 `META-INF/spring.factories` 파일이 다르기 때문에, 동일한 파일 경로라도 `ClassLoader`에 따라 다른 결과가 나올 수 있다. 따라서 캐시 키로 `ClassLoader`를 사용해 같은 `ClassLoader`로 반복 조회 시에는 캐시를 활용하되, 다른 `ClassLoader`에서는 별도로 로딩하도록 한다.

---

<div align="center">

**[⬅️ 이전: @EnableAutoConfiguration 동작 원리](./03-enable-auto-configuration.md)** | **[홈으로 🏠](../README.md)** | **[다음: @Conditional 어노테이션 평가 순서 ➡️](./05-conditional-evaluation-order.md)**

</div>
