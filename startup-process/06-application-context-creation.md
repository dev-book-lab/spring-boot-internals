# ApplicationContext 생성 과정 — 웹 타입에 따른 컨텍스트 선택과 refresh() 흐름

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Spring Boot는 어떤 기준으로 어떤 `ApplicationContext` 타입을 생성하는가?
- `AnnotationConfigServletWebServerApplicationContext`가 일반 `AnnotationConfigApplicationContext`와 다른 점은?
- `refresh()` 12단계 중 Spring Boot에서 중요한 단계는 어디인가?
- `onRefresh()`에서 내장 서버가 생성되는 과정은?
- `ApplicationContextInitializer`는 언제, 어떤 역할로 호출되는가?
- `GenericApplicationContext`와 `AbstractRefreshableApplicationContext`의 차이는?

---

## 🔍 왜 이게 존재하는가

### 문제: 실행 환경마다 필요한 컨텍스트 기능이 다르다

```java
// 서블릿 웹 환경
//   → 내장 Tomcat/Jetty 시작
//   → DispatcherServlet 등록
//   → WebApplicationContext 기능 (request/session scope 등)

// Reactive 웹 환경
//   → Netty 기반 내장 서버
//   → WebFlux 관련 Bean

// 일반 배치/CLI 환경
//   → 서버 없음, 순수 Bean 컨테이너만 필요
```

```
Spring Boot의 해결:
  WebApplicationType 감지 → 타입에 맞는 ApplicationContext 자동 선택
  
  SERVLET  → AnnotationConfigServletWebServerApplicationContext
  REACTIVE → AnnotationConfigReactiveWebServerApplicationContext
  NONE     → AnnotationConfigApplicationContext

각 타입이 refresh() 중 다른 동작:
  SERVLET  onRefresh() → 내장 Tomcat 생성
  REACTIVE onRefresh() → 내장 Netty 생성
  NONE     onRefresh() → 서버 생성 없음
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: ApplicationContext는 refresh() 호출 시 생성된다

```
❌ 잘못된 이해:
  "SpringApplication.run()이 refresh()를 호출하면
   그때 ApplicationContext 객체가 만들어진다"

✅ 실제:
  1. createApplicationContext() → ApplicationContext 객체 생성 (new)
  2. prepareContext()           → 환경, Initializer 설정
  3. refreshContext()           → refresh() 호출 → Bean 생성 등 실제 초기화

  객체 생성(new)과 초기화(refresh)는 분리됨
  refresh() 전에도 Context 객체는 존재하지만 아직 준비 안 된 상태
```

### Before: refresh()는 한 번만 호출할 수 있다

```java
// ❌ 잘못된 이해:
// "refresh()를 두 번 호출해도 괜찮다"

// ✅ 실제:
// GenericApplicationContext (Boot가 사용하는 타입)는 refresh() 1회만 허용
// 두 번 호출 시 IllegalStateException:
// "GenericApplicationContext does not support multiple refresh attempts"

// AbstractRefreshableApplicationContext (XML 기반)는 재refresh 가능
// → 하지만 Boot는 GenericApplicationContext 계열 사용 → 재refresh 불가
```

---

## ✨ 올바른 이해와 사용

### After: 컨텍스트 생성 3단계

```
1단계: createApplicationContext()
  WebApplicationType에 따라 클래스 인스턴스 생성 (빈 컨텍스트)
  → 내부 BeanFactory (DefaultListableBeanFactory) 초기화

2단계: prepareContext()
  Environment 연결
  ApplicationContextInitializer 실행 (커스텀 초기화)
  BeanDefinition 등록 시작 (primarySources = App.class 로딩)

3단계: refreshContext() → refresh()
  BeanDefinition 등록 완료 (ComponentScan, Auto-configuration)
  BeanPostProcessor 등록
  내장 서버 생성 및 시작
  Singleton Bean 전체 생성
```

---

## 🔬 내부 동작 원리

### 1. createApplicationContext() — 타입 결정과 생성

```java
// SpringApplication.createApplicationContext()
protected ConfigurableApplicationContext createApplicationContext() {
    return this.applicationContextFactory.create(this.webApplicationType);
}

