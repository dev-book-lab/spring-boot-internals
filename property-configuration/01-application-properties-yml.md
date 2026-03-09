# application.properties vs application.yml 처리 — 로딩 시점과 멀티 문서 YAML

---

## 🎯 핵심 질문

- 두 포맷은 어느 클래스가, 어느 시점에 처리하는가?
- `PropertySourceLoader` 구현체의 차이는?
- 멀티 문서 YAML(`---`)은 어떻게 처리되는가?
- 같은 키가 두 파일 모두에 있으면 어느 것이 이기는가?
- `spring.config.import`로 외부 파일을 불러오는 원리는?

---

## 🔍 왜 이게 존재하는가

```
두 포맷의 특성:

application.properties:
  key=value 평면 구조
  → 간단, 빠른 파싱
  → 리스트: my.list[0]=a, my.list[1]=b
  → 중첩: my.db.url=... (key 반복)

application.yml:
  들여쓰기 계층 구조
  → 중첩 표현 간결
  → 리스트 자연스러움
  → 멀티 문서(---) 지원 → 하나의 파일에 Profile별 설정
  → 타입(boolean, int) 자동 추론 주의 필요
```

---

## 🔬 내부 동작 원리

### 1. ConfigDataEnvironmentPostProcessor — 설정 파일 로딩 진입점

```java
// 호출 시점: SpringApplication.run()
//   → prepareEnvironment()
//     → EnvironmentPostProcessor 실행
//       → ConfigDataEnvironmentPostProcessor

public class ConfigDataEnvironmentPostProcessor implements EnvironmentPostProcessor {

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment,
                                        SpringApplication application) {
        new ConfigDataEnvironment(this.logFactory, this.bootstrapContext,
            environment, application.getResourceLoader(),
            application.getAdditionalProfiles(), this.environmentUpdateListener)
            .processAndApply();
    }
}
```

### 2. PropertySourceLoader — 두 포맷 처리

```java
// PropertySourceLoader 인터페이스
public interface PropertySourceLoader {
    // 지원하는 파일 확장자
    String[] getFileExtensions();
    // 파일 → PropertySource 목록으로 변환
    List<PropertySource<?>> load(String name, Resource resource) throws IOException;
}

// ① PropertiesPropertySourceLoader — .properties, .xml
public class PropertiesPropertySourceLoader implements PropertySourceLoader {
    @Override
    public String[] getFileExtensions() {
        return new String[]{"properties", "xml"};
    }

    @Override
    public List<PropertySource<?>> load(String name, Resource resource) {
        // Properties.load() → UTF-8 읽기
        // 결과: PropertiesPropertySource 1개 반환 (멀티 문서 없음)
        List<Map<String, ?>> properties = loadProperties(resource);
        return properties.stream()
            .map(p -> new OriginTrackedMapPropertySource(name, p, true))
            .collect(Collectors.toList());
    }
}

// ② YamlPropertySourceLoader — .yml, .yaml
public class YamlPropertySourceLoader implements PropertySourceLoader {
    @Override
    public String[] getFileExtensions() {
        return new String[]{"yml", "yaml"};
    }

    @Override
    public List<PropertySource<?>> load(String name, Resource resource) {
        // SnakeYAML로 파싱
        // --- 구분자마다 별도 PropertySource 생성 → 멀티 문서 지원
        List<Map<String, Object>> loaded = new OriginTrackedYamlLoader(resource).load();
        List<PropertySource<?>> propertySources = new ArrayList<>(loaded.size());
        for (int i = 0; i < loaded.size(); i++) {
            String documentNumber = (loaded.size() != 1) ? " (document #" + i + ")" : "";
            propertySources.add(new OriginTrackedMapPropertySource(
                name + documentNumber, loaded.get(i), true));
        }
        return propertySources;
    }
}
```

### 3. 멀티 문서 YAML — `---` 처리

```yaml
# application.yml
spring:
  application:
    name: my-app
server:
  port: 8080
---
# 두 번째 문서 — prod 프로파일에만 적용
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 443
  ssl:
    enabled: true
---
# 세 번째 문서 — dev 프로파일에만 적용
spring:
  config:
    activate:
      on-profile: dev
spring:
  datasource:
    url: jdbc:h2:mem:devdb
```

```java
// OriginTrackedYamlLoader — --- 구분으로 문서 분리
class OriginTrackedYamlLoader extends YamlProcessor {

    List<Map<String, Object>> load() {
        List<Map<String, Object>> documents = new ArrayList<>();
        process((properties, map) -> documents.add(getFlattenedMap(map)));
        return documents;
        // 결과: 3개의 Map — 각 문서가 별도 PropertySource
    }
}

// 멀티 문서 처리:
// 문서 1: {spring.application.name=my-app, server.port=8080}
// 문서 2: {spring.config.activate.on-profile=prod, server.port=443, ...}
// 문서 3: {spring.config.activate.on-profile=dev, spring.datasource.url=...}

// 프로파일 활성화 시:
//   -Dspring.profiles.active=prod
//   → 문서 1 + 문서 2 활성화 (문서 3 스킵)
//   → server.port = 443 (문서 2가 문서 1을 덮어씀)
```

### 4. ConfigDataLocationResolver — 파일 탐색 경로

```java
// 기본 탐색 경로 (우선순위 역순 — 낮은 것부터 높은 것으로)
// 1. classpath:/                (클래스패스 루트)
// 2. classpath:/config/         (클래스패스 config 디렉토리)
// 3. file:./                    (현재 디렉토리)
// 4. file:./config/             (현재 디렉토리 config 서브디렉토리)
// 5. file:./config/*/           (config 하위 디렉토리 전체)

// 같은 위치에서 .properties와 .yml이 모두 있으면:
// properties 먼저 로딩, yml 나중 로딩
// → 같은 키라면 yml이 이김 (나중에 로딩된 것이 앞 것을 덮어씀)

// spring.config.name으로 파일명 변경 가능 (기본: application)
// spring.config.location으로 경로 완전 교체
// spring.config.additional-location으로 경로 추가
```

