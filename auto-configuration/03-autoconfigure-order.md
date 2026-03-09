# @AutoConfigureAfter/@AutoConfigureBefore 순서 제어 — 위상 정렬로 결정되는 Auto-configuration 순서

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Auto-configuration 클래스들의 처리 순서는 어떻게 결정되는가?
- `@AutoConfigureAfter`와 `@AutoConfigureBefore`는 내부에서 어떻게 처리되는가?
- `AutoConfigurationSorter`가 사용하는 위상 정렬(topological sort) 알고리즘은?
- `@AutoConfigureOrder`는 `@AutoConfigureAfter`와 어떻게 다른가?
- 순환 의존이 발생하면 어떻게 되는가?
- Boot 3.x `@AutoConfiguration`의 `before/after` 속성은 어떻게 다른가?

---

## 🔍 왜 이게 존재하는가

### 문제: Auto-configuration 간 의존 관계

```java
// HibernateJpaAutoConfiguration은 DataSource가 먼저 설정돼야 한다
@AutoConfiguration
public class HibernateJpaAutoConfiguration {

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource, ...) {  // DataSource 필요!
        ...
    }
}
```

```
문제:
  .imports 파일은 순서가 없는 목록
  → HibernateJpaAutoConfiguration이 DataSourceAutoConfiguration보다
     먼저 처리되면 DataSource BeanDefinition이 아직 없음

  @ConditionalOnBean(DataSource.class)를 붙여도:
    처리 순서가 불확정이면 조건 평가 시 DataSource가 없을 수 있음
    → 올바른 동작 보장 불가

해결:
  @AutoConfigureAfter(DataSourceAutoConfiguration.class)
  → "나는 DataSourceAutoConfiguration 이후에 처리되어야 한다"
  → AutoConfigurationSorter가 이 관계를 수집해 순서를 결정
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @AutoConfigureAfter가 즉시 실행 순서를 강제한다

```
❌ 잘못된 이해:
  "@AutoConfigureAfter를 붙이면 A가 완전히 처리된 후 B가 시작된다"

✅ 실제:
  @AutoConfigureAfter는 BeanDefinition 등록 순서만 제어
  → A의 @Bean 메서드가 BeanDefinition으로 등록된 후 B가 처리됨
  → A의 Bean 인스턴스가 생성되는 것이 아님 (인스턴스 생성은 refresh() 후반부)

  실제 효과:
    B의 @ConditionalOnBean(DataSource.class) 평가 시
    A에서 등록한 DataSource BeanDefinition이 존재함
    → 조건 평가 정확성 보장
```

### Before: @Order로도 Auto-configuration 순서를 제어할 수 있다

```java
// ❌ 잘못된 시도
@AutoConfiguration
@Order(1)  // 우선순위 1번으로 먼저 처리?
public class MyFirstAutoConfiguration { }

// ✅ 실제:
// @Order는 Auto-configuration 처리 순서에 영향 없음
// Auto-configuration 순서 제어:
//   @AutoConfigureOrder → 같은 계층에서 상대적 순서
//   @AutoConfigureAfter → 특정 클래스 이후 처리
//   @AutoConfigureBefore → 특정 클래스 이전 처리
```

---

## ✨ 올바른 이해와 사용

### After: 세 가지 순서 제어 방법

```
1. @AutoConfigureAfter(A.class)
   "나는 A 이후에 처리되어야 한다"
   A → 나 (A가 나보다 먼저)

2. @AutoConfigureBefore(B.class)
   "나는 B 이전에 처리되어야 한다"
   나 → B (내가 B보다 먼저)

3. @AutoConfigureOrder(int)
   의존 관계 없는 클래스들 간 상대적 우선순위
   낮은 값 = 먼저 처리 (Ordered.HIGHEST_PRECEDENCE에 가까울수록 먼저)

Boot 3.x @AutoConfiguration 통합:
  @AutoConfiguration(
    after = DataSourceAutoConfiguration.class,
    before = JpaRepositoriesAutoConfiguration.class
  )
  = @AutoConfigureAfter(DataSourceAutoConfiguration.class)
    + @AutoConfigureBefore(JpaRepositoriesAutoConfiguration.class)
```

---

## 🔬 내부 동작 원리

### 1. AutoConfigurationSorter — 위상 정렬

```java
// AutoConfigurationSorter.java
class AutoConfigurationSorter {

    private final AutoConfigurationMetadata autoConfigurationMetadata;

