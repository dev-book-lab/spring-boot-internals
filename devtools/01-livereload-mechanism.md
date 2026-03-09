# LiveReload 동작 원리 — WebSocket 통신과 파일 변경 감지 전 과정

---

## 🎯 핵심 질문

- 내장 LiveReload 서버는 어떻게 브라우저와 WebSocket으로 통신하는가?
- 파일 변경 감지 → 브라우저 리프레시까지의 전체 흐름은?
- `ClassPathFileSystemWatcher`는 어떤 방식으로 파일 변경을 감지하는가?
- LiveReload와 Restart의 실행 순서는 어떻게 되는가?
- LiveReload 서버 포트(35729)는 왜 고정인가?

---

## 🔍 왜 이게 존재하는가

```
전통적인 개발 사이클:
  코드 수정 → 서버 재시작(수 초~수십 초) → 브라우저 새로고침(수동)

LiveReload가 해결하는 문제:
  브라우저 새로고침을 자동화
  정적 리소스(HTML, CSS, JS) 변경 시 서버 재시작 없이 즉시 반영
  → "저장하면 바로 브라우저에서 확인" 개발 경험 제공
```

---

## 😱 흔한 오해 또는 설정 실수

```java
// ❌ 오해: LiveReload가 있으면 서버 재시작 없이 Java 코드 변경도 즉시 반영된다
// → 실제로는 .class 파일 변경 시 Restart가 먼저 실행되고,
//   Restart 완료 후 LiveReload 신호가 브라우저로 전송됨
//   LiveReload 자체는 브라우저 새로고침만 담당

// ❌ 오해: LiveReload 서버 포트(35729)를 임의로 바꿔도 된다
// → 브라우저 확장 프로그램이 35729를 하드코딩으로 감지
//   포트 변경 시 확장 프로그램이 연결 못 함

// ❌ 실수: DevTools를 추가했는데 브라우저 자동 새로고침이 안 된다
// → LiveReload 브라우저 확장 프로그램 설치 누락
//   또는 spring.devtools.livereload.enabled=false 설정
```

---

## ✨ 올바른 이해와 설정

```
LiveReload 동작 조건:
  ① spring-boot-devtools 의존성 추가 (developmentOnly)
  ② 브라우저에 LiveReload 확장 프로그램 설치
     (Chrome: "LiveReload" 확장, 또는 페이지에 livereload.js 포함)
  ③ 확장 프로그램 활성화 상태에서 앱 접속

변경 종류별 동작:
  .class 변경    → Restart → LiveReload (브라우저 새로고침)
  .html/.css/.js → Restart 없이 LiveReload만 (빠름)
  CSS만 변경     → liveCSS: 페이지 새로고침 없이 CSS만 교체
```

---

## 🔬 내부 동작 원리

### 1. DevTools 전체 구성

```
spring-boot-devtools 의존성 추가 시 자동 활성화:

┌─────────────────────────────────────────┐
│         Spring Boot App (8080)          │
│                                         │
│  ClassPathFileSystemWatcher             │
│    → classpath 파일 변경 감지              │
│                                         │
│  ┌─────────────────┐                    │
│  │ LiveReload 서버  │ (포트 35729)        │
│  │ WebSocket 대기   │                    │
│  └────────┬────────┘                    │
└───────────┼─────────────────────────────┘
            │ WebSocket (ws://localhost:35729)
            ▼
    ┌───────────────────┐
    │ 브라우저            │
    │ (LiveReload 확장)  │
    │ 또는 내장 스크립트    │
    └───────────────────┘
```

### 2. LiveReloadServer — 내장 WebSocket 서버

```java
// LiveReloadServer.java (spring-boot-devtools)
public class LiveReloadServer {

    // LiveReload 표준 포트 (고정)
    // → LiveReload 브라우저 확장이 이 포트만 감지
    // → 변경 불가능한 것은 아니지만 표준이므로 그대로 사용
    public static final int DEFAULT_PORT = 35729;

    private final int port;
    private ServerSocket serverSocket;
    private Thread listenThread;

    // WebSocket 연결된 브라우저 목록
    private final List<Connection> connections = new ArrayList<>();

    public void start() throws IOException {
        // TCP 서버 소켓 열기 (포트 35729)
        this.serverSocket = new ServerSocket(this.port, 1,
            InetAddress.getByName("127.0.0.1"));
        // 로컬호스트만 (외부 접근 불가)

        // 연결 수락 루프 (별도 스레드)
        this.listenThread = new Thread(this::acceptConnections, "LiveReload Server");
        this.listenThread.setDaemon(true);
        this.listenThread.start();
    }

    private void acceptConnections() {
        // 브라우저에서 WebSocket 업그레이드 요청 수락
        while (!this.serverSocket.isClosed()) {
            try {
                Socket socket = this.serverSocket.accept();
                Connection connection = new Connection(socket, ...);
                this.connections.add(connection);
                connection.run();  // HTTP Upgrade → WebSocket 처리
            } catch (IOException ex) { ... }
        }
    }

    // 브라우저에 리프레시 신호 전송
    public synchronized void triggerReload() {
        // 연결된 모든 브라우저에 WebSocket 메시지 전송
        List<Connection> liveConnections = new ArrayList<>(this.connections);
        liveConnections.forEach(connection -> {
            try {
                connection.triggerReload();
            } catch (Exception ex) {
                this.connections.remove(connection);
            }
        });
    }
}
```

