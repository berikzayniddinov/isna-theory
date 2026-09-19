# 52. Resilience patterns в микросервисах: Circuit Breaker, Retry, Bulkhead, Rate Limit

## Зачем понимать resilience глубоко

Разработчик который только начинает работать с микросервисами обычно думает про resilience как о something to «добавить потом когда работает». Reality — distributed systems fundamentally unreliable. Networks fail. Services crash. Databases become slow. External APIs return errors. Не заложить resilience patterns с самого начала означает построить fragile system которая ломается при первой transient проблеме.

Разница между разработчиком «использующим Circuit Breaker» и «понимающим resilience» проявляется в production stability. Первый добавляет @CircuitBreaker к отдельным methods, надеется что достаточно. Второй знает fundamental proble — cascading failure. Один сервис падает медленно. Callers wait для timeout. Their thread pools забиваются. Их callers wait. Chain reaction. Целый system down. Знает что защита требует layered approach — timeout как absolute minimum, plus retry для transient issues, plus circuit breaker для sustained problems, plus bulkhead для resource isolation, plus rate limiting для controlling input, plus graceful degradation для user experience.

В этом файле разберём resilience patterns comprehensively. Fundamental проблема — cascading failure. Timeout — first line of defense. Retry с exponential backoff и jitter. Circuit Breaker mechanics — three states, thresholds, configuration. Bulkhead для resource isolation. Rate Limiting algorithms. Fallback strategies. Deadline / cancellation propagation. Health checks правильные. Feature flags как runtime kill switches. Graceful shutdown. Chaos engineering. Production configuration полная с Resilience4j. Correct ordering annotations (важно). ИСНА real cases.

## Fundamental problem: cascading failure

Микросервисы это distributed system. Сеть ненадёжна. Services fail. Latency variable. Understanding this reality shapes всю resilience design.

Классический пример:
```
Service A ──sync HTTP──► Service B (медленный)
                              │
                              ▼
                        Service C (умер)
```

Что происходит step-by-step. Service C умер (crash, network partition, whatever). B calls C — не получает ответ. B waits для read timeout (default может быть 30 seconds или больше). Пока B ждёт — его request thread blocked. Same happens для all concurrent requests к B — thread pool забивается.

A ждёт B. Thread pool A тоже блокируется waiting. Clients A ждут responses.

Result — cascading failure. Один downstream problem cascades всё upstream. Whole system down несмотря на что root cause это just one service degraded.

Это common pattern в production incidents. Sometimes traced к single downstream that got slow, потом weeks debugging показывает full impact chain.

Защита — набор resilience patterns applied layered. No single pattern sufficient — combination обеспечивает reliability.

## Timeout: первая линия защиты

Первое и самое важное patterns. Правило simple — никогда без timeout. Default в большинстве библиотек равно бесконечно — плохо.

Что таймаутить. HTTP connect timeout (TCP connection establishment). HTTP read timeout (waiting for response). DB connection timeout (getting connection from pool). DB statement timeout (query execution time limit). Kafka producer send timeout. Rabbit publish timeout. Message consumer processing time.

Значения guidelines.

Connect: 1-5 seconds. Fast fail когда downstream unreachable. TCP handshake должен complete быстро — если dovlyc — network problem.

Read for user-facing: 5-30 seconds. Balance между allowing legitimate slow operations и fast failing bad ones.

Read для background jobs: до нескольких minutes. Batch operations могут требовать longer duration.

Правило. Timeout < timeout вызывающего. Иначе caller уже отвалился, а мы ещё ждём. Waste ресурсов plus confusion. Deadline propagation (см. ниже) более principled approach.

Spring RestClient / WebClient configuration:
```java
RestClient client = RestClient.builder()
    .requestFactory(new HttpComponentsClientHttpRequestFactory(
        HttpClientBuilder.create()
            .setDefaultRequestConfig(RequestConfig.custom()
                .setConnectTimeout(Timeout.ofSeconds(5))
                .setResponseTimeout(Timeout.ofSeconds(30))
                .build())
            .build()))
    .build();
```

Feign configuration через application.yml:
```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 30000
```

HikariCP уже обсуждали в файле 29:
```yaml
spring.datasource.hikari:
  connection-timeout: 10000
```

## Retry с exponential backoff и jitter

При временной ошибке — попробовать снова. Not всегда appropriate но powerful в правильных cases.

