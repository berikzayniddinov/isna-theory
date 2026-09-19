# 52. Паттерны надёжности микросервисов

Circuit Breaker, Retry, Timeout, Bulkhead, Rate Limit, Fallback. Как не упасть каскадно.

---

## 1. Проблема: cascading failure

Микросервисы = distributed system. Сеть ненадёжна.

Пример:
```
Service A ──sync HTTP──► Service B (медленный)
                              │
                              ▼
                        Service C (умер)
```

Что происходит:
1. Service C умер → B не получает ответ → таймаутится через 30 сек.
2. Пока B ждёт — его thread pool забит.
3. A ждёт B → thread pool A тоже забит.
4. Клиенты A ждут → **весь стек упал**.

**Cascading failure**. Одна проблема снежным комом валит всё.

Защита — набор resilience patterns.

---

## 2. Timeout

**Первое и самое важное**.

Правило: **никогда без timeout**. Default в большинстве библиотек = **бесконечно** — плохо.

### 2.1 Что таймаутить

- HTTP connect timeout (установление TCP).
- HTTP read timeout (ожидание ответа).
- DB connection timeout.
- DB statement timeout.
- Kafka producer send timeout.
- Rabbit publish timeout.
- Message consumer processing time.

### 2.2 Значения

- **Connect**: 1-5 сек (быстро понять что нет соединения).
- **Read** для user-facing: 5-30 сек.
- **Read** для background jobs: до нескольких минут.

Правило: **timeout < timeout вызывающего**. Иначе caller уже отвалился, а мы ещё ждём.

### 2.3 Spring RestClient / WebClient

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

### 2.4 Feign

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 30000
```

### 2.5 HikariCP

Уже разбирали в `29-postgresql-spring-hikaricp.md`:
```yaml
spring.datasource.hikari:
  connection-timeout: 10000
```

---

## 3. Retry

При временной ошибке — попробовать снова.

### 3.1 Когда retry OK

- Network glitch.
- Timeout (возможно).
- HTTP 503 Service Unavailable.
- Rate limit (429) — с backoff.
- Deadlock БД.

### 3.2 Когда retry НЕ OK

- 400 Bad Request — код не станет валидным.
- 401 Unauthorized — токен не станет валидным сам.
- 404 Not Found — не появится.
- Business validation errors.

### 3.3 Exponential backoff

Не сразу retry — ждать увеличивающееся время:
```
Попытка 1: сразу
Попытка 2: ждать 1s
Попытка 3: ждать 2s
Попытка 4: ждать 4s
Попытка 5: ждать 8s
```

Плюс **jitter** (случайность):
```
delay = base * 2^attempt + random(0, base)
```

Зачем jitter: если 1000 клиентов одновременно retry без jitter — все повторят одновременно → усугубят перегрузку.

### 3.4 Idempotency

**Retry без идемпотентности = дубли**.

Пример: `POST /transfer` — retry задваивает перевод.

Решения:
- Idempotency-Key header.
- Idempotent semantics операции (state machine).

### 3.5 Spring Retry

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

Через AOP — retry прозрачно для caller.

### 3.6 Resilience4j Retry

```java
Retry retry = Retry.of("myService", RetryConfig.custom()
    .maxAttempts(3)
    .waitDuration(Duration.ofSeconds(1))
    .retryOnException(e -> e instanceof IOException)
    .build());

String result = retry.executeSupplier(() -> externalClient.call());
```

---

## 4. Circuit Breaker

Защита от каскадных отказов.

### 4.1 Идея

Каждый вызов downstream — статистика (успех/провал). Если провалов много → **разомкнуть цепь** — новые вызовы **fail fast** без обращения к downstream.

Через N времени → полу-открытый → пробный вызов → если ок → замкнуть; если нет → снова открыть.

### 4.2 Три состояния

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

- **CLOSED** — normal, все вызовы идут в downstream.
- **OPEN** — все вызовы **fail fast** (не идут в downstream).
- **HALF-OPEN** — пробуем один-два запроса; если ок → CLOSED; если нет → OPEN.

### 4.3 Параметры

- **Failure rate threshold** — сколько % failures → OPEN (обычно 50%).
- **Minimum calls** — минимум вызовов для оценки (иначе одна ошибка = OPEN).
- **Sliding window** — окно оценки (count-based или time-based).
- **Wait duration** — сколько ждать в OPEN до HALF-OPEN.
- **Permitted calls in HALF-OPEN** — сколько тестовых.

### 4.4 Resilience4j

```gradle
implementation 'io.github.resilience4j:resilience4j-spring-boot3'
```

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

При OPEN → сразу вызывается `fallback`.

### 4.5 Мониторинг

Actuator endpoint `/actuator/circuitbreakers` показывает состояние всех CB.

Метрики → Prometheus:
- `resilience4j_circuitbreaker_state`.
- `resilience4j_circuitbreaker_calls`.
- Alert: state = OPEN.

---

## 5. Bulkhead

**Изоляция ресурсов** между зависимостями.

### 5.1 Проблема

Downstream A медленный → thread pool у caller забит запросами к A → downstream B получить не могу.

### 5.2 Решение

Разные thread pools / semaphores для разных зависимостей:

```
Caller
  │
  ├─ Pool for downstream A (max 10 concurrent)
  ├─ Pool for downstream B (max 20)
  └─ Pool for downstream C (max 5)