// ApplicationContextFactory — 기본 구현
ApplicationContextFactory DEFAULT = (webApplicationType) -> {
    try {
        return switch (webApplicationType) {
            case SERVLET ->
                new AnnotationConfigServletWebServerApplicationContext();
            case REACTIVE ->
                new AnnotationConfigReactiveWebServerApplicationContext();
            default ->
                new AnnotationConfigApplicationContext();
        };
    }
    catch (Exception ex) {
        throw new IllegalStateException("...", ex);
    }
};
```

```java
// AnnotationConfigServletWebServerApplicationContext 클래스 계층
AnnotationConfigServletWebServerApplicationContext
  └── ServletWebServerApplicationContext
        └── GenericWebApplicationContext
              └── GenericApplicationContext           ← BeanFactory 보유
                    └── AbstractApplicationContext    ← refresh() 정의
                          └── ConfigurableApplicationContext
                                └── ApplicationContext

// 내장 BeanFactory: DefaultListableBeanFactory
// → GenericApplicationContext 생성자에서 new DefaultListableBeanFactory()
```

```java
// AnnotationConfigServletWebServerApplicationContext 생성자
public AnnotationConfigServletWebServerApplicationContext() {
    // AnnotatedBeanDefinitionReader: @Bean, @Configuration 처리용
    this.reader = new AnnotatedBeanDefinitionReader(this);
    // ClassPathBeanDefinitionScanner: @ComponentScan 처리용
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

### 2. prepareContext() — Context 준비

```java
// SpringApplication.prepareContext()
private void prepareContext(DefaultBootstrapContext bootstrapContext,
                             ConfigurableApplicationContext context,
                             ConfigurableEnvironment environment, ...) {

    // ① Environment 연결 — PropertySource 체인 등록
    context.setEnvironment(environment);

    // ② Context 후처리 — ConversionService, BeanNameGenerator 등록
    postProcessApplicationContext(context);

    // ③ ApplicationContextInitializer 실행
    applyInitializers(context);

    // ④ contextPrepared 이벤트 발행
    listeners.contextPrepared(context);

    // ⑤ BootstrapContext 닫기 (Bootstrap → ApplicationContext 전환)
    bootstrapContext.close(context);

    // ⑥ primarySources 로딩 — App.class의 BeanDefinition 등록
    Set<Object> sources = getAllSources();
    load(context, sources.toArray(new Object[0]));
    // → BeanDefinitionLoader.load()
    // → AnnotatedBeanDefinitionReader.register(App.class)
    // → App.class의 @SpringBootApplication이 BeanDefinition으로 등록

    // ⑦ contextLoaded 이벤트 발행
    listeners.contextLoaded(context);
}
```

```java
// ApplicationContextInitializer — 커스텀 초기화 확장점
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {
    void initialize(C applicationContext);
}

// spring.factories에 등록된 기본 Initializer (Boot 내장):
// ConfigurationWarningsApplicationContextInitializer
//   → @ComponentScan에 org, org.springframework 같은 너무 넓은 패키지 사용 시 경고
// ContextIdApplicationContextInitializer
//   → spring.application.name으로 Context ID 설정
// DelegatingApplicationContextInitializer
//   → context.initializer.classes 프로퍼티로 추가 Initializer 위임
// ServerPortInfoApplicationContextInitializer
//   → 내장 서버 시작 후 실제 포트를 Environment에 등록 (local.server.port)
```

### 3. refresh() — 12단계 핵심 흐름

```java
// AbstractApplicationContext.refresh()
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {

        // ① prepareRefresh()
        //    시작 시간 기록, PropertySource 활성화, requiredProperties 검증
        prepareRefresh();

        // ② obtainFreshBeanFactory()
        //    GenericApplicationContext: 이미 생성된 BeanFactory 반환
        //    (refreshBeanFactory() → getBeanFactory())
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // ③ prepareBeanFactory()
        //    기본 BeanPostProcessor 등록:
        //      ApplicationContextAwareProcessor (Aware 인터페이스 처리)
        //      ApplicationListenerDetector (ApplicationListener Bean 감지)
        //    기본 의존성 등록: BeanFactory, ApplicationContext, Environment
        prepareBeanFactory(beanFactory);

        // ④ postProcessBeanFactory()
        //    서블릿 환경 전용 추가:
        //      WebApplicationContextServletContextAwareProcessor 등록
        //      request/session scope 등록
        postProcessBeanFactory(beanFactory);

        // ★ ⑤ invokeBeanFactoryPostProcessors()
        //    ConfigurationClassPostProcessor 실행:
        //      @SpringBootApplication → @ComponentScan → Bean 등록
        //      Auto-configuration → DeferredImportSelector 처리
        //    이 단계에서 모든 BeanDefinition 등록 완료
        invokeBeanFactoryPostProcessors(beanFactory);

        // ★ ⑥ registerBeanPostProcessors()
        //    AutowiredAnnotationBeanPostProcessor 등록 (@Autowired 처리)
        //    CommonAnnotationBeanPostProcessor 등록 (@PostConstruct, @Resource 처리)
        //    PersistenceAnnotationBeanPostProcessor 등록 (@PersistenceContext 처리)
        registerBeanPostProcessors(beanFactory);

        // ⑦ initMessageSource() — 국제화(i18n) 지원
        initMessageSource();

        // ⑧ initApplicationEventMulticaster() — 이벤트 발행 인프라
        initApplicationEventMulticaster();

        // ★ ⑨ onRefresh() — 서브클래스별 특수 초기화
        //    ServletWebServerApplicationContext:
        //      createWebServer() → TomcatServletWebServerFactory.getWebServer()
        //      → 내장 Tomcat 인스턴스 생성, DispatcherServlet 등록
        //      → 아직 포트 바인딩 안 됨
        onRefresh();

        // ⑩ registerListeners() — ApplicationListener Bean 이벤트 멀티캐스터에 등록
        registerListeners();

        // ★ ⑪ finishBeanFactoryInitialization()
        //    preInstantiateSingletons():
        //      @Lazy 아닌 모든 Singleton Bean 생성
        //      → @Autowired 주입, @PostConstruct 실행
        finishBeanFactoryInitialization(beanFactory);

        // ★ ⑫ finishRefresh()
        //    LifecycleProcessor.onRefresh() → WebServerStartStopLifecycle.start()
        //      → 내장 서버 포트 바인딩 (실제 요청 수신 시작)
        //    ContextRefreshedEvent 발행
        finishRefresh();
    }
}
```

### 4. onRefresh() — 내장 서버 생성 (SERVLET 전용)

```java
// ServletWebServerApplicationContext.onRefresh()
@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}

private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();

    if (webServer == null && servletContext == null) {
        // ① WebServerFactory Bean 조회
        //    TomcatServletWebServerFactory (기본값)
        //    또는 JettyServletWebServerFactory / UndertowServletWebServerFactory
        ServletWebServerFactory factory = getWebServerFactory();

        // ② WebServerFactory.getWebServer() 호출
        //    → Tomcat 객체 생성
        //    → ServletContextInitializer 콜백 등록
        //    → DispatcherServlet 등록 (DispatcherServletRegistrationBean)
        this.webServer = factory.getWebServer(getSelfInitializer());

        // ③ 서버 종료를 위한 Bean 등록
        this.getBeanFactory().registerSingleton(
            "webServerGracefulShutdown",
            new WebServerGracefulShutdownLifecycle(this.webServer));
        this.getBeanFactory().registerSingleton(
            "webServerStartStop",
            new WebServerStartStopLifecycle(this, this.webServer));
        // → WebServerStartStopLifecycle은 finishRefresh()에서 start() 호출됨
    }
}
```

```java
// finishRefresh() → 포트 바인딩
// WebServerStartStopLifecycle.start()
@Override
public void start() {
    this.webServer.start();
    // → TomcatWebServer.start()
    //   → this.tomcat.start() → 포트 바인딩
    //   → "Tomcat started on port(s): 8080 (http)" 로그 출력
    this.running = true;
    this.applicationContext.publishEvent(
        new WebServerInitializedEvent(this.webServer, this.applicationContext));
    // → WebServerInitializedEvent 발행 (포트 번호 담겨 있음)
}
```

### 5. GenericApplicationContext vs AbstractRefreshableApplicationContext

```java
// GenericApplicationContext (Boot가 사용)
public class GenericApplicationContext extends AbstractApplicationContext
        implements BeanDefinitionRegistry {

    private final DefaultListableBeanFactory beanFactory;
    private boolean refreshed = false;

    @Override
    protected final void refreshBeanFactory() throws IllegalStateException {
        // refresh 중복 방지
        if (!this.refreshed.compareAndSet(false, true)) {
            throw new IllegalStateException(
                "GenericApplicationContext does not support multiple refresh attempts: " +
                "just call 'refresh' once");
        }
        // BeanFactory ID 설정
        this.beanFactory.setSerializationId(getId());
    }
}

// AbstractRefreshableApplicationContext (XML 기반 레거시)
// refresh() 호출마다 BeanFactory를 새로 생성 → 재refresh 가능
// → Boot에서는 사용하지 않음
```

---

## 💻 실험으로 확인하기

### 실험 1: 컨텍스트 타입 직접 확인

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = SpringApplication.run(App.class, args);
        System.out.println("Context 타입: " + ctx.getClass().getSimpleName());
        // 출력: AnnotationConfigServletWebServerApplicationContext

        // BeanFactory 타입
        System.out.println("BeanFactory: " +
            ((AbstractApplicationContext) ctx).getBeanFactory().getClass().getSimpleName());
        // 출력: DefaultListableBeanFactory
    }
}
```

### 실험 2: WebApplicationType.NONE으로 변경 시

```java
SpringApplication app = new SpringApplication(App.class);
app.setWebApplicationType(WebApplicationType.NONE);
ConfigurableApplicationContext ctx = app.run(args);
System.out.println(ctx.getClass().getSimpleName());
// 출력: AnnotationConfigApplicationContext
// → 내장 서버 없음, 포트 없음
```

### 실험 3: ApplicationContextInitializer 커스텀 등록

```java
// Initializer 구현
public class MyInitializer
        implements ApplicationContextInitializer<ConfigurableApplicationContext> {

    @Override
    public void initialize(ConfigurableApplicationContext ctx) {
        System.out.println("Initializer 실행: " + ctx.getClass().getSimpleName());
        // Environment에 프로퍼티 추가 가능
        ConfigurableEnvironment env = ctx.getEnvironment();
        env.getPropertySources().addFirst(
            new MapPropertySource("custom", Map.of("my.prop", "hello")));
    }
}

