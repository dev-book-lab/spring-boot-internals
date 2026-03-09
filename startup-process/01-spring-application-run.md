# SpringApplication.run() 전체 과정 — 한 줄이 실행될 때 벌어지는 일

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `SpringApplication.run(App.class, args)` 한 줄이 내부적으로 거치는 단계는 몇 개이고, 각 단계에서 무슨 일이 벌어지는가?
- `SpringApplication` 객체가 생성될 때 무엇을 초기화하는가? 어떻게 웹 애플리케이션인지 감지하는가?
- `ApplicationContext`는 언제, 어떤 타입으로 생성되는가?
- `SpringApplicationRunListener`와 `ApplicationRunner`는 어떤 차이가 있고 각각 언제 호출되는가?
- `refresh()` 이전과 이후에 하는 일의 차이는 무엇인가?
- 시작 시간이 출력되는 정확한 위치는 어디인가?

---

## 🔍 왜 이게 존재하는가

### 문제: `main()` 메서드로 Spring 애플리케이션을 시작하려면 무엇이 필요한가?

```java
// Spring Framework만 사용할 경우 — 직접 컨텍스트를 만들어야 한다
public static void main(String[] args) {
    AnnotationConfigApplicationContext ctx =
        new AnnotationConfigApplicationContext(AppConfig.class);

    // 서블릿 컨테이너도 직접 설정
    Tomcat tomcat = new Tomcat();
    tomcat.setPort(8080);
    // DispatcherServlet 등록, Context 설정...
    tomcat.start();

    ctx.registerShutdownHook();
}
```

```
문제:
  - 환경(웹/비웹/reactive) 마다 다른 컨텍스트 타입 선택 필요
  - 내장 서버 수동 설정
  - application.properties 수동 로딩
  - 배너 출력, 시작 시간 측정, 실패 시 상세 오류 분석 → 모두 직접 구현

Spring Boot의 해결:
  SpringApplication.run(App.class, args)
  → 위 모든 것을 자동으로 처리하는 단일 진입점
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: run()은 단순히 ApplicationContext를 refresh()하는 것이다

```java
// ❌ 잘못된 이해
// "SpringApplication.run()은 결국 ctx.refresh()를 호출하는 것 아닌가?"

// 실제로 run()이 하는 일 목록:
// 1. SpringApplication 인스턴스 생성 (웹 타입 감지, Initializer/Listener 로딩)
// 2. StopWatch 시작 (시작 시간 측정)
// 3. BootstrapContext 생성
// 4. 헤드리스 모드 설정
// 5. SpringApplicationRunListener 알림 (starting)
// 6. Environment 준비 (PropertySource 17단계 구성)
// 7. Banner 출력
// 8. ApplicationContext 생성 (타입 결정)
// 9. ApplicationContext 준비 (prepareContext)
// 10. ApplicationContext refresh
// 11. afterRefresh 훅
// 12. StopWatch 종료 → 시작 시간 로깅
// 13. SpringApplicationRunListener 알림 (started)
// 14. ApplicationRunner / CommandLineRunner 실행
// 15. SpringApplicationRunListener 알림 (ready)
```

### Before: `SpringApplication.run()`에 실패하면 아무 정보가 없다

```java
// ❌ 잘못된 이해
// "오류 나면 그냥 스택 트레이스만 출력된다"

// ✅ 실제:
// FailureAnalyzer가 등록되어 있어 일반적인 실패 원인을 사람이 읽기 좋게 분석
// 예: 포트 충돌 시
// ***************************
// APPLICATION FAILED TO START
// ***************************
// Description:
// Embedded server could not start. Port 8080 was already in use.
// Action:
// Identify and stop the process that's listening on port 8080...
```

---

## ✨ 올바른 이해와 사용

### After: run()은 두 단계로 나뉜다 — SpringApplication 생성 + run()

```java
// 단축 형태 (내부적으로 동일)
SpringApplication.run(App.class, args);

