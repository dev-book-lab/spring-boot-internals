# spring-boot-autoconfigure-processor 역할 — 컴파일 타임 최적화의 핵심

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `spring-boot-autoconfigure-processor`는 컴파일 타임에 무엇을 생성하는가?
- `AutoConfigurationMetadata`가 1차 필터링에서 어떻게 시작 성능을 향상시키는가?
- APT(Annotation Processing Tool)가 어떻게 동작하는가?
- 생성되는 `spring-autoconfigure-metadata.properties` 파일의 구조와 내용은?
- 커스텀 Auto-configuration을 만들 때 이 프로세서를 추가하지 않으면 어떤 차이가 생기는가?
- `spring-boot-configuration-processor`와의 차이는?

---

## 🔍 왜 이게 존재하는가

### 문제: 런타임에 @ConditionalOnClass 어노테이션을 읽는 비용

```
Auto-configuration 처리 없이 런타임 평가 시:

1. .imports 파일에서 150개 클래스명 로딩
2. 각 클래스를 하나씩 처리하기 위해 클래스 로딩 시도
3. @ConditionalOnClass 어노테이션 읽기 (어노테이션 파싱)
4. 어노테이션 value의 클래스 존재 확인
5. 클래스 없으면 제외, 있으면 계속

문제:
  - 150개 클래스 × (클래스 로딩 + 어노테이션 파싱) = 큰 오버헤드
  - 결국 대부분이 @ConditionalOnClass로 제외되는데
    클래스를 로딩하고 파싱한 후에야 제외 결정
  - 시작 시간 불필요하게 증가
```

```
AutoConfigurationMetadata 사전 생성 후:

1. spring-autoconfigure-metadata.properties 파일 읽기 (단순 텍스트 I/O)
2. 클래스명 문자열 → ClassLoader.getResource() → 파일 존재만 확인
3. 없으면 즉시 제외 (클래스 로딩 없음)

결과:
  - 150개 중 대부분 클래스 로딩 없이 제외됨
  - 실제로 클래스 로딩/파싱이 일어나는 것은 소수
  - 시작 시간 단축 (수십~수백 ms)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: spring-boot-autoconfigure-processor와 spring-boot-configuration-processor는 같다

```
❌ 잘못된 이해:
  "두 processor가 같은 역할을 한다"

✅ 차이:

spring-boot-autoconfigure-processor:
  목적: Auto-configuration 클래스의 조건 어노테이션 메타데이터 추출
  입력: @AutoConfiguration, @ConditionalOnClass, @ConditionalOnBean,
        @AutoConfigureAfter, @AutoConfigureBefore, @AutoConfigureOrder
  출력: META-INF/spring-autoconfigure-metadata.properties
  사용: OnClassCondition 1차 필터링, AutoConfigurationSorter 순서 정렬

spring-boot-configuration-processor:
  목적: @ConfigurationProperties 클래스의 메타데이터 추출
  입력: @ConfigurationProperties, @ConfigurationPropertiesScan
  출력: META-INF/spring-configuration-metadata.json
  사용: IDE 자동완성, 프로퍼티 문서화
```

### Before: 이 processor가 없으면 Auto-configuration이 동작하지 않는다

```
❌ 잘못된 이해:
  "processor 없으면 Auto-configuration 자체가 실패한다"

✅ 실제:
  processor는 선택사항 (optional)
  없으면:
    1차 필터링이 spring-autoconfigure-metadata.properties 없이 진행
    → OnClassCondition이 직접 @ConditionalOnClass 어노테이션 읽기
    → 클래스 로딩 비용 발생
    → 시작 시간 증가 (수십 ms ~ 수백 ms)
    → 동작 자체는 정상

  결론:
    동작 정확성: 차이 없음
    시작 성능: processor 있으면 빠름 (특히 Auto-configuration 많은 환경)
```

---

## ✨ 올바른 이해와 사용

### After: APT 동작 흐름

```
컴파일 시간:

