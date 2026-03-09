# @ConditionalOnClass 동작 메커니즘 — 클래스 로딩 없이 클래스를 감지한다

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@ConditionalOnClass`는 어떻게 클래스 로딩 없이 클래스 존재를 확인하는가?
- `FilteringSpringBootCondition`과 일반 `Condition`의 처리 경로가 다른 이유는?
- `OnClassCondition`이 `AutoConfigurationMetadata`를 활용하는 방식은?
- `@ConditionalOnClass`를 클래스 레벨에 두는 것과 `@Bean` 메서드에 두는 것의 차이는?
- `ClassNameFilter`가 내부적으로 `ClassLoader.getResource()`를 사용하는 이유는?
- `@ConditionalOnMissingClass`와의 관계는?

---

## 🔍 왜 이게 존재하는가

### 문제: 클래스가 없는데 타입 참조를 하면 ClassNotFoundException

```java
// 문제 상황: HikariCP가 없는 프로젝트에서 DataSource Auto-configuration 처리 시
@Configuration
public class DataSourceAutoConfiguration {

    @Bean
    public HikariDataSource dataSource() {  // HikariDataSource 타입 참조!
        return new HikariDataSource();       // → HikariCP가 없으면 ClassNotFoundException
    }
}
```

```
더 큰 문제: @Configuration 클래스 로딩 자체
  Auto-configuration 후보 클래스들을 처리하려면 먼저 로딩해야 함
  → 클래스를 로딩하면 클래스 파일이 없을 때 NoClassDefFoundError
  → 150개 Auto-configuration 클래스를 다 로딩하면 시작 시간 폭증

해결:
  클래스 로딩 없이 클래스 파일 존재 여부만 확인
  → ClassLoader.getResource()로 .class 파일을 리소스로 탐색
  → Class.forName() / classLoader.loadClass() 사용하지 않음
  → NoClassDefFoundError 없이 안전하게 조건 평가
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @ConditionalOnClass의 value에 존재하지 않는 클래스를 참조할 수 없다

```java
// ❌ 잘못된 이해
// "classpath에 없는 클래스를 value에 쓰면 컴파일 오류가 난다"

@Configuration
@ConditionalOnClass(HikariDataSource.class)  // 컴파일 오류?
public class HikariAutoConfiguration { }

// ✅ 실제:
// @ConditionalOnClass의 value = Class<?>[] → 컴파일 타임에 클래스 존재 필요
// 컴파일 오류 아님 (그 클래스의 JAR가 compile scope로 있어야 함)
// 런타임에 해당 JAR가 없어도 문제 없음 → Condition이 false 반환

// ⚠️ 실제로 컴파일 오류가 나는 경우:
// @Bean 메서드 시그니처에 해당 타입 직접 사용 시
@Bean
public HikariDataSource ds() { ... }  // HikariCP JAR 없으면 컴파일 오류
→ 해결: 내부 @Configuration 클래스로 분리 (다음 섹션 참조)
```

### Before: @ConditionalOnClass가 클래스 로딩을 수행한다

```
❌ 잘못된 이해:
  "어차피 조건 평가하려면 클래스를 로딩해야 한다"

✅ 실제:
  1차 필터링 (AutoConfigurationMetadata 사용):
    .class 파일을 리소스로 탐색 (getResource)
    → 클래스 로딩 없음
    → 수십~수백 개를 빠르게 제거

  2차 평가 (OnClassCondition.getMatchOutcome):
    ClassNameFilter.MISSING.filter(classes, classLoader)
    → ClassUtils.isPresent() 사용
    → 내부적으로 classLoader.loadClass() 시도하지만 예외 처리로 안전
```

---

## ✨ 올바른 이해와 사용

### After: 두 단계의 클래스 존재 확인

