# 66. Синхронная vs асинхронная обработка

Разница на всех уровнях: вызов метода, HTTP, messaging, thread model.

---

## 1. Что такое sync и async

### 1.1 Sync (синхронный)

Вызывающий **ждёт** пока операция завершится. Пока ждёт — **не делает ничего другого**.

```java
User u = userRepo.findById(id);   // блокируется на JDBC
process(u);                        // выполняется после
```

Поток **заблокирован** на время I/O.

### 1.2 Async (асинхронный)

Вызывающий **запускает** операцию и продолжает работать. Результат получит потом (callback / future / event).

```java
CompletableFuture<User> future = userService.findByIdAsync(id);
// поток свободен, может делать другое
future.thenAccept(u -> process(u));
```

Поток не заблокирован.

---

## 2. Аналогия

**Sync**: заказал кофе в кафе → стоишь у стойки → получаешь → уходишь.

**Async**: заказал → официант дал номер → сел за стол → работаешь → номер зажёгся → идёшь забирать.

Пока ждёшь кофе async — можешь читать, писать, звонить. Sync — только ждать.

---

## 3. Уровни sync/async

Понятия применимы к разным уровням:

- **Метод (function call)** — blocking vs non-blocking.
- **HTTP** — request-response vs one-way.
- **Messaging** — sync (RPC) vs async (event).
- **Server** — thread-per-request vs event loop.
- **DB** — JDBC (blocking) vs R2DBC (reactive).
- **UI** — Ajax callback vs Promise vs async/await.

Разберём каждый.

---

## 4. Sync/async на уровне метода

### 4.1 Blocking method

Классика:
```java
public User getUser(Long id) {
    User u = repo.findById(id);   // ждём БД
    return u;
}
```

Поток блокируется на I/O.

### 4.2 Non-blocking через callback (уродливо)

```java
public void getUser(Long id, Consumer<User> callback) {
    repo.findByIdAsync(id, u -> callback.accept(u));
}
```

Callback hell при цепочке.

### 4.3 Non-blocking через Future

```java
public Future<User> getUser(Long id) {
    return executor.submit(() -> repo.findById(id));
}

Future<User> future = getUser(1L);
// поток свободен
User u = future.get();   // блокируется тут
```

`Future.get()` блокирует — не совсем async.

### 4.4 CompletableFuture (Java 8+)

```java
public CompletableFuture<User> getUser(Long id) {
    return CompletableFuture.supplyAsync(() -> repo.findById(id));
}

getUser(1L)
    .thenCompose(u -> loadOrders(u.getId()))
    .thenApply(orders -> new UserOrders(user, orders))
    .thenAccept(uo -> render(uo))
    .exceptionally(e -> { log.error(e); return null; });
```

Настоящий async. Цепочка операций.

### 4.5 Reactor (Mono / Flux)

Reactive Streams:
```java
public Mono<User> getUser(Long id) {
    return userRepo.findById(id);
}

getUser(1L)
    .flatMap(u -> loadOrders(u.getId()))
    .map(orders -> new UserOrders(...))
    .subscribe(uo -> render(uo));
```

Не выполняется пока не `subscribe`. Lazy.

### 4.6 Virtual threads (Java 21+)

```java
public User getUser(Long id) {
    return repo.findById(id);   // выглядит как sync!
}

Thread.ofVirtual().start(() -> {
    User u = getUser(1L);        // блокируется, но virtual thread — cheap
    process(u);
});
```

Синтаксически sync, но JVM освобождает carrier thread на I/O. См. `19-java-21-virtual-threads.md`.

---

## 5. Sync/async на уровне HTTP

### 5.1 Sync HTTP (обычный)

```java
RestClient client = ...;
User u = client.get().uri("/users/1").retrieve().body(User.class);
// поток ждёт ответа
```

Клиент блокирует поток на время round-trip.

### 5.2 Async HTTP (CompletableFuture)

```java
CompletableFuture<User> future = client.get().uri("/users/1").exchange().toEntity(User.class);
// поток свободен
```

Не блокирует. Даёт `CompletableFuture`.

### 5.3 Reactive HTTP

```java
WebClient webClient = ...;
Mono<User> user = webClient.get().uri("/users/1").retrieve().bodyToMono(User.class);
user.subscribe(u -> process(u));
```

Non-blocking, event loop.

### 5.4 Java 11+ HttpClient

```java
HttpClient client = HttpClient.newHttpClient();

// sync
HttpResponse<String> resp = client.send(request, BodyHandlers.ofString());

// async
CompletableFuture<HttpResponse<String>> future = client.sendAsync(request, BodyHandlers.ofString());
future.thenAccept(r -> process(r.body()));
```

---

## 6. Sync/async на уровне messaging

### 6.1 Sync messaging (request-response)

Client шлёт запрос, ждёт ответ:
- **RPC поверх broker** — Rabbit reply-to queue, correlation id.
- **HTTP** — обычный.

По сути = sync HTTP через broker. Редко используется (сложнее HTTP).

