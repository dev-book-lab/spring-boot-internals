# Relaxed Binding — 표기법이 달라도 같은 프로퍼티로 인식하는 원리

---

## 🎯 핵심 질문

- `kebab-case`, `camelCase`, `SCREAMING_SNAKE_CASE`를 같은 키로 처리하는 알고리즘은?
- `ConfigurationPropertyName`의 정규화 과정은?
- `@Value`에서는 Relaxed Binding이 왜 동작하지 않는가?
- 환경변수(`SPRING_DATASOURCE_URL`)가 `spring.datasource.url`로 매핑되는 경로는?
- Relaxed Binding이 실패하는 경우는?

---

## 🔍 왜 이게 존재하는가

```
설정 소스마다 관행이 다르다:

application.yml:      spring.datasource.max-pool-size=10  (kebab-case)
Java 코드:            maxPoolSize (camelCase)
환경변수:             SPRING_DATASOURCE_MAX_POOL_SIZE (SCREAMING_SNAKE_CASE)
시스템 프로퍼티:       spring.datasource.maxPoolSize (mixed)

→ 각 소스를 별도 키로 관리하면 중복·불일치 발생
→ Relaxed Binding: 모두 같은 키로 정규화해서 통합
```

---

## 🔬 내부 동작 원리

### 1. ConfigurationPropertyName — 정규화 핵심

```java
// ConfigurationPropertyName.java
// 모든 키를 "요소(element) 배열"로 파싱

// 정규화 규칙:
// 1. 소문자화
// 2. 하이픈(-) 구분자 통일 (_, ., 대소문자 경계 → 요소 구분)
// 3. 숫자 인덱스 추출 [0], [1] 등

// 입력 → 정규화 결과:
// "spring.datasource.max-pool-size"   → ["spring", "datasource", "max-pool-size"]
// "spring.datasource.maxPoolSize"     → ["spring", "datasource", "max-pool-size"]
// "SPRING_DATASOURCE_MAX_POOL_SIZE"   → ["spring", "datasource", "max-pool-size"]
// "spring.datasource.max_pool_size"   → ["spring", "datasource", "max-pool-size"]

ConfigurationPropertyName name1 = ConfigurationPropertyName.of("spring.datasource.max-pool-size");
ConfigurationPropertyName name2 = ConfigurationPropertyName.of("spring.datasource.maxPoolSize");

name1.equals(name2);  // → false! (equals는 정확히 같을 때만 true)
name1.isAncestorOf(name2); // 계층 비교는 별도 메서드

// 실제 매핑은 ConfigurationPropertyNameAliases 또는
// SpringConfigurationPropertySource의 containsDescendantOf로 처리
```

```java
// ConfigurationPropertyName 내부 파싱
private static Elements parseElements(String name, ...) {
    // camelCase → 대소문자 경계에서 분리
    // "maxPoolSize" → "max", "pool", "size"
    // → 다시 합쳐서 "max-pool-size"
    
    // 세부 파싱 로직
    for (int i = 0; i < name.length(); i++) {
        char c = name.charAt(i);
        if (Character.isUpperCase(c)) {
            // 대문자 직전에 '-' 삽입 효과
            // "maxPool" → "max-pool"
        }
    }
}
```

### 2. SpringConfigurationPropertySource — 환경변수 처리

```java
// 환경변수는 SystemEnvironmentPropertySource로 관리됨
// 이것을 ConfigurationPropertySource로 래핑 시 키 변환 처리

// SystemEnvironmentPropertySource:
//   "SPRING_DATASOURCE_URL" → "spring.datasource.url" 변환 제공
//   내부에서 '.' '_' 혼용 처리

// SystemEnvironmentPropertySourceAdapter (ConfigDataEnvironment에서)
// SCREAMING_SNAKE 규칙:
//   대문자 + 언더스코어 → 소문자 + 하이픈/점
//   "MY_APP_MAX_POOL_SIZE" → "my.app.max-pool-size"
//     또는 → "my.app.max_pool_size"
//   Binder가 Relaxed Binding으로 둘 다 시도
```

### 3. Relaxed Binding 매핑 테이블

```
@ConfigurationProperties(prefix = "my.app")
필드: private int maxPoolSize;

인식되는 키:
  my.app.max-pool-size      ← 권장 (kebab-case)
  my.app.maxPoolSize        ← camelCase
  my.app.max_pool_size      ← 언더스코어
  MY_APP_MAX_POOL_SIZE      ← 환경변수 (SCREAMING_SNAKE)
  my.app.MAX-POOL-SIZE      ← 대문자 kebab (비권장이나 인식됨)

@Value에서는 Relaxed Binding 없음:
  @Value("${my.app.max-pool-size}")  ← 정확한 키만 동작
  @Value("${my.app.maxPoolSize}")    ← 다른 키로 인식, 다른 값 or 오류
```

### 4. Relaxed Binding이 실패하는 경우

```java
// ① Map 키: Relaxed Binding 미적용
@ConfigurationProperties(prefix = "my")
public class MyProps {
    private Map<String, String> config = new HashMap<>();
}

// my.config.some-key=value  → map.get("some-key")  ✅
// my.config.someKey=value   → map.get("someKey")   ✅ (다른 키로 저장!)
// → Map 키는 정규화하지 않음 → 대소문자/형식 그대로 저장

// ② @Value: Relaxed Binding 없음
@Value("${my.max-pool-size}")   // "my.max-pool-size" 키만 탐색
// MY_MAX_POOL_SIZE 환경변수 → @Value로 읽으려면 정확히 일치해야 함
// (단, SystemEnvironmentPropertySource가 '_' → '.' 변환을 일부 처리하긴 함)

// ③ 접두사 mismatch
// @ConfigurationProperties(prefix = "myApp")  ← camelCase prefix
// → "my.app.*" 키를 못 읽을 수 있음
// → prefix는 항상 kebab-case로 작성 권장
//    @ConfigurationProperties(prefix = "my-app")  ← 올바른 방식
```