    // 전체 정렬 진입점
    List<String> getInPriorityOrder(Collection<String> classNames) {

        AutoConfigurationClasses classes =
            new AutoConfigurationClasses(this.metadataReaderFactory,
                                          this.autoConfigurationMetadata,
                                          classNames);

        List<String> orderedClassNames = new ArrayList<>(classNames);

        // 1단계: @AutoConfigureOrder 기준 정렬 (기본 순서)
        orderedClassNames.sort((a, b) ->
            classes.get(a).getOrder() - classes.get(b).getOrder());

        // 2단계: after/before 관계 기반 위상 정렬 (순서 보정)
        orderedClassNames = sortByAnnotation(classes, orderedClassNames);

        return orderedClassNames;
    }

    // sortByAnnotation() — 위상 정렬 핵심
    private List<String> sortByAnnotation(AutoConfigurationClasses classes,
                                           List<String> classNames) {
        List<String> toSort = new ArrayList<>(classNames);
        List<String> sorted = new ArrayList<>();
        Set<String> processing = new LinkedHashSet<>();

        while (!toSort.isEmpty()) {
            doSortByAfterAnnotation(classes, toSort, sorted, processing, null);
        }
        return sorted;
    }

    private void doSortByAfterAnnotation(AutoConfigurationClasses classes,
                                          List<String> toSort,
                                          List<String> sorted,
                                          Set<String> processing,
                                          String current) {

        if (current == null) {
            current = toSort.remove(0);
        }

        // 순환 참조 감지
        processing.add(current);

        // @AutoConfigureAfter로 선언된 "나보다 먼저 와야 할 것들" 처리
        for (String after : classes.getClassesRequestedAfter(current)) {
            // after가 아직 sorted에 없으면 먼저 처리
            if (!sorted.contains(after)) {
                if (toSort.contains(after)) {
                    // after를 재귀적으로 먼저 처리
                    doSortByAfterAnnotation(classes, toSort, sorted, processing, after);
                }
                // after가 toSort에 없으면 후보 밖 → 무시
            }
        }

        // 순환 참조
        if (processing.contains(current) && sorted.contains(current)) {
            // 이미 처리됨 → 건너뜀
        } else {
            sorted.add(current);
        }
        processing.remove(current);
    }
}
```

### 2. AutoConfigurationClasses — 순서 메타데이터 수집

```java
// AutoConfigurationClasses 내부
private static class AutoConfigurationClasses {

    private final Map<String, AutoConfigurationClass> classes = new HashMap<>();

    AutoConfigurationClasses(MetadataReaderFactory metadataReaderFactory,
                               AutoConfigurationMetadata autoConfigurationMetadata,
                               Collection<String> classNames) {
        for (String className : classNames) {
            this.classes.put(className, new AutoConfigurationClass(
                className, metadataReaderFactory, autoConfigurationMetadata));
        }
    }

    // after 관계 수집 — A가 "B 이후에 처리되어야" 하는 B들의 목록
    Set<String> getClassesRequestedAfter(String className) {
        Set<String> requestedAfter = new LinkedHashSet<>();

        // 방법 1: AutoConfigurationMetadata (컴파일 타임 메타데이터)
        requestedAfter.addAll(get(className).getAfter());

        // 방법 2: 실제 @AutoConfigureAfter 어노테이션 읽기
        // (메타데이터에 없는 경우 폴백)
        return requestedAfter;
    }
}

// AutoConfigurationClass.getAfter()
Set<String> getAfter() {
    // 우선 AutoConfigurationMetadata에서 조회 (빠름)
    Set<String> after = getFromMetadata("AutoConfigureAfter");

    // 없으면 어노테이션 직접 읽기
    if (after == null) {
        after = getAnnotationValue(AutoConfigureAfter.class);
    }
    return after;
}
```

```
AutoConfigurationMetadata에서 순서 정보 예시:
  (spring-autoconfigure-metadata.properties)

org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration.AutoConfigureAfter=\
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
  org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration

org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration.AutoConfigureAfter=\
  org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration

→ 정렬 결과:
  DataSourceAutoConfiguration
  DataSourceTransactionManagerAutoConfiguration
  HibernateJpaAutoConfiguration
  JpaRepositoriesAutoConfiguration
```

### 3. 위상 정렬 예시 — 단계별 추적

```
예시 Auto-configuration 그래프:

A → B → D
A → C → D

(A가 B보다 먼저, A가 C보다 먼저, B와 C가 D보다 먼저)

초기: toSort = [D, B, C, A] (임의 순서)
sorted = []

