# @ConditionalOnBean vs @ConditionalOnMissingBean — Bean 탐색 시점의 함정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@ConditionalOnBean`과 `@ConditionalOnMissingBean`은 어느 시점에 어떤 범위에서 Bean을 탐색하는가?
- 사용자 정의 Bean이 Auto-configuration의 `@ConditionalOnMissingBean`을 통해 자동 오버라이드되는 원리는?
- `@ConditionalOnBean`을 사용자 `@Configuration`에서 쓰면 왜 위험한가?
- `type`, `name`, `annotation`, `value` 속성 간 탐색 순서와 AND/OR 관계는?
- `OnBeanCondition`이 `BeanDefinitionRegistry`를 직접 조회하는 이유는?
- `@ConditionalOnSingleCandidate`는 언제 필요한가?

---

## 🔍 왜 이게 존재하는가

### 문제: 자동 등록 Bean과 사용자 정의 Bean의 충돌

```java
// Auto-configuration이 DataSource를 자동 등록한다면
@Bean
public DataSource autoDataSource() {
    return new HikariDataSource(autoConfig);
}

// 사용자가 테스트용 DataSource를 따로 정의하면?
@Bean
public DataSource testDataSource() {
    return new EmbeddedDatabase(H2);
}

// 결과: NoUniqueBeanDefinitionException — DataSource Bean 두 개!
```

```
해결:
  Auto-configuration이 사용자 Bean 존재를 먼저 확인하고
  없을 때만 자동 Bean을 등록

  @ConditionalOnMissingBean(DataSource.class)
  → DataSource Bean이 이미 있으면 자동 등록 스킵
  → 사용자 Bean이 항상 우선

@ConditionalOnBean:
  특정 Bean이 있을 때만 등록
  → 예: ConnectionFactory Bean이 있을 때만 JMS 관련 Bean 등록
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @ConditionalOnBean을 사용자 @Configuration에서 사용한다

```java
// ❌ 위험한 패턴
@Configuration
public class MyConfig {

    @Bean
    @ConditionalOnBean(DataSource.class)  // DataSource가 있을 때만 등록
    public MyRepository myRepository(DataSource ds) {
        return new MyRepository(ds);
    }
}
```

```
왜 위험한가:
  @Configuration 처리 순서가 불확정
  
  시나리오 A (DataSourceConfig가 먼저 처리됨):
    DataSource BeanDefinition 등록됨
    → MyConfig 처리 시 @ConditionalOnBean 평가 → DataSource 있음 → 등록됨 ✅

  시나리오 B (MyConfig가 먼저 처리됨):
    @ConditionalOnBean 평가 시 DataSource 아직 없음 → 등록 안 됨 ❌
    → 이후 DataSource가 등록되어도 MyRepository는 이미 제외됨

  → 순서에 따라 결과가 달라지는 비결정적 동작
  
  올바른 해결:
    파라미터 주입 또는 @Autowired(required=false) 사용
    또는 @AutoConfigureAfter로 순서 명시 (Auto-configuration에서만 가능)
```

### Before: @ConditionalOnMissingBean은 Bean 인스턴스를 확인한다

```
❌ 잘못된 이해:
  "런타임에 실제 Bean 객체가 존재하는지 확인한다"

✅ 실제:
  BeanDefinition 등록 여부를 확인한다
  → refresh() → invokeBeanFactoryPostProcessors() 단계
  → 실제 Bean 인스턴스는 아직 생성 전 (finishBeanFactoryInitialization 이전)
  → BeanDefinitionRegistry.getBeanDefinitionNames() 등으로 정의 유무 파악
```

---

## ✨ 올바른 이해와 사용

### After: 탐색 범위와 평가 시점의 정확한 이해

```
@ConditionalOnMissingBean 탐색 대상:
  현재까지 등록된 모든 BeanDefinition
  (REGISTER_BEAN Phase → 사용자 @Configuration 처리 완료 후 Auto-configuration 처리 시)

탐색 기준 (기본: 현재 @Bean 메서드의 반환 타입):
  type      → 지정 타입과 일치하거나 서브타입인 BeanDefinition
  name      → 지정 이름의 BeanDefinition
  annotation → 지정 어노테이션이 붙은 Bean 클래스의 BeanDefinition
  value     → type의 alias (기본값)