### 5. spring.config.import — 외부 파일 불러오기

```yaml
# application.yml
spring:
  config:
    import:
      - optional:classpath:custom-config.yml  # 없어도 오류 없음
      - optional:file:/etc/myapp/secrets.yml  # 외부 파일
      - configtree:/run/secrets/              # 쿠버네티스 Secret 마운트
```

```java
// ConfigDataImporter — import 처리
// import된 파일은 가져온 파일보다 높은 우선순위
// → application.yml 에서 import한 secrets.yml의 값이
//    application.yml 자체 값보다 우선
```

### 6. OriginTrackedMapPropertySource — 출처 추적

```java
// 모든 프로퍼티에 로딩 위치(파일명, 라인번호) 기록
// → 바인딩 실패 시 정확한 오류 위치 출력
// 예: "Binding to target ... failed:
//      Property: spring.datasource.url
//      Value: "jdbc:invalid"
//      Origin: class path resource [application.yml] - 5:10"
```

---

## 💻 실험으로 확인하기

### 실험 1: properties vs yml 우선순위

```
같은 위치에 두 파일 모두 존재 시:
classpath:application.properties: server.port=8080
classpath:application.yml:        server.port=9090

결과: 9090 (yml이 이김 — 나중에 로딩)

이유:
  StandardConfigDataLocationResolver가 같은 location에서
  properties → yml 순으로 로딩
  → yml이 동일 이름의 PropertySource를 덮어씀
```

### 실험 2: 멀티 문서 YAML 활성화 확인

```bash
java -jar app.jar --spring.profiles.active=prod
# server.port = 443 (prod 문서 적용)

java -jar app.jar
# server.port = 8080 (첫 번째 문서만)
```

### 실험 3: config/application.yml vs classpath:application.yml

```
file:./config/application.yml   port=9000  (높은 우선순위)
classpath:application.yml       port=8080  (낮은 우선순위)

결과: 9000
→ file 경로가 classpath보다 높은 우선순위
```

---

## ⚙️ 설정 최적화 팁

```yaml
# Profile별 파일 분리 vs 멀티 문서 YAML
# 소규모: 멀티 문서 YAML (하나의 파일 관리 편리)
# 대규모: 파일 분리 (application-prod.yml, application-dev.yml)
#   → PR 리뷰 시 변경 명확, 파일별 권한 분리 가능

# spring.config.import 활용 — 비밀값 분리
spring:
  config:
    import: optional:file:/run/secrets/db-password.properties
# → Kubernetes Secret을 파일로 마운트해서 읽기
# → Git에 비밀값 없이 관리 가능
```

---

## 📌 핵심 정리

```
로딩 시점
  prepareEnvironment() → ConfigDataEnvironmentPostProcessor
  → ConfigDataEnvironment.processAndApply()

처리 클래스
  .properties → PropertiesPropertySourceLoader
  .yml        → YamlPropertySourceLoader (SnakeYAML)

같은 위치 충돌
  yml > properties (나중 로딩이 앞 것 덮어씀)

멀티 문서 YAML
  --- 구분자마다 별도 PropertySource 생성
  spring.config.activate.on-profile로 조건부 활성화

탐색 우선순위 (높은 순)
  file:./config/*/ > file:./config/ > file:./ > classpath:/config/ > classpath:/
```

---

## 🤔 생각해볼 문제

**Q1.** `application.yml`과 `application.properties`가 동일 키를 가질 때 어느 것이 우선하는가? 이 동작에 의존하는 것이 왜 위험한가?

**Q2.** 멀티 문서 YAML에서 첫 번째 문서에 `spring.config.activate.on-profile`을 지정하면 어떻게 되는가?

**Q3.** `spring.config.import: optional:file:/etc/app/config.yml`에서 `optional:`을 제거하면 파일이 없을 때 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** 같은 `classpath:` 위치에서는 yml이 이긴다. properties가 먼저 로딩되고 yml이 나중에 로딩되어 동일 키를 덮어쓰기 때문이다. 하지만 이 동작에 의존하면 위험하다. 파일 로딩 순서는 Spring Boot 버전이나 설정 변경에 따라 달라질 수 있고, 두 파일을 동시에 유지하면 어느 값이 실제로 사용되는지 혼란을 준다. 의도적으로 오버라이드가 필요하다면 Profile별 파일(`application-override.properties`)이나 `spring.config.import`를 사용하는 것이 명확하다.
>
> **Q2.** 첫 번째 문서에 `spring.config.activate.on-profile`을 지정하면 해당 프로파일이 활성화되지 않은 환경에서는 첫 번째 문서도 스킵된다. 멀티 문서 YAML에서 프로파일 없는 기본값은 반드시 첫 번째 문서에 조건 없이 배치해야 한다.
>
> **Q3.** `optional:` 없이 파일이 존재하지 않으면 `ConfigDataLocationNotFoundException`이 발생하여 애플리케이션 시작이 실패한다. 운영 환경에서 반드시 존재해야 하는 설정 파일은 `optional:` 없이 지정해 파일 누락을 빠르게 감지할 수 있다.

---

<div align="center">

**[⬅️ Chapter 2 — autoconfigure-processor](../auto-configuration/08-autoconfigure-processor.md)** | **[홈으로 🏠](../README.md)** | **[다음: @ConfigurationProperties 바인딩 메커니즘 ➡️](./02-configuration-properties-binding.md)**

</div>
