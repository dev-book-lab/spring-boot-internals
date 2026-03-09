# Web MVC Auto-configuration — DispatcherServlet부터 ViewResolver까지 자동 구성 경로

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `WebMvcAutoConfiguration`이 등록하는 Bean 목록과 각 역할은?
- `DispatcherServlet`은 어떻게, 어느 경로에 등록되는가?
- `HandlerMapping` 우선순위가 어떻게 결정되는가?
- `WebMvcConfigurer`로 Auto-configuration을 확장하는 것이 안전한 이유는?
- `@EnableWebMvc`를 붙이면 Auto-configuration이 왜 비활성화되는가?
- `ContentNegotiationManager`는 Accept 헤더와 URL 확장자 처리를 어떻게 결정하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: Spring MVC 수동 설정의 복잡성

```java
// Spring MVC를 직접 설정한다면 (web.xml 없이)
public class MvcInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    @Override protected Class<?>[] getRootConfigClasses() { return new Class[]{RootConfig.class}; }
    @Override protected Class<?>[] getServletConfigClasses() { return new Class[]{WebConfig.class}; }
    @Override protected String[] getServletMappings() { return new String[]{"/"}; }
}

@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Bean public ViewResolver viewResolver() { ... }
    @Bean public MessageConverter jsonConverter() { ... }
    @Bean public HandlerMapping staticResourceMapping() { ... }
    // 수십 개의 Bean 직접 등록
}
```

```
Spring Boot의 해결:
  WebMvcAutoConfiguration이 모든 MVC 인프라 Bean을 자동 등록
  DispatcherServlet → "/"에 등록
  ViewResolver → Thymeleaf, FreeMarker, JSP 등 classpath 감지로 선택
  MessageConverter → Jackson, Gson 등 classpath 감지
  Static resource → /static, /public, /resources, /META-INF/resources
  
  확장: WebMvcConfigurer 구현 → 자동 설정 위에 추가 설정만 하면 됨
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @EnableWebMvc를 붙여야 Spring MVC가 동작한다

```java
// ❌ Spring Boot에서 불필요하고 오히려 해로운 설정
@Configuration
@EnableWebMvc  // ← Spring Boot에서는 붙이면 안 됨
public class WebConfig implements WebMvcConfigurer { }
```

```
왜 문제인가:
  @EnableWebMvc → @Import(DelegatingWebMvcConfiguration.class)
  → WebMvcConfigurationSupport Bean 등록

  WebMvcAutoConfiguration:
    @ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
    → WebMvcConfigurationSupport가 이미 있으면 → 전체 스킵!
  
  결과:
    WebMvcAutoConfiguration 제공하는 모든 기본값 사라짐
    Jackson 자동 설정, 정적 리소스 경로, ViewResolver 등 사라짐
    수동으로 전부 재설정해야 함

올바른 패턴:
  @Configuration + WebMvcConfigurer 구현 (MVC 부분 커스터마이징)
  @SpringBootApplication만 있어도 MVC 자동 동작
```

### Before: 커스텀 ViewResolver를 등록하면 기존 것이 무시된다

```
❌ 잘못된 이해:
  "내가 ViewResolver를 등록하면 Thymeleaf ViewResolver가 사라진다"

✅ 실제:
  ViewResolver는 체인(chain)으로 구성됨
  → ContentNegotiatingViewResolver가 모든 ViewResolver에 위임해 선택
  → 사용자 ViewResolver도 체인에 추가됨 (우선순위로 결정)
  
  모든 ViewResolver 대체:
    @Bean으로 ContentNegotiatingViewResolver 직접 등록
    → @ConditionalOnMissingBean → Auto-configuration 스킵
```

---

## ✨ 올바른 이해와 사용

### After: WebMvcAutoConfiguration이 등록하는 주요 Bean

```
WebMvcAutoConfiguration 등록 Bean 목록:

─ DispatcherServlet 관련
  DispatcherServlet                   "/"에 등록된 프론트 컨트롤러
  DispatcherServletRegistrationBean   Servlet 등록 설정

─ HandlerMapping
  RequestMappingHandlerMapping        @RequestMapping 처리 (order=0)
  BeanNameUrlHandlerMapping           Bean 이름으로 URL 매핑 (order=2)
  RouterFunctionMapping               WebFlux 호환용
  WelcomePageHandlerMapping           / 요청 → index.html

