# JPA Auto-configuration 과정 — HibernateJpaAutoConfiguration에서 EntityManagerFactory까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `HibernateJpaAutoConfiguration`이 `LocalContainerEntityManagerFactoryBean`을 생성하는 전체 체인은?
- `spring.jpa.properties.*` 설정이 Hibernate `SessionFactory`에 적용되는 경로는?
- `@AutoConfigurationPackage`가 등록한 패키지가 JPA Entity 스캔에 사용되는 원리는?
- `spring.jpa.hibernate.ddl-auto`의 기본값이 환경마다 다른 이유는?
- `spring.jpa.open-in-view`는 무엇이고 왜 경고 로그가 출력되는가?
- `JpaRepositoriesAutoConfiguration`이 `HibernateJpaAutoConfiguration` 이후에 처리되어야 하는 이유는?

---

## 🔍 왜 이게 존재하는가

### 문제: JPA 설정의 복잡성

```java
// JPA를 직접 설정한다면
@Bean
public LocalContainerEntityManagerFactoryBean entityManagerFactory(DataSource dataSource) {
    LocalContainerEntityManagerFactoryBean emf = new LocalContainerEntityManagerFactoryBean();
    emf.setDataSource(dataSource);
    emf.setPackagesToScan("com.example.domain");  // @Entity 스캔 패키지
    emf.setJpaVendorAdapter(new HibernateJpaVendorAdapter());

    Properties props = new Properties();
    props.setProperty("hibernate.dialect", "org.hibernate.dialect.MySQL8Dialect");
    props.setProperty("hibernate.hbm2ddl.auto", "validate");
    props.setProperty("hibernate.show_sql", "true");
    emf.setJpaProperties(props);
    return emf;
}

@Bean
public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
    return new JpaTransactionManager(emf);
}
```

```
Spring Boot의 해결:
  spring.jpa.* 통일 인터페이스 제공
  → JpaProperties / HibernateProperties로 바인딩
  → 자동으로 Dialect 추론, 패키지 스캔, 트랜잭션 관리자 등록
  → 필요 시 일부만 오버라이드
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: spring.jpa.hibernate.ddl-auto=create-drop이 테스트에 좋다

```
❌ 잘못된 이해:
  "테스트에서는 create-drop을 써서 테이블을 매번 새로 만들면 좋다"

✅ 실제:
  create-drop은 SessionFactory 종료 시 drop
  → @SpringBootTest에서 전체 테스트 실행 중 한 번만 종료
  → 중간에 drop이 일어나지 않음 → 의도한 "매번 새로" 효과 없음

  테스트 격리를 위한 올바른 방법:
    @Transactional + @Rollback(true) (기본값) → 각 테스트 후 롤백
    또는 @DataJpaTest (자동으로 rollback 설정)
    또는 @Sql(executionPhase = BEFORE_TEST_METHOD)로 데이터 초기화
```

### Before: spring.jpa.open-in-view=false가 무조건 좋다

```
❌ 잘못된 이해:
  "open-in-view 경고 로그가 나오니까 false로 설정해야 한다"

✅ 실제:
  open-in-view=true (기본값):
    HTTP 요청 수명 동안 EntityManager 세션 유지
    → View 렌더링 중에도 Lazy 로딩 가능
    → 하지만 트랜잭션 종료 후에도 DB 연결 유지 → 연결 풀 점유

  open-in-view=false:
    서비스 레이어 트랜잭션 종료 시 세션 종료
    → View에서 Lazy 로딩 시 LazyInitializationException
    → 연결 풀 효율 향상

  권장:
    새 프로젝트: false (명시적 Eager 로딩 또는 DTO 프로젝션 권장)
    레거시: 기존 동작 유지를 위해 true 유지 후 점진적 개선
```

---

## ✨ 올바른 이해와 사용

### After: JPA Auto-configuration 처리 순서

```
처리 순서 (AutoConfigureAfter 관계):

DataSourceAutoConfiguration
  ↓ (after)
HibernateJpaAutoConfiguration
  ├── JpaBaseConfiguration
  │     ├── EntityManagerFactory (LocalContainerEntityManagerFactoryBean)
  │     ├── JpaTransactionManager
  │     └── OpenEntityManagerInViewInterceptor (open-in-view=true 시)
  └── HibernateJpaConfiguration
        └── Hibernate 전용 설정 (dialect, naming strategy 등)

  ↓ (after)
