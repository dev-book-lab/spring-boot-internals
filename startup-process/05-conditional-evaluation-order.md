# @Conditional 어노테이션 평가 순서 — ConditionEvaluator 완전 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Conditional`이 평가되는 시점은 Bean 생성 시점인가, BeanDefinition 등록 시점인가?
- `ConfigurationPhase.PARSE_CONFIGURATION`과 `REGISTER_BEAN`의 차이는 무엇인가?
- Auto-configuration의 `@ConditionalOnMissingBean`이 사용자 Bean 이후에 평가되는 보장은 어디서 오는가?
- `@Conditional` 조건 평가 중 다른 Bean의 존재를 확인하면 왜 위험한가?
- 여러 `@Conditional`이 겹쳐 있을 때 평가 순서는?
- `@Profile`은 내부적으로 `@Conditional`과 어떻게 연결되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: Bean 등록 여부를 런타임 조건에 따라 결정해야 한다

```java
// 조건이 없으면 — 항상 DataSource Bean 등록
@Bean
public DataSource dataSource() {
    return new HikariDataSource();
}
// → MySQL 드라이버가 없어도 등록 시도 → ClassNotFoundException
// → 사용자가 이미 DataSource를 정의했어도 중복 등록 → 충돌
```

```
필요한 것:
  "이 Bean은 다음 조건이 모두 만족될 때만 등록해라"
  
  조건 예시:
    클래스패스에 특정 클래스 존재    → @ConditionalOnClass
    특정 Bean이 없을 때만            → @ConditionalOnMissingBean
    특정 프로퍼티가 설정됐을 때      → @ConditionalOnProperty
    웹 환경일 때                     → @ConditionalOnWebApplication

  → @Conditional(MyCondition.class)로 임의의 조건 구현 가능
  → Spring Boot가 자주 쓰는 패턴을 @ConditionalOnXxx로 제공
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Conditional은 Bean 생성 시점에 평가된다

```
❌ 잘못된 이해:
  "getBean() 호출 시 조건을 평가한다"

✅ 실제:
  @Conditional은 BeanDefinition 등록 시점에 평가됨
  → refresh() 내 invokeBeanFactoryPostProcessors() 단계
  → ConfigurationClassParser가 @Configuration 클래스 파싱 중 평가
  → 조건 false → BeanDefinition 자체가 등록되지 않음
  → getBean() 시점에는 이미 결정이 끝나 있음
```

### Before: 같은 클래스에 붙은 @Conditional은 순서가 없다

```java
// ❌ 잘못된 이해
@Configuration
@ConditionalOnClass(DataSource.class)       // 어느 것이 먼저?
@ConditionalOnProperty("my.db.enabled")
@ConditionalOnMissingBean(DataSource.class)
public class MyDataSourceConfig { }

// ✅ 실제:
// 같은 대상에 여러 @Conditional → 모두 AND 조건 (하나라도 false면 등록 안 됨)
// 평가 순서: @ConditionalOnClass가 가장 먼저 (1차 필터링 + PARSE_CONFIGURATION Phase)
//             → 클래스 없으면 나머지 평가 생략
```

---

## ✨ 올바른 이해와 사용

### After: 평가 시점 두 단계

```
Phase 1: PARSE_CONFIGURATION
  ConfigurationClassParser가 @Configuration 클래스를 파싱할 때
  → 클래스 레벨 @Conditional 평가
  → 조건 false → 이 @Configuration 클래스 전체 스킵
  → 내부의 모든 @Bean, @Import 무시

Phase 2: REGISTER_BEAN
  @Bean 메서드를 BeanDefinitionRegistry에 등록할 때
  → 메서드 레벨 @Conditional 평가
  → 조건 false → 해당 @Bean 메서드만 스킵
  → @Configuration 클래스의 다른 @Bean은 영향 없음

ConfigurationPhase를 구현하는 Condition:
  Auto-configuration의 @ConditionalOnMissingBean → REGISTER_BEAN Phase
  → 다른 모든 @Configuration이 파싱된 후 평가 가능
```

---

## 🔬 내부 동작 원리

### 1. Condition 인터페이스와 ConfigurationCondition

```java
// 기본 Condition 인터페이스
@FunctionalInterface
public interface Condition {
    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}

// ConfigurationCondition — Phase 지정 가능
public interface ConfigurationCondition extends Condition {
    ConfigurationPhase getConfigurationPhase();

