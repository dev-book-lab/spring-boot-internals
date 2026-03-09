# Profile 활성화 전략 — spring.profiles.active부터 Profile 그룹까지

---

## 🎯 핵심 질문

- `spring.profiles.active`와 `spring.profiles.include`의 차이는?
- `application-{profile}.yml`은 언제, 어떤 순서로 로딩되는가?
- Profile 그룹(`spring.profiles.group`)은 어떻게 동작하는가?
- `@Profile`이 Bean 등록을 제어하는 내부 메커니즘은?
- 멀티 문서 YAML의 `spring.config.activate.on-profile`과 별도 파일 방식의 차이는?

---

## 🔍 왜 이게 존재하는가

```
환경별 다른 설정 필요:

개발(dev):  H2 인메모리 DB, DEBUG 로그, 빠른 시작
테스트(test): Testcontainers, 느린 로그
운영(prod): MySQL, WARN 로그, 커넥션 풀 최적화

Profile 없이:
  모든 환경 설정이 하나의 파일에 → if-else 로직 → 복잡
  또는 배포마다 파일 수동 수정 → 실수 발생

Profile 있으면:
  application-dev.yml, application-prod.yml 분리
  → 환경 변수 하나로 전체 설정 전환
  SPRING_PROFILES_ACTIVE=prod java -jar app.jar
```

---

## 🔬 내부 동작 원리

### 1. Profile 활성화 3가지 방법

```bash
# ① 시스템 프로퍼티 (JVM 인수)
java -Dspring.profiles.active=prod,monitoring -jar app.jar

# ② 환경변수
SPRING_PROFILES_ACTIVE=prod,monitoring java -jar app.jar

# ③ application.yml (기본값 설정용)
spring:
  profiles:
    active: dev   # 코드에 active 설정은 권장 안 함 — 실수로 prod 배포 위험
```

```java
// 프로그래밍 방식 (테스트용)
@ActiveProfiles("test")           // @SpringBootTest에서
class MyServiceTest { ... }

// 또는
SpringApplication app = new SpringApplication(App.class);
app.setAdditionalProfiles("monitoring");
app.run(args);
```

### 2. application-{profile}.yml 로딩 순서

```
로딩 순서 (우선순위 낮은 것 → 높은 것):

1. application.yml              (기본값)
2. application-default.yml      (active profile 없을 때만)
3. application-{profile}.yml    (활성 Profile 순서대로)

여러 Profile 활성화 시:
  spring.profiles.active=db,monitoring
  
  로딩 순서:
  1. application.yml
  2. application-db.yml
  3. application-monitoring.yml   ← 나중에 로딩된 것이 높은 우선순위
  
  → monitoring 설정이 db 설정을 덮어씀 (나중이 이김)
  → active 목록에서 뒤에 있는 Profile이 높은 우선순위
```

```java
// ConfigDataEnvironment — Profile별 파일 탐색
// StandardConfigDataLocationResolver가 각 location에서
// "application-{profile}" 파일 탐색

// 탐색 경로 예시 (prod Profile):
// classpath:application-prod.properties
// classpath:application-prod.yml
// classpath:application-prod.yaml
// classpath:/config/application-prod.yml
// file:./application-prod.yml
// file:./config/application-prod.yml
```

### 3. spring.profiles.active vs spring.profiles.include

```yaml
# spring.profiles.active — 활성 Profile 지정 (외부에서 완전히 교체됨)
spring:
  profiles:
    active: prod
# 외부에서 --spring.profiles.active=dev 하면 prod 대신 dev

# spring.profiles.include — 현재 Profile에 추가 (교체 안 됨)
# application-prod.yml:
spring:
  profiles:
    include:
      - db-prod
      - cache-prod
      - monitoring
# → prod Profile 활성화 시 db-prod, cache-prod, monitoring도 항상 함께 활성화
# → 외부에서 덮어쓸 수 없음 (include는 항상 포함)

# 사용 기준:
# active: 환경(dev/prod)처럼 선택적으로 전환되는 것
# include: prod에 항상 필요한 부가 설정 (모니터링, 캐시 등)
```

```
⚠️ Boot 2.4+ include 제한:
  Profile별 파일(application-prod.yml) 안에서 spring.profiles.include는 사용 가능
  하지만 spring.profiles.active는 Profile별 파일 내부에서 사용 불가 (무시됨)
  → 활성 Profile은 외부에서만 제어
```

### 4. Profile 그룹 — spring.profiles.group