```

Если A завис — только 10 threads залипло. B, C работают.

Название от корабельных переборок (bulkhead) — если один отсек затопило, остальные держат корабль на плаву.

### 5.3 Resilience4j Bulkhead

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

Два типа:
- **Semaphore** — просто счётчик, легковесный.
- **ThreadPool** — отдельный executor per dependency, тяжелее.

---

## 6. Rate Limiting

Ограничение rate от одного клиента или на API endpoint.

### 6.1 Зачем

- Защита от abuse (DDoS, brute force).
- Защита downstream (не перегрузить).
- Fair usage.

### 6.2 Алгоритмы

**Token bucket**:
- Bucket с N токенами.
- Каждый запрос — берёт токен.
- Токены восстанавливаются по времени (например, 10 в секунду).
- Bucket пустой → отказать / ждать.

**Sliding window**:
- Считать запросы за последние N секунд.
- Больше threshold → отказать.

### 6.3 Resilience4j RateLimiter

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

### 6.4 nginx rate limit

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
    }
}
```

10 req/sec per IP, burst 20.

### 6.5 Redis-based distributed

Для limits через все реплики:

```java
// использует Redis для counter
if (redis.incr("rate:user:42") > 100) {
    throw new RateLimitException();
}
redis.expire("rate:user:42", 60);
```

Много готовых библиотек: Bucket4j, Resilience4j + Redis.

---

## 7. Fallback

Что вернуть когда downstream упал.

### 7.1 Простой fallback

```java
public UserDto fallback(Long id, Throwable t) {
    return UserDto.empty();
}
```

Пустой ответ / default.

### 7.2 Cached fallback

Хранить последний успешный ответ:
```java
public UserDto fallback(Long id, Throwable t) {
    return cache.get(id);   // last known good
}
```

### 7.3 Degraded response

Показать что можешь:
- Список постов есть, автор пусто → показать посты без автора.
- Каталог есть, рекомендации не работают → показать каталог без рекомендаций.

**Graceful degradation** — пользователь получает частичный функционал, но не «сломано всё».

### 7.4 Fail loud

Иногда лучше 5xx чем неправильный ответ (финансы, критичные операции).

Правило: **выбирай осознанно**.

---

## 8. Deadline / cancellation propagation

Клиент имеет **бюджет времени** на запрос → пробрасывать downstream.

### 8.1 Идея

Клиент: «у меня 5 сек».
Service A: получил в 1 сек → передаёт «осталось 4 сек» downstream.
Service B: получил в 2 сек → передаёт «осталось 3 сек» дальше.
Service C: работает не больше 3 сек.

Если C увидит что осталось <500 ms → сразу вернуть, не запускать долгую операцию.

### 8.2 Реализация

HTTP header:
```
Deadline: 1704067200000     # unix timestamp когда expires
```

Каждый service:
- При приёме — считать оставшееся.
- Передавать downstream с обновлённым значением.
- Не запускать операцию если бюджет истёк.

**gRPC** имеет это встроено. HTTP/REST — свой custom header.

### 8.3 Практика

Мало кто делает. Но помогает в высоко-нагруженных системах.

---

## 9. Health checks

Уже разбирали в K8s. Ключевые:

- **Liveness** — процесс жив.
- **Readiness** — готов принимать трафик.
- **Startup** — для медленных стартов.

### 9.1 Хорошая readiness

Должна отражать реальную способность работать:
- Есть connection к БД?
- Consul registration OK?
- Rabbit / Kafka connection?

**Плохая readiness**: только `return 200 OK` без проверок.

Пример из ИСНА `knp-form-hz5-actuator-cache-nosuchmethod` — TCP-only readiness врала. Приложение сломалось при старте, порт открыт → K8s думал что healthy.

### 9.2 Кавет с DB in readiness

Если БД временно недоступна → readiness DOWN → K8s исключает под из Service → все запросы 503.

Но БД восстановится через минуту. Хотим ли мы «убрать» реплику?

Обычно **нет**. БД temporary issue → приложение продолжает работать (retry / circuit breaker), не должно быть исключено из LB.

