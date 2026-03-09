# ServletWebServerFactory 동작 원리 — TomcatServletWebServerFactory가 내장 Tomcat을 초기화하는 과정

---

## 🎯 핵심 질문

- `TomcatServletWebServerFactory`는 어떤 순서로 내장 Tomcat을 초기화하는가?
- `WebServer` 인터페이스와 `start()`/`stop()` 생명주기는 어떻게 관리되는가?
- `ServletWebServerApplicationContext`가 서버 시작을 트리거하는 시점은?
- `TomcatEmbeddedWebappClassLoader`가 필요한 이유는?
- `WebServerFactoryCustomizer`는 Factory 초기화 어느 단계에서 적용되는가?

---

## 🔬 내부 동작 원리

### 1. 전체 초기화 흐름

```
SpringApplication.run()
  → createApplicationContext()
    → ServletWebServerApplicationContext 생성
  → refreshContext()
    → AbstractApplicationContext.refresh()
      → finishBeanFactoryInitialization()  // Bean 생성
      → finishRefresh()
        → ServletWebServerApplicationContext.createWebServer()
          → getWebServerFactory()          // Factory Bean 조회
          → factory.getWebServer(...)      // ← 핵심: 서버 생성
          → webServer.start()              // 포트 바인딩
        → publishEvent(ServletWebServerInitializedEvent)
          // "Started App in X seconds" 로그 출력
```

### 2. TomcatServletWebServerFactory.getWebServer()

```java
// TomcatServletWebServerFactory.java
@Override
public WebServer getWebServer(ServletContextInitializer... initializers) {
    // ① Tomcat 인스턴스 생성
    Tomcat tomcat = new Tomcat();

    // ② 작업 디렉토리 설정 (임시 디렉토리)
    File baseDir = (this.baseDirectory != null) ? this.baseDirectory
            : createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());

    // ③ Connector 생성 및 설정
    Connector connector = new Connector(this.protocol);
    // 기본 protocol: "org.apache.coyote.http11.Http11NioProtocol"
    connector.setThrowOnFailure(true);
    tomcat.getService().addConnector(connector);

    // ④ Connector 커스터마이징 적용
    customizeConnector(connector);
    tomcat.setConnector(connector);

    // ⑤ Host 설정 (기본: "localhost")
    tomcat.getHost().setAutoDeploy(false);

    // ⑥ Engine 설정
    configureEngine(tomcat.getEngine());

    // ⑦ 추가 Connector (HTTPS 등)
    for (Connector additionalConnector : this.additionalTomcatConnectors) {
        tomcat.getService().addConnector(additionalConnector);
    }

    // ⑧ Context 준비 (ServletContext 생성)
    prepareContext(tomcat.getHost(), initializers);

    // ⑨ TomcatWebServer 생성 (start는 여기서 안 함)
    return getTomcatWebServer(tomcat);
}
```

### 3. prepareContext() — ServletContext 초기화

```java
protected void prepareContext(Host host, ServletContextInitializer[] initializers) {
    // ① TomcatEmbeddedContext 생성
    TomcatEmbeddedContext context = new TomcatEmbeddedContext();
    context.setName(getContextPath());
    context.setDisplayName(getDisplayName());
    context.setPath(getContextPath());

    // ② 클래스로더 설정
    // TomcatEmbeddedWebappClassLoader — 클래스 로딩 충돌 방지
    // (내장 Tomcat의 클래스로더와 앱 클래스로더 분리)
    ClassLoader parentClassLoader = getClass().getClassLoader();
    context.setParentClassLoader(parentClassLoader);
    context.addLifecycleListener(
        new StaticResourceConfigurer(context));

    // ③ ServletContextInitializer 등록
    // → DispatcherServletRegistrationBean, FilterRegistrationBean 등 실행
    ServletContextInitializerBeans initializerBeans =
        new ServletContextInitializerBeans(getBeanFactory(), initializers);

    // ④ 기본 서블릿, 오류 처리 등 기본 설정
    configureContext(context, initializerBeans.toArray(new ServletContextInitializer[0]));

    // ⑤ Host에 Context 추가
    host.addChild(context);
}
```

### 4. TomcatWebServer — 생명주기 관리