Когда retry OK. Network glitch — transient issue that may pass. Timeout — connection dropped, retry might succeed. HTTP 503 Service Unavailable — server temporarily overloaded. Rate limit (429) — с backoff, respect Retry-After header. Deadlock БД — concurrent contention resolved through retry.

Когда retry НЕ OK. 400 Bad Request — request malformed, won't become valid. 401 Unauthorized — token won't fix itself. 404 Not Found — resource won't appear. Business validation errors — semantically failed.

Exponential backoff. Не сразу retry — ждать увеличивающееся время:
```
Попытка 1: сразу
Попытка 2: ждать 1s
Попытка 3: ждать 2s
Попытка 4: ждать 4s
Попытка 5: ждать 8s
```

Doubling backoff intervals. Prevents overwhelming downstream still recovering. Balances persistence с backing off.

Плюс jitter (случайность):
```
delay = base * 2^attempt + random(0, base)
```

Зачем jitter. Если 1000 клиентов одновременно retry без jitter — все повторят одновременно → усугубят перегрузку. Thundering herd. Jitter spreads retries randomly — smoother load pattern.

Idempotency обязательна. Retry без idempotency = дубли. Пример POST /transfer — retry задваивает перевод. Money moved twice. Financial correctness violated.

Решения. Idempotency-Key header — server dedupes based на key. Idempotent semantics операции (state machine transitions). Ensures repeat safety.

Spring Retry через AOP:
```gradle
implementation 'org.springframework.retry:spring-retry'
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

```java
@Configuration
@EnableRetry
class Config {}

@Service
class ExternalClient {

    @Retryable(
        retryFor = {IOException.class, TimeoutException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000)
    )
    public String call() { ... }

    @Recover
    public String recover(Exception e) {
        return "fallback";
    }
}
```

@Retryable configuration retry attempts. @Recover method invoked when retries exhausted — fallback logic. Transparent для caller — retries happen через AOP interception.

Resilience4j Retry alternative approach:
```java
Retry retry = Retry.of("myService", RetryConfig.custom()
    .maxAttempts(3)
    .waitDuration(Duration.ofSeconds(1))
    .retryOnException(e -> e instanceof IOException)
    .build());

String result = retry.executeSupplier(() -> externalClient.call());
```

Functional programming style. More explicit control. Composable с other Resilience4j patterns.

## Circuit Breaker подробно

Защита от каскадных отказов. Один of most important patterns для microservices.

Идея. Каждый вызов downstream — статистика (успех/провал). Если провалов много — разомкнуть цепь — новые вызовы fail fast без обращения к downstream. Через N времени — полу-открытый — пробный вызов — если ok замкнуть, если нет снова открыть.

Три состояния circuit breaker:
```
                  timeout/many errors
   ┌────────┐  ─────────────────►  ┌────────┐
   │ CLOSED │                       │  OPEN  │
   │        │  ◄─────────────────  │        │
   └────┬───┘   success in trial   └───┬────┘
        │                              │
        │ many errors                  │ timer (wait duration)
        │                              │
        │                              ▼
        │                        ┌─────────────┐
        │                        │  HALF-OPEN  │
        │                        │  (trial)    │
        │                        └─────────────┘
        │                              │
        │                              │ success
        └──────────────────────────────┘
```

CLOSED — normal, все вызовы идут в downstream. Statistics tracking outcomes.

OPEN — все вызовы fail fast (не идут в downstream). Immediate failure returned to caller. Reduces load on failing downstream. Wait duration passes.

HALF-OPEN — пробуем один-два запроса. Если ok — CLOSED (recovery detected). Если нет — снова OPEN (still failing).

Параметры важны.

Failure rate threshold — сколько процентов failures triggers OPEN (обычно 50%).

Minimum calls — минимум вызовов для оценки. Prevents opening на one или two errors. Statistical significance требуется.

Sliding window — окно оценки. Count-based (last N calls) или time-based (last N seconds).

Wait duration — сколько ждать в OPEN до HALF-OPEN transition (обычно 30 seconds).

Permitted calls в HALF-OPEN — сколько test requests allow (typically 3).

Resilience4j configuration:
```yaml
resilience4j.circuitbreaker:
  instances:
    userService:
      failure-rate-threshold: 50
      minimum-number-of-calls: 10
      sliding-window-type: COUNT_BASED
      sliding-window-size: 20
      wait-duration-in-open-state: 30s
      permitted-number-of-calls-in-half-open-state: 3
