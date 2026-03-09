# Remote Debug & Remote Update — HTTP 터널링으로 원격 서버에 클래스 변경 푸시

---

## 🎯 핵심 질문

- `spring.devtools.remote.secret` 설정으로 원격 서버에 클래스 변경을 푸시하는 메커니즘은?
- Remote DevTools가 HTTP를 터널로 활용하는 방식은?
- 로컬에서 원격 서버로 변경된 클래스를 전송하는 구체적 경로는?
- Remote Debug 세션은 어떻게 연결되는가?
- 왜 Remote DevTools는 운영 환경에서 절대 사용하면 안 되는가?

---

## 🔍 왜 이게 존재하는가

```
사용 시나리오:
  로컬 개발 환경에서 재현 불가한 버그
  → 스테이징 서버에 배포된 앱에서만 발생
  → 원격 서버를 재시작하지 않고 클래스 핫 업데이트
  → 또는 원격 서버에 디버거 연결

Remote DevTools 구성:
  로컬 PC (IDE + Remote Client)
       ↕ HTTP (터널링)
  원격 서버 (Remote Server Handler)
```

---

## 🔬 내부 동작 원리

### 1. 원격 서버 설정

```yaml
# 원격 서버 application.yml
spring:
  devtools:
    remote:
      secret: my-secret-key    # 공유 비밀키 (인증용)
      # secret이 설정되면 원격 처리 핸들러 활성화
```

```java
// RemoteDevToolsAutoConfiguration
@AutoConfiguration
@ConditionalOnProperty("spring.devtools.remote.secret")
// secret이 설정된 경우에만 활성화
public class RemoteDevToolsAutoConfiguration {

    @Bean
    public DispatcherFilter remoteDevToolsDispatcherFilter(
            HttpTunnelServer httpTunnelServer,
            RemoteDevToolsProperties properties) {
        // DevTools 전용 필터 등록
        // /.~~spring-boot!~/restart 경로로 오는 요청 처리
        return new DispatcherFilter(
            new UrlHandlerMapper("/.~~spring-boot!~/restart",
                new HttpRestartServerHandler(httpRestartServer)));
    }
}
```

### 2. HTTP 터널링 — RemoteClient 동작

```java
// 로컬 PC에서 실행되는 RemoteClient (IDE에서 별도 실행)
// 실행 방법:
// main: org.springframework.boot.devtools.RemoteSpringApplication
// 인수: https://staging.example.com (원격 서버 URL)

public class RemoteSpringApplication {
    public static void main(String[] args) {
        // args[0] = 원격 서버 URL
        // → RemoteClientConfiguration 시작
    }
}

// RemoteClientConfiguration
@Configuration
public class RemoteClientConfiguration {

    // ① 원격 서버로 HTTP 요청을 터널링하는 클라이언트 연결
    @Bean
    public ClassPathChangeUploader classPathChangeUploader(
            ClientHttpRequestFactory requestFactory,
            RemoteDevToolsProperties properties) {
        // 원격 서버의 /.~~spring-boot!~/restart 엔드포인트로
        // 변경된 클래스 파일 전송
        return new ClassPathChangeUploader(
            properties.getRemoteUrl() + "/.~~spring-boot!~/restart",
            requestFactory);
    }

    // ② 로컬 파일 변경 감지
    @Bean
    public ClassPathFileSystemWatcher classPathFileSystemWatcher() {
        // 로컬 빌드 출력 디렉토리 감시
        // 변경 감지 → ClassPathChangeUploader 호출
        return new ClassPathFileSystemWatcher(...);
    }
}
```

### 3. 클래스 업로드 전체 흐름

