# JarLauncher vs WarLauncher — ClassLoader 전략 차이와 WAR 배포 선택 기준

---

## 🎯 핵심 질문

- `JarLauncher`와 `WarLauncher`의 ClassLoader 전략은 어떻게 다른가?
- WAR 배포 시 `SpringBootServletInitializer`는 왜 필요한가?
- 외부 Tomcat에 배포할 때 내장 Tomcat을 제외해야 하는 이유는?
- WAR 구조(`WEB-INF/classes`, `WEB-INF/lib`, `WEB-INF/lib-provided`)의 의미는?
- 임베디드 서버 vs 외부 컨테이너 — 어느 것을 선택해야 하는가?

---

## 🔬 내부 동작 원리

### 1. JarLauncher vs WarLauncher 구조 비교

```
Fat JAR (JarLauncher):
  app.jar
  ├── BOOT-INF/classes/      앱 코드
  ├── BOOT-INF/lib/          모든 의존성 (내장 서버 포함)
  └── org/.../JarLauncher    진입점

  실행: java -jar app.jar
  서버: 내장 Tomcat/Jetty/Undertow

WAR (WarLauncher):
  app.war
  ├── WEB-INF/
  │   ├── classes/           앱 코드 (서블릿 표준 위치)
  │   ├── lib/               런타임 의존성
  │   └── lib-provided/      제공된 의존성 (내장 서버 등)
  │                          → 외부 컨테이너에서는 무시
  │                          → java -jar 실행 시에는 포함
  ├── index.jsp 등           웹 리소스
  └── org/.../WarLauncher    진입점

  실행1: java -jar app.war (내장 서버로 실행)
  실행2: 외부 Tomcat webapps/에 배포
```

### 2. WarLauncher — WAR 전용 ClassLoader 전략

```java
// WarLauncher.java
public class WarLauncher extends ExecutableArchiveLauncher {

    // java -jar app.war 실행 시 진입점
    public static void main(String[] args) throws Exception {
        new WarLauncher().launch(args);
    }

    @Override
    protected boolean isNestedArchive(Archive.Entry entry) {
        // WAR 구조에서 클래스패스에 포함할 항목 정의
        if (entry.isDirectory()) {
            // WEB-INF/classes/ → 앱 코드
            return entry.getName().equals("WEB-INF/classes/");
        }
        // WEB-INF/lib/*.jar         → 런타임 의존성 (포함)
        // WEB-INF/lib-provided/*.jar → 제공된 의존성 (포함!)
        return entry.getName().startsWith("WEB-INF/lib/")
            || entry.getName().startsWith("WEB-INF/lib-provided/");
    }
    // java -jar 실행 시 lib-provided도 포함 → 내장 서버 동작
    // 외부 Tomcat 배포 시 lib-provided 무시 → 컨테이너가 서블릿 API 제공
}
```

### 3. lib-provided 동작 원리

```
의존성 스코프 → WAR 위치 매핑:

compile/runtimeOnly → WEB-INF/lib/
  → java -jar, 외부 컨테이너 모두 포함

provided/compileOnly → WEB-INF/lib-provided/
  → java -jar: 포함 (내장 서버 동작에 필요)
  → 외부 Tomcat: 무시 (컨테이너가 이미 제공)

예시:
  spring-boot-starter-tomcat (내장 Tomcat)
  → Gradle: providedRuntime("...spring-boot-starter-tomcat")
  → Maven:  <scope>provided</scope>
  → WEB-INF/lib-provided/에 위치
  → 외부 Tomcat 배포: 컨테이너의 Tomcat 사용
  → java -jar: lib-provided도 로딩 → 내장 Tomcat 사용
```

```kotlin
// Gradle — WAR 설정
plugins {
    id("war")
    id("org.springframework.boot") version "3.4.0"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")

    // 내장 Tomcat을 provided로 (WAR 배포 시 외부 컨테이너 Tomcat 사용)
    providedRuntime("org.springframework.boot:spring-boot-starter-tomcat")
    // → WEB-INF/lib-provided/에 위치
    // → java -jar로 실행 시에도 내장 Tomcat으로 정상 동작
}
```

