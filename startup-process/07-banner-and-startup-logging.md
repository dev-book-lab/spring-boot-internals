# Banner 출력과 Startup Logging — 시작 로그에서 읽을 수 있는 것들

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Spring Boot Banner는 어떤 순서로 탐색되고 어떤 파일 포맷을 지원하는가?
- `SpringApplicationBannerPrinter`는 내부에서 어떻게 동작하는가?
- `StartupInfoLogger`가 측정하는 시작 시간은 정확히 어떤 시점부터 어떤 시점까지인가?
- 시작 로그의 각 필드(Spring Boot 버전, PID, 프로파일 등)는 어디서 가져오는가?
- `ApplicationStartup`으로 단계별 소요 시간을 어떻게 측정하는가?
- 운영 환경에서 배너와 시작 로그를 어떻게 제어해야 하는가?

---

## 🔍 왜 이게 존재하는가

### Banner — 시각적 피드백과 버전 정보

```
Spring Boot 기본 배너:
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.0)
```

```
배너의 역할:
  1. 시각적 확인 — 어떤 애플리케이션이 시작됐는지 빠른 확인
  2. 버전 정보 — Spring Boot 버전, Java 버전, 애플리케이션 버전 등
  3. 커스텀 브랜딩 — 팀/서비스별 고유 배너로 구분

시작 로그의 역할:
  PID, 호스트명, 포트, 프로파일, 시작 시간 등
  → 로그 분석 시 이 애플리케이션 인스턴스를 식별하는 기준
  → 운영 환경에서 배포 확인, 장애 시 컨텍스트 파악에 중요
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Banner 파일은 resources 루트에만 배치할 수 있다

```
❌ 잘못된 이해:
  "banner.txt를 src/main/resources/에 놓으면 됩니다"

✅ 실제:
  기본 탐색 경로는 classpath 루트 (banner.txt, banner.gif, banner.jpg, banner.png)
  spring.banner.location으로 경로 변경 가능
  
  지원 포맷:
    .txt  → 텍스트 아트 (변수 치환 가능)
    .gif  → 애니메이션 GIF (각 프레임을 ASCII로 변환해 순서대로 출력)
    .jpg  → 이미지를 ASCII 아트로 변환해 출력
    .png  → 이미지를 ASCII 아트로 변환해 출력
  
  이미지 배너의 경우:
    AnsiOutput이 비활성화된 환경에서는 그레이스케일 ASCII로 출력
    터미널에 따라 보이는 품질 차이 큼
```

### Before: ${spring-boot.version} 같은 변수를 직접 쓸 수 없다

```
❌ 잘못된 이해:
  "배너 파일은 순수 텍스트라 동적 내용 추가 불가"

✅ 실제:
  .txt 배너에서 다음 변수 사용 가능:

  ${spring-boot.version}         Spring Boot 버전 (예: 3.2.0)
  ${spring-boot.formatted-version} 포맷된 버전 (예: (v3.2.0))
  ${spring.version}              Spring Framework 버전
  ${application.version}         Manifest의 Implementation-Version
  ${application.formatted-version} 포맷된 앱 버전
  ${application.title}           Manifest의 Implementation-Title
  ${AnsiColor.BRIGHT_GREEN}      ANSI 색상 코드
  ${AnsiBackground.RED}          ANSI 배경색
  ${AnsiStyle.BOLD}              ANSI 스타일
```

---

## ✨ 올바른 이해와 사용

### After: 배너 탐색 우선순위와 시작 로그 구성

```
배너 탐색 순서 (위에서 아래로, 먼저 찾으면 사용):

1. spring.banner.image.location 프로퍼티 지정 경로 (이미지)
2. spring.banner.location 프로퍼티 지정 경로 (텍스트)
3. classpath:banner.gif (이미지)
4. classpath:banner.jpg (이미지)
5. classpath:banner.png (이미지)
6. classpath:banner.txt (텍스트)
7. 기본 Spring Boot 배너 (하드코딩)

시작 로그 예시:
2024-01-15T10:30:00.123+09:00  INFO 12345 --- [main] com.example.App :

  .   ____          _            __ _ _
  ...배너...

2024-01-15T10:30:00.456+09:00  INFO 12345 --- [main] com.example.App :
  Started App in 2.345 seconds (process running for 2.678)
```

---

## 🔬 내부 동작 원리

### 1. SpringApplicationBannerPrinter — 배너 탐색과 출력

```java
// SpringApplicationBannerPrinter.java
class SpringApplicationBannerPrinter {

    static final String DEFAULT_BANNER_LOCATION = "banner.txt";
    static final String[] IMAGE_EXTENSION = { "gif", "jpg", "png" };

