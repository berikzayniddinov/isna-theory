# 61. Spring MVC контроллеры под капотом: DispatcherServlet, HandlerMapping, ArgumentResolvers

## Зачем понимать Spring MVC глубже

Разработчик впервые пишущий Spring контроллер видит абстракцию. Ставит @GetMapping, возвращает объект — что-то magic превращает в JSON response. Работает — moving on. Production reality приносит nuances требующие deeper understanding. Куда идёт запрос между Tomcat и controller method? Почему @RequestBody OrderDto deserialized автоматически но custom types требуют HttpMessageConverter? Почему @Transactional на controller ломает LazyInit? Почему Filter и Interceptor различаются и когда какой использовать?

Разница между разработчиком «использующим MVC» и «понимающим Spring MVC internals» проявляется в troubleshooting сложных request handling issues. Первый видит 500 error и searches для fix — либо копирует Stack Overflow solution. Второй знает DispatcherServlet flow. HandlerMapping matches URL к method. HandlerAdapter invokes method с ArgumentResolvers resolving parameters. ReturnValueHandlers process return value через HttpMessageConverter converting к JSON. Interceptors wrap invocation. Exception handlers process errors. Знание flow позволяет предсказывать behavior и pinpointing exact place где что-то не работает.

В этом файле разберём Spring MVC глубоко. Общая картина flow HTTP request. DispatcherServlet front controller. HandlerMapping для URL routing. HandlerAdapter для method invocation. HandlerMethodArgumentResolver для parameter resolution. HandlerMethodReturnValueHandler для response processing. HttpMessageConverter — JSON/XML serialization. Content Negotiation strategies. Interceptors vs Filters. Exception handling с @ControllerAdvice. Custom ArgumentResolver. Async controllers (Callable, DeferredResult, CompletableFuture, SSE). Bean Validation. ResponseEntity plus WebMvcConfigurer. Production caveats.

## Общая картина flow

Complete HTTP request path через Spring MVC:
```
HTTP request → Servlet Container (Tomcat)
                    │
                    │  Filter chain
                    ▼
             DispatcherServlet
                    │
                    ├─ HandlerMapping → HandlerMethod (метод в @Controller)
                    │
                    ├─ Interceptors preHandle
                    │
                    ├─ HandlerAdapter вызывает метод
                    │        │
                    │        ├─ Argument resolvers (@PathVariable, @RequestBody, ...)
                    │        │
                    │        ├─ метод выполняется
                    │        │
                    │        └─ Return value handlers (ResponseEntity, @ResponseBody, ...)
                    │
                    ├─ Interceptors postHandle
                    │
                    ├─ ViewResolver + View (для не-REST)
                    │       или HttpMessageConverter → JSON body (для REST)
                    │
                    └─ Interceptors afterCompletion
                    │
                    ▼
             HTTP response
```

Каждый компонент имеет specific responsibility. Understanding flow enables reasoning about behavior at each step.

## DispatcherServlet: front controller

Единственный Servlet зарегистрированный Spring Boot в servlet-контейнере. Обрабатывает все URL приложения. Central point where all requests enter Spring MVC processing.

Что делает doDispatch(HttpServletRequest, HttpServletResponse):
```
1. getHandler(request)              → HandlerExecutionChain (handler + interceptors)
2. getHandlerAdapter(handler)       → HandlerAdapter
3. interceptors.preHandle()          → если false — stop
4. ha.handle(request, response, handler)   → ModelAndView (или null для REST)
5. interceptors.postHandle()
6. render (ViewResolver + View) или уже отписано в response
7. interceptors.afterCompletion()   → всегда, даже при exception
```

Sequence critical. HandlerMapping first — determine what to invoke. HandlerAdapter — actually invoke. Interceptors wrap invocation. Return value processed. Cleanup через afterCompletion.

При exception. Ищется HandlerExceptionResolver (для @ExceptionHandler). Если найден — обрабатывает. Если нет — пробрасывает контейнеру — 500. Structured error handling через specific handlers.

Регистрация. Spring Boot автоматически через DispatcherServletAutoConfiguration. Регистрирует DispatcherServlet на / (по default). Настраивается через spring.mvc.* properties.

Можно поменять URL:
```yaml
spring.mvc.servlet.path: /api
```

Sometimes useful для multi-servlet applications или explicit prefix organization.

## HandlerMapping

Определяет какой handler обрабатывает URL. Core routing component.

