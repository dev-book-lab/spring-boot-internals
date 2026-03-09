# HTTP/2 활성화 — ALPN 협상 메커니즘과 서버별 H2 지원 차이

---

## 🎯 핵심 질문

- `server.http2.enabled=true`가 내부적으로 적용되는 경로는?
- ALPN(Application-Layer Protocol Negotiation)이란 무엇이고 어떻게 동작하는가?
- Tomcat / Undertow / Jetty별 HTTP/2 지원 구현 차이는?
- HTTP/2가 HTTP/1.1보다 빠른 이유는?
- h2c(평문 HTTP/2)와 h2(TLS HTTP/2)의 차이는?
- Spring Boot에서 HTTP/2를 활성화할 때 TLS가 사실상 필수인 이유는?

---

## 🔍 왜 이게 존재하는가

```
HTTP/1.1 한계:
  Head-of-line blocking: 하나의 TCP 연결에서 요청 순서대로 처리
  → 앞 요청 지연 → 뒤 요청 모두 대기
  → 해결책: 여러 TCP 연결 동시 사용 (브라우저 기본 6개)
    → 서버 연결 수 증가, 핸드셰이크 오버헤드

HTTP/2 개선:
  멀티플렉싱: 하나의 TCP 연결에서 여러 요청/응답 동시 처리
  헤더 압축 (HPACK): 반복 헤더 크기 80-90% 감소
  서버 푸시: 요청 전 리소스 선제 전송
  스트림 우선순위: 중요 리소스 먼저 처리
  바이너리 프레이밍: 텍스트 대신 바이너리 → 파싱 효율 향상
```

---

## 🔬 내부 동작 원리

### 1. server.http2.enabled → 서버 설정 적용 경로

```java
// ServerProperties.Http2 — server.http2.* 바인딩
public static class Http2 {
    private boolean enabled = false;
}

// 적용 경로 (Tomcat 기준):
// TomcatServletWebServerFactory.customizeConnector()
//   → Http2TomcatCustomizer.customize(factory)
//     → factory.addConnectorCustomizers(connector -> {
//           UpgradeProtocol upgradeProtocol = new Http2Protocol();
//           connector.addUpgradeProtocol(upgradeProtocol);
//       })
// → Tomcat Connector가 HTTP/2 프로토콜 업그레이드 지원

// Spring Boot 내부 처리:
// TomcatServletWebServerFactory.customizeHttp2()
protected void customizeHttp2(Connector connector) {
    if (getHttp2() != null && getHttp2().isEnabled()) {
        connector.addUpgradeProtocol(new Http2Protocol());
    }
}
```

### 2. ALPN — Application-Layer Protocol Negotiation

```
ALPN은 TLS 핸드셰이크 중 애플리케이션 프로토콜을 협상하는 TLS 확장

TLS 핸드셰이크 + ALPN 흐름:

클라이언트 → 서버: ClientHello
  + ALPN 확장: ["h2", "http/1.1"]  (지원하는 프로토콜 목록)

서버 → 클라이언트: ServerHello
  + ALPN 확장: "h2"  (선택된 프로토콜)
  (서버가 h2를 지원하면 h2 선택, 아니면 http/1.1)

이후: 선택된 프로토콜로 통신 시작
  → h2 선택 시: 별도 업그레이드 없이 바로 HTTP/2 통신

장점:
  프로토콜 협상을 위한 별도 왕복(RTT) 없음
  → TLS 핸드셰이크와 동시에 협상 완료
  → 연결 수립 지연 최소화
```

```java
// Java 11+에서 ALPN 기본 지원 (JDK 내장)
// Java 8: ALPN을 위해 별도 Jetty-ALPN 에이전트 필요 (Boot 2.x 시절)
// → Spring Boot 3.x: Java 17+ 필수 → ALPN 기본 지원

// Tomcat 내부 ALPN 처리:
// NioEndpoint → TLSClientHelloExtractor → ALPN 확장 파싱
// Http11NioProtocol + Http2Protocol 등록
// → ClientHello에 ALPN 있으면 Http2Protocol 선택
// → ALPN 없으면 Http11Protocol 폴백
```

### 3. 서버별 HTTP/2 구현 차이

```java
// ① Tomcat — HTTP/2 활성화
// HTTPS 필수 (h2c는 제한적 지원)
// JDK 11+ 필요 (ALPN 내장)

server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: changeit
  http2:
    enabled: true

// 내부: Http2Protocol을 Connector에 UpgradeProtocol로 추가
// → Tomcat이 ALPN으로 h2 협상 → HTTP/2 스트림 처리
// → HTTP/2 스트림 → 내부적으로 기존 서블릿 처리로 변환
```

