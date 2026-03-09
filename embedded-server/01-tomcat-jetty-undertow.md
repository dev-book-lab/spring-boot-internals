# Tomcat vs Jetty vs Undertow 비교 — 아키텍처 차이와 교체 방법

---

## 🎯 핵심 질문

- 세 서버의 스레드 모델은 어떻게 다른가?
- 커넥터 구조 차이가 성능 특성에 어떤 영향을 주는가?
- Spring Boot에서 서버를 교체하는 의존성 변경 방법은?
- 어떤 상황에서 Tomcat 대신 Undertow나 Jetty를 선택해야 하는가?
- 세 서버 모두 `WebServer` 인터페이스로 동일하게 다뤄지는 원리는?

---

## 🔍 왜 이게 존재하는가

```
Spring Boot 이전:
  WAR 파일 → 외부 Tomcat/JBoss/WebLogic에 배포
  → 서버 설치·관리 별도 필요
  → 개발 환경과 운영 환경의 서버 버전 불일치 문제

Spring Boot 내장 서버:
  JAR 파일에 서버 포함 → java -jar 로 실행
  → 서버 설치 불필요
  → 개발/운영 동일 환경 보장
  → 세 가지 선택지: Tomcat(기본), Jetty, Undertow
```

---

## 🔬 내부 동작 원리

### 1. 세 서버 아키텍처 비교

```
Tomcat — BIO/NIO/NIO2 Connector 구조

  AcceptorThread (1개)
    → 새 연결 수락 → SocketChannel 생성
  
  Poller Thread (CPU 수 기반)
    → 준비된 소켓 감지 (Selector 기반)
  
  ThreadPool (기본 200 스레드)
    → 요청당 스레드 할당 (thread-per-request)
    → 처리 완료까지 스레드 점유

  특징:
    성숙한 에코시스템, 가장 많은 레퍼런스
    thread-per-request → 동시 연결 수 = 스레드 풀 크기
    메모리: 스레드당 ~1MB 스택 → 200 스레드 → ~200MB
```

```
Jetty — Connector + Handler 체인 구조

  ServerConnector (NIO 기반)
    → 연결 수락 + I/O 이벤트 처리
  
  QueuedThreadPool
    → 요청 처리 스레드 풀
    → 기본 설정: min=8, max=200
  
  Handler 체인:
    HandlerList → HandlerCollection
    → ContextHandler → SessionHandler → ServletHandler
  
  특징:
    OSGi, WebSocket 지원 우수
    더 유연한 Handler 체인 구성
    임베디드 사용 사례에 강점
    Eclipse Foundation 관리
```

```
Undertow — XNIO 기반 Non-blocking 아키텍처

  XNIO Worker (Non-blocking I/O 프레임워크)
    → IO Thread (CPU 수 * 2, 기본)
       Non-blocking 연결 처리, 경량 작업
    → Worker Thread Pool (CPU 수 * 8, 기본)
       블로킹 서블릿 처리

  특징:
    가장 낮은 메모리 풋프린트
    높은 동시성 처리 우수
    JBoss/WildFly의 기본 서버
    리소스 효율성 최고 (메모리, CPU)
    web.xml 불필요 (프로그래밍 설정)
```

### 2. 스레드 모델 상세 비교

```java
// Tomcat — thread-per-request (서블릿 표준)
// 요청 1개 → 스레드 1개 점유 → 응답 완료까지 블로킹

// 기본 스레드 풀 설정
server.tomcat.threads.max=200         // 최대 스레드
server.tomcat.threads.min-spare=10    // 최소 유휴 스레드
server.tomcat.accept-count=100        // 큐 대기 요청 수

// 연결 처리 흐름:
// 요청 → AcceptorThread → Poller → ThreadPool 스레드 할당
//   → 서블릿 실행 (블로킹 허용)
//   → 응답 → 스레드 반환
```

```java
// Undertow — IO Thread + Worker Thread 분리
// IO Thread: 연결 수락, 헤더 파싱, 경량 작업 (블로킹 금지)
// Worker Thread: 서블릿 실행 (블로킹 허용)

// 기본 설정
server.undertow.threads.io=4          // IO 스레드 (CPU * 2)
server.undertow.threads.worker=32     // Worker 스레드 (CPU * 8)
server.undertow.buffer-size=1024      // 버퍼 크기 (byte)
server.undertow.direct-buffers=true   // Direct ByteBuffer 사용
```

### 3. 서버 교체 방법

```xml
<!-- Maven: Tomcat 제외 + Undertow 추가 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

```kotlin
// Gradle (Kotlin DSL)
configurations {
    all {
        exclude(group = "org.springframework.boot",
                module = "spring-boot-starter-tomcat")
    }
}
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-undertow")
    // 또는 spring-boot-starter-jetty
}
```

### 4. Auto-configuration — 서버 자동 선택

```java
// ServletWebServerFactoryAutoConfiguration
@AutoConfiguration
@ConditionalOnWebApplication
@EnableConfigurationProperties(ServerProperties.class)
@Import({
    ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
    EmbeddedTomcat.class,   // @ConditionalOnClass(Tomcat.class)
    EmbeddedJetty.class,    // @ConditionalOnClass(Server.class)
    EmbeddedUndertow.class  // @ConditionalOnClass(Undertow.class)
})
public class ServletWebServerFactoryAutoConfiguration { }