// 풀어쓴 형태 — 커스터마이징 가능
SpringApplication app = new SpringApplication(App.class);
app.setBannerMode(Banner.Mode.OFF);
app.addListeners(new MyApplicationListener());
app.setDefaultProperties(Map.of("server.port", "9090"));
app.run(args);
```

```
두 단계를 구분하면:

1단계: new SpringApplication(App.class)
  - 웹 환경 타입 감지 (SERVLET / REACTIVE / NONE)
  - spring.factories에서 ApplicationContextInitializer 로딩
  - spring.factories에서 ApplicationListener 로딩
  - main 클래스 추정 (스택 트레이스 분석)

2단계: app.run(args)
  - Environment 구성 → Context 생성 → refresh() → Runner 실행
  - 각 단계마다 RunListener에게 이벤트 전파
```

---

## 🔬 내부 동작 원리

### 1. SpringApplication 생성자 — 시작 전 초기화

```java
// SpringApplication.java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));

    // ① 웹 애플리케이션 타입 결정
    this.webApplicationType = WebApplicationType.deduceFromClasspath();

    // ② BootstrapRegistryInitializer 로딩 (spring.factories)
    this.bootstrapRegistryInitializers = new ArrayList<>(
        getSpringFactoriesInstances(BootstrapRegistryInitializer.class));

    // ③ ApplicationContextInitializer 로딩 (spring.factories)
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));

    // ④ ApplicationListener 로딩 (spring.factories)
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));

    // ⑤ main 클래스 추정 (스택 트레이스 분석)
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

```java
// WebApplicationType.deduceFromClasspath() — 웹 타입 감지
static WebApplicationType deduceFromClasspath() {
    // reactive 환경 감지: DispatcherHandler가 있고 DispatcherServlet이 없으면
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    // 서블릿 관련 클래스가 하나라도 없으면 일반 앱
    for (String className : SERVLET_INDICATOR_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    // 나머지는 서블릿 웹
    return WebApplicationType.SERVLET;
}

// WEBFLUX_INDICATOR_CLASS  = "org.springframework.web.reactive.DispatcherHandler"
// WEBMVC_INDICATOR_CLASS   = "org.springframework.web.servlet.DispatcherServlet"
// SERVLET_INDICATOR_CLASSES = { "jakarta.servlet.Servlet",
//                                "org.springframework.web.context.ConfigurableWebApplicationContext" }
```

```
웹 타입 감지 결과:
  SERVLET   → AnnotationConfigServletWebServerApplicationContext
  REACTIVE  → AnnotationConfigReactiveWebServerApplicationContext
  NONE      → AnnotationConfigApplicationContext
```

### 2. run() 메서드 전체 흐름

```java
// SpringApplication.run() — 핵심 흐름 (단순화)
public ConfigurableApplicationContext run(String... args) {

    // ① 시작 시간 측정 시작
    long startTime = System.nanoTime();
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    ConfigurableApplicationContext context = null;

    // ② 헤드리스 모드 설정 (AWT 없는 서버 환경)
    configureHeadlessProperty();

    // ③ SpringApplicationRunListener 목록 준비 (starting 이벤트 발행)
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);

    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

        // ④ Environment 준비 (PropertySource 구성, 프로파일 활성화)
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);

        // ⑤ Banner 출력
        Banner printedBanner = printBanner(environment);

        // ⑥ ApplicationContext 생성 (웹 타입에 따라)
        context = createApplicationContext();

        // ⑦ 시작 실패 분석기 등록
        context.setApplicationStartup(this.applicationStartup);

        // ⑧ ApplicationContext 준비 (Initializer 실행, BeanDefinition 등록)
        prepareContext(bootstrapContext, context, environment, listeners,
                       applicationArguments, printedBanner);

        // ⑨ ApplicationContext refresh (Bean 생성, 내장 서버 시작)
        refreshContext(context);

        // ⑩ refresh 후 훅 (현재는 빈 구현)
        afterRefresh(context, applicationArguments);

        // ⑪ 시작 시간 출력
        Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass)
                .logStarted(getApplicationLog(), timeTakenToStartup);
        }

        // ⑫ started 이벤트 발행
        listeners.started(context, timeTakenToStartup);

        // ⑬ ApplicationRunner / CommandLineRunner 실행
        callRunners(context, applicationArguments);

    } catch (Throwable ex) {
        // FailureAnalyzer로 실패 원인 분석 + 보기 좋은 오류 출력
        handleRunFailure(context, ex, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        // ⑭ ready 이벤트 발행 (트래픽 수신 준비 완료)
        Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
        listeners.ready(context, timeTakenToReady);
    } catch (Throwable ex) {
        handleRunFailure(context, ex, null);
        throw new IllegalStateException(ex);
    }

    return context;
}
```