JpaRepositoriesAutoConfiguration
  └── @EnableJpaRepositories (Repository 스캔)
```

---

## 🔬 내부 동작 원리

### 1. HibernateJpaAutoConfiguration — 진입점

```java
// HibernateJpaAutoConfiguration.java
@AutoConfiguration(
    after = { DataSourceAutoConfiguration.class,          // DataSource 먼저
              DataSourceTransactionManagerAutoConfiguration.class }
)
@ConditionalOnClass({ LocalContainerEntityManagerFactoryBean.class,
                      EntityManager.class, SessionImplementor.class })
@ConditionalOnSingleCandidate(DataSource.class)  // 단일 DataSource만 있을 때
@EnableConfigurationProperties(JpaProperties.class)
@Import(HibernateJpaConfiguration.class)  // ← 실제 설정 위임
public class HibernateJpaAutoConfiguration { }

// @ConditionalOnSingleCandidate(DataSource.class):
// DataSource가 하나뿐이거나 @Primary가 하나일 때만 동작
// → 멀티 DataSource 환경에서는 자동 JPA 설정 스킵
```

```java
// HibernateJpaConfiguration.java (실제 설정 담당)
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(HibernateProperties.class)
@Import({ HibernateDefaultDdlAutoProvider.class, ... })
class HibernateJpaConfiguration extends JpaBaseConfiguration {

    HibernateJpaConfiguration(DataSource dataSource,
                               JpaProperties jpaProperties,
                               HibernateProperties hibernateProperties, ...) {
        super(dataSource, jpaProperties, ...);
        this.hibernateProperties = hibernateProperties;
    }

    @Override
    protected AbstractJpaVendorAdapter createJpaVendorAdapter() {
        return new HibernateJpaVendorAdapter();
    }

    @Override
    protected Map<String, Object> getVendorProperties() {
        // spring.jpa.properties.* + Hibernate 전용 설정 병합
        Supplier<String> defaultDdlMode = () -> this.defaultDdlAutoProvider.getDefaultDdlAuto(
            getDataSource());

        return new LinkedHashMap<>(this.hibernateProperties.determineHibernateProperties(
            this.jpaProperties.getProperties(),   // spring.jpa.properties.*
            new HibernateSettings()
                .ddlAuto(defaultDdlMode)           // ddl-auto 기본값
                .hibernatePropertiesCustomizers(
                    this.hibernatePropertiesCustomizers)
        ));
    }
}
```

### 2. JpaBaseConfiguration — EntityManagerFactory 생성

```java
// JpaBaseConfiguration.java — 핵심 Bean 등록
@Configuration(proxyBeanMethods = false)
public abstract class JpaBaseConfiguration {

    // ① EntityManagerFactory Bean
    @Bean
    @ConditionalOnMissingBean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            EntityManagerFactoryBuilder factoryBuilder) {

        // 스캔 패키지 결정 (핵심!)
        Map<String, Object> vendorProperties = getVendorProperties();
        customizeVendorProperties(vendorProperties);

        return factoryBuilder
            .dataSource(this.dataSource)
            .packages(getPackagesToScan())     // @Entity 스캔 패키지
            .properties(vendorProperties)       // Hibernate 프로퍼티
            .mappingResources(getMappingResources())
            .jta(isJta())
            .build();
    }

    // ② 스캔 패키지 결정
    protected String[] getPackagesToScan() {
        // AutoConfigurationPackages에서 등록된 패키지 조회
        // (@SpringBootApplication 위치 패키지)
        List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
        if (logger.isDebugEnabled()) {
            logger.debug("Using auto-configuration base packages " + packages);
        }
        return StringUtils.toStringArray(packages);
    }

    // ③ JpaTransactionManager Bean
    @Bean
    @ConditionalOnMissingBean(TransactionManager.class)
    public PlatformTransactionManager transactionManager(
            ObjectProvider<TransactionManagerCustomizers> transactionManagerCustomizers) {

        JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManagerCustomizers.ifAvailable(
            customizers -> customizers.customize(transactionManager));
        return transactionManager;
    }

    // ④ EntityManagerFactoryBuilder Bean
    @Bean
    @ConditionalOnMissingBean
    public EntityManagerFactoryBuilder entityManagerFactoryBuilder(
            JpaVendorAdapter jpaVendorAdapter,
            ObjectProvider<PersistenceUnitManager> persistenceUnitManager, ...) {
        return new EntityManagerFactoryBuilder(jpaVendorAdapter, ...);
    }
}
```

### 3. @AutoConfigurationPackage → Entity 스캔 연결

```java
// getPackagesToScan() → AutoConfigurationPackages.get()

