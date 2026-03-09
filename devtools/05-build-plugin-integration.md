# Build Plugin 통합 — Fat JAR 빌드 방식과 spring-boot:run의 ClassLoader 차이

---

## 🎯 핵심 질문

- Maven / Gradle Spring Boot Plugin이 Fat JAR를 빌드하는 내부 방식은?
- `spring-boot:run`(또는 `bootRun`)과 일반 `run`의 ClassLoader 차이는?
- `BOOT-INF/` 구조가 왜 필요한가?
- Layered JAR는 어떻게 Docker 이미지 빌드를 최적화하는가?
- `JarLauncher`가 중첩 JAR를 로딩하는 원리는?

---

## 🔬 내부 동작 원리

### 1. Fat JAR(Uber JAR) 구조

```
일반 JAR:
  com/example/App.class
  lib/ 디렉토리 없음 (의존성 JAR 포함 안 됨)
  → java -cp ".:lib/*.jar" com.example.App 으로 실행

Spring Boot Fat JAR:
  app.jar
  ├── META-INF/
  │   └── MANIFEST.MF
  ├── BOOT-INF/
  │   ├── classes/               ← 앱 코드
  │   │   └── com/example/...
  │   ├── classpath.idx          ← 클래스패스 순서 인덱스
  │   └── lib/                   ← 의존성 JAR (중첩 JAR)
  │       ├── spring-core-6.x.jar
  │       ├── spring-web-6.x.jar
  │       └── ...
  └── org/springframework/boot/loader/
      ├── JarLauncher.class      ← 진입점
      ├── LaunchedURLClassLoader.class
      └── ...

MANIFEST.MF:
  Main-Class: org.springframework.boot.loader.launch.JarLauncher
  Start-Class: com.example.App  ← 실제 앱 main 클래스
```

### 2. JarLauncher — 중첩 JAR 로딩 원리

```java
// java -jar app.jar 실행 시:
// JVM이 MANIFEST.MF의 Main-Class(JarLauncher)를 실행

public class JarLauncher extends ExecutableArchiveLauncher {

    @Override
    public void launch(String[] args) throws Exception {
        // ① 중첩 JAR URL ClassLoader 생성
        ClassLoader classLoader = createClassLoader(getClassPathUrls());

        // ② 앱 main 클래스 로딩
        // MANIFEST.MF의 Start-Class 조회
        String mainClass = getMainClass();

        // ③ main 메서드 실행 (새 ClassLoader로)
        launch(args, mainClass, classLoader);
    }

    @Override
    protected boolean isNestedArchive(Archive.Entry entry) {
        // BOOT-INF/lib/*.jar → 중첩 JAR로 인식
        // BOOT-INF/classes/ → 앱 클래스로 인식
        return entry.isDirectory()
            ? entry.getName().equals("BOOT-INF/classes/")
            : entry.getName().startsWith("BOOT-INF/lib/");
    }
}
```

```java
// LaunchedURLClassLoader — 중첩 JAR 내부 리소스 접근
// java.util.jar.JarFile은 중첩 JAR 직접 지원 안 함
// → Spring Boot가 커스텀 URL 핸들러 구현

// jar:file:app.jar!/BOOT-INF/lib/spring-core.jar!/
// → 중첩된 jar: URL을 처리하는 커스텀 URLStreamHandler
// → JarFile → NestedJarFile → 중첩 JAR 내부 클래스 로딩

// 등록:
// URL.setURLStreamHandlerFactory(new TomcatURLStreamHandlerFactory())
// 또는 JVM 시작 시 -Djava.protocol.handler.pkgs 설정
```

### 3. Gradle `bootRun` vs `run` ClassLoader 차이

```
일반 `run` (Gradle application 플러그인):
  → 빌드된 .class 파일 직접 실행
  → ClassLoader: AppClassLoader (단일)
  → DevTools: classpath에 있으면 동작
  → 모든 의존성이 같은 ClassLoader에 로딩

`bootRun` (Spring Boot Gradle Plugin):
  → Fat JAR 빌드 없이 바로 실행
  → ClassLoader: 두 개 분리
    ① Base ClassLoader → 의존성 JAR
    ② Restart ClassLoader → 앱 클래스 (변경 감지 대상)
  → DevTools Restart 기능 활성화
  → spring.devtools.restart.enabled=true 자동 적용

// build.gradle.kts
tasks.named<BootRun>("bootRun") {
    // JVM 인수 추가
    jvmArgs("-Xmx512m")
    // 환경변수
    environment("SPRING_PROFILES_ACTIVE", "dev")
    // 추가 classpath (리소스 즉시 반영)
    sourceResources(sourceSets["main"])
}
```

