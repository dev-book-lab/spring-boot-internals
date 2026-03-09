# Fat JAR 구조와 JarLauncher — 중첩 JAR 로딩 커스텀 ClassLoader

---

## 🎯 핵심 질문

- `BOOT-INF/classes`와 `BOOT-INF/lib` 구조가 필요한 이유는?
- `JarLauncher`가 중첩 JAR를 로딩하기 위해 커스텀 ClassLoader를 만드는 원리는?
- `java -jar app.jar`가 실행되면 내부적으로 어떤 일이 일어나는가?
- `MANIFEST.MF`의 `Main-Class`와 `Start-Class`는 각각 무슨 역할인가?
- `LaunchedURLClassLoader`는 `jar:nested:` 형태의 중첩 URL을 어떻게 처리하는가?

---

## 🔍 왜 이게 존재하는가

```
표준 Java JAR 스펙의 한계:
  java -jar app.jar 실행 시
  JVM은 MANIFEST.MF의 Main-Class만 로딩
  JAR 내부의 중첩 JAR는 java.util.jar.JarFile이 직접 지원 안 함

해결 방법 비교:
  ① Shade/Shadow JAR: 모든 클래스를 하나로 압축
     문제: 동일 경로 리소스 충돌, 서명 깨짐
  ② 의존성 외부 lib/ 폴더: 배포 시 폴더 구조 유지 필요
     문제: 단일 파일 배포 불가
  ③ Spring Boot Fat JAR: 중첩 JAR + 커스텀 ClassLoader
     → 단일 JAR로 배포, 클래스 충돌 없음, 서명 보존
```

---

## 🔬 내부 동작 원리

### 1. Fat JAR 디렉토리 구조

```
app.jar (Fat JAR)
├── META-INF/
│   └── MANIFEST.MF
│       Main-Class: org.springframework.boot.loader.launch.JarLauncher
│       Start-Class: com.example.Application
│       Spring-Boot-Version: 3.4.0
│       Spring-Boot-Classes: BOOT-INF/classes/
│       Spring-Boot-Lib: BOOT-INF/lib/
│       Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
│
├── BOOT-INF/
│   ├── classes/                          ← 앱 코드
│   │   ├── com/example/Application.class
│   │   └── application.yml
│   │
│   ├── lib/                              ← 의존성 JAR (중첩)
│   │   ├── spring-core-6.2.0.jar
│   │   ├── spring-web-6.2.0.jar
│   │   └── ...
│   │
│   └── classpath.idx                     ← 클래스패스 순서 인덱스
│
└── org/springframework/boot/loader/
    ├── launch/
    │   └── JarLauncher.class             ← java -jar 진입점
    └── jar/
        └── NestedJarFile.class           ← 중첩 JAR 처리
```

### 2. MANIFEST.MF — 두 가지 Main 항목

```
Manifest-Version: 1.0
Main-Class: org.springframework.boot.loader.launch.JarLauncher
Start-Class: com.example.Application

Main-Class  → JVM이 java -jar 실행 시 호출하는 진입점
             JarLauncher가 ClassLoader 준비 후 Start-Class 실행
Start-Class → 실제 앱의 main() 메서드를 가진 클래스
             SpringApplication.run()이 여기서 시작
```

### 3. JarLauncher — 실행 흐름

```java
public class JarLauncher extends ExecutableArchiveLauncher {

    static final EntryFilter NESTED_ARCHIVE_ENTRY_FILTER = (entry) -> {
        if (entry.isDirectory()) {
            return entry.name().equals("BOOT-INF/classes/");
        }
        return entry.name().startsWith("BOOT-INF/lib/");
    };

    public static void main(String[] args) throws Exception {
        new JarLauncher().launch(args);
    }
}

// ExecutableArchiveLauncher.launch() 처리 흐름:
void launch(String[] args) throws Exception {
    // ① 현재 JAR를 Archive로 열기
    Archive archive = createArchive();

    // ② 중첩 JAR URL 목록 수집
    //    BOOT-INF/lib/*.jar + BOOT-INF/classes/
    List<URL> urls = getClassPathUrls();

    // ③ LaunchedURLClassLoader 생성
    ClassLoader classLoader = createClassLoader(urls);

    // ④ Start-Class 로딩 후 main() 실행
    String mainClass = getMainClass();  // = "com.example.Application"
    launch(args, mainClass, classLoader);
}
```

