# Native Image with GraalVM — AOT 컴파일과 리플렉션 힌트 자동 생성

---

## 🎯 핵심 질문

- AOT(Ahead-Of-Time) 컴파일이 JIT 컴파일과 다른 점은?
- GraalVM `native-image`가 Spring Boot 앱을 컴파일하는 과정은?
- 리플렉션·프록시·직렬화 힌트가 필요한 이유는?
- Spring Boot AOT 엔진이 힌트를 자동 생성하는 원리는?
- Native Image의 제약(Closed World Assumption)은 무엇인가?

---

## 🔍 왜 이게 존재하는가

```
JVM 앱의 기동 특성:
  JVM 초기화 → 클래스 로딩 → JIT 컴파일 워밍업
  → 기동 시간: 수 초~수십 초
  → 첫 요청 응답 느림 (JIT 워밍업 전)
  → 메모리: JVM 런타임 + 클래스 메타데이터

Native Image 특성:
  컴파일 타임에 전체 앱을 네이티브 바이너리로 변환
  → 기동 시간: 수십~수백 ms
  → 메모리 사용량 대폭 감소 (JVM 런타임 없음)
  → 컨테이너 환경에서 빠른 스케일아웃

트레이드오프:
  빌드 시간: 수십 분 (vs JIT: 수초)
  최대 처리량: JIT보다 낮음 (JIT 최적화 없음)
  동적 기능 제한: 리플렉션, 동적 프록시, 클래스 로딩
```

---

## 😱 흔한 오해 또는 설정 실수

```java
// ❌ 오해: Native Image는 JVM 앱보다 항상 빠르다
// 기동 시간: Native가 압도적으로 빠름 (수십 ms vs 수 초)
// 최대 처리량: JIT 최적화 없어 JVM보다 낮을 수 있음
// → 장기 실행 고처리량 서비스에는 JVM이 유리할 수 있음

// ❌ 오해: Spring Boot AOT가 모든 힌트를 자동 생성한다
// 놓치는 경우:
//   Class.forName("com.example." + dynamicName)  // 동적 문자열
//   외부 라이브러리 내부 리플렉션 사용
//   @ConditionalOnProperty로 AOT 시뮬레이션과 런타임 조건 불일치

// ❌ 실수: native-image 빌드 메모리 부족
// 기본 JVM 힙으로 빌드 시 OutOfMemoryError
// → jvmArgs.add("-Xmx8g") 최소 8GB 권장
```

---

## ✨ 올바른 이해와 설정

```kotlin
// 올바른 Native Image 빌드 설정
plugins {
    id("org.graalvm.buildtools.native") version "0.10.3"
}

graalvmNative {
    binaries.named("main") {
        imageName.set("app")
        jvmArgs.add("-Xmx8g")           // 빌드 메모리 충분히 확보
        buildArgs.add("--no-fallback")  // JVM 폴백 비활성화 (실패 시 명확히)
        buildArgs.add("-H:+ReportExceptionStackTraces")
    }
    // 빌드 전 AOT 호환성 테스트
    testSupport.set(true)
}
```

```bash
# 힌트 누락 감지 — native-image-agent로 런타임 수집
java -agentlib:native-image-agent=config-output-dir=./hints \
     -jar build/libs/app.jar
# 앱 전체 기능 실행 후 Ctrl+C
# → hints/reflect-config.json 생성
# → src/main/resources/META-INF/native-image/에 복사
```

---

## 🔬 내부 동작 원리

### 1. Closed World Assumption

```
GraalVM native-image의 핵심 전제:

  빌드 시점에 실행될 모든 코드를 알 수 있다
  → 런타임에 새 클래스를 로딩하지 않음
  → 런타임에 새 JAR를 추가하지 않음
  → 동적으로 생성되는 코드 없음

문제:
  Spring Framework는 동적 기능을 광범위하게 사용
  - 리플렉션: Bean 생성, 의존성 주입
  - 동적 프록시: @Transactional, @Cacheable, AOP
  - CGLIB 프록시: 클래스 기반 프록시 (서브클래스 생성)
  - 직렬화: JSON, JPA 엔티티
  - 리소스 로딩: classpath:*.properties

  → native-image는 이 동적 기능들을 정적 분석으로 파악 불가
  → 명시적 힌트(reachability metadata) 필요
```

