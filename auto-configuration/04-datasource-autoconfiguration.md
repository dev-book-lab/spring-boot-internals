# DataSource Auto-configuration 분석 — spring.datasource.*가 HikariCP로 변환되는 경로

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `spring.datasource.url`을 설정하면 어떻게 HikariCP `DataSource`가 만들어지는가?
- `DataSourceAutoConfiguration`이 탐색하는 DataSource 타입 결정 순서는?
- `DataSourceProperties`는 어떻게 `HikariConfig`로 변환되는가?
- Embedded Database(H2, HSQLDB, Derby)는 어떤 별도 경로로 처리되는가?
- 연결 풀이 초기화될 때 실제 DB 연결 테스트가 일어나는 시점은?
- 여러 DataSource를 등록할 때 `@Primary`가 필요한 이유는?

---

## 🔍 왜 이게 존재하는가

### 문제: DataSource 설정이 라이브러리마다 다르다

```java
// HikariCP 직접 설정
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
config.setUsername("user");
config.setPassword("pass");
config.setMaximumPoolSize(10);
config.setMinimumIdle(2);
DataSource ds = new HikariDataSource(config);

// Tomcat DBCP 직접 설정
org.apache.tomcat.jdbc.pool.DataSource ds = new ...
ds.setUrl(...); ds.setDriverClassName(...);
// 다른 프로퍼티명, 다른 설정 방법
```

```
Spring Boot의 해결:
  spring.datasource.* 라는 통일된 인터페이스 제공
  → DataSourceProperties로 변환
  → 감지된 연결 풀에 맞게 자동 적용

  우선순위: HikariCP > Tomcat DBCP2 > Oracle UCP > DBCP2
  → classpath에 있는 것 중 가장 높은 우선순위 선택
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: spring.datasource.url만 설정하면 Driver 클래스를 직접 지정해야 한다

```yaml
# ❌ 불필요한 설정
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    driver-class-name: com.mysql.cj.jdbc.Driver  # 굳이 지정 안 해도 됨
    username: user
    password: pass
```

```
✅ 실제:
  DataSourceProperties.determineDriverClassName()이
  URL에서 드라이버 클래스명을 자동 추론

  "jdbc:mysql://" → "com.mysql.cj.jdbc.Driver"
  "jdbc:postgresql://" → "org.postgresql.Driver"
  "jdbc:h2://" → "org.h2.Driver"
  "jdbc:oracle:thin:" → "oracle.jdbc.OracleDriver"

  DriverManagerDataSource.setUrl()이 내부적으로 DriverManager.getDriver()로 탐색하기도 함
  → 대부분 driver-class-name 생략 가능
```

### Before: Embedded Database는 설정 없이 자동으로만 된다

```
❌ 잘못된 이해:
  "H2만 추가하면 무조건 자동으로 Embedded DB 사용"

✅ 실제:
  spring.datasource.url을 설정하지 않아야 Embedded DB 자동 구성
  spring.datasource.url이 있으면 → EmbeddedDataSourceConfiguration 스킵
  → 명시된 URL로 연결 시도

  Embedded DB 자동 구성 조건:
    spring.datasource.url 미설정
    classpath에 H2, HSQLDB, Derby 중 하나 존재
    → EmbeddedDatabaseAutoConfiguration 동작
```

---

## ✨ 올바른 이해와 사용

### After: DataSourceAutoConfiguration 처리 경로 두 갈래

```
DataSourceAutoConfiguration
  │
  ├── EmbeddedDatabaseConfiguration (URL 없음 + Embedded DB 있음)
  │     → EmbeddedDatabaseBuilder로 인메모리 DB 생성
  │
  └── PooledDataSourceConfiguration (URL 있거나 Embedded 아님)
        │
        ├── HikariDataSourceConfiguration  (@ConditionalOnClass HikariDataSource)
        ├── TomcatDataSourceConfiguration  (@ConditionalOnClass TomcatDataSource)
        ├── UcpDataSourceConfiguration     (@ConditionalOnClass PoolDataSource)
        └── Dbcp2DataSourceConfiguration   (@ConditionalOnClass BasicDataSource)