1. current = D (dequeue)
   D.getAfter() = {B, C}
   → B가 sorted에 없음 → 재귀: current = B
     B.getAfter() = {A}
     → A가 sorted에 없음 → 재귀: current = A
       A.getAfter() = {} → sorted.add(A) = [A]
     → sorted.add(B) = [A, B]
   → C가 sorted에 없음 → 재귀: current = C
     C.getAfter() = {A}
     → A가 이미 sorted에 있음 → 건너뜀
     → sorted.add(C) = [A, B, C]
   → sorted.add(D) = [A, B, C, D]

최종 결과: [A, B, C, D] ✅
```

### 4. 순환 의존 감지

```java
// 순환 의존 예시
// A는 B 이후, B는 A 이후 → 순환!

@AutoConfiguration(after = B.class)
class A { }

@AutoConfiguration(after = A.class)
class B { }
```

```
순환 의존 처리:

AutoConfigurationSorter의 doSortByAfterAnnotation()에서
processing Set으로 현재 처리 중인 클래스 추적

A 처리 시작: processing = {A}
  A.after = {B} → B 재귀 처리
  B 처리 시작: processing = {A, B}
    B.after = {A} → A가 processing에 있음 → 순환!
    → A는 이미 처리 중 → 재귀 중단

실제 Spring Boot의 처리:
  순환 의존이 있어도 예외를 던지지 않고
  감지 가능한 범위에서 최선의 순서로 정렬
  → 순환 있는 경우 동작은 정의되지 않음 (사용 자제)
```

### 5. @AutoConfigureOrder — 상대적 우선순위

```java
// @AutoConfigureOrder — 의존 관계 없는 클래스들 간 순서
@AutoConfiguration
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)  // 가장 먼저
public class EarlyAutoConfiguration { }

@AutoConfiguration
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)   // 가장 나중
public class LateAutoConfiguration { }

// 기본값: Ordered.LOWEST_PRECEDENCE (마지막)
// @AutoConfigureAfter/Before와의 관계:
//   @AutoConfigureOrder: 1단계 정렬 (상대적 순서)
//   @AutoConfigureAfter/Before: 2단계 정렬 (after/before 관계 강제 반영)
//   → 2단계가 1단계를 덮어씀 (after/before가 더 강한 제약)
```

---

## 💻 실험으로 확인하기

### 실험 1: 순서 의존 @ConditionalOnBean 안전하게 사용

```java
// DataSource 등록 후 처리 보장
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
@ConditionalOnBean(DataSource.class)  // 안전: after 덕분에 DataSource 확보됨
public class MyDataAwareAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyDataService myDataService(DataSource dataSource) {
        return new MyDataService(dataSource);
    }
}
```

### 실험 2: 정렬 결과 로그로 확인

```bash
# Auto-configuration 처리 순서 확인 (TRACE 레벨)
logging:
  level:
    org.springframework.boot.autoconfigure: TRACE
```

```
TRACE 출력 예시:
Loaded auto-configuration classes:
  ...DataSourceAutoConfiguration
  ...DataSourceTransactionManagerAutoConfiguration
  ...HibernateJpaAutoConfiguration  ← DataSource 이후
  ...JpaRepositoriesAutoConfiguration  ← Hibernate 이후
```

### 실험 3: 커스텀 Auto-configuration 순서 테스트

```java
// 테스트로 순서 검증
@SpringBootTest
class AutoConfigOrderTest {

    @Autowired
    private ApplicationContext context;

    @Test
    void configurationOrderIsCorrect() {
        // 순서 의존 Bean이 정상 생성되는지 확인
        assertThat(context.getBean(MyDataService.class)).isNotNull();
        assertThat(context.getBean(DataSource.class)).isNotNull();
    }
}
```

---

## ⚙️ 설정 최적화 팁

```java
// 1. after/before 명시 → @ConditionalOnBean 정확성 보장
@AutoConfiguration(
    after = DataSourceAutoConfiguration.class   // DataSource 확보 후
)
@ConditionalOnBean(DataSource.class)           // 안전한 조건

// 2. spring-autoconfigure-metadata.properties에 순서 정보 포함
//    → spring-boot-autoconfigure-processor가 자동 생성
//    → 런타임에 어노테이션 읽기 없이 메타데이터로 빠른 정렬

// 3. @AutoConfigureOrder 사용 시 명확한 이유 주석 달기
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
// 이유: PropertySource 설정이 다른 Auto-configuration보다 먼저 필요
```

---

## 🤔 트레이드오프

```
@AutoConfigureAfter vs @AutoConfigureBefore:
  의미적으로 동일 (A after B = B before A)
  관행: 자신이 의존하는 쪽에 after 사용
       "나는 DataSource 이후" → @AutoConfigureAfter(DataSource..)
  → 코드 읽기 자연스러움