    private final ResourceLoader resourceLoader;
    private final Banner fallbackBanner;  // 기본 Spring Boot 배너

    Banner print(Environment environment, Class<?> sourceClass, PrintStream out) {
        Banner banner = getBanner(environment);
        banner.printBanner(environment, sourceClass, out);
        return new PrintedBanner(banner, sourceClass);
    }

    private Banner getBanner(Environment environment) {
        Banners banners = new Banners();

        // ① 이미지 배너 탐색
        banners.addIfNotNull(getImageBanner(environment));
        // ② 텍스트 배너 탐색
        banners.addIfNotNull(getTextBanner(environment));

        if (banners.hasAtLeastOneBanner()) {
            return banners;  // 찾은 배너 반환
        }
        if (this.fallbackBanner != null) {
            return this.fallbackBanner;
        }
        return DEFAULT_BANNER;  // 기본 Spring Boot 배너
    }

    private Banner getTextBanner(Environment environment) {
        // spring.banner.location 또는 기본 banner.txt
        String location = environment.getProperty(BANNER_LOCATION_PROPERTY,
                                                    DEFAULT_BANNER_LOCATION);
        Resource resource = this.resourceLoader.getResource(location);
        try {
            if (resource.exists() && !resource.getURL().toExternalForm().contains("liquibase-core")) {
                return new ResourceBanner(resource);
            }
        } catch (IOException ex) { }
        return null;
    }
}
```

```java
// ResourceBanner — 텍스트 배너 출력 (변수 치환 포함)
class ResourceBanner implements Banner {

    @Override
    public void printBanner(Environment environment, Class<?> sourceClass,
                             PrintStream out) {
        String banner = StreamUtils.copyToString(this.resource.getInputStream(),
                                                   StandardCharsets.UTF_8);

        // 변수 치환
        for (PropertyResolver resolver : getPropertyResolvers(environment, sourceClass)) {
            banner = resolver.resolvePlaceholders(banner);
        }

        // ANSI 색상 처리 후 출력
        out.println(AnsiOutput.toString(banner));
    }

    private List<PropertyResolver> getPropertyResolvers(Environment environment,
                                                          Class<?> sourceClass) {
        List<PropertyResolver> resolvers = new ArrayList<>();

        // Environment PropertySource (application.properties 값 등)
        resolvers.add(environment);

        // 앱 버전 정보 (Manifest 기반)
        resolvers.add(new MutablePropertySources(
            getVersionResolvers(sourceClass)));

        // AnsiPropertySource (색상 변수)
        resolvers.add(new AnsiPropertySource("ansi", true));
        return resolvers;
    }
}
```

### 2. SpringApplication의 배너 출력 시점

```java
// SpringApplication.run() 내 배너 출력 위치
public ConfigurableApplicationContext run(String... args) {
    // ...
    ConfigurableEnvironment environment = prepareEnvironment(...);  // ① Environment 준비

    Banner printedBanner = printBanner(environment);  // ② 배너 출력 ← 여기

    context = createApplicationContext();  // ③ Context 생성
    // ...
}

// printBanner()
private Banner printBanner(ConfigurableEnvironment environment) {
    if (this.bannerMode == Banner.Mode.OFF) {
        return null;
    }
    ResourceLoader resourceLoader = (this.resourceLoader != null)
        ? this.resourceLoader : new DefaultResourceLoader(null);

    SpringApplicationBannerPrinter bannerPrinter =
        new SpringApplicationBannerPrinter(resourceLoader, this.banner);

    if (this.bannerMode == Banner.Mode.LOG) {
        // 로그 파일에 출력 (TRACE 레벨)
        return bannerPrinter.print(environment, this.mainApplicationClass, logger);
    }
    // 콘솔에 출력 (System.out)
    return bannerPrinter.print(environment, this.mainApplicationClass, System.out);
}
```

```
배너 출력 시점:
  prepareEnvironment() 완료 후
  createApplicationContext() 전
  
  → Environment가 준비되어 있어 변수 치환 가능
  → 배너는 항상 맨 먼저 보임 (Context 준비 전)
```

### 3. StartupInfoLogger — 시작 시간 측정

```java
// SpringApplication.run() 내 시간 측정
public ConfigurableApplicationContext run(String... args) {
    long startTime = System.nanoTime();  // ← 시작 시점 기록

    // ... (전체 처리 과정) ...

    refreshContext(context);
    afterRefresh(context, applicationArguments);

    // 시작 시간 계산
    Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);

    if (this.logStartupInfo) {
        new StartupInfoLogger(this.mainApplicationClass)
            .logStarted(getApplicationLog(), timeTakenToStartup);
    }
    // ...
}
```

```java
// StartupInfoLogger.logStarted()
void logStarted(Log applicationLog, Duration timeTakenToStartup) {
    if (applicationLog.isInfoEnabled()) {
        applicationLog.info(getStartedMessage(timeTakenToStartup));
    }
}

