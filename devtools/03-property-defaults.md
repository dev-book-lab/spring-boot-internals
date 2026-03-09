# Property Defaults — DevTools가 자동 적용하는 개발용 기본값

---

## 🎯 핵심 질문

- DevTools가 자동으로 적용하는 개발용 기본값 목록은?
- 이 기본값들이 어느 시점에, 어떤 우선순위로 적용되는가?
- 운영 환경에서 DevTools가 자동 비활성화되는 조건은?
- `spring.thymeleaf.cache=false` 같은 설정 없이도 동작하는 원리는?
- DevTools PropertySource의 우선순위는 `application.yml`과 어떻게 다른가?

---

## 🔍 왜 이게 존재하는가

```
문제: 개발 환경마다 반복되는 설정 복사
  # 개발 시작할 때마다 application-dev.yml에 추가
  spring.thymeleaf.cache=false
  spring.web.resources.cache.period=0
  spring.h2.console.enabled=true
  logging.level.web=DEBUG
  → 팀원마다 다르게 설정, 누락 시 동작 차이 발생

DevTools PropertyDefaults 해결:
  개발 환경에서 흔히 필요한 설정을 자동 적용
  → 아무 설정 없어도 개발에 최적화된 상태로 시작
  → 운영 배포 시 자동 비활성화 (DevTools가 JAR에 미포함)
```

---

## 😱 흔한 오해 또는 설정 실수

```yaml
# ❌ 오해: DevTools가 있으면 운영에서도 Thymeleaf 캐시가 꺼진다
# → DevTools는 Gradle: developmentOnly, Maven: optional
#   → bootJar 패키징 시 포함 안 됨 → 운영에선 기본값(cache=true) 적용

# ❌ 실수: application-dev.yml에 DevTools 기본값과 동일한 설정을 중복 작성
spring:
  thymeleaf:
    cache: false    # DevTools가 이미 false로 설정 → 중복

# ❌ 오해: spring.devtools.add-properties=false 하면 재시작도 멈춘다
# → PropertySource 추가만 비활성화, 재시작/LiveReload는 별개
```

---

## ✨ 올바른 이해와 설정

```yaml
# DevTools 기본값에 의존 (application.yml에 명시 불필요)
# 아무 설정 없어도 개발 시 자동 적용:
#   - 템플릿 캐시 비활성화
#   - 정적 리소스 HTTP 캐시 비활성화
#   - H2 콘솔 활성화
#   - web 로그 DEBUG

# 기본값을 덮어쓰고 싶을 때만 명시
spring:
  devtools:
    add-properties: true  # 기본값 (생략 가능)
  h2:
    console:
      enabled: false  # H2 콘솔만 비활성화하고 싶을 때
logging:
  level:
    web: INFO  # 요청 로그가 너무 많을 때만 낮춤
```

---

## 🔬 내부 동작 원리

### 1. DevToolsPropertyDefaultsPostProcessor — 기본값 적용

```java
// DevToolsPropertyDefaultsPostProcessor.java
// EnvironmentPostProcessor 구현
// → prepareEnvironment() 단계에서 실행

public class DevToolsPropertyDefaultsPostProcessor implements EnvironmentPostProcessor {

    // DevTools가 자동 적용하는 기본값 전체 목록
    static final Map<String, Object> PROPERTIES;

    static {
        Map<String, Object> properties = new HashMap<>();

        // ─── 웹 레이어 캐싱 비활성화 ───
        properties.put("spring.thymeleaf.cache", "false");
        properties.put("spring.freemarker.cache", "false");
        properties.put("spring.groovy.template.cache", "false");
        properties.put("spring.mustache.servlet.cache", "false");

        // ─── MVC 레이어 ───
        properties.put("spring.mvc.log-resolved-exception", "true");
        // → ExceptionHandlerExceptionResolver가 예외 로깅 활성화

        // ─── 리소스 핸들러 캐싱 비활성화 ───
        properties.put("spring.web.resources.cache.period", "0");
        properties.put("spring.web.resources.chain.cache", "false");
        // → 정적 리소스(CSS, JS, 이미지) HTTP 캐시 헤더 제거
        // → 브라우저 캐시 없이 항상 최신 파일 로딩

        // ─── H2 콘솔 ───
        properties.put("spring.h2.console.enabled", "true");
        // → H2 인메모리 DB 사용 시 /h2-console 자동 활성화

        // ─── 로깅 ───
        properties.put("logging.level.web", "DEBUG");
        // → DispatcherServlet 요청 로깅 활성화
        // → "GET /api/users, parameters={}" 등 자세한 로그

        // ─── 액추에이터 ───
        // (별도 설정 없음 — DevTools 자체 기본값 없음)

        PROPERTIES = Collections.unmodifiableMap(properties);
    }

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment,
                                        SpringApplication application) {
        if (isLocalApplication(environment)) {  // 운영 환경 확인
            if (canAddProperties(environment)) {
                // DevTools PropertySource를 가장 낮은 우선순위로 추가
                environment.getPropertySources().addLast(
                    new MapPropertySource("devtools", PROPERTIES));
            }
        }
    }
}
```