// AutoConfigurationPackages.java
public static List<String> get(BeanFactory beanFactory) {
    try {
        // AutoConfigurationPackages.BasePackages Bean 조회
        return beanFactory.getBean(BEAN, BasePackages.class).get();
    }
    catch (NoSuchBeanDefinitionException ex) {
        throw new IllegalStateException(...);
    }
}

// BasePackages Bean은 어디서?
// @SpringBootApplication
//   → @EnableAutoConfiguration
//     → @AutoConfigurationPackage
//       → @Import(AutoConfigurationPackages.Registrar.class)
//         → Registrar.registerBeanDefinitions()
//           → AutoConfigurationPackages.register(registry, packages)
//             → "autoConfigurationPackages" 이름의 BasePackages Bean 등록
//               → 값: @SpringBootApplication 위치 패키지 (예: "com.example")

// 결과:
// @SpringBootApplication이 com.example.App에 있으면
// JPA Entity 스캔 범위 = "com.example" (하위 포함)
```

```
@Entity 클래스가 스캔되지 않는 흔한 원인:

1. @SpringBootApplication이 서브 패키지에 있는 경우
   com.example.config.App → 스캔 범위: com.example.config
   com.example.domain.User (@Entity) → 스캔 안 됨!

2. 멀티 모듈 프로젝트
   core 모듈의 com.example.core.domain.User
   → @SpringBootApplication이 app 모듈의 com.example.app에 있으면
   → com.example.core는 스캔 범위 밖

   해결: @EntityScan("com.example") 명시
```

### 4. HibernateProperties — Hibernate 전용 설정

```java
// HibernateProperties.java
@ConfigurationProperties(prefix = "spring.jpa.hibernate")
public class HibernateProperties {

    private final Ddl ddl = new Ddl();
    private String namingStrategy;

    // spring.jpa.hibernate.ddl-auto 처리
    public Map<String, Object> determineHibernateProperties(
            Map<String, String> jpaProperties, HibernateSettings settings) {

        Map<String, Object> result = new HashMap<>(jpaProperties);

        // ddl-auto 결정
        String ddlAuto = determineDdlAuto(settings, result);
        if (StringUtils.hasText(ddlAuto) && !"none".equals(ddlAuto)) {
            result.put("hibernate.hbm2ddl.auto", ddlAuto);
        }
        else {
            result.remove("hibernate.hbm2ddl.auto");
        }
        return result;
    }
}
```

```java
// HibernateDefaultDdlAutoProvider — 환경별 기본값 결정
@Bean
@ConditionalOnMissingBean
HibernateDefaultDdlAutoProvider hibernateDefaultDdlAutoProvider(
        ObjectProvider<SchemaManagementProvider> providers) {
    return new HibernateDefaultDdlAutoProvider(providers);
}

// 기본값 결정 로직
String getDefaultDdlAuto(DataSource dataSource) {
    // Embedded DB (H2 인메모리)인지 확인
    if (!isEmbedded(dataSource)) {
        logger.debug("spring.jpa.hibernate.ddl-auto is not explicitly set, using 'none'");
        return "none";          // 외부 DB → 안전하게 none (스키마 자동 변경 없음)
    }
    return "create-drop";       // 인메모리 DB → 시작 시 생성, 종료 시 drop
}
```

```
ddl-auto 기본값 정책:
  Embedded DB (H2 인메모리):  create-drop (테스트 편의)
  외부 DB (MySQL, PostgreSQL): none         (운영 안전)