```java
// TomcatWebServer — WebServer 구현체
public class TomcatWebServer implements WebServer {

    private final Tomcat tomcat;
    private volatile boolean started;

    // 생성자에서 이미 start() 호출 (초기화 시작)
    public TomcatWebServer(Tomcat tomcat, boolean autoStart, ...) {
        this.tomcat = tomcat;
        initialize();  // ← tomcat.start() 내부 호출
    }

    private void initialize() throws WebServerException {
        synchronized (this.monitor) {
            try {
                // Tomcat 시작 (포트 바인딩은 아직 아님)
                this.tomcat.start();

                // Context가 실패 없이 시작됐는지 확인
                checkThatConnectorsHaveStarted();

                // "Tomcat started on port(s): 8080 (http)" 로그
                logger.info("Tomcat started on port(s): " + getActualPortsDescription());
                started = true;
            } catch (Exception ex) {
                stopSilently();
                destroySilently();
                throw new WebServerException("Unable to start embedded Tomcat", ex);
            }
        }
    }

    @Override
    public void start() throws WebServerException {
        // initialize()에서 이미 시작됨 → 중복 방지
        synchronized (this.monitor) {
            if (!this.started) {
                try {
                    addPreviouslyRemovedConnectors();
                    Connector connector = this.tomcat.getConnector();
                    if (connector != null && this.autoStart) {
                        performDeferredLoadOnStartup();
                    }
                    checkThatConnectorsHaveStarted();
                    this.started = true;
                } catch (Exception ex) { ... }
            }
        }
    }

    @Override
    public void stop() throws WebServerException {
        synchronized (this.monitor) {
            boolean wasStarted = this.started;
            try {
                this.started = false;
                this.tomcat.stop();
                this.tomcat.destroy();
            } catch (Exception ex) { ... }
        }
    }

    @Override
    public int getPort() {
        Connector connector = this.tomcat.getConnector();
        if (connector != null) {
            return connector.getLocalPort();  // 실제 바인딩된 포트 (0 → 랜덤 포트)
        }
        return -1;
    }
}
```

### 5. ServletWebServerApplicationContext — 서버 시작 트리거

```java
// ServletWebServerApplicationContext.java
@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();  // ← refresh() 중 호출
    } catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}

private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();

    if (webServer == null && servletContext == null) {
        StartupStep createWebServer = getApplicationStartup()
            .start("spring.boot.webserver.create");

        // Factory Bean 조회 (TomcatServletWebServerFactory)
        ServletWebServerFactory factory = getWebServerFactory();
        createWebServer.tag("factory", factory.getClass().toString());

        // ← 서버 생성 (포트 바인딩 포함)
        this.webServer = factory.getWebServer(getSelfInitializer());
        createWebServer.end();

        // Graceful Shutdown Hook 등록
        getBeanFactory().registerSingleton(
            "webServerGracefulShutdown",
            new WebServerGracefulShutdownLifecycle(this.webServer));
        getBeanFactory().registerSingleton(
            "webServerStartStop",
            new WebServerStartStopLifecycle(this, this.webServer));
    }
    // ...
}
```

### 6. WebServerFactoryCustomizer 적용 시점

```java
// WebServerFactoryCustomizerBeanPostProcessor
// BeanPostProcessor — Factory Bean 초기화 직전에 실행

@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) {
    if (bean instanceof WebServerFactory webServerFactory) {
        postProcessBeforeInitialization(webServerFactory);
    }
    return bean;
}

private void postProcessBeforeInitialization(WebServerFactory webServerFactory) {
    // 모든 WebServerFactoryCustomizer<T> Bean 수집
    LambdaSafe.callbacks(WebServerFactoryCustomizer.class,
            getCustomizers(), webServerFactory)
        .withLogger(WebServerFactoryCustomizerBeanPostProcessor.class)
        .invoke((customizer) -> customizer.customize(webServerFactory));
}

// 적용 순서:
// Factory Bean 생성 → BeanPostProcessor 실행 → customize() 호출
// → getWebServer() 호출 (customizer 적용 후)
// → 서버 시작
```

---

## 💻 실험으로 확인하기

### 실험 1: 서버 시작 이벤트 감지

```java
@Component
public class ServerStartListener
        implements ApplicationListener<ServletWebServerInitializedEvent> {

    @Override
    public void onApplicationEvent(ServletWebServerInitializedEvent event) {
        int port = event.getWebServer().getPort();
        System.out.println("서버 시작 완료: 포트 " + port);
        // → 실제 바인딩된 포트 출력 (server.port=0 시 랜덤 포트 확인용)
    }
}
```

### 실험 2: 서버 타입 런타임 확인

```java
@Autowired
ApplicationContext context;

void checkServerType() {
    if (context instanceof ServletWebServerApplicationContext webCtx) {
        WebServer server = webCtx.getWebServer();
        System.out.println("서버: " + server.getClass().getSimpleName());
        System.out.println("포트: " + server.getPort());
    }
}
```