### 2. 우선순위 — 왜 기본값을 덮어쓸 수 있는가

```
PropertySource 우선순위 (높음 → 낮음):

1. 커맨드라인 인수
2. 시스템 환경변수
3. application.yml        ← 사용자 설정
4. ...
...
마지막. devtools           ← DevTools 기본값 (가장 낮음)

→ application.yml에서 spring.thymeleaf.cache=true로 설정하면
  DevTools 기본값(false)을 덮어씀
→ DevTools 기본값은 사용자가 명시하지 않은 경우에만 적용
```

```java
// addLast() → 가장 낮은 우선순위로 추가
environment.getPropertySources().addLast(
    new MapPropertySource("devtools", PROPERTIES));

// 비교: application.yml은 중간 우선순위
// → 사용자가 application.yml에서 같은 키를 설정하면 DevTools 기본값 무시
// → DevTools는 "있으면 쓰고 없으면 쓰는" 안전한 기본값 제공
```

### 3. 운영 환경 자동 비활성화 조건

```java
// DevToolsPropertyDefaultsPostProcessor.isLocalApplication()
private boolean isLocalApplication(ConfigurableEnvironment environment) {
    // ① 시스템 프로퍼티/환경변수로 명시적 비활성화
    if ("false".equals(environment.getProperty("spring.devtools.add-properties"))) {
        return false;
    }

    // ② JAR 실행 감지
    // Manifest에 "Spring-Boot-Application-Class" 항목 확인
    // → fat JAR로 패키징된 경우 이 항목이 없음
    // → java -jar 실행 시 DevTools 자동 비활성화

    // ③ spring-boot-devtools가 optional 의존성
    // → Maven: <optional>true</optional>
    // → Gradle: developmentOnly 설정
    // → fat JAR 패키징 시 포함 안 됨 → DevTools 클래스 없음 → 자동 비활성화
}
```

```kotlin
// Gradle 의존성 설정 (DevTools는 개발 전용)
dependencies {
    developmentOnly("org.springframework.boot:spring-boot-devtools")
    // → bootJar, bootWar에 포함 안 됨
    // → ./gradlew bootRun 시에만 활성화
    // → 운영 배포 JAR에는 없음 → DevTools 동작 없음
}
```

```xml
<!-- Maven 의존성 설정 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
    <!-- optional=true → 이 프로젝트를 의존하는 다른 프로젝트에 전파 안 됨 -->
    <!-- fat JAR 패키징 시 포함 여부는 별도 설정 (기본: 포함됨) -->
</dependency>
```

### 4. DevTools 기본값 목록 상세