private CharSequence getStartedMessage(Duration timeTakenToStartup) {
    StringBuilder message = new StringBuilder();
    message.append("Started ");
    message.append(getApplicationName());           // 클래스명 (App)

    message.append(" in ");
    message.append(timeTakenToStartup.toMillis() / 1000.0);  // 초 단위
    message.append(" seconds");

    Long uptimeMs = ManagementFactory.getRuntimeMXBean().getUptime();
    if (uptimeMs != null) {
        double uptime = uptimeMs / 1000.0;
        message.append(" (process running for ");
        message.append(uptime);   // JVM 시작부터의 전체 업타임
        message.append(")");
    }
    return message;
}
```

```
시작 시간 측정 범위:

"X seconds" = SpringApplication.run() 시작 ~ refresh() + Runner 실행 완료
  → prepareEnvironment + createContext + refresh + callRunners 전체

"process running for Y" = JVM 프로세스 시작(main() 진입) ~ 동일 완료 시점
  → JVM 부팅 시간 포함 (ClassLoader 초기화 등)

Y > X 인 이유:
  JVM 자체 시작 시간은 SpringApplication.run() 이전에 발생
  → Y - X = JVM 부팅 오버헤드
```

### 4. ApplicationStartup — 단계별 성능 측정

```java
// ApplicationStartup 인터페이스
public interface ApplicationStartup {
    // 기본 구현: NoOp (아무 측정 안 함, 오버헤드 없음)
    ApplicationStartup DEFAULT = new DefaultApplicationStartup();

    StartupStep start(String name);  // 측정 시작
}

// StartupStep — 단계 측정 단위
public interface StartupStep {
    StartupStep tag(String key, String value);   // 태그 추가
    StartupStep tag(String key, Supplier<String> value);
    void end();                                    // 측정 종료
}

// BufferingApplicationStartup — 측정값 보관
// /actuator/startup으로 조회 가능
SpringApplication app = new SpringApplication(App.class);
app.setApplicationStartup(new BufferingApplicationStartup(2048));
```

```java
// 스프링이 내부적으로 ApplicationStartup을 사용하는 예시
// AbstractApplicationContext.refresh() 내부
StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

// invokeBeanFactoryPostProcessors
StartupStep bfppStep = this.applicationStartup.start("spring.context.beans.post-process");
invokeBeanFactoryPostProcessors(beanFactory);
bfppStep.end();

// finishBeanFactoryInitialization
StartupStep beanInstantiation = this.applicationStartup.start("spring.beans.instantiate")
    .tag("beanName", beanName);
getBean(beanName);
beanInstantiation.end();

contextRefresh.end();
```

```
/actuator/startup 응답 예시:
{
  "springBootVersion": "3.2.0",
  "timeline": {
    "startTime": "2024-01-15T01:30:00Z",
    "events": [
      {
        "startTime": "2024-01-15T01:30:00.001Z",
        "endTime":   "2024-01-15T01:30:00.012Z",
        "duration":  "PT0.011S",
        "startupStep": {
          "name": "spring.context.beans.post-process",
          "id": 1,
          "tags": []
        }
      },
      {
        "startupStep": {
          "name": "spring.beans.instantiate",
          "tags": [{"key": "beanName", "value": "dataSource"}]
        },
        "duration": "PT0.230S"   ← DataSource 생성에 230ms
      }
    ]
  }
}
```

### 5. 시작 로그 전체 구조

```
2024-01-15T10:30:00.000+09:00  INFO 12345 --- [           main] com.example.App :

  .   ____          _            __ _ _    ← 배너
 ...

2024-01-15T10:30:00.123+09:00  INFO 12345 --- [           main] com.example.App :
  Starting App using Java 17.0.9 with PID 12345 (/app/app.jar started by appuser in /app)

2024-01-15T10:30:00.124+09:00  INFO 12345 --- [           main] com.example.App :
  The following 1 profile is active: "prod"

  ... (auto-configuration, bean 생성 로그) ...

2024-01-15T10:30:02.345+09:00  INFO 12345 --- [           main] o.s.b.w.e.tomcat.TomcatWebServer :
  Tomcat started on port 8080 (http) with context path ''

