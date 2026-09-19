# 61. Spring MVC контроллеры под капотом

Как обрабатывается HTTP-запрос в Spring MVC. Полная цепочка изнутри.

---

## 1. Общая картина

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

Разберём каждый компонент.

---

## 2. DispatcherServlet — front controller

Единственный Servlet, регистрируемый Spring Boot в servlet-контейнере. Обрабатывает **все URL** приложения.

### 2.1 Что делает

`doDispatch(HttpServletRequest, HttpServletResponse)`:

```
1. getHandler(request)              → HandlerExecutionChain (handler + interceptors)
2. getHandlerAdapter(handler)       → HandlerAdapter
3. interceptors.preHandle()          → если false — stop
4. ha.handle(request, response, handler)   → ModelAndView (или null для REST)
5. interceptors.postHandle()
6. render (ViewResolver + View) или уже отписано в response
7. interceptors.afterCompletion()   → всегда, даже при exception
```

При exception:
- Ищется `HandlerExceptionResolver` (для `@ExceptionHandler`).
- Если найден → обрабатывает.
- Если нет → пробрасывает контейнеру → 500.

### 2.2 Регистрация

Spring Boot автоматически через `DispatcherServletAutoConfiguration`:
- Регистрирует `DispatcherServlet` на `/` (по default).
- Настраивает через `spring.mvc.*` properties.

Можно поменять URL:
```yaml
spring.mvc.servlet.path: /api
```

---

## 3. HandlerMapping

Определяет: **какой handler обрабатывает URL**.

### 3.1 Основной — RequestMappingHandlerMapping

При старте сканирует все `@RequestMapping` / `@GetMapping` / etc в `@Controller` / `@RestController`. Строит **карту**:

```
GET  /api/orders           → OrderController.list()
POST /api/orders           → OrderController.create()
GET  /api/orders/{id}      → OrderController.get(Long)
DELETE /api/orders/{id}    → OrderController.delete(Long)
```

При запросе — ищет match по URL + method + headers + params.

### 3.2 Другие HandlerMapping

- **BeanNameUrlHandlerMapping** — legacy (bean name = URL).
- **SimpleUrlHandlerMapping** — явные mappings.
- **WebMvcConfigurer.addResourceHandlers()** — статические ресурсы.

Порядок: Spring перебирает handlerMappings по `@Order`, первый нашёл — выигрывает.

### 3.3 Как выбирается лучший match

```
GET /api/orders/1

Кандидаты:
  GET /api/orders/{id}
  GET /api/orders/*
  GET /**
```

Приоритеты:
- Более специфичный path выигрывает.
- Exact match > pattern.
- Meta-request info (headers, params) — тоже влияет.

---

## 4. HandlerAdapter

Вызывает handler.

### 4.1 Основной — RequestMappingHandlerAdapter