```
단계 1: AutoConfigurationMetadata 기반 사전 필터링 (가장 빠름)
  컴파일 타임에 생성된 spring-autoconfigure-metadata.properties 읽기
  → "DataSourceAutoConfiguration.ConditionalOnClass=javax.sql.DataSource,..."
  → 클래스명 문자열로 ClassLoader.getResource() 호출
  → .class 파일 존재 여부만 확인 (로딩 없음)
  → 없으면 즉시 제외

단계 2: OnClassCondition.getMatchOutcome() (통과한 것들만)
  ClassNameFilter를 통한 정밀 체크
  → ClassUtils.isPresent() → classLoader.loadClass() 시도
  → ClassNotFoundException 포착 → false 반환
  → 실제 Class 객체까지 로딩될 수 있지만 이미 필터링된 소수만 처리
```

---

## 🔬 내부 동작 원리

### 1. FilteringSpringBootCondition — 배치 처리용 Condition

```java
// FilteringSpringBootCondition.java
// 일반 Condition과 달리 배치로 여러 후보를 한 번에 처리
abstract class FilteringSpringBootCondition extends SpringBootCondition
        implements AutoConfigurationImportFilter, BeanFactoryAware, BeanClassLoaderAware {

    // AutoConfigurationImportFilter.match() — 1차 필터링
    @Override
    public boolean[] match(String[] autoConfigurationClasses,
                            AutoConfigurationMetadata autoConfigurationMetadata) {

        ConditionEvaluationReport report = ...;
        ConditionOutcome[] outcomes =
            getOutcomes(autoConfigurationClasses, autoConfigurationMetadata);

        boolean[] match = new boolean[outcomes.length];
        for (int i = 0; i < outcomes.length; i++) {
            match[i] = (outcomes[i] == null || outcomes[i].isMatch());
            if (!match[i] && outcomes[i] != null) {
                // 조건 미충족 이유를 리포트에 기록 (--debug 출력용)
                logOutcome(autoConfigurationClasses[i], outcomes[i]);
                if (report != null) {
                    report.recordConditionEvaluation(autoConfigurationClasses[i],
                        this, outcomes[i]);
                }
            }
        }
        return match;
    }

    // 서브클래스가 구현: 실제 조건 평가
    protected abstract ConditionOutcome[] getOutcomes(
        String[] autoConfigurationClasses,
        AutoConfigurationMetadata autoConfigurationMetadata);
}
```

### 2. OnClassCondition — 두 단계 구현

```java
// OnClassCondition.java
@Order(Ordered.HIGHEST_PRECEDENCE + 6)
class OnClassCondition extends FilteringSpringBootCondition {

    // ─── 1단계: 배치 필터링 (AutoConfigurationMetadata 사용) ───

    @Override
    protected final ConditionOutcome[] getOutcomes(
            String[] autoConfigurationClasses,
            AutoConfigurationMetadata autoConfigurationMetadata) {

        // 멀티스레드 분할 처리 (후보가 많을 경우 병렬화)
        if (autoConfigurationClasses.length > 1
                && Runtime.getRuntime().availableProcessors() > 1) {
            return resolveOutcomesThreaded(autoConfigurationClasses, autoConfigurationMetadata);
        }
        return resolveOutcomes(autoConfigurationClasses, autoConfigurationMetadata);
    }

    private ConditionOutcome[] resolveOutcomes(String[] classes,
                                                AutoConfigurationMetadata metadata) {
        OutcomesResolver resolver = new StandardOutcomesResolver(classes, 0,
            classes.length, metadata, getBeanClassLoader());
        return resolver.resolveOutcomes();
    }

    // StandardOutcomesResolver.resolveOutcomes()
    // → 각 Auto-configuration 클래스에 대해:
    //   metadata.get(className, "ConditionalOnClass") 조회
    //   → "javax.sql.DataSource,org.springframework.jdbc..." 같은 문자열 반환
    //   → getOutcome(candidates, ClassNameFilter.MISSING, classLoader)

    // ─── 2단계: 개별 @Conditional 평가 (springBootCondition) ───

    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context,
                                             AnnotatedTypeMetadata metadata) {
        ClassLoader classLoader = context.getClassLoader();
        ConditionMessage matchMessage = ConditionMessage.empty();

        // @ConditionalOnClass value 속성
        List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
        if (onClasses != null) {
            // MISSING: 없는 클래스만 필터링 → 있어야 하는데 없으면 실패
            List<String> missing = filter(onClasses, ClassNameFilter.MISSING, classLoader);
            if (!missing.isEmpty()) {
                return ConditionOutcome.noMatch(
                    ConditionMessage.forCondition(ConditionalOnClass.class)
                        .didNotFind("required class", "required classes")
                        .items(Style.QUOTE, missing));
            }
            matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
                .found("required class", "required classes")
                .items(Style.QUOTE, onClasses);
        }

        // @ConditionalOnMissingClass value 속성
        List<String> onMissingClasses = getCandidates(metadata, ConditionalOnMissingClass.class);
        if (onMissingClasses != null) {
            // PRESENT: 있는 클래스만 필터링 → 없어야 하는데 있으면 실패
            List<String> present = filter(onMissingClasses, ClassNameFilter.PRESENT, classLoader);
            if (!present.isEmpty()) {
                return ConditionOutcome.noMatch(
                    ConditionMessage.forCondition(ConditionalOnMissingClass.class)
                        .found("unwanted class", "unwanted classes")
                        .items(Style.QUOTE, present));
            }
            matchMessage = matchMessage.andCondition(ConditionalOnMissingClass.class)
                .didNotFind("unwanted class", "unwanted classes")
                .items(Style.QUOTE, onMissingClasses);
        }

        return ConditionOutcome.match(matchMessage);
    }
}
```