2024-01-15T10:30:02.400+09:00  INFO 12345 --- [           main] com.example.App :
  Started App in 2.345 seconds (process running for 2.678)
```

```java
// "Starting App using Java 17.0.9 with PID 12345..." 로그 생성
// StartupInfoLogger.logStarting()
private CharSequence getStartingMessage() {
    StringBuilder message = new StringBuilder();
    message.append("Starting ");
    appendApplicationName(message);      // 클래스명
    appendVersion(message, this.mainApplicationClass);  // 앱 버전 (Manifest)
    appendJavaVersion(message);          // Java 버전
    appendPid(message);                  // PID
    appendContext(message);              // JAR 경로, 실행 사용자, 작업 디렉토리
    return message;
}

private void appendPid(StringBuilder message) {
    // ProcessHandle.current().pid() (Java 9+)
    // 또는 ManagementFactory.getRuntimeMXBean().getName() 파싱
    String pid = ApplicationPid.get().toString();
    message.append(" with PID ").append(pid);
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 커스텀 배너 작성

```
// src/main/resources/banner.txt

${AnsiColor.BRIGHT_CYAN}
  __  __        _                       
 |  \/  |  _  | |    _    _     _      
 | |\/| | | | | |   | |  | |  / _|   
 | |  | | |_| | |___| |  | | |  _|   
 |_|  |_|\  _/|_____|_|  |_| |_|     
          | |                          
          |_|   ${AnsiColor.BRIGHT_WHITE}MY-SERVICE${AnsiColor.DEFAULT}

  Spring Boot ${spring-boot.formatted-version}
  Java ${java.version}
  Profile: ${spring.profiles.active:default}
${AnsiColor.DEFAULT}
```

### 실험 2: 배너 비활성화 방법들

```yaml
# application.yml
spring:
  main:
    banner-mode: off     # 완전 비활성화
    # banner-mode: log   # 콘솔 대신 로그 파일에 출력 (운영 환경 권장)
    # banner-mode: console # 기본값
```

```java
// 코드로 설정
SpringApplication app = new SpringApplication(App.class);
app.setBannerMode(Banner.Mode.OFF);
```

### 실험 3: BufferingApplicationStartup으로 병목 찾기

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(App.class);
        app.setApplicationStartup(new BufferingApplicationStartup(2048));
        app.run(args);
    }
}
```

```bash
# /actuator/startup 조회 (actuator 의존성 필요)
curl http://localhost:8080/actuator/startup | jq '.timeline.events
  | sort_by(.duration)
  | reverse
  | .[:5]
  | .[] | {name: .startupStep.name, duration: .duration, bean: .startupStep.tags}'
```

```
출력 예시 (소요 시간 상위 5개):
  spring.beans.instantiate / dataSource         / 0.230s  ← 느린 Bean 발견
  spring.beans.instantiate / entityManagerFactory / 0.180s
  spring.context.beans.post-process              / 0.050s
  spring.beans.instantiate / tomcatEmbeddedWebServer / 0.040s
  spring.beans.instantiate / dispatcherServlet   / 0.010s
```

### 실험 4: 시작 로그 읽기

```
Started App in 2.345 seconds (process running for 2.678)
                ↑                                  ↑
           SpringApplication.run() 소요      JVM 시작부터 전체 시간
           (2.345초)                          (2.678초)

차이 0.333초 = JVM 부팅 오버헤드
  → 이 값이 크면: static initializer, class loading 문제 의심
  → Native Image로 전환하면 process 시간도 극적으로 감소 (수십 ms)
```

---

## ⚙️ 설정 최적화 팁

```yaml
# 운영 환경 권장 설정
spring:
  main:
    banner-mode: log          # 배너를 로그 파일에만 출력, 콘솔 오염 방지
    log-startup-info: true    # 시작 정보 로그 유지 (false로 하면 PID 등 누락)

logging:
  level:
    org.springframework.boot.autoconfigure: WARN  # Auto-configuration 로그 최소화
    # INFO가 기본값이나 운영에서는 WARN으로 줄여도 됨
```

```java
// 커스텀 배너를 Bean으로 등록
@Bean
public Banner banner() {
    return (environment, sourceClass, out) -> {
        out.println("== " + environment.getProperty("spring.application.name") + " ==");
        out.println("== Version: " + BuildInfo.VERSION + " ==");
    };
}

// SpringApplication에 직접 설정
app.setBanner((env, src, out) -> out.println("MY-SERVICE v" + BuildInfo.VERSION));
```

---

## 🤔 트레이드오프

```
배너 출력:
  장점  시작 확인 편의, 버전 정보 빠른 파악
  단점  로그 파일 오염 (특히 배포 자동화 환경), 의미 없는 I/O