```

Usage:
```java
@Service
class UserClient {

    @CircuitBreaker(name = "userService", fallbackMethod = "fallback")
    public UserDto getUser(Long id) {
        return restClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .body(UserDto.class);
    }

    public UserDto fallback(Long id, Throwable t) {
        return UserDto.empty();
    }
}
```

При OPEN — сразу вызывается fallback без attempt к downstream.

Monitoring. Actuator endpoint /actuator/circuitbreakers показывает состояние всех CB. Metrics в Prometheus — resilience4j_circuitbreaker_state, resilience4j_circuitbreaker_calls. Alert on sustained OPEN state — indicates unresolved downstream problem.

## Bulkhead: resource isolation

Изоляция ресурсов между зависимостями. Название от корабельных переборок (bulkhead) — если один отсек затопило, остальные держат корабль на плаву.

Проблема. Downstream A медленный — thread pool у caller забит запросами к A — downstream B получить thread не могу — B tratil become slow тоже.

Решение — разные thread pools или semaphores для разных зависимостей:
```
Caller
  │
  ├─ Pool for downstream A (max 10 concurrent)
  ├─ Pool for downstream B (max 20)
  └─ Pool for downstream C (max 5)
```

Если A завис — только 10 threads залипло. B, C работают normally. Failure contained к specific pool.

Resilience4j Bulkhead configuration:
```yaml
resilience4j.bulkhead:
  instances:
    userService:
      max-concurrent-calls: 10
      max-wait-duration: 100ms
```

```java
@Bulkhead(name = "userService", type = Bulkhead.Type.SEMAPHORE)
public UserDto getUser(Long id) { ... }
```

Два типа bulkhead.

Semaphore — просто счётчик, легковесный. Каждый call decrements counter. Zero counter — reject new call. Zero overhead beyond counter tracking.

ThreadPool — отдельный executor per dependency. More heavyweight но provides thread isolation. Each dependency has separate thread resource. True isolation но more memory.

Semaphore usually sufficient. ThreadPool когда downstream can block callers significantly и isolation critical.

## Rate Limiting

Ограничение rate от одного клиента или на API endpoint.

Зачем. Защита от abuse (DDoS, brute force attacks). Защита downstream (не перегрузить). Fair usage (prevent one client hogging resources).

Алгоритмы.

Token bucket. Bucket с N токенами. Каждый запрос — берёт токен. Токены восстанавливаются по времени (например, 10 в секунду). Bucket пустой — отказать или ждать.

Sliding window. Считать запросы за последние N секунд. Больше threshold — отказать. Different from fixed window (counts within specific interval) — smoother behavior.

Resilience4j RateLimiter:
```yaml
resilience4j.ratelimiter:
  instances:
    userService:
      limit-for-period: 100        # запросов за refresh период
      limit-refresh-period: 1s
      timeout-duration: 100ms       # сколько ждать токен