### 3. prepareEnvironment() — PropertySource 구성

```java
// SpringApplication.prepareEnvironment()
private ConfigurableEnvironment prepareEnvironment(
        SpringApplicationRunListeners listeners, ...) {

    // ① 웹 타입에 맞는 Environment 생성
    //   SERVLET  → ApplicationServletEnvironment
    //   REACTIVE → ApplicationReactiveWebEnvironment
    //   NONE     → ApplicationEnvironment
    ConfigurableEnvironment environment = getOrCreateEnvironment();

    // ② 커맨드라인 인수를 PropertySource로 추가
    //    args → SimpleCommandLinePropertySource → 최우선 순위
    configureEnvironment(environment, applicationArguments.getSourceArgs());

    // ③ ConfigurationPropertySources 연결
    //    (Relaxed Binding을 위한 래퍼 추가)
    ConfigurationPropertySources.attach(environment);

    // ④ environmentPrepared 이벤트 발행
    //    → ConfigFileApplicationListener가 반응
    //    → application.properties / application.yml 로딩
    listeners.environmentPrepared(bootstrapContext, environment);

    // ⑤ defaultProperties를 PropertySource 체인 맨 뒤로 이동
    DefaultPropertiesPropertySource.moveToEnd(environment);

    // ⑥ spring.main.* 프로퍼티를 SpringApplication 객체에 바인딩
    bindToSpringApplication(environment);

    return environment;
}
```

```
prepareEnvironment() 완료 후 PropertySource 체인 예시:

순위  PropertySource 이름
 1   commandLineArgs          ← --server.port=9090 등
 2   configurationProperties  ← Relaxed Binding 래퍼
 3   servletConfigInitParams  ← (서블릿 환경)
 4   servletContextInitParams ← (서블릿 환경)
 5   systemProperties         ← -Dkey=value JVM 옵션
 6   systemEnvironment        ← OS 환경변수
 7   random                   ← ${random.int} 등
 8   applicationConfig: [classpath:/application.yml]
 9   defaultProperties        ← setDefaultProperties()로 설정한 값
```

### 4. prepareContext() — Context 준비

```java
// SpringApplication.prepareContext()
private void prepareContext(DefaultBootstrapContext bootstrapContext,
                             ConfigurableApplicationContext context,
                             ConfigurableEnvironment environment,
                             SpringApplicationRunListeners listeners,
                             ApplicationArguments applicationArguments,
                             Banner printedBanner) {

    // ① Environment를 Context에 연결
    context.setEnvironment(environment);

    // ② ApplicationContext 후처리 (BeanNameGenerator, ResourceLoader, ConversionService 등록)
    postProcessApplicationContext(context);

    // ③ ApplicationContextInitializer 실행
    //    (spring.factories에 등록된 Initializer들)
    applyInitializers(context);

    // ④ contextPrepared 이벤트 발행
    listeners.contextPrepared(context);

    // ⑤ BootstrapContext 닫기
    bootstrapContext.close(context);

    // ⑥ 시작 로그 출력 (Spring Boot 버전, 활성 프로파일)
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }

    // ⑦ 특수 Bean 등록: springApplicationArguments, springBootBanner
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }

    // ⑧ primarySources (App.class)를 BeanDefinition으로 로딩
    //    → @SpringBootApplication이 여기서 처음 처리됨
    Set<Object> sources = getAllSources();
    load(context, sources.toArray(new Object[0]));

    // ⑨ contextLoaded 이벤트 발행
    listeners.contextLoaded(context);
}
```