### 3. ClassNameFilter — 클래스 로딩 최소화 핵심

```java
// FilteringSpringBootCondition.ClassNameFilter
protected enum ClassNameFilter {

    PRESENT {
        @Override
        public boolean matches(String className, ClassLoader classLoader) {
            return isPresent(className, classLoader);
        }
    },

    MISSING {
        @Override
        public boolean matches(String className, ClassLoader classLoader) {
            return !isPresent(className, classLoader);
        }
    };

    abstract boolean matches(String className, ClassLoader classLoader);

    // 클래스 존재 여부 확인
    static boolean isPresent(String className, ClassLoader classLoader) {
        if (classLoader == null) {
            classLoader = ClassUtils.getDefaultClassLoader();
        }
        try {
            // resolve=false: 클래스 초기화(static initializer) 실행 안 함
            // 클래스 파일을 메모리에 올리지만 초기화는 생략
            Class.forName(className, false, classLoader);
            return true;
        }
        catch (ClassNotFoundException | NoClassDefFoundError ex) {
            return false;
        }
    }

    static List<String> filter(List<String> classNames,
                                ClassNameFilter classNameFilter,
                                ClassLoader classLoader) {
        if (CollectionUtils.isEmpty(classNames)) {
            return Collections.emptyList();
        }
        return classNames.stream()
            .filter(c -> classNameFilter.matches(c, classLoader))
            .collect(Collectors.toList());
    }
}
```

```
Class.forName(name, false, loader) vs Class.forName(name):

Class.forName(name, false, loader):
  resolve = false → 클래스 링킹(verification, preparation, resolution)은 하지만
  초기화(static initializer, static 필드 초기화) 실행 안 함
  → 클래스 파일을 읽어 JVM에 등록하지만 코드는 실행 안 됨
  → 존재 확인 용도로 적합 (부작용 최소화)

Class.forName(name):
  = Class.forName(name, true, currentLoader)
  → 초기화까지 실행 → 부작용 가능성 있음

1차 필터링 (AutoConfigurationMetadata):
  classLoader.getResource(className.replace('.', '/') + ".class")
  → 파일 존재 여부만 확인, 클래스 로딩 없음 (가장 빠름)
```

### 4. @ConditionalOnClass를 클래스 vs @Bean 메서드에 위치시키는 전략