### 4. SpringBootServletInitializer — 외부 컨테이너 진입점

```java
// 외부 Tomcat 배포 시 필수
// Servlet 3.0+ ServletContainerInitializer → SpringServletContainerInitializer
// → WebApplicationInitializer 구현체 탐색 → SpringBootServletInitializer 발견

@SpringBootApplication
public class Application extends SpringBootServletInitializer {

    // java -jar 실행 시 사용되는 main
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    // 외부 서블릿 컨테이너(Tomcat, Jetty 등)에 배포 시 사용
    // 컨테이너가 이 메서드를 호출해 Spring ApplicationContext 초기화
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(Application.class);
    }
}
```

```java
// SpringBootServletInitializer 동작 내부:
// 1. Servlet 3.0 ServletContainerInitializer SPI
//    → META-INF/services/ServletContainerInitializer 파일
//    → SpringServletContainerInitializer 등록

// 2. SpringServletContainerInitializer.onStartup()
//    → classpath에서 WebApplicationInitializer 구현체 탐색
//    → SpringBootServletInitializer 발견 → onStartup() 호출

// 3. SpringBootServletInitializer.onStartup()
//    → configure()로 SpringApplicationBuilder 설정
//    → SpringApplication 생성 + run()
//    → ApplicationContext 초기화
//    → DispatcherServlet을 ServletContext에 등록
```

### 5. 외부 컨테이너 배포 전체 흐름

```
외부 Tomcat webapps/에 app.war 배포:

1. Tomcat 시작
   → webapps/app.war 감지 → 압축 해제

2. Tomcat ClassLoader 생성
   → WEB-INF/classes/ + WEB-INF/lib/ 로딩
   → WEB-INF/lib-provided/ 무시

3. Servlet 3.0 초기화
   → SpringServletContainerInitializer.onStartup()
   → SpringBootServletInitializer.configure() 호출

4. Spring ApplicationContext 초기화
   → @SpringBootApplication 처리
   → Bean 생성, Auto-configuration

5. DispatcherServlet 등록
   → ServletContext에 "dispatcherServlet" 등록
   → URL 패턴: "/" (기본)

6. 요청 처리 시작
   → Tomcat → DispatcherServlet → @Controller
```

### 6. 임베디드 vs 외부 컨테이너 선택 기준

```
임베디드 서버 (Fat JAR) 선택:
  ✔ 마이크로서비스 / 컨테이너(Docker/K8s) 환경
  ✔ 독립 배포, 단순한 운영
  ✔ 서비스별 독립 서버 설정 필요
  ✔ 클라우드 네이티브 새 프로젝트
  ✔ Spring Boot 권장 방식

외부 컨테이너 (WAR) 선택:
  ✔ 기존 WAS(WebLogic, JBoss, Tomcat) 인프라 유지
  ✔ 하나의 WAS에 다중 앱 배포 (비용 절감)
  ✔ WAS 수준의 클러스터링/세션 공유 필요
  ✔ 레거시 Java EE 마이그레이션 단계
  ✗ 새 프로젝트에는 비권장
```

---

## 💻 실험으로 확인하기

### 실험 1: WAR 구조 확인

```bash
./gradlew bootWar
jar -tf build/libs/app.war | grep "WEB-INF/" | head -30

# 출력 예시:
# WEB-INF/classes/com/example/Application.class
# WEB-INF/lib/spring-web-6.2.0.jar         (runtime)
# WEB-INF/lib-provided/tomcat-embed-core.jar (provided)
```

### 실험 2: WAR를 java -jar로 실행

```bash
# WAR 파일을 java -jar로도 실행 가능
java -jar build/libs/app.war
# → WarLauncher가 lib-provided도 포함해 내장 Tomcat 실행
# → "Tomcat started on port(s): 8080" 정상 출력
```