```

---

## 🔬 내부 동작 원리

### 1. DataSourceAutoConfiguration 전체 구조

```java
// DataSourceAutoConfiguration.java (단순화)
@AutoConfiguration(before = SqlInitializationAutoConfiguration.class)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)  // ← 프로퍼티 바인딩
@Import({ DataSourcePoolMetadataProvidersConfiguration.class,
          DataSourceCheckpointRestoreConfiguration.class })
public class DataSourceAutoConfiguration {

    // Embedded DB 설정
    @Configuration(proxyBeanMethods = false)
    @Conditional(EmbeddedDatabaseCondition.class)  // 커스텀 조건
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import(EmbeddedDataSourceConfiguration.class)
    protected static class EmbeddedDatabaseConfiguration { }

    // 연결 풀 DataSource 설정
    @Configuration(proxyBeanMethods = false)
    @Conditional(PooledDataSourceCondition.class)  // URL 있음 or Embedded 아님
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import({ DataSourceConfiguration.Hikari.class,
              DataSourceConfiguration.Tomcat.class,
              DataSourceConfiguration.Dbcp2.class,
              DataSourceConfiguration.OracleUcp.class,
              DataSourceConfiguration.Generic.class,
              DataSourceJmxConfiguration.class })
    protected static class PooledDataSourceConfiguration {
        // 실제 연결 풀 설정은 @Import된 내부 클래스들이 담당
    }
}
```

### 2. DataSourceProperties — 프로퍼티 바인딩

```java
// DataSourceProperties.java
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties implements BeanFactoryAware, InitializingBean {

    private String driverClassName;
    private String url;
    private String username;
    private String password;
    private String name;    // DataSource 이름 (기본값: "testdb" for embedded)
    private Class<? extends DataSource> type;  // 명시적 연결 풀 타입 지정

    // DataSource 빌더 생성
    public DataSourceBuilder<?> initializeDataSourceBuilder() {
        return DataSourceBuilder.create(getClassLoader())
            .type(getType())                  // 연결 풀 타입 결정
            .driverClassName(determineDriverClassName())  // 드라이버 자동 추론
            .url(determineUrl())              // URL (없으면 Embedded URL 생성)
            .username(determineUsername())
            .password(determinePassword());
    }

    // 드라이버 클래스명 자동 추론
    public String determineDriverClassName() {
        if (StringUtils.hasText(this.driverClassName)) {
            // 명시적 지정 있으면 그대로 사용
            return this.driverClassName;
        }
        String driverClassName = null;
        if (StringUtils.hasText(this.url)) {
            // URL에서 추론
            driverClassName = DatabaseDriver.fromJdbcUrl(this.url).getDriverClassName();
        }
        if (!StringUtils.hasText(driverClassName)) {
            // 마지막: EmbeddedDatabase 타입에서 추론
            driverClassName = this.embeddedDatabaseConnection.getDriverClassName();
        }
        return driverClassName;
    }
}
```

```java
// DatabaseDriver — URL 패턴으로 드라이버 매핑
public enum DatabaseDriver {

    H2("H2", "org.h2.Driver", "jdbc:h2", ...),
    MYSQL("MySQL", "com.mysql.cj.jdbc.Driver", "jdbc:mysql", ...),
    POSTGRESQL("PostgreSQL", "org.postgresql.Driver", "jdbc:postgresql", ...),
    ORACLE("Oracle", "oracle.jdbc.OracleDriver", "jdbc:oracle", ...),
    // ...

    public static DatabaseDriver fromJdbcUrl(String url) {
        // "jdbc:mysql://" → DatabaseDriver.MYSQL
        for (DatabaseDriver driver : values()) {
            if (url.startsWith(driver.urlPrefixes)) {
                return driver;
            }
        }
        return UNKNOWN;
    }
}
```

### 3. HikariDataSourceConfiguration — HikariCP 생성

```java
// DataSourceConfiguration.Hikari
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(HikariDataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@ConditionalOnProperty(name = "spring.datasource.type",
                        havingValue = "com.zaxxer.hikari.HikariDataSource",
                        matchIfMissing = true)  // type 미지정 시도 적용
static class Hikari {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.hikari")  // HikariCP 전용 설정
    HikariDataSource dataSource(DataSourceProperties properties) {
        // DataSourceProperties → HikariDataSource 변환
        HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);

        if (StringUtils.hasText(properties.getName())) {
            dataSource.setPoolName(properties.getName());
        }
        return dataSource;
    }
}