```yaml
# application.yml
spring:
  profiles:
    group:
      production:         # "production" 활성화 시 자동으로 포함되는 Profile들
        - proddb
        - prodmq
        - prodcache
      development:
        - devdb
        - devtools

# application-proddb.yml — DB 전용 설정
spring:
  datasource:
    url: jdbc:mysql://prod-db:3306/mydb
    hikari:
      maximum-pool-size: 20

# application-prodmq.yml — MQ 전용 설정
spring:
  rabbitmq:
    host: prod-mq.internal
```

```bash
# 실행
java -jar app.jar --spring.profiles.active=production
# → production + proddb + prodmq + prodcache 모두 활성화
# → 모듈별로 Profile 파일을 분리해 관리 가능
```

### 5. @Profile — Bean 등록 제어

```java
// @Profile 내부: ProfileCondition → AbstractEnvironment.matchesProfiles()

@Configuration
@Profile("dev")     // dev Profile일 때만 이 @Configuration 전체 활성화
public class DevConfig {

    @Bean
    public DataSource h2DataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {

    @Bean
    public DataSource mysqlDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:mysql://prod:3306/db")
            .build();
    }
}

// @Bean 메서드 레벨도 가능
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDs() { ... }

    @Bean
    @Profile("prod")
    public DataSource prodDs() { ... }
}
```

```java
// ProfileCondition 내부
class ProfileCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        MultiValueMap<String, Object> attrs =
            metadata.getAllAnnotationAttributes(Profile.class.getName());

        if (attrs != null) {
            String[] value = (String[]) attrs.getFirst("value");
            // Environment.matchesProfiles()로 활성 Profile 확인
            return context.getEnvironment().matchesProfiles(value);
        }
        return true;
    }
}

// matchesProfiles() — Profile 표현식 지원 (Boot 2.1+)
@Profile("prod & !eu-region")   // prod이고 eu-region이 아닌 경우
@Profile("dev | test")          // dev 또는 test
@Profile("!prod")               // prod가 아닌 경우
```

### 6. 멀티 문서 YAML vs 파일 분리

```yaml
# 방식 1: 멀티 문서 YAML (하나의 파일)
---
# 기본값 (모든 환경)
spring.application.name: my-app
server.port: 8080
---
spring.config.activate.on-profile: dev
spring.datasource.url: jdbc:h2:mem:devdb
logging.level.root: DEBUG
---
spring.config.activate.on-profile: prod
spring.datasource.url: jdbc:mysql://prod:3306/mydb
logging.level.root: WARN
```

```yaml
# 방식 2: 파일 분리
# application.yml
spring.application.name: my-app
server.port: 8080

# application-dev.yml
spring.datasource.url: jdbc:h2:mem:devdb
logging.level.root: DEBUG

# application-prod.yml
spring.datasource.url: jdbc:mysql://prod:3306/mydb
logging.level.root: WARN
```

```
선택 기준:

멀티 문서 YAML:
  장점  하나의 파일에서 전체 구조 파악 가능
  단점  파일 길어짐, 실수로 Profile 누락 가능
       환경별 설정 분리가 파일 수준에서 안 됨 (권한, 암호화 등)
  적합  소규모 프로젝트, 설정이 단순할 때

파일 분리:
  장점  환경별 파일 독립, PR 리뷰 시 명확
       파일별 Secret 관리 도구 적용 가능 (Vault, Sealed Secrets)
  단점  파일 수 증가
  적합  대규모 프로젝트, 운영 환경 설정 별도 관리 필요 시
```

---

## 💻 실험으로 확인하기

### 실험 1: Profile 로딩 순서 확인

```bash
java -jar app.jar --spring.profiles.active=base,override --debug 2>&1 \
  | grep "Loaded config"
# 출력:
# Loaded config file 'classpath:application.yml'
# Loaded config file 'classpath:application-base.yml'
# Loaded config file 'classpath:application-override.yml'  ← 높은 우선순위
```

### 실험 2: Profile 그룹 활성화 확인

```java
@SpringBootTest
@ActiveProfiles("production")
class ProductionProfileTest {

    @Autowired Environment environment;

    @Test
    void productionGroupActivatesSubProfiles() {
        assertThat(environment.getActiveProfiles())
            .contains("production", "proddb", "prodmq", "prodcache");
    }
}
```

### 실험 3: @Profile 표현식

```java
@Bean
@Profile("prod & monitoring")    // prod이면서 monitoring도 활성화된 경우만
public MetricsCollector prodMetrics() { ... }

@Bean
@Profile("!prod")                 // prod가 아닌 모든 환경
public MockExternalService mockService() { ... }
```