명시적 설정 권장:
  spring.jpa.hibernate.ddl-auto: validate  (운영: 스키마 일치 검증만)
  spring.jpa.hibernate.ddl-auto: update    (개발: 필드 추가 자동 반영)
  spring.jpa.hibernate.ddl-auto: create    (초기 개발: 매번 재생성)
```

### 5. spring.jpa.properties.* — Hibernate 직접 설정

```yaml
spring:
  jpa:
    properties:
      hibernate:
        # 배치 처리 최적화
        jdbc.batch_size: 50
        order_inserts: true
        order_updates: true
        # 2차 캐시
        cache.use_second_level_cache: true
        cache.region.factory_class: org.hibernate.cache.ehcache.EhCacheRegionFactory
        # 통계
        generate_statistics: true
        # 쿼리 계획 캐시
        query.plan_cache_max_size: 2048
```

```java
// 적용 경로:
// application.yml spring.jpa.properties.hibernate.jdbc.batch_size=50
//   → JpaProperties.getProperties() → {"hibernate.jdbc.batch_size": "50"}
//   → HibernateJpaConfiguration.getVendorProperties() → vendorProperties에 포함
//   → LocalContainerEntityManagerFactoryBean.setJpaProperties()
//   → Hibernate SessionFactory 생성 시 적용
```

### 6. OpenEntityManagerInViewInterceptor — open-in-view 구현

```java
// JpaWebConfiguration — open-in-view 관련
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(WebMvcConfigurer.class)
@ConditionalOnMissingBean({ OpenEntityManagerInViewInterceptor.class,
                             OpenEntityManagerInViewFilter.class })
@ConditionalOnMissingFilterBean(OpenEntityManagerInViewFilter.class)
@ConditionalOnProperty(prefix = "spring.jpa", name = "open-in-view",
                        havingValue = "true", matchIfMissing = true)  // 기본값 true
protected static class JpaWebMvcConfiguration {

    // 경고 로그 출력
    @Bean
    public OpenEntityManagerInViewInterceptor openEntityManagerInViewInterceptor() {
        if (jpaProperties.getOpenInView() == null) {
            // spring.jpa.open-in-view 명시 안 됐을 때 경고
            logger.warn("spring.jpa.open-in-view is enabled by default. "
                + "Therefore, database queries may be performed during view rendering. "
                + "Explicitly configure spring.jpa.open-in-view to disable this warning");
        }
        return new OpenEntityManagerInViewInterceptor();
    }
}

// OpenEntityManagerInViewInterceptor 동작:
// HTTP 요청 시작: EntityManager 세션 열기
// HTTP 요청 완료: EntityManager 세션 닫기 (commit/rollback은 서비스 트랜잭션에서)
// → View 렌더링 중 Lazy 로딩 가능
// → 단, 트랜잭션 없이 Lazy 로딩 → 성능 문제, N+1 숨김
```

### 7. JpaRepositoriesAutoConfiguration

```java
// JpaRepositoriesAutoConfiguration.java
@AutoConfiguration(
    after = { HibernateJpaAutoConfiguration.class,  // EMF 먼저 필요
              TaskExecutionAutoConfiguration.class }
)
@ConditionalOnClass(JpaRepository.class)
@ConditionalOnMissingBean({ JpaRepositoryFactoryBean.class,
                             JpaRepositoryConfigExtension.class })