```xml
<!-- Maven: spring-boot:run vs exec:java -->
<!-- spring-boot:run -->
<!--   → Spring Boot Plugin이 실행 제어 -->
<!--   → DevTools 인식, Restart ClassLoader 분리 -->
<!--   → fork=false 시 같은 JVM에서 실행 -->

<!-- exec:java -->
<!--   → 단순 main 메서드 실행 -->
<!--   → 단일 ClassLoader -->
<!--   → DevTools Restart ClassLoader 분리 없음 -->

<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <jvmArguments>-Xmx512m</jvmArguments>
        <profiles>
            <profile>dev</profile>
        </profiles>
    </configuration>
</plugin>
```

### 4. Gradle Spring Boot Plugin 주요 태스크

```kotlin
// build.gradle.kts

plugins {
    id("org.springframework.boot") version "3.4.0"
    id("io.spring.dependency-management") version "1.1.6"
}

// bootJar — Fat JAR 생성
// ./gradlew bootJar
// → build/libs/app-1.0.0.jar (Fat JAR)
// → MANIFEST: Main-Class=JarLauncher, Start-Class=com.example.App

tasks.named<BootJar>("bootJar") {
    archiveFileName.set("app.jar")
    // 특정 main 클래스 지정 (자동 감지 안 되는 경우)
    mainClass.set("com.example.Application")
    // Layered JAR 설정
    layered {
        enabled.set(true)
    }
}

// bootRun — 개발 서버 실행
// ./gradlew bootRun
tasks.named<BootRun>("bootRun") {
    mainClass.set("com.example.Application")
    jvmArgs("-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005")
}

// DevTools — developmentOnly 설정
configurations {
    developmentOnly  // bootJar에 포함 안 됨
    runtimeClasspath {
        extendsFrom(configurations.developmentOnly.get())
    }
}

dependencies {
    developmentOnly("org.springframework.boot:spring-boot-devtools")
}
```

### 5. Layered JAR — Docker 이미지 최적화

```
일반 Fat JAR → Docker 이미지 빌드 문제:
  변경 1줄 → 전체 JAR(수백 MB) 재업로드
  → CI/CD 느림, 레이어 캐시 무효화

Layered JAR 구조:
  app.jar
  ├── BOOT-INF/
  │   ├── layers.idx          ← 레이어 정의
  │   └── lib/
  │       ├── dependencies/   ← layer1: 거의 안 변하는 의존성
  │       ├── snapshot-dependencies/  ← layer2: SNAPSHOT 의존성
  │       ├── resources/      ← layer3: 정적 리소스
  │       └── application/    ← layer4: 앱 코드 (가장 자주 변경)

레이어 변경 빈도 (낮음 → 높음):
  dependencies      (수개월에 한 번)
  snapshot-deps     (주 단위)
  resources         (일 단위)
  application       (시간 단위)

Docker 레이어 캐시 활용:
  앱 코드만 변경 → application 레이어만 재빌드/업로드
  → 나머지 3 레이어는 캐시 사용
  → 이미지 빌드/푸시 시간 대폭 단축
```

```dockerfile
# Layered JAR Dockerfile
FROM eclipse-temurin:21-jre AS builder
WORKDIR /application
COPY app.jar .
# Layered JAR 추출
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre
WORKDIR /application
# 레이어별로 COPY (Docker 레이어 캐시 활용)
COPY --from=builder /application/dependencies/ ./
COPY --from=builder /application/snapshot-dependencies/ ./
COPY --from=builder /application/resources/ ./
COPY --from=builder /application/application/ ./

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

```kotlin
// Layered JAR 활성화 (Spring Boot 2.4+ 기본 활성화)
tasks.named<BootJar>("bootJar") {
    layered {
        enabled.set(true)
        // 커스텀 레이어 정의
        application {
            intoLayer("spring-boot-loader") {
                include("org/springframework/boot/loader/**")
            }
            intoLayer("application")
        }
        dependencies {
            intoLayer("snapshot-dependencies") {
                include("*:*:*SNAPSHOT")
            }
            intoLayer("dependencies")
        }
        layerOrder.set(listOf(
            "dependencies", "snapshot-dependencies",
            "spring-boot-loader", "application"))
    }
}
```

### 6. bootJar vs jar 태스크 관계

```kotlin
// Spring Boot Plugin 적용 시 자동 설정:
// jar 태스크 → enabled = false (일반 JAR 생성 안 함)
// bootJar 태스크 → enabled = true (Fat JAR만 생성)

