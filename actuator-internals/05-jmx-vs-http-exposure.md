# JMX vs HTTP Exposure — 두 노출 방식의 내부 처리 차이

---

## 🎯 핵심 질문

- JMX와 HTTP Exposure의 처리 경로가 어떻게 다른가?
- `management.endpoints.web.exposure.include` 필터가 적용되는 정확한 시점은?
- JMX Endpoint를 MBean으로 변환하는 과정은?
- `management.server.port`를 별도로 설정하면 어떤 효과가 있는가?
- Actuator 포트를 앱 포트와 분리하는 이유와 방법은?

---

## 🔬 내부 동작 원리

### 1. 처리 경로 비교

```
HTTP Exposure 경로:
  @Endpoint Bean 탐색 (WebEndpointDiscoverer)
    → Exposure 필터 (management.endpoints.web.exposure.include)
    → ExposableWebEndpoint 목록 생성
    → WebMvcEndpointHandlerMapping에 경로 등록
    → DispatcherServlet → HTTP 요청 처리

JMX Exposure 경로:
  @Endpoint Bean 탐색 (JmxEndpointDiscoverer)
    → Exposure 필터 (management.endpoints.jmx.exposure.include)
    → ExposableJmxEndpoint 목록 생성
    → EndpointMBeanRegistrar → MBean 생성
    → MBeanServer에 등록
    → JConsole/VisualVM → JMX 요청 처리
```

### 2. JMX 처리 — EndpointMBean 등록

```java
// JmxEndpointAutoConfiguration
@AutoConfiguration
@ConditionalOnProperty(prefix = "spring.jmx", name = "enabled",
                        havingValue = "true", matchIfMissing = true)
public class JmxEndpointAutoConfiguration {

    @Bean
    public JmxEndpointDiscoverer jmxAnnotationEndpointDiscoverer(...) {
        return new JmxEndpointDiscoverer(
            applicationContext, parameterValueMapper,
            invokerAdvisors, filters);
    }

    @Bean
    public EndpointMBeanRegistrar endpointMBeanRegistrar(
            MBeanServer mBeanServer,
            JmxEndpointDiscoverer discoverer) {
        return new EndpointMBeanRegistrar(mBeanServer, discoverer.getEndpoints());
    }
}

// EndpointMBean — JMX MBean 래퍼
// 각 @ReadOperation, @WriteOperation → MBean attribute/operation으로 매핑
// ObjectName 형식:
//   org.springframework.boot:type=Endpoint,name=Health
//   org.springframework.boot:type=Endpoint,name=Info
```

### 3. Exposure 필터 적용 시점

```java
// EndpointDiscoverer.isEndpointExposed() — 탐색 시점에 필터 적용
// Bean 등록 후, HandlerMapping 등록 전

// HTTP: WebEndpointDiscoverer
//   management.endpoints.web.exposure.include 기반
//   기본값: health, info (Boot 기본)

// JMX: JmxEndpointDiscoverer
//   management.endpoints.jmx.exposure.include 기반
//   기본값: * (전체 노출)

// 주의: JMX 기본값이 전체 노출이므로 JMX 비활성화 권장 (운영 환경)
```

```yaml
# JMX 비활성화 (운영 환경 권장)
spring:
  jmx:
    enabled: false

# 또는 JMX Endpoint만 제한
management:
  endpoints:
    jmx:
      exposure:
        include: health  # JMX도 health만 노출
```

### 4. management.server.port — Actuator 포트 분리

```yaml
# Actuator를 별도 포트로 분리
management:
  server:
    port: 8081          # 앱: 8080, Actuator: 8081
    address: 127.0.0.1  # 내부망에서만 접근 (로드밸런서 우회)
```

```java
// 포트 분리 시 내부 동작:
// ManagementContextAutoConfiguration
//   → management.server.port가 server.port와 다르면
//   → 별도 WebServerApplicationContext 생성
//   → 별도 DispatcherServlet 인스턴스 (또는 Reactive) 시작
//   → 앱 서버와 완전히 독립된 Servlet 컨테이너

// 분리 이점:
//   앱 포트: 외부 트래픽 (로드밸런서 공개)
//   Actuator 포트: 내부 트래픽만 (모니터링 시스템, 운영 도구)
//   → Actuator가 앱 성능에 미치는 영향 분리
//   → 방화벽으로 Actuator 포트 외부 차단 가능
```

