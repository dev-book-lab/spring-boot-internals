# SSL/TLS 설정 — Keystore에서 내장 서버 적용까지, mTLS 구성

---

## 🎯 핵심 질문

- `server.ssl.*` 프로퍼티가 내장 Tomcat Connector에 적용되는 경로는?
- Keystore와 Truststore의 역할 차이는?
- mTLS(상호 TLS 인증)는 어떻게 설정하는가?
- Let's Encrypt 인증서를 내장 서버에 적용하는 방법은?
- TLS 버전과 Cipher Suite를 제한해야 하는 이유는?
- HTTPS와 HTTP를 동시에 제공하려면 어떻게 하는가?

---

## 🔍 왜 이게 존재하는가

```
전통적 배포:
  Nginx/Apache → SSL 처리 → Tomcat(HTTP)으로 프록시
  → SSL은 외부 서버가 담당, 앱 서버는 HTTP만

내장 서버 직접 SSL:
  컨테이너 환경: 사이드카 없이 단순 배포
  mTLS: 서비스 간 인증 (서비스 메시 없이)
  개발 환경: 로컬 HTTPS 테스트
  Kubernetes: Ingress 없이 Pod가 직접 TLS 처리

server.ssl.* 프로퍼티 → SslServerCustomizer → Connector SSL 설정
```

---

## 🔬 내부 동작 원리

### 1. server.ssl.* 프로퍼티 → Connector 적용 경로

```java
// ServerProperties.Ssl — server.ssl.* 바인딩
@ConfigurationProperties(prefix = "server.ssl")
public class Ssl {
    private boolean enabled = true;
    private String keyStore;
    private String keyStorePassword;
    private String keyStoreType = "JKS";  // 또는 PKCS12
    private String keyAlias;
    private String keyPassword;
    private String trustStore;
    private String trustStorePassword;
    private String trustStoreType = "JKS";
    private ClientAuth clientAuth;  // NONE, WANT, NEED (mTLS)
    private String[] ciphers;       // 허용 Cipher Suite
    private String[] enabledProtocols;  // TLSv1.2, TLSv1.3
    private String protocol = "TLS";
}

// 적용 경로:
// TomcatServletWebServerFactory.customizeConnector()
//   → SslConnectorCustomizer.customize(connector, ssl)
//     → connector.setScheme("https")
//     → connector.setSecure(true)
//     → SSLHostConfig 생성 → Keystore/Truststore 설정
//     → connector.addSslHostConfig(sslHostConfig)
```

```java
// SslConnectorCustomizer (Spring Boot 내부)
class SslConnectorCustomizer {

    void customize(Connector connector, Ssl ssl) {
        connector.setPort(getPort());
        connector.setScheme("https");
        connector.setSecure(true);

        Http11NioProtocol protocol =
            (Http11NioProtocol) connector.getProtocolHandler();
        protocol.setSSLEnabled(true);

        SSLHostConfig sslHostConfig = new SSLHostConfig();

        // Cipher Suite 설정
        if (ssl.getCiphers() != null) {
            sslHostConfig.setCiphers(String.join(",", ssl.getCiphers()));
        }

        // TLS 버전 설정
        if (ssl.getEnabledProtocols() != null) {
            sslHostConfig.setProtocols(
                String.join("+", ssl.getEnabledProtocols()));
        }

        // mTLS 설정
        if (ssl.getClientAuth() != null) {
            switch (ssl.getClientAuth()) {
                case NEED -> sslHostConfig.setCertificateVerification("required");
                case WANT -> sslHostConfig.setCertificateVerification("optional");
                case NONE -> sslHostConfig.setCertificateVerification("none");
            }
        }

        SSLHostConfigCertificate certificate =
            new SSLHostConfigCertificate(sslHostConfig, Type.RSA);

        // Keystore 설정
        certificate.setCertificateKeystoreFile(ssl.getKeyStore());
        certificate.setCertificateKeystorePassword(ssl.getKeyStorePassword());
        certificate.setCertificateKeystoreType(ssl.getKeyStoreType());
        if (ssl.getKeyAlias() != null) {
            certificate.setCertificateKeyAlias(ssl.getKeyAlias());
        }

        sslHostConfig.addCertificate(certificate);
        connector.addSslHostConfig(sslHostConfig);
    }
}
```

