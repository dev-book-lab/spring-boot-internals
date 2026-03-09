# Layered JAR for Docker 최적화 — 레이어 분리 원리와 빌드 시간 단축 전략

---

## 🎯 핵심 질문

- `jarmode=layertools`로 레이어를 분리하는 원리는?
- `layers.idx`는 어떻게 파일을 레이어에 배치하는가?
- 의존성 레이어 캐싱으로 Docker 빌드 시간을 얼마나 줄일 수 있는가?
- 기본 4개 레이어(dependencies, snapshot-dependencies, resources, application)의 변경 빈도는 왜 다른가?
- 커스텀 레이어를 정의해야 하는 경우는?

---

## 🔍 왜 이게 존재하는가

```
일반 Fat JAR → Docker 문제:
  FROM eclipse-temurin:21
  COPY app.jar .           ← 한 레이어 (200MB+)
  ENTRYPOINT ["java", "-jar", "app.jar"]

  코드 1줄 변경 → app.jar 전체(200MB) 재빌드 + 재업로드
  → CI/CD 파이프라인 10~30분 소요
  → Docker Hub, ECR 등 레지스트리에 매번 대용량 전송

Layered JAR 해결:
  의존성(180MB) 레이어 → 거의 변하지 않음 → 캐시 재사용
  앱 코드(1MB) 레이어  → 자주 변함 → 재빌드
  → 코드 변경 시 1MB만 재업로드
```

---

## 🔬 내부 동작 원리

### 1. layers.idx — 레이어 정의 파일

```yaml
# BOOT-INF/layers.idx (Spring Boot가 자동 생성)
# 형식: 레이어이름 → 포함할 파일 패턴

- "dependencies":
  - "BOOT-INF/lib/"          # 정식 release 의존성 JAR
- "snapshot-dependencies":
  - "BOOT-INF/lib/"          # SNAPSHOT 의존성 (별도 분리)
- "resources":
  - "META-INF/resources/"
  - "resources/"
  - "static/"
  - "public/"
  - "BOOT-INF/classes/"      # 정적 리소스
- "application":
  - "org/"                   # JarLauncher 클래스
  - "BOOT-INF/classes/"      # 앱 코드
  - "BOOT-INF/classpath.idx"
  - "BOOT-INF/layers.idx"
  - "META-INF/"
```

```
레이어 변경 빈도 (낮음 → 높음):

dependencies       release 버전 JAR → 수개월에 한 번 변경
snapshot-deps      SNAPSHOT JAR     → 주 단위 변경 가능
resources          정적 파일        → 간헐적 변경
application        앱 코드          → 수시로 변경

Docker 레이어 캐시 원리:
  낮은 레이어(변경 빈도 낮음)가 먼저 → 캐시 활용 최대화
  높은 레이어(변경 빈도 높음)가 나중 → 재빌드 최소화
```

### 2. jarmode=layertools — 레이어 추출

```bash
# Fat JAR에서 레이어 목록 확인
java -Djarmode=layertools -jar app.jar list
# dependencies
# snapshot-dependencies
# resources
# application

# 레이어별 파일 추출
java -Djarmode=layertools -jar app.jar extract
# 현재 디렉토리에 레이어별 디렉토리 생성:
# ./dependencies/
# ./snapshot-dependencies/
# ./resources/
# ./application/

# 특정 레이어만 추출
java -Djarmode=layertools -jar app.jar extract --layers application
```

```java
// jarmode=layertools 동작 원리:
// JVM 시스템 프로퍼티 "jarmode" = "layertools" 설정 시
// JarLauncher가 앱 대신 LayerToolsJarMode 실행

// LayerToolsJarMode.main()
//   → "list": layers.idx 읽어서 레이어 이름 출력
//   → "extract": layers.idx 기반으로 파일을 레이어별 디렉토리에 복사
//   → "describe": 각 레이어에 어떤 파일이 속하는지 상세 출력
```

### 3. Layered JAR Dockerfile — 두 가지 패턴

```dockerfile
# 패턴 1: Multi-stage build (권장)
FROM eclipse-temurin:21-jre AS builder
WORKDIR /application
COPY app.jar .
# layertools로 레이어 추출
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre
WORKDIR /application

# 레이어 순서: 변경 빈도 낮은 것 먼저 (Docker 캐시 최대 활용)
COPY --from=builder /application/dependencies/ ./
COPY --from=builder /application/snapshot-dependencies/ ./
COPY --from=builder /application/resources/ ./
COPY --from=builder /application/application/ ./

# JarLauncher로 Exploded 모드 실행
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

```dockerfile
# 패턴 2: Exploded 모드 직접 (builder 없이)
FROM eclipse-temurin:21-jre
WORKDIR /application
COPY app.jar .
RUN java -Djarmode=layertools -jar app.jar extract && rm app.jar

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