AND 조건:
  지정된 모든 조건을 동시에 만족하는 Bean이 없어야 함
```

---

## 🔬 내부 동작 원리

### 1. OnBeanCondition — BeanDefinitionRegistry 직접 탐색

```java
// OnBeanCondition.java
@Order(Ordered.LOWEST_PRECEDENCE - 8)
class OnBeanCondition extends FilteringSpringBootCondition
        implements ConfigurationCondition {

    @Override
    public ConfigurationPhase getConfigurationPhase() {
        return ConfigurationPhase.REGISTER_BEAN;  // ← @Bean 메서드 처리 시 평가
    }

    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context,
                                             AnnotatedTypeMetadata metadata) {

        ConditionMessage matchMessage = ConditionMessage.empty();
        MergedAnnotations annotations = MergedAnnotations.from(metadata);

        // @ConditionalOnBean 처리
        if (metadata.isAnnotated(ConditionalOnBean.class.getName())) {
            BeanSearchSpec spec = new BeanSearchSpec(context, metadata, ConditionalOnBean.class);
            MatchResult matchResult = getMatchingBeans(context, spec);

            if (!matchResult.isAllMatched()) {
                // 필요한 Bean이 없음 → 조건 false
                String reason = createOnBeanNoMatchReason(matchResult);
                return ConditionOutcome.noMatch(
                    ConditionMessage.forCondition(ConditionalOnBean.class).because(reason));
            }
            matchMessage = ...;
        }

        // @ConditionalOnMissingBean 처리
        if (metadata.isAnnotated(ConditionalOnMissingBean.class.getName())) {
            BeanSearchSpec spec = new BeanSearchSpec(context, metadata, ConditionalOnMissingBean.class);
            MatchResult matchResult = getMatchingBeans(context, spec);

            if (matchResult.isAnyMatched()) {
                // 이미 Bean이 있음 → 조건 false (등록 안 함)
                String reason = createOnMissingBeanNoMatchReason(matchResult);
                return ConditionOutcome.noMatch(
                    ConditionMessage.forCondition(ConditionalOnMissingBean.class).because(reason));
            }
            matchMessage = ...;
        }

        return ConditionOutcome.match(matchMessage);
    }
}
```

### 2. getMatchingBeans() — 실제 Bean 탐색

```java
// OnBeanCondition.getMatchingBeans()
private MatchResult getMatchingBeans(ConditionContext context, Spec<?> spec) {

    ClassLoader classLoader = context.getClassLoader();
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();

    // 현재 Bean을 포함할지 여부 (자기 자신은 제외)
    boolean considerHierarchy = spec.getStrategy() != SearchStrategy.CURRENT;

    Set<String> beanNames = new HashSet<>();

    // ① 타입 기반 탐색
    for (String type : spec.getTypes()) {
        beanNames.addAll(getBeanNamesForType(classLoader, considerHierarchy,
                                              beanFactory, type));
    }

    // ② 어노테이션 기반 탐색
    for (String annotation : spec.getAnnotations()) {
        beanNames.addAll(getBeanNamesForAnnotation(classLoader, beanFactory,
                                                    annotation, considerHierarchy));
    }

    // ③ 이름 기반 탐색
    for (String beanName : spec.getNames()) {
        if (beanFactory.containsBeanDefinition(beanName)) {
            beanNames.add(beanName);
        }
    }

    return matchResult(beanNames, spec);
}

// getBeanNamesForType() — BeanDefinition에서 타입 탐색
private Collection<String> getBeanNamesForType(ClassLoader classLoader,
                                                 boolean considerHierarchy,
                                                 ListableBeanFactory beanFactory,
                                                 String type) {
    try {
        // Class.forName으로 타입 로딩 (이 시점에는 필요)
        return BeanTypeRegistry.get(beanFactory)
            .getNamesForType(resolve(type, classLoader), TypeMatcher.isAssignableTo);
        // → BeanDefinition의 beanClassName과 타입 계층 비교
        // → 실제 Bean 인스턴스 없이 BeanDefinition만으로 탐색
    }
    catch (ClassNotFoundException | NoClassDefFoundError ex) {
        return Collections.emptySet();
    }
}
```

```
BeanDefinition 기반 탐색의 의미:

BeanDefinition에는 beanClass 또는 beanClassName이 있음
→ BeanTypeRegistry가 beanClass로 타입 계층 분석
→ Class.isAssignableFrom()으로 서브타입 관계 확인

실제 예시:
  사용자 Bean:
    @Bean HikariDataSource myDs() { ... }
    → BeanDefinition: beanClassName = "com.zaxxer.hikari.HikariDataSource"

  Auto-configuration 조건:
    @ConditionalOnMissingBean(DataSource.class)
    → HikariDataSource extends DataSource 확인
    → DataSource 서브타입이 이미 있음 → 조건 false → 자동 등록 스킵
```

### 3. BeanSearchSpec — 탐색 기준 구성

```java
// BeanSearchSpec 내부 — 여러 속성 파싱
static class BeanSearchSpec {
    private final List<String> names;         // name 속성
    private final List<String> types;         // type/value 속성 (클래스명)
    private final List<String> annotations;  // annotation 속성
    private final List<String> ignoredTypes; // ignored 속성
    private final SearchStrategy strategy;   // CURRENT, ANCESTORS, ALL

    BeanSearchSpec(ConditionContext context, AnnotatedTypeMetadata metadata,
                   Class<?> annotationType) {
        MergedAnnotation<?> annotation = metadata.getAnnotations()
            .get(annotationType.getName());

        // value와 type은 동일 (value가 type의 alias)
        Set<String> types = new LinkedHashSet<>();
        collectTypes(annotation, types, "value", metadata);
        collectTypes(annotation, types, "type", metadata);

        // @Bean 메서드의 반환 타입이 기본값 (value/type 미지정 시)
        if (types.isEmpty() && names.isEmpty()) {
            addDeducedBeanType(context, metadata, types);
        }
        this.types = List.copyOf(types);
        // ...
    }

    // 반환 타입 자동 추론
    private void addDeducedBeanType(ConditionContext context,
                                     AnnotatedTypeMetadata metadata,
                                     Set<String> beanTypes) {
        if (metadata instanceof MethodMetadata methodMetadata) {
            // @Bean 메서드의 반환 타입 추론
            beanTypes.add(methodMetadata.getReturnTypeName());
        }
    }
}
```

```
속성 미지정 시 기본값:

@Bean
@ConditionalOnMissingBean  // type/name 미지정
public DataSource dataSource() { ... }

→ 반환 타입 DataSource를 자동으로 탐색 대상으로 사용
→ DataSource 또는 그 서브타입 Bean이 이미 있으면 → 등록 안 함

명시적 지정:
@ConditionalOnMissingBean(DataSource.class)
→ 동일 효과 (명시적 버전)

@ConditionalOnMissingBean(name = "mySpecialDataSource")
→ "mySpecialDataSource"라는 이름의 Bean이 없을 때만 등록
```

### 4. SearchStrategy — 탐색 범위

```java
// SearchStrategy
public enum SearchStrategy {
    CURRENT,    // 현재 ApplicationContext에서만
    ANCESTORS,  // 부모 Context에서만 (현재 제외)
    ALL         // 현재 + 부모 모두 (기본값)
}

// 사용 예시
@ConditionalOnMissingBean(value = DataSource.class,
                           search = SearchStrategy.CURRENT)
// → 현재 Context에만 DataSource 없을 때
// → 부모 Context의 DataSource는 무시
```

### 5. @ConditionalOnSingleCandidate

```java
// @ConditionalOnSingleCandidate — 단일 후보 보장
// 해당 타입의 Bean이 정확히 1개이거나, 여러 개라도 @Primary가 하나일 때만 조건 충족

@Bean
@ConditionalOnSingleCandidate(DataSource.class)
public JdbcTemplate jdbcTemplate(DataSource dataSource) {
    return new JdbcTemplate(dataSource);
}

// 조건 충족 케이스:
//   DataSource Bean 1개 → 충족
//   DataSource Bean 2개 + @Primary 1개 → 충족 (@Primary가 주입됨)

// 조건 미충족 케이스:
//   DataSource Bean 0개 → 미충족
//   DataSource Bean 2개 + @Primary 없음 → 미충족 (어느 걸 쓸지 모름)