### 4. LaunchedURLClassLoader — 중첩 JAR URL 처리

```java
// 핵심 문제:
// URL: jar:nested:/path/app.jar/!BOOT-INF/lib/spring-core.jar!/
// → 표준 JVM URL 핸들러는 이 이중 중첩 처리 불가

// Spring Boot 해결책 (Boot 3.2+):
// NestedJarFile — java.util.jar.JarFile 기반 중첩 JAR 지원
// jar:nested: 커스텀 프로토콜 URL 핸들러 등록

public class NestedJarFile extends JarFile {
    // app.jar 스트림 → BOOT-INF/lib/spring-core.jar 엔트리 추출
    // → 해당 스트림을 통해 spring-core.jar 내부 클래스 접근
    // → 임시 파일 없이 메모리에서 처리 (Boot 3.x 개선)
}

// LaunchedURLClassLoader — parent-first 표준 위임 모델
public class LaunchedURLClassLoader extends URLClassLoader {

    // classpath.idx 순서대로 URL 배열 유지:
    // 1. BOOT-INF/classes/              (앱 클래스)
    // 2. BOOT-INF/lib/spring-core.jar  (의존성, 순서 보장)
    // ...

    @Override
    protected Class<?> loadClass(String name, boolean resolve)
            throws ClassNotFoundException {
        // parent-first: 부모(AppClassLoader/PlatformClassLoader) 먼저
        // → 표준 Java 클래스 우선 보장
        // → 이후 BOOT-INF/* 탐색
        return super.loadClass(name, resolve);
    }
}
```

### 5. classpath.idx — 결정론적 로딩 순서

```yaml
# BOOT-INF/classpath.idx
# (ZIP 파일 엔트리 순서는 비결정적 → 인덱스로 순서 보장)
- "BOOT-INF/lib/spring-core-6.2.0.jar"
- "BOOT-INF/lib/spring-context-6.2.0.jar"
- "BOOT-INF/lib/spring-beans-6.2.0.jar"
- "BOOT-INF/lib/spring-web-6.2.0.jar"
- "BOOT-INF/lib/tomcat-embed-core-10.1.x.jar"
- "BOOT-INF/lib/jackson-databind-2.17.x.jar"
# ...
```

### 6. 전체 실행 흐름

```
java -jar app.jar
  ↓
JVM: MANIFEST.MF → Main-Class = JarLauncher
  ↓
JarLauncher.main()
  ↓
① app.jar → Archive
② BOOT-INF/classes/ + BOOT-INF/lib/*.jar → URL 목록 (classpath.idx 순서)
③ LaunchedURLClassLoader 생성 (jar:nested: 핸들러 포함)
④ Start-Class = com.example.Application 로딩
⑤ Application.main(args) 실행
  ↓
SpringApplication.run() → 일반 Spring Boot 기동
```

---

## 💻 실험으로 확인하기

### 실험 1: Fat JAR 내부 구조 탐색

```bash
./gradlew bootJar

# 내부 구조
jar -tf build/libs/app.jar | grep -E "MANIFEST|BOOT-INF|JarLauncher" | head -10

# MANIFEST.MF 확인
unzip -p build/libs/app.jar META-INF/MANIFEST.MF

# classpath.idx 확인
unzip -p build/libs/app.jar BOOT-INF/classpath.idx | head -5
```

### 실험 2: ClassLoader 타입 런타임 확인

```java
@GetMapping("/loader-info")
public Map<String, String> loaderInfo() {
    ClassLoader cl = getClass().getClassLoader();
    return Map.of(
        "type", cl.getClass().getSimpleName(),
        // Fat JAR 실행:  LaunchedURLClassLoader
        // IDE 직접 실행: AppClassLoader
        "parent", cl.getParent().getClass().getSimpleName()
    );
}
```