### 4. JVM 옵션 최적화 (컨테이너 환경)

```dockerfile
FROM eclipse-temurin:21-jre AS builder
WORKDIR /application
COPY app.jar .
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre
WORKDIR /application
COPY --from=builder /application/dependencies/ ./
COPY --from=builder /application/snapshot-dependencies/ ./
COPY --from=builder /application/resources/ ./
COPY --from=builder /application/application/ ./

# 컨테이너 최적화 JVM 옵션
ENV JAVA_OPTS="\
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+ExitOnOutOfMemoryError \
  -Djava.security.egd=file:/dev/./urandom"
# UseContainerSupport: cgroup 메모리 제한 인식 (Java 11+ 기본)
# MaxRAMPercentage:    컨테이너 메모리의 75%를 힙으로
# ExitOnOutOfMemoryError: OOM 시 즉시 종료 → K8s가 재시작
# security.egd:        난수 생성 블로킹 감소 → 시작 시간 단축

ENTRYPOINT ["sh", "-c", \
  "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

### 5. Layered JAR 활성화 설정

```kotlin
// build.gradle.kts
tasks.named<BootJar>("bootJar") {
    // Spring Boot 2.4+ 기본 활성화, 명시적 설정
    layered {
        enabled.set(true)

        // 커스텀 레이어 정의 (필요한 경우)
        application {
            // JarLauncher는 별도 레이어로 분리 (앱 코드와 다른 변경 주기)
            intoLayer("spring-boot-loader") {
                include("org/springframework/boot/loader/**")
            }
            intoLayer("application")
        }
        dependencies {
            // SNAPSHOT 의존성 별도 레이어
            intoLayer("snapshot-dependencies") {
                include("*:*:*SNAPSHOT")
            }
            // 내부 공통 라이브러리 별도 레이어
            intoLayer("company-dependencies") {
                include("com.mycompany:*")
            }
            intoLayer("dependencies")
        }
        layerOrder.set(listOf(
            "dependencies",
            "company-dependencies",
            "snapshot-dependencies",
            "spring-boot-loader",
            "resources",
            "application"
        ))
    }
}
```

### 6. 빌드 시간 비교

```
시나리오: 200MB Fat JAR (180MB 의존성 + 20MB 앱 코드)

일반 Fat JAR Dockerfile:
  코드 변경 → 200MB 재빌드 + 200MB 레지스트리 업로드
  → CI/CD 소요: 3~10분 (네트워크 속도에 따라)

Layered JAR Dockerfile:
  코드 변경 → application 레이어(~1MB)만 재빌드 + 업로드
  → CI/CD 소요: 10~30초
  → dependencies 레이어: 레지스트리 캐시 재사용

절약 효과:
  초기 빌드: 비슷 (캐시 없음)
  이후 코드 변경: 10~20배 빠름
  CI 파이프라인 비용 절감 (컴퓨팅 시간, 네트워크 비용)
```

---

## 💻 실험으로 확인하기

### 실험 1: 레이어별 크기 확인

```bash
./gradlew bootJar
java -Djarmode=layertools -jar build/libs/app.jar extract -d /tmp/layers