```
1. 로컬 IDE에서 코드 수정 + 컴파일
   → build/classes/java/main/UserController.class 갱신

2. 로컬 ClassPathFileSystemWatcher 감지
   → ClassPathChangeUploader.upload(changedFiles)

3. HTTP POST 요청 전송
   POST https://staging.example.com/.~~spring-boot!~/restart
   Headers:
     X-AUTH-TOKEN: my-secret-key   (인증)
     Content-Type: application/octet-stream
   Body: 변경된 .class 파일 바이너리 (직렬화된 ClassLoaderFiles)

4. 원격 서버 수신 (HttpRestartServerHandler)
   → X-AUTH-TOKEN 검증 (비밀키 불일치 시 403 거절)
   → ClassLoaderFiles 역직렬화
   → 변경된 클래스 파일 임시 저장

5. 원격 서버 재시작 (RestartServer)
   → 새 RestartClassLoader 생성 (업로드된 파일 포함)
   → ApplicationContext 재시작
   → 원격 서버 응답 200 OK

6. 로컬 브라우저 자동 리프레시 (선택적)
```

### 4. Remote Debug — HTTP 터널로 JDWP 프록시

```java
// JDWP (Java Debug Wire Protocol) HTTP 터널링
// 원격 서버에 JDWP 직접 포트 노출 없이 HTTP로 우회

// 원격 서버에서 JVM 디버그 포트 활성화 필요:
// java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 \
//      -jar app.jar

// 로컬 RemoteClient가 HTTP 터널 생성:
//   로컬 포트 (예: 8000) → HTTP 터널 → 원격 서버 → 원격 5005 포트

// IDE 디버거 연결:
//   localhost:8000 (터널 로컬 끝)
//   → RemoteClient → HTTP → 원격 서버 → JDWP 5005

@Bean
public RemoteDebugPortProvider remoteDebugPortProvider(
        ClientHttpRequestFactory requestFactory) {
    // 로컬 임시 서버 소켓 → HTTP 터널 브리지
    return new RemoteDebugPortProvider(
        requestFactory,
        remoteUrl + "/.~~spring-boot!~/debug");
}
```

```yaml
# Remote Debug 설정
spring:
  devtools:
    remote:
      secret: my-secret-key
      debug:
        enabled: true
        local-port: 8000  # 로컬 디버거 연결 포트
```

### 5. 보안 경고 — 운영 환경 절대 사용 금지

```
Remote DevTools의 보안 위험:

① 인증: X-AUTH-TOKEN 단일 비밀키
   → 비밀키 유출 시 원격 코드 실행 가능
   → 임의 클래스 파일 업로드 → 서버에서 실행
   → 사실상 원격 코드 실행(RCE) 취약점

② HTTP 평문 전송 (HTTPS가 아니라면)
   → 네트워크 스니핑으로 클래스 파일 탈취

③ 재시작 기능
   → 공격자가 서버 재시작 가능 → DoS

④ Debug 포트 노출
   → 디버거 연결 시 코드 전체 접근 가능

→ Remote DevTools는 스테이징 환경에서 일시적으로만 사용
→ 운영 환경 절대 금지
→ DevTools가 optional 의존성인 이유 중 하나
```

---

## 💻 실험으로 확인하기

### 실험 1: RemoteClient 실행

```bash
# 원격 서버가 HTTPS로 실행 중이고 secret이 설정된 경우
java -cp ".:app.jar:spring-boot-devtools.jar" \
  org.springframework.boot.devtools.RemoteSpringApplication \
  https://staging.example.com

# 로그:
# INFO: Connecting to remote application at https://staging.example.com
# INFO: Remote application has started
# INFO: Watching classpath: build/classes/java/main
```

### 실험 2: 클래스 변경 전송 확인

```bash
# 원격 서버 로그에서 클래스 업데이트 확인
# "Restarting application due to class path change"
# "Classes updated: com.example.UserController"

# 원격 서버 측 수신 엔드포인트 확인
curl -v \
  -H "X-AUTH-TOKEN: my-secret-key" \
  https://staging.example.com/.~~spring-boot!~/restart
# 올바른 SECRET → 200
# 잘못된 SECRET → 403 Forbidden
```

---

## ⚙️ 설정 최적화 팁

