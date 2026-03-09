# Cloud Native Buildpacks — Dockerfile 없이 OCI 이미지 생성

---

## 🎯 핵심 질문

- `spring-boot:build-image` 명령이 내부적으로 어떤 과정을 거치는가?
- Buildpack이 언어/런타임을 자동 감지하는 방법은?
- Dockerfile 없이 OCI 표준 이미지를 생성할 수 있는 원리는?
- Buildpack 기반 이미지가 Layered JAR Dockerfile과 다른 점은?
- 베이스 이미지 보안 패치를 Buildpack으로 어떻게 처리하는가?

---

## 🔍 왜 이게 존재하는가

```
Dockerfile의 문제점:
  개발자마다 Dockerfile 작성 방식 다름 → 품질 편차
  보안 모범 사례 누락 (non-root 실행, 최소 이미지 등)
  베이스 이미지 보안 패치: 모든 팀이 각자 업데이트해야 함
  Node.js, Java, Python 등 언어별 최적화 지식 필요

Cloud Native Buildpacks (CNB) 해결:
  표준화된 이미지 빌드 프로세스
  플랫폼팀이 Buildpack 관리 → 개발팀은 앱 코드만 집중
  보안 패치: Buildpack 업데이트 → 재빌드로 일괄 적용
  OCI 표준 이미지 보장
```

---

## 🔬 내부 동작 원리

### 1. Buildpack 아키텍처

```
빌드 과정 구성 요소:

Builder (빌더 이미지):
  Buildpack 모음 + 빌드 환경
  Spring Boot 기본: paketobuildpacks/builder:base

Stack (스택):
  Build Image  빌드 환경 (컴파일, 패키징)
  Run Image    런타임 환경 (최소 이미지, non-root)

Buildpack:
  detect  단계: 이 앱에 적용 가능한가? (pom.xml 있으면 Java)
  build   단계: 앱을 레이어로 변환

Lifecycle (lifecycler):
  detect → analyze → restore → build → export
  각 단계를 표준화된 인터페이스로 실행
```

### 2. spring-boot:build-image 실행 흐름

```
./gradlew bootBuildImage (또는 mvn spring-boot:build-image)
  ↓
① Docker 데몬에서 Builder 이미지 풀
   paketobuildpacks/builder-noble-java-tiny:latest

② 소스(Fat JAR) → Docker 볼륨으로 전달

③ detect 단계 (어떤 Buildpack 적용?)
   - 파일 구조 분석
   - app.jar 발견 → Java Buildpack 선택
   - JVM 버전 감지 (MANIFEST.MF의 Build-Jdk 항목)

④ build 단계 (Buildpack별 실행)
   ① BellSoft Liberica JRE Buildpack: JRE 설치
   ② Spring Boot Buildpack:
      - Layered JAR 분리 (jarmode=layertools)
      - 레이어별 이미지 레이어 생성
   ③ Process Types Buildpack: 실행 명령 설정

⑤ export 단계
   → OCI 이미지 생성
   → 로컬 Docker 데몬에 로드 또는 Registry 푸시

결과: ghcr.io/myorg/myapp:latest (OCI 이미지)
```

### 3. Spring Boot Buildpack의 레이어 처리

```
Spring Boot Buildpack 내부:
  Fat JAR 입력
  → java -Djarmode=layertools -jar app.jar extract
  → dependencies/, snapshot-dependencies/, resources/, application/

각 레이어를 별도 이미지 레이어로 매핑:
  OCI 이미지 레이어 1: JRE (BellSoft Liberica)
  OCI 이미지 레이어 2: dependencies/
  OCI 이미지 레이어 3: snapshot-dependencies/
  OCI 이미지 레이어 4: resources/
  OCI 이미지 레이어 5: application/

→ 수동 Layered JAR Dockerfile과 동일한 최적화 효과
→ 자동으로 처리 (Dockerfile 불필요)
```

### 4. 빌드 설정