// 등록 방법 1: SpringApplication에 직접 추가
app.addInitializers(new MyInitializer());

// 등록 방법 2: spring.factories (또는 imports)
// org.springframework.context.ApplicationContextInitializer=com.example.MyInitializer

// 등록 방법 3: context.initializer.classes 프로퍼티
// context.initializer.classes=com.example.MyInitializer
```

### 실험 4: local.server.port로 랜덤 포트 확인

```yaml
# application.yml
server:
  port: 0  # 랜덤 포트
```

```java
@Value("${local.server.port}")
private int port;
// → ServerPortInfoApplicationContextInitializer가
//    WebServerInitializedEvent를 받아 local.server.port를 Environment에 등록
// → @Value로 주입 가능
```

---

## ⚙️ 설정 최적화 팁

```java
// 1. 시작 단계별 소요 시간 확인
SpringApplication app = new SpringApplication(App.class);
app.setApplicationStartup(new BufferingApplicationStartup(2048));
// → /actuator/startup으로 각 단계 소요 시간 확인

// 2. BeanFactory 커스터마이징 (드문 경우)
app.setApplicationContextFactory(webApplicationType -> {
    AnnotationConfigServletWebServerApplicationContext ctx =
        new AnnotationConfigServletWebServerApplicationContext();
    // BeanFactory 직접 커스터마이징
    ctx.getBeanFactory().setAllowCircularReferences(false);
    return ctx;
});