```

```java
@RateLimiter(name = "userService")
public UserDto getUser(Long id) { ... }
```

nginx rate limit (also обсуждался в файле 46):
```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
    }
}
```

10 req/sec per IP, burst 20 (short spikes allowed).

Redis-based distributed rate limiting. Для limits через все реплики:
```java
if (redis.incr("rate:user:42") > 100) {
    throw new RateLimitException();
}
redis.expire("rate:user:42", 60);
```

Много готовых библиотек. Bucket4j — comprehensive rate limiting. Resilience4j плюс Redis — distributed variant.

## Fallback strategies

Что вернуть когда downstream упал. Multiple approaches.

Простой fallback:
```java
public UserDto fallback(Long id, Throwable t) {
    return UserDto.empty();
}
```

Empty response или default value. User sees «empty» state instead of error.

Cached fallback. Хранить последний успешный ответ:
```java
public UserDto fallback(Long id, Throwable t) {
    return cache.get(id);   // last known good
}
```

Stale data preferred over no data. Cache TTL determines staleness tolerance.

Degraded response. Показать что можешь. Список постов есть, автор пусто — показать посты без автора. Каталог есть, рекомендации не работают — показать каталог без рекомендаций.

Graceful degradation. Пользователь получает частичный функционал, но не «сломано всё». Better UX than error page.

Fail loud. Иногда лучше 5xx чем неправильный ответ (финансы, критичные операции). Not все operations acceptable для degraded response — money-related operations should not silently proceed with stale или default data.

Правило — выбирай осознанно based на business impact. Not one-size-fits-all decision.

## Deadline / cancellation propagation

Клиент имеет бюджет времени на запрос — пробрасывать downstream. Advanced pattern но powerful для high-load systems.

Идея. Клиент: «у меня 5 сек». Service A: получил в 1 сек — передаёт «осталось 4 сек» downstream. Service B: получил в 2 сек — передаёт «осталось 3 сек» дальше. Service C: работает не больше 3 сек.

Если C увидит что осталось <500 ms → сразу вернуть, не запускать долгую операцию. Save resources для operations that can complete в time.

Реализация. HTTP header:
```
Deadline: 1704067200000     # unix timestamp когда expires
```

Каждый service — при приёме считать оставшееся, передавать downstream с обновлённым значением, не запускать операцию если бюджет истёк.

gRPC имеет это встроено. HTTP/REST — свой custom header. Practical в systems where deadlines really matter for user experience.

Мало кто делает. But helps в highly-loaded systems where legitimate deadline propagation preserves resources.

## Health checks правильные

Уже разбирали в K8s context (файл 10). Ключевые types.

Liveness — процесс жив. Simplest — accepts TCP connection or responds к endpoint.

Readiness — готов принимать трафик. More sophisticated — checks downstream dependencies, initialization complete.

Startup — для медленных стартов. Delayed liveness/readiness checks until startup complete.

Хорошая readiness должна отражать реальную способность работать. Есть connection к БД? Consul registration OK? Rabbit / Kafka connection?

Плохая readiness. Только return 200 OK без проверок. TCP-only check при open port но application broken.

Пример из КНП knp-form-hz5-actuator-cache-nosuchmethod — TCP-only readiness врала. Приложение сломалось при старте, порт открыт — K8s думал что healthy. Traffic routed к broken instance. Users saw errors несмотря на «healthy» status.

Caveat с DB в readiness. Если БД временно недоступна — readiness DOWN — K8s исключает под из Service — все запросы 503.

Но БД восстановится через минуту. Хотим ли мы «убрать» реплику? Обычно нет. БД temporary issue — приложение продолжает работать (retry / circuit breaker), не должно быть исключено из LB.

Правило. Readiness = «могу принимать НОВЫЕ запросы», а не «работают ВСЕ downstream». Downstream problems handled through circuit breakers, not through pod exclusion.

## Feature flags как kill switches

Runtime переключатели функционала.

Зачем. Постепенное включение фичи (canary для features). Мгновенное отключение проблемной фичи (kill switch). A/B testing different implementations. Различное поведение per user / tenant.

Реализации. LaunchDarkly — SaaS. Unleash — open-source. FF4J — Java. Custom — через БД / Consul KV / ConfigMap.

Пример:
```java
if (featureFlags.isEnabled("new-checkout")) {
    newCheckout(order);
} else {
    oldCheckout(order);
}
```

При проблеме с new-checkout — админ выключает flag — все идут на old. Instant mitigation без redeployment.

Powerful pattern для production safety. Deploy new code disabled, gradually enable через flag flipping, disable instantly if problems detected.

## Graceful shutdown

При SIGTERM (K8s scale down или rolling update).

Stop accepting new requests. Wait for in-flight to finish. Close connections. Deregister from Consul. Exit.

Spring Boot:
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

K8s должен дать время. terminationGracePeriodSeconds: 60 — больше чем timeout-per-shutdown-phase. Ensures Spring shutdown completes before K8s force-kills.

Иначе SIGKILL — in-flight потерялись. Users see errors для requests in progress at shutdown time.

Proper graceful shutdown critical для zero-downtime deployments. Rolling updates depend on it. Users shouldn't see errors when deployments happen.

## Chaos engineering

Как узнать что resilience patterns работают? Ломать намеренно.

Netflix Chaos Monkey. Раз в день случайно убивает production под. Приложение должно пережить. Continuous chaos verifies resilience.

Tools. Chaos Monkey для K8s. Litmus. Gremlin. Chaos Mesh.

Что тестировать. Kill random pod. Slow network to downstream. Return 500 from downstream. Full DB pool. Full disk. Various failure modes.

Если приложение переживает — resilience работает. Discovery bugs before they cause real incidents.

Начинай в staging. Только зрелые системы — в production. Chaos in production requires substantial maturity и monitoring.

## Полная production конфигурация

Собранная воедино resilience configuration:
```yaml
resilience4j:
  circuitbreaker:
    instances:
      userService:
        failure-rate-threshold: 50
        minimum-number-of-calls: 10
        sliding-window-size: 20
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
  retry:
    instances:
      userService:
        max-attempts: 3
        wait-duration: 1s
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.io.IOException
          - org.springframework.web.client.ResourceAccessException
  bulkhead:
    instances:
      userService:
        max-concurrent-calls: 10
  timelimiter:
    instances:
      userService:
        timeout-duration: 5s