### 2. Spring Boot AOT 엔진 — 힌트 자동 생성

```java
// Spring Boot 3.x: AOT 처리 단계를 빌드 프로세스에 통합
// ./gradlew nativeCompile 시:
//   ① processAot 태스크 실행 (Spring AOT 엔진)
//   ② native-image 컴파일러 실행

// AOT 엔진 동작:
// SpringApplicationAotProcessor
//   → ApplicationContext를 "시뮬레이션" 실행
//   → Bean 정의 분석 → 소스 코드 생성
//   → 리플렉션 힌트 등록

// 생성되는 결과물 (build/generated/aotSources/):
// ① Bean 팩토리 소스 코드 (리플렉션 제거)
//    AppBeanFactoryRegistrations.java
// ② 리플렉션/프록시/리소스 힌트 파일
//    reflect-config.json
//    proxy-config.json
//    resource-config.json
```

```java
// AOT가 생성하는 Bean 등록 코드 예시
// Before (런타임 리플렉션):
//   ApplicationContext가 "UserService" 이름으로
//   리플렉션으로 클래스 찾고 생성자 호출

// After (AOT 생성 코드):
public class AppBeanFactoryRegistrations
        implements BeanFactoryInitializationAotContribution {

    @Override
    public void applyTo(GenerationContext ctx, BeanFactoryInitializationCode code) {
        // 리플렉션 없이 직접 Bean 생성 코드 생성
        code.getMethodReference("registerUserService",
            (method) -> {
                method.addStatement(
                    "context.registerBean(UserService.class, UserService::new)");
            });
    }
}
// → 리플렉션 없이 직접 생성자 호출
// → native-image 분석 가능 (정적 코드)
```

### 3. 리플렉션 힌트 — reflect-config.json

```json
// build/generated/aotResources/META-INF/native-image/reflect-config.json
// AOT 엔진이 자동 생성 + 사용자 힌트 병합

[
  {
    "name": "com.example.User",
    "allDeclaredFields": true,        // Jackson 직렬화
    "allDeclaredConstructors": true,  // 기본 생성자
    "allDeclaredMethods": true        // Getter/Setter
  },
  {
    "name": "com.example.UserRepository",
    "allDeclaredMethods": true        // JPA 리포지토리 메서드
  },
  {
    "name": "org.springframework.data.jpa.repository.JpaRepository",
    "allDeclaredMethods": true
  }
]
```

```java
// 수동 힌트 등록 (AOT 엔진이 놓친 경우)
@ImportRuntimeHints(MyRuntimeHints.class)
@SpringBootApplication
public class Application { ... }

public class MyRuntimeHints implements RuntimeHintsRegistrar {

    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {

        // 리플렉션 힌트
        hints.reflection()
            .registerType(MyDynamicClass.class,
                MemberCategory.INVOKE_DECLARED_CONSTRUCTORS,
                MemberCategory.INVOKE_DECLARED_METHODS);

        // 리소스 힌트 (classpath 파일)
        hints.resources()
            .registerPattern("templates/*.html")
            .registerPattern("config/*.yml");

        // 동적 프록시 힌트
        hints.proxies()
            .registerJdkProxy(MyInterface.class);

        // 직렬화 힌트
        hints.serialization()
            .registerType(MySerializable.class);
    }
}
```

### 4. native-image 컴파일 과정

```
빌드 입력:
  앱 클래스 + 의존성 JAR
  + AOT 생성 소스 코드
  + reachability metadata (reflect-config.json 등)

native-image 분석 단계:
  ① Static Analysis (정적 분석)
     엔트리포인트(main)부터 도달 가능한 모든 코드 탐색
     → 사용되지 않는 클래스/메서드 제거 (Dead Code Elimination)

  ② Heap Snapshotting
     초기화 코드 실행 → 힙 상태 스냅샷
     → 런타임에 재실행 없이 미리 계산된 객체 사용

  ③ 코드 생성
     정적 분석 결과 + 힌트 → 네이티브 바이너리 생성
     (GCC 또는 LLVM 백엔드)

출력:
  단일 실행 파일 (Linux: ELF, macOS: Mach-O)
  → JVM 없이 실행 가능
  → 크기: 수십~수백 MB (앱 크기에 따라)
```

### 5. 빌드 및 실행