    enum ConfigurationPhase {
        PARSE_CONFIGURATION,  // @Configuration 파싱 중 평가
        REGISTER_BEAN         // @Bean 등록 중 평가
    }
}
```

```java
// 사용 예시: Phase를 직접 지정하는 커스텀 Condition
public class OnProductionEnvironmentCondition implements ConfigurationCondition {

    @Override
    public ConfigurationPhase getConfigurationPhase() {
        return ConfigurationPhase.PARSE_CONFIGURATION;  // 클래스 레벨 평가
    }

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String profile = context.getEnvironment().getActiveProfiles()[0];
        return "prod".equals(profile);
    }
}
```

### 2. ConditionEvaluator — 평가 로직

```java
// ConditionEvaluator.java
class ConditionEvaluator {

    // BeanDefinition 등록 여부 결정
    public boolean shouldSkip(AnnotatedTypeMetadata metadata, ConfigurationPhase phase) {
        if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
            return false;  // @Conditional 없으면 스킵 안 함
        }

        if (phase == null) {
            // Phase 미지정 → @Configuration이면 PARSE_CONFIGURATION, 아니면 REGISTER_BEAN
            if (metadata instanceof AnnotationMetadata am
                    && ConfigurationClassUtils.isConfigurationCandidate(am)) {
                return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
            }
            return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
        }

        // @Conditional에 등록된 모든 Condition 수집
        List<Condition> conditions = new ArrayList<>();
        for (String[] conditionClasses : getConditionClasses(metadata)) {
            for (String conditionClass : conditionClasses) {
                Condition condition = getCondition(conditionClass, this.context.getClassLoader());
                conditions.add(condition);
            }
        }

        // Ordered/PriorityOrdered로 정렬
        AnnotationAwareOrderComparator.sort(conditions);

        // 모든 Condition 평가
        for (Condition condition : conditions) {
            ConfigurationPhase requiredPhase = null;
            if (condition instanceof ConfigurationCondition cc) {
                requiredPhase = cc.getConfigurationPhase();
            }

            // Phase가 다르면 지금 평가 안 함 (나중에 평가)
            if ((requiredPhase == null || requiredPhase == phase)
                    && !condition.matches(this.context, metadata)) {
                return true;  // 조건 false → 스킵
            }
        }
        return false;  // 모든 조건 통과 → 등록
    }
}
```

### 3. @ConditionalOnClass — PARSE_CONFIGURATION Phase

```java
// OnClassCondition.java
@Order(Ordered.HIGHEST_PRECEDENCE + 6)  // 가장 먼저 평가 (높은 우선순위)
class OnClassCondition extends FilteringSpringBootCondition {

    @Override
    public ConfigurationPhase getConfigurationPhase() {
        return ConfigurationPhase.PARSE_CONFIGURATION;
    }

    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context,
                                             AnnotatedTypeMetadata metadata) {
        ClassLoader classLoader = context.getClassLoader();

        // @ConditionalOnClass의 value 속성 (클래스 배열)
        List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
        // @ConditionalOnMissingClass의 value 속성
        List<String> onMissingClasses = getCandidates(metadata, ConditionalOnMissingClass.class);

        ConditionMessage matchMessage = ConditionMessage.empty();

        if (onClasses != null) {
            // ClassNameFilter.MISSING: 클래스가 없는 것들만 반환
            List<String> missing = filter(onClasses, ClassNameFilter.MISSING, classLoader);
            if (!missing.isEmpty()) {
                // 클래스가 없으면 조건 실패
                return ConditionOutcome.noMatch(
                    ConditionMessage.forCondition(ConditionalOnClass.class)
                        .didNotFind("required class", "required classes").items(missing));
            }
            matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
                .found("required class", "required classes").items(onClasses);
        }

        return ConditionOutcome.match(matchMessage);
    }
}
```

```
ClassNameFilter — 클래스 로딩 최소화
  MISSING:
    ClassUtils.isPresent(className, classLoader)
    → ClassLoader.loadClass() 대신 ClassLoader.getResource() 사용
    → 클래스 파일 존재 여부만 확인, 실제 Class 객체 생성 안 함
    → ClassNotFoundException 없이 빠른 체크 가능