### 2. 자체 서명 인증서 생성 (개발용)

```bash
# PKCS12 Keystore 생성 (keytool)
keytool -genkeypair \
  -alias myapp \
  -keyalg RSA \
  -keysize 2048 \
  -storetype PKCS12 \
  -keystore src/main/resources/keystore.p12 \
  -validity 3650 \
  -storepass changeit \
  -dname "CN=localhost, OU=Dev, O=MyOrg, L=Seoul, S=Seoul, C=KR" \
  -ext "SAN=DNS:localhost,IP:127.0.0.1"

# 인증서 확인
keytool -list -v -keystore keystore.p12 -storetype PKCS12 -storepass changeit
```

```yaml
# application.yml — HTTPS 설정
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12      # 또는 file:/etc/ssl/keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
    key-alias: myapp
    # TLS 버전 제한 (TLS 1.0, 1.1 비활성화)
    enabled-protocols:
      - TLSv1.2
      - TLSv1.3
    # 강력한 Cipher Suite만 허용
    ciphers:
      - TLS_AES_128_GCM_SHA256        # TLS 1.3
      - TLS_AES_256_GCM_SHA384        # TLS 1.3
      - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384   # TLS 1.2
      - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256   # TLS 1.2
```

### 3. mTLS (상호 TLS 인증) 설정

```
단방향 TLS (일반 HTTPS):
  클라이언트 → 서버 인증서 확인
  서버 → 클라이언트 인증 없음

mTLS (Mutual TLS):
  클라이언트 → 서버 인증서 확인
  서버 → 클라이언트 인증서도 확인 (client-auth: need)
  → 서비스 간 인증, API Gateway 없이 서비스 신원 보장
```

```bash
# 1. CA(Certificate Authority) 생성
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 1826 -key ca.key -out ca.crt \
  -subj "/CN=MyCA/O=MyOrg/C=KR"

# 2. 서버 인증서 생성 (CA 서명)
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr \
  -subj "/CN=my-service/O=MyOrg/C=KR"
openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt

# 3. 클라이언트 인증서 생성 (CA 서명)
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr \
  -subj "/CN=my-client/O=MyOrg/C=KR"
openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out client.crt

# 4. Keystore/Truststore 생성
# 서버 Keystore (서버 인증서 + 개인키)
openssl pkcs12 -export -in server.crt -inkey server.key \
  -out server.p12 -name server -passout pass:serverpass

# Truststore (클라이언트 인증서를 신뢰하기 위한 CA 인증서)
keytool -importcert -file ca.crt -alias myca \
  -keystore truststore.p12 -storetype PKCS12 \
  -storepass trustpass -noprompt
```

```yaml
# mTLS 서버 설정
server:
  ssl:
    enabled: true
    key-store: classpath:server.p12
    key-store-password: serverpass
    key-store-type: PKCS12
    # Truststore — 클라이언트 인증서의 CA를 신뢰
    trust-store: classpath:truststore.p12
    trust-store-password: trustpass
    trust-store-type: PKCS12
    # NEED: 클라이언트 인증서 필수 (없으면 연결 거부)
    # WANT: 클라이언트 인증서 있으면 검증, 없어도 연결 허용
    client-auth: need
```

```java
// mTLS 클라이언트 (RestTemplate)
@Bean
public RestTemplate mtlsRestTemplate() throws Exception {
    // 클라이언트 Keystore (클라이언트 인증서 + 개인키)
    KeyStore clientKeyStore = KeyStore.getInstance("PKCS12");
    clientKeyStore.load(
        getClass().getResourceAsStream("/client.p12"),
        "clientpass".toCharArray());

    // Truststore (서버 인증서의 CA 신뢰)
    KeyStore trustStore = KeyStore.getInstance("PKCS12");
    trustStore.load(
        getClass().getResourceAsStream("/truststore.p12"),
        "trustpass".toCharArray());

    SSLContext sslContext = SSLContextBuilder.create()
        .loadKeyMaterial(clientKeyStore, "clientpass".toCharArray())
        .loadTrustMaterial(trustStore, null)
        .build();

    HttpClient httpClient = HttpClients.custom()
        .setSSLContext(sslContext)
        .build();

    return new RestTemplate(
        new HttpComponentsClientHttpRequestFactory(httpClient));
}
```