명시적 순서 vs 암묵적 순서:
  명시적 (@AutoConfigureAfter)  의존 관계 명확, 실수 방지
  암묵적 (정의 순서)             유지 보수 어려움, 순서 변경 시 동작 불확정

성능:
  위상 정렬은 O(V + E) (V=Auto-configuration 수, E=의존 관계 수)
  → 150개 클래스에서 매우 빠름 (< 1ms)
  → 런타임보다 컴파일 타임 메타데이터로 처리 시 더 빠름
```

---

## 📌 핵심 정리

```
순서 제어 3가지
  @AutoConfigureAfter(A.class)  나는 A 이후에 처리
  @AutoConfigureBefore(B.class) 나는 B 이전에 처리
  @AutoConfigureOrder(int)      의존 없는 클래스 간 상대 순위

정렬 알고리즘
  AutoConfigurationSorter
  1단계: @AutoConfigureOrder 기준 정렬
  2단계: @AutoConfigureAfter/Before 위상 정렬 (DFS 기반)
  → 2단계가 1단계 덮어씀

정렬 데이터 소스
  우선: AutoConfigurationMetadata (컴파일 타임 생성, 빠름)
  폴백: @AutoConfigureAfter 어노테이션 직접 읽기

순환 의존
  예외 없이 최선으로 정렬 (동작 정의되지 않음)
  → 순환 관계 설계 자체를 피해야 함

Boot 3.x @AutoConfiguration
  before/after 속성으로 통합
  = @AutoConfigureAfter + @AutoConfigureBefore 조합
```

---

## 🤔 생각해볼 문제

**Q1.** `@AutoConfigureAfter`에 지정한 클래스가 실제 classpath에 없거나 Auto-configuration 대상이 아니면 어떻게 되는가?

**Q2.** `@AutoConfigureAfter(DataSourceAutoConfiguration.class)`와 `@ConditionalOnBean(DataSource.class)`를 함께 사용하는 이유는 무엇인가? 하나만 있어도 충분하지 않은가?

**Q3.** Auto-configuration 클래스 A, B, C가 있고 서로 아무런 after/before 관계가 없을 때 처리 순서는 어떻게 결정되는가?

> 💡 **해설**
>
> **Q1.** `@AutoConfigureAfter`에 지정한 클래스가 Auto-configuration 후보 목록에 없으면(classpath에 없거나 @ConditionalOnClass 등으로 이미 제외됐으면) 해당 after 관계는 단순히 무시된다. `AutoConfigurationClasses.getClassesRequestedAfter()`가 반환한 클래스가 `toSort` 목록에 없으면 재귀 처리 없이 건너뛴다. 따라서 `@AutoConfigureAfter(HibernateJpaAutoConfiguration.class)`를 붙여도 JPA 없는 프로젝트에서는 아무 효과 없이 정상 동작한다.
>
> **Q2.** 두 역할이 다르다. `@AutoConfigureAfter`는 처리 순서를 보장하지만, `DataSourceAutoConfiguration`이 `@ConditionalOnClass` 등의 이유로 스킵될 경우 DataSource Bean이 등록되지 않을 수 있다. 이때 `@ConditionalOnBean(DataSource.class)`가 없으면 DataSource Bean 없이 나의 Auto-configuration이 처리되어 `DataSource` 파라미터를 주입받을 수 없어 시작 실패가 된다. `@ConditionalOnBean(DataSource.class)`를 함께 쓰면 DataSource가 없는 환경에서 나의 Auto-configuration 전체가 안전하게 스킵된다. 즉 after는 순서 보장, ConditionalOnBean은 존재 조건 보장으로 역할이 다르다.
>
> **Q3.** `@AutoConfigureOrder` 기준으로 정렬된다. 기본값은 `Ordered.LOWEST_PRECEDENCE`이므로 동일한 Order 값을 가진 클래스들끼리는 `AutoConfigurationSorter`가 받은 초기 목록 순서(`.imports` 파일의 줄 순서)가 그대로 유지된다. 실용적으로는 after/before 관계 없는 클래스들의 순서는 보장되지 않으므로, 순서에 의존하는 로직은 반드시 after/before 또는 @ConditionalOnBean으로 명시해야 한다.

---

<div align="center">

**[⬅️ 이전: @ConditionalOnBean vs @ConditionalOnMissingBean](./02-conditional-on-bean-missing.md)** | **[홈으로 🏠](../README.md)** | **[다음: DataSource Auto-configuration 분석 ➡️](./04-datasource-autoconfiguration.md)**

</div>