```

### 4. @ConditionalOnMissingBean — REGISTER_BEAN Phase

```java
// OnBeanCondition.java
@Order(Ordered.LOWEST_PRECEDENCE - 8)  // 늦게 평가 (낮은 우선순위)
class OnBeanCondition extends FilteringSpringBootCondition
        implements ConfigurationCondition {

    @Override
    public ConfigurationPhase getConfigurationPhase() {
        return ConfigurationPhase.REGISTER_BEAN;  // ← 핵심
    }

    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context,
                                             AnnotatedTypeMetadata metadata) {
        BeanFactory beanFactory = context.getBeanFactory();
        // 현재까지 등록된 BeanDefinition에서 타입/이름으로 Bean 존재 여부 확인
        // → REGISTER_BEAN Phase이므로 사용자 @Configuration이 먼저 처리됨
        // → 사용자 DataSource Bean이 이미 등록되어 있음
        Set<String> matchingBeans = getBeans(beanFactory, metadata, ...);
        // matchingBeans가 비어있어야 @ConditionalOnMissingBean 조건 충족
        ...
    }
}
```

```
@ConditionalOnMissingBean이 정확하게 동작하는 이유:

처리 순서:
  1. 사용자 @Configuration 클래스 파싱 (PARSE_CONFIGURATION)
     → UserConfig.dataSource() @Bean → DataSource BeanDefinition 등록
  
  2. DeferredImportSelector 처리 (Auto-configuration)
     → DataSourceAutoConfiguration @Bean 처리
     → @ConditionalOnMissingBean(DataSource.class) 평가 (REGISTER_BEAN)
     → BeanDefinitionRegistry에 DataSource가 이미 있음 → 조건 false
     → DataSourceAutoConfiguration의 @Bean 스킵

만약 @ConditionalOnMissingBean이 PARSE_CONFIGURATION Phase였다면:
  Auto-configuration 클래스가 사용자 Config보다 먼저 파싱될 경우
  → 사용자 DataSource가 아직 등록 전 → 없다고 판단 → Auto DataSource 등록
  → 이후 사용자 DataSource도 등록 → 충돌
```

### 5. @Profile의 내부 구현

```java
// @Profile — @Conditional로 구현됨
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(ProfileCondition.class)  // ← @Conditional 활용
public @interface Profile {
    String[] value();
}

// ProfileCondition.java
class ProfileCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        MultiValueMap<String, Object> attrs =
            metadata.getAllAnnotationAttributes(Profile.class.getName());
        if (attrs != null) {
            for (Object value : attrs.get("value")) {
                if (context.getEnvironment().acceptsProfiles(
                        Profiles.of((String[]) value))) {
                    return true;
                }
            }
            return false;
        }
        return true;
    }
}
```

```
@Profile("prod") 동작:
  @Conditional(ProfileCondition.class)로 변환
  → ConditionEvaluator.shouldSkip() 호출
  → ProfileCondition.matches()
  → Environment.acceptsProfiles("prod")
  → spring.profiles.active에 "prod" 없으면 false → BeanDefinition 스킵
```

### 6. 여러 @Conditional 겹침 — AND 처리

```java
// 여러 @Conditional이 붙은 경우
@Configuration
@ConditionalOnClass(DataSource.class)         // 조건 1
@ConditionalOnProperty("my.datasource.url")  // 조건 2
@ConditionalOnMissingBean(DataSource.class)  // 조건 3
public class MyDataSourceAutoConfiguration { }
```

```
평가 순서와 처리:

1. @ConditionalOnClass (PARSE_CONFIGURATION, 높은 우선순위)
   → DataSource.class가 classpath에 없으면 즉시 false → 나머지 평가 생략

2. @ConditionalOnProperty (PARSE_CONFIGURATION)
   → my.datasource.url 프로퍼티가 없으면 false → 3번 평가 생략

3. @ConditionalOnMissingBean (REGISTER_BEAN)
   → 이미 DataSource Bean이 있으면 false

세 조건 모두 true일 때만 BeanDefinition 등록

단락 평가:
  앞의 조건이 false이면 뒤 조건 평가 안 함
  → @ConditionalOnClass를 먼저 배치 → 불필요한 클래스 로딩 방지
```

---

## 💻 실험으로 확인하기

### 실험 1: Phase 차이 직접 확인

```java
// PARSE_CONFIGURATION Phase Condition
public class AlwaysFalseParseCondition implements ConfigurationCondition {
    @Override
    public ConfigurationPhase getConfigurationPhase() {
        return ConfigurationPhase.PARSE_CONFIGURATION;
    }
    @Override
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata meta) {
        return false;  // 항상 false
    }
}