management:
  endpoints.web.exposure.include: health,circuitbreakers,retries,ratelimiters,bulkheads
  health:
    circuitbreakers.enabled: true
```

Combining multiple patterns via annotations:
```java
@Service
class UserClient {

    private final RestClient client;

    @CircuitBreaker(name = "userService", fallbackMethod = "fallback")
    @Retry(name = "userService")
    @Bulkhead(name = "userService")
    @TimeLimiter(name = "userService")
    public CompletableFuture<UserDto> getUser(Long id) {
        return CompletableFuture.supplyAsync(() ->
            client.get().uri("/users/{id}", id).retrieve().body(UserDto.class));
    }

    public CompletableFuture<UserDto> fallback(Long id, Throwable t) {
        log.warn("Fallback for user {}: {}", id, t.getMessage());
        return CompletableFuture.completedFuture(UserDto.empty());
    }
}
```

## Correct ordering annotations

Order важен. @CircuitBreaker снаружи, @Retry внутри. Иначе CB считает каждый retry attempt как отдельный вызов — CB opens после fewer actual failures than expected.

Correct order (outer to inner). CircuitBreaker — outermost. Retry — inside. Bulkhead — around actual call. TimeLimiter — closest к actual.

Wrong order examples. Retry outermost — retries увеличивают calls seen by CB, CB может open предварительно. TimeLimiter вне Retry — retries not counted against overall time limit.

Resilience4j documentation specifies correct nesting. Follow guidance.

## Каскад: что и когда

Правильный стек на каждый downstream call:

1. Timeout — first line of defense. Absolute minimum.

2. Retry для transient errors. Handles glitches.

3. Circuit Breaker для длительных проблем. Fast fail during outages.

4. Bulkhead для isolation. Prevent one dependency taking whole thread pool.

5. Rate Limit на входе (input control). Prevent abuse.

6. Fallback для user-friendly degradation. Graceful UX during failures.

Не все нужны везде. Timeout — обязательно. Others — по situation.

## ИСНА real cases

Из memory. Sync HTTP chains — potential cascading failure. Circuit breakers required.

Feign clients — timeout configuration explicit. Feign default timeouts плохие для production.

HikariCP — connection timeout, leak detection threshold. Prevent pool exhaustion.

@Transactional patterns — external calls вне transaction. Don't hold connections during network waits.

## Итоги

Cascading failure fundamental risk в distributed systems. Layered defense через resilience patterns.

Timeout обязательно на всё. Никогда default (usually бесконечно). Fast fail preferred.

Retry с exponential backoff plus jitter для transient. Idempotency обязательна. Not для всех exception types.

Circuit Breaker для длительных проблем downstream. Three states — CLOSED, OPEN, HALF-OPEN. Fast fail в OPEN state. Recovery detection через HALF-OPEN.

Bulkhead для isolation ресурсов. Semaphore lightweight. ThreadPool для true isolation.

Rate Limit на входе. Token bucket или sliding window algorithms. Protects downstream, prevents abuse.

Fallback plus graceful degradation. Empty response, cached data, degraded functionality. Fail loud для critical operations.

Deadline propagation в highly-loaded systems. Explicit budgets flow downstream.

Health checks правильные. Liveness plus readiness plus startup. Real dependency checks, not just TCP open. But not too coupled к downstream либо всё falls over together.

Feature flags как runtime kill switches. Instant mitigation через flag flipping.

Graceful shutdown обязательно. SIGTERM handling. Zero-downtime deployments.

Chaos engineering для verification. Verify resilience through controlled failure injection.

Resilience4j — стандарт в Java. Hystrix deprecated.

Correct annotation ordering matters. CircuitBreaker outermost, TimeLimiter innermost.

Layered approach — Timeout plus Retry plus CB plus Bulkhead plus Rate Limit plus Fallback. Combined provides robust resilience.

Дальше — блок testing. Unit testing detailed с JUnit, Mockito, AssertJ.