```java
// 문제: @Bean 반환 타입에 외부 라이브러리 타입을 직접 쓰면
//       해당 JAR 없을 때 클래스 로딩 시 오류

// ❌ 위험한 방식 — 같은 @Configuration 클래스에서
@Configuration
@ConditionalOnClass(HikariDataSource.class)
public class DataSourceAutoConfiguration {

    @Bean
    public HikariDataSource hikariDataSource(...) {  // HikariDataSource 타입 참조
        // HikariCP JAR가 없으면 이 클래스 로딩 시 오류
        return new HikariDataSource();
    }
}
// @ConditionalOnClass가 PARSE_CONFIGURATION에서 평가되어도
// 이미 @Configuration 클래스 자체가 로딩될 때 문제 발생 가능

// ✅ 올바른 방식 — 내부 static @Configuration 클래스로 분리
@Configuration
@ConditionalOnClass(HikariDataSource.class)
public class DataSourceAutoConfiguration {

    // 외부 타입을 안 쓰는 외부 Configuration
    @Bean
    @ConditionalOnMissingBean
    public DataSourceProperties dataSourceProperties() {
        return new DataSourceProperties();
    }

    // 외부 타입(HikariDataSource)을 쓰는 내부 Configuration
    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(HikariDataSource.class)
    static class HikariDataSourceConfiguration {

        @Bean
        @ConditionalOnMissingBean(DataSource.class)
        public HikariDataSource dataSource(DataSourceProperties properties) {
            // 이 내부 클래스가 로딩될 때는 이미 HikariCP 존재 확인됨
            HikariDataSource dataSource = properties.initializeDataSourceBuilder()
                .type(HikariDataSource.class).build();
            return dataSource;
        }
    }
}
```

```
내부 static @Configuration 분리 패턴이 안전한 이유:

외부 클래스 DataSourceAutoConfiguration 로딩:
  → @ConditionalOnClass 평가 → HikariCP 없으면 전체 스킵
  → 있으면 파싱 진행

내부 클래스 HikariDataSourceConfiguration 로딩:
  → 이 시점에는 이미 HikariCP 존재 확인됨
  → HikariDataSource 타입 참조 안전

Spring Boot 실제 소스에서도 이 패턴 광범위하게 사용
```

### 5. AutoConfigurationMetadata — 컴파일 타임 최적화

```java
// AutoConfigurationMetadata.java — 메타데이터 조회 인터페이스
public interface AutoConfigurationMetadata {
    boolean wasProcessed(String className);

    // ConditionalOnClass 조건 클래스명 목록 반환
    Set<String> getSet(String className, String key);
    // 예: getSet("...DataSourceAutoConfiguration", "ConditionalOnClass")
    //   → {"javax.sql.DataSource", "org.springframework.jdbc...EmbeddedDatabaseType"}
}

// 실제 파일: META-INF/spring-autoconfigure-metadata.properties
// 생성: spring-boot-autoconfigure-processor (APT)

// 파일 내용 예시:
// org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration=
// org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.ConditionalOnClass=\
//   javax.sql.DataSource,\
//   org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType
// org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.AutoConfigureAfter=\
//   org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration
```

---

## 💻 실험으로 확인하기

### 실험 1: --debug로 @ConditionalOnClass 평가 결과 확인

```bash
java -jar app.jar --debug 2>&1 | grep -A 5 "ConditionalOnClass"
```

```
출력 예시:
ActiveMQAutoConfiguration:
   Did not match:
      - @ConditionalOnClass did not find required class
        'jakarta.jms.ConnectionFactory' (OnClassCondition)

DataSourceAutoConfiguration matched:
   - @ConditionalOnClass found required classes
     'javax.sql.DataSource',
     'org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType'
     (OnClassCondition)
```

### 실험 2: ClassNameFilter 동작 직접 확인

```java
// ClassLoader.getResource()로 클래스 파일 존재 확인 (로딩 없음)
ClassLoader cl = Thread.currentThread().getContextClassLoader();

// .class 파일 경로로 변환
String classPath = "com/zaxxer/hikari/HikariDataSource.class";
URL resource = cl.getResource(classPath);

System.out.println("HikariCP 존재: " + (resource != null));
// classpath에 hikari JAR 있으면 true, 없으면 false
// Class.forName() 없이 존재 확인!
```