javac 실행
  → spring-boot-autoconfigure-processor가 APT로 실행
  → @AutoConfiguration 어노테이션이 붙은 클래스 탐색
  → 각 클래스의 @ConditionalOnClass, @AutoConfigureAfter 등 수집
  → META-INF/spring-autoconfigure-metadata.properties 파일 생성

런타임:

spring-boot 시작
  → AutoConfigurationImportSelector.getAutoConfigurationEntry()
  → AutoConfigurationMetadataLoader.loadMetadata()
    → META-INF/spring-autoconfigure-metadata.properties 읽기
  → OnClassCondition.getOutcomes() → 1차 필터링
    → properties에서 ConditionalOnClass 조건 조회
    → ClassLoader.getResource()로 파일 존재만 확인
    → 클래스 없으면 즉시 제외 (클래스 로딩 없음)
```

---

## 🔬 내부 동작 원리

### 1. AutoConfigureAnnotationProcessor — APT 구현체

```java
// spring-boot-autoconfigure-processor 모듈
// AutoConfigureAnnotationProcessor.java

@SupportedAnnotationTypes({
    "org.springframework.boot.autoconfigure.AutoConfiguration",
    // Boot 2.x 호환
    "org.springframework.boot.autoconfigure.EnableAutoConfiguration",
    // 조건 어노테이션
    "org.springframework.boot.autoconfigure.condition.ConditionalOnClass",
    "org.springframework.boot.autoconfigure.condition.ConditionalOnBean",
    "org.springframework.boot.autoconfigure.condition.ConditionalOnMissingClass",
    "org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean",
    // 순서 어노테이션
    "org.springframework.boot.autoconfigure.AutoConfigureAfter",
    "org.springframework.boot.autoconfigure.AutoConfigureBefore",
    "org.springframework.boot.autoconfigure.AutoConfigureOrder"
})
public class AutoConfigureAnnotationProcessor extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations,
                            RoundEnvironment roundEnv) {
        for (TypeElement annotation : annotations) {
            // @AutoConfiguration 붙은 클래스들 처리
            if (isAutoConfigurationAnnotation(annotation)) {
                processAutoConfiguration(roundEnv.getElementsAnnotatedWith(annotation));
            }
        }
        return false;
    }

    private void processAutoConfiguration(
            Set<? extends Element> autoConfigurationElements) {

        Properties properties = new Properties();

        for (Element element : autoConfigurationElements) {
            // 각 @AutoConfiguration 클래스에서 메타데이터 추출
            String className = ((TypeElement) element).getQualifiedName().toString();

            // 존재 마커 (처리됨 표시)
            properties.put(className, "");

            // @ConditionalOnClass 처리
            extractConditionalOnClass(element, className, properties);

            // @AutoConfigureAfter 처리
            extractAutoConfigureAfter(element, className, properties);

            // @AutoConfigureBefore 처리
            extractAutoConfigureBefore(element, className, properties);

            // @AutoConfigureOrder 처리
            extractAutoConfigureOrder(element, className, properties);
        }

        // META-INF/spring-autoconfigure-metadata.properties 파일 쓰기
        writeProperties(properties);
    }
}
```

### 2. 생성되는 메타데이터 파일 형식

```properties
# META-INF/spring-autoconfigure-metadata.properties (실제 파일 내용 일부)

# 존재 마커 (처리됨)
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration=
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration=

# @ConditionalOnClass 정보
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.ConditionalOnClass=\
  javax.sql.DataSource,\
  org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType

org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration.ConditionalOnClass=\
  jakarta.persistence.EntityManager,\
  org.hibernate.engine.spi.SessionImplementor,\
  org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean

# @AutoConfigureAfter 정보
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration.AutoConfigureAfter=\
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
  org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration

# @AutoConfigureBefore 정보
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.AutoConfigureBefore=\
  org.springframework.boot.autoconfigure.sql.init.SqlInitializationAutoConfiguration

# @AutoConfigureOrder 정보
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration.AutoConfigureOrder=\
  -2147483638
