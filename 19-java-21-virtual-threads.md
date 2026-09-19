# 19. Java 21: Virtual Threads (Project Loom)

Главная фича Java 21. Меняет способ писать concurrent-код.

---

## 1. Проблема, которую решают virtual threads

### 1.1 Классические (platform) threads

- 1 Java-поток = 1 OS-поток.
- OS-поток дорогой: ~1 MB стека, kernel-level context switch.
- На сервере с 8 GB RAM — ~4-5 тысяч потоков максимум.
- Пул потоков Tomcat = 200. При I/O-bound задачах CPU простаивает (потоки блокируются).

### 1.2 Reactive-подход как альтернатива

Netty + WebFlux + Reactor: event loop с 4-8 потоками обрабатывает тысячи соединений через non-blocking I/O.

Плюсы: масштабируется.
Минусы:
- Сложный код (Mono/Flux цепочки).
- Каждая библиотека должна быть reactive (JDBC → R2DBC, etc).
- Отладка кошмар (stacktrace в разных потоках).

### 1.3 Virtual threads — компромисс

Оставить простой блокирующий код, но сделать потоки дешёвыми.

- **Virtual thread** = «зелёный поток», управляется JVM.
- Стек — маленький, в heap (динамический).
- Миллионы virtual threads на одной JVM — легко.
- Когда virtual thread блокируется на I/O — JVM отключает его от OS-потока (**carrier thread**), тот идёт обслуживать другой virtual thread.

---

## 2. Как это работает

### 2.1 Аналогия

Раньше: каждый посетитель ресторана = свой официант (OS thread). Мало официантов, длинная очередь.

Теперь: официантов немного, но каждый обслуживает многих. Пока клиент читает меню (I/O wait) — официант обслужит другого.

### 2.2 Механика

```
Virtual Thread              Carrier (OS) Thread                Kernel
    │                              │                              │
    │  new virtual thread          │                              │
    │  Runs code                   │  Executes                    │
    │  ...                         │  ...                         │
    │  socket.read()  ─────────►   │  делает epoll_wait          │
    │                              │                              │
    │  (unmounted)                 │  Goes idle                   │
    │                              │  Runs another VT             │
    │                              │  ...                         │
    │  data arrived  ◄─────────    │                              │
    │  (remounted)                 │  Resumes VT                  │
    │  process data                │                              │
```

Ключевое: JVM компилирует блокирующие вызовы в yield-точки. При блокировке virtual thread «размонтируется» с carrier thread → carrier занят другим.

### 2.3 Carrier threads

Пул OS-потоков, на которых бегают virtual threads. По умолчанию — ForkJoinPool с `CommonPool.ParallelismLevel = availableProcessors()`.

Настройка:
```
-Djdk.virtualThreadScheduler.parallelism=8
```

---

## 3. Как создать

### 3.1 Простой запуск

```java
// как platform thread
Thread.ofPlatform().start(() -> System.out.println("Hello"));

// как virtual thread
Thread.ofVirtual().start(() -> System.out.println("Hello"));

// или короче
Thread.startVirtualThread(() -> System.out.println("Hello"));
```

### 3.2 Executor

```java
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        final int id = i;
        executor.submit(() -> {
            Thread.sleep(1000);
            System.out.println("done " + id);
            return null;
        });
    }
}   // executor.close() ждёт все задачи
```

Обрати внимание: **`newVirtualThreadPerTaskExecutor`** — не «пул»! Каждой задаче — свой virtual thread. Их дёшево создавать.

### 3.3 ExecutorService — обычные virtual threads

```java
ExecutorService exec = Executors.newThreadPerTaskExecutor(
    Thread.ofVirtual().name("worker-", 0).factory());
```

---

## 4. Pinning — важный кавет

Иногда virtual thread НЕ размонтируется с carrier'а — «пинится». Причины:

### 4.1 `synchronized`

Блок synchronized держит carrier thread:
```java
synchronized (lock) {
    // если тут blocking I/O — carrier thread pinned!
    socket.read();
}
```