```kotlin
// build.gradle.kts
plugins {
    id("org.springframework.boot") version "3.4.0"
    id("io.spring.dependency-management") version "1.1.6"
    id("org.graalvm.buildtools.native") version "0.10.3"
}

// Native 빌드 설정
graalvmNative {
    binaries {
        named("main") {
            imageName.set("app")
            buildArgs.add("-O2")  // 최적화 레벨
            buildArgs.add("--initialize-at-build-time=com.example.Config")
            // 빌드 시점 초기화 (런타임보다 빠름)
        }
    }
}
```

```bash
# GraalVM 설치 (SDKMAN)
sdk install java 21.0.5-graalce

# Native Image 빌드 (수십 분 소요)
./gradlew nativeCompile

# 실행 (JVM 없이)
./build/native/nativeCompile/app

# 기동 시간 비교:
# JVM: "Started App in 3.245 seconds"
# Native: "Started App in 0.083 seconds"

# 메모리 비교 (RSS):
# JVM: ~250MB
# Native: ~50MB
```

### 6. Native Image 테스트

```kotlin
// Native Image 통합 테스트
// (실제 네이티브 바이너리로 테스트 실행)
tasks.named<Test>("nativeTest") {
    // ./gradlew nativeTest
    // 네이티브 이미지로 테스트 실행
    // → JVM과 다른 동작 조기 발견
}
```

```java
// @NativeHint — 개별 테스트에서 힌트 추가
@SpringBootTest
@TestPropertySource(properties = "spring.aot.enabled=true")
class NativeCompatibilityTest {

    @Test
    void contextLoads() {
        // AOT 모드로 컨텍스트 로딩 테스트
        // native-image 빌드 전 호환성 확인
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: AOT 생성 코드 확인

```bash
# AOT 처리만 실행 (native-image 제외)
./gradlew processAot

# 생성된 파일 확인
ls build/generated/aotSources/com/example/
# Application__BeanDefinitions.java
# Application__BeanFactoryRegistrations.java

# 생성된 힌트 파일
ls build/generated/aotResources/META-INF/native-image/
# reflect-config.json
# proxy-config.json
# resource-config.json
```

### 실험 2: 기동 시간 및 메모리 비교

```bash
# JVM 기동
java -jar build/libs/app.jar &
PID=$!
# "Started App in X seconds" 로그 확인
ps -o pid,rss $PID | tail -1  # RSS 메모리 (KB)

# Native Image 기동
./build/native/nativeCompile/app &
PID=$!
# "Started App in X seconds" 로그 확인
ps -o pid,rss $PID | tail -1
```

### 실험 3: 힌트 누락 감지

```bash
# GraalVM Agent로 실행 시 힌트 자동 수집
java -agentlib:native-image-agent=config-output-dir=./hints \
     -jar build/libs/app.jar

# 앱 사용 후 Ctrl+C
# → hints/ 디렉토리에 reflect-config.json 등 생성
# → 이를 src/main/resources/META-INF/native-image/ 에 복사
```

---

## ⚙️ 설정 최적화 팁

```yaml
# application.yml - Native Image 최적화
spring:
  aot:
    enabled: true  # AOT 모드 명시 (기본: native 빌드 시 자동)

# Native 빌드 JVM 메모리 (build.gradle.kts)
graalvmNative {
    binaries.named("main") {
        jvmArgs.add("-Xmx8g")  # 빌드에 8GB 메모리 (권장)
        buildArgs.addAll(
            "--no-fallback",           # JVM 폴백 비활성화
            "--install-exit-handlers",
            "-H:+ReportExceptionStackTraces"
        )
    }
}
```

---

## 🤔 트레이드오프

```
Native Image vs JVM
  기동 시간:   Native 압도적 우위 (수십 ms vs 수 초)
  메모리:      Native 우위 (JVM 런타임 없음)
  최대 처리량: JVM 우위 (JIT 프로파일 최적화)
  빌드 시간:   JVM 압도적 우위 (수초 vs 수십 분)
  동적 기능:   JVM 우위 (리플렉션, 동적 클래스 로딩 자유)

적합한 환경
  Native: 서버리스(AWS Lambda), 빠른 스케일아웃, 메모리 민감
  JVM:    장기 실행 고처리량 서비스, 복잡한 리플렉션 의존 레거시