─ HandlerAdapter
  RequestMappingHandlerAdapter        @RequestMapping 실행
  HttpRequestHandlerAdapter           정적 리소스 등
  SimpleControllerHandlerAdapter      레거시 Controller 인터페이스

─ ViewResolver
  ContentNegotiatingViewResolver      모든 ViewResolver 위임 (order=Highest)
  BeanNameViewResolver                Bean 이름으로 View 조회
  ThymeleafViewResolver               (Thymeleaf 있을 때)
  InternalResourceViewResolver        JSP 등 fallback

─ MessageConverter
  ByteArrayHttpMessageConverter
  StringHttpMessageConverter
  ResourceHttpMessageConverter
  MappingJackson2HttpMessageConverter (Jackson 있을 때)
  ...

─ 기타
  LocaleResolver                      요청 로케일 결정
  ThemeResolver                       테마 처리
  HandlerExceptionResolver            예외 처리 체인
  ResourceHttpRequestHandler          정적 리소스 서빙
```

---

## 🔬 내부 동작 원리

### 1. WebMvcAutoConfiguration 조건과 구조

```java
// WebMvcAutoConfiguration.java
@AutoConfiguration(
    after = { DispatcherServletAutoConfiguration.class,
              TaskExecutionAutoConfiguration.class,
              ValidationAutoConfiguration.class }
)
@ConditionalOnWebApplication(type = Type.SERVLET)  // Servlet 웹 환경만
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)  // ← @EnableWebMvc 방지
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
public class WebMvcAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Import(EnableWebMvcConfiguration.class)  // ← 핵심 MVC 설정 위임
    @EnableConfigurationProperties({ WebMvcProperties.class, WebProperties.class })
    public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer {
        // 기본 MessageConverter, ViewResolver 등 등록
    }
}
```

```java
// EnableWebMvcConfiguration — DelegatingWebMvcConfiguration 상속
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(WebMvcProperties.class)
public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration
        implements ResourceLoaderAware {

    // DelegatingWebMvcConfiguration:
    //   WebMvcConfigurerComposite로 모든 WebMvcConfigurer Bean을 수집
    //   각 설정 메서드 호출 시 모든 Configurer에 위임

    @Override
    protected ConfigurableWebBindingInitializer getConfigurableWebBindingInitializer(
            FormattingConversionService mvcConversionService,
            Validator mvcValidator) {
        // 커스텀 ConversionService, Validator 주입
    }
}
```

### 2. DispatcherServletAutoConfiguration — DispatcherServlet 등록

```java
// DispatcherServletAutoConfiguration.java
@AutoConfiguration(after = ServletWebServerFactoryAutoConfiguration.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
public class DispatcherServletAutoConfiguration {

    // ① DispatcherServlet Bean 생성
    @Configuration(proxyBeanMethods = false)
    @Conditional(DefaultDispatcherServletCondition.class)
    @ConditionalOnClass(ServletRegistration.class)
    @EnableConfigurationProperties(WebMvcProperties.class)
    protected static class DispatcherServletConfiguration {

        @Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
            DispatcherServlet dispatcherServlet = new DispatcherServlet();
            // spring.mvc.dispatch-options-request 등 적용
            dispatcherServlet.setDispatchOptionsRequest(
                webMvcProperties.isDispatchOptionsRequest());
            dispatcherServlet.setDispatchTraceRequest(
                webMvcProperties.isDispatchTraceRequest());
            dispatcherServlet.setThrowExceptionIfNoHandlerFound(
                webMvcProperties.isThrowExceptionIfNoHandlerFound());
            dispatcherServlet.setPublishEvents(
                webMvcProperties.isPublishRequestHandledEvents());
            dispatcherServlet.setEnableLoggingRequestDetails(
                webMvcProperties.isLogRequestDetails());
            return dispatcherServlet;
        }
    }

    // ② Servlet 등록 (URL 매핑)
    @Configuration(proxyBeanMethods = false)
    @Conditional(DispatcherServletRegistrationCondition.class)
    @ConditionalOnClass(ServletRegistration.class)
    @EnableConfigurationProperties(WebMvcProperties.class)
    @Import(DispatcherServletConfiguration.class)
    protected static class DispatcherServletRegistrationConfiguration {

        @Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
        @ConditionalOnBean(value = DispatcherServlet.class,
                            name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServletRegistrationBean dispatcherServletRegistration(
                DispatcherServlet dispatcherServlet,
                WebMvcProperties webMvcProperties, ...) {

            DispatcherServletRegistrationBean registration =
                new DispatcherServletRegistrationBean(
                    dispatcherServlet,
                    webMvcProperties.getServlet().getPath());  // 기본값: "/"
            // spring.mvc.servlet.path 로 변경 가능
            // 예: "/api" → "/api/*"에만 DispatcherServlet 적용

            registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
            registration.setLoadOnStartup(
                webMvcProperties.getServlet().getLoadOnStartup());
            // multipart 설정 등
            return registration;
        }
    }
}
```

### 3. RequestMappingHandlerMapping — @RequestMapping 처리

```java
// WebMvcAutoConfiguration 내 RequestMappingHandlerMapping 생성
@Bean
@Primary
@Override
public RequestMappingHandlerMapping requestMappingHandlerMapping(
        @Qualifier("mvcContentNegotiationManager")
        ContentNegotiationManager contentNegotiationManager,
        @Qualifier("mvcConversionService")
        FormattingConversionService conversionService,
        @Qualifier("mvcResourceUrlProvider")
        ResourceUrlProvider resourceUrlProvider) {

    RequestMappingHandlerMapping mapping = super.requestMappingHandlerMapping(
        contentNegotiationManager, conversionService, resourceUrlProvider);

    // spring.mvc.pathmatch.matching-strategy 적용
    // PATH_PATTERN_PARSER (기본, Boot 2.6+) 또는 ANT_PATH_MATCHER
    mapping.setOrder(0);  // 최고 우선순위
    return mapping;
}
```

```
HandlerMapping 처리 순서 (order 기준):

1. RequestMappingHandlerMapping (order=0)
   → @GetMapping, @PostMapping 등 처리

2. BeanNameUrlHandlerMapping (order=2)
   → "/beanName" URL로 등록된 Controller Bean

3. RouterFunctionMapping (order=-1, Reactive 아닌 경우 skip)

4. WelcomePageHandlerMapping (order=2, "/" 요청 전용)
   → src/main/resources/static/index.html 서빙

5. SimpleUrlHandlerMapping (정적 리소스, order=Integer.MAX-1)
   → /static/**, /public/**, /resources/**, /META-INF/resources/**
```

### 4. WebMvcConfigurer — 확장 포인트

```java
// WebMvcConfigurer 주요 메서드
public interface WebMvcConfigurer {

    // MessageConverter 추가
    default void configureMessageConverters(List<HttpMessageConverter<?>> converters) {}
    default void extendMessageConverters(List<HttpMessageConverter<?>> converters) {}

    // Interceptor 등록
    default void addInterceptors(InterceptorRegistry registry) {}

    // CORS 설정
    default void addCorsMappings(CorsRegistry registry) {}

    // ViewController 등록 (Controller 없이 URL→View 직접 매핑)
    default void addViewControllers(ViewControllerRegistry registry) {}

    // 정적 리소스 핸들러
    default void addResourceHandlers(ResourceHandlerRegistry registry) {}

    // Formatter/Converter 추가
    default void addFormatters(FormatterRegistry registry) {}

    // ArgumentResolver 추가
    default void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {}
}
```

```java
// DelegatingWebMvcConfiguration이 수집하는 방식
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

    private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();

    @Autowired(required = false)
    public void setConfigurers(List<WebMvcConfigurer> configurers) {
        // 모든 WebMvcConfigurer Bean 자동 수집 (@Autowired List)
        if (!CollectionUtils.isEmpty(configurers)) {
            this.configurers.addWebMvcConfigurers(configurers);
        }
    }

    @Override
    protected void addInterceptors(InterceptorRegistry registry) {
        // 수집된 모든 Configurer의 addInterceptors 호출
        this.configurers.addInterceptors(registry);
    }
    // ... 모든 메서드 위임
}
```

```
WebMvcConfigurer vs WebMvcConfigurationSupport:

WebMvcConfigurer (권장):
  → 자동 설정 위에 추가 설정만
  → @ConditionalOnMissingBean(WebMvcConfigurationSupport.class) 영향 없음
  → Auto-configuration 유지

WebMvcConfigurationSupport (사용 자제):
  → 이 Bean이 등록되면 WebMvcAutoConfiguration 전체 스킵
  → 모든 MVC 설정을 직접 관리해야 함
  → @EnableWebMvc를 붙이면 이 Bean이 등록됨
```

### 5. ContentNegotiationManager — 응답 타입 결정

```java
// ContentNegotiationManagerFactoryBean — ContentNegotiationManager 생성
@Bean
@Override
public ContentNegotiationManager mvcContentNegotiationManager() {
    ContentNegotiationManagerFactoryBean factory = new ContentNegotiationManagerFactoryBean();

    // spring.mvc.contentnegotiation.* 적용
    factory.setDefaultContentType(MediaType.APPLICATION_JSON);
    // Accept 헤더 기반 (기본 활성화)
    factory.setFavorParameter(mvcProperties.getContentnegotiation().isFavorParameter());
    // URL 경로 확장자 기반 (.json, .xml) — Boot 2.6+ 기본 비활성화
    factory.setFavorPathExtension(false);

    return factory.build();
}
```

```
ContentNegotiation 전략 (우선순위 순):
1. URL 파라미터 (?format=json) — favorParameter=true 시
2. Accept 헤더 (Accept: application/json) — 기본 활성화
3. Path Extension (.json, .xml) — Boot 2.6+ 기본 비활성화 (보안 이유)
4. 기본값 (defaultContentType)
```

---

## 💻 실험으로 확인하기

### 실험 1: @EnableWebMvc 영향 확인

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = SpringApplication.run(App.class, args);
        // @EnableWebMvc 없이: WebMvcAutoConfiguration 적용됨
        System.out.println(ctx.containsBean("requestMappingHandlerMapping")); // true
        System.out.println(ctx.containsBean("mvcContentNegotiationManager")); // true
    }
}

// @EnableWebMvc 붙이면:
// WebMvcAutoConfiguration 스킵 → 기본 설정 사라짐
// Jackson MessageConverter, 정적 리소스 등 수동 재설정 필요
```

### 실험 2: WebMvcConfigurer로 CORS 전역 설정

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://frontend.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoggingInterceptor())
            .addPathPatterns("/api/**")
            .excludePathPatterns("/api/health");
    }
}
// WebMvcAutoConfiguration은 그대로 유지됨 + CORS, Interceptor 추가
```

### 실험 3: DispatcherServlet 경로 변경

```yaml
# DispatcherServlet을 /api/** 에만 매핑
spring:
  mvc:
    servlet:
      path: /api
# → /api/users → DispatcherServlet 처리
# → /static/img.png → 정적 리소스 (Servlet 처리 전 Tomcat이 직접 처리)
```

### 실험 4: MessageConverter 우선순위 확인

```java
@Autowired
private RequestMappingHandlerAdapter adapter;

@PostConstruct
void printConverters() {
    adapter.getMessageConverters().forEach(c ->
        System.out.println(c.getClass().getSimpleName()));
}
// 출력 순서:
// ByteArrayHttpMessageConverter
// StringHttpMessageConverter
// ResourceHttpMessageConverter
// ...
// MappingJackson2HttpMessageConverter  ← Jackson이 있으면 등록됨
```

---

## ⚙️ 설정 최적화 팁

```yaml
spring:
  mvc:
    # HTTP PUT/PATCH에서 form 데이터 처리
    formcontent:
      filter:
        enabled: true
    # DispatcherServlet 로그 (디버깅용)
    log-request-details: false  # true로 하면 헤더/파라미터 로그 출력
    # Path matching 전략
    pathmatch:
      matching-strategy: path-pattern-parser  # 기본값 (Boot 2.6+)
  web:
    resources:
      # 정적 리소스 캐시 설정
      cache:
        period: 3600          # 1시간
        cachecontrol:
          max-age: 3600
          must-revalidate: true
      # 정적 리소스 경로 커스터마이징
      static-locations:
        - classpath:/static/
        - classpath:/public/
        - file:/opt/app/static/  # 외부 파일시스템도 가능
```

---

## 🤔 트레이드오프

```
Auto-configuration 활용:
  장점  기본값으로 즉시 동작, Jackson, Thymeleaf 등 자동 연동
  단점  내부 Bean 교체 시 @ConditionalOnMissingBean 주의 필요

WebMvcConfigurer vs @EnableWebMvc:
  WebMvcConfigurer  Auto-configuration 유지 + 부분 커스터마이징
  @EnableWebMvc     완전 제어 + 모든 MVC 설정 직접 관리
  → 거의 모든 경우 WebMvcConfigurer로 충분
  → @EnableWebMvc는 Spring Boot 없이 사용하는 경우에만 필요

ContentNegotiation 경로 확장자:
  장점  /users.json → JSON 응답 (직관적)
  단점  RFD(Reflected File Download) 보안 취약점
       Spring Boot 2.6+에서 기본 비활성화
  → Accept 헤더 사용 권장
```

---

## 📌 핵심 정리

```
WebMvcAutoConfiguration 비활성화 조건
  @EnableWebMvc 또는 WebMvcConfigurationSupport Bean 존재 시 전체 스킵
  → Spring Boot에서 @EnableWebMvc 사용 금지

DispatcherServlet 등록
  DispatcherServletAutoConfiguration → "/"에 매핑
  spring.mvc.servlet.path로 경로 변경 가능

HandlerMapping 우선순위
  RequestMappingHandlerMapping (order=0) → @RequestMapping
  정적 리소스 (order=MAX-1) → /static/**, /public/** 등

WebMvcConfigurer 확장
  DelegatingWebMvcConfiguration이 모든 WebMvcConfigurer Bean 자동 수집
  → Auto-configuration 유지하면서 부분 커스터마이징

ContentNegotiation
  Accept 헤더 기본 활성화
  경로 확장자 Boot 2.6+ 기본 비활성화 (보안)
```

---

## 🤔 생각해볼 문제

**Q1.** `WebMvcConfigurer.configureMessageConverters()`와 `extendMessageConverters()`의 차이는 무엇인가? 각각을 사용해야 하는 경우는?

**Q2.** `@RestController`가 붙은 클래스에서 `@RequestMapping("/users")`로 반환하는 `List<User>`가 JSON으로 직렬화되는 전체 경로를 설명하라.

**Q3.** `spring.web.resources.static-locations`에 외부 파일 시스템 경로(`file:/opt/app/static/`)를 추가하면 보안상 어떤 주의가 필요한가?

> 💡 **해설**
>
> **Q1.** `configureMessageConverters(List)`는 Auto-configuration이 등록하는 기본 MessageConverter 목록을 완전히 교체한다. 이 메서드에 하나라도 추가하면 기본 Jackson Converter 등이 모두 사라지므로, 완전히 새로운 Converter 목록을 구성할 때 사용한다. `extendMessageConverters(List)`는 Auto-configuration이 이미 등록한 기본 Converter 목록에 추가하거나 순서를 변경한다. 기존 기능을 유지하면서 특정 Converter만 추가/교체할 때 사용한다. 대부분의 경우 `extendMessageConverters`를 사용하는 것이 안전하다.
>
> **Q2.** 요청이 들어오면 `DispatcherServlet`이 `RequestMappingHandlerMapping`을 통해 `@RequestMapping("/users")` 메서드를 찾는다. `RequestMappingHandlerAdapter`가 해당 메서드를 실행해 `List<User>`를 반환받는다. `@RestController`는 `@ResponseBody`를 포함하므로 반환값을 HTTP 응답 본문에 직접 쓴다. `HttpEntityMethodProcessor`가 클라이언트의 `Accept` 헤더(예: `application/json`)와 등록된 `HttpMessageConverter` 목록을 비교해 `MappingJackson2HttpMessageConverter`를 선택한다. Jackson의 `ObjectMapper`가 `List<User>`를 JSON 바이트 배열로 직렬화해 응답 스트림에 쓴다.
>
> **Q3.** 외부 파일 시스템 경로를 정적 리소스로 노출하면 Path Traversal 공격에 취약해질 수 있다. 예를 들어 `/static/../../../etc/passwd` 같은 요청으로 의도치 않은 파일이 노출될 수 있다. Spring의 `ResourceHttpRequestHandler`는 경로 정규화를 수행해 기본적으로 방어하지만, 심볼릭 링크나 OS별 경로 처리 차이로 우회될 수 있다. 외부 경로는 서빙할 파일 종류를 `spring.web.resources.static-locations`에 구체적으로 지정하고, 웹 서버(Nginx 등)를 앞단에 두어 정적 리소스는 Spring Boot를 거치지 않도록 하는 것이 더 안전하다.

---

<div align="center">

**[⬅️ 이전: JPA Auto-configuration 과정](./05-jpa-autoconfiguration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Custom Auto-configuration 작성 가이드 ➡️](./07-custom-autoconfiguration.md)**

</div>