운영 환경 권장:
  banner-mode: log → 로그 파일에 한 번만 기록
  또는 banner-mode: off → 완전 제거

log-startup-info:
  true   PID, 호스트, 포트, 프로파일 등 운영 필수 정보 포함
  false  로그 깔끔하지만 장애 분석 시 컨텍스트 부족
  → 운영 환경에서는 true 유지 강력 권장

BufferingApplicationStartup:
  장점  시작 병목 Bean 정확히 파악 가능
  단점  버퍼 메모리 사용, 약간의 성능 오버헤드
  → 개발/스테이징 환경에서 프로파일링용으로 사용
  → 운영 환경: DEFAULT (NoOp) 사용
```

---

## 📌 핵심 정리

```
배너 탐색 순서
  spring.banner.image.location → spring.banner.location
  → classpath:banner.{gif,jpg,png} → classpath:banner.txt
  → 기본 Spring Boot 배너

배너 변수 치환
  ${spring-boot.version}, ${AnsiColor.RED} 등
  Environment PropertyResolver + AnsiPropertySource로 처리

배너 출력 시점
  prepareEnvironment() 완료 후, createApplicationContext() 전

StartupInfoLogger 측정 범위
  "X seconds"          = run() 시작 ~ refresh() + Runner 완료
  "process running Y"  = JVM 프로세스 시작부터

ApplicationStartup
  DEFAULT            NoOp (운영 환경)
  BufferingApplicationStartup  단계별 소요 시간 수집 (개발/프로파일링)
  → /actuator/startup로 Bean별 초기화 시간 확인

시작 로그 필드
  PID: ProcessHandle.current().pid()
  Java 버전: System.getProperty("java.version")
  JAR 경로, 실행 사용자: RuntimeMXBean
```

---

## 🤔 생각해볼 문제

**Q1.** `banner-mode: log`로 설정하면 배너가 어떤 로그 레벨로 출력되는가? 로그 레벨을 조정해 배너가 보이지 않게 하려면 어떻게 하는가?

**Q2.** `BufferingApplicationStartup`의 버퍼 크기를 2048로 설정했을 때, 측정 이벤트가 2048개를 초과하면 어떻게 되는가?

**Q3.** CI/CD 파이프라인에서 `Started App in 2.345 seconds`를 파싱해 시작 시간을 모니터링하고 싶다. 더 견고한 방법이 있는가?

> 💡 **해설**
>
> **Q1.** `banner-mode: log`일 때 배너는 `TRACE` 레벨로 로그에 출력된다. `SpringApplicationBannerPrinter.print()`가 `Logger`를 받는 오버로드를 호출하며, 내부에서 `logger.trace(banner)`를 실행한다. 따라서 `logging.level.root=DEBUG`로도 배너가 보이지 않는다. `logging.level.org.springframework.boot=TRACE` 또는 전체 로그를 TRACE로 설정해야 볼 수 있다. 운영 환경에서 `banner-mode: log`를 쓰면 배너는 기록되지만 일반적으로 보이지 않아 로그 가독성을 해치지 않는다.
>
> **Q2.** `BufferingApplicationStartup`은 내부적으로 고정 크기 링 버퍼를 사용한다. 버퍼가 가득 차면 가장 오래된 이벤트부터 덮어쓴다(ring buffer 방식). 따라서 버퍼 크기를 초과하면 초기 단계의 이벤트가 손실될 수 있다. 애플리케이션 시작 단계가 많은 경우 버퍼 크기를 늘려야 전체 타임라인을 볼 수 있다. 이벤트 손실이 발생했는지는 `BufferingApplicationStartup.getBufferedTimeline().isFiltered()` 메서드로 확인할 수 있다.
>
> **Q3.** 로그 텍스트 파싱보다 `/actuator/startup` 또는 `/actuator/metrics`를 사용하는 것이 더 견고하다. Micrometer는 `application.started.time`과 `application.ready.time`을 자동으로 메트릭으로 수집한다. Prometheus + Grafana를 연동하거나 `/actuator/metrics/application.started.time`을 직접 호출해 정확한 시작 시간을 숫자로 가져올 수 있다. 로그 텍스트 파싱은 로그 포맷 변경 시 깨질 수 있으므로 메트릭 엔드포인트 활용이 권장된다.

---

<div align="center">

**[⬅️ 이전: ApplicationContext 생성 과정](./06-application-context-creation.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — @ConditionalOnClass 동작 메커니즘 ➡️](../auto-configuration/01-conditional-on-class.md)**

</div>