Внутри `synchronized` любое блокирование прикрепляет virtual thread к carrier'у. Если много таких — общий пул carrier'ов истощается.

**Решение**: используй `ReentrantLock` вместо `synchronized` — он pinning-safe.

### 4.2 Native code (JNI)

Пока virtual thread в native — не отмонтировать. Используй избирательно.

### 4.3 Диагностика

```
-Djdk.tracePinnedThreads=full
```

Печатает stack trace каждого pinning-события.

---

## 5. Когда использовать virtual threads

### 5.1 Подходит

- **I/O-bound**: HTTP-клиенты, JDBC, Kafka, файлы. Один запрос ждёт → JVM отпустит carrier.
- **Много одновременных соединений**: WebSocket, SSE, long polling.
- **Fan-out**: параллельно позвать 10 сервисов и собрать результаты.
- **Классический thread-per-request Tomcat**: заменяет пул на «поток на запрос без лимита».

### 5.2 Не подходит

- **CPU-bound**: если поток реально считает — virtual thread не быстрее platform.
- **С synchronized блоками с I/O** — pinning убивает преимущество.
- **JNI-heavy**: подобное pinning.
- **Мелкие задачи** (создание virtual thread хоть и дёшево, но не бесплатно): использовать общий пул.

---

## 6. Structured Concurrency (preview в 21, финал в 23)

Что делать когда одна задача fan-out'ит несколько:

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<User> user = scope.fork(() -> userService.get(id));
    Subtask<Address> addr = scope.fork(() -> addrService.get(id));

    scope.join();               // ждём все или первую ошибку
    scope.throwIfFailed();

    return new UserAndAddress(user.get(), addr.get());
}
```

Плюсы:
- Все subtasks завершаются вместе или все отменяются при ошибке.
- Читабельный код, единый scope.
- Stack trace имеет смысл.

### 6.1 Варианты

- `ShutdownOnFailure` — при ошибке одной — всех отменить.
- `ShutdownOnSuccess` — при первом успехе — остальных отменить.

---

## 7. ScopedValue (preview в 21)

Замена ThreadLocal для virtual threads. ThreadLocal плохо работает с миллионами VT — каждая копия жрёт память.

```java
static final ScopedValue<String> USER = ScopedValue.newInstance();