```

### 3. AutoConfigurationMetadataLoader — 메타데이터 로딩

```java
// AutoConfigurationMetadataLoader.java
final class AutoConfigurationMetadataLoader {

    protected static final String PATH =
        "META-INF/spring-autoconfigure-metadata.properties";

    static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader) {
        return loadMetadata(classLoader, PATH);
    }

    static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader, String path) {
        Enumeration<URL> urls = (classLoader != null)
            ? classLoader.getResources(path)
            : ClassLoader.getSystemResources(path);

        Properties properties = new Properties();
        while (urls.hasMoreElements()) {
            properties.putAll(PropertiesLoaderUtils.loadProperties(
                new UrlResource(urls.nextElement())));
        }
        return new PropertiesAutoConfigurationMetadata(properties);
    }
}

// PropertiesAutoConfigurationMetadata — 메타데이터 조회 구현
private static class PropertiesAutoConfigurationMetadata
        implements AutoConfigurationMetadata {

    private final Properties properties;

    @Override
    public boolean wasProcessed(String className) {
        // className= 키가 있으면 processor가 처리한 것
        return this.properties.containsKey(className);
    }

    @Override
    public Set<String> getSet(String className, String key) {
        // "ClassName.ConditionalOnClass" 키로 조회
        String value = this.properties.getProperty(className + "." + key);
        if (value == null) {
            return null;  // 메타데이터 없음 → 런타임 어노테이션으로 폴백
        }
        return StringUtils.commaDelimitedListToSet(value);
    }
}
```

### 4. 1차 필터링에서의 실제 활용

```java
// OnClassCondition.StandardOutcomesResolver.getOutcome()
private ConditionOutcome getOutcome(String candidates,
                                     ClassNameFilter classNameFilter,
                                     ClassLoader classLoader) {
    // candidates = "javax.sql.DataSource,org.springframework.jdbc..."
    // → 이미 문자열로 가져옴 (클래스 로딩 없음)

    List<String> missingClasses = filter(
        Arrays.asList(StringUtils.commaDelimitedListToStringArray(candidates)),
        ClassNameFilter.MISSING,  // 없는 클래스만 필터
        classLoader);

    if (!missingClasses.isEmpty()) {
        // 필요한 클래스가 없음 → 즉시 제외
        return ConditionOutcome.noMatch(
            ConditionMessage.forCondition(ConditionalOnClass.class)
                .didNotFind("required class", "required classes")
                .items(missingClasses));
    }
    return null;  // null = 1차 통과, 2차 평가로
}

// ClassNameFilter.MISSING.matches():
// Class.forName(className, false, classLoader) 시도
// → ClassNotFoundException 시 없다고 판단
// 하지만 1차에서는 ClassLoader.getResource()로 더 빠르게 체크
```

### 5. processor 없을 때 폴백 동작

```java
// AutoConfigurationMetadata.wasProcessed(className) = false
// → OnClassCondition이 직접 어노테이션 읽기로 폴백

// FilteringSpringBootCondition.match()
protected ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
                                          AutoConfigurationMetadata metadata) {

    // 각 클래스에 대해:
    String candidates = metadata.get(className, "ConditionalOnClass");

    if (candidates == null) {
        // 메타데이터 없음 → 클래스를 실제로 로딩해 어노테이션 읽기
        // → 클래스 로딩 비용 발생
        // → 어노테이션 파싱 비용 발생
        return null;  // 2차 평가(getMatchOutcome)로 넘어감
    }
    // 메타데이터 있음 → 문자열 비교만으로 빠르게 처리
    return getOutcome(candidates, ClassNameFilter.MISSING, classLoader);
}
```

---

## 💻 실험으로 확인하기

### 실험 1: processor 추가 여부에 따른 시작 시간 비교

```gradle
// build.gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-autoconfigure'

    // 이것 추가 여부로 시작 시간 비교
    annotationProcessor 'org.springframework.boot:spring-boot-autoconfigure-processor'
}
```

```bash
# 시작 시간 측정
time java -jar app.jar