---

## ⚙️ 설정 최적화 팁

```yaml
# Profile 설계 원칙

# ① 환경 Profile: 완전히 상호 배타적
#    dev, staging, prod (동시에 하나만)

# ② 기능 Profile: 환경 위에 독립적으로 추가
#    monitoring, feature-flags, debug-sql

# ③ 기본값은 application.yml에 (Profile 없이도 동작)
#    Profile별 파일은 변경할 것만 기술

# application.yml (모든 환경 기본값)
spring:
  datasource:
    hikari:
      maximum-pool-size: 10  # 기본값

# application-prod.yml (prod에서만 변경)
spring:
  datasource:
    hikari:
      maximum-pool-size: 50  # prod에서 더 크게
```

---

## 📌 핵심 정리

```
파일 로딩 우선순위 (높은 순)
  application-{나중 Profile}.yml > application-{앞 Profile}.yml > application.yml

active vs include
  active  외부에서 교체 가능, 환경 전환용
  include 항상 포함, 교체 불가, 부가 기능 구성용

Profile 그룹
  spring.profiles.group.production: [proddb, prodmq]
  → production 활성화 시 하위 Profile 자동 포함

@Profile
  Bean/Configuration 레벨 등록 제어
  ProfileCondition → Environment.matchesProfiles()
  표현식 지원: !, &, | (Boot 2.1+)

application-{profile}.yml 로딩
  ConfigDataEnvironment → StandardConfigDataLocationResolver
  → active Profile 순서대로 파일 탐색 및 로딩
```

---

## 🤔 생각해볼 문제

**Q1.** `application.yml`에 `spring.profiles.active=prod`를 설정하고 운영 서버에 배포했을 때 `--spring.profiles.active=dev`로 오버라이드가 가능한가?

**Q2.** `@Profile("dev")` Bean과 `@Profile("prod")` Bean이 같은 타입(`DataSource`)인데, `@Profile` 없이 세 번째 `DataSource` Bean을 `@Bean`으로 추가하면 어떻게 되는가?

**Q3.** `application-prod.yml`에 `spring.profiles.include=monitoring`을 설정했을 때, `monitoring` Profile 활성화 순서(우선순위)는 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** 가능하다. `application.yml`로 로딩된 `spring.profiles.active`보다 커맨드라인 인수(`--spring.profiles.active`)가 훨씬 높은 우선순위를 가진다(17단계 중 1위). 그러나 이 설계는 위험하다. 운영 서버에 `spring.profiles.active=prod`가 코드에 박혀 있으면 모든 개발자가 명시적으로 오버라이드해야 한다는 것을 알아야 하고, 실수로 prod 설정이 개발 환경에 적용될 위험이 있다. 권장 방법은 기본값 Profile을 `default` 또는 `dev`로 설정하고, 운영 배포 시 외부에서 `prod`를 지정하는 것이다.
>
> **Q2.** `@Profile` 없는 세 번째 `DataSource`는 모든 Profile에서 항상 등록된다. dev Profile 활성화 시 `@Profile("dev")` DataSource와 `@Profile` 없는 DataSource 두 개가 동시에 등록되어 `NoUniqueBeanDefinitionException`이 발생한다. `@Profile` 없는 Bean은 "모든 환경에서 등록"을 의미하므로, 환경별로 다른 타입의 Bean이 필요하다면 모든 Bean에 `@Profile`을 붙이거나, `@ConditionalOnMissingBean`을 조합해야 한다.
>
> **Q3.** `application-prod.yml` 로딩 → `spring.profiles.include=monitoring` 발견 → `monitoring` Profile이 활성화됨 → `application-monitoring.yml` 로딩. 우선순위는 `application.yml` < `application-prod.yml` < `application-monitoring.yml`이다. `include`로 추가된 Profile의 파일이 가장 나중에 로딩되므로 가장 높은 우선순위를 가진다. 이 순서는 `include`를 통해 세부 설정이 상위 설정(prod)을 덮어쓸 수 있게 해주므로 `include`로 공통 기능 Profile을 추가할 때 주의해야 한다.

---

<div align="center">

**[⬅️ 이전: @Value vs @ConfigurationProperties](./05-value-vs-configuration-properties.md)** | **[홈으로 🏠](../README.md)** | **[다음: Environment PropertySource 우선순위 17단계 ➡️](./07-propertysource-priority.md)**

</div>