// 일반 JAR도 필요한 경우 (라이브러리 모듈):
tasks.named<Jar>("jar") {
    enabled = true
    // Classifier 없이 plain JAR
    archiveClassifier.set("")
}
tasks.named<BootJar>("bootJar") {
    archiveClassifier.set("boot")  // app-1.0.0-boot.jar
}

// 멀티 모듈 프로젝트 구조:
// :core (라이브러리) → 일반 JAR
// :web  (앱)         → Fat JAR (bootJar)
// :api  (라이브러리) → 일반 JAR
```

### 7. PropertiesLauncher — 외부 JAR/클래스 추가

```java
// JarLauncher 대신 PropertiesLauncher 사용 시:
// → loader.path로 실행 시 외부 의존성 추가 가능
// → 컨테이너 환경에서 드라이버 JAR 외부 주입 등

// MANIFEST.MF:
// Main-Class: org.springframework.boot.loader.launch.PropertiesLauncher

// 실행:
// java -Dloader.path="lib/,/opt/drivers/" -jar app.jar
// → lib/ 와 /opt/drivers/ 의 JAR도 classpath에 추가

// 사용 사례:
// DB 드라이버를 앱 JAR에 포함하지 않고 외부에서 주입
// 보안 정책상 특정 라이브러리를 분리 관리
```

---

## 💻 실험으로 확인하기

### 실험 1: Fat JAR 구조 확인

```bash
# Fat JAR 내부 구조 확인
./gradlew bootJar
jar -tf build/libs/app.jar | head -30

# 레이어 정보 확인
java -Djarmode=layertools -jar build/libs/app.jar list
# dependencies
# snapshot-dependencies
# resources
# application

# 레이어 추출 (내용 확인)
mkdir extracted && cd extracted
java -Djarmode=layertools -jar ../build/libs/app.jar extract
ls -la
# dependencies/  snapshot-dependencies/  resources/  application/
```

### 실험 2: bootRun vs run ClassLoader 비교

```java
// ClassLoader 정보 출력 Endpoint
@GetMapping("/classloader-info")
public Map<String, String> classLoaderInfo() {
    ClassLoader cl = getClass().getClassLoader();
    return Map.of(
        "type", cl.getClass().getSimpleName(),
        "parent", cl.getParent().getClass().getSimpleName()
    );
}

// bootRun 실행 시:
// type: RestartClassLoader
// parent: LaunchedURLClassLoader

// 일반 run 실행 시:
// type: AppClassLoader
// parent: PlatformClassLoader
```

### 실험 3: Layered JAR Docker 빌드 시간 비교

```bash
# 코드 변경 후 일반 Fat JAR Docker 빌드:
time docker build -t app:v1 -f Dockerfile.plain .
# → 전체 JAR 레이어 재빌드: ~30초 (JAR 크기에 따라)

# Layered JAR Docker 빌드 (dependencies 레이어 캐시 활용):
time docker build -t app:v2 -f Dockerfile.layered .
# → application 레이어만 재빌드: ~5초
```

---

## ⚙️ 설정 최적화 팁

```kotlin
// 운영 배포 최적화
tasks.named<BootJar>("bootJar") {
    // 1. Layered JAR 활성화 (Docker 최적화)
    layered { enabled.set(true) }

    // 2. 메타데이터 포함 (Actuator info 엔드포인트)
    // build.gradle.kts에 추가:
    // springBoot { buildInfo() }
    // → /actuator/info에 빌드 시간, 버전, git SHA 포함

    // 3. 압축 최소화 (컨테이너에서 불필요)
    // 기본: ZIP_STORED (압축 없음, 성능 최적)

    // 4. 실행 가능한 JAR 설정
    // Linux에서 ./app.jar 직접 실행 가능
    // → /etc/init.d 스크립트 또는 systemd 유닛으로 사용
}