```yaml
# DevTools가 자동으로 설정하는 값들 (application.yml에 명시 불필요)

# 템플릿 엔진 캐싱 비활성화
spring:
  thymeleaf:
    cache: false          # 템플릿 매번 파일에서 읽음 (변경사항 즉시 반영)
  freemarker:
    cache: false
  groovy:
    template:
      cache: false
  mustache:
    servlet:
      cache: false

# 정적 리소스 캐싱 비활성화
  web:
    resources:
      cache:
        period: 0         # max-age=0 → 브라우저 캐시 없음
      chain:
        cache: false      # 리소스 체인 캐시 비활성화

# MVC 설정
  mvc:
    log-resolved-exception: true  # ExceptionResolver 예외 로깅

# H2 콘솔
  h2:
    console:
      enabled: true      # /h2-console 자동 활성화

# 로깅
logging:
  level:
    web: DEBUG           # HTTP 요청/응답 상세 로깅
```

### 5. 개발 환경과 운영 환경 차이

```
개발 환경 (DevTools 활성화):
  Thymeleaf cache = false
  → 템플릿 파일을 매 요청마다 디스크에서 읽음
  → 파일 저장 후 새로고침하면 즉시 변경 반영
  → 성능 낮음 (I/O 증가) → 개발 편의성 우선

운영 환경 (DevTools 비활성화):
  Thymeleaf cache = true (기본값)
  → 파싱된 템플릿 메모리 캐시
  → 파일 변경해도 반영 안 됨 (캐시 유효기간 내)
  → 성능 높음 → 운영 효율성 우선

정적 리소스:
  개발: max-age=0 → 브라우저 캐시 없음 → 항상 최신 CSS/JS
  운영: max-age=설정값 → 브라우저 캐시 활용 → 대역폭 절약

H2 콘솔:
  개발: /h2-console 활성화 → DB 직접 조회 편의
  운영: 비활성화 필수 (보안 위험)
```

---

## 💻 실험으로 확인하기

### 실험 1: DevTools 기본값 확인

```bash
# DevTools 활성화 상태에서 설정값 확인
curl http://localhost:8080/actuator/env/spring.thymeleaf.cache | jq .
# {
#   "property": {
#     "source": "devtools",    ← DevTools PropertySource
#     "value": "false"
#   }
# }

# application.yml에 명시하면 덮어씀
spring.thymeleaf.cache=true 설정 후:
# {
#   "property": {
#     "source": "Config resource 'classpath:application.yml'",
#     "value": "true"     ← application.yml이 이김
#   }
# }
```

### 실험 2: H2 콘솔 자동 활성화

```yaml
# application.yml — H2 의존성 추가 시
spring:
  datasource:
    url: jdbc:h2:mem:devdb
    driver-class-name: org.h2.Driver
# DevTools가 spring.h2.console.enabled=true 자동 설정
# → http://localhost:8080/h2-console 접근 가능
```

### 실험 3: 로깅 레벨 확인

```bash
# DevTools 없이: DispatcherServlet 요청 로그 없음
# DevTools 있음: DEBUG 레벨로 자세한 요청 로그
# 예시 출력:
# DEBUG DispatcherServlet - GET "/api/users", parameters={}
# DEBUG RequestMappingHandlerMapping - Mapped to UserController#getUsers()
# DEBUG DispatcherServlet - Completed 200 OK, headers={...}
```

---

## ⚙️ 설정 최적화 팁

```yaml
# DevTools 기본값을 선택적으로 덮어쓰기

# H2 콘솔 비활성화 (DevTools 있어도)
spring:
  h2:
    console:
      enabled: false

# 로깅 레벨만 DEBUG에서 INFO로 낮추기 (요청 로그 너무 많을 때)
logging:
  level:
    web: INFO  # DevTools 기본 DEBUG → INFO로 낮춤

# DevTools 기본값 전체 비활성화
spring:
  devtools:
    add-properties: false
  # → DevTools PropertySource 추가 안 함
  # → 재시작/LiveReload는 여전히 동작
```

---

## 🤔 트레이드오프