### 3. WebSocket 핸드셰이크 — HTTP → WebSocket Upgrade

```java
// Connection.java — HTTP Upgrade 처리
class Connection implements Runnable {

    @Override
    public void run() {
        try {
            // HTTP 요청 읽기
            Header header = new Header(this.inputStream);

            // WebSocket Upgrade 핸드셰이크
            if (header.isWebSocket()) {
                runWebSocket(header);
            } else {
                // 일반 HTTP: 브라우저가 livereload.js 요청할 수 있음
                runHttp(header);
            }
        } catch (Exception ex) { ... }
    }

    private void runWebSocket(Header header) throws Exception {
        // WebSocket 핸드셰이크 응답
        // Sec-WebSocket-Accept 키 계산 후 응답
        String accept = generateAcceptKey(header.get("Sec-WebSocket-Key"));
        writeWebSocketUpgradeResponse(accept);

        // WebSocket 연결 유지 — 리프레시 신호 대기
        awaitReloadTrigger();
    }

    void triggerReload() throws Exception {
        // LiveReload 프로토콜 메시지 전송
        // {"command":"reload","path":"*","liveCSS":true}
        byte[] payload = "{\"command\":\"reload\",\"path\":\"*\",\"liveCSS\":true}"
            .getBytes(StandardCharsets.UTF_8);
        writeWebSocketFrame(payload);
    }
}
```

### 4. ClassPathFileSystemWatcher — 파일 변경 감지

```java
// ClassPathFileSystemWatcher — classpath 파일 변경 감시
public class ClassPathFileSystemWatcher implements InitializingBean, DisposableBean {

    private final FileSystemWatcher fileSystemWatcher;
    private final ClassPathRestartStrategy restartStrategy;

    @Override
    public void afterPropertiesSet() {
        // 감시 대상: classpath 루트 디렉토리들
        // → 보통 build/classes/java/main, build/resources/main 등
        Set<File> watchDirectories = getWatchDirectories();
        this.fileSystemWatcher.addSourceDirectories(watchDirectories);
        this.fileSystemWatcher.start();
    }
}

// FileSystemWatcher — 폴링 기반 변경 감지
public class FileSystemWatcher {

    // 폴링 간격: 기본 1초 (조정 가능)
    private final long pollInterval;

    // 조용한 기간: 마지막 변경 후 대기 시간 (기본 400ms)
    // 파일 저장 시 여러 파일이 연속 변경되므로
    // 마지막 변경 후 일정 시간 동안 추가 변경 없으면 이벤트 발생
    private final long quietPeriod;

    // 폴링 스레드
    private Thread watchThread;

    private void watch() {
        while (!Thread.currentThread().isInterrupted()) {
            // 각 감시 디렉토리의 파일 스냅샷 비교
            for (SourceDirectory directory : this.directories) {
                Map<File, FileSnapshot> current = takeSnapshot(directory);
                Map<File, FileSnapshot> previous = directory.getSnapshot();

                // 추가, 수정, 삭제된 파일 감지
                Set<ChangedFile> changes = getChanges(previous, current);
                if (!changes.isEmpty()) {
                    // quietPeriod 대기 후 이벤트 발생
                    fireChangedEvent(new ChangedFiles(directory, changes));
                }
                directory.setSnapshot(current);
            }
            Thread.sleep(this.pollInterval);  // 1초 대기
        }
    }
}
```

### 5. 파일 변경 → 브라우저 리프레시 전체 흐름