@Configuration
@Conditional(AlwaysFalseParseCondition.class)
public class MyConfig {
    @Bean public ServiceA serviceA() { return new ServiceA(); }
    @Bean public ServiceB serviceB() { return new ServiceB(); }
}

// 결과: ServiceA, ServiceB 모두 등록 안 됨 (Config 전체 스킵)
```

```java
// REGISTER_BEAN Phase Condition — @Bean 메서드 레벨에서만 작동
public class AlwaysFalseBeanCondition implements ConfigurationCondition {
    @Override
    public ConfigurationPhase getConfigurationPhase() {
        return ConfigurationPhase.REGISTER_BEAN;
    }
    @Override
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata meta) {
        return false;
    }
}

@Configuration
public class MyConfig {
    @Bean
    @Conditional(AlwaysFalseBeanCondition.class)
    public ServiceA serviceA() { return new ServiceA(); }  // 등록 안 됨

    @Bean
    public ServiceB serviceB() { return new ServiceB(); }  // 정상 등록됨
}
```

### 실험 2: @ConditionalOnMissingBean 우선순위 테스트

```java
// 사용자 설정
@Configuration
public class UserConfig {
    @Bean
    public DataSource myDataSource() {
        return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.H2).build();
    }
}

// Auto-configuration이 적용되는지 확인
// → application.properties에 spring.datasource.url 없어도
//   UserConfig.myDataSource()가 우선
// → DataSourceAutoConfiguration의 @Bean 스킵됨

// --debug로 확인:
// DataSourceAutoConfiguration:
//   Did not match:
//     - @ConditionalOnMissingBean (types: javax.sql.DataSource)
//       found bean 'myDataSource' (OnBeanCondition)
```

### 실험 3: 커스텀 Condition 작성

```java
// OS 타입에 따라 Bean 등록 여부 결정
public class OnLinuxCondition implements ConfigurationCondition {

    @Override
    public ConfigurationPhase getConfigurationPhase() {
        return ConfigurationPhase.PARSE_CONFIGURATION;
    }

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return System.getProperty("os.name").toLowerCase().contains("linux");
    }
}

@Configuration
@Conditional(OnLinuxCondition.class)
public class LinuxOnlyConfig {
    @Bean
    public MonitoringService linuxMonitoring() {
        return new LinuxMonitoringService();
    }
}
```

---

## ⚙️ 설정 최적화 팁

```java
// 1. @ConditionalOnClass를 클래스 레벨에 배치 — 1차 필터링 활용
@AutoConfiguration
@ConditionalOnClass(HikariDataSource.class)  // 먼저 배치
@ConditionalOnMissingBean(DataSource.class)  // 나중에 평가
public class HikariAutoConfiguration { }

// 2. 복잡한 조건은 별도 Condition 클래스로 분리
// → 재사용 가능, 테스트 용이

// 3. @ConditionalOnProperty matchIfMissing 활용
@ConditionalOnProperty(
    name = "my.feature.enabled",
    havingValue = "true",
    matchIfMissing = true  // 프로퍼티 없을 때 기본값 = 조건 true
)
```

---

## 🤔 트레이드오프

```
PARSE_CONFIGURATION Phase:
  장점  @Configuration 전체를 조기에 스킵 → 내부 @Bean 파싱 비용 절약
  단점  다른 Bean의 존재 여부를 확인하면 부정확할 수 있음
        (아직 모든 @Configuration 처리 전)

REGISTER_BEAN Phase:
  장점  모든 사용자 Bean 등록 후 평가 → @ConditionalOnMissingBean 정확
  단점  @Configuration 클래스 파싱 비용은 이미 발생

@ConditionalOnBean 주의점:
  @Configuration 클래스가 처리된 순서에 따라 결과가 달라질 수 있음
  Auto-configuration끼리의 순서는 @AutoConfigureAfter로 명시 필요
  사용자 @Configuration에서 @ConditionalOnBean 사용 시
  → 의존하는 Bean의 @Configuration이 먼저 처리됐는지 보장 어려움
```

---

## 📌 핵심 정리

```
@Conditional 평가 시점
  BeanDefinition 등록 시 (Bean 생성 시점 아님)
  조건 false → BeanDefinition 자체 미등록

ConfigurationPhase
  PARSE_CONFIGURATION  @Configuration 파싱 중 → 클래스 전체 스킵 가능
  REGISTER_BEAN        @Bean 메서드 등록 중 → 해당 @Bean만 스킵