### 5. refreshContext() — 핵심: Bean 생성과 내장 서버 시작

```java
// SpringApplication.refreshContext()
private void refreshContext(ConfigurableApplicationContext context) {
    if (this.registerShutdownHook) {
        shutdownHook.registerApplicationContext(context);  // JVM shutdown hook 등록
    }
    refresh(context);  // → AbstractApplicationContext.refresh() 호출
}

// AbstractApplicationContext.refresh() — 12단계 (핵심만)
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {

        prepareRefresh();                     // 시작 플래그, PropertySource 초기화

        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        prepareBeanFactory(beanFactory);      // BeanPostProcessor 기본 등록

        postProcessBeanFactory(beanFactory);  // 서블릿 관련 BeanPostProcessor 추가

        // ★ BeanDefinitionRegistryPostProcessor 실행
        // → ConfigurationClassPostProcessor: @SpringBootApplication 처리
        // → @ComponentScan 실행, Auto-configuration 후보 로딩
        invokeBeanFactoryPostProcessors(beanFactory);

        // ★ BeanPostProcessor 등록 (AutowiredAnnotationBeanPostProcessor 등)
        registerBeanPostProcessors(beanFactory);

        initMessageSource();
        initApplicationEventMulticaster();

        // ★ 내장 서버 생성 및 시작 (서블릿 환경)
        // → createWebServer() → TomcatServletWebServerFactory.getWebServer()
        onRefresh();

        registerListeners();

        // ★ Non-Lazy Singleton Bean 전체 생성
        finishBeanFactoryInitialization(beanFactory);

        // ★ 내장 서버 포트 바인딩 (실제 요청 수신 시작)
        // → WebServerStartStopLifecycle.start()
        finishRefresh();
    }
}
```

```
refresh() 안에서 내장 서버가 두 번 등장하는 이유:

onRefresh():
  TomcatServletWebServerFactory.getWebServer() 호출
  Tomcat 객체 생성, DispatcherServlet 등록
  → 아직 포트 바인딩 안 됨 (요청 수신 불가)

finishRefresh():
  WebServerStartStopLifecycle.start() 호출
  → connector.start() → 포트 바인딩 완료
  → 이 시점부터 HTTP 요청 수신 가능
```

### 6. callRunners() — 시작 후 사용자 코드 실행

```java
// SpringApplication.callRunners()
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<>();

    // ApplicationRunner 수집
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    // CommandLineRunner 수집
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());

    // @Order / Ordered 인터페이스로 정렬
    AnnotationAwareOrderComparator.sort(runners);

    for (Object runner : new LinkedHashSet<>(runners)) {
        if (runner instanceof ApplicationRunner ar) {
            ar.run(args);                          // ApplicationArguments 전달
        }
        if (runner instanceof CommandLineRunner clr) {
            clr.run(args.getSourceArgs());         // String[] 전달
        }
    }
}
```

```
ApplicationRunner vs CommandLineRunner:

ApplicationRunner:
  run(ApplicationArguments args)
  → args.getOptionValues("name") 등 파싱된 형태로 접근 가능
  → --name=Alice → getOptionValues("name") = ["Alice"]

CommandLineRunner:
  run(String... args)
  → run(new String[]{"--name=Alice"}) 원시 문자열 그대로

둘 다 refresh() 완료 후, ApplicationContext가 완전히 준비된 상태에서 실행
→ 데이터 초기화, 캐시 워밍업, 시작 시 작업에 사용
```