Spring Boot AOT 엔진의 한계
  Closed World 가정을 만족하기 위한 추가 작업 필요
  외부 라이브러리 호환성 검증 필수 (GraalVM Reachability Metadata)
  → 새 의존성 추가마다 Native 빌드 테스트 권장

빌드 파이프라인 영향
  수십 분 빌드 시간 → CI/CD 파이프라인 설계 변경 필요
  별도 Native 빌드 스테이지 운영, 충분한 빌드 머신 사양 필요
```

---

## 📌 핵심 정리

```
AOT vs JIT
  JIT: 런타임에 자주 실행되는 코드 최적화 → 워밍업 후 최고 성능
  AOT: 빌드 타임에 전체 컴파일 → 즉시 최고 성능, 최대 처리량은 낮음

Spring Boot AOT 엔진 (processAot)
  Bean 정의 분석 → 소스 코드 생성 (리플렉션 제거)
  reflect/proxy/resource-config.json 자동 생성
  → native-image 빌드 입력으로 사용

힌트 등록 방법
  자동: Spring Boot AOT 엔진 (대부분 자동 처리)
  수동: @ImportRuntimeHints + RuntimeHintsRegistrar
  에이전트: native-image-agent로 런타임 수집

Native Image 장단점
  장점: 기동 수십ms, 메모리 대폭 감소
  단점: 빌드 수십 분, JIT 최대 처리량 미달, 동적 기능 제한
  적합: 서버리스, 빠른 스케일아웃, 메모리 민감 환경
```

---

## 🤔 생각해볼 문제

**Q1.** Native Image로 빌드한 앱에서 `Class.forName("com.example.Plugin")`이 런타임에 실행되면 어떻게 되는가?

**Q2.** Spring Boot AOT 엔진이 힌트를 자동 생성해도 완벽하지 않은 경우가 있다. 어떤 상황에서 수동 힌트 등록이 필요한가?

**Q3.** Native Image의 "Heap Snapshotting"이란 무엇이며, `@Bean` 메서드에 `@Lazy`를 붙이면 어떤 영향이 있는가?

> 💡 **해설**
>
> **Q1.** `reflect-config.json`에 `com.example.Plugin` 클래스에 대한 리플렉션 힌트가 등록되어 있으면 정상 동작한다. 힌트가 없으면 `ClassNotFoundException`이 발생한다. Native Image는 빌드 시 도달 불가능한 코드를 제거하므로, 힌트 없이 `Class.forName()`으로 동적 로딩을 시도하면 해당 클래스 자체가 바이너리에 포함되지 않아 찾을 수 없다.
>
> **Q2.** Spring AOT 엔진이 놓치는 경우: 문자열 상수로 클래스 이름을 전달하는 리플렉션(`Class.forName("com.example." + name)`), 외부 라이브러리가 내부적으로 리플렉션 사용(드라이버 로딩 등), 조건부 Bean(`@ConditionalOnProperty`)이 AOT 시뮬레이션과 런타임에서 다르게 평가되는 경우, YAML/Properties 파일을 직접 파싱해 클래스명을 읽는 경우가 있다. 이때 `RuntimeHintsRegistrar`로 수동 등록하거나 `native-image-agent`로 런타임 힌트를 수집해야 한다.
>
> **Q3.** Heap Snapshotting은 `native-image` 빌드 중 초기화 코드(static initializer 등)를 실제로 실행하고 그 결과 힙 상태를 바이너리에 직접 포함시키는 기술이다. 런타임에 초기화를 다시 하지 않아도 되므로 기동이 빠르다. `@Lazy` Bean은 첫 참조 시점까지 생성을 지연하므로 Heap Snapshotting 단계에서 초기화되지 않는다. 결과적으로 `@Lazy` Bean의 초기화 비용이 런타임 첫 접근 시로 미뤄지며, 기동 시간 단축 효과가 줄어들 수 있다. Native 환경에서는 필요하지 않으면 `@Lazy`를 제거하는 것이 일반적이다.

---

<div align="center">

**[⬅️ 이전: Layered JAR for Docker](./03-layered-jar-docker.md)** | **[홈으로 🏠](../README.md)** | **[다음: Cloud Native Buildpacks ➡️](./05-cloud-native-buildpacks.md)**

</div>