### 실험 3: TomcatServletWebServerFactory 직접 커스터마이징

```java
@Bean
public TomcatServletWebServerFactory tomcatFactory() {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
    factory.setPort(8080);
    factory.setContextPath("/api");
    factory.addConnectorCustomizers(connector -> {
        connector.setMaxPostSize(10 * 1024 * 1024);  // 10MB
        connector.setAttribute("compression", "on");
    });
    return factory;
}
// @ConditionalOnMissingBean(ServletWebServerFactory.class) →
// 이 Bean이 있으면 Auto-configuration의 Factory 스킵
```

---

## ⚙️ 설정 최적화 팁

```yaml
server:
  tomcat:
    # 스레드 설정
    threads:
      max: 200         # 최대 스레드 (기본)
      min-spare: 10    # 최소 유휴 스레드
    accept-count: 100  # 큐 대기 요청 수 (스레드 풀 포화 시)
    
    # 연결 설정
    max-connections: 8192     # 최대 동시 연결 수
    connection-timeout: 20000 # 연결 타임아웃 (ms)
    
    # 요청 설정
    max-http-form-post-size: 2MB  # POST body 최대 크기
    max-swallow-size: 2MB         # 응답 중단 후 body 소모 최대 크기
    
    # 액세스 로그
    accesslog:
      enabled: true
      pattern: "%h %l %u %t \"%r\" %s %b %D"
      directory: /var/log/app
```

---

## 📌 핵심 정리

```
초기화 순서
  refresh() → onRefresh() → createWebServer()
  → TomcatServletWebServerFactory.getWebServer()
  → prepareContext() → TomcatWebServer 생성 + start()
  → ServletWebServerInitializedEvent 발행

WebServerFactoryCustomizer 적용 시점
  Factory Bean 생성 직후, getWebServer() 호출 전
  BeanPostProcessor로 자동 적용

WebServer 인터페이스
  start(), stop(), getPort()
  → TomcatWebServer, JettyWebServer, UndertowWebServer
  → 서버 종류와 무관하게 동일 인터페이스

Graceful Shutdown
  webServerGracefulShutdown Bean 자동 등록
  → 종료 신호 시 새 요청 거부 + 처리 중 요청 완료 대기
```

---

## 🤔 생각해볼 문제

**Q1.** `ServletWebServerFactory` Bean을 직접 등록하면 Auto-configuration의 `TomcatServletWebServerFactory`는 어떻게 되는가?

**Q2.** `server.port=0`으로 설정할 때 실제 바인딩된 포트를 애플리케이션 코드에서 알아내는 방법은?

**Q3.** `WebServerFactoryCustomizer<TomcatServletWebServerFactory>`와 `WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>`를 둘 다 등록했을 때 어떻게 처리되는가?

> 💡 **해설**
>
> **Q1.** Auto-configuration의 `EmbeddedTomcat` 설정 클래스는 `@ConditionalOnMissingBean(value = ServletWebServerFactory.class)`를 가진다. 직접 등록한 `ServletWebServerFactory` Bean이 있으면 조건 불만족 → Auto-configuration Factory Bean이 등록되지 않는다. 직접 등록한 Factory만 사용되므로 모든 서버 설정을 직접 관리해야 한다.
>
> **Q2.** `ServletWebServerInitializedEvent`를 수신하면 `event.getWebServer().getPort()`로 실제 포트를 알 수 있다. 또는 `@Value("${local.server.port}")`를 사용하면 바인딩 완료 후 주입된 포트 번호를 얻는다. 테스트에서는 `@SpringBootTest(webEnvironment = RANDOM_PORT)`와 `@LocalServerPort`를 조합한다.
>
> **Q3.** `WebServerFactoryCustomizerBeanPostProcessor`는 `LambdaSafe.callbacks()`를 사용해 제네릭 타입을 검사한다. `WebServerFactoryCustomizer<TomcatServletWebServerFactory>`는 실제 Factory가 `TomcatServletWebServerFactory`일 때만 호출된다. `WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>`는 모든 `ConfigurableServletWebServerFactory` 구현체(Tomcat, Jetty, Undertow 모두)에 적용된다. 두 Customizer가 모두 등록되어 있으면 둘 다 적용되며, `@Order`로 순서를 제어할 수 있다.

---

<div align="center">

**[⬅️ 이전: Tomcat vs Jetty vs Undertow](./01-tomcat-jetty-undertow.md)** | **[홈으로 🏠](../README.md)** | **[다음: Embedded Server 커스터마이징 ➡️](./03-embedded-server-customization.md)**

</div>
