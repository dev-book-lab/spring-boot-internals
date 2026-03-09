# Context Path & Port 설정 — 랜덤 포트 원리, 다중 포트, DispatcherServlet 매핑

---

## 🎯 핵심 질문

- `server.port=0`으로 설정하면 OS가 랜덤 포트를 할당하는 원리는?
- 할당된 랜덤 포트를 애플리케이션 코드에서 얻는 방법은?
- `server.context-path`가 `DispatcherServlet` 경로 매핑에 미치는 영향은?
- 다중 포트 리스닝(HTTP + HTTPS 동시)은 어떻게 구현하는가?
- Actuator의 `management.server.port` 분리가 별도 컨텍스트를 생성하는 이유는?

---

## 🔬 내부 동작 원리

### 1. server.port=0 — 랜덤 포트 할당 원리

```java
// 포트 0의 의미:
// ServerSocket(0) → OS 커널이 사용 가능한 포트 자동 할당
// → 에페머럴 포트 범위에서 선택 (Linux: 32768~60999)
// → 바인딩 완료 후 getLocalPort()로 실제 포트 확인

// Tomcat 내부 처리:
// NioEndpoint.bind()
//   → serverSock = ServerSocketChannel.open()
//   → serverSock.socket().bind(addr)  // addr.port = 0
//   → 바인딩 완료 후 실제 포트 결정
//   → localPort = serverSock.socket().getLocalPort()

// WebServer.getPort() 구현:
public class TomcatWebServer implements WebServer {
    @Override
    public int getPort() {
        Connector connector = this.tomcat.getConnector();
        if (connector != null) {
            return connector.getLocalPort();  // 바인딩 후 실제 포트
        }
        return -1;
    }
}
```

### 2. 랜덤 포트를 코드에서 얻는 방법

```java
// 방법 1: ApplicationListener<WebServerInitializedEvent>
@Component
public class PortDiscoverer
        implements ApplicationListener<WebServerInitializedEvent> {

    private int port;

    @Override
    public void onApplicationEvent(WebServerInitializedEvent event) {
        this.port = event.getWebServer().getPort();
        System.out.println("바인딩된 포트: " + port);
    }

    public int getPort() { return port; }
}

// 방법 2: @LocalServerPort (테스트에서 주로 사용)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class MyControllerTest {

    @LocalServerPort  // = @Value("${local.server.port}")
    private int port;

    @Autowired
    TestRestTemplate restTemplate;

    @Test
    void test() {
        String url = "http://localhost:" + port + "/api/hello";
        ResponseEntity<String> response = restTemplate.getForEntity(url, String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}

// 방법 3: Environment 직접 조회
@Autowired Environment environment;

String port = environment.getProperty("local.server.port");
// → 서버 시작 후 Environment에 "local.server.port" 자동 등록됨
```

```java
// local.server.port 등록 시점:
// WebServerStartedCondition → WebServerInitializedEvent 수신
//   → localPort = event.getWebServer().getPort()
//   → environment.getPropertySources().addFirst(
//       new MapPropertySource("server.ports",
//           Map.of("local.server.port", localPort)));
```

### 3. Context Path 동작 원리

```yaml
server:
  port: 8080
  servlet:
    context-path: /api  # Context Path 설정
```

```
Context Path 설정 효과:

설정 없음 (기본 "/"):
  URL: http://localhost:8080/users
  DispatcherServlet 매핑: "/"
  → @RequestMapping("/users") → /users

Context Path "/api":
  URL: http://localhost:8080/api/users
  DispatcherServlet 매핑: "/"  (Context Path 기준 상대 경로)
  → @RequestMapping("/users") → /api/users
  → @RequestMapping("/api/users") → /api/api/users (잘못된 설정)

구분:
  Context Path: 서블릿 컨테이너 레벨 경로 접두사
  → HttpServletRequest.getContextPath() = "/api"
  Spring MVC 경로: DispatcherServlet 레벨
  → HttpServletRequest.getServletPath() = "/users"
```

```java
// Context Path 적용 내부:
// TomcatEmbeddedContext.setPath("/api")
// → 모든 URL이 "/api"로 시작해야 컨텍스트로 라우팅
// → DispatcherServlet은 Context Path 이후의 경로를 처리
// → @RequestMapping 경로는 Context Path 이후 경로로 매핑

// 실용 예시:
@RestController
@RequestMapping("/users")  // → /api/users (context-path=/api일 때)
public class UserController {

    @GetMapping("/{id}")   // → /api/users/{id}
    public User getUser(@PathVariable Long id) { ... }
}

// URL 생성 시 Context Path 포함:
// UriComponentsBuilder.fromCurrentRequest() → 자동 포함
// @{/users} (Thymeleaf) → /api/users 자동 생성
```