### 5. HTTP와 JMX 기본 차이

```
                HTTP                    JMX
─────────────────────────────────────────────────────
기본 노출       health, info           전체 (*)
프로토콜        HTTP/HTTPS              RMI/IIOP
접근 방법       curl, 브라우저          JConsole, VisualVM
인증 통합       Spring Security         JMX 인증 설정
운영 환경 권고  필요한 것만 노출        비활성화 권장
원격 접근       가능 (방화벽 제어)      기본 비활성화 권장
```

---

## 💻 실험으로 확인하기

### 실험 1: 현재 노출된 Endpoint 확인

```bash
# HTTP로 노출된 목록
curl http://localhost:8080/actuator | jq '._links | keys'

# JMX로 노출된 목록 (JConsole 또는)
java -jar app.jar &
jconsole localhost:<jmx-port>
# MBeans 탭 → org.springframework.boot → Endpoint
```

### 실험 2: 포트 분리 확인

```bash
# 앱 포트
curl http://localhost:8080/api/users  # 앱 API
curl http://localhost:8080/actuator   # 404 (Actuator 없음)

# Actuator 포트
curl http://localhost:8081/actuator   # Actuator만 응답
curl http://localhost:8081/api/users  # 404 (앱 없음)
```

---

## 📌 핵심 정리

```
HTTP vs JMX 처리 경로
  HTTP: WebEndpointDiscoverer → WebMvcEndpointHandlerMapping → DispatcherServlet
  JMX:  JmxEndpointDiscoverer → EndpointMBeanRegistrar → MBeanServer

기본 노출 차이
  HTTP: health, info 만 (보안 지향 기본값)
  JMX: 전체 노출 → spring.jmx.enabled=false 권장

Actuator 포트 분리
  management.server.port=8081
  management.server.address=127.0.0.1
  → 별도 WebServerApplicationContext 생성
  → 앱/Actuator 트래픽 완전 분리
```

---

## 🤔 생각해볼 문제

**Q1.** `@WebEndpoint`로 정의한 Endpoint는 JMX에서 접근 가능한가?

**Q2.** `management.server.port`를 `-1`로 설정하면 어떻게 되는가?

**Q3.** Actuator HTTP 포트를 분리했을 때 Spring Security 설정이 두 포트에 모두 적용되는가?

> 💡 **해설**
>
> **Q1.** 아니다. `@WebEndpoint`는 `WebEndpointFilter`로 필터링되어 `WebEndpointDiscoverer`에서만 탐색된다. `JmxEndpointDiscoverer`는 `@Endpoint`(기술 독립)와 `@JmxEndpoint`만 처리한다. `@WebEndpoint`는 HTTP 전용으로 JMX에서는 보이지 않는다.
>
> **Q2.** `management.server.port=-1`로 설정하면 Actuator HTTP Endpoint 전체가 비활성화된다. 별도 포트도 없고, 앱 포트에서도 Actuator가 응답하지 않는다. JMX는 영향받지 않는다. 완전히 HTTP Actuator를 끄고 싶을 때 사용한다.
>
> **Q3.** 아니다. Actuator 포트가 분리되면 별도 `WebServerApplicationContext`가 생성되며, 이 컨텍스트에는 앱의 Spring Security 설정이 자동으로 적용되지 않는다. Actuator 포트에는 별도 Security 설정이 필요하다. `ManagementWebSecurityAutoConfiguration`이 Actuator 컨텍스트에 기본 Security를 적용하지만, 세밀한 제어가 필요하면 `SecurityConfiguration`을 별도로 작성하고 `@Order`로 Actuator 경로에 우선 적용해야 한다.

---

<div align="center">

**[⬅️ 이전: Custom Actuator Endpoint 작성](./04-custom-endpoint.md)** | **[홈으로 🏠](../README.md)** | **[다음: Actuator Security 설정 ➡️](./06-actuator-security.md)**

</div>