// 사용 이유:
//   JdbcTemplate은 DataSource가 하나로 특정될 때만 자동 생성 의미 있음
//   여러 DataSource가 있으면 어떤 것과 연결할지 자동 결정 불가
```

---

## 💻 실험으로 확인하기

### 실험 1: @ConditionalOnMissingBean 오버라이드 동작

```java
// 사용자 커스텀 DataSource 정의
@Configuration
public class MyDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

// 결과 확인
@Component
public class DataSourceChecker {
    @Autowired
    DataSource dataSource;

    @PostConstruct
    public void check() {
        System.out.println("DataSource 타입: " + dataSource.getClass().getSimpleName());
        // 출력: EmbeddedDatabase (사용자 정의가 우선)
        // HikariDataSource가 아님 (Auto-configuration 스킵됨)
    }
}
```

```
--debug 출력으로 확인:
DataSourceAutoConfiguration:
   Did not match:
      - @ConditionalOnMissingBean (types: javax.sql.DataSource)
        found bean 'dataSource' (OnBeanCondition)
```

### 실험 2: @ConditionalOnBean 순서 함정 재현

```java
@Configuration
@Import({ConfigA.class, ConfigB.class})
public class TestConfig { }

@Configuration
class ConfigA {
    @Bean
    @ConditionalOnBean(ServiceB.class)  // 순서 함정!
    public ServiceA serviceA() {
        System.out.println("ServiceA 등록됨");
        return new ServiceA();
    }
}

@Configuration
class ConfigB {
    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}

// ConfigA가 먼저 처리되면 ServiceA 미등록
// ConfigB가 먼저 처리되면 ServiceA 등록
// → 재현 불가능한 순서 의존 버그
```

### 실험 3: @ConditionalOnSingleCandidate 동작

```java
@Configuration
public class MultiDataSourceConfig {
    @Bean
    @Primary
    public DataSource primaryDs() { ... }

    @Bean
    public DataSource secondaryDs() { ... }
}

// 결과:
// @ConditionalOnSingleCandidate(DataSource.class) → 충족
//   (DataSource가 2개지만 @Primary가 1개이므로)
// JdbcTemplate 자동 생성됨 (primaryDs 사용)
```

---

## ⚙️ 설정 최적화 팁

```java
// 1. 커스텀 Auto-configuration에서 @ConditionalOnMissingBean 기본 패턴
@Bean
@ConditionalOnMissingBean  // 반환 타입으로 자동 추론
public MyService myService() {
    return new MyService();
}
// 사용자가 MyService @Bean을 정의하면 자동으로 이 Bean 스킵

// 2. 여러 타입 동시 체크
@ConditionalOnMissingBean({DataSource.class, EmbeddedDatabase.class})
// DataSource AND EmbeddedDatabase 둘 다 없을 때만 등록
// → 두 조건 모두 만족해야 함 (AND 조건)

// 3. 이름 기반 — 특정 인프라 Bean 존재 확인
@ConditionalOnBean(name = "entityManagerFactory")
public TransactionManager txManager(EntityManagerFactory emf) { ... }
// JPA가 설정됐을 때만 트랜잭션 관리자 등록

// 4. @ConditionalOnMissingBean + ignored
@ConditionalOnMissingBean(
    value = DataSource.class,
    ignored = EmbeddedDatabaseAutoConfiguration.class  // 이 클래스에서 등록한 것은 무시
)
```

---

## 🤔 트레이드오프

```
@ConditionalOnMissingBean:
  장점  사용자 Bean이 자동으로 Auto-configuration을 오버라이드
        명시적 exclude 없이 커스터마이징 가능
  단점  반환 타입 추론이 의도와 다르게 동작할 수 있음
        예: 인터페이스 반환 타입 vs 구현체 반환 타입

@ConditionalOnBean 사용자 코드에서:
  위험  @Configuration 처리 순서가 보장되지 않음
  대안  파라미터 주입으로 의존성 명시 (필요 Bean 없으면 시작 실패 → 명확한 오류)
       @Autowired(required=false) + null 체크
       @ConditionalOnBean은 Auto-configuration에서만 @AutoConfigureAfter와 함께 사용

SearchStrategy:
  ALL (기본값)  부모 Context까지 탐색 → Spring MVC Root/Servlet Context 구조에서 주의
  CURRENT       현재 Context만 → 테스트에서 Context 격리 시 유용