```java
// ② Undertow — HTTP/2 활성화 (h2c도 지원)
// XNIO 기반이라 HTTP/2 자연스러운 지원

server:
  http2:
    enabled: true
  # SSL 없이도 h2c(평문) 동작 가능

// 내부: UndertowBuilderCustomizer
// builder.setServerOption(UndertowOptions.ENABLE_HTTP2, true)
// XNIO의 Non-blocking 특성상 HTTP/2 멀티플렉싱에 유리

// h2c (cleartext HTTP/2) Undertow 활성화:
@Component
public class UndertowHttp2Customizer
        implements WebServerFactoryCustomizer<UndertowServletWebServerFactory> {
    @Override
    public void customize(UndertowServletWebServerFactory factory) {
        factory.addBuilderCustomizers(builder ->
            builder.setServerOption(UndertowOptions.ENABLE_HTTP2, true));
    }
}
```

```java
// ③ Jetty — HTTP/2 활성화
// HTTP2C ConnectionFactory 추가 필요

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-jetty'
    implementation 'org.eclipse.jetty.http2:jetty-http2-server'
}

server:
  http2:
    enabled: true

// 내부: JettyServerCustomizer
// ServerConnector에 ALPNServerConnectionFactory + HTTP2ServerConnectionFactory 추가
// → Jetty의 ALPN 처리: jetty-alpn-java-server (JDK 9+ ALPN 내장 활용)
```

### 4. h2 vs h2c

```
h2 (HTTP/2 over TLS):
  표준 브라우저가 지원하는 방식
  TLS 핸드셰이크 중 ALPN으로 협상
  보안 + 성능 동시 확보
  → Spring Boot 권장 방식

h2c (HTTP/2 Cleartext, 평문 HTTP/2):
  TLS 없는 HTTP/2
  브라우저는 h2c 거의 미지원 (보안 이유)
  서비스 내부 통신에서만 실용적
  → Kubernetes 클러스터 내부 서비스 간 통신
  → 로드밸런서(Nginx/Envoy)가 TLS 종료 후 h2c로 백엔드 통신

Upgrade 헤더 방식 (HTTP/1.1 → h2c 업그레이드):
  GET / HTTP/1.1
  Host: example.com
  Connection: Upgrade, HTTP2-Settings
  Upgrade: h2c
  → 추가 왕복 발생 → 실제 잘 사용 안 함
```

### 5. HTTP/2 멀티플렉싱 동작

```
HTTP/1.1 (6개 TCP 연결):
  연결1: GET /style.css     → 응답 대기 → 응답 수신
  연결2: GET /script.js     → 응답 대기 → 응답 수신
  연결3: GET /image.png     → 응답 대기 → 응답 수신
  ...

HTTP/2 (1개 TCP 연결, 여러 스트림):
  스트림1: GET /style.css   ─────────────→ 응답
  스트림2: GET /script.js   ─────────────→ 응답  (동시)
  스트림3: GET /image.png   ─────────────→ 응답  (동시)
  → 하나의 연결에서 모두 처리
  → Head-of-line blocking: TCP 레벨(패킷 손실)로 이동
    (HTTP 레벨 해소 → HTTP/3/QUIC으로 완전 해소)

Spring MVC + HTTP/2:
  각 스트림 → 별도 HttpServletRequest로 변환
  → 기존 서블릿 코드 변경 없이 HTTP/2 혜택
  (HTTP/2 스트림 처리는 서버 레이어에서 추상화)
```

### 6. HTTP/2 + gRPC

```java
// Spring Boot 3.x + HTTP/2 → gRPC 지원 (spring-grpc)
// gRPC는 HTTP/2 기반 (필수)
// → server.http2.enabled=true 설정 필요

// gRPC 서버 Bean
@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {
    @Override
    public void getUser(GetUserRequest request,
                        StreamObserver<GetUserResponse> responseObserver) {
        // HTTP/2 스트리밍 자동 처리
        responseObserver.onNext(GetUserResponse.newBuilder()
            .setId(request.getId())
            .setName("User " + request.getId())
            .build());
        responseObserver.onCompleted();
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: HTTP/2 활성화 확인

```bash
# curl로 HTTP 버전 확인
curl -k --http2 -v https://localhost:8443/api/hello 2>&1 | grep "HTTP/"
# → HTTP/2 200

# h2 프로토콜 협상 과정 확인
curl -k --http2 -v https://localhost:8443/ 2>&1 | grep -E "ALPN|h2|HTTP"
# → ALPN: offers h2
# → ALPN: server accepted h2
# → HTTP/2 200
```

### 실험 2: 멀티플렉싱 효과 측정

```bash
# HTTP/1.1로 50개 요청
time for i in {1..50}; do
  curl -k -s -o /dev/null https://localhost:8443/api/data &
done
wait

# HTTP/2로 50개 요청 (멀티플렉싱)
time curl -k --http2 -s -o /dev/null \
  "https://localhost:8443/api/data" \
  "https://localhost:8443/api/users" \
  "https://localhost:8443/api/orders"