### 실험 3: @ConditionalOnClass + @ConditionalOnMissingClass 조합

```java
// Reactive가 없을 때만 동작하는 MVC Auto-configuration 패턴
@AutoConfiguration
@ConditionalOnClass(DispatcherServlet.class)         // MVC 클래스 있어야 함
@ConditionalOnMissingClass("org.springframework.web.reactive.DispatcherHandler") // Reactive 없어야 함
public class OnlyMvcAutoConfiguration {
    // Reactive 환경에서는 이 설정 자동 스킵
}
```

### 실험 4: spring-autoconfigure-metadata.properties 직접 확인

```bash
unzip -p ~/.m2/.../spring-boot-autoconfigure-3.2.0.jar \
  META-INF/spring-autoconfigure-metadata.properties \
  | grep "DataSource" | head -5
```

```
출력:
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.ConditionalOnClass=\
  javax.sql.DataSource,\
  org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType

→ 이 데이터를 읽어 Class.forName() 없이 1차 필터링
→ 두 클래스 모두 classpath에 없으면 즉시 제외
```

---

## ⚙️ 설정 최적화 팁

```java
// 1. 커스텀 Auto-configuration 작성 시 클래스 레벨 @ConditionalOnClass 필수
//    → 1차 필터링 효과 활용
@AutoConfiguration
@ConditionalOnClass(MyLibrary.class)  // 반드시 클래스 레벨에
public class MyAutoConfiguration { }

// 2. 타입 참조가 필요한 @Bean은 내부 클래스로 분리
@AutoConfiguration
@ConditionalOnClass(MyLibrary.class)
public class MyAutoConfiguration {
    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(MyLibrary.class)
    static class InnerConfig {
        @Bean
        public MyLibraryService myService() {  // 안전하게 타입 참조
            return new MyLibraryService();
        }
    }
}

// 3. spring-boot-autoconfigure-processor 의존성 추가
//    → 컴파일 타임에 메타데이터 자동 생성
// build.gradle:
// annotationProcessor 'org.springframework.boot:spring-boot-autoconfigure-processor'
```

---

## 🤔 트레이드오프

```
클래스 로딩 없는 조건 평가:
  장점  NoClassDefFoundError 없음, 빠른 1차 필터링
  단점  Class.forName(name, false, loader) 사용 시
        클래스 파일은 읽히지만 초기화 안 됨
        → 초기화 필요한 코드는 실제 사용 전에 실행 안 됨

AutoConfigurationMetadata 사전 생성:
  장점  런타임에 @ConditionalOnClass 어노테이션 파싱 불필요
        → 시작 시간 단축 (특히 많은 Auto-configuration이 있을 때)
  단점  spring-boot-autoconfigure-processor APT 필요
        빌드 시간 증가 (미미)
        메타데이터 파일이 코드와 동기화되어야 함

내부 static @Configuration 분리:
  장점  안전한 타입 참조, 클래스 로딩 순서 문제 방지
  단점  코드 구조 복잡해짐 (중첩 클래스)
       → Spring Boot 소스 패턴을 따르면 일관성 유지 가능
```

---

## 📌 핵심 정리

```
@ConditionalOnClass 두 단계 처리

1단계 (1차 필터링 — 가장 빠름):
  AutoConfigurationMetadata에서 ConditionalOnClass 조건 읽기
  ClassLoader.getResource()로 .class 파일 존재만 확인
  → 클래스 로딩 없음, 대부분의 후보 제거

2단계 (개별 평가 — 통과한 것만):
  Class.forName(name, false, loader)로 존재 확인
  ClassNotFoundException → false (조건 미충족)

ClassNameFilter
  MISSING: 없어야 하는 클래스 목록 반환 (@ConditionalOnClass에서 사용)
  PRESENT: 있어야 하는 클래스 목록 반환 (@ConditionalOnMissingClass에서 사용)

Order: HIGHEST_PRECEDENCE + 6 (가장 먼저 평가)
  → @ConditionalOnClass false이면 @ConditionalOnBean 등 나머지 평가 생략

내부 static @Configuration 분리 패턴
  외부 타입 참조하는 @Bean → 내부 클래스로 분리
  → 클래스 로딩 순서 문제 방지
```