### 7. SpringApplicationRunListener vs ApplicationListener

```java
// SpringApplicationRunListener — 스프링 시작 과정 각 단계에 끼어들 때
// spring.factories에 등록 (주로 Spring Boot 내부용)
public interface SpringApplicationRunListener {
    void starting(ConfigurableBootstrapContext bootstrapContext);
    void environmentPrepared(ConfigurableBootstrapContext bootstrapContext,
                              ConfigurableEnvironment environment);
    void contextPrepared(ConfigurableApplicationContext context);
    void contextLoaded(ConfigurableApplicationContext context);
    void started(ConfigurableApplicationContext context, Duration timeTaken);
    void ready(ConfigurableApplicationContext context, Duration timeTaken);
    void failed(ConfigurableApplicationContext context, Throwable exception);
}

// 실제 구현체: EventPublishingRunListener
// → 각 단계를 ApplicationEvent로 변환해서 발행
// starting()        → ApplicationStartingEvent
// environmentPrepared() → ApplicationEnvironmentPreparedEvent
// contextPrepared() → ApplicationContextInitializedEvent
// contextLoaded()   → ApplicationPreparedEvent
// started()         → ApplicationStartedEvent
// ready()           → ApplicationReadyEvent
// failed()          → ApplicationFailedEvent
```

```java
// ApplicationListener — 이벤트로 반응할 때 (애플리케이션 코드에서 주로 사용)
@Component
public class StartupListener implements ApplicationListener<ApplicationReadyEvent> {
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        // ApplicationContext 완전히 준비되고 Runner도 실행된 직후
        System.out.println("애플리케이션 준비 완료: " + event.getTimeTaken());
    }
}
```

```
이벤트 발행 순서:

ApplicationStartingEvent          ← 아무것도 준비 안 됨
ApplicationEnvironmentPreparedEvent ← Environment 준비됨 (properties 로딩 완료)
ApplicationContextInitializedEvent  ← Context 생성 + Initializer 실행됨
ApplicationPreparedEvent            ← BeanDefinition 로딩 완료, refresh 전
ApplicationStartedEvent             ← refresh 완료, Runner 실행 완료
ApplicationReadyEvent               ← 트래픽 수신 준비 완료
ApplicationFailedEvent              ← 시작 실패
```

---

## 💻 실험으로 확인하기

### 실험 1: 각 단계 소요 시간 측정

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(App.class);

        // ApplicationStartup으로 각 단계 추적
        app.setApplicationStartup(ApplicationStartup.DEFAULT);
        app.run(args);
    }
}
```

```
// --debug 옵션으로 실행 시 상세 로그:
// java -jar app.jar --debug

2024-01-01 12:00:00 DEBUG o.s.b.SpringApplication - Loading source class com.example.App
2024-01-01 12:00:00 DEBUG o.s.b.SpringApplication - Running with Spring Boot v3.2.0, Spring v6.1.0
...
Started App in 2.345 seconds (process running for 2.678)
```

### 실험 2: 이벤트 발행 순서 직접 확인

```java
@Component
public class StartupEventLogger {

    @EventListener(ApplicationEnvironmentPreparedEvent.class)
    public void onEnvPrepared(ApplicationEnvironmentPreparedEvent e) {
        System.out.println("[1] EnvironmentPrepared — properties 로딩 완료");
    }

    @EventListener(ApplicationContextInitializedEvent.class)
    public void onContextInit(ApplicationContextInitializedEvent e) {
        System.out.println("[2] ContextInitialized — Initializer 실행 완료");
    }

    @EventListener(ApplicationPreparedEvent.class)
    public void onPrepared(ApplicationPreparedEvent e) {
        System.out.println("[3] ApplicationPrepared — BeanDefinition 등록 완료, refresh 전");
    }

    @EventListener(ApplicationStartedEvent.class)
    public void onStarted(ApplicationStartedEvent e) {
        System.out.println("[4] ApplicationStarted — refresh 완료, Runner 실행됨");
    }

