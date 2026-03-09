<div align="center">

# 🚀 Spring Boot Internals

**"@SpringBootApplication 하나로 어떻게 모든 게 자동 설정되는가"**

<br/>

> *"Spring Boot를 사용하는 것과, Spring Boot가 어떻게 마법을 부리는지 아는 것은 다르다"*

Auto-configuration 메커니즘부터 Fat JAR 실행 원리, Embedded Server 커스터마이징, GraalVM Native Image까지  
**왜 이렇게 설계됐는가** 라는 질문으로 Spring Boot 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-17%2B-orange?style=flat-square&logo=openjdk)](https://www.java.com)
[![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?style=flat-square&logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![Docs](https://img.shields.io/badge/Docs-45개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

Spring Boot에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`@SpringBootApplication`을 붙이면 자동 설정됩니다" | `@EnableAutoConfiguration` → `AutoConfigurationImportSelector` → `spring.factories` 로딩 전 과정 |
| "`spring.datasource.url`을 설정하면 됩니다" | `DataSourceAutoConfiguration`이 `@ConditionalOnMissingBean`으로 기존 Bean을 감지하는 내부 코드 |
| "Fat JAR로 패키징해서 실행합니다" | `BOOT-INF/classes`, `BOOT-INF/lib` 구조, `JarLauncher`가 커스텀 ClassLoader로 중첩 JAR를 실행하는 원리 |
| "`application.yml`에 설정합니다" | `PropertySource` 17단계 우선순위, Relaxed Binding이 `max-pool-size` ↔ `maxPoolSize`를 동일하게 처리하는 방식 |
| "Actuator로 모니터링합니다" | `@Endpoint`, `@ReadOperation` 어노테이션이 HTTP/JMX Endpoint로 변환되는 `EndpointDiscoverer` 처리 경로 |
| 이론 나열 | 실행 가능한 코드 + Spring Boot 소스코드 직접 추적 + `--debug` 옵션 Auto-configuration 리포트 분석 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Startup](https://img.shields.io/badge/🔹_Startup_Process-SpringApplication.run()_전체_과정-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)](./startup-process/01-spring-application-run.md)
[![AutoConfig](https://img.shields.io/badge/🔹_Auto--configuration-ConditionalOnClass_동작_메커니즘-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)](./auto-configuration/01-conditional-on-class.md)
[![Property](https://img.shields.io/badge/🔹_Property_Management-ConfigurationProperties_바인딩-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)](./property-configuration/01-application-properties-yml.md)
[![Actuator](https://img.shields.io/badge/🔹_Actuator-Endpoint_구조_분석-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)](./actuator-internals/01-actuator-endpoint-structure.md)
[![Server](https://img.shields.io/badge/🔹_Embedded_Server-Tomcat_vs_Jetty_vs_Undertow-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)](./embedded-server/01-tomcat-jetty-undertow.md)
[![DevTools](https://img.shields.io/badge/🔹_DevTools-LiveReload_동작_원리-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)](./devtools/01-livereload-mechanism.md)
[![Packaging](https://img.shields.io/badge/🔹_Packaging-Fat_JAR_구조와_JarLauncher-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)](./packaging-deployment/01-fat-jar-structure.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: Spring Boot Startup Process

> **핵심 질문:** `SpringApplication.run()` 한 줄이 실행될 때, 내부에서 정확히 어떤 순서로 무슨 일이 벌어지는가?

<details>
<summary><b>SpringApplication 초기화부터 ApplicationContext 준비까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. SpringApplication.run() 전체 과정](./startup-process/01-spring-application-run.md) | `prepareEnvironment()` → `createApplicationContext()` → `refreshContext()` 전 과정, 각 단계에서 호출되는 핵심 메서드 추적 |
| [02. @SpringBootApplication 3개 어노테이션 분해](./startup-process/02-springbootapplication-decompose.md) | `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan` 각각의 역할과 조합 효과, 없앴을 때 차이 |
| [03. @EnableAutoConfiguration 동작 원리](./startup-process/03-enable-auto-configuration.md) | `AutoConfigurationImportSelector`가 후보 설정 클래스를 수집하는 과정, `ImportCandidates` 로딩 메커니즘 |
| [04. spring.factories vs @AutoConfiguration (Boot 3.x)](./startup-process/04-spring-factories-vs-autoconfiguration.md) | Boot 2.x `spring.factories` 방식과 Boot 3.x `@AutoConfiguration`+`imports` 파일 방식의 차이, 마이그레이션 이유 |
| [05. @Conditional 어노테이션 평가 순서](./startup-process/05-conditional-evaluation-order.md) | `ConditionEvaluator`가 `@Conditional`을 평가하는 시점, Phase(`PARSE_CONFIGURATION` vs `REGISTER_BEAN`) 차이, 평가 순서 충돌 |
| [06. ApplicationContext 생성 과정](./startup-process/06-application-context-creation.md) | `AnnotationConfigServletWebServerApplicationContext` 생성 조건, 웹/비웹 타입에 따른 컨텍스트 선택 로직, `refresh()` 호출 흐름 |
| [07. Banner 출력과 Startup Logging](./startup-process/07-banner-and-startup-logging.md) | `SpringApplicationBannerPrinter` 동작, `StartupInfoLogger`가 시작 시간을 측정하는 방식, 시작 로그에 담긴 정보 읽는 법 |

</details>

<br/>

### 🔹 Chapter 2: Auto-configuration Deep Dive

> **핵심 질문:** Spring Boot는 클래스패스에 무엇이 있는지 어떻게 감지하고, 어떤 기준으로 Bean을 자동 생성하는가?

<details>
<summary><b>Auto-configuration 조건 처리와 커스텀 설정 작성까지 (8개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. @ConditionalOnClass 동작 메커니즘](./auto-configuration/01-conditional-on-class.md) | ASM 기반 클래스 존재 확인 원리(클래스 로딩 없이), `FilteringSpringBootCondition`이 후보를 필터링하는 방식 |
| [02. @ConditionalOnBean vs @ConditionalOnMissingBean](./auto-configuration/02-conditional-on-bean-missing.md) | Bean 탐색 시점의 미묘한 차이, 사용자 정의 Bean이 Auto-configuration을 오버라이드하는 원리, 순서 의존성 함정 |
| [03. @AutoConfigureAfter/@AutoConfigureBefore 순서 제어](./auto-configuration/03-autoconfigure-order.md) | Auto-configuration 클래스 간 의존 순서 선언 방식, `AutoConfigurationSorter`가 위상 정렬로 순서를 결정하는 과정 |
| [04. DataSource Auto-configuration 분석](./auto-configuration/04-datasource-autoconfiguration.md) | `DataSourceAutoConfiguration` 소스 전체 추적, `spring.datasource.*` 프로퍼티가 HikariCP로 변환되는 경로 |
| [05. JPA Auto-configuration 과정](./auto-configuration/05-jpa-autoconfiguration.md) | `HibernateJpaAutoConfiguration` → `LocalContainerEntityManagerFactoryBean` 생성 체인, `spring.jpa.*` 옵션이 적용되는 지점 |
| [06. Web MVC Auto-configuration](./auto-configuration/06-web-mvc-autoconfiguration.md) | `WebMvcAutoConfiguration`이 `DispatcherServlet`, `HandlerMapping`, `ViewResolver`를 등록하는 과정, `WebMvcConfigurer` 확장 포인트 |
| [07. Custom Auto-configuration 작성 가이드](./auto-configuration/07-custom-autoconfiguration.md) | 라이브러리에 Auto-configuration 추가하는 전 과정, `@AutoConfiguration`, 조건 어노테이션 조합 실전 패턴 |
| [08. spring-boot-autoconfigure-processor 역할](./auto-configuration/08-autoconfigure-processor.md) | 컴파일 타임에 `@ConditionalOnClass` 조건을 미리 처리해 시작 성능을 높이는 APT 프로세서 동작 원리 |

</details>

<br/>

### 🔹 Chapter 3: Property & Configuration Management

> **핵심 질문:** `application.yml`의 값이 어떻게 Java 객체로 변환되고, 여러 소스의 설정이 충돌할 때 무엇이 이기는가?

<details>
<summary><b>PropertySource 우선순위부터 Type-safe 바인딩까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. application.properties vs application.yml 처리](./property-configuration/01-application-properties-yml.md) | 두 포맷의 로딩 시점과 처리 클래스 차이, 멀티 문서 YAML(`---`)이 동작하는 방식 |
| [02. @ConfigurationProperties 바인딩 메커니즘](./property-configuration/02-configuration-properties-binding.md) | `ConfigurationPropertiesBindingPostProcessor` → `Binder` → `BindHandler` 체인, 타입 변환과 검증 흐름 |
| [03. Relaxed Binding](./property-configuration/03-relaxed-binding.md) | `kebab-case`, `camelCase`, `SCREAMING_SNAKE_CASE`를 동일하게 처리하는 `ConfigurationPropertyName` 정규화 알고리즘 |
| [04. Type-safe Configuration Properties](./property-configuration/04-type-safe-configuration.md) | `record` + `@ConfigurationProperties` + `@Validated` 조합으로 불변·검증된 설정 객체 만드는 패턴, `@DefaultValue` 처리 |
| [05. @Value vs @ConfigurationProperties 비교](./property-configuration/05-value-vs-configuration-properties.md) | 두 방식의 바인딩 시점 차이, 타입 안전성과 리팩터링 용이성 트레이드오프, 각각을 선택해야 하는 기준 |
| [06. Profile 활성화 전략](./property-configuration/06-profile-activation.md) | `spring.profiles.active` vs `spring.profiles.include` vs `@Profile`, Profile 그룹과 Profile별 설정 파일 로딩 순서 |
| [07. Environment PropertySource 우선순위 17단계](./property-configuration/07-propertysource-priority.md) | 커맨드라인 인수부터 JAR 내부 기본값까지 17단계 전체 정리, 환경변수 vs `application.yml` 충돌 시 무엇이 이기는가 |

</details>

<br/>

### 🔹 Chapter 4: Actuator Internals

> **핵심 질문:** `/actuator/health`는 어떻게 동작하고, 커스텀 Endpoint와 Health Indicator는 어떻게 만드는가?

<details>
<summary><b>Actuator Endpoint 구조부터 Micrometer 통합까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Actuator Endpoint 구조](./actuator-internals/01-actuator-endpoint-structure.md) | `@Endpoint` / `@WebEndpoint` / `@JmxEndpoint` 처리 경로, `EndpointDiscoverer`가 Endpoint를 HTTP 경로로 변환하는 과정 |
| [02. Custom Health Indicator 작성](./actuator-internals/02-custom-health-indicator.md) | `HealthIndicator` 구현 방법, `CompositeHealthContributor`로 여러 하위 시스템을 묶는 패턴, 상태 집계 알고리즘 |
| [03. Metrics 수집 — Micrometer 통합](./actuator-internals/03-micrometer-metrics.md) | `MeterRegistry` 계층 구조, `@Timed` / `Counter` / `Gauge` 동작 원리, Prometheus 연동 시 데이터 흐름 |
| [04. Custom Actuator Endpoint 작성](./actuator-internals/04-custom-endpoint.md) | `@ReadOperation` / `@WriteOperation` / `@DeleteOperation` 처리 방식, 파라미터 바인딩과 반환 타입 변환 |
| [05. JMX vs HTTP Exposure](./actuator-internals/05-jmx-vs-http-exposure.md) | 두 노출 방식의 내부 처리 차이, `management.endpoints.web.exposure.include` 설정이 적용되는 필터링 지점 |
| [06. Actuator Security 설정](./actuator-internals/06-actuator-security.md) | `EndpointRequest.toAnyEndpoint()` 매처 동작 원리, Spring Security와의 통합 방식, 운영 환경 노출 최소화 전략 |

</details>

<br/>

### 🔹 Chapter 5: Embedded Server Configuration

> **핵심 질문:** Spring Boot는 어떻게 외부 서버 없이 내장 서버로 요청을 받는가? 서버를 어떻게 교체하고 커스터마이징하는가?

<details>
<summary><b>Embedded Tomcat 동작부터 SSL·HTTP/2 설정까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Tomcat vs Jetty vs Undertow 비교](./embedded-server/01-tomcat-jetty-undertow.md) | 세 서버의 아키텍처 차이(스레드 모델, 커넥터 구조), 벤치마크 특성, 교체 시 의존성 변경 방법 |
| [02. ServletWebServerFactory 동작 원리](./embedded-server/02-servlet-web-server-factory.md) | `TomcatServletWebServerFactory`가 내장 Tomcat을 초기화하는 과정, `WebServer` 인터페이스와 생명주기 관리 |
| [03. Embedded Server 커스터마이징](./embedded-server/03-embedded-server-customization.md) | `WebServerFactoryCustomizer<T>` 구현 방식, 커넥터 설정, 스레드 풀 튜닝, 요청 큐 크기 조정 |
| [04. SSL/TLS 설정](./embedded-server/04-ssl-tls-configuration.md) | Keystore/Truststore 설정 방법, `server.ssl.*` 프로퍼티가 내장 서버에 적용되는 과정, mTLS 설정 |
| [05. HTTP/2 활성화](./embedded-server/05-http2-activation.md) | `server.http2.enabled=true`가 적용되는 내부 과정, ALPN 협상 메커니즘, Tomcat/Undertow/Jetty별 H2 지원 차이 |
| [06. Context Path & Port 설정](./embedded-server/06-context-path-port.md) | `server.port=0` 랜덤 포트 할당 원리, 다중 포트 리스닝, Context Path 설정이 `DispatcherServlet` 매핑에 미치는 영향 |

</details>

<br/>

### 🔹 Chapter 6: Spring Boot DevTools

> **핵심 질문:** DevTools는 어떻게 코드 변경을 감지하고 서버를 재시작하는가? Restart와 Reload는 무엇이 다른가?

<details>
<summary><b>개발 생산성을 높이는 DevTools 내부 동작 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. LiveReload 동작 원리](./devtools/01-livereload-mechanism.md) | 내장 LiveReload 서버가 브라우저와 WebSocket으로 통신하는 방식, 파일 변경 감지 → 브라우저 리프레시 전 과정 |
| [02. Restart vs Reload 차이](./devtools/02-restart-vs-reload.md) | 두 개의 ClassLoader(Base / Restart) 분리 전략, Restart가 Full Restart보다 빠른 이유, JRebel과의 차이 |
| [03. Property Defaults — 캐싱 비활성화](./devtools/03-property-defaults.md) | DevTools가 자동으로 적용하는 개발용 기본값 목록(`spring.thymeleaf.cache=false` 등), 운영 환경에서 자동 비활성화 조건 |
| [04. Remote Debug & Remote Update](./devtools/04-remote-debug-update.md) | `spring.devtools.remote.secret` 설정으로 원격 서버에 클래스 변경을 푸시하는 메커니즘, HTTP 터널링 방식 |
| [05. Build Plugin 통합](./devtools/05-build-plugin-integration.md) | Maven / Gradle Spring Boot Plugin이 Fat JAR를 빌드하는 방식, `spring-boot:run`과 일반 `run`의 ClassLoader 차이 |

</details>

<br/>

### 🔹 Chapter 7: Packaging & Deployment

> **핵심 질문:** `java -jar app.jar`는 어떻게 동작하는가? 컨테이너 환경과 Native Image에 맞는 패키징은 무엇이 다른가?

<details>
<summary><b>Fat JAR 구조부터 GraalVM Native Image까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Fat JAR 구조와 JarLauncher](./packaging-deployment/01-fat-jar-structure.md) | `BOOT-INF/classes`, `BOOT-INF/lib` 디렉토리 구조, `JarLauncher`가 중첩 JAR를 로딩하는 커스텀 ClassLoader 구현 |
| [02. JarLauncher vs WarLauncher](./packaging-deployment/02-jar-vs-war-launcher.md) | 두 Launcher의 ClassLoader 전략 차이, WAR 배포 시 `ServletInitializer` 역할, 임베디드 vs 외부 컨테이너 배포 선택 기준 |
| [03. Layered JAR for Docker 최적화](./packaging-deployment/03-layered-jar-docker.md) | `jarmode=layertools`로 레이어를 분리하는 원리, 의존성 레이어 캐싱으로 Docker 빌드 시간을 줄이는 전략 |
| [04. Native Image with GraalVM](./packaging-deployment/04-native-image-graalvm.md) | AOT(Ahead-Of-Time) 컴파일 과정, 리플렉션·프록시 힌트 등록 방법, Spring Boot AOT 엔진이 힌트를 자동 생성하는 원리 |
| [05. Cloud Native Buildpacks](./packaging-deployment/05-cloud-native-buildpacks.md) | `spring-boot:build-image` 명령이 Buildpack으로 컨테이너 이미지를 만드는 과정, Dockerfile 없이 OCI 이미지 생성 |
| [06. Production-ready Checklist](./packaging-deployment/06-production-checklist.md) | Actuator 보안 설정, 로깅 레벨 전략, JVM 메모리 튜닝, Graceful Shutdown 설정, Health Check 엔드포인트 설계 |

</details>

---

## 🗺️ 목적별 학습 경로

<details>
<summary><b>🟢 "Spring Boot가 어떻게 자동 설정하는지 궁금하다" — Auto-configuration 입문 (1주)</b></summary>

<br/>

```
Day 1  Ch1-01  SpringApplication.run() 전체 과정
Day 2  Ch1-02  @SpringBootApplication 3개 어노테이션 분해
Day 3  Ch1-03  @EnableAutoConfiguration 동작 원리
Day 4  Ch2-01  @ConditionalOnClass 동작 메커니즘
Day 5  Ch2-02  @ConditionalOnBean vs @ConditionalOnMissingBean
Day 6  Ch2-04  DataSource Auto-configuration 분석 (실전 예시)
Day 7  Ch2-07  Custom Auto-configuration 작성 가이드
```

</details>

<details>
<summary><b>🔵 "설정 관리와 운영 환경 배포를 제대로 하고 싶다" — 실무 중심 (2주)</b></summary>

<br/>

**Week 1 — 설정 관리**
```
Ch3-02  @ConfigurationProperties 바인딩 메커니즘
Ch3-04  Type-safe Configuration Properties (record + @Validated)
Ch3-06  Profile 활성화 전략
Ch3-07  PropertySource 우선순위 17단계
Ch4-01  Actuator Endpoint 구조
Ch4-02  Custom Health Indicator 작성
```

**Week 2 — 패키징과 배포**
```
Ch5-01  Tomcat vs Jetty vs Undertow (서버 선택 기준)
Ch5-03  Embedded Server 커스터마이징 (스레드 풀 튜닝)
Ch7-01  Fat JAR 구조와 JarLauncher
Ch7-03  Layered JAR for Docker 최적화
Ch7-06  Production-ready Checklist
```

</details>

<details>
<summary><b>🔴 "Spring Boot 소스코드를 직접 읽고 내부를 완전히 이해하고 싶다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — 시작 과정 완전 추적
        → SpringApplication.run() 실행 중 --debug 옵션으로 Auto-configuration 리포트 직접 확인

2주차  Chapter 2 전체 — Auto-configuration 조건 평가
        → spring-boot-autoconfigure 소스에서 DataSource, JPA, Web MVC 설정 클래스 직접 읽기

3주차  Chapter 3 전체 — Property 바인딩 내부
        → Binder 소스 추적 + 직접 @ConfigurationProperties 클래스 설계해보기

4주차  Chapter 4 전체 — Actuator
        → 커스텀 Health Indicator + 커스텀 Endpoint 직접 구현

5주차  Chapter 5 전체 — 내장 서버
        → WebServerFactoryCustomizer로 Tomcat 스레드 풀 튜닝 실험

6주차  Chapter 6 전체 — DevTools 동작 원리
        → Remote Update 설정해보기 + Restart ClassLoader 계층 확인

7주차  Chapter 7 전체 — 패키징과 배포
        → Layered JAR + Docker 이미지 빌드 실습
        → GraalVM Native Image 빌드 및 리플렉션 힌트 등록 실습
```

</details>

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이게 존재하는가** | 문제 상황과 설계 배경 |
| 😱 **흔한 오해 또는 설정 실수** | Before — 많은 개발자가 틀리는 방식 |
| ✨ **올바른 이해와 설정** | After — 원리를 알고 난 후의 올바른 접근 |
| 🔬 **내부 동작 원리** | Spring Boot 소스코드 직접 추적 + 다이어그램 |
| 💻 **실험으로 확인하기** | 직접 실행 가능한 코드 + `--debug` 리포트 분석 |
| ⚙️ **설정 최적화 팁** | 운영 환경 설정 중심의 실전 팁 |
| 🤔 **트레이드오프** | 이 설계의 장단점, 언제 다른 방법을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🔗 선행 학습 레포지토리

이 레포는 다음 레포지토리의 이해를 전제로 합니다.

| 레포 | 주요 내용 | 필요 챕터 |
|------|----------|-----------|
| [spring-core-deep-dive](https://github.com/dev-book-lab/spring-core-deep-dive) | IoC 컨테이너, DI, AOP, Bean 생명주기 | `@Configuration`, `@Conditional` 이해 필수 |
| [spring-data-transaction](https://github.com/dev-book-lab/spring-data-transaction) | JPA, 트랜잭션, 쿼리 최적화 | Chapter 4(JPA Auto-configuration) 학습 시 참고 |

> 💡 Auto-configuration 챕터(Ch1~Ch2)만 목표라면 Spring Core 선행 없이도 독립적으로 학습 가능합니다.

---

## 🙏 Reference

- [Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Boot Source Code (GitHub)](https://github.com/spring-projects/spring-boot)
- [Spring Boot Auto-configuration Report (`--debug`)](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.spring-application.startup-tracking)
- [Micrometer Documentation](https://micrometer.io/docs)
- [GraalVM Native Image Documentation](https://www.graalvm.org/latest/reference-manual/native-image/)
- [Cloud Native Buildpacks](https://buildpacks.io/)
- [Baeldung Spring Boot Guides](https://www.baeldung.com/spring-boot)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"Spring Boot를 사용하는 것과, Spring Boot가 어떻게 마법을 부리는지 아는 것은 다르다"*

</div>