# processor 없을 때: Started in 2.3 seconds
# processor 있을 때: Started in 1.9 seconds (차이는 환경마다 다름)
```

### 실험 2: 생성된 메타데이터 파일 확인

```bash
# 빌드 후 생성된 파일 확인
cat build/classes/java/main/META-INF/spring-autoconfigure-metadata.properties

# 또는 JAR 내부
jar -tf my-lib.jar | grep "autoconfigure-metadata"
unzip -p my-lib.jar META-INF/spring-autoconfigure-metadata.properties
```

### 실험 3: 커스텀 Auto-configuration에 processor 적용

```gradle
// 커스텀 라이브러리 build.gradle
dependencies {
    compileOnly 'org.springframework.boot:spring-boot-autoconfigure'
    annotationProcessor 'org.springframework.boot:spring-boot-autoconfigure-processor'
}
```

```java
// 이후 @AutoConfiguration 클래스 작성 → 빌드 시 자동으로 메타데이터 생성
@AutoConfiguration
@ConditionalOnClass(MyLibrary.class)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MyLibAutoConfiguration { ... }

// 생성되는 메타데이터:
// com.example.MyLibAutoConfiguration=
// com.example.MyLibAutoConfiguration.ConditionalOnClass=com.example.MyLibrary
// com.example.MyLibAutoConfiguration.AutoConfigureAfter=\
//   org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### 실험 4: spring-boot-configuration-processor와 차이 비교

```gradle
dependencies {
    // Auto-configuration 조건 메타데이터 (시작 성능)
    annotationProcessor 'org.springframework.boot:spring-boot-autoconfigure-processor'

    // @ConfigurationProperties IDE 자동완성 메타데이터
    annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
}
```

```
각각 생성하는 파일:
  autoconfigure-processor → META-INF/spring-autoconfigure-metadata.properties
  configuration-processor → META-INF/spring-configuration-metadata.json

독립적으로 동작, 둘 다 추가해도 충돌 없음
```

---

## ⚙️ 설정 최적화 팁

```gradle
// Gradle — annotationProcessor scope 사용
dependencies {
    // 런타임에 필요 없음 (컴파일 타임만)
    annotationProcessor 'org.springframework.boot:spring-boot-autoconfigure-processor'
}
```

```xml
<!-- Maven — optional=true -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure-processor</artifactId>
    <optional>true</optional>
    <!-- optional: 이 라이브러리를 사용하는 프로젝트에 전이되지 않음 -->
    <!-- 컴파일 타임에만 사용 -->
</dependency>
```

```
Spring Boot 공식 라이브러리들:
  spring-boot-autoconfigure 빌드 시 이 processor 사용
  → spring-boot-autoconfigure-3.x.jar 내부에
    META-INF/spring-autoconfigure-metadata.properties 포함됨
  → 사용자 프로젝트는 이 파일을 runtime에 그냥 읽음
```

---

## 🤔 트레이드오프

```
processor 추가:
  장점  시작 시간 단축 (소수~수십 ms, 많을수록 효과 큼)
       순서 정보도 컴파일 타임에 추출 → AutoConfigurationSorter 최적화
  단점  빌드 시간 소폭 증가 (어노테이션 처리 과정)
       processor 버전이 Spring Boot 버전과 맞아야 함

processor 미추가:
  장점  빌드 설정 간단
  단점  Auto-configuration이 적은 프로젝트에서는 차이 미미
       많은 Auto-configuration이 있는 대규모 프로젝트에서는 체감

권장:
  공개 라이브러리: 반드시 추가 (사용자 시작 성능 영향)
  사내 공통 라이브러리: 추가 권장
  단순 스타터/데모: 생략 가능
```

---

## 📌 핵심 정리