# → HTTP/2가 더 빠름 (특히 많은 작은 요청에서)
```

### 실험 3: 서버 푸시 (Server Push)

```java
// HTTP/2 서버 푸시 (브라우저 리소스 선제 전송)
@GetMapping("/")
public String index(HttpServletRequest request) {
    PushBuilder pushBuilder = request.newPushBuilder();
    if (pushBuilder != null) {
        // 클라이언트가 /index.html 요청하기 전에 미리 push
        pushBuilder.path("/style.css").push();
        pushBuilder.path("/script.js").push();
    }
    return "index";
}
// HTTP/2 지원 클라이언트 → pushBuilder != null
// HTTP/1.1 클라이언트 → pushBuilder == null (폴백)
```

---

## ⚙️ 설정 최적화 팁

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
    enabled-protocols:
      - TLSv1.3  # HTTP/2와 TLS 1.3 조합이 최고 성능
      - TLSv1.2
  http2:
    enabled: true

# Tomcat HTTP/2 튜닝
server:
  tomcat:
    # HTTP/2 초기 윈도우 크기 (스트림 흐름 제어)
    # 기본값으로 충분한 경우가 많음
    max-connections: 8192  # HTTP/2는 하나의 연결에 여러 스트림
    # HTTP/2에서는 연결 수 줄고 스트림 수 증가
    # max-connections는 낮게, 스트림 처리량 높게 설정
```

---

## 📌 핵심 정리

```
HTTP/2 활성화 경로
  server.http2.enabled=true
  → TomcatServletWebServerFactory.customizeHttp2()
  → Connector에 Http2Protocol 추가
  → ALPN으로 h2 협상

ALPN 동작
  TLS 핸드셰이크에 포함 → 추가 RTT 없음
  클라이언트: ["h2", "http/1.1"] 제안
  서버: "h2" 선택 → HTTP/2 통신 시작

서버별 지원
  Tomcat   h2 (TLS 필수), JDK 11+ ALPN 내장
  Undertow h2 + h2c 모두 지원 (XNIO 기반)
  Jetty    h2, jetty-http2-server 의존성 추가 필요

HTTP/2 핵심 이점
  멀티플렉싱: 하나의 TCP 연결에 여러 스트림
  헤더 압축: HPACK (반복 헤더 80-90% 절감)
  → 기존 서블릿 코드 변경 없이 성능 향상
```

---

## 🤔 생각해볼 문제

**Q1.** HTTP/2가 활성화된 Tomcat에서 스레드 모델은 어떻게 달라지는가? 스레드 수를 줄여야 하는가?

**Q2.** HTTP/2는 TCP Head-of-line blocking을 해소한다고 알려져 있다. 이것이 완전한 해소인가?

**Q3.** 내부 서비스 간 통신(gRPC 또는 REST)에서 h2c를 사용하는 것이 h2보다 유리한 경우는 언제인가?

> 💡 **해설**
>
> **Q1.** HTTP/2가 활성화되어도 Spring MVC(서블릿 기반)의 스레드 모델은 동일하다. HTTP/2 스트림은 서버 레이어에서 처리되고, 각 스트림은 독립적인 서블릿 요청(`HttpServletRequest`)으로 변환되어 기존 스레드 풀에서 처리된다. HTTP/1.1에서는 클라이언트 하나가 6개 TCP 연결을 열어 6개 스레드를 사용했다면, HTTP/2에서는 하나의 TCP 연결에서 여러 스트림이 병렬로 처리되므로 동시 스트림 수만큼 스레드가 사용된다. 결과적으로 클라이언트당 필요 스레드는 유사하거나 소폭 증가할 수 있다. 스레드 수를 크게 줄이려면 WebFlux(Reactive)로 전환해야 한다.
>
> **Q2.** HTTP/2는 HTTP 레벨의 Head-of-line blocking(앞 요청이 지연되면 뒤 요청도 대기)을 멀티플렉싱으로 해소한다. 그러나 TCP 레벨 Head-of-line blocking은 여전히 존재한다. TCP 패킷 하나가 손실되면 해당 패킷이 재전송되기까지 이후 모든 스트림의 데이터가 대기한다(모든 스트림이 하나의 TCP 연결을 공유하므로). 이를 완전히 해소하는 것이 HTTP/3이다. HTTP/3은 QUIC(UDP 기반) 위에서 동작하며, 스트림별 독립적 패킷 손실 처리로 TCP Head-of-line blocking을 없앴다.
>
> **Q3.** h2c가 유리한 경우는 클러스터 내부 통신에서 TLS 오버헤드를 제거하고 싶을 때다. Kubernetes에서 Ingress 또는 서비스 메시(Istio, Linkerd)가 외부 TLS를 처리하고 클러스터 내부는 이미 신뢰된 네트워크로 간주하는 경우, h2c로 백엔드 간 통신하면 TLS 핸드셰이크 비용과 암호화/복호화 CPU 오버헤드를 줄일 수 있다. 단, 클러스터 내부 네트워크도 신뢰하지 않는 제로 트러스트 보안 정책을 적용한다면 h2(TLS)를 사용해야 한다.

---

<div align="center">

**[⬅️ 이전: SSL/TLS 설정](./04-ssl-tls-configuration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Context Path & Port 설정 ➡️](./06-context-path-port.md)**

</div>