Основной — RequestMappingHandlerMapping. При старте сканирует все @RequestMapping / @GetMapping / etc в @Controller / @RestController. Строит карту:
```
GET  /api/orders           → OrderController.list()
POST /api/orders           → OrderController.create()
GET  /api/orders/{id}      → OrderController.get(Long)
DELETE /api/orders/{id}    → OrderController.delete(Long)
```

При запросе — ищет match по URL plus method plus headers plus params. Multi-dimensional matching.

Другие HandlerMapping implementations. BeanNameUrlHandlerMapping — legacy (bean name = URL). SimpleUrlHandlerMapping — явные mappings. WebMvcConfigurer.addResourceHandlers() — статические ресурсы.

Порядок. Spring перебирает handlerMappings по @Order, первый нашёл — выигрывает. Configurable priority.

Как выбирается лучший match. Для request GET /api/orders/1:
```
Кандидаты:
  GET /api/orders/{id}
  GET /api/orders/*
  GET /**
```

Приоритеты. Более специфичный path выигрывает. Exact match > pattern. Meta-request info (headers, params) — тоже влияет.

Specificity ordering ensures deterministic routing decisions. Explicit paths preferred over wildcards.

## HandlerAdapter

Вызывает handler.

Основной — RequestMappingHandlerAdapter. Работает с HandlerMethod (метод plus инстанс controller'а).

Основная логика. Резолвить аргументы метода через HandlerMethodArgumentResolver. Вызвать метод через reflection. Обработать возвращаемое значение через HandlerMethodReturnValueHandler.

Другие HandlerAdapters. HttpRequestHandlerAdapter — для HttpRequestHandler. SimpleControllerHandlerAdapter — legacy Controller.

99 percent работы — RequestMappingHandlerAdapter. Modern Spring MVC everything goes through this adapter.

## HandlerMethodArgumentResolver

Резолвит каждый параметр handler-метода. Rich extension point.

Примеры:
```java
@GetMapping("/orders/{id}")
public Order get(@PathVariable Long id,
                 @RequestParam(defaultValue = "0") int page,
                 @RequestHeader("X-Trace-Id") String traceId,
                 @RequestBody OrderDto dto,
                 HttpServletRequest request,
                 @AuthenticationPrincipal Jwt jwt) { ... }
```

Каждый параметр обрабатывает свой resolver.

PathVariableMethodArgumentResolver — @PathVariable. Extracts from URL path.

RequestParamMethodArgumentResolver — @RequestParam. Query params.

RequestHeaderMethodArgumentResolver — @RequestHeader. HTTP headers.

RequestBodyMethodProcessor — @RequestBody. Использует HttpMessageConverter для body deserialization.

ServletRequestMethodArgumentResolver — HttpServletRequest / HttpServletResponse. Raw servlet objects.

ModelAttributeMethodProcessor — @ModelAttribute. Form data binding.

PrincipalMethodArgumentResolver — Principal. Security abstraction.

AuthenticationPrincipalArgumentResolver — @AuthenticationPrincipal. Extracts from SecurityContext.

Plus кастомные (см. section 12).

Как выбирается resolver. Каждый resolver имеет supportsParameter(MethodParameter). Spring перебирает — первый true выигрывает. Chain of responsibility pattern.

## HandlerMethodReturnValueHandler

Обрабатывает возвращаемое значение метода.

Примеры different return types:
```java
public String list() { return "orders"; }              // ViewName
public ModelAndView list() { ... }                      // ModelAndView
public List<Order> list() { ... }                       // JSON (если @ResponseBody)
public ResponseEntity<Order> get() { ... }              // ResponseEntity
public Callable<Order> async() { ... }                  // async servlet
public CompletableFuture<Order> future() { ... }        // async
public Mono<Order> reactive() { ... }                   // WebFlux (не MVC)
```

Handlers processing different return types.

RequestResponseBodyMethodProcessor — @ResponseBody / @RestController. Serialization через HttpMessageConverter.

ViewNameMethodReturnValueHandler — String → view name resolution.

ModelAndViewMethodReturnValueHandler — explicit ModelAndView.

HttpEntityMethodProcessor — ResponseEntity. Full response control.

CallableMethodReturnValueHandler — async processing.

DeferredResultMethodReturnValueHandler — async с external event.

## HttpMessageConverter: JSON/XML

Ключевая часть. Преобразует между Java-объектом и HTTP body. Core serialization/deserialization mechanism.

Как работает.

Для @RequestBody. Читает Content-Type header (например application/json). Ищет converter который supports это. Читает body — создаёт объект. Deserialization pipeline.

Для @ResponseBody. Читает Accept header (например application/json). Ищет converter. Сериализует объект — пишет в body. Устанавливает Content-Type. Serialization pipeline.

Стандартные converters. MappingJackson2HttpMessageConverter — JSON (Jackson). Standard для modern APIs. MappingJackson2XmlHttpMessageConverter — XML. StringHttpMessageConverter — String. ByteArrayHttpMessageConverter — byte[]. FormHttpMessageConverter — form-urlencoded. ResourceHttpMessageConverter — Resource (файлы, downloads). AtomFeedHttpMessageConverter — RSS/Atom.

По default. Jackson (JSON) активен в Spring Boot если jackson-databind в classpath. Auto-configuration handles setup.

Настройка Jackson:
```yaml
spring:
  jackson:
    serialization:
      write-dates-as-timestamps: false
      indent-output: false
    deserialization:
      fail-on-unknown-properties: false
    property-naming-strategy: SNAKE_CASE
    default-property-inclusion: non_null
    time-zone: UTC
```

Или программно:
```java
@Bean
Jackson2ObjectMapperBuilderCustomizer jackson() {
    return builder -> builder
        .modules(new JavaTimeModule())
        .serializationInclusion(JsonInclude.Include.NON_NULL);
}
```

Custom converter:
```java
@Configuration
class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new MyCustomConverter());
    }
}
```

Extension point для non-standard formats.

## Content Negotiation

Spring определяет какой формат возвращать. Multiple strategies determine response format.

Strategy. По default. Accept header клиента (Accept: application/json). URL suffix (/orders.json) — deprecated. Parameter (?format=json) — если включено.

Настройка:
```yaml
spring.mvc.contentnegotiation:
  favor-parameter: true
  parameter-name: format
  media-types:
    json: application/json
    xml: application/xml
```

Producing / consuming annotations control:
```java
@GetMapping(value = "/orders", produces = "application/json")
public List<Order> listJson() { }

@GetMapping(value = "/orders", produces = "application/xml")
public List<Order> listXml() { }

@PostMapping(value = "/orders", consumes = "application/json")
public Order create(@RequestBody OrderDto dto) { }
```

Spring выберет метод по Accept / Content-Type headers. Method-level content negotiation.

## Interceptors

Middleware уровня Spring MVC (не сервлета):
```java
@Component
class LoggingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse resp, Object handler) {
        log.info("Before: {}", req.getRequestURI());
        return true;   // false → stop
    }

    @Override
    public void postHandle(HttpServletRequest req, HttpServletResponse resp, Object handler, ModelAndView mv) {
        log.info("After: {}", resp.getStatus());
    }

    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse resp, Object handler, Exception ex) {
        // always, even on error
    }
}

@Configuration
class WebConfig implements WebMvcConfigurer {
    @Autowired LoggingInterceptor interceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(interceptor).addPathPatterns("/api/**");
    }
}
```

Interceptors vs Filters — важное различие.

Filter — уровень Servlet API, срабатывает до DispatcherServlet. Servlet-level abstraction. Access to raw request/response.

Interceptor — уровень Spring MVC, срабатывает после DispatcherServlet, но до/после handler. Access to MVC constructs — HandlerMethod, ModelAndView.

Filters — для cross-cutting всего (auth, CORS, gzip). Applies к all requests uniformly.

Interceptors — для MVC-специфичного (аудит handler-специфичный, model manipulation). MVC context aware.

## Filters

Обычные Servlet filters:
```java
@Component
class TraceIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse resp, FilterChain chain)
            throws ServletException, IOException {
        String traceId = req.getHeader("X-Trace-Id");
        if (traceId == null) traceId = UUID.randomUUID().toString();
        MDC.put("traceId", traceId);
        try {
            chain.doFilter(req, resp);
        } finally {
            MDC.clear();
        }
    }
}
```

Spring Boot автоматически регистрирует @Component filter.

Порядок. @Order или FilterRegistrationBean. Explicit control важен для dependent filters.

Стандартные filters Spring Boot автоматически registers. CharacterEncodingFilter — UTF-8. HiddenHttpMethodFilter — для form-based PUT/DELETE. RequestContextFilter. Security filters (если security on classpath).

Spring Security сама — цепочка Filter'ов. Full deep dive в файле 24 spring-security-basics.

## Exception handling

Multiple mechanisms available. Different scopes.

@ExceptionHandler in controller. Локально для одного controller:
```java
@RestController
class OrderController {
    @GetMapping("/orders/{id}")
    public Order get(@PathVariable Long id) { ... }

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorDto> notFound(EntityNotFoundException e) {
        return ResponseEntity.status(404).body(new ErrorDto(e.getMessage()));
    }
}
```

Работает только для этого controller. Scoped exception handling.

@ControllerAdvice global:
```java
@ControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorDto> notFound(EntityNotFoundException e) {
        return ResponseEntity.status(404).body(new ErrorDto(e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorDto> validation(MethodArgumentNotValidException e) {
        Map<String, String> errors = new HashMap<>();
        e.getBindingResult().getFieldErrors().forEach(err ->
            errors.put(err.getField(), err.getDefaultMessage()));
        return ResponseEntity.badRequest().body(new ErrorDto("validation", errors));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorDto> generic(Exception e) {
        log.error("Unhandled exception", e);
        return ResponseEntity.status(500).body(new ErrorDto("internal error"));
    }
}
```

Работает для всех controllers. Централизованная обработка ошибок.

Scoped @ControllerAdvice:
```java
@ControllerAdvice(basePackages = "kz.gov.kgd.isna.knp.api")
```

Applies to controllers в specified package.

ResponseStatusException быстрый способ бросить с кодом:
```java
if (order == null) {
    throw new ResponseStatusException(HttpStatus.NOT_FOUND, "order not found");
}
```

Без ControllerAdvice. Convenience для simple cases.

@ResponseStatus на exception классе:
```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class OrderNotFoundException extends RuntimeException { }
```

При throw — Spring вернёт 404. Semantic exception classes с explicit HTTP status.

ProblemDetail стандартный problem+json (RFC 7807). Spring 6 / Boot 3+ поддерживает:
```java
@ExceptionHandler
public ProblemDetail handle(EntityNotFoundException e) {
    return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
}
```

Response:
```json
{
  "type": "about:blank",
  "title": "Not Found",
  "status": 404,
  "detail": "order 42 not found"
}
```

Стандартизация ошибок API. Consistent format across services.

## Custom ArgumentResolver

Хочешь свой параметр:
```java
@GetMapping("/orders")
public List<Order> list(@CurrentUser User user) { }
```

```java
public class CurrentUserResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CurrentUser.class);
    }

    @Override
    public Object resolveArgument(MethodParameter param, ModelAndViewContainer mavContainer,
                                   NativeWebRequest req, WebDataBinderFactory binderFactory) {
        String username = SecurityContextHolder.getContext().getAuthentication().getName();
        return userRepo.findByUsername(username);
    }
}

@Configuration
class WebConfig implements WebMvcConfigurer {
    @Autowired CurrentUserResolver resolver;

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(resolver);
    }
}
```

Extension point для encapsulating common parameter extraction patterns. Cleaner controllers.

## Async controllers

Не блокировать поток на долгие операции. Multiple mechanisms.

Callable:
```java
@GetMapping("/slow")
public Callable<Order> slow() {
    return () -> {
        Thread.sleep(5000);
        return service.compute();
    };
}
```

Spring. Освобождает Tomcat thread. Запускает Callable на другом executor (spring.mvc.async.request-timeout). По завершении — resume async servlet, отвечает клиенту.

DeferredResult для external completion:
```java
@GetMapping("/notify")
public DeferredResult<Notification> waitForEvent() {
    DeferredResult<Notification> result = new DeferredResult<>(30000L);
    eventRegistry.register(result);
    return result;
}

// где-то потом
result.setResult(notification);
```

Полезно для long polling — клиент ждёт event. External code completes result когда ready.

CompletableFuture:
```java
@GetMapping("/orders/{id}")
public CompletableFuture<Order> getAsync(@PathVariable Long id) {
    return CompletableFuture.supplyAsync(() -> service.get(id));
}
```

Modern async style. Composable через CompletableFuture chain.

Server-Sent Events (SSE):
```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter events() {
    SseEmitter emitter = new SseEmitter(0L);   // no timeout
    executor.submit(() -> {
        try {
            for (int i = 0; i < 100; i++) {
                emitter.send("event " + i);
                Thread.sleep(1000);
            }
            emitter.complete();
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    });
    return emitter;
}
```

Streaming events клиенту. One-way server-to-client push.

Virtual Threads (Java 21). Spring Boot 3.2+ с virtual threads делает всё async автоматически:
```yaml
spring.threads.virtual.enabled: true
```

Каждый request — свой virtual thread. Блокирование не съедает carrier thread. Automatic concurrency без async wrappers.

## Validation через @Valid

Bean Validation (JSR-380):
```java
public class OrderDto {
    @NotBlank(message = "customerId required")
    private String customerId;

    @Min(value = 1, message = "amount must be positive")
    @Max(value = 1_000_000)
    private BigDecimal amount;

    @Email
    private String contactEmail;

    @Valid   // recursive validation
    private List<OrderItemDto> items;
}
```

```java
@PostMapping("/orders")
public Order create(@Valid @RequestBody OrderDto dto) { }
```

При invalid — MethodArgumentNotValidException — 400 (handle через @ControllerAdvice).

Custom validator:
```java
@Target({ FIELD })
@Retention(RUNTIME)
@Constraint(validatedBy = CustomerIdValidator.class)
public @interface ValidCustomerId { }

class CustomerIdValidator implements ConstraintValidator<ValidCustomerId, String> {
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        return value != null && value.matches("^C\\d{6}$");
    }
}
```

Custom validation logic для business-specific constraints.

## RequestMappingInfo

Каждый @RequestMapping внутри → RequestMappingInfo. Internal representation.

Patterns (URL). Methods. Params conditions. Headers conditions. Consumes / produces.

RequestMappingHandlerMapping держит Map<RequestMappingInfo, HandlerMethod>.

При запросе. Ищет все matching (может быть несколько по разным критериям). Выбирает most specific.

Отсюда — понимание почему /orders/{id} не конфликтует с /orders/search. Path variable {id} matches literals но more specific literals preferred.

## Model plus View (для не-REST)

Классический MVC для traditional web apps:
```java
@Controller
class OrderController {
    @GetMapping("/orders")
    public String list(Model model) {
        model.addAttribute("orders", service.findAll());
        return "orders";   // orders.html (Thymeleaf) / orders.jsp
    }
}
```

ViewResolver ищет template. InternalResourceViewResolver (JSP). ThymeleafViewResolver. FreeMarkerViewResolver.

Redirect:
```java
return "redirect:/orders";
return "forward:/other";
```

Для микросервисов (REST APIs) — не используется. Frontend отдельно (React, Vue, Angular).

## ResponseEntity

Полный контроль над response:
```java
@GetMapping("/orders/{id}")
public ResponseEntity<Order> get(@PathVariable Long id) {
    Order o = service.find(id);
    if (o == null) return ResponseEntity.notFound().build();
    return ResponseEntity.ok()
        .header("X-Order-Version", o.getVersion().toString())
        .cacheControl(CacheControl.maxAge(30, TimeUnit.SECONDS))
        .body(o);
}

@PostMapping("/orders")
public ResponseEntity<Order> create(@RequestBody OrderDto dto) {
    Order created = service.create(dto);
    URI location = URI.create("/api/orders/" + created.getId());
    return ResponseEntity.created(location).body(created);
}
```

Более гибко чем просто @ResponseBody. Explicit status codes, headers, body.

## WebMvcConfigurer

Central point для customization:
```java
@Configuration
class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) { ... }

    @Override
    public void addInterceptors(InterceptorRegistry registry) { ... }

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) { ... }

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) { ... }

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) { ... }

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) { ... }
}
```

Extension interface для customizing Spring MVC. Override только необходимые methods.

## WebFlux reactive alternative

Не Spring MVC, но связано.

MVC = Servlet-based, thread-per-request, blocking API.

WebFlux = Reactive, event-loop, Mono/Flux, non-blocking.

WebFlux не разбираем детально (отдельный мир). ИСНА обычно использует MVC. Different programming paradigm requiring extensive learning.

## Пример полного flow

Complete trace HTTP request:
```
1. Client: POST /api/orders
    Content-Type: application/json
    Body: {"customerId": "c1", "amount": 100}

2. Tomcat принимает соединение
    → передаёт DispatcherServlet

3. Filters (Security, CORS, TraceId) — до DispatcherServlet

4. DispatcherServlet.doDispatch
    → HandlerMapping находит: OrderController.create()
    → Interceptors.preHandle() — если false, stop

5. RequestMappingHandlerAdapter
    → ArgumentResolvers резолвят параметры
       - @RequestBody OrderDto: RequestResponseBodyMethodProcessor
         - Content-Type: application/json
         - MappingJackson2HttpMessageConverter десериализует
       - @Valid: HibernateValidator валидирует
       - если invalid → MethodArgumentNotValidException
    → reflection.invoke(controllerBean, ..., args)

6. Controller выполняется
    return service.create(dto);

7. ReturnValueHandlers обрабатывают
    → @ResponseBody: RequestResponseBodyMethodProcessor
      → Accept: application/json
      → MappingJackson2HttpMessageConverter сериализует Order → JSON
      → пишет в response body
      → Content-Type: application/json устанавливается

8. Interceptors.postHandle()

9. DispatcherServlet возвращает

10. Filters (обратный порядок)

11. Response клиенту
```

Understanding this flow enables predicting behavior at each step. Debugging becomes systematic.

## Production caveats

Timeouts:
```yaml
server:
  tomcat:
    connection-timeout: 30s
    keep-alive-timeout: 60s
spring:
  mvc:
    async:
      request-timeout: 30s
```

Explicit timeouts vs defaults (often unlimited).

Body size limits:
```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 20MB
server:
  tomcat:
    max-http-form-post-size: 10MB
    max-swallow-size: 10MB
```

Prevent runaway uploads consuming memory или disk.

Not respecting @Transactional. Controller НЕ должен иметь @Transactional. Транзакция — на сервисе. Иначе. Transaction открывается до сериализации — долгая tx. LazyInit exceptions при сериализации. Сложнее тестировать.

Return Entity → LazyInit issue:
```java
@GetMapping("/orders/{id}")
public Order get(@PathVariable Long id) {
    return orderRepo.findById(id).orElseThrow();
}
```

Entity — Jackson пытается сериализовать все fields (включая lazy) — LazyInit exception. Session closed после service method returns.

Fix — возвращать DTO, не Entity. Convert inside service (within transaction) to plain DTO. DTO safely serializable.

CORS configuration:
```java
@RestController
@CrossOrigin(origins = "https://knp.kgd.gov.kz")
class OrderController { }
```

Или global через WebMvcConfigurer.addCorsMappings. Prevents CORS-related failures когда frontend на different origin.

## Итоги

DispatcherServlet — front controller. Все URL через него. Регистрируется Spring Boot автоматически.

HandlerMapping определяет какой метод обрабатывает URL. RequestMappingHandlerMapping основной. Multi-dimensional matching (URL, method, headers, params).

HandlerAdapter вызывает handler. RequestMappingHandlerAdapter стандартный. Coordinates ArgumentResolvers plus ReturnValueHandlers.

ArgumentResolvers резолвят каждый параметр — @PathVariable, @RequestParam, @RequestBody plus custom. Chain of responsibility.

ReturnValueHandlers обрабатывают response. Convert к JSON через HttpMessageConverter. Handle async types (Callable, DeferredResult, CompletableFuture).

HttpMessageConverter — JSON/XML serialization/deserialization. Jackson default. Content negotiation через Accept plus Content-Type.

Interceptors vs Filters. Filter уровень Servlet до DispatcherServlet. Interceptor уровень MVC до/после handler. Different scopes и capabilities.

Exception handling через @ExceptionHandler (local) plus @ControllerAdvice (global). ResponseStatusException для quick throws. ProblemDetail для RFC 7807 standard error format.

Custom ArgumentResolver для encapsulating common parameter patterns. Cleaner controllers.

Async controllers через Callable, DeferredResult, CompletableFuture, SseEmitter. Virtual Threads (Java 21+) automatic async без wrapper types.

Bean Validation через @Valid. Custom validators для business-specific rules.

ResponseEntity для полного control response. Status codes, headers, body.

WebMvcConfigurer central customization point. Override только needed methods.

Production caveats. Explicit timeouts. Body size limits. No @Transactional on controller. Return DTO not Entity (LazyInit). CORS configuration.

Понимание internals enables predicting behavior plus effective troubleshooting. Not magic — well-defined mechanisms.

Дальше — REST API theory с Fielding constraints, Richardson maturity model, HTTP semantics, URI design, pagination.