```
1. IDE에서 파일 저장 (예: HomeController.java 수정)

2. 빌드 도구가 컴파일
   → HomeController.class 갱신
   → build/classes/java/main/com/example/HomeController.class

3. FileSystemWatcher 폴링 (1초 간격)
   → 파일 변경 감지
   → quietPeriod(400ms) 대기
   → 추가 변경 없음 → ChangedFilesEvent 발생

4. ClassPathRestartStrategy.isRestartRequired(changedFiles) 판단
   → .class 파일 변경 → Restart 필요
   → .html, .css, .js, 리소스만 변경 → Restart 불필요

5a. Restart 필요 시:
    → RestartApplicationListener → ApplicationContext 재시작
    → Restart 완료 후 → LiveReloadServer.triggerReload()
    → 브라우저에 WebSocket "reload" 메시지 전송

5b. Restart 불필요 (정적 리소스만) 시:
    → LiveReloadServer.triggerReload() 즉시 호출
    → 브라우저에 WebSocket "reload" 메시지 전송
    (CSS만 변경 시 → liveCSS:true → 페이지 새로고침 없이 CSS만 교체)

6. 브라우저(LiveReload 확장 또는 livereload.js)
   → "reload" 메시지 수신
   → 페이지 새로고침 (또는 CSS 교체)
```

### 6. DevToolsAutoConfiguration — 자동 설정

```java
// DevToolsAutoConfiguration.java
@AutoConfiguration
@ConditionalOnInitializedRestarter  // Restarter 초기화 후
@EnableConfigurationProperties(DevToolsProperties.class)
public class LocalDevToolsAutoConfiguration {

    // LiveReload 서버 Bean 등록
    @Configuration
    @ConditionalOnProperty(prefix = "spring.devtools.livereload",
                            name = "enabled", matchIfMissing = true)
    static class LiveReloadConfiguration {

        @Bean
        @RestartScope  // 재시작 후에도 같은 인스턴스 유지
        // → 브라우저 WebSocket 연결 유지 (재연결 불필요)
        public LiveReloadServer liveReloadServer(DevToolsProperties properties) {
            return new LiveReloadServer(properties.getLivereload().getPort(),
                ThreadFactory.forName("LiveReload"));
        }

        @Bean
        public OptionalLiveReloadServer optionalLiveReloadServer(
                LiveReloadServer liveReloadServer) {
            return new OptionalLiveReloadServer(liveReloadServer);
        }
    }

    // 파일 변경 감지 → 재시작/리로드 트리거
    @Bean
    public ClassPathFileSystemWatcher classPathFileSystemWatcher(
            DevToolsProperties properties,
            Optional<ClassPathRestartStrategy> classPathRestartStrategy) {

        URL[] urls = Restarter.getInstance().getInitialUrls();
        ClassPathFileSystemWatcher watcher = new ClassPathFileSystemWatcher(
            getFileSystemWatcherFactory(properties),
            classPathRestartStrategy.orElse(null),
            urls);
        watcher.setStopWatcherOnRestart(true);
        return watcher;
    }
}
```

### 7. Thymeleaf 템플릿에 livereload.js 자동 삽입

```java
// LiveReload 브라우저 확장 없이도 동작하려면
// 페이지에 livereload.js 스크립트 추가 필요

// Spring Boot DevTools: 자동으로 삽입하지는 않음
// → 브라우저 확장 설치 권장 (Chrome: LiveReload 확장)

// 또는 수동으로 Thymeleaf 레이아웃에 추가:
// <script src="http://localhost:35729/livereload.js"></script>

// DevTools가 35729 포트에서 livereload.js 스크립트 파일도 제공
// → Connection.runHttp()에서 livereload.js 응답
```

---

## 💻 실험으로 확인하기

### 실험 1: LiveReload 활성화 확인

```bash
# 앱 시작 로그에서 LiveReload 서버 확인
# "LiveReload server is running on port 35729"

# WebSocket 연결 테스트
curl -v --include \
  --no-buffer \
  --header "Connection: Upgrade" \
  --header "Upgrade: websocket" \
  --header "Sec-WebSocket-Key: SGVsbG8sIHdvcmxkIQ==" \
  --header "Sec-WebSocket-Version: 13" \
  http://localhost:35729
# → 101 Switching Protocols 응답
```

### 실험 2: 파일 변경 감지 타이밍 조정

```yaml
spring:
  devtools:
    restart:
      poll-interval: 2s    # 폴링 간격 (기본 1s)
      quiet-period: 1s     # 조용한 기간 (기본 400ms)
    livereload:
      port: 35729          # 포트 변경 (비표준, 확장과 불일치 가능)
```

### 실험 3: 특정 파일 변경 시 재시작 제외

```yaml
spring:
  devtools:
    restart:
      exclude: static/**,public/**,templates/**
      # → 정적 리소스 변경 시 Restart 없이 LiveReload만
      additional-paths: src/main/resources
      # → 추가 감시 경로
```

---

## ⚙️ 설정 최적화 팁