@ConditionalOnSingleCandidate(DataSource.class)
@Import(JpaRepositoriesRegistrar.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
public class JpaRepositoriesAutoConfiguration { }

// JpaRepositoriesRegistrar.class:
// → @EnableJpaRepositories 효과
// → AutoConfigurationPackages에서 패키지 조회
// → 그 패키지 하위의 JpaRepository 구현 인터페이스 스캔
// → 프록시 생성 → Bean 등록
```

---

## 💻 실험으로 확인하기

### 실험 1: Entity 스캔 범위 확인

```java
@SpringBootApplication  // com.example.App
public class App { }

// com.example.domain.User — @Entity → 스캔됨
// com.other.legacy.Product — @Entity → 스캔 안 됨!

// 해결 방법 1: @EntityScan 추가
@SpringBootApplication
@EntityScan(basePackages = {"com.example", "com.other"})
public class App { }

// 해결 방법 2: @SpringBootApplication 위치를 상위 패키지로
// com.App (상위 패키지) → com.example, com.other 모두 스캔됨
```

### 실험 2: ddl-auto 동작 확인

```yaml
# application-test.yml (테스트 환경)
spring:
  datasource:
    url: jdbc:h2:mem:testdb   # 인메모리 → ddl-auto 기본값: create-drop
  jpa:
    show-sql: true

# application.yml (운영 환경)
spring:
  datasource:
    url: jdbc:mysql://...      # 외부 DB → ddl-auto 기본값: none
  jpa:
    hibernate:
      ddl-auto: validate        # 명시적으로 validate 권장
```

### 실험 3: Hibernate 통계 활성화

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
logging:
  level:
    org.hibernate.stat: DEBUG
```

```
출력 예시:
HHH000117: HQL: select u from User u, time: 12ms, rows: 100
Session Metrics {
    2041 nanoseconds spent acquiring 1 JDBC connections;
    ...
    48730 nanoseconds spent executing 1 JDBC statements;
}
```

### 실험 4: open-in-view 비활성화 후 Lazy 로딩 처리

```java
// open-in-view=false 설정 후
@Service
@Transactional(readOnly = true)
public class UserService {

    public UserWithOrdersDto getUserWithOrders(Long userId) {
        User user = userRepository.findById(userId).orElseThrow();
        // 트랜잭션 내에서 Lazy 로딩 → 정상
        List<Order> orders = user.getOrders(); // Lazy 로딩 실행
        return new UserWithOrdersDto(user, orders);
        // 반환 후 트랜잭션 종료 → 세션 닫힘
        // Controller에서 DTO 접근 → 이미 로딩된 데이터 → 정상
    }
}
// Controller에서 user.getOrders() 직접 접근 시도 → LazyInitializationException
// → 서비스 레이어에서 모든 로딩 완료 후 DTO로 반환하는 패턴 강제
```

---

## ⚙️ 설정 최적화 팁

```yaml
# 운영 환경 JPA 권장 설정
spring:
  jpa:
    open-in-view: false               # 연결 풀 효율화, Lazy 로딩 명시 강제
    hibernate:
      ddl-auto: validate              # 스키마 자동 변경 없음, 불일치 시 시작 실패
    properties:
      hibernate:
        # 배치 최적화 (대량 insert/update 시)
        jdbc.batch_size: 50
        order_inserts: true
        order_updates: true
        # 쿼리 계획 캐시 (복잡한 JPQL 많을 때)
        query.plan_cache_max_size: 2048
        # 통계 (개발/모니터링 환경만)
        # generate_statistics: true
```

```java
// @EntityScan, @EnableJpaRepositories 커스터마이징
@SpringBootApplication
@EntityScan("com.example.domain")     // Entity만 다른 패키지에 있을 때
@EnableJpaRepositories(              // Repository 스캔 커스터마이징
    basePackages = "com.example.repository",
    entityManagerFactoryRef = "entityManagerFactory",
    transactionManagerRef = "transactionManager"
)
public class App { }
```

---

## 🤔 트레이드오프

```
open-in-view=true (기본):
  장점  Lazy 로딩 유연, View에서 접근 편의
  단점  트랜잭션 종료 후 세션 유지 → 연결 풀 점유 시간 길어짐
       N+1 문제를 숨겨 개발 중 발견 어려움

ddl-auto=update (개발):
  장점  필드 추가 시 자동 반영
  단점  컬럼 삭제, 타입 변경은 자동 처리 안 됨 → 불일치 발생 가능
       운영에서 절대 사용 금지

@ConditionalOnSingleCandidate(DataSource.class):
  장점  멀티 DataSource 환경에서 혼란 방지
  단점  멀티 DataSource 시 JPA 자동 설정 전체 스킵
       → @EnableJpaRepositories 등 수동 설정 필요

Hibernate 배치:
  장점  대량 insert/update 성능 대폭 향상 (네트워크 왕복 감소)
  단점  @GeneratedValue(IDENTITY) 사용 시 배치 insert 불가
       → SEQUENCE 전략으로 변경 필요
```

---

## 📌 핵심 정리

```
Auto-configuration 처리 순서
  DataSourceAutoConfiguration
  → HibernateJpaAutoConfiguration (@AutoConfigureAfter)
  → JpaRepositoriesAutoConfiguration (@AutoConfigureAfter)

EntityManagerFactory 생성 체인
  HibernateJpaAutoConfiguration
  → HibernateJpaConfiguration extends JpaBaseConfiguration
  → LocalContainerEntityManagerFactoryBean
  → 패키지 스캔: AutoConfigurationPackages (= @SpringBootApplication 위치)

ddl-auto 기본값
  Embedded DB: create-drop (테스트 편의)
  외부 DB:     none (운영 안전)

spring.jpa.properties.hibernate.*
  Hibernate 전용 설정 직접 전달
  → SessionFactory 생성 시 적용

open-in-view
  기본값: true → 경고 로그
  false 권장: 연결 풀 효율, Lazy 로딩 명시 강제
```

---

## 🤔 생각해볼 문제

**Q1.** `@EntityScan("com.example")`과 `@SpringBootApplication(scanBasePackages = "com.example")`를 함께 사용하는 경우와 `@EntityScan` 없이 `@SpringBootApplication`만 `com.example.App`에 두는 경우의 차이는 무엇인가?

**Q2.** 멀티 DataSource 환경에서 JPA를 사용하려면 어떻게 해야 하는가? `HibernateJpaAutoConfiguration`이 스킵되는 이유와 함께 설명하라.

**Q3.** `spring.jpa.hibernate.ddl-auto=update`를 운영 환경에서 절대 사용하면 안 되는 구체적인 이유 두 가지를 설명하라.

> 💡 **해설**
>
> **Q1.** `@SpringBootApplication(scanBasePackages = "com.example")`는 `@ComponentScan`의 범위를 설정하며, JPA Entity 스캔 범위는 `@AutoConfigurationPackage`로 등록되는 패키지에 의존한다. `@AutoConfigurationPackage`는 `@EnableAutoConfiguration`이 붙은 클래스의 패키지를 등록하므로, `@SpringBootApplication`이 `com.example.App`에 있으면 `com.example`이 자동으로 Entity 스캔 범위가 된다. `@EntityScan("com.example")`은 이 자동 범위를 명시적으로 덮어쓰는 것이므로 동일한 패키지를 지정하면 결과가 같다. 다만 `@EntityScan`은 `AutoConfigurationPackages`에서 가져오는 경로를 무시하고 직접 `packagesToScan`을 설정하므로, 멀티 모듈에서 Entity가 다른 패키지에 있을 때 명시적으로 지정해야 한다.
>
> **Q2.** `HibernateJpaAutoConfiguration`은 `@ConditionalOnSingleCandidate(DataSource.class)`를 가지므로 DataSource Bean이 2개 이상이면 스킵된다. 멀티 DataSource에서 JPA를 사용하려면 각 DataSource별로 `LocalContainerEntityManagerFactoryBean`과 `JpaTransactionManager`를 직접 `@Bean`으로 등록해야 한다. 또한 `@EnableJpaRepositories(entityManagerFactoryRef = "primaryEmf", transactionManagerRef = "primaryTx", basePackages = "...")`를 각 DataSource에 맞게 별도로 선언해야 한다. `@Primary`를 붙인 DataSource로는 기본 JPA 설정이 연결되고, 나머지는 수동으로 구성한다.
>
> **Q3.** 첫째, `update`는 컬럼 추가만 자동 처리하고 컬럼 삭제·타입 변경·제약조건 변경은 처리하지 않는다. Entity 클래스에서 필드를 삭제해도 DB 컬럼은 그대로 남아 스키마 불일치가 누적된다. 둘째, 여러 인스턴스가 동시에 시작할 때 `update`가 동시에 DDL을 실행하면 Race Condition으로 스키마가 손상될 수 있다. 운영에서는 Flyway나 Liquibase 같은 마이그레이션 도구로 버전 관리된 DDL 스크립트를 적용하고 `ddl-auto: validate`(또는 `none`)를 사용해야 한다.

---

<div align="center">

**[⬅️ 이전: DataSource Auto-configuration 분석](./04-datasource-autoconfiguration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Web MVC Auto-configuration ➡️](./06-web-mvc-autoconfiguration.md)**

</div>