// 세 클래스 모두 @ConditionalOnClass + @ConditionalOnMissingBean
// → classpath에 있는 서버가 자동 선택
// → 여러 서버 JAR가 있으면 Tomcat > Jetty > Undertow 순서 (Auto-configuration order)

@Configuration
@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
@ConditionalOnMissingBean(value = ServletWebServerFactory.class,
                            search = SearchStrategy.CURRENT)
static class EmbeddedTomcat {
    @Bean
    TomcatServletWebServerFactory tomcatServletWebServerFactory(...) { ... }
}
```

### 5. 성능 특성 요약

```
워크로드별 권장 서버:

CPU 집중 / 짧은 요청:
  → Tomcat (thread-per-request, 단순한 모델, 충분히 빠름)
  → 레퍼런스 가장 많음, 운영 경험 풍부

높은 동시 연결 / 긴 요청 (WebSocket, SSE):
  → Undertow (낮은 메모리, 높은 동시성)
  → IO Thread가 많은 연결을 적은 리소스로 처리

유연한 커스터마이징 / OSGi 환경:
  → Jetty (Handler 체인 유연성)
  → Eclipse 에코시스템 선호 시

메모리 최소화 (컨테이너, Serverless):
  → Undertow (가장 낮은 풋프린트)
```

---

## 💻 실험으로 확인하기

### 실험 1: 현재 서버 확인

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = SpringApplication.run(App.class, args);
        ServletWebServerApplicationContext webCtx =
            (ServletWebServerApplicationContext) ctx;
        WebServer server = webCtx.getWebServer();
        System.out.println("서버 타입: " + server.getClass().getSimpleName());
        System.out.println("포트: " + server.getPort());
    }
}
// Tomcat: TomcatWebServer
// Jetty:  JettyWebServer
// Undertow: UndertowWebServer
```

### 실험 2: 스레드 풀 크기 비교

```bash
# 시작 로그로 스레드 풀 크기 확인
java -jar app.jar --logging.level.org.apache.tomcat=DEBUG
# Tomcat: "Starting ProtocolHandler ["http-nio-8080"]"
# 스레드: http-nio-8080-exec-1 ~ http-nio-8080-exec-200

java -jar app-undertow.jar
# Undertow: "XNIO-1 I/O-1" ~ "XNIO-1 I/O-4" (IO 스레드)
#           "XNIO-1 task-1" ~ "XNIO-1 task-32" (Worker 스레드)
```

---

## 📌 핵심 정리

```
세 서버 핵심 차이
  Tomcat    thread-per-request, 성숙한 생태계, Spring Boot 기본
  Jetty     유연한 Handler 체인, OSGi/WebSocket 강점
  Undertow  IO/Worker Thread 분리, 최저 메모리, 높은 동시성

교체 방법
  spring-boot-starter-tomcat 제외
  spring-boot-starter-undertow or jetty 추가
  → Auto-configuration이 classpath 감지로 자동 전환

서버 선택 기준
  기본 선택: Tomcat (레퍼런스, 안정성)
  고동시성·저메모리: Undertow
  특수 요구사항: Jetty
```

---

## 🤔 생각해볼 문제

**Q1.** `spring-boot-starter-web`을 사용하는 프로젝트에서 `spring-boot-starter-undertow`를 추가하되 `spring-boot-starter-tomcat`을 제외하지 않으면 어떤 서버가 실행되는가?

**Q2.** Undertow의 IO Thread에서 블로킹 작업을 직접 실행하면 어떤 문제가 발생하는가?

**Q3.** WebFlux 프로젝트에서 내장 서버로 Tomcat을 사용하는 것이 가능한가? 가능하다면 동작 방식은 어떻게 달라지는가?

> 💡 **해설**
>
> **Q1.** Tomcat이 실행된다. `EmbeddedTomcat`, `EmbeddedUndertow` 둘 다 조건을 만족하지만, `TomcatServletWebServerFactory`가 `@AutoConfigureOrder`와 `@ConditionalOnMissingBean` 상호작용으로 먼저 등록된다. 두 서버를 동시에 classpath에 두는 것은 의도하지 않은 동작을 유발할 수 있으므로, 서버 교체 시 반드시 기존 서버를 `exclude`해야 한다.
>
> **Q2.** Undertow의 IO Thread는 Non-blocking I/O 이벤트 루프를 실행한다. 여기서 블로킹 작업(`Thread.sleep`, DB 쿼리, 파일 I/O 등)을 실행하면 해당 IO Thread가 블로킹되어 다른 연결의 이벤트를 처리하지 못한다. 전체 서버의 I/O 처리가 멈추는 심각한 성능 저하가 발생한다. 블로킹 작업은 반드시 Worker Thread Pool에서 실행해야 하며, 서블릿 요청은 자동으로 Worker Thread에서 실행된다.
>
> **Q3.** 가능하다. WebFlux는 기본적으로 Netty를 사용하지만 Tomcat, Jetty, Undertow도 지원한다. WebFlux + Tomcat 조합에서는 Tomcat의 Non-blocking Connector(NIO2)가 사용되고, Reactive 요청 처리가 Tomcat 스레드 풀 위에서 실행된다. 순수 Non-blocking 성능은 Netty보다 낮지만, 기존 Tomcat 운영 경험을 활용하거나 서블릿과 WebFlux를 혼용할 때 선택할 수 있다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: ServletWebServerFactory 동작 원리 ➡️](./02-servlet-web-server-factory.md)**

</div>