### 5. 환경변수 바인딩 실전 패턴

```bash
# Docker/Kubernetes 환경변수로 설정 오버라이드
# application.yml: spring.datasource.url=jdbc:h2:mem:test

# 컨테이너 실행 시 환경변수로 오버라이드
SPRING_DATASOURCE_URL=jdbc:mysql://prod-db:3306/mydb
SPRING_DATASOURCE_PASSWORD=secretpassword
SPRING_JPA_HIBERNATE_DDL_AUTO=validate

# 점(.) → 언더스코어(_) 변환
# 하이픈(-) → 언더스코어(_) 변환
# 소문자 → 대문자 변환
# → Relaxed Binding으로 올바르게 매핑됨
```

---

## 💻 실험으로 확인하기

### 실험 1: 동일 키 여러 표기법 테스트

```java
@SpringBootTest(properties = {
    "my.max-pool-size=20"
})
class RelaxedBindingTest {
    @Autowired MyProperties props;

    @Test
    void kebabCaseBinds() {
        assertThat(props.getMaxPoolSize()).isEqualTo(20);
    }
}

@SpringBootTest(properties = {
    "MY_MAX_POOL_SIZE=30"  // 환경변수 스타일
})
class EnvVarBindingTest {
    @Test
    void screamingSnakeCaseBinds() {
        assertThat(props.getMaxPoolSize()).isEqualTo(30);
    }
}
```

### 실험 2: Map 키 Relaxed Binding 없음 확인

```yaml
my:
  config:
    feature-a: enabled
    featureB: disabled
```

```java
Map<String, String> config = props.getConfig();
config.get("feature-a");  // "enabled" ← 키 그대로
config.get("featureB");   // "disabled" ← 키 그대로 (정규화 없음)
config.get("feature-b");  // null ← "featureB"와 다른 키
```

---

## ⚙️ 설정 최적화 팁

```yaml
# 권장: 항상 kebab-case 사용
# → 모든 소스에서 명확하게 인식
# → 환경변수: UNDERSCORE 변환만 하면 됨

# properties prefix도 kebab-case
@ConfigurationProperties(prefix = "my-service")
# 환경변수: MY_SERVICE_HOST=...
# yml:      my-service.host=...
```

---

## 📌 핵심 정리

```
Relaxed Binding 대상
  @ConfigurationProperties → Binder가 처리 → Relaxed 적용
  @Value → SpEL/PropertyPlaceholder → 정확한 키만 인식

정규화 알고리즘 (ConfigurationPropertyName)
  camelCase → kebab-case (대소문자 경계 → 하이픈)
  SCREAMING_SNAKE → kebab-case (대문자+언더스코어 → 소문자+점/하이픈)
  → 비교 시 정규화된 형태로 일치 확인

Relaxed Binding 미적용 케이스
  Map 키 (그대로 저장)
  @Value (정확 매칭)
  @ConfigurationProperties prefix (kebab-case 권장)
```

---

## 🤔 생각해볼 문제

**Q1.** `@ConfigurationProperties(prefix = "myApp")`과 `@ConfigurationProperties(prefix = "my-app")`의 차이는 무엇인가?

**Q2.** 환경변수 `SPRING_APPLICATION_NAME=my-service`와 `application.yml`의 `spring.application.name: app`이 동시에 있을 때 어느 것이 사용되는가?

**Q3.** `Map<String, String>` 필드에 환경변수를 바인딩할 때 키 정규화가 일어나지 않는 것이 설계 의도인 이유는 무엇인가?

> 💡 **해설**
>
> **Q1.** `prefix = "myApp"` (camelCase)는 실제로 `my.app` 또는 `myapp`으로 정규화될 수 있어 예상치 못한 동작을 한다. Spring Boot 공식 문서는 prefix를 반드시 kebab-case(`my-app`)나 소문자 점 표기법(`my.app`)으로 작성하도록 권장한다. `my-app`을 사용하면 `my.app.*`, `MY_APP_*` 환경변수 모두 올바르게 매핑된다.
>
> **Q2.** 환경변수가 이긴다. PropertySource 우선순위에서 시스템 환경변수(`systemEnvironment`)는 `application.yml`로 로딩된 PropertySource보다 높은 우선순위를 가진다 (17단계 중 상위). 따라서 `SPRING_APPLICATION_NAME=my-service`가 `spring.application.name: app`을 덮어쓴다.
>
> **Q3.** Map의 키는 사용자 정의 키로서 임의의 문자열이 가능하다. 정규화하면 `feature-a`와 `featureA`가 같은 키가 되어 둘 중 어느 것이 저장될지 알 수 없고 원래 키 정보가 손실된다. Map의 값에 접근할 때 코드에서 어떤 형태로 키를 사용하는지 모르므로 그대로 보존하는 것이 올바른 설계다. 원한다면 TreeMap에 case-insensitive comparator를 사용해 대소문자 무관 접근을 직접 구현할 수 있다.

---

<div align="center">

**[⬅️ 이전: @ConfigurationProperties 바인딩](./02-configuration-properties-binding.md)** | **[홈으로 🏠](../README.md)** | **[다음: Type-safe Configuration Properties ➡️](./04-type-safe-configuration.md)**

</div>