ScopedValue.where(USER, "berik").run(() -> {
    doWork();   // внутри USER.get() == "berik"
});
```

Immutable, ограничена scope выполнения.

---

## 8. Virtual threads в Spring Boot

### 8.1 Spring Boot 3.2+

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

- Tomcat request-handler → virtual threads.
- Async tasks → virtual threads.
- Scheduled → тоже.

### 8.2 Явно

```java
@Bean
TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
    return protocolHandler -> {
        protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
    };
}
```

### 8.3 @Async

```java
@Bean
AsyncTaskExecutor asyncTaskExecutor() {
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}
```

Теперь `@Async` методы бегут на virtual threads.

---

## 9. Пример: fan-out c virtual threads

Собрать данные из 3 сервисов параллельно:

```java
// со structured concurrency
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user = scope.fork(() -> userClient.get(userId));
    var orders = scope.fork(() -> orderClient.getAll(userId));
    var address = scope.fork(() -> addressClient.get(userId));

    scope.join().throwIfFailed();

    return new UserFullInfo(user.get(), orders.get(), address.get());
}
```

Три HTTP-запроса параллельно на virtual threads. Latency = max(t1, t2, t3), не сумма.

---

## 10. Отличия от reactive/WebFlux

| | Virtual Threads | WebFlux (Reactor) |
|---|---|---|
| Стиль кода | Обычный императивный | Reactive цепочки (Mono/Flux) |
| Blocking код | Работает (даже полезно) | Ломает event loop |
| Кривая обучения | Знакомая | Крутая |
| Debugging | Обычный stacktrace | Не всегда понятный |
| Backpressure | Нет из коробки | Есть |
| Fit для JDBC | Да | Только через R2DBC |

Часто выбор — **virtual threads** для нового кода / миграции старого. WebFlux — если реально нужна backpressure (streaming) или очень много соединений (>100k).

---

## 11. Bench-марки (примерные)

Один Spring Boot сервис, endpoint делает 2 внешних HTTP-запроса + DB query:

| | Throughput | Latency p99 | Memory |
|---|---|---|---|
| Tomcat 200 threads | 500 req/s | 400ms | 500 MB |
| Tomcat + virtual threads | 3000 req/s | 200ms | 400 MB |
| WebFlux + Netty | 3500 req/s | 180ms | 350 MB |

Virtual threads дают почти WebFlux-performance с императивным кодом.

---

## 12. Кавет: DB connection pool

Virtual threads легко создать миллион. Но JDBC connection pool — 20 соединений в HikariCP. Если 1M virtual threads ждут DB — 999,980 в ожидании соединения (не in DB!).

Решение:
- Увеличить пул (умеренно — БД тоже имеет лимит).
- Batch операций.
- Асинхронный I/O (R2DBC — но тогда нет смысла в virtual threads).
- Не выделять virtual thread per задача — семафоры для контроля concurrency.

**Правило**: virtual threads не отменяют backpressure. Просто дают дешёвые потоки. Ограничения БД/API/пулов — те же.

---

## 13. Реальность в ИСНА

По memory:
- **`knp-fo-prod-java21-migration-clean-baseline`**: прод-ФО на Java 21, virtual threads не включены явно (тут просто Java 21 upgrade).
- В ИСНА Spring Boot 2.x массово — VT нужен Boot 3+ (Java 17 min для Boot 3). Пока не массовое использование.

Ожидание: virtual threads станут стандартом после миграции всех модулей на Boot 3+.

---

## 14. Собесные вопросы

1. **Что такое virtual thread?** — Легковесный поток, управляется JVM, монтируется на carrier thread (OS thread) для выполнения.
2. **Разница virtual и platform thread?** — VT дешёвый (миллионы), стек в heap; platform = OS thread, ~1 MB, тысячи.
3. **Как работает carrier thread?** — Пул OS-потоков (обычно ForkJoinPool), на которых JVM выполняет virtual threads.
4. **Что такое pinning?** — Ситуация когда VT не размонтируется с carrier: synchronized блок с I/O, native code.
5. **Когда VT не эффективны?** — CPU-bound, много synchronized, JNI, мелкие задачи.
6. **Что такое structured concurrency?** — Управление жизненным циклом группы задач вместе (fork/join/shutdown).
7. **ScopedValue vs ThreadLocal?** — ScopedValue immutable, ограничен scope, лучше для VT.
8. **Как включить VT в Spring Boot?** — `spring.threads.virtual.enabled=true` в Boot 3.2+.
9. **VT vs WebFlux?** — VT для императивного кода с I/O; WebFlux для backpressure / >100k connections.
10. **Что делать с DB connection pool при VT?** — Пул не растёт — семафоры или увеличенный пул; VT не отменяют backpressure.
11. **Как найти pinning в проде?** — `-Djdk.tracePinnedThreads=full`.

---

## Итог

- **Virtual threads** = дешёвые потоки, JVM их размонтирует при I/O.
- **Замена thread-per-request** без reactive-сложностей.
- **Pinning** от synchronized + I/O — используй ReentrantLock.
- **Structured concurrency** — правильный fan-out.
- **ScopedValue** — правильный ThreadLocal для VT.
- **Spring Boot 3.2+** — включается флагом.
- **Не серебряная пуля** — DB pool, CPU-bound, JNI требуют внимания.

---

## Итог блока Java 11 → 21

- 16 — синтаксис (var, records, sealed, pattern matching, switch, text blocks).
- 17 — JVM/GC (G1, ZGC, container awareness, CDS, JFR).
- 18 — API и миграция (javax→jakarta, HttpClient, Stream.toList, грабли).
- 19 — Virtual Threads (Loom, structured concurrency, ScopedValue).

Следующий блок — RabbitMQ (файлы 20-23). Пишу `20-rabbitmq-amqp-basics.md`.