평가 우선순위 (Order 기반)
  @ConditionalOnClass    최우선 (HIGHEST_PRECEDENCE + 6)
  @ConditionalOnMissingBean  나중 (LOWEST_PRECEDENCE - 8)

@ConditionalOnMissingBean 정확성 보장
  DeferredImportSelector → 사용자 Config 모두 처리 후 Auto-configuration 처리
  REGISTER_BEAN Phase → 사용자 Bean 등록 후 평가

@Profile 내부
  @Conditional(ProfileCondition.class)로 구현
  → 일반 @Conditional과 동일한 평가 경로

여러 @Conditional → AND 조건 + 단락 평가
  앞 조건 false → 뒤 조건 평가 생략
  @ConditionalOnClass를 먼저 → 불필요한 클래스 로딩 방지
```

---

## 🤔 생각해볼 문제

**Q1.** `@ConditionalOnBean(DataSource.class)`를 사용자 `@Configuration` 클래스에 붙이면 왜 위험한가? Auto-configuration에서는 괜찮고 사용자 코드에서는 위험한 이유는?

**Q2.** 다음 두 코드는 동작이 같은가, 다른가?

```java
// 코드 A
@Configuration
@ConditionalOnClass(Flyway.class)
public class FlywayConfig {
    @Bean
    @ConditionalOnMissingBean
    public Flyway flyway() { return new Flyway(); }
}

// 코드 B
@Configuration
public class FlywayConfig {
    @Bean
    @ConditionalOnClass(Flyway.class)
    @ConditionalOnMissingBean
    public Flyway flyway() { return new Flyway(); }
}
```

**Q3.** `ConditionEvaluator`가 `Condition` 구현체를 `Ordered`로 정렬하는 이유는 무엇인가? 정렬하지 않으면 어떤 문제가 생길 수 있는가?

> 💡 **해설**
>
> **Q1.** 사용자 `@Configuration` 클래스들은 처리 순서가 보장되지 않는다. `@ConditionalOnBean(DataSource.class)`를 사용자 Config에 붙이면, 의존하는 `DataSource` Bean을 정의한 다른 `@Configuration`이 먼저 처리됐는지 알 수 없다. 처리 순서에 따라 조건이 맞기도 하고 틀리기도 하는 비결정적 동작이 발생한다. Auto-configuration에서는 `@AutoConfigureAfter(DataSourceAutoConfiguration.class)`로 `DataSource`가 먼저 처리됨을 명시할 수 있고, `DeferredImportSelector`가 Auto-configuration끼리의 순서를 이 어노테이션을 참고해 위상 정렬하므로 안전하다.
>
> **Q2.** 동작이 다르다. 코드 A에서 `@ConditionalOnClass(Flyway.class)`는 `PARSE_CONFIGURATION` Phase에 `FlywayConfig` 클래스 전체에 적용된다. Flyway 클래스가 없으면 `FlywayConfig` 자체가 파싱되지 않으므로 내부의 `@Bean` 메서드도 처리되지 않는다. 코드 B에서는 `@ConditionalOnClass(Flyway.class)`가 `@Bean` 메서드 레벨에 적용되어 `REGISTER_BEAN` Phase에 평가된다. 그런데 `Flyway` 타입이 반환 타입으로 선언되어 있어 클래스가 없으면 `FlywayConfig` 파싱 자체에서 `ClassNotFoundException`이 발생할 수 있다. 따라서 코드 A가 올바른 방식이다.
>
> **Q3.** 여러 `Condition` 구현체가 서로 의존 관계를 가질 수 있기 때문이다. 예를 들어 `@ConditionalOnClass`가 `false`이면 해당 클래스를 사용하는 `@ConditionalOnBean`은 클래스 자체가 없어 `ClassNotFoundException`을 일으킬 수 있다. `@ConditionalOnClass`를 먼저 평가해 `false`이면 나머지를 생략하는 단락 평가가 가능하도록 `Order` 값을 `HIGHEST_PRECEDENCE`에 가깝게 설정한다. 정렬하지 않으면 `@ConditionalOnBean`이 먼저 실행되어 존재하지 않는 클래스를 타입으로 조회하다가 예외가 발생할 수 있다.

---

<div align="center">

**[⬅️ 이전: spring.factories vs @AutoConfiguration](./04-spring-factories-vs-autoconfiguration.md)** | **[홈으로 🏠](../README.md)** | **[다음: ApplicationContext 생성 과정 ➡️](./06-application-context-creation.md)**

</div>