// 3. 커스텀 BeanDefinition 추가 (Initializer 활용)
public class EarlyBeanRegistrar
        implements ApplicationContextInitializer<GenericApplicationContext> {
    @Override
    public void initialize(GenericApplicationContext ctx) {
        ctx.registerBean("earlyBean", EarlyBean.class, EarlyBean::new);
    }
}
```

---

## 🤔 트레이드오프

```
GenericApplicationContext (Boot 사용):
  장점  BeanDefinitionRegistry 직접 구현 → Bean 등록이 단순
        refresh 1회 강제 → 상태 예측 가능, 실수 방지
  단점  재refresh 불가 → 동적 Bean 교체 어려움

ApplicationContextInitializer 사용:
  장점  Context 준비 단계에서 Environment 조작, Bean 선등록 가능
       spring.factories로 외부에서 주입 → 유연한 확장
  단점  너무 이른 시점 → BeanFactory 완전히 준비되지 않음
       refresh() 이후에 하는 작업은 ApplicationRunner 사용 권장

내장 서버 생성 분리 (onRefresh vs finishRefresh):
  장점  Bean 생성(finishBeanFactoryInitialization)과
       포트 바인딩(finishRefresh) 사이에 DispatcherServlet이 완전히 준비됨
       → 포트 열리자마자 요청 처리 가능한 상태 보장
  단점  onRefresh에서 생성된 서버 객체가 finishRefresh까지 유지되어야 함
```

---

## 📌 핵심 정리

```
ApplicationContext 타입 결정
  SERVLET  → AnnotationConfigServletWebServerApplicationContext
  REACTIVE → AnnotationConfigReactiveWebServerApplicationContext
  NONE     → AnnotationConfigApplicationContext
  모두 GenericApplicationContext 계열 → refresh 1회만 가능

