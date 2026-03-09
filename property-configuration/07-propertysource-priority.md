# Environment PropertySource 우선순위 17단계 — 충돌 시 무엇이 이기는가

---

## 🎯 핵심 질문

- Spring Boot의 PropertySource 우선순위 전체 순서는?
- 환경변수와 `application.yml`이 충돌할 때 어느 것이 이기는가?
- 커맨드라인 인수는 왜 가장 높은 우선순위인가?
- `@PropertySource`로 추가한 파일은 어디에 위치하는가?
- 우선순위를 직접 확인하고 디버깅하는 방법은?

---

## 🔍 핵심 개념

```
PropertySource 체인:

MutablePropertySources (Environment 내부)
  → 순서가 있는 PropertySource 목록
  → 앞에 있는 것이 높은 우선순위
  → 같은 키가 여러 소스에 있으면 첫 번째 소스 값 사용

Environment.getProperty("key"):
  → 등록된 PropertySource를 앞에서부터 순회
  → 첫 번째로 key가 있는 소스의 값 반환
  → 모두 없으면 null
```

---

## 🔬 내부 동작 원리

### 1. PropertySource 우선순위 전체 목록

```
우선순위 (1 = 가장 높음, 17 = 가장 낮음)

Servlet/Reactive 웹 환경 기준:

1.  커맨드라인 인수
    --server.port=9090 또는 --spring.profiles.active=prod
    SimpleCommandLinePropertySource

2.  SPRING_APPLICATION_JSON 프로퍼티
    환경변수 또는 시스템 프로퍼티로 JSON 문자열 전달
    SPRING_APPLICATION_JSON='{"server":{"port":9090}}'

3.  ServletConfig init 파라미터
    web.xml 또는 WebApplicationInitializer에서 설정

4.  ServletContext init 파라미터
    web.xml의 <context-param>

5.  JNDI 속성 (java:comp/env/)

6.  Java 시스템 프로퍼티 (System.getProperties())
    -Dserver.port=9090

7.  OS 환경변수 (System.getenv())
    SERVER_PORT=9090 (Relaxed Binding으로 server.port 매핑)

8.  RandomValuePropertySource
    random.int, random.long, random.uuid 등
    ${random.int} → 매번 다른 랜덤값

9.  application-{profile}.properties / .yml (외부 파일)
    file:./config/{profile}/ > file:./{profile}/ > ...

10. application.properties / .yml (외부 파일)
    file:./config/ > file:./ 경로

11. @PropertySource 어노테이션으로 로딩한 파일
    @PropertySource("classpath:custom.properties")
    ⚠️ YAML 파일 미지원 (properties만)

12. application-{profile}.properties / .yml (내부 classpath)
    classpath:/config/ > classpath:/

13. application.properties / .yml (내부 classpath)
    classpath:/config/ > classpath:/

14. @SpringBootTest의 properties 속성
    @SpringBootTest(properties = "server.port=9999")

15. SpringApplication.setDefaultProperties()
    코드로 설정한 기본값

16. @TestPropertySource (테스트 전용)
    @TestPropertySource(properties = "...", locations = "...")

17. 기본값 PropertySource
    SpringApplication 내부 기본값
```

### 2. 가장 중요한 충돌 시나리오

```
시나리오: 환경변수 vs application.yml

application.yml:
  spring:
    datasource:
      url: jdbc:h2:mem:devdb   ← 우선순위 13위

환경변수:
  SPRING_DATASOURCE_URL=jdbc:mysql://prod:3306/mydb  ← 우선순위 7위

결과: jdbc:mysql://prod:3306/mydb (환경변수가 이김)

→ Kubernetes/Docker에서 환경변수로 설정 오버라이드하는 패턴의 기반
```

```
시나리오: 시스템 프로퍼티 vs 환경변수

-Dserver.port=8181   ← 우선순위 6위
SERVER_PORT=9090     ← 우선순위 7위

결과: 8181 (시스템 프로퍼티가 이김)

시나리오: 커맨드라인 vs 모든 것

--server.port=7070   ← 우선순위 1위
-Dserver.port=8181   ← 우선순위 6위
SERVER_PORT=9090     ← 우선순위 7위
application.yml      ← 우선순위 13위

결과: 7070 (커맨드라인이 항상 이김)
```

### 3. 외부 파일 vs 내부 파일 우선순위

```
외부 파일(file:)이 내부 파일(classpath:)보다 높은 우선순위:

우선순위 9-10: 외부 application.yml
  file:./config/*/application.yml   (가장 높음)
  file:./config/application.yml
  file:./application.yml

우선순위 12-13: 내부 application.yml
  classpath:/config/application.yml
  classpath:/application.yml        (가장 낮음)

→ JAR 파일 외부에 config/application.yml을 두면
  JAR 내부 설정 전체 오버라이드 가능
  (클라우드 ConfigMap, 사이드카 방식 설정 변경에 활용)
```