// DataSourceConfiguration.createDataSource()
protected static <T> T createDataSource(DataSourceProperties properties,
                                          Class<? extends DataSource> type) {
    return (T) properties.initializeDataSourceBuilder()
        .type(type)      // HikariDataSource.class
        .build();        // DataSourceBuilder.build() → HikariDataSource 인스턴스 생성
}
```

```
@ConfigurationProperties(prefix = "spring.datasource.hikari") 효과:

spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000

→ HikariDataSource의 setMaximumPoolSize(20) 등 자동 호출
→ DataSourceProperties + HikariCP 전용 설정 모두 적용
```

### 4. EmbeddedDatabaseCondition — URL 없을 때 Embedded DB 판정

```java
// EmbeddedDatabaseCondition
static class EmbeddedDatabaseCondition extends SpringBootCondition {

    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context,
                                             AnnotatedTypeMetadata metadata) {

        ConditionMessage.Builder message = ConditionMessage.forCondition("EmbeddedDataSource");

        // 연결 풀 Auto-configuration이 조건 충족하면 Embedded 사용 안 함
        if (anyMatches(context, metadata, PooledDataSourceCondition.class)) {
            return ConditionOutcome.noMatch(
                message.found("supported connection pool").items("pool"));
        }

        // Embedded DB 지원 여부 확인 (H2, HSQLDB, Derby)
        EmbeddedDatabaseType type = EmbeddedDatabaseConnection.get(
            context.getClassLoader()).getType();

        if (type == null) {
            return ConditionOutcome.noMatch(message.didNotFind("embedded database").atAll());
        }

        return ConditionOutcome.match(message.found("embedded database").items(type));
    }
}
```

```java
// EmbeddedDatabaseConnection — Embedded DB 타입 감지
public enum EmbeddedDatabaseConnection {

    NONE(null, null, null) {},
    H2(EmbeddedDatabaseType.H2, "org.h2.Driver",
       "jdbc:h2:mem:%s;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE"),
    HSQLDB(EmbeddedDatabaseType.HSQL, "org.hsqldb.jdbcDriver",
           "jdbc:hsqldb:mem:%s"),
    DERBY(EmbeddedDatabaseType.DERBY, "org.apache.derby.iapi.jdbc.AutoloadedDriver",
          "jdbc:derby:memory:%s;create=true");

    public static EmbeddedDatabaseConnection get(ClassLoader classLoader) {
        for (EmbeddedDatabaseConnection candidate : values()) {
            if (candidate != NONE && ClassUtils.isPresent(candidate.getDriverClassName(), classLoader)) {
                return candidate;
            }
        }
        return NONE;
    }
}
```

### 5. DataSource 초기화 — 실제 DB 연결 시점

```java
// HikariDataSource 생성 후 Pool 초기화
// HikariPool 생성자에서 initializeConnections() 호출

// 연결 테스트 시점:
// 1. HikariDataSource Bean 생성 (finishBeanFactoryInitialization)
// 2. new HikariPool(config)
// 3. 최소 연결 수(minimumIdle)만큼 연결 생성 시도
// 4. 연결 실패 시 → PoolInitializationException
//    → BeanCreationException → 컨텍스트 시작 실패

// spring.datasource.hikari.initialization-fail-timeout=-1 설정 시:
//   연결 실패해도 Pool 생성 성공 (지연 연결)
//   → getConnection() 호출 시점에 연결 재시도
```

---

## 💻 실험으로 확인하기

### 실험 1: 자동 설정된 DataSource 타입 확인

```java
@Autowired DataSource dataSource;