// 5. GraalVM Native Image (Spring Boot 3+)
// bootJar 대신 nativeCompile 태스크 사용
// → JVM 없이 실행 가능한 네이티브 바이너리 생성
// → 시작 시간 수십 ms, 메모리 사용량 극적 감소
// → Chapter 7에서 상세 다룸
```

---

## 📌 핵심 정리

```
Fat JAR 구조
  BOOT-INF/classes/    앱 코드
  BOOT-INF/lib/*.jar   의존성 JAR (중첩 JAR)
  org/springframework/boot/loader/  JarLauncher

JarLauncher 역할
  MANIFEST의 Main-Class로 진입
  LaunchedURLClassLoader 생성 (중첩 JAR 지원)
  Start-Class(앱 main) 실행

bootRun vs run
  bootRun: Base + Restart ClassLoader 분리 → DevTools 완전 지원
  run:     단일 AppClassLoader → DevTools Restart 분리 없음

Layered JAR
  레이어 변경 빈도순 분리 (deps → snapshot → resources → app)
  Docker 빌드 시 앱 코드 레이어만 재빌드
  java -Djarmode=layertools -jar app.jar extract 로 추출

developmentOnly
  Gradle: developmentOnly 설정 → bootJar 미포함
  → DevTools 운영 JAR 포함 방지
```

---

## 🤔 생각해볼 문제

**Q1.** Fat JAR를 `java -jar`로 실행할 때 `MANIFEST.MF`에 일반 `Main-Class`가 아닌 `JarLauncher`가 지정된 이유는 무엇인가?

**Q2.** Layered JAR에서 `snapshot-dependencies` 레이어가 `dependencies`와 분리된 이유는 무엇인가?

**Q3.** 멀티 모듈 Gradle 프로젝트에서 `:core` 라이브러리 모듈에 Spring Boot Plugin을 적용하면 어떤 문제가 발생하는가?

> 💡 **해설**
>
> **Q1.** Java의 `java -jar` 명령은 JAR 내부에 다른 JAR를 중첩(nested)해서 클래스를 로딩하는 기능을 기본 제공하지 않는다. `java.util.jar.JarFile`은 중첩 JAR의 클래스를 직접 로딩할 수 없다. `BOOT-INF/lib/`에 있는 `spring-core.jar` 같은 의존성 JAR 내부의 클래스를 로딩하려면 커스텀 `URLStreamHandler`와 `ClassLoader`가 필요하다. `JarLauncher`가 이 역할을 한다. `LaunchedURLClassLoader`를 생성해 중첩 JAR URL 형식(`jar:file:app.jar!/BOOT-INF/lib/spring-core.jar!/`)을 처리하도록 만들고, 이후 실제 앱 `main` 메서드를 이 ClassLoader 컨텍스트에서 실행한다.
>
> **Q2.** SNAPSHOT 의존성은 같은 버전 문자열이어도 내용이 변경될 수 있다. CI 빌드 때마다 최신 SNAPSHOT을 받아야 할 수 있어 정식 release 의존성보다 훨씬 자주 바뀐다. 분리하지 않으면 SNAPSHOT 하나가 바뀔 때 안정적인 release 의존성 레이어까지 무효화되어 Docker 캐시 효율이 낮아진다. 분리하면 `dependencies` 레이어는 release 버전 업그레이드 시에만, `snapshot-dependencies` 레이어는 SNAPSHOT 갱신 시에만 재빌드된다.
>
> **Q3.** Spring Boot Plugin을 적용하면 `jar` 태스크가 비활성화되고 `bootJar`만 활성화된다. 라이브러리 모듈은 다른 모듈에서 의존해야 하므로 일반 JAR가 필요하다. `bootJar`로 생성된 Fat JAR는 `JarLauncher`가 진입점이고 `BOOT-INF/classes/` 내부에 클래스가 있어, 다른 모듈의 `implementation(project(":core"))`에서 클래스를 찾지 못한다. 라이브러리 모듈에는 Spring Boot Plugin을 적용하지 않거나, `bootJar.enabled = false`와 `jar.enabled = true`를 명시적으로 설정해야 한다.

---

<div align="center">

**[⬅️ 이전: Remote Debug & Remote Update](./04-remote-debug-update.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — Fat JAR 구조 ➡️](../packaging-deployment/01-fat-jar-structure.md)**

</div>