### 4. @PropertySource의 한계와 위치

```java
// @PropertySource는 우선순위 11위 — application.yml보다 낮음!
@SpringBootApplication
@PropertySource("classpath:custom.properties")
public class App { }

// 결과:
// custom.properties의 값 < application.yml의 같은 키
// → application.yml이 이김

// ⚠️ @PropertySource는 YAML 미지원
@PropertySource("classpath:custom.yml")  // ❌ YAML 로딩 안 됨
// → PropertiesPropertySourceLoader만 사용 → yml 미지원

// YAML 추가 로딩 방법:
// ① spring.config.import 사용 (권장)
// application.yml:
// spring.config.import: classpath:custom.yml
// → application.yml과 같은 우선순위로 처리

// ② 직접 PropertySourcesPlaceholderConfigurer 커스터마이징 (복잡)
```

### 5. MutablePropertySources 내부 확인

```java
// 실제 등록된 PropertySource 목록 확인
@Autowired
ConfigurableEnvironment environment;

@PostConstruct
void printPropertySources() {
    environment.getPropertySources().forEach(ps ->
        System.out.printf("[%s] %s%n",
            ps.getClass().getSimpleName(),
            ps.getName()));
}
```

```
출력 예시:
[SimpleCommandLinePropertySource] commandLineArgs
[MapPropertySource] spring.application.json
[PropertiesPropertySource] systemProperties
[SystemEnvironmentPropertySource] systemEnvironment
[RandomValuePropertySource] random
[OriginTrackedMapPropertySource] Config resource 'class path resource [application-dev.yml]'
[OriginTrackedMapPropertySource] Config resource 'class path resource [application.yml]'
```

### 6. Environment 직접 탐색 디버깅

```java
// 특정 키가 어느 PropertySource에서 왔는지 확인
@Autowired ConfigurableEnvironment environment;

public String findPropertySource(String key) {
    for (PropertySource<?> ps : environment.getPropertySources()) {
        if (ps.containsProperty(key)) {
            return String.format("키 '%s' = '%s' (소스: %s)",
                key, ps.getProperty(key), ps.getName());
        }
    }
    return "키 없음: " + key;
}

// 또는 Actuator env 엔드포인트
// GET /actuator/env/server.port
// → {
//     "property": {"value": "8080", "origin": "...application.yml:5:14"},
//     "activeProfiles": ["dev"],
//     "propertySources": [...]
//   }
```

### 7. 특수 PropertySource — RandomValuePropertySource

```yaml
# RandomValuePropertySource (우선순위 8위)
# 매번 다른 값 생성:
my.secret: ${random.uuid}        # UUID
my.random-int: ${random.int}     # 랜덤 정수
my.token: ${random.value}        # 랜덤 32자 hex
my.limited: ${random.int[1,100]} # 1~100 사이 정수

# 사용 사례:
# 인스턴스별 고유 ID (로드밸런서 뒤에서 각 인스턴스 구별)
# 테스트용 임시 비밀값
# 주의: 애플리케이션 재시작마다 값이 바뀜 → 영속적 값에는 부적합
```

---

## 💻 실험으로 확인하기

### 실험 1: 우선순위 직접 확인

```bash
# application.yml: server.port=8080
# 환경변수로 오버라이드
SERVER_PORT=9090 java -jar app.jar
# → 9090 (환경변수 7위 > application.yml 13위)

# 시스템 프로퍼티로 오버라이드
SERVER_PORT=9090 java -Dserver.port=8181 -jar app.jar
# → 8181 (시스템 프로퍼티 6위 > 환경변수 7위)

# 커맨드라인으로 오버라이드
SERVER_PORT=9090 java -Dserver.port=8181 -jar app.jar --server.port=7070
# → 7070 (커맨드라인 1위 > 모두)
```

### 실험 2: Actuator env로 소스 확인

```bash
# Actuator 활성화 후
curl http://localhost:8080/actuator/env/server.port

# 응답:
# {
#   "property": {
#     "source": "systemEnvironment",
#     "value": "9090"
#   },
#   "activeProfiles": ["dev"],
#   "propertySources": [
#     {"name": "commandLineArgs", "property": null},
#     {"name": "systemEnvironment", "property": {"value": "9090"}},
#     {"name": "application.yml", "property": {"value": "8080"}}
#   ]
# }
```

### 실험 3: 외부 config 파일로 JAR 내부 설정 오버라이드