@PostConstruct
void check() {
    System.out.println("타입: " + dataSource.getClass().getSimpleName());
    // HikariCP가 classpath에 있으면: HikariDataSource
    // 없고 Tomcat DBCP 있으면: TomcatDataSource
    // 모두 없으면: GenericDataSource

    if (dataSource instanceof HikariDataSource hikari) {
        System.out.println("Pool 크기: " + hikari.getMaximumPoolSize());
        System.out.println("Pool 이름: " + hikari.getPoolName());
    }
}
```

### 실험 2: 드라이버 자동 추론 테스트

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb   # driver-class-name 생략
    # Spring Boot가 "jdbc:h2" 패턴으로 org.h2.Driver 자동 선택
```

### 실험 3: 멀티 DataSource 구성

```java
@Configuration
public class MultiDataSourceConfig {

    @Bean
    @Primary                                        // ← 기본 DataSource 지정
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

```yaml
spring:
  datasource:
    primary:
      url: jdbc:mysql://primary-host:3306/maindb
      hikari:
        maximum-pool-size: 20
    secondary:
      url: jdbc:mysql://secondary-host:3306/reportdb
      hikari:
        maximum-pool-size: 5
```

```
멀티 DataSource 시 주의:
  Auto-configuration이 @ConditionalOnMissingBean(DataSource.class)로 스킵됨
  → JPA, JdbcTemplate 등도 Auto-configuration 스킵될 수 있음
  → @EnableJpaRepositories(entityManagerFactoryRef = ...) 등 수동 설정 필요
```

### 실험 4: 연결 풀 설정 최적화 비교

```yaml
# 기본값 (HikariCP)
spring:
  datasource:
    hikari:
      maximum-pool-size: 10       # 최대 연결 수
      minimum-idle: 10            # 최소 유지 연결 (기본=maximum-pool-size)
      connection-timeout: 30000   # 연결 대기 최대 시간 (ms)
      idle-timeout: 600000        # 유휴 연결 유지 시간 (ms)
      max-lifetime: 1800000       # 연결 최대 수명 (ms) - DB 타임아웃보다 짧게
      connection-test-query:      # 연결 유효성 쿼리 (JDBC4 드라이버는 불필요)
```

---

## ⚙️ 설정 최적화 팁

```yaml
# 운영 환경 HikariCP 권장 설정
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?useSSL=true&serverTimezone=UTC
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 3000      # 3초 (너무 크면 응답 지연)
      max-lifetime: 1740000         # 29분 (MySQL wait_timeout=30분 기준)
      keepalive-time: 60000         # 1분마다 유휴 연결 유효성 확인
      pool-name: MyApp-Pool         # 모니터링 식별용
```

```java
// DataSource 헬스 체크 커스터마이징
@Component
public class DataSourceHealthContributor implements HealthContributor {
    @Autowired DataSource dataSource;

    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            // DB 버전 확인
            String dbVersion = conn.getMetaData().getDatabaseProductVersion();
            return Health.up().withDetail("version", dbVersion).build();
        } catch (SQLException e) {
            return Health.down(e).build();
        }
    }
}
```

---

## 🤔 트레이드오프

```
HikariCP (기본):
  장점  최고 성능, 안정적 Pool 관리, 상세 메트릭
  단점  native image 지원 제한 (Boot 3.x에서 개선 중)

Embedded DB (H2):
  장점  설정 불필요, 테스트 용이
  단점  재시작 시 데이터 사라짐, 프로덕션 사용 불가

멀티 DataSource:
  장점  서로 다른 DB 연결 가능
  단점  JPA, 트랜잭션 관련 Auto-configuration 수동 설정 필요
       실수로 @Transactional이 잘못된 DataSource에 적용될 수 있음

spring.datasource.hikari.maximum-pool-size:
  너무 크면  DB 서버 연결 수 초과 → DB 측 거부
  너무 작으면 요청 큐에서 connection-timeout 초과 → 에러
  권장: DB 최대 연결 수의 80% / 애플리케이션 인스턴스 수
