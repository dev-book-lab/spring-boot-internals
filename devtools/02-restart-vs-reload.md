# Restart vs Reload 차이 — 두 개의 ClassLoader 분리 전략

---

## 🎯 핵심 질문

- Base ClassLoader와 Restart ClassLoader는 어떤 클래스를 각각 담당하는가?
- Restart가 JVM 전체 재시작보다 빠른 이유는?
- `Restarter`는 재시작을 어떻게 실행하는가?
- JRebel/HotSwap과 DevTools Restart의 기술적 차이는?
- Restart 시 새로 생성되는 것과 유지되는 것은 무엇인가?

---

## 🔬 내부 동작 원리

### 1. 두 ClassLoader 구조

```
JVM 기동 시 DevTools 초기화:

┌─────────────────────────────────────────────────┐
│                      JVM                        │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │        Base ClassLoader                 │    │
│  │  (변경되지 않는 클래스 — 한 번만 로딩)          │    │
│  │                                         │    │
│  │  • spring-*.jar (Spring Framework)      │    │
│  │  • hibernate-*.jar (JPA)                │    │
│  │  • jackson-*.jar (JSON)                 │    │
│  │  • 외부 라이브러리 전체                      │    │
│  │  → classpath:/BOOT-INF/lib/*.jar        │    │
│  └──────────────┬──────────────────────────┘    │
│                 │ 부모                           │
│  ┌──────────────▼──────────────────────────┐    │
│  │        Restart ClassLoader              │    │
│  │  (개발자 코드 — 변경 시 교체)                 │    │
│  │                                         │    │
│  │  • com.example.*.class (앱 코드)          │    │
│  │  • application.yml, templates/          │    │
│  │  → classpath:/BOOT-INF/classes/         │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘

재시작 시:
  Restart ClassLoader만 새 인스턴스로 교체
  Base ClassLoader는 그대로 유지 (라이브러리 재로딩 없음)
  → 라이브러리 로딩 시간 절약 → 빠른 재시작
```

### 2. Restarter — 재시작 핵심 클래스

```java
// Restarter.java (spring-boot-devtools)
public class Restarter {

    private static Restarter instance;

    // 최초 기동 시 초기화
    public static void initialize(String[] args, boolean forceReferenceCleanup,
            RestartInitializer initializer) {
        Restarter restarter = new Restarter(Thread.currentThread(), args,
                forceReferenceCleanup, initializer);
        restarter.start();
        instance = restarter;
    }

    private final URL[] initialUrls;  // Restart ClassLoader가 로딩할 URL 목록
    private final String mainClassName;
    private final ClassLoader applicationClassLoader;

    // 재시작 실행
    public void restart() {
        restart(FailureHandler.NONE);
    }

    void restart(FailureHandler failureHandler) {
        doWithLock(() -> {
            // ① 현재 ApplicationContext 종료
            Restarter.this.stop();

            // ② 새 Restart ClassLoader 생성
            RestartClassLoader restartClassLoader =
                new RestartClassLoader(this.applicationClassLoader,
                    this.urls, this.updatedFiles);

            // ③ 새 ClassLoader로 main 메서드 재실행
            start(failureHandler);
        });
    }

    private synchronized void stop() {
        try {
            // Spring ApplicationContext.close() 호출
            // → @PreDestroy, DisposableBean.destroy() 실행
            // → Singleton Bean 소멸
            this.stopLock.lock();
        } finally {
            this.stopLock.unlock();
        }
    }
}
```

### 3. 재시작 전 과정 타임라인

```
파일 변경 감지 (FileSystemWatcher)
  ↓
ClassPathRestartStrategy.isRestartRequired() → true
  ↓
RestartApplicationListener.onApplicationEvent(ClassPathChangedEvent)
  ↓
Restarter.getInstance().restart()
  ↓
① 현재 ApplicationContext 종료
  - SmartLifecycle.stop() 호출 (Graceful Shutdown)
  - @PreDestroy 메서드 실행
  - DisposableBean.destroy() 실행
  - 약 수백 ms
  ↓
② 새 RestartClassLoader 생성
  - 변경된 .class 파일 포함
  - Base ClassLoader 재사용 (라이브러리 재로딩 없음)
  - 수십 ms
  ↓
③ 새 ClassLoader로 main() 재실행
  - SpringApplication.run() 다시 호출
  - Bean 재생성, ApplicationContext 초기화
  - 약 수백 ms ~ 1~2초 (앱 규모에 따라)
  ↓
④ LiveReloadServer.triggerReload()
  - 브라우저 WebSocket에 reload 신호

전체 소요: 보통 1~3초
JVM 전체 재시작: 5~30초+ (라이브러리 로딩 포함)
```

### 4. RestartClassLoader 구현