Работает с `HandlerMethod` (метод + инстанс controller'а).

Основная логика:
1. Резолвить **аргументы** метода через **HandlerMethodArgumentResolver**.
2. Вызвать метод через reflection.
3. Обработать **возвращаемое значение** через **HandlerMethodReturnValueHandler**.

### 4.2 Другие

- **HttpRequestHandlerAdapter** — для `HttpRequestHandler`.
- **SimpleControllerHandlerAdapter** — legacy `Controller`.

99% работы — `RequestMappingHandlerAdapter`.

---

## 5. HandlerMethodArgumentResolver

Резолвит каждый параметр handler-метода.

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

Каждый параметр обрабатывает **свой** resolver:

- **PathVariableMethodArgumentResolver** — `@PathVariable`.
- **RequestParamMethodArgumentResolver** — `@RequestParam`.
- **RequestHeaderMethodArgumentResolver** — `@RequestHeader`.
- **RequestBodyMethodProcessor** — `@RequestBody` (использует HttpMessageConverter).
- **ServletRequestMethodArgumentResolver** — `HttpServletRequest` / `HttpServletResponse`.
- **ModelAttributeMethodProcessor** — `@ModelAttribute`.
- **PrincipalMethodArgumentResolver** — `Principal`.
- **AuthenticationPrincipalArgumentResolver** — `@AuthenticationPrincipal`.

Плюс кастомные (см. §12).

### 5.1 Как выбирается resolver

Каждый resolver имеет `supportsParameter(MethodParameter)`. Spring перебирает — первый `true` выигрывает.

---

## 6. HandlerMethodReturnValueHandler

Обрабатывает возвращаемое значение метода.

Примеры:
```java
public String list() { return "orders"; }              // → ViewName
public ModelAndView list() { ... }                      // → ModelAndView
public List<Order> list() { ... }                       // → JSON (если @ResponseBody)
public ResponseEntity<Order> get() { ... }              // → ResponseEntity
public Callable<Order> async() { ... }                  // → async servlet
public CompletableFuture<Order> future() { ... }        // → async
public Mono<Order> reactive() { ... }                   // → WebFlux (не MVC)
```

Handlers:
- **RequestResponseBodyMethodProcessor** — `@ResponseBody` / `@RestController`.
- **ViewNameMethodReturnValueHandler** — String → view name.
- **ModelAndViewMethodReturnValueHandler**.
- **HttpEntityMethodProcessor** — `ResponseEntity`.
- **CallableMethodReturnValueHandler** — async.
- **DeferredResultMethodReturnValueHandler** — async.

---

## 7. HttpMessageConverter — JSON/XML

Ключевая часть. Преобразует между Java-объектом и HTTP body.

### 7.1 Как работает

Для `@RequestBody`:
1. Читает `Content-Type` header (например `application/json`).
2. Ищет converter который supports это.
3. Читает body → создаёт объект.

Для `@ResponseBody`:
1. Читает `Accept` header (например `application/json`).
2. Ищет converter.
3. Сериализует объект → пишет в body.
4. Устанавливает `Content-Type`.

### 7.2 Стандартные converters

- **MappingJackson2HttpMessageConverter** — JSON (Jackson).
- **MappingJackson2XmlHttpMessageConverter** — XML.
- **StringHttpMessageConverter** — String.
- **ByteArrayHttpMessageConverter** — byte[].
- **FormHttpMessageConverter** — form-urlencoded.
- **ResourceHttpMessageConverter** — Resource (файлы).
- **AtomFeedHttpMessageConverter** — RSS/Atom.

По default — Jackson (JSON) активен в Spring Boot если `jackson-databind` в classpath.

### 7.3 Настройка Jackson

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

### 7.4 Custom converter

```java
@Configuration
class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new MyCustomConverter());
    }
}
```

---

## 8. Content Negotiation

Spring определяет **какой формат** возвращать.

### 8.1 Strategy

По default:
1. **Accept header** клиента (`Accept: application/json`).
2. **URL suffix** (`/orders.json`) — deprecated.
3. **Parameter** (`?format=json`) — если включено.

Настройка:
```yaml
spring.mvc.contentnegotiation:
  favor-parameter: true
  parameter-name: format
  media-types:
    json: application/json
    xml: application/xml
```

### 8.2 Producing / consuming

```java
@GetMapping(value = "/orders", produces = "application/json")
public List<Order> listJson() { }

@GetMapping(value = "/orders", produces = "application/xml")
public List<Order> listXml() { }

@PostMapping(value = "/orders", consumes = "application/json")
public Order create(@RequestBody OrderDto dto) { }
```

Spring выберет метод по `Accept` / `Content-Type`.

---

## 9. Interceptors

Middleware **уровня Spring MVC** (не сервлета).

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

### 9.1 Interceptors vs Filters

- **Filter** — уровень Servlet API, срабатывает **до** DispatcherServlet.
- **Interceptor** — уровень Spring MVC, срабатывает **после** DispatcherServlet, но до/после handler.

Filters — для cross-cutting **всего** (auth, CORS, gzip).
Interceptors — для MVC-специфичного (аудит handler-специфичный, model manipulation).

---

## 10. Filters

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

Spring Boot автоматически регистрирует `@Component` filter.

Порядок: `@Order` или `FilterRegistrationBean`.

### 10.1 Стандартные filters

Spring Boot регистрирует:
- **CharacterEncodingFilter** — UTF-8.
- **HiddenHttpMethodFilter** — для form-based PUT/DELETE.
- **RequestContextFilter**.
- **Security filters** (если security on classpath).

Spring Security сама — цепочка Filter'ов (см. `24-spring-security-basics.md`).

---

## 11. Exception handling

### 11.1 @ExceptionHandler (in controller)

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

Работает только для этого controller.

### 11.2 @ControllerAdvice (global)

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

Работает для **всех** controllers.

Scoped:
```java
@ControllerAdvice(basePackages = "kz.gov.kgd.isna.knp.api")
```

### 11.3 ResponseStatusException

Быстрый способ бросить с кодом:
```java
if (order == null) {
    throw new ResponseStatusException(HttpStatus.NOT_FOUND, "order not found");
}
```

Без ControllerAdvice.

### 11.4 @ResponseStatus

На exception классе:
```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class OrderNotFoundException extends RuntimeException { }
```

При throw — Spring вернёт 404.

### 11.5 Стандартные problem+json (RFC 7807)

Spring 6 / Boot 3+ поддерживает **`ProblemDetail`** — стандартный формат ошибок:

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

Стандартизация ошибок API.

---

## 12. Custom ArgumentResolver

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

---

## 13. Async controllers

Не блокировать поток на долгие операции.

### 13.1 Callable

```java
@GetMapping("/slow")
public Callable<Order> slow() {
    return () -> {
        Thread.sleep(5000);
        return service.compute();
    };
}
```

Spring:
1. Освобождает Tomcat thread.
2. Запускает Callable на **другом executor** (`spring.mvc.async.request-timeout`).
3. По завершении — resume async servlet, отвечает клиенту.

### 13.2 DeferredResult

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

Полезно для **long polling** — клиент ждёт event.

### 13.3 CompletableFuture

```java
@GetMapping("/orders/{id}")
public CompletableFuture<Order> getAsync(@PathVariable Long id) {
    return CompletableFuture.supplyAsync(() -> service.get(id));
}
```

### 13.4 Server-Sent Events (SSE)

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

Streaming events клиенту.

### 13.5 Virtual Threads (Java 21)

Spring Boot 3.2+ с virtual threads делает всё async автоматически:
```yaml
spring.threads.virtual.enabled: true
```

Каждый request — свой virtual thread; блокирование не съедает carrier.

---

## 14. Validation через @Valid

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

    @Valid   // ← recursive validation
    private List<OrderItemDto> items;
}
```

```java
@PostMapping("/orders")
public Order create(@Valid @RequestBody OrderDto dto) { }
```

При invalid → `MethodArgumentNotValidException` → 400 (handle через `@ControllerAdvice`).

### 14.1 Custom validator

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

---

## 15. RequestMappingInfo (внутреннее представление)

Каждый `@RequestMapping` внутри → `RequestMappingInfo`:
- Patterns (URL).
- Methods.
- Params conditions.
- Headers conditions.
- Consumes / produces.

`RequestMappingHandlerMapping` держит `Map<RequestMappingInfo, HandlerMethod>`.

При запросе:
- Ищет все matching (может быть несколько по разным критериям).
- Выбирает **most specific**.

Отсюда — понимание почему `/orders/{id}` не конфликтует с `/orders/search`.

---

## 16. Model + View (для не-REST)

Классический MVC:
```java
@Controller
class OrderController {
    @GetMapping("/orders")
    public String list(Model model) {
        model.addAttribute("orders", service.findAll());
        return "orders";   // → orders.html (Thymeleaf) / orders.jsp
    }
}
```

**ViewResolver** ищет template:
- `InternalResourceViewResolver` (JSP).
- `ThymeleafViewResolver`.
- `FreeMarkerViewResolver`.

Redirect:
```java
return "redirect:/orders";
return "forward:/other";
```

Для микросервисов (REST APIs) — не используется. Frontend отдельно (React).

---

## 17. ResponseEntity

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

Более гибко чем просто `@ResponseBody`.

---

## 18. WebMvcConfigurer

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

---

## 19. WebFlux — reactive alternative

Не Spring MVC, но связано.

- MVC = **Servlet-based**, thread-per-request, blocking API.
- WebFlux = **Reactive**, event-loop, Mono/Flux, non-blocking.

WebFlux не разбираем детально (отдельный мир, обычно ИСНА использует MVC).

---

## 20. Пример полного flow

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

---

## 21. Production caveats

### 21.1 Timeouts

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

### 21.2 Body size limits

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

### 21.3 Not respecting @Transactional

Controller НЕ должен иметь `@Transactional`. Транзакция — на **сервисе**. Иначе:
- Transaction открывается до сериализации → долгая tx.
- LazyInit exceptions при сериализации.
- Сложнее тестировать.

### 21.4 Return Entity → LazyInit

```java
@GetMapping("/orders/{id}")
public Order get(@PathVariable Long id) {
    return orderRepo.findById(id).orElseThrow();
}
```

Entity → Jackson пытается сериализовать все fields (включая lazy) → LazyInit exception.

**Fix**: возвращать DTO, не Entity.

### 21.5 CORS

```java
@RestController
@CrossOrigin(origins = "https://knp.kgd.gov.kz")
class OrderController { }
```

Или global через `WebMvcConfigurer.addCorsMappings`.

---

## 22. Собесные вопросы

1. **Что такое DispatcherServlet?** — Front controller Spring MVC; регистрируется в servlet-контейнере, обрабатывает все URL.
2. **Что делает HandlerMapping?** — Определяет какой метод controller'а обрабатывает URL.
3. **Что делает HandlerAdapter?** — Вызывает handler (метод controller'а), резолвит аргументы, обрабатывает return.
4. **Что такое ArgumentResolver?** — Резолвит один параметр метода (@PathVariable → PathVariableResolver).
5. **Что такое HttpMessageConverter?** — Java ↔ HTTP body (Jackson JSON, XML).
6. **Content Negotiation — как?** — По Accept header / URL suffix / query param → выбирает converter.
7. **Filter vs Interceptor?** — Filter уровень Servlet (до DispatcherServlet); Interceptor уровень MVC (до/после handler).
8. **Как обработать exception централизованно?** — `@ControllerAdvice` + `@ExceptionHandler`.
9. **ResponseStatusException — когда?** — Быстро вернуть кастомный статус без @ControllerAdvice.
10. **@RestController vs @Controller?** — RestController = Controller + @ResponseBody (JSON everywhere).
11. **Как сделать async controller?** — Return Callable / DeferredResult / CompletableFuture / SseEmitter.
12. **Почему нельзя @Transactional на controller?** — Долгая tx включая сериализацию, LazyInit проблемы; tx на service.
13. **Почему нельзя возвращать Entity из controller?** — LazyInit при сериализации; проще возвращать DTO.
14. **ProblemDetail (RFC 7807)?** — Стандартный формат ошибок API (title, status, detail).
15. **Как настроить Jackson?** — `spring.jackson.*` или `Jackson2ObjectMapperBuilderCustomizer`.

---

## Итог

- **DispatcherServlet** — front controller, все URL через него.
- **HandlerMapping** ищет метод; **HandlerAdapter** вызывает.
- **ArgumentResolvers** + **ReturnValueHandlers** — параметры + response.
- **HttpMessageConverter** для JSON/XML (Jackson).
- **Interceptors** — MVC middleware; **Filters** — Servlet middleware.
- **@ControllerAdvice** + **@ExceptionHandler** — глобальная обработка.
- **Async** через Callable/DeferredResult/CompletableFuture.
- Возвращай **DTO**, не Entity.
- **Tx на сервисе**, не controller.

Следующий — `62-rest-api.md`.