### 실험 3: 외부 Tomcat 배포 확인

```bash
# Tomcat webapps에 배포
cp build/libs/app.war $CATALINA_HOME/webapps/ROOT.war

# Tomcat 시작
$CATALINA_HOME/bin/startup.sh

# 로그 확인 (Spring 초기화 로그)
tail -f $CATALINA_HOME/logs/catalina.out
# "Initializing Spring embedded WebApplicationContext" → 정상
```

---

## 📌 핵심 정리

```
JAR vs WAR 구조 차이
  JAR: BOOT-INF/classes/, BOOT-INF/lib/
  WAR: WEB-INF/classes/, WEB-INF/lib/, WEB-INF/lib-provided/

lib-provided 역할
  provided 의존성 (내장 서버 등)
  java -jar: 포함 → 내장 서버 동작
  외부 컨테이너: 무시 → 컨테이너가 제공

SpringBootServletInitializer
  외부 컨테이너 배포 필수
  Servlet 3.0 SPI → SpringBootServletInitializer.configure()
  → SpringApplication 초기화 → DispatcherServlet 등록

선택 기준
  신규 프로젝트/컨테이너 환경 → Fat JAR (임베디드)
  레거시 WAS 인프라 유지 → WAR (외부 컨테이너)
```

---

## 🤔 생각해볼 문제

**Q1.** `SpringBootServletInitializer`를 상속하면서 `main` 메서드도 유지하는 이유는 무엇인가?

**Q2.** 외부 Tomcat에 배포할 때 `spring-boot-starter-tomcat`을 `provided`로 설정하지 않으면 어떤 문제가 발생하는가?

**Q3.** WAR로 배포된 Spring Boot 앱에서 `server.port=9090` 설정은 효과가 있는가?

> 💡 **해설**
>
> **Q1.** 두 실행 방법을 모두 지원하기 위해서다. `main` 메서드는 개발 시 IDE에서 실행하거나 `java -jar app.war`로 임베디드 서버로 실행할 때 사용된다. `SpringBootServletInitializer`는 외부 서블릿 컨테이너에 배포할 때 컨테이너가 Spring을 초기화하는 진입점이다. 하나의 WAR 파일이 두 가지 실행 방식을 모두 지원해야 할 때 이 패턴이 유용하다.
>
> **Q2.** `spring-boot-starter-tomcat`이 `WEB-INF/lib/`에 포함되어 외부 Tomcat의 클래스와 충돌한다. 외부 Tomcat도 `catalina.jar`, `servlet-api.jar` 등을 제공하는데, WAR의 `WEB-INF/lib/`에 다른 버전의 Tomcat 클래스가 있으면 `ClassCastException`, `ClassNotFoundException`, 또는 예측 불가능한 동작이 발생한다. 특히 `javax.servlet.ServletContext`를 두 ClassLoader(WAR ClassLoader vs 컨테이너 ClassLoader)에서 각각 로딩하면 타입 불일치가 생긴다.
>
> **Q3.** 효과가 없다. 외부 Tomcat에 배포된 경우 포트는 Tomcat 서버 설정(`server.xml`의 `<Connector port="8080">`)이 결정한다. Spring Boot의 `server.port` 프로퍼티는 임베디드 서버를 생성할 때 `TomcatServletWebServerFactory`가 사용하는데, 외부 컨테이너 배포 시에는 임베디드 서버를 생성하지 않으므로 이 프로퍼티가 무시된다. 마찬가지로 `server.ssl.*`, `server.tomcat.*` 등 임베디드 서버 관련 프로퍼티도 모두 무시된다.

---

<div align="center">

**[⬅️ 이전: Fat JAR 구조와 JarLauncher](./01-fat-jar-structure.md)** | **[홈으로 🏠](../README.md)** | **[다음: Layered JAR for Docker ➡️](./03-layered-jar-docker.md)**

</div>