Правило: **readiness = «могу принимать НОВЫЕ запросы»**, а не «работают ВСЕ downstream».

---

## 10. Feature flags

Runtime переключатели функционала.

### 10.1 Зачем

- Постепенное включение фичи (canary для features).
- Мгновенное отключение проблемной фичи (kill switch).
- A/B testing.
- Различное поведение per user / tenant.

### 10.2 Реализации

- **LaunchDarkly** — SaaS.
- **Unleash** — open-source.
- **FF4J** — Java.
- **Custom** — через БД / Consul KV / ConfigMap.

### 10.3 Пример

```java
if (featureFlags.isEnabled("new-checkout")) {
    newCheckout(order);
} else {
    oldCheckout(order);
}
```

При проблеме с new-checkout → админ выключает flag → все идут на old.

---

## 11. Graceful shutdown

При SIGTERM (K8s scale down / rolling update):
1. **Stop accepting new requests**.
2. **Wait** for in-flight to finish.
3. **Close** connections.
4. **Deregister** from Consul.
5. **Exit**.

Spring Boot:
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

K8s должен дать время (`terminationGracePeriodSeconds: 60` — больше чем timeout-per-shutdown-phase).

Иначе SIGKILL → in-flight потерялись.

---

## 12. Chaos engineering

Как узнать что resilience patterns работают? **Ломать намеренно**.

### 12.1 Netflix Chaos Monkey

Раз в день случайно убивает production под. Приложение должно пережить.

### 12.2 Tools

- **Chaos Monkey** для K8s.
- **Litmus**.
- **Gremlin**.

### 12.3 Что тестировать

- Kill random pod.
- Slow network to downstream.
- Return 500 from downstream.
- Full DB pool.
- Full disk.

Если приложение переживает — resilience работает.

Начинай в **staging**, только зрелые — в prod.

---

## 13. Пример полной конфигурации

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

Порядок аннотаций matters: `@CircuitBreaker` снаружи, `@Retry` внутри (иначе CB считает retry как отдельные вызовы).

---

## 14. Каскад: что и когда

**Правильный стек** на каждый downstream call:

1. **Timeout** — first line of defense.
2. **Retry** для transient errors.
3. **Circuit Breaker** для длительных проблем.
4. **Bulkhead** для isolation.
5. **Rate Limit** на входе (input control).
6. **Fallback** для user-friendly degradation.

Не все нужны везде. **Timeout — обязательно**, остальное — по situation.

---

## 15. Собесные вопросы

1. **Что такое cascading failure?** — Один сервис упал → thread pools вверх по цепочке забиты → все упали.
2. **Как избежать?** — Timeout + Circuit Breaker + Bulkhead + Retry с backoff.
3. **Что такое Circuit Breaker?** — Защита от повторных вызовов упавшего downstream; 3 состояния CLOSED/OPEN/HALF_OPEN.
4. **Когда retry OK, когда нет?** — OK при transient (network, 503, timeout); НЕ OK при 400/401/404, business errors.
5. **Зачем exponential backoff + jitter?** — Ждать растущее время; jitter — избежать «thundering herd» при массовом retry.
6. **Что такое Bulkhead?** — Изоляция ресурсов (threads) между зависимостями.
7. **Что такое Rate Limit?** — Ограничение запросов; token bucket / sliding window.
8. **Что такое Fallback?** — Что вернуть когда downstream упал (empty, cached, degraded response).
9. **Что такое graceful shutdown?** — SIGTERM → stop new requests → wait in-flight → close → exit.
10. **Разница Hystrix и Resilience4j?** — Hystrix (Netflix, deprecated); Resilience4j (современная замена, functional style).
11. **Порядок аннотаций CB + Retry?** — CircuitBreaker снаружи, Retry внутри (иначе retry считается как отдельные вызовы CB).
12. **Что такое chaos engineering?** — Намеренная поломка prod (Chaos Monkey) чтобы проверить resilience.
13. **Что такое feature flags?** — Runtime переключатели функционала (kill switch, canary, A/B).
14. **Deadline propagation?** — Пробрасывать оставшийся бюджет времени downstream (gRPC встроено, HTTP через header).
15. **Timeout — самое важное — почему?** — Без timeout любой freeze downstream → thread pool залипает → cascading failure.

---

## Итог

- **Timeout** обязательно на всё.
- **Retry** с exponential backoff + jitter для transient.
- **Circuit Breaker** для длительных проблем downstream.
- **Bulkhead** для isolation ресурсов.
- **Rate Limit** на входе.
- **Fallback + graceful degradation** для UX.
- **Health checks** правильные (не только TCP).
- **Graceful shutdown** обязательно.
- **Resilience4j** — стандарт в Java.
- **Chaos engineering** для проверки в prod.

Следующий — блок testing (`53-testing-unit.md`).