### 4. 다중 포트 리스닝 (HTTP + HTTPS 동시)

```java
// Tomcat: 추가 Connector로 HTTP 포트 추가
@Component
public class MultiPortCustomizer
        implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Value("${server.http.port:8080}")
    private int httpPort;

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        // HTTP Connector 추가 (기본 포트는 HTTPS)
        Connector httpConnector =
            new Connector(TomcatServletWebServerFactory.DEFAULT_PROTOCOL);
        httpConnector.setScheme("http");
        httpConnector.setPort(httpPort);
        httpConnector.setSecure(false);
        httpConnector.setRedirectPort(8443);  // HTTP → HTTPS 리다이렉트 포트

        factory.addAdditionalTomcatConnectors(httpConnector);
    }
}
```

```yaml
# 설정
server:
  port: 8443          # 기본 포트 (HTTPS)
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: changeit
  http:
    port: 8080        # 커스텀 프로퍼티 (HTTP)
```

```java
// HTTP → HTTPS 강제 리다이렉트
@Configuration
public class HttpsRedirectConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .requiresChannel(channel ->
                channel.anyRequest().requiresSecure())  // 모든 요청 HTTPS 요구
            .build();
    }
}

// 또는 Tomcat 레벨에서 리다이렉트 규칙 (Spring Security 없이)
factory.addContextCustomizers(context -> {
    SecurityConstraint constraint = new SecurityConstraint();
    constraint.setUserConstraint("CONFIDENTIAL");  // HTTPS 요구
    SecurityCollection collection = new SecurityCollection();
    collection.addPattern("/*");
    constraint.addCollection(collection);
    context.addConstraint(constraint);
});
```

### 5. management.server.port — 별도 컨텍스트 생성 원리

```java
// ManagementContextAutoConfiguration 처리 흐름:
// management.server.port가 server.port와 다를 때:

// ① 별도 WebServerApplicationContext 생성
// ChildManagementContextInitializer
//   → new AnnotationConfigServletWebServerApplicationContext() 생성
//   → 부모 컨텍스트(앱)의 Bean 접근 가능
//   → 별도 TomcatServletWebServerFactory (port=8081)
//   → 별도 DispatcherServlet 인스턴스

// ② 두 서버의 독립성:
// 앱 서버 (8080): @RequestMapping Bean 처리
// 관리 서버 (8081): Actuator Endpoint만 처리
//   → 앱 서버 장애가 관리 서버에 영향 없음 (별도 스레드 풀)
//   → 방화벽으로 8081은 내부망에서만 접근

// ③ management.server.port = -1:
// 별도 서버 없음 → Actuator HTTP Endpoint 비활성화
// JMX Actuator는 영향 없음
```

### 6. server.port 관련 환경 변수 / 시스템 프로퍼티

```bash
# 포트 설정 우선순위 (높은 것이 이김):
# 1. 커맨드라인 인수
java -jar app.jar --server.port=9090

# 2. 환경변수 (Relaxed Binding 적용)
SERVER_PORT=9090 java -jar app.jar

# 3. 시스템 프로퍼티
java -Dserver.port=9090 -jar app.jar

# 4. application.yml
server:
  port: 8080

# 5. 기본값: 8080

# Docker / Kubernetes 환경에서:
# → 환경변수로 포트 오버라이드가 일반적
docker run -e SERVER_PORT=8080 myapp:latest
```

---

## 💻 실험으로 확인하기

### 실험 1: 랜덤 포트 + 포트 주입 테스트

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class RandomPortTest {

    @LocalServerPort
    int port;

    @Autowired
    TestRestTemplate restTemplate;

    @Test
    void randomPortIsAssigned() {
        assertThat(port).isGreaterThan(0);
        assertThat(port).isLessThan(65536);

        String response = restTemplate
            .getForObject("http://localhost:" + port + "/actuator/health", String.class);
        assertThat(response).contains("UP");
    }
}
```

### 실험 2: Context Path 효과 확인

```java
@SpringBootTest(
    properties = "server.servlet.context-path=/myapp",
    webEnvironment = WebEnvironment.RANDOM_PORT
)
class ContextPathTest {

    @Autowired TestRestTemplate restTemplate;