```java
// RestartClassLoader.java
class RestartClassLoader extends URLClassLoader {

    private final ClassLoaderFiles updatedFiles;

    public RestartClassLoader(ClassLoader parent, URL[] urls,
            ClassLoaderFiles updatedFiles) {
        super(urls, parent);
        this.updatedFiles = updatedFiles;
    }

    @Override
    protected Class<?> loadClass(String name, boolean resolve)
            throws ClassNotFoundException {
        // 변경된 파일 목록 확인
        String path = name.replace('.', '/').concat(".class");
        ClassLoaderFile file = this.updatedFiles.getFile(path);

        if (file != null) {
            // 변경된 클래스: 새 바이트로 직접 defineClass
            if (file.getKind() == Kind.DELETED) {
                throw new ClassNotFoundException(name);
            }
            return defineFileClass(name, file);
        }

        // 앱 코드 클래스: 이 ClassLoader에서 직접 로딩 (부모에게 위임 안 함)
        // → 부모(Base ClassLoader)가 같은 클래스를 캐시해도 새 버전 로딩
        try {
            return findClass(name);
        } catch (ClassNotFoundException ex) {
            // 앱에 없으면 부모(Base ClassLoader)에 위임
            return super.loadClass(name, resolve);
        }
    }
}

// 중요: 위임 모델 역전
// 일반 ClassLoader: 부모 먼저(parent-first)
// RestartClassLoader: 자신 먼저(child-first) → 변경된 클래스 우선 로딩
```

### 5. 재시작 시 유지되는 것 vs 재생성되는 것

```
유지되는 것 (Base ClassLoader 영역):
  ✔ JVM 인스턴스 (프로세스 유지)
  ✔ 라이브러리 클래스 (Spring, Hibernate 등)
  ✔ LiveReloadServer (@ RestartScope)
  ✔ JVM 힙 워밍업 상태 일부

재생성되는 것 (Restart ClassLoader 교체):
  ✗ 모든 Spring Bean
  ✗ ApplicationContext
  ✗ DataSource 커넥션 풀 (재생성)
  ✗ 앱 코드 클래스 인스턴스
  ✗ 스레드 풀 (서버 스레드 풀 재생성)
  ✗ 캐시 (인메모리 캐시 초기화)

→ DB 커넥션 풀 재생성으로 인해 DB 연결 잠깐 끊김
→ 로그인 세션 유지 여부: 세션 저장소 위치에 따라 다름
   (인메모리 → 재시작 시 세션 소멸)
   (Redis 세션 → 재시작 후에도 세션 유지)
```

### 6. JRebel / HotSwap과 비교

```
방법           원리                    속도    비용
──────────────────────────────────────────────────
DevTools       ClassLoader 교체         중간    무료
               (ApplicationContext 재생성)
               Bean 재생성 필요

JVM HotSwap    바이트코드 Hot Replace    빠름    무료
(DCEVM)        메서드 본문 교체         (DCEVM) 설정 필요
               새 메서드/필드는 제한적

JRebel         에이전트 기반 클래스 재정의 가장 빠름 유료
               Bean 재생성 없음         ~1초    (상업용)
               Spring 컨텍스트 유지

선택 기준:
  무료 + 간단: DevTools (Spring Boot 기본 내장)
  빠른 반영 + 무료: DCEVM + HotSwapAgent (설정 복잡)
  최고 생산성 + 비용: JRebel (기업 환경)
```

### 7. 재시작이 필요한 변경 vs 불필요한 변경

```java
// ClassPathRestartStrategy — 재시작 필요 여부 판단
// DefaultRestartStrategy.java

@Override
public boolean isRestartRequired(ChangedFiles changedFiles) {
    for (ChangedFile changedFile : changedFiles) {
        // .class 파일 변경 → 재시작 필요
        if (changedFile.getRelativeName().endsWith(".class")) {
            return true;
        }
    }
    return false;
}

// 재시작 필요:
//   .class 파일 (Java 소스 변경 후 컴파일)
//   application.yml / application.properties (설정 변경)

// 재시작 불필요 (LiveReload만):
//   src/main/resources/static/** (HTML, CSS, JS)
//   src/main/resources/templates/** (Thymeleaf)
//   → application.yml 에서 제외 패턴 추가 가능
```

---

## 💻 실험으로 확인하기

### 실험 1: 재시작 시간 측정

```java
// 재시작 시간 측정 (로그로)
@Component
public class StartupTimer {

    private static final long startTime = System.currentTimeMillis();

    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        long elapsed = System.currentTimeMillis() - startTime;
        log.info("재시작 완료: {}ms", elapsed);
    }
}
// → DevTools 재시작: 보통 1000~3000ms
// → JVM 첫 기동: 보통 5000~15000ms
```

### 실험 2: ClassLoader 확인