```

---

## 📌 핵심 정리

```
DataSourceAutoConfiguration 두 경로
  EmbeddedDatabaseConfiguration   URL 없음 + Embedded DB 있음
  PooledDataSourceConfiguration   URL 있음 또는 Embedded 조건 미충족

연결 풀 우선순위
  HikariCP > Tomcat DBCP2 > Oracle UCP > DBCP2
  spring.datasource.type으로 명시 가능

DataSourceProperties → HikariDataSource 변환
  initializeDataSourceBuilder() → DataSourceBuilder.build()
  드라이버 클래스명 URL에서 자동 추론
  spring.datasource.hikari.* → HikariCP 전용 설정 자동 바인딩

멀티 DataSource
  @ConditionalOnMissingBean(DataSource.class) → Auto-configuration 스킵
  JPA, JdbcTemplate 등 수동 설정 필요
  @Primary로 기본 DataSource 지정

연결 초기화 시점
  finishBeanFactoryInitialization() → HikariDataSource Bean 생성
  → new HikariPool() → minimumIdle 연결 생성 시도
  → 실패 시 컨텍스트 시작 실패
```

---

## 🤔 생각해볼 문제

**Q1.** `spring.datasource.url`을 설정하지 않고 H2 의존성만 추가했을 때 생성되는 DB URL은 무엇인가? 애플리케이션을 재시작하면 DB URL이 바뀌는가?

**Q2.** `spring.datasource.hikari.maximum-pool-size=100`으로 설정했는데 실제 Pool이 100개 연결을 즉시 생성하지 않는 이유는 무엇인가?

**Q3.** `DataSourceAutoConfiguration`이 스킵되는 상황에서도 `DataSourceProperties` Bean은 등록되는가?

> 💡 **해설**
>
> **Q1.** URL은 `jdbc:h2:mem:{applicationName};DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE` 형태로 생성된다. `{applicationName}`은 `spring.application.name` 값 또는 없으면 `"testdb"`가 사용된다. 인메모리 H2 DB이므로 애플리케이션 재시작마다 새 메모리 공간이 생성되어 데이터가 초기화된다. URL의 DB 이름 자체는 동일하지만 JVM 인스턴스가 새로 시작되므로 데이터가 사라진다.
>
> **Q2.** HikariCP는 `minimumIdle` 개수만큼 초기 연결을 생성하며, 기본값은 `maximumPoolSize`와 같다. 따라서 시작 시 100개 연결이 시도될 수 있다. 하지만 이 연결 생성은 백그라운드 스레드에서 비동기로 일어나며, 실제로는 빠른 시간 내에 순차적으로 생성된다. `minimumIdle`을 별도로 낮게 설정하면(`minimum-idle: 5`) 시작 시 5개만 생성하고 나머지는 요청에 따라 동적으로 추가된다.
>
> **Q3.** 아니다. `DataSourceProperties`는 `@EnableConfigurationProperties(DataSourceProperties.class)`로 등록되는데, 이 어노테이션은 `DataSourceAutoConfiguration` 클래스에 붙어 있다. `DataSourceAutoConfiguration` 자체가 `@ConditionalOnMissingBean(DataSource.class)` 조건으로 스킵되면(정확히는 내부 `PooledDataSourceConfiguration`이 스킵되면), `@EnableConfigurationProperties`도 함께 스킵될 수 있다. 단, `DataSourceAutoConfiguration` 클래스 수준이 아니라 내부 static 클래스 수준에서 스킵되는 경우에는 외부 클래스의 `@EnableConfigurationProperties`는 처리된다. 정확한 동작은 어느 레벨에서 스킵이 일어나는지에 따라 다르므로, 멀티 DataSource 환경에서는 `DataSourceProperties`가 필요하면 직접 `@EnableConfigurationProperties(DataSourceProperties.class)`를 붙이는 것이 안전하다.

---

<div align="center">

**[⬅️ 이전: @AutoConfigureAfter/@AutoConfigureBefore 순서 제어](./03-autoconfigure-order.md)** | **[홈으로 🏠](../README.md)** | **[다음: JPA Auto-configuration 과정 ➡️](./05-jpa-autoconfiguration.md)**

</div>