### 4. HTTPS + HTTP 동시 제공

```java
// 기본 server.port는 HTTPS → HTTP 추가 Connector로
@Component
public class HttpConnectorCustomizer
        implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Value("${server.http.port:8080}")
    private int httpPort;

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        // HTTP Connector 추가
        Connector httpConnector =
            new Connector(TomcatServletWebServerFactory.DEFAULT_PROTOCOL);
        httpConnector.setScheme("http");
        httpConnector.setPort(httpPort);
        httpConnector.setSecure(false);
        factory.addAdditionalTomcatConnectors(httpConnector);
    }
}
```

```yaml
server:
  port: 8443        # HTTPS (기본)
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: changeit
  http:
    port: 8080      # 커스텀 프로퍼티
```

### 5. Let's Encrypt 인증서 적용

```bash
# certbot으로 인증서 발급 (도메인 필요)
certbot certonly --standalone -d myapp.example.com

# PEM → PKCS12 변환
openssl pkcs12 -export \
  -in /etc/letsencrypt/live/myapp.example.com/fullchain.pem \
  -inkey /etc/letsencrypt/live/myapp.example.com/privkey.pem \
  -out /etc/ssl/myapp.p12 \
  -name myapp \
  -passout pass:mypassword
```

```yaml
server:
  ssl:
    key-store: file:/etc/ssl/myapp.p12
    key-store-password: mypassword
    key-store-type: PKCS12
    key-alias: myapp
```

```java
// 인증서 자동 갱신 (Let's Encrypt 90일 만료)
// 갱신 후 Spring Boot 재시작 없이 인증서 교체:
@Component
public class SslCertificateRefresher {

    @Scheduled(cron = "0 0 3 * * *")  // 매일 새벽 3시
    public void checkAndReloadCertificate() {
        // TomcatWebServer의 SSLHostConfig 동적 재로딩
        // (Tomcat 8.5+에서 지원)
        // 또는: k8s cert-manager + 주기적 pod 재시작
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: HTTPS 연결 테스트

```bash
# 자체 서명 인증서 (CA 검증 건너뜀)
curl -k https://localhost:8443/api/health

# 특정 CA로 검증
curl --cacert ca.crt https://localhost:8443/api/health

# TLS 버전 확인
openssl s_client -connect localhost:8443 2>&1 | grep "Protocol"
# → Protocol: TLSv1.3

# Cipher Suite 확인
openssl s_client -connect localhost:8443 2>&1 | grep "Cipher"
# → Cipher: TLS_AES_256_GCM_SHA384
```

### 실험 2: mTLS 클라이언트 인증 테스트

```bash
# 클라이언트 인증서 없이 (client-auth: need → 연결 거부)
curl -k https://localhost:8443/api/resource
# → curl: (56) OpenSSL SSL_read: ... alert certificate required

# 클라이언트 인증서와 함께
curl --cert client.crt --key client.key \
     --cacert ca.crt \
     https://localhost:8443/api/resource
# → 200 OK
```

### 실험 3: X.509 클라이언트 인증서 정보 추출

```java
// mTLS에서 클라이언트 인증서 정보 활용
@RestController
public class SecureController {

    @GetMapping("/api/resource")
    public String getResource(HttpServletRequest request) {
        // 클라이언트 인증서 추출
        X509Certificate[] certs = (X509Certificate[])
            request.getAttribute("jakarta.servlet.request.X509Certificate");

        if (certs != null && certs.length > 0) {
            String subject = certs[0].getSubjectX500Principal().getName();
            // "CN=my-client, O=MyOrg, C=KR"
            return "Authenticated client: " + subject;
        }
        return "No client certificate";
    }
}
```

---

## ⚙️ 설정 최적화 팁

```yaml
# 운영 환경 TLS 보안 강화 설정
server:
  ssl:
    enabled-protocols:
      - TLSv1.3     # 가능하면 TLS 1.3만 (최고 보안)
      - TLSv1.2     # 레거시 클라이언트 호환
    ciphers:
      # TLS 1.3 (자동 협상, 별도 지정 불필요하지만 명시)
      - TLS_AES_256_GCM_SHA384
      - TLS_AES_128_GCM_SHA256
      - TLS_CHACHA20_POLY1305_SHA256
      # TLS 1.2 (Forward Secrecy 지원 ECDHE만)
      - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
      - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
      - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
    # 취약한 것들 제외:
    # - TLS_RSA_* (Forward Secrecy 없음)
    # - TLS_DHE_RSA_WITH_AES_*_CBC_* (LUCKY13 취약점)
    # - TLS_*_RC4_* (RC4 취약점)
    # - *_MD5, *_SHA (해시 취약)