```java
@RestController
public class ClassLoaderController {

    @GetMapping("/classloader")
    public Map<String, String> info() {
        return Map.of(
            "controller", getClass().getClassLoader().getClass().getName(),
            "spring", ApplicationContext.class.getClassLoader().getClass().getName()
        );
    }
}
// 재시작 전: controller = RestartClassLoader (첫 번째 인스턴스)
// 재시작 후: controller = RestartClassLoader (새 인스턴스, 번호 변경)
// spring   = AppClassLoader (항상 동일 — Base ClassLoader)
```

### 실험 3: @RestartScope 동작 확인

```java
@Bean
@RestartScope
public ExpensiveResource expensiveResource() {
    System.out.println("ExpensiveResource 생성: " + System.currentTimeMillis());
    return new ExpensiveResource();
}
// → 재시작 시 이 Bean은 재생성되지 않음 (로그 한 번만 출력)
// → 초기화 비용이 높은 리소스 (파일 핸들, 외부 연결 등)에 적용
// ⚠️ 단, 앱 클래스를 참조하는 Bean에 @RestartScope를 쓰면
//    재시작 후 ClassCastException 발생 가능 (ClassLoader 불일치)
```

---

## 📌 핵심 정리

```
두 ClassLoader 역할
  Base ClassLoader     라이브러리 (변경 안 됨, 재사용)
  Restart ClassLoader  앱 코드 (변경 시 새 인스턴스로 교체)
  → child-first 로딩으로 변경된 클래스 우선 사용

재시작 속도 이점
  라이브러리 재로딩 없음 → Base ClassLoader 유지
  전형적: 1~3초 (vs JVM 재시작 5~30초+)
  앱 코드가 많을수록 차이 감소

Restarter 동작
  ① ApplicationContext.close()
  ② 새 RestartClassLoader 생성
  ③ SpringApplication.run() 재실행
  ④ LiveReloadServer.triggerReload()

JRebel vs DevTools
  DevTools: ClassLoader 교체, Bean 재생성, 무료
  JRebel: 에이전트 기반 클래스 재정의, Bean 유지, 유료
```

---

## 🤔 생각해볼 문제

**Q1.** 재시작 시 DataSource(커넥션 풀)이 재생성될 때 DB 커넥션이 즉시 해제되지 않으면 어떤 문제가 발생할 수 있는가?

**Q2.** Base ClassLoader의 클래스와 Restart ClassLoader의 클래스 사이에 타입을 공유할 때 `ClassCastException`이 발생할 수 있는 경우는?

**Q3.** `spring.devtools.restart.trigger-file` 설정을 사용하는 것이 유리한 경우는?

> 💡 **해설**
>
> **Q1.** HikariCP의 경우 `DataSource.close()`가 호출되면 풀의 모든 커넥션에 `Connection.close()`를 보내고 DB에 연결 해제 신호를 전송한다. 재시작 속도가 빠르면 DB 서버에서 이전 연결이 완전히 정리되기 전에 새 연결이 들어올 수 있다. 대부분의 DB는 이를 정상 처리한다. 그러나 MySQL의 `wait_timeout`이나 Oracle의 세션 제한처럼 연결 수 제한이 있는 DB에서 빈번한 재시작을 반복하면 "Too many connections" 오류가 발생할 수 있다. HikariCP의 `minimumIdle`을 낮게 설정하면 재시작 시 DB 연결 수를 줄일 수 있다.
>
> **Q2.** `@RestartScope` Bean이 Restart ClassLoader의 클래스 타입을 참조할 때 발생한다. 예를 들어 `@RestartScope MyService service`처럼 Base ClassLoader에 유지되는 Bean이 Restart ClassLoader로 로딩된 `MyService` 타입을 참조하면, 재시작 후 `MyService`가 새 ClassLoader로 로딩된 새 클래스가 되어 이전 타입과 다른 것으로 인식된다. Java에서 같은 이름이라도 다른 ClassLoader가 로딩한 클래스는 다른 타입이므로 캐스팅 시 `ClassCastException`이 발생한다. `@RestartScope`는 ClassLoader에 독립적인 타입(String, primitive 등)만 참조하는 Bean에 안전하게 적용할 수 있다.
>
> **Q3.** IDE가 자동 저장을 자주 하거나, 빌드 도구가 점진적 컴파일로 여러 `.class` 파일을 연속으로 생성할 때 유리하다. `trigger-file`을 설정하면 해당 파일이 변경될 때만 재시작한다. 다른 파일이 수십 개 변경되어도 재시작하지 않으므로 불필요한 중간 재시작을 방지한다. IntelliJ의 "Build Project" 완료 후 trigger 파일을 touch하는 스크립트를 연결하면 빌드 완료 시점에만 정확히 재시작할 수 있어 효율적이다.

---

<div align="center">

**[⬅️ 이전: LiveReload 동작 원리](./01-livereload-mechanism.md)** | **[홈으로 🏠](../README.md)** | **[다음: Property Defaults — 캐싱 비활성화 ➡️](./03-property-defaults.md)**

</div>