### 6.2 Async messaging (fire-and-forget)

Producer шлёт event → уходит. Не знает кто получил, не ждёт response.

```java
kafkaTemplate.send("orders", event);
// producer продолжает работать
```

Consumer(s) получают асинхронно.

**Стандартный** async pattern для микросервисов.

### 6.3 Pub-sub

Один event → **много** consumers:
```
Publisher → Topic ─┬─→ Consumer A
                   ├─→ Consumer B
                   └─→ Consumer C
```

Все параллельно обрабатывают.

### 6.4 Event-driven vs Command-based

**Command**: «сделай X» (одному consumer'у).
**Event**: «X произошло» (кто хочет, тот подписывается).

Event — более decoupled. Publisher не знает про consumers.

---

## 7. Sync/async на уровне сервера

### 7.1 Thread-per-request (Tomcat)

Каждый HTTP-запрос → **свой поток** обрабатывает от начала до конца.

```
Request 1 → Thread 1 (обрабатывает, блокируется на I/O)
Request 2 → Thread 2
Request 3 → Thread 3
...
```

Плюсы: простой model.
Минусы: пул потоков ограничен (обычно 200). При I/O-heavy — потоки простаивают.

### 7.2 Event loop (Netty / WebFlux)

**Малый пул потоков** (по 1 на CPU) обрабатывает **много** соединений:

```
Event Loop 1 → соединения [1, 5, 12, 200, ...]
Event Loop 2 → соединения [2, 3, 44, 89, ...]
```

Каждый event loop:
1. Ждёт event на любом соединении (epoll).
2. Обрабатывает быстро (без blocking).
3. При I/O — регистрирует callback, идёт к следующему event.

Плюсы: десятки тысяч соединений одним потоком.
Минусы: **никакого blocking** — иначе event loop stall.

### 7.3 Virtual threads (Java 21)

Комбинация лучших сторон:
- Пишешь синтаксически blocking код.
- JVM освобождает carrier thread на I/O.

Middle ground. См. `19-java-21-virtual-threads.md`.

---

## 8. Sync/async на уровне БД

### 8.1 JDBC (blocking)

```java
Connection conn = ds.getConnection();
ResultSet rs = conn.createStatement().executeQuery("SELECT ...");
while (rs.next()) { ... }
```

Каждый вызов блокирует поток. Стандарт.

### 8.2 R2DBC (Reactive)

```java
DatabaseClient client = DatabaseClient.create(...);
Flux<User> users = client.sql("SELECT * FROM users")
    .map(row -> new User(row.get("id", Long.class)))
    .all();
users.subscribe(u -> process(u));
```

Non-blocking. Работает через event loop.

### 8.3 Virtual threads + JDBC

С Java 21 — virtual thread делает JDBC "прозрачно async" (mount/unmount на I/O).

**Правило**: JDBC + virtual threads сейчас часто лучше R2DBC (проще, экосистема JPA работает).

---

## 9. Spring MVC vs WebFlux

Уже разбирали в `07-embedded-servers-tomcat-undertow.md`.

- **MVC** — thread-per-request, blocking API (`RestClient`, JDBC).
- **WebFlux** — event loop, non-blocking (`WebClient`, R2DBC).

Одно приложение — обычно **один** стек (не миксовать).

---

## 10. Async в Spring — @Async

Метод выполняется в **другом потоке**.

```java
@Service
class NotificationService {

    @Async
    public void sendEmail(String to) {
        // выполнится на TaskExecutor
    }

    @Async
    public CompletableFuture<String> asyncCompute() {
        return CompletableFuture.completedFuture("result");
    }
}
```

Требует `@EnableAsync`. Реализация — AOP-proxy.

Кавет **self-invocation** — тот же что @Transactional.

По умолчанию пул `SimpleAsyncTaskExecutor` (создаёт new thread на вызов!). **Настрой свой**:
```java
@Bean("taskExecutor")
TaskExecutor executor() {
    var e = new ThreadPoolTaskExecutor();
    e.setCorePoolSize(10);
    e.setMaxPoolSize(50);
    e.setQueueCapacity(200);
    e.initialize();
    return e;
}
```

---

## 11. Backpressure

Ключевая проблема async: **producer быстрее consumer**.

Sync: producer ждёт consumer → нет проблемы.

Async:
- Consumer накапливает работы → **память переполняется**.
- Или consumer дропает → **потеря данных**.

### 11.1 Решения

- **Bounded queue** — если полна, block producer или reject.
- **Reactive Streams** (`Publisher/Subscriber`) — subscriber говорит "я готов принять N" через `request(n)`.
- **Rate limiting** producer'а.
- **Batching**.

Reactor / Kafka имеют встроенный backpressure. CompletableFuture — нет.

---

## 12. Error handling

### 12.1 Sync

Простой:
```java
try {
    User u = getUser(1L);
} catch (Exception e) {
    // ...
}
```

Stack trace показывает где упало.

### 12.2 Async — сложнее

```java
getUser(1L)
    .thenApply(u -> transform(u))
    .thenCompose(u -> save(u))
    .exceptionally(e -> {
        log.error(e);   // stack trace может не показать причину
        return null;
    });
```

Stack trace обычно показывает "где subscribe" — не где реальная ошибка.

Reactor / CompletableFuture имеют `.onError`, `.doOnError`, `.exceptionally`.

**Правило**: **всегда** обработчик ошибок в async-цепочке. Иначе — silent failures.

---

## 13. Debugging

### 13.1 Sync легко

Точка останова → stack trace → идёшь по вызовам.

### 13.2 Async сложно

Callback / continuation в другом потоке → нет вертикальной картины.

Reactor Context, MDC propagation, `.log()` в цепочке.

Virtual threads улучшают ситуацию — stack trace выглядит как sync.

---

## 14. Trade-offs

### 14.1 Когда sync лучше

- **Простой запрос-ответ** без большой нагрузки.
- **Простой код** приоритетнее производительности.
- **Debugging** важен.
- **Legacy** экосистема (JDBC, JPA).
- **Малое количество concurrent connections**.

### 14.2 Когда async лучше

- **Много concurrent connections** (>1k).
- **I/O-heavy** (много downstream calls fan-out).
- **Streaming** (long polling, SSE, WebSocket).
- **Backpressure** нужен.
- **Reactive downstream** already (Kafka, R2DBC).

### 14.3 Правило микросервисов

**Sync HTTP** для request-response между клиентом и сервисом.
**Async events** для state propagation между backend сервисами.

Никогда **sync chain** длиннее 2 сервисов (cascading failure).

---

## 15. Async patterns в микросервисах

### 15.1 Async command

Client шлёт command → сервис принимает в очередь → возвращает 202 Accepted → обрабатывает.

```
POST /orders
Response: 202 Accepted
Location: /orders/status/abc123

GET /orders/status/abc123
Response: {"status": "processing"} или {"status": "done", "orderId": 42}
```

Для долгих операций.

### 15.2 Fire and forget

Публикует event, забывает. Не ждёт результат.

Хорошо для side effects: audit, notifications.

### 15.3 Event choreography

Каждый сервис публикует events → другие подписываются. См. `50-saga-pattern.md`.

### 15.4 CQRS

Разделение commands (write, async) и queries (read, sync).

---

## 16. Реальные примеры ИСНА

Из memory:

- **`knp-fno-outer-sync-esb-dead-route`** — `KnpOuterSystemFnoSyncService` **synchronous** SOAP-запрос в ЕСБ → падает → надо retry / отключить. **Async через outbox** был бы правильнее.
- **`knp-fo-sync-notification-bugs`** — `NotificationSyncService` async processing очередей; 6 багов включая молчаливые потери.
- **`knp-filter-sent-documents-perf`** — sync RestTemplate на АРМ, блокирует downstream. Async / batch решило бы.
- **RabbitMQ approvals queue** — async обмен между КНП и tax-rep.

Урок: **sync для user-facing, async для internal state**.

---

## 17. Собесные вопросы

1. **Sync vs async — разница?** — Sync блокирует поток; async запускает и продолжает.
2. **Что даёт async?** — Больше concurrent operations без большего пула потоков.
3. **Callback hell?** — Много вложенных callbacks; трудно читать; решение — CompletableFuture / async-await / Reactor.
4. **CompletableFuture vs Future?** — Future только `.get()` (блокирует); CompletableFuture — chaining.
5. **Что такое backpressure?** — Producer быстрее consumer; в async без backpressure — OOM.
6. **Reactive Streams — что даёт?** — Publisher/Subscriber с request(n) для backpressure.
7. **@Async в Spring?** — Метод в другом потоке; AOP-proxy + TaskExecutor.
8. **Self-invocation в @Async?** — Та же проблема что @Transactional (прокси).
9. **Thread-per-request vs event loop?** — Первый простой (Tomcat, JDBC); второй масштабируется (Netty, R2DBC).
10. **Virtual threads — что дают?** — Sync-код с async-performance (JVM управляет carrier).
11. **JDBC vs R2DBC?** — JDBC blocking; R2DBC reactive non-blocking.
12. **Когда sync лучше?** — Простой код, малая нагрузка, legacy JPA, простой debug.
13. **Когда async лучше?** — Много connections, fan-out, streaming, backpressure нужен.
14. **Sync HTTP chain — почему плохо?** — Cascading failure: один медленный/упал → все зависшие потоки.
15. **Как избежать sync chain?** — Async events через broker, circuit breaker + timeout.

---

## Итог

- **Sync** блокирует поток; **async** — нет.
- Уровни: **метод / HTTP / messaging / server / DB**.
- **CompletableFuture / Reactor / Virtual Threads** — Java async инструменты.
- **Backpressure** — критичен в async.
- **Sync для user-facing**, **async для internal state propagation**.
- Не создавай **sync chains** длиннее 2 сервисов.
- **Virtual threads (Java 21)** — часто оптимальный компромисс.

Следующий — `67-gateway-detailed.md`.