```yaml
# Remote DevTools 최소 설정
spring:
  devtools:
    remote:
      secret: ${DEVTOOLS_SECRET}  # 환경변수로 관리 (코드에 하드코딩 금지)
      # HTTPS 사용 강력 권장
      proxy-host: proxy.company.com  # 프록시 환경이면 설정
      proxy-port: 8080
      restart:
        enabled: true  # 원격 재시작 활성화
      debug:
        enabled: false  # 원격 디버그는 필요 시에만 활성화
```

---

## 📌 핵심 정리

```
Remote DevTools 동작
  로컬 RemoteClient → HTTP POST → 원격 서버 /.~~spring-boot!~/restart
  변경된 .class 파일 바이너리 전송 (ClassLoaderFiles 직렬화)
  X-AUTH-TOKEN 헤더로 인증
  원격 서버에서 RestartClassLoader 갱신 → 재시작

HTTP 터널 Debug
  JDWP → HTTP 터널 → 원격 JDWP 포트
  직접 디버그 포트 노출 없이 HTTP를 통해 디버거 연결

보안 위험
  클래스 업로드 = 원격 코드 실행 가능
  스테이징 환경 일시적 사용만 허용
  운영 환경 절대 금지
  비밀키 환경변수 관리, HTTPS 필수
```

---

## 🤔 생각해볼 문제

**Q1.** 원격 서버에 `spring.devtools.remote.secret`이 설정되면 `/.~~spring-boot!~/restart` 경로가 일반 DispatcherServlet 경로 매핑에 노출되는가?

**Q2.** Remote DevTools로 전송된 클래스가 원격 서버에서 어떻게 로딩되는가? 로컬 Restart와 메커니즘이 같은가?

**Q3.** HTTPS를 사용하더라도 Remote DevTools가 위험한 이유는 무엇인가?

> 💡 **해설**
>
> **Q1.** 아니다. `RemoteDevToolsAutoConfiguration`은 `DispatcherFilter`를 서블릿 필터로 등록하는데, 이 필터가 `/.~~spring-boot!~/` 경로 요청을 가로채 Spring MVC `DispatcherServlet`에 도달하기 전에 처리한다. 일반 `@RequestMapping` 핸들러에는 도달하지 않는다. 경로 이름 자체도 일반 URL에서 사용하지 않는 특수문자(`!~`)를 포함해 충돌 가능성을 낮췄다.
>
> **Q2.** 기본적으로 같은 메커니즘이다. 원격 서버의 `RestartServer`가 업로드된 `.class` 파일을 받아 `ClassLoaderFiles`에 저장하고, 새 `RestartClassLoader`를 생성할 때 이 파일들을 포함시킨다. 이후 `ApplicationContext`를 재시작해 새 ClassLoader로 앱 코드를 다시 로딩한다. 로컬 Restart는 파일 시스템에서 직접 읽고, Remote Update는 HTTP로 전송받은 바이너리를 사용하는 차이만 있다.
>
> **Q3.** HTTPS는 전송 중 도청을 막지만, 엔드포인트 자체의 위험성은 해소하지 못한다. 비밀키가 짧거나 유추 가능하면 무차별 대입 공격으로 탈취할 수 있고, 비밀키를 아는 사람이라면 임의의 클래스를 업로드해 서버에서 실행할 수 있다. 이는 RCE(Remote Code Execution) 취약점과 동일하다. 또한 원격 재시작 기능만으로도 서비스 가용성을 저해할 수 있다. 진정한 운영 디버깅이 필요하다면 APM, 분산 추적(OpenTelemetry), 로그 집중화 등 안전한 관찰 가능성 도구를 사용해야 한다.

---

<div align="center">

**[⬅️ 이전: Property Defaults](./03-property-defaults.md)** | **[홈으로 🏠](../README.md)** | **[다음: Build Plugin 통합 ➡️](./05-build-plugin-integration.md)**

</div>