```

---

## 📌 핵심 정리

```
server.ssl.* 적용 경로
  SslConnectorCustomizer → SSLHostConfig → SSLHostConfigCertificate
  → Tomcat Connector에 SSL 설정

Keystore vs Truststore
  Keystore: 서버/클라이언트 본인 인증서 + 개인키 (비밀)
  Truststore: 신뢰할 CA 인증서 목록 (공개)

mTLS 설정
  server.ssl.client-auth: need (클라이언트 인증서 필수)
  server.ssl.trust-store: 클라이언트 CA 인증서
  → 요청에서 X509Certificate 속성으로 인증서 정보 추출

TLS 보안 강화
  enabled-protocols: TLSv1.2, TLSv1.3 (1.0, 1.1 제외)
  ciphers: ECDHE 기반만 허용 (Forward Secrecy)
  TLS 1.3 권장 (더 강력, 더 빠름)
```

---

## 🤔 생각해볼 문제

**Q1.** `server.ssl.client-auth=want`와 `need`의 차이는 무엇이며, 각각 언제 사용하는가?

**Q2.** PKCS12와 JKS Keystore 형식 중 어느 것을 사용하는 것이 권장되는가? 그 이유는?

**Q3.** Kubernetes 환경에서 TLS 인증서를 Spring Boot 내장 서버에 적용할 때 Secret을 안전하게 마운트하는 방법은?

> 💡 **해설**
>
> **Q1.** `want`는 클라이언트 인증서를 요청하지만, 클라이언트가 제공하지 않아도 연결을 허용한다. 인증서가 있으면 검증하고, 없으면 익명으로 처리한다. 점진적 mTLS 도입 단계나 일부 클라이언트가 인증서를 지원하지 않는 혼합 환경에 적합하다. `need`는 클라이언트 인증서가 없으면 TLS 핸드셰이크 단계에서 즉시 연결을 거부한다. 모든 클라이언트가 인증서를 가진 환경(서비스 메시, 내부 서비스 간 통신)에서 사용한다.
>
> **Q2.** PKCS12(`.p12`, `.pfx`)를 권장한다. JKS는 Java 전용 독점 형식이지만 PKCS12는 업계 표준이며 OpenSSL 등 다양한 도구와 호환된다. Java 9부터 PKCS12가 기본 Keystore 형식으로 지정되었고, JDK의 `keytool`도 기본적으로 PKCS12를 생성한다. Spring Boot도 PKCS12를 기본값으로 설정하는 방향으로 발전하고 있다. JKS는 AES-128 기반 암호화를 사용하는 반면 PKCS12는 더 강력한 암호화를 지원한다.
>
> **Q3.** Kubernetes Secret을 Volume으로 마운트해 파일로 읽는 것이 가장 안전하다. `server.ssl.key-store: file:/etc/ssl/keystore.p12`처럼 파일 경로를 지정한다. Secret을 환경변수로 전달하는 방법(`JAVA_OPTS=-Dserver.ssl.key-store-password=...`)도 있지만, 환경변수는 `ps aux`나 `/proc/{pid}/environ`으로 노출될 수 있어 권장하지 않는다. 가장 안전한 방법은 HashiCorp Vault나 AWS Secrets Manager와 통합해 런타임에 동적으로 시크릿을 가져오는 것이다.

---

<div align="center">

**[⬅️ 이전: Embedded Server 커스터마이징](./03-embedded-server-customization.md)** | **[홈으로 🏠](../README.md)** | **[다음: HTTP/2 활성화 ➡️](./05-http2-activation.md)**

</div>