```
자동 기본값 (편의성)
  장점: 설정 없이 개발 최적화 상태, 팀 간 일관성
  단점: 동작이 숨겨져 있어 파악하기 어려움
        "왜 Thymeleaf 캐시가 꺼져 있지?" 혼란 가능

명시적 설정 (가시성)
  장점: application-dev.yml에 모든 설정이 보임, 의도 명확
  단점: 팀원마다 다를 수 있음, 반복 작업

H2 콘솔 자동 활성화
  장점: 인메모리 DB 즉시 확인 편리
  단점: H2 의존성이 없어도 설정이 활성화됨 (무해하지만 혼란)
        보안 설정 없이 Spring Security와 함께 쓰면 접근 차단됨
        → security.ignored 또는 h2-console 경로 예외 설정 필요
```

---

## 📌 핵심 정리

```
적용 방식
  DevToolsPropertyDefaultsPostProcessor (EnvironmentPostProcessor)
  → prepareEnvironment() 단계에서 PropertySource 추가
  → addLast() → 가장 낮은 우선순위

주요 기본값
  템플릿 캐시 비활성화 (Thymeleaf, Freemarker 등)
  정적 리소스 HTTP 캐시 비활성화 (max-age=0)
  H2 콘솔 활성화
  Web 로깅 레벨 DEBUG

우선순위
  application.yml > DevTools 기본값
  → 사용자 명시 설정이 항상 우선

자동 비활성화 조건
  spring-boot-devtools가 classpath에 없으면 → 비활성화
  Gradle: developmentOnly → bootJar에 미포함
  spring.devtools.add-properties=false → PropertySource 추가 안 함
```

---

## 🤔 생각해볼 문제

**Q1.** `spring.devtools.add-properties=false`로 설정하면 재시작(Restart)과 LiveReload도 비활성화되는가?

**Q2.** DevTools가 `logging.level.web=DEBUG`를 기본으로 설정하면 운영 환경에서 실수로 DevTools가 포함된 JAR를 배포했을 때 어떤 위험이 있는가?

**Q3.** Thymeleaf 캐시를 비활성화하면 성능에 어느 정도 영향이 있는가? 개발 환경에서 허용 가능한 이유는 무엇인가?

> 💡 **해설**
>
> **Q1.** 아니다. `spring.devtools.add-properties=false`는 오직 PropertySource 추가만 비활성화한다. 재시작(Restart), LiveReload, Remote DevTools 기능은 별도로 동작한다. 기본값 PropertySource가 필요 없지만 재시작은 계속 사용하고 싶을 때 유용하다. 재시작을 비활성화하려면 `spring.devtools.restart.enabled=false`, LiveReload를 비활성화하려면 `spring.devtools.livereload.enabled=false`를 설정해야 한다.
>
> **Q2.** DevTools가 운영 JAR에 포함되면 `logging.level.web=DEBUG` 때문에 모든 HTTP 요청의 URL, 파라미터, 헤더, 응답 코드가 DEBUG 로그로 출력된다. 민감한 쿼리 파라미터(비밀번호, 토큰 등)나 개인정보가 로그에 노출될 위험이 있다. 또한 H2 콘솔이 활성화되어 보안 취약점이 된다. 이런 이유로 Gradle의 `developmentOnly` 설정이나 Maven의 `<optional>true</optional>`으로 운영 JAR에 포함되지 않도록 반드시 설정해야 한다.
>
> **Q3.** Thymeleaf 캐시 비활성화 시 매 요청마다 템플릿 파일을 디스크에서 읽고 파싱한다. 운영 환경에서는 캐시가 활성화되면 첫 번째 요청 후 파싱된 AST를 메모리에 유지해 이후 요청은 렌더링만 한다. 파일 I/O와 파싱 비용이 추가되므로 응답 시간이 수 ms~수십 ms 늘어날 수 있다. 개발 환경에서는 요청 수가 적고 단일 사용자가 사용하므로 이 오버헤드가 문제 없다. 오히려 템플릿 변경 후 재시작 없이 즉시 확인할 수 있는 편의성이 훨씬 중요하다.

---

<div align="center">

**[⬅️ 이전: Restart vs Reload 차이](./02-restart-vs-reload.md)** | **[홈으로 🏠](../README.md)** | **[다음: Remote Debug & Remote Update ➡️](./04-remote-debug-update.md)**

</div>