    @Test
    void contextPathPrefixedUrl() {
        // /myapp/api/hello → 200
        ResponseEntity<String> ok = restTemplate
            .getForEntity("/myapp/api/hello", String.class);
        assertThat(ok.getStatusCode()).isEqualTo(HttpStatus.OK);

        // /api/hello (context path 없이) → 404
        ResponseEntity<String> notFound = restTemplate
            .getForEntity("/api/hello", String.class);
        assertThat(notFound.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
    }
}
```

### 실험 3: 다중 포트 바인딩 확인

```bash
# 앱 시작 후 포트 바인딩 확인
ss -tlnp | grep java
# tcp LISTEN 0 100 *:8080 (HTTP)
# tcp LISTEN 0 100 *:8443 (HTTPS)

# HTTP 요청 → 정상 응답
curl http://localhost:8080/api/hello

# HTTPS 요청 → 정상 응답
curl -k https://localhost:8443/api/hello
```

---

## ⚙️ 설정 최적화 팁

```yaml
# 운영 환경 포트 전략

# 단일 포트 (로드밸런서/Nginx가 SSL 처리):
server:
  port: 8080  # HTTP만, SSL은 앞단에서

# 이중 포트 (앱이 직접 SSL):
server:
  port: 8443
  ssl:
    enabled: true
  http:
    port: 8080  # HTTP → HTTPS 리다이렉트용

# Actuator 분리:
management:
  server:
    port: 8081
    address: 127.0.0.1  # 내부망만

# Context Path (마이크로서비스 게이트웨이 통합 시):
server:
  servlet:
    context-path: /user-service
# → API 게이트웨이: /user-service/** → 이 서비스
# → 서비스 내부: @RequestMapping("/users") → /user-service/users
```

---

## 📌 핵심 정리

```
server.port=0 동작
  ServerSocket(0) → OS가 에페머럴 포트 자동 할당
  TomcatWebServer.getPort() = connector.getLocalPort()
  → WebServerInitializedEvent로 실제 포트 공개
  → Environment에 "local.server.port" 자동 등록

랜덤 포트 획득
  @LocalServerPort (테스트용)
  ApplicationListener<WebServerInitializedEvent>
  Environment.getProperty("local.server.port")

Context Path
  서블릿 컨테이너 레벨 URL 접두사
  @RequestMapping은 Context Path 이후 경로
  HttpServletRequest.getContextPath()로 확인

다중 포트
  factory.addAdditionalTomcatConnectors(httpConnector)
  → HTTP + HTTPS 동시 리스닝

management.server.port 분리
  별도 WebServerApplicationContext 생성
  독립된 스레드 풀, 독립된 DispatcherServlet
  port=-1 → Actuator HTTP 완전 비활성화
```

---

## 🤔 생각해볼 문제

**Q1.** `server.port=0`으로 두 개의 Spring Boot 앱을 동시에 시작하면 같은 포트를 사용할 가능성이 있는가?

**Q2.** Context Path가 `/api`로 설정된 상태에서 Spring Security의 `antMatchers("/users/**")`는 어느 경로를 매칭하는가?

**Q3.** `management.server.port`를 앱 포트와 동일하게 설정하면 어떻게 동작하는가?

> 💡 **해설**
>
> **Q1.** 사실상 없다. OS가 포트 0을 바인딩할 때 이미 사용 중인 포트는 제외하고 할당한다. 두 앱이 거의 동시에 포트를 요청하더라도 OS의 소켓 바인딩은 원자적이므로 충돌이 발생하지 않는다. 단, 에페머럴 포트 범위가 고갈되면(동시에 매우 많은 소켓이 사용 중이면) 바인딩 실패가 발생할 수 있다. 테스트 환경에서 수백 개의 Spring Boot 테스트를 병렬 실행할 때도 안전하게 동작하는 이유다.
>
> **Q2.** Context Path가 `/api`여도 Spring Security의 경로 매처는 Context Path를 제외한 경로를 기준으로 한다. `antMatchers("/users/**")`는 실제 URL `/api/users/1`에서 Context Path `/api`를 제거한 `/users/1`을 매칭한다. 따라서 Security 설정에서 Context Path를 포함할 필요가 없다. 반면 JavaScript 코드나 외부에서 URL을 구성할 때는 Context Path를 포함해야 한다.
>
> **Q3.** `management.server.port`가 `server.port`와 동일하면 `ManagementContextAutoConfiguration`이 별도 컨텍스트를 생성하지 않는다. 하나의 서버에서 앱 API와 Actuator Endpoint를 함께 처리하며, `DispatcherServlet`도 하나만 사용된다. Actuator는 기본 경로(`/actuator`)로 앱 컨텍스트에서 직접 처리된다. 포트 분리의 이점(독립 스레드 풀, 방화벽 차단)이 없어진다. Spring Boot 기본 설정(포트 명시 없음)이 이 경우다.

---

<div align="center">

**[⬅️ 이전: HTTP/2 활성화](./05-http2-activation.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — DevTools LiveReload 메커니즘 ➡️](../devtools/01-livereload-mechanism.md)**

</div>