```
spring-boot-autoconfigure-processor
  역할: @AutoConfiguration 클래스의 조건·순서 어노테이션 → 컴파일 타임 추출
  출력: META-INF/spring-autoconfigure-metadata.properties
  효과: 런타임 1차 필터링에서 클래스 로딩 없이 조건 평가 가능

vs spring-boot-configuration-processor
  대상: @ConfigurationProperties 클래스
  출력: META-INF/spring-configuration-metadata.json
  효과: IDE 자동완성, 프로퍼티 문서화

processor 없을 때
  동작 자체: 정상
  성능: 클래스 로딩 + 어노테이션 파싱 비용 발생 → 시작 느려짐
  → OnClassCondition이 직접 어노테이션 읽기로 폴백

커스텀 Auto-configuration 라이브러리에서 추가 방법
  Gradle: annotationProcessor 'org.springframework.boot:spring-boot-autoconfigure-processor'
  Maven: optional=true 의존성
```

---

## 🤔 생각해볼 문제

**Q1.** `spring-boot-autoconfigure-metadata.properties`에 어떤 정보가 있으면 `AutoConfigurationSorter`가 파일을 직접 파싱해서 어노테이션을 읽는 비용을 줄일 수 있는가?

**Q2.** 커스텀 라이브러리 JAR를 빌드할 때 `spring-boot-autoconfigure-processor`를 `compile` scope 대신 `annotationProcessor`(Gradle) 또는 `optional=true`(Maven)로 지정하는 이유는?

**Q3.** `AutoConfigurationMetadata.wasProcessed(className)`이 `false`를 반환하는 경우는 어떤 상황이며, 이때 처리 흐름은 어떻게 달라지는가?

> 💡 **해설**
>
> **Q1.** `AutoConfigurationSorter`는 `@AutoConfigureAfter`, `@AutoConfigureBefore`, `@AutoConfigureOrder` 정보를 사용해 위상 정렬을 수행한다. 메타데이터 파일에 `ClassName.AutoConfigureAfter`, `ClassName.AutoConfigureBefore`, `ClassName.AutoConfigureOrder` 키가 있으면 정렬 시 클래스를 실제로 로딩해 어노테이션을 읽을 필요가 없다. 메타데이터에서 직접 문자열로 값을 읽어 순서 결정에 사용하므로, 많은 Auto-configuration 클래스가 있는 환경에서 클래스 로딩 비용을 줄이고 정렬 속도를 향상시킨다.
>
> **Q2.** `annotationProcessor`(Gradle) 또는 `optional=true`(Maven)으로 지정하면 이 라이브러리가 최종 JAR의 런타임 의존성에 포함되지 않고, 이 라이브러리를 사용하는 다른 프로젝트에도 전이되지 않는다. `spring-boot-autoconfigure-processor`는 컴파일 시에만 필요하고 런타임에는 필요 없다. 메타데이터 파일은 이미 컴파일 결과물에 포함되어 있으므로, 프로세서 JAR 자체를 런타임 클래스패스에 포함시킬 이유가 없다. 만약 `compile` scope로 지정하면 불필요한 의존성이 사용자 프로젝트의 클래스패스에 포함되어 크기가 커진다.
>
> **Q3.** `wasProcessed(className)`이 `false`인 경우는 두 가지다. 첫째, `spring-boot-autoconfigure-processor`가 빌드 시 추가되지 않아 해당 클래스의 메타데이터가 생성되지 않은 경우다. 둘째, processor가 있더라도 해당 클래스가 처리 대상 어노테이션(`@AutoConfiguration`, `@EnableAutoConfiguration` 등)을 가지지 않는 경우다. `false`인 경우 `OnClassCondition`은 메타데이터 기반 1차 필터링을 건너뛰고 해당 클래스에 대한 `getMatchOutcome()` 2차 평가로 직접 넘어간다. 2차 평가에서는 클래스를 실제로 로딩해 `@ConditionalOnClass` 어노테이션을 직접 읽고 조건을 평가한다.

---

<div align="center">

**[⬅️ 이전: Custom Auto-configuration 작성 가이드](./07-custom-autoconfiguration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — application.properties vs application.yml 처리 ➡️](../property-configuration/01-application-properties-yml.md)**

</div>