```kotlin
// build.gradle.kts
tasks.named<BootBuildImage>("bootBuildImage") {
    // 이미지 이름
    imageName.set("ghcr.io/myorg/myapp:{projectVersion}")

    // 빌더 이미지 지정 (기본: paketobuildpacks)
    builder.set("paketobuildpacks/builder-noble-java-tiny:latest")
    // tiny: 최소 런타임 이미지 (bash, package manager 없음)
    // base: 표준 런타임 이미지

    // 환경변수로 Buildpack 설정
    environment.set(mapOf(
        // JVM 메모리 설정
        "BPL_JVM_THREAD_COUNT" to "50",
        "BPL_JVM_HEAD_ROOM" to "10",      // 시스템 메모리 10% 여유
        // JVM 버전 강제 지정
        "BP_JVM_VERSION" to "21",
        // Spring Boot Actuator 프로파일 활성화
        "BPE_SPRING_PROFILES_ACTIVE" to "prod"
    ))

    // Docker 레지스트리 인증
    docker {
        publishRegistry {
            username.set(System.getenv("REGISTRY_USERNAME"))
            password.set(System.getenv("REGISTRY_PASSWORD"))
            url.set("https://ghcr.io")
        }
    }

    // 빌드 후 자동 푸시
    publish.set(true)
}
```

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <image>
            <name>ghcr.io/myorg/myapp:${project.version}</name>
            <builder>paketobuildpacks/builder-noble-java-tiny</builder>
            <env>
                <BP_JVM_VERSION>21</BP_JVM_VERSION>
                <BPL_JVM_THREAD_COUNT>50</BPL_JVM_THREAD_COUNT>
            </env>
        </image>
    </configuration>
</plugin>
```

### 5. 보안 패치 자동 적용

```
Buildpack의 보안 패치 워크플로우:

① CVE 발견: JRE에 취약점
② Buildpack 제공자(Paketo)가 패치된 JRE로 Buildpack 업데이트
③ 팀에서 Buildpack 버전 업데이트
④ 재빌드: ./gradlew bootBuildImage
   → 패치된 JRE가 자동으로 이미지에 포함
⑤ 배포

Dockerfile 방식과 비교:
  Dockerfile:
    모든 팀이 각자 FROM 이미지 업데이트
    → 누락 팀 발생 가능, 추적 어려움
  Buildpack:
    플랫폼팀이 Buildpack 업데이트
    → 재빌드만으로 일괄 적용
    → 사용 Buildpack 버전 추적 가능 (SBOM 내장)
```

### 6. SBOM (Software Bill of Materials)

```bash
# Buildpack이 생성한 이미지에는 SBOM이 자동 포함
# → 이미지에 포함된 모든 컴포넌트 목록

# SBOM 추출
docker sbom myapp:latest

# 출력:
# NAME                    VERSION    TYPE
# BellSoft Liberica JRE   21.0.5+11  java
# spring-boot             3.4.0      java-archive
# spring-core             6.2.0      java-archive
# ...

# CVE 스캔 (Trivy 등)
trivy image myapp:latest
# → SBOM 기반으로 취약점 분석
```

---

## 💻 실험으로 확인하기

### 실험 1: 이미지 빌드

```bash
./gradlew bootBuildImage

# 출력 확인:
# ===> DETECTING
# 8 of 24 buildpacks participating
# paketo-buildpacks/ca-certificates    3.6.3
# paketo-buildpacks/bellsoft-liberica  10.4.1
# paketo-buildpacks/syft               1.42.0
# ...

# ===> BUILDING
# Paketo Buildpack for BellSoft Liberica 10.4.1
# Resolving JRE version → 21
# Downloading BellSoft Liberica JRE 21.0.5+11

# 이미지 확인
docker images | grep myapp
docker inspect myapp:latest | jq '.[0].Config.Labels'
```

### 실험 2: 이미지 레이어 구조 확인

```bash
# 이미지 레이어 분석
docker history myapp:latest

# IMAGE          CREATED BY        SIZE
# xxx  /cnb/process/web              0B
# xxx  Spring Boot                   2MB   (application 레이어)
# xxx  Spring Boot                   3MB   (resources 레이어)
# xxx  Spring Boot                 180MB   (dependencies 레이어)
# xxx  BellSoft Liberica JRE        80MB

# 레이어 캐시 재활용 확인 (코드 변경 후 재빌드)
./gradlew bootBuildImage
# → dependencies 레이어: 캐시 재사용 (빠름)
# → application 레이어: 재빌드 (느림)
```

### 실험 3: Native Image Buildpack

```bash
# Native Image 빌드 (GraalVM Buildpack)
tasks.named<BootBuildImage>("bootBuildImage") {
    builder.set("paketobuildpacks/builder-noble-java-tiny")
    environment.set(mapOf(
        "BP_NATIVE_IMAGE" to "true"  // Native Image Buildpack 활성화
    ))
}