### 실험 3: 중첩 JAR URL 직접 확인

```java
ClassLoader cl = Thread.currentThread().getContextClassLoader();
if (cl instanceof URLClassLoader ucl) {
    Arrays.stream(ucl.getURLs())
        .limit(5)
        .forEach(System.out::println);
}
// jar:nested:/path/to/app.jar/!BOOT-INF/lib/spring-core-6.2.0.jar!/
// jar:nested:/path/to/app.jar/!BOOT-INF/classes!/
```

---

## 📌 핵심 정리

```
Fat JAR 핵심 구조
  BOOT-INF/classes/     앱 클래스 (컴파일 결과)
  BOOT-INF/lib/         의존성 JAR (중첩)
  BOOT-INF/classpath.idx  결정론적 로딩 순서
  org/.../loader/       JarLauncher + ClassLoader 구현체

MANIFEST.MF 두 항목
  Main-Class: JarLauncher         JVM 진입점 (ClassLoader 준비)
  Start-Class: com.example.App   실제 앱 main 클래스

LaunchedURLClassLoader
  jar:nested: 커스텀 프로토콜로 중첩 JAR 접근
  classpath.idx 순서 준수, parent-first 위임 모델
  NestedJarFile로 임시 파일 없이 스트림 처리

Shade JAR 대비 장점
  클래스/리소스 경로 충돌 없음
  JAR 서명(Signature) 보존
  중첩 JAR 형태 유지 → 원본 의존성 구조 보존
```

---

## 🤔 생각해볼 문제

**Q1.** `Start-Class`를 `Main-Class`로 직접 지정하면 안 되는 이유는?

**Q2.** `LaunchedURLClassLoader`가 parent-first를 사용하는 반면 DevTools의 `RestartClassLoader`는 child-first를 사용한다. 이 차이가 필요한 이유는?

**Q3.** `classpath.idx` 파일이 없으면 어떤 문제가 발생할 수 있는가?

> 💡 **해설**
>
> **Q1.** `Start-Class`를 `Main-Class`로 직접 지정하면 JVM은 기본 `AppClassLoader`로 실행한다. 이 시점에는 `BOOT-INF/lib/*.jar` 중첩 JAR를 위한 커스텀 URL 핸들러(`jar:nested:`)가 등록되지 않았으므로 의존성 클래스를 찾지 못해 `ClassNotFoundException`이 발생한다. `JarLauncher`가 먼저 실행되어 `LaunchedURLClassLoader`를 준비해야만 `Start-Class`의 `main()`이 올바른 ClassLoader 컨텍스트에서 실행될 수 있다.
>
> **Q2.** `LaunchedURLClassLoader`는 운영 환경 안정성이 목표다. 상위 ClassLoader가 로딩한 타입(JDK 클래스, Spring 프레임워크)과 하위가 로딩한 타입이 동일해야 하므로 표준 parent-first를 따른다. `RestartClassLoader`는 개발 중 변경된 클래스를 최신 버전으로 교체하는 것이 목표다. parent-first면 Base ClassLoader에 이미 캐시된 이전 버전 클래스가 반환되므로, child-first로 변경된 `.class` 파일을 우선 로딩한다.
>
> **Q3.** `classpath.idx`가 없으면 ZIP 파일 엔트리 순서에 의존한다. ZIP 스펙은 순서를 보장하지 않으며 빌드 도구나 OS에 따라 달라질 수 있다. 동일 클래스가 여러 JAR에 존재할 때(버전 충돌) 어느 JAR의 클래스가 로딩될지 비결정적이 되어 환경에 따라 다른 버그가 발생한다. Spring Boot는 인덱스 파일이 없으면 경고를 출력하고 파일시스템 탐색 순서로 폴백한다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: JarLauncher vs WarLauncher ➡️](./02-jar-vs-war-launcher.md)**

</div>