```

---

## 📌 핵심 정리

```
평가 시점
  REGISTER_BEAN Phase
  DeferredImportSelector 이후 → 사용자 Bean 등록 완료 후

탐색 대상
  BeanDefinition (인스턴스 아님)
  → BeanTypeRegistry로 타입 계층 분석

@ConditionalOnMissingBean 기본 동작
  타입 미지정 → @Bean 반환 타입 자동 추론
  서브타입 포함해서 탐색
  → HikariDataSource도 DataSource로 감지

@ConditionalOnBean 안전한 사용
  Auto-configuration에서 @AutoConfigureAfter와 함께 사용
  사용자 @Configuration에서는 사용 자제

@ConditionalOnSingleCandidate
  타입 Bean 1개 또는 @Primary 1개일 때만 충족
  JdbcTemplate, JmsTemplate 등 단일 인프라 Bean에 의존하는 경우 사용
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 `myService` Bean이 등록되는가?

```java
@Configuration
class LibraryAutoConfig {
    @Bean
    @ConditionalOnMissingBean
    public MyService myService() { return new MyServiceImpl(); }
}

@Configuration
class UserConfig {
    @Bean
    public MyServiceImpl myServiceImpl() { return new MyServiceImpl(); }
    // 주의: 반환 타입이 MyServiceImpl (MyService의 서브타입)
}
```

**Q2.** `@ConditionalOnMissingBean(ignored = SomeConfig.class)`의 `ignored` 속성은 어떤 경우에 필요한가?

**Q3.** `@ConditionalOnBean(type = "com.example.MyService")`와 `@ConditionalOnBean(MyService.class)`의 평가 결과가 다를 수 있는 경우는?

> 💡 **해설**
>
> **Q1.** `myService` Bean은 등록되지 않는다. `@ConditionalOnMissingBean`(타입 미지정)은 `myService` 메서드의 반환 타입 `MyService`를 기준으로 탐색한다. `UserConfig`에서 등록한 `myServiceImpl` Bean의 실제 타입은 `MyServiceImpl`이지만, `MyServiceImpl extends MyService`이므로 `MyService` 서브타입으로 감지된다. `OnBeanCondition`이 `BeanTypeRegistry.getNamesForType(MyService.class, isAssignableTo)`를 실행하면 `myServiceImpl`이 결과에 포함된다. 따라서 조건이 false가 되어 `myService` Bean은 등록되지 않는다.
>
> **Q2.** `ignored`는 특정 Auto-configuration이 등록한 Bean을 "없는 것처럼" 취급하게 한다. 예를 들어 `EmbeddedDatabaseAutoConfiguration`이 테스트용 H2 DataSource를 자동으로 등록할 수 있는데, 이 Auto-configuration의 DataSource가 있어도 내 Auto-configuration의 DataSource를 등록해야 하는 경우에 사용한다. 실제 사례로 `DataSourceAutoConfiguration`이 `EmbeddedDatabaseAutoConfiguration`의 DataSource를 무시하고 독립적으로 DataSource를 등록할 수 있도록 하는 패턴이 있다.
>
> **Q3.** `type = "com.example.MyService"` (String)과 `MyService.class`는 대부분 동일하게 동작한다. 다를 수 있는 경우는 `ClassLoader` 불일치 상황이다. 두 개의 `ClassLoader`가 각각 `MyService`를 로딩한 경우, `MyService.class` 리터럴은 현재 `ClassLoader`의 것이고 문자열로 지정한 경우 `BeanFactory`의 `ClassLoader`로 로딩된다. 이 둘이 다른 `ClassLoader`에 의해 로딩된 클래스면 `Class.isAssignableFrom()`이 false를 반환할 수 있다. 실용적으로는 같은 `ClassLoader`를 쓰는 일반적인 Spring Boot 앱에서 차이가 없지만, 플러그인 시스템이나 OSGi 환경에서는 주의가 필요하다.

---

<div align="center">

**[⬅️ 이전: @ConditionalOnClass 동작 메커니즘](./01-conditional-on-class.md)** | **[홈으로 🏠](../README.md)** | **[다음: @AutoConfigureAfter/@AutoConfigureBefore 순서 제어 ➡️](./03-autoconfigure-order.md)**

</div>