```yaml
spring:
  devtools:
    livereload:
      enabled: true   # 기본 true, false로 비활성화
    restart:
      enabled: true   # 기본 true
      # 재시작 제외 패턴 (변경 빈번한 파일)
      exclude: "static/**,public/**,templates/**,**/*.html"
      # 추가 감시 경로 (기본 classpath 외)
      additional-paths: "src/main/resources"
      # 특정 파일에서 재시작 트리거 (기본값 무시하고 명시적 트리거)
      trigger-file: ".reloadtrigger"
      # → IDE가 이 파일을 touch하면 재시작 (다른 파일 무시)
```

---

## 🤔 트레이드오프

```
LiveReload (브라우저 자동 새로고침)
  장점: 정적 리소스 변경 시 즉각 확인, 개발 속도 향상
  단점: 브라우저 확장 의존, 포트(35729) 고정, 외부 접근 불가
        폼 입력 중 새로고침 → 작성 내용 날아감

폴링 방식 파일 감지 (vs inotify/FSEvents)
  장점: OS/파일시스템 독립적, 단순한 구현
  단점: 최대 1.4초 지연(poll 1s + quiet 400ms)
        파일 수가 많으면 CPU 사용량 증가

@RestartScope
  장점: 재시작 후 WebSocket 연결 유지 → 브라우저 재연결 불필요
  단점: 앱 클래스 타입을 참조하면 ClassCastException 위험
        잘못 사용하면 재시작 후 상태 불일치
```

---

## 📌 핵심 정리

```
LiveReload 서버
  포트 35729에서 TCP 서버 소켓 대기
  브라우저 확장(또는 livereload.js)이 WebSocket 연결
  파일 변경 감지 후 → triggerReload() → WebSocket 메시지 전송
  → 브라우저 자동 리프레시

파일 변경 감지 메커니즘
  FileSystemWatcher — 폴링 기반 (1초 간격)
  quietPeriod (400ms) — 연속 변경 묶어서 처리
  .class 변경 → Restart 필요
  정적 리소스만 변경 → LiveReload만 (Restart 없음)

@RestartScope
  LiveReloadServer Bean에 적용
  → Restart 후에도 같은 Bean 인스턴스 유지
  → 브라우저 WebSocket 연결이 끊기지 않음

운영 환경 자동 비활성화
  DevTools는 완전한 JAR 패키징 시 자동 비활성화
  (spring-boot-devtools는 optional 의존성이므로)
```

---

## 🤔 생각해볼 문제

**Q1.** LiveReload 서버가 `127.0.0.1`(로컬호스트)에만 바인딩하는 이유는 무엇인가?

**Q2.** `@RestartScope`가 LiveReloadServer Bean에 적용된 이유는 무엇이며, 이것이 없으면 어떤 문제가 발생하는가?

**Q3.** IDE에서 파일을 저장할 때 FileSystemWatcher가 즉각 반응하지 않고 최대 1.4초(poll 1s + quiet 400ms) 후에 반응하는 이유는 무엇인가?

> 💡 **해설**
>
> **Q1.** 보안 때문이다. LiveReload 서버는 인증 없이 연결된 클라이언트에 페이지 리프레시 신호를 보낼 수 있다. 외부 네트워크에 노출되면 다른 시스템에서 개발 중인 브라우저 탭을 조작하거나 서비스 거부 공격이 가능하다. 개발 도구이므로 로컬 머신 내부에서만 동작하도록 제한한다.
>
> **Q2.** Restart가 발생하면 `ApplicationContext`가 재생성되고 Singleton Bean들도 새로 생성된다. `LiveReloadServer`가 일반 Bean이면 Restart마다 새 인스턴스가 생성되고 이전 인스턴스의 소켓은 닫힌다. 브라우저 입장에서는 WebSocket 연결이 끊어지고 재연결이 필요하다. `@RestartScope`는 Restart ClassLoader가 교체되어도 이 Bean만 기존 인스턴스를 유지하므로 브라우저 연결이 유지된다. 결과적으로 Restart 후에도 브라우저 WebSocket이 살아있어 리프레시 신호를 즉시 수신할 수 있다.
>
> **Q3.** 두 가지 이유가 있다. 첫째, 폴링 방식은 파일 시스템 이벤트(inotify 등)가 아닌 주기적 스냅샷 비교이므로 최대 pollInterval(1초) 지연이 발생한다. 둘째, `quietPeriod`는 IDE가 여러 파일을 동시에 저장할 때(예: Gradle 빌드로 여러 .class 파일 생성) 모든 파일이 저장된 후 한 번에 재시작하기 위한 대기 시간이다. 마지막 변경 이후 400ms 동안 추가 변경이 없으면 재시작을 트리거한다. 이를 통해 불완전한 빌드 상태에서 재시작하는 문제를 방지한다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Restart vs Reload 차이 ➡️](./02-restart-vs-reload.md)**

</div>