---

## 🤔 생각해볼 문제

**Q1.** `@ConditionalOnClass`의 `value` 속성과 `name` 속성의 차이는 무엇인가? 각각을 사용해야 하는 상황은?

**Q2.** `Class.forName(className, false, classLoader)`에서 `initialize=false`로 설정해도, 이 클래스가 다른 클래스를 참조하는 경우 그 참조 클래스도 로딩되는가?

**Q3.** 다음 Auto-configuration에서 `@ConditionalOnClass`를 클래스 레벨과 `@Bean` 메서드 레벨 양쪽에 모두 붙이는 것이 권장되는 패턴인가?

```java
@AutoConfiguration
@ConditionalOnClass(Flyway.class)
public class FlywayAutoConfiguration {

    @Bean
    @ConditionalOnClass(Flyway.class)  // 중복?
    @ConditionalOnMissingBean
    public Flyway flyway(...) { ... }
}
```

> 💡 **해설**
>
> **Q1.** `value = Class<?>[]`는 컴파일 타임에 클래스가 존재해야 한다 (IDE 자동완성 지원, 리팩터링 안전). `name = String[]`은 클래스명을 문자열로 지정하므로 컴파일 타임에 클래스가 없어도 된다. 선택 기준: 해당 JAR를 `optional` 의존성으로 추가할 수 있으면 `value` 사용(편의성), 해당 JAR 자체가 없는 환경에서도 컴파일해야 한다면 `name` 사용. 실제로 Spring Boot 내부 Auto-configuration은 대부분 해당 라이브러리를 `optional` 의존성으로 추가해 `value` 방식을 사용한다.
>
> **Q2.** `initialize=false`여도 해당 클래스의 필드, 메서드 시그니처에 등장하는 다른 클래스들은 링킹(resolution) 단계에서 로딩될 수 있다. 다만 `resolve=false`가 아닌 `initialize=false`이므로 링킹은 수행된다. 실제로는 lazy resolution이 적용되어 클래스가 실제로 사용될 때 링킹이 일어나는 경우도 있다. 이 때문에 반환 타입에 외부 라이브러리 타입이 직접 등장하는 `@Bean` 메서드를 가진 `@Configuration` 클래스는 내부 클래스로 분리하는 것이 권장된다.
>
> **Q3.** 권장되지 않는 패턴이다. 클래스 레벨 `@ConditionalOnClass(Flyway.class)`가 이미 `PARSE_CONFIGURATION` Phase에서 평가되어 Flyway가 없으면 `FlywayAutoConfiguration` 전체가 스킵된다. `@Bean` 메서드 레벨에 다시 `@ConditionalOnClass(Flyway.class)`를 붙이면 `REGISTER_BEAN` Phase에 한 번 더 평가되는데 이 중복 평가는 불필요하다. 올바른 패턴은 클래스 레벨에는 `@ConditionalOnClass`, 메서드 레벨에는 `@ConditionalOnMissingBean`처럼 역할을 분리하는 것이다. 단, 반환 타입에 `Flyway`를 직접 쓰는 경우 내부 클래스 분리 패턴을 사용하고 내부 클래스에만 `@ConditionalOnClass`를 붙인다.

---

<div align="center">

**[⬅️ 이전: Chapter 1 — Banner 출력과 Startup Logging](../startup-process/07-banner-and-startup-logging.md)** | **[홈으로 🏠](../README.md)** | **[다음: @ConditionalOnBean vs @ConditionalOnMissingBean ➡️](./02-conditional-on-bean-missing.md)**

</div>