```
배포 구조:
  /app/
    myapp.jar
    config/
      application.yml    ← 외부 파일 (우선순위 10위)

# 외부 config/application.yml:
server.port: 9999
spring.datasource.url: jdbc:mysql://prod:3306/mydb

→ JAR 내부 classpath:application.yml보다 높은 우선순위
→ JAR 교체 없이 설정 변경 가능
```

---

## ⚙️ 설정 최적화 팁

```
운영 환경 설정 전략 (우선순위 활용):

보안이 필요한 값 (DB 패스워드, API 키):
  → 환경변수 또는 Vault (7위)
  → application.yml에 절대 기록 안 함

환경별 다른 값 (DB URL, 서버 주소):
  → 환경변수 또는 외부 config 파일 (7위, 9-10위)

기능 플래그, 로그 레벨:
  → application-{profile}.yml (12위)

기본값 (로컬 개발용):
  → application.yml (13위)

긴급 설정 변경:
  → 커맨드라인 인수 (1위)
  → 재배포 없이 즉시 적용 후 영구 설정 파일에 반영
```

---

## 📌 핵심 정리

```
핵심 우선순위 (실무에서 자주 충돌하는 것)

1위  커맨드라인 --key=value       (모든 것 오버라이드)
6위  시스템 프로퍼티 -Dkey=value
7위  환경변수 KEY=VALUE            (컨테이너 환경 설정)
9~10 외부 application.yml          (JAR 외부 config/)
12~13 내부 application.yml          (JAR 내부 classpath)
15위  SpringApplication 기본값       (코드 기본값)

환경변수 > application.yml
  컨테이너/Kubernetes 환경에서 환경변수로 설정 오버라이드
  Relaxed Binding으로 SCREAMING_SNAKE → kebab-case 매핑

@PropertySource 주의점
  우선순위 11위 → application.yml보다 낮음
  YAML 미지원 → spring.config.import 사용 권장

디버깅
  /actuator/env/{key} → 어느 소스에서 왔는지 확인
  environment.getPropertySources() 직접 순회
  --debug 로그로 Config 파일 로딩 순서 확인
```

---

## 🤔 생각해볼 문제

**Q1.** `System.getProperties()`에 `server.port=8181`을 설정하고 `System.getenv()`에 `SERVER_PORT=9090`이 있을 때, 어느 값이 사용되는가? 그 이유는?

**Q2.** `@TestPropertySource(locations = "classpath:test.properties")`와 `@SpringBootTest(properties = "server.port=9999")`를 동시에 사용하면 어느 것이 높은 우선순위인가?

**Q3.** `spring.config.import: classpath:secret.properties`로 가져온 파일의 우선순위는 `@PropertySource("classpath:secret.properties")`와 비교해 어떻게 다른가?

> 💡 **해설**
>
> **Q1.** `System.getProperties()`의 `server.port=8181`이 사용된다. Java 시스템 프로퍼티(우선순위 6위)가 OS 환경변수(우선순위 7위)보다 높기 때문이다. `MutablePropertySources`에서 `systemProperties` PropertySource가 `systemEnvironment`보다 앞에 등록되므로, `Environment.getProperty("server.port")` 조회 시 `systemProperties`에서 먼저 값을 찾아 반환한다. `-D` 옵션이 `SERVER_PORT` 환경변수보다 우선한다는 의미다.
>
> **Q2.** `@SpringBootTest(properties = "server.port=9999")`가 더 높은 우선순위다. `@SpringBootTest.properties`는 우선순위 14위로 처리되고, `@TestPropertySource`는 우선순위 16위(테스트 전용 소스)로 처리된다. 즉 `@SpringBootTest(properties = ...)` > `@TestPropertySource`. 다만 `@TestPropertySource`는 `locations`로 파일을 지정하면 Properties 파일을 통째로 로딩하는 장점이 있어 테스트 전용 설정 파일 관리에 유리하다.
>
> **Q3.** `spring.config.import`로 가져온 파일은 `ConfigDataEnvironment`가 처리하며 가져온 파일 자체보다 높은 우선순위를 가진다(즉 import된 파일이 application.yml보다 높은 우선순위). 반면 `@PropertySource`로 추가된 파일은 우선순위 11위로 `application.yml`(13위)보다 높지만 환경변수(7위)보다 낮다. 따라서 비밀값을 외부 파일로 관리할 때 `spring.config.import`를 사용하면 환경변수와 동등하거나 더 유연한 우선순위 제어가 가능하다. 실무에서는 비밀값은 `spring.config.import: optional:configtree:/run/secrets/` 방식으로 쿠버네티스 Secret을 파일로 마운트해서 읽는 패턴을 많이 사용한다.

---

<div align="center">

**[⬅️ 이전: Profile 활성화 전략](./06-profile-activation.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — Actuator 엔드포인트 구조 ➡️](../actuator-internals/01-actuator-endpoint-structure.md)**

</div>