    @EventListener(ApplicationReadyEvent.class)
    public void onReady(ApplicationReadyEvent e) {
        System.out.println("[5] ApplicationReady — 트래픽 수신 준비 완료");
    }
}
```

```
주의:
  ApplicationEnvironmentPreparedEvent는 ApplicationContext가 아직 없으므로
  @Component로는 수신 불가 (컨텍스트 생성 전에 발행됨)
  → spring.factories에 ApplicationListener로 등록하거나
  → SpringApplication.addListeners()로 직접 등록해야 수신 가능
```

### 실험 3: ApplicationRunner vs CommandLineRunner 실행 순서

```java
@Component
@Order(1)
public class FirstRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        System.out.println("[Runner 1] args: " + args.getOptionNames());
    }
}

@Component
@Order(2)
public class SecondRunner implements CommandLineRunner {
    @Override
    public void run(String... args) {
        System.out.println("[Runner 2] raw args: " + Arrays.toString(args));
    }
}

// 실행: java -jar app.jar --env=prod
// 출력:
// [Runner 1] args: [env]
// [Runner 2] raw args: [--env=prod]
```

### 실험 4: 웹 타입 감지 오버라이드

```java
// classpath에 서블릿 있어도 비웹으로 강제
SpringApplication app = new SpringApplication(App.class);
app.setWebApplicationType(WebApplicationType.NONE);
app.run(args);
// → 내장 서버 시작 안 됨, AnnotationConfigApplicationContext 사용
```

---

## ⚙️ 설정 최적화 팁

```java
// 1. 배너 비활성화 (불필요한 I/O 제거)
app.setBannerMode(Banner.Mode.OFF);

// 2. 시작 로그 비활성화
app.setLogStartupInfo(false);

// 3. 전체 Lazy 초기화 (개발 환경 시작 속도 개선)
// application.yml
// spring.main.lazy-initialization: true
// → 운영 환경에서는 주의 (첫 요청 지연, 오류 조기 감지 불가)

// 4. ApplicationStartup 상세 추적 (병목 분석)
app.setApplicationStartup(new BufferingApplicationStartup(2048));
// → /actuator/startup 에서 각 단계 소요 시간 확인 가능
```

---

## 🤔 트레이드오프

```
SpringApplication의 자동화:
  장점  환경별 컨텍스트 타입 자동 선택, PropertySource 자동 구성,
        FailureAnalyzer 자동 등록, Runner 자동 실행
  단점  내부 흐름을 모르면 커스터마이징 지점을 찾기 어려움
        시작 과정이 블랙박스처럼 느껴짐

ApplicationRunner vs CommandLineRunner:
  ApplicationRunner   파싱된 인수 접근, 코드 가독성 높음
  CommandLineRunner   단순 String[] 처리, 기존 코드 재활용 쉬움

SpringApplicationRunListener vs ApplicationListener:
  RunListener   시작 과정 자체를 제어 (프레임워크 내부용)
  EventListener Bean으로 등록, 특정 이벤트에만 반응 (애플리케이션 코드용)
  → 일반 개발자는 ApplicationListener / @EventListener를 사용하는 것이 적절

lazy-initialization:
  개발 환경  시작 속도 개선 효과 큼 (수 초 단축 가능)
  운영 환경  첫 요청에 지연 스파이크 + 설정 오류가 런타임까지 숨음
             → 운영에서는 선택적 @Lazy만 사용 권장
```

---

## 📌 핵심 정리

```
SpringApplication 생성 (new 단계)
  WebApplicationType 감지 (클래스패스 기반)
  Initializer / Listener를 spring.factories에서 로딩
  main 클래스 스택 트레이스로 추정