Context 준비 3단계
  createApplicationContext()  객체 생성 (빈 BeanFactory 포함)
  prepareContext()            Environment + Initializer + primarySources 로딩
  refreshContext()            Bean 생성 + 내장 서버 시작

refresh() 핵심 단계
  ⑤ invokeBeanFactoryPostProcessors  BeanDefinition 등록 완료
  ⑥ registerBeanPostProcessors       @Autowired 등 처리기 등록
  ⑨ onRefresh                        내장 서버 객체 생성
  ⑪ finishBeanFactoryInitialization  Singleton Bean 전체 생성
  ⑫ finishRefresh                    포트 바인딩 (요청 수신 시작)

ApplicationContextInitializer
  prepareContext() 내 applyInitializers()에서 실행
  Environment 조작, 초기 Bean 등록에 활용
  spring.factories 또는 addInitializers()로 등록
```

---

## 🤔 생각해볼 문제

**Q1.** `ApplicationContextInitializer`와 `BeanFactoryPostProcessor`는 모두 `ApplicationContext` 준비 과정에서 실행된다. 각각이 실행되는 시점의 차이는 무엇이며, 각각을 사용해야 하는 경우는?

**Q2.** `AnnotationConfigServletWebServerApplicationContext`에서 `createWebServer()`가 `onRefresh()`에서 호출되고 실제 시작은 `finishRefresh()`에서 된다. 만약 순서를 바꿔 `finishBeanFactoryInitialization()` 이후에 서버를 시작하면 어떤 문제가 생기는가? (현재 구조의 장점은?)

**Q3.** Spring Boot 테스트에서 `@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)`를 사용하면 테스트 코드에서 포트 번호를 어떻게 알 수 있는가? 내부 메커니즘은?

> 💡 **해설**
>
> **Q1.** `ApplicationContextInitializer`는 `prepareContext()` 내부에서 `refresh()` 호출 전에 실행된다. 이 시점은 `BeanDefinition`이 아직 등록되지 않은 상태로, 주로 `Environment` 조작이나 아주 이른 시점의 초기화에 사용한다. `BeanFactoryPostProcessor`는 `refresh()` 내부 `invokeBeanFactoryPostProcessors()` 단계에서 실행되며, 이 시점에는 이미 모든 `BeanDefinition`이 등록된 상태다. `BeanDefinition`을 읽거나 수정하는 작업(예: `PropertySourcesPlaceholderConfigurer`의 `${...}` 치환)에 적합하다. 즉, `BeanDefinition` 등록 전 초기화 → `Initializer`, `BeanDefinition` 등록 후 수정 → `BeanFactoryPostProcessor`를 사용한다.
>
> **Q2.** 현재 구조의 핵심 장점은 `finishBeanFactoryInitialization()`이 완료된 후 포트를 여는 것이다. 즉 `DispatcherServlet`이 완전히 초기화되고, `@Controller`, `@Service` 등 모든 Bean이 생성 완료된 상태에서 포트가 열린다. 만약 `onRefresh()`에서 바로 포트를 열면 Bean이 아직 생성 중인 상태에서 HTTP 요청이 들어와 `NullPointerException`이나 불완전한 응답이 발생할 수 있다. 서버 객체 생성(`onRefresh`)과 포트 바인딩(`finishRefresh`)을 분리함으로써 "준비 완료 후 트래픽 수신"을 보장한다.
>
> **Q3.** `server.port=0`이나 `WebEnvironment.RANDOM_PORT`일 때 OS가 랜덤 포트를 배정한다. 내장 서버가 시작되면 `WebServerStartStopLifecycle.start()`에서 `WebServerInitializedEvent`를 발행하는데, `ServerPortInfoApplicationContextInitializer`가 이 이벤트를 수신해 실제 포트 번호를 `local.server.port`라는 이름으로 `Environment`의 `PropertySource`에 추가한다. 테스트 코드에서 `@Value("${local.server.port}")` 또는 `@LocalServerPort` 어노테이션으로 이 값을 주입받을 수 있다. `@LocalServerPort`는 내부적으로 `@Value("${local.server.port}")`의 단축 어노테이션이다.

---

<div align="center">

**[⬅️ 이전: @Conditional 어노테이션 평가 순서](./05-conditional-evaluation-order.md)** | **[홈으로 🏠](../README.md)** | **[다음: Banner 출력과 Startup Logging ➡️](./07-banner-and-startup-logging.md)**

</div>  