./gradlew bootBuildImage
# → GraalVM native-image 컴파일 (수십 분)
# → 결과: JVM 없는 네이티브 실행 이미지
```

---

## ⚙️ 설정 최적화 팁

```kotlin
// CI/CD 환경 최적화
tasks.named<BootBuildImage>("bootBuildImage") {
    // 캐시 볼륨 지정 (재빌드 시 재사용)
    buildCache {
        volume {
            name.set("cache-${rootProject.name}.build")
        }
    }
    launchCache {
        volume {
            name.set("cache-${rootProject.name}.launch")
        }
    }

    // 빌드 로그 상세도
    verboseLogging.set(true)  // CI에서 디버깅용

    // 이미지 태그 전략
    tags.set(listOf(
        "ghcr.io/myorg/myapp:${version}",
        "ghcr.io/myorg/myapp:latest"
    ))
}
```

---

## 📌 핵심 정리

```
Cloud Native Buildpacks 핵심 흐름
  Fat JAR 입력
  → Builder 이미지에서 detect/build/export 단계 실행
  → Spring Boot Buildpack: Layered JAR 자동 분리
  → OCI 표준 이미지 출력

Dockerfile 대비 장점
  표준화: 플랫폼팀이 Buildpack 관리
  보안: 패치된 Buildpack으로 재빌드 → 일괄 적용
  SBOM 자동 생성: 이미지 내 컴포넌트 추적
  Layered JAR 최적화 자동 적용

주요 환경변수
  BP_JVM_VERSION     JVM 버전 지정
  BPL_JVM_THREAD_COUNT  스레드 수 (메모리 계산 기반)
  BPL_JVM_HEAD_ROOM  시스템 메모리 여유율
  BP_NATIVE_IMAGE    Native Image 빌드 활성화
```

---

## 🤔 생각해볼 문제

**Q1.** Cloud Native Buildpacks로 빌드한 이미지에서 JVM 메모리 설정(`-Xmx`)을 어떻게 관리하는가?

**Q2.** Buildpack 기반 이미지 빌드가 Docker buildkit 기반 Layered JAR Dockerfile보다 느린 경우가 있다. 왜 그런가?

**Q3.** Paketo Buildpack의 `tiny` 스택과 `base` 스택의 차이는? 어떤 상황에서 `base`를 선택해야 하는가?

> 💡 **해설**
>
> **Q1.** Paketo Buildpack의 JVM Buildpack은 컨테이너 메모리 제한을 자동으로 감지하고 JVM 메모리 설정을 계산한다. 환경변수 `BPL_JVM_THREAD_COUNT`(스레드 수)와 `BPL_JVM_HEAD_ROOM`(시스템 여유 메모리 비율)으로 `-Xmx` 등을 자동 산정한다. 예를 들어 컨테이너 메모리 512MB, 스레드 50개, 헤드룸 10% 설정 시 JVM이 자동으로 적절한 `-Xmx`를 계산한다. 수동으로 `JAVA_TOOL_OPTIONS=-Xmx256m`을 환경변수로 전달해 오버라이드할 수도 있다.
>
> **Q2.** Buildpack 빌드는 Builder 이미지를 Docker 레지스트리에서 풀하는 시간, detect/analyze/restore/build/export 각 단계를 별도 컨테이너에서 실행하는 오버헤드, Buildpack 자체의 초기화 비용이 추가된다. 첫 빌드 시 특히 느리고, 캐시 볼륨을 설정하면 이후 빌드가 빨라진다. 반면 Docker buildkit은 로컬에서 직접 레이어를 처리해 오버헤드가 적다. 대규모 CI 환경에서는 캐시 전략이 핵심이다.
>
> **Q3.** `tiny` 스택은 최소한의 런타임 환경(C 표준 라이브러리 수준)으로 공격 표면을 최소화한다. 셸, 패키지 매니저, 대부분의 Unix 유틸리티가 없다. `base` 스택은 일반적인 Linux 환경으로 셸 스크립트 실행, 추가 패키지 설치, `exec`으로 다른 프로세스 실행이 가능하다. `tiny`를 기본 선택하되, 앱이 셸 스크립트를 실행하거나(`ProcessBuilder`, `Runtime.exec()`), 외부 바이너리(ffmpeg, imagemagick 등)를 호출하거나, `proc filesystem` 접근이 필요한 경우에는 `base`를 선택한다.

---

<div align="center">

**[⬅️ 이전: Native Image with GraalVM](./04-native-image-graalvm.md)** | **[홈으로 🏠](../README.md)** | **[다음: Production-ready Checklist ➡️](./06-production-checklist.md)**

</div>