run() 핵심 단계
  ① prepareEnvironment  → PropertySource 체인 구성 (17단계)
  ② createApplicationContext → 웹 타입에 맞는 Context 생성
  ③ prepareContext       → Initializer 실행, primarySources 로딩
  ④ refreshContext       → Bean 생성 + 내장 서버 시작
  ⑤ callRunners          → ApplicationRunner / CommandLineRunner 실행

내장 서버 시작 시점
  onRefresh()      → 서버 객체 생성, DispatcherServlet 등록
  finishRefresh()  → 포트 바인딩 (실제 요청 수신 시작)

이벤트 발행 순서
  Starting → EnvironmentPrepared → ContextInitialized →
  Prepared → Started → Ready

ApplicationRunner vs CommandLineRunner
  둘 다 refresh() 완료 후 실행
  ApplicationRunner  파싱된 인수 (getOptionValues)
  CommandLineRunner  원시 String[]
```

---

## 🤔 생각해볼 문제

**Q1.** `ApplicationEnvironmentPreparedEvent`를 `@Component` Bean에서 `@EventListener`로 받으려고 했더니 이벤트가 수신되지 않았다. 왜 그런가? 어떻게 해결해야 하는가?

**Q2.** `SpringApplication.run()` 실행 중 `refreshContext()` 단계에서 예외가 발생하면 어떤 일이 벌어지는가? `FailureAnalyzer`는 어떻게 동작하는가?

**Q3.** 동일한 Spring Boot 애플리케이션을 두 가지 방식으로 실행할 수 있다. 차이는 무엇인가?

```java
// 방식 A
SpringApplication.run(App.class, args);

// 방식 B
SpringApplication app = new SpringApplication(App.class);
app.setWebApplicationType(WebApplicationType.NONE);
app.run(args);
```

> 💡 **해설**
>
> **Q1.** `ApplicationEnvironmentPreparedEvent`는 `prepareEnvironment()` 내부에서 발행되는데, 이 시점은 아직 `ApplicationContext`가 생성되기 전이다. `@Component` Bean은 `refreshContext()` 이후에야 존재하므로, `@EventListener`가 붙은 Bean이 아직 등록되어 있지 않아 이벤트를 받을 수 없다. 해결 방법은 두 가지다. `spring.factories`(또는 Boot 3.x에서 `META-INF/spring/ApplicationListener` imports 파일)에 `ApplicationListener` 구현 클래스를 등록하거나, `SpringApplication.addListeners(new MyListener())`로 직접 등록해야 Context 생성 전부터 이벤트를 수신할 수 있다.
>
> **Q2.** `refreshContext()`에서 예외 발생 시 `handleRunFailure(context, ex, listeners)`가 호출된다. 이 메서드는 `SpringBootExceptionReporter` 목록(spring.factories에 등록된 `FailureAnalyzer`들)을 순서대로 실행해 예외의 원인을 분석한다. 예를 들어 포트 충돌 시 `PortInUseFailureAnalyzer`가 감지해 "Port 8080 was already in use" 메시지와 해결 방법을 출력한다. 분석 후에는 `ApplicationFailedEvent`를 발행하고, Context를 닫은 뒤(`context.close()`) 예외를 `IllegalStateException`으로 감싸 던진다.
>
> **Q3.** 방식 B는 `WebApplicationType.NONE`으로 설정되어 내장 서버가 시작되지 않으며 `AnnotationConfigApplicationContext`가 생성된다. 방식 A는 클래스패스에 서블릿 관련 클래스가 있으면 `WebApplicationType.SERVLET`으로 감지되어 `AnnotationConfigServletWebServerApplicationContext`가 생성되고 내장 Tomcat이 시작된다. 방식 B는 배치 처리나 CLI 도구처럼 HTTP 서버가 필요 없는 경우에 적합하다. Spring Boot 배치 애플리케이션이 `spring.main.web-application-type=none`을 자주 설정하는 이유이기도 하다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: @SpringBootApplication 3개 어노테이션 분해 ➡️](./02-springbootapplication-decompose.md)**

</div>