# 레이어별 크기 비교
du -sh /tmp/layers/*/
# dependencies/        → 150M+ (대부분의 크기)
# snapshot-dependencies → 수MB
# resources/           → 수MB
# application/         → 수백KB~수MB (앱 코드만)
```

### 실험 2: Docker 빌드 캐시 효과

```bash
# 첫 번째 빌드 (캐시 없음)
time docker build -t app:v1 .
# → 약 60초 (모든 레이어 빌드)

# 코드 변경 후 두 번째 빌드 (dependencies 캐시 활용)
time docker build -t app:v2 .
# → 약 5초 (application 레이어만 재빌드)

# Docker 이미지 레이어 확인
docker image inspect app:v2 | jq '.[0].RootFS.Layers | length'
# → 기본 이미지 레이어 + 4개 (Layered JAR 레이어)
```

### 실험 3: layers.idx 내용 확인

```bash
# 레이어 구성 상세 확인
java -Djarmode=layertools -jar build/libs/app.jar describe
# Layer: 'dependencies'
#   BOOT-INF/lib/spring-core-6.2.0.jar
#   BOOT-INF/lib/spring-web-6.2.0.jar
#   ...
# Layer: 'application'
#   BOOT-INF/classes/com/example/
#   ...
```

---

## ⚙️ 설정 최적화 팁

```yaml
# Kubernetes 배포 최적화
# imagePullPolicy: IfNotPresent → 레이어 캐시 활용
# → 의존성 레이어가 이미 노드에 있으면 pull 불필요

# Dockerfile 추가 최적화:
# 1. 비루트 사용자로 실행 (보안)
RUN addgroup --system spring && adduser --system spring --ingroup spring
USER spring:spring

# 2. 읽기 전용 파일시스템
# tmpfs를 /tmp에 마운트 (K8s volumes)

# 3. 멀티 아키텍처 이미지 (arm64 + amd64)
# docker buildx build --platform linux/amd64,linux/arm64

# 4. Distroless 기본 이미지 (공격 표면 최소화)
# FROM gcr.io/distroless/java21-debian12
# → 셸, 패키지 관리자 없음 → 보안 강화
```

---

## 📌 핵심 정리

```
Layered JAR 핵심 아이디어
  의존성(자주 안 변함) ≠ 앱 코드(자주 변함)
  → Docker 레이어로 분리 → 캐시 최대 활용

layers.idx
  레이어 → 파일 패턴 매핑
  jarmode=layertools로 레이어별 추출

기본 4 레이어 순서
  dependencies → snapshot-dependencies → resources → application
  변경 빈도 낮은 것이 앞 (Docker 캐시 효율)

Dockerfile 패턴
  Multi-stage: builder(추출) → final(실행)
  COPY 순서: 변경 빈도 낮은 레이어부터
  ENTRYPOINT: JarLauncher (Exploded 모드)

효과
  코드 변경 시 ~1MB 레이어만 재빌드/업로드
  CI/CD 10~20배 빠름
```

---

## 🤔 생각해볼 문제

**Q1.** `snapshot-dependencies` 레이어를 `dependencies` 레이어보다 먼저 Docker COPY하면 어떤 문제가 생기는가?

**Q2.** 팀 내부에서 개발 중인 공통 라이브러리(`com.mycompany:common:1.0-SNAPSHOT`)는 어느 레이어에 배치하는 것이 최적인가?

**Q3.** Layered JAR 사용 시 `application` 레이어가 50MB 이상으로 크다면 무엇을 확인해야 하는가?

> 💡 **해설**
>
> **Q1.** Docker는 `COPY` 명령이 나타나는 순서대로 레이어를 생성한다. 상위 레이어(먼저 COPY된 것)가 변경되면 그 이후 모든 레이어의 캐시가 무효화된다. `snapshot-dependencies`를 먼저 COPY하면 SNAPSHOT이 변경될 때마다 그 이후의 `resources`, `application` 레이어 캐시도 무효화된다. 반대로 `dependencies`를 먼저 COPY하면 release 의존성이 변경되지 않는 한 `dependencies` 레이어 캐시가 유지된다. 변경 빈도가 낮은 레이어를 먼저 COPY하는 것이 핵심이다.
>
> **Q2.** 내부 공통 라이브러리가 SNAPSHOT 버전이라면 `snapshot-dependencies` 레이어가 적합하다. 그러나 `snapshot-dependencies`에는 외부 SNAPSHOT과 함께 배치되어 외부 라이브러리 SNAPSHOT 변경 시에도 불필요하게 캐시가 무효화된다. Gradle 커스텀 레이어 설정에서 `intoLayer("company-dependencies") { include("com.mycompany:*") }`로 별도 레이어를 만드는 것이 더 좋다. 내부 라이브러리 릴리즈 주기와 외부 SNAPSHOT 갱신 주기가 다르므로 분리하면 캐시 효율이 높아진다.
>
> **Q3.** `application` 레이어가 크다면 정적 리소스(이미지, 동영상, 대용량 JSON 등)가 `src/main/resources/static/`에 포함됐을 가능성이 높다. 이런 파일들은 코드와 달리 자주 변경되지 않지만 `application` 레이어에 포함되면 코드 변경 시 함께 재빌드된다. 대용량 정적 리소스는 CDN(CloudFront, CloudFlare)으로 분리하거나, Layered JAR 커스텀 설정으로 별도 레이어에 배치하는 것이 좋다. 또한 개발 전용 파일(테스트 픽스처, 샘플 데이터)이 잘못 포함되지 않았는지 확인해야 한다.

---

<div align="center">

**[⬅️ 이전: JarLauncher vs WarLauncher](./02-jar-vs-war-launcher.md)** | **[홈으로 🏠](../README.md)** | **[다음: Native Image with GraalVM ➡️](./04-native-image-graalvm.md)**

</div>
