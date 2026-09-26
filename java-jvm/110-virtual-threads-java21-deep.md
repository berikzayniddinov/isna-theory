# 110. Virtual Threads в Java 21: continuations, pinning, structured concurrency

## Зачем это знать

Virtual Threads (JEP 444, финализированы в Java 21) — самая заметная фича JDK за последние 10 лет. Обещают решить фундаментальную проблему thread-per-request архитектуры: OS-тред дорогой, их можно иметь тысячи, но не миллионы. Раньше приложение с 10 000 concurrent соединений упиралось в OS-треды и требовало reactive-подхода (Netty, WebFlux) с его callback hell и сложным debugging. Virtual threads обещают миллион concurrent «тредов» в обычной thread-per-request модели без изменения кода.

Обещание работает — если понимаешь механику. Иначе VT дают ложное чувство «просто добавь `Executors.newVirtualThreadPerTaskExecutor()` и всё быстрее». На деле — тонны нюансов, которые определяют реально ли ты получил выигрыш. **Pinning**: тред застрял на carrier из-за `synchronized`, JNI, native call — vt не паркуется, carrier тратится, потолок параллельности возвращается к числу carriers (обычно = ncores). **`ThreadLocal` pattern**: миллион VT × ThreadLocal с непустым value = гигабайты памяти, GC пресс. **Connection pool**: db не масштабируется под миллион, любой async-код всё равно упирается в пул. **synchronized внутри JDBC-драйверов**: пиннит carrier на все секунды выполнения SQL.

Разница между «использую VT» и «понимаю VT» — умение читать thread dump с carrier vs virtual, диагностировать pinning через JFR event `jdk.VirtualThreadPinned`, понимать когда VT реально даёт выигрыш (I/O-bound, много одновременных соединений), а когда бесполезен (CPU-bound, малое число концаррентных задач).

Разберём: что такое virtual thread на уровне JVM (объект в куче, не OS thread), continuations как фундамент (JEP 425 предшественник, потом внутренняя механика). Как работает mount/unmount на carrier thread, что физически происходит при блокировке. Список **всех** причин pinning — почему synchronized до Java 24, JNI/FFM, monitorenter в spin loop. ForkJoinPool как carrier scheduler и почему нельзя менять на custom. Structured Concurrency (JEP 453) — как правильно управлять группами VT. Scoped Values (JEP 464) — замена ThreadLocal для VT. Debugging: `jcmd Thread.dump_to_file -format=json`, JFR events для pinning, thread dump читка с carrier/virtual разделением. Практические ловушки: JDBC-драйверы и pinning, Spring `@Transactional`, semaphore vs connection pool. Миграция существующего кода: чек-лист что менять, что можно оставить. Когда VT не даёт выигрыша.

Базовый intro — в 19. Сравнение platform vs virtual с числами — в 83. Здесь — глубокая механика JVM + практические подводные камни, которые проявляются только в проде.

## Что такое virtual thread на самом деле

Virtual thread — **объект Java в куче**, реализующий `java.lang.Thread`, но не привязанный к OS thread. Хранит своё состояние (stack frames, локальные переменные) в heap. Выполняется на **carrier thread** (обычный platform thread из пула ForkJoinPool). Может отсоединиться от carrier (unmount) при блокирующей операции, освободив carrier для других VT, и снова смонтироваться (mount) когда блокировка снята.

Разница с platform thread. Platform thread — обёртка над `pthread` в Linux: OS выделяет ~1-2 MB виртуальной памяти на стек, JVM держит соответствующий Java-объект. Максимум — тысячи тредов на процесс (упирается в virtual memory и context switching overhead). Virtual thread — просто объект в куче, стек хранится динамически (несколько KB для тонкого VT, растёт по мере вложенности вызовов). Максимум — **миллионы**.

Ключевое: код внутри VT — обычный синхронный Java-код. `Thread.sleep(1000)`, `socket.read()`, `connection.executeQuery()` — всё выглядит блокирующим и пишется как обычно. Магия — под капотом. При блокирующем syscall JVM превращает его в non-blocking + unmount VT + перезапись continuation на карте heap-based scheduling. Приложение считает что тред «спит», carrier занят другим VT.

Простой пример:

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return null;
        });
    });
}
```

10 000 «тредов», каждый спит секунду. С platform threads это либо OOM (10K × 1MB стека = 10 GB), либо thread pool с очередью (мучительно долго). С virtual threads — работает за секунду с копейками, поверх десятка carriers.

## Continuations: фундамент VT

Механика unmount/mount построена на **continuations** — механизме сохранения и восстановления состояния выполнения. Continuations были добавлены в HotSpot как внутренний API специально для VT (JEP 425 в preview, финализированы вместе с VT в 444).

Continuation — объект, инкапсулирующий стек вызовов в моменте плюс program counter. Позволяет:
- `Continuation.yield()` — сохранить текущий стек, вернуться в вызвавший `Continuation.run()`.
- `Continuation.run()` — запустить или продолжить continuation с сохранённой точки.

Псевдокод механики VT:

```
Carrier thread T выполняет continuation C для VT vt1
  → vt1 делает socket.read()
    → JDK внутри превращает в epoll_wait + Continuation.yield()
      → yield() копирует stack frames из T в heap-объект C
      → yield() возвращает управление carrier
    → Carrier T возвращается в scheduler, берёт следующий VT vt2
  ...
  → эpoll_wait завершился, есть данные для vt1
    → Scheduler ставит continuation vt1 в очередь
    → Свободный carrier T' берёт vt1
      → Continuation.run() копирует stack из heap обратно на T' stack
      → выполнение продолжается прямо после socket.read()
```

Стек VT — не отдельная фиксированная память, а **snapshot копируется между heap и carrier stack**. При unmount — frames копируются в heap. При mount — обратно на carrier stack. Копирование, но фреймов обычно немного (5-50 в типичной обработке запроса) — стоимость приемлемая.

Отсюда важное свойство: **память VT растёт по мере глубины вызовов**. Тонкий VT — единицы KB. Глубокий рекурсивный или с сложным стеком — сотни KB. Миллион VT с сложным стеком = гигабайты в heap. Планировать под свой profile.

## Carrier scheduler: ForkJoinPool под капотом

VT выполняются на **carrier threads**, которые формируют внутренний ForkJoinPool JVM. По умолчанию его размер = `Runtime.getRuntime().availableProcessors()`. Настраивается через `-Djdk.virtualThreadScheduler.parallelism=N`.

Почему ForkJoinPool: work-stealing даёт отличное распределение мелких задач между потоками, low-overhead push/pop локальной очереди. Идеально для миллиона мелких VT.

Carrier pool — **общий на всё JVM**. Все VT конкурируют за одни и те же carriers. Нельзя создать «свой» carrier scheduler для отдельного executor (по крайней мере в стабильном API). Все `Executors.newVirtualThreadPerTaskExecutor()` шедулятся через тот же пул.

Что это значит на практике. Если один код в приложении делает pinning (например, драйвер JDBC с synchronized) — pinned VT занимает carrier целиком, пока не отработает. Остальные VT ждут свободного carrier. При 8 carriers и 8 pinned VT — вся система стоит, независимо от того сколько «свободных» virtual threads в heap.

Настраиваемый параметр `-Djdk.virtualThreadScheduler.maxPoolSize` (default 256) — при pinning JVM может добавить carriers сверх initial parallelism, чтобы не заблокировать всё. Но это compensation mechanism, не решение — правильно устранять причину pinning.

## Pinning: полный список причин и как ловить

**Pinning** — состояние когда VT не может отсоединиться от carrier, даже при блокирующей операции. Причины разбиты на классы:

**1. `synchronized` до Java 24**. Метод или блок `synchronized` устанавливает monitor lock. VT, войдя в synchronized-блок, не может unmount на блокировке внутри блока — carrier остаётся занят. С Java 24 (JEP 491) это исправлено: synchronized стал VT-friendly через изменение внутренней реализации ObjectMonitor. Но в Java 21 (и большинстве enterprise deployments сейчас) — `synchronized` по-прежнему pin'ит.

Пример:

```java
class LegacyCache {
    private final Map<String, Value> cache = new HashMap<>();
    
    public synchronized Value get(String key) {
        Value v = cache.get(key);
        if (v == null) {
            v = fetchFromDb(key);  // blocking I/O!
            cache.put(key, v);
        }
        return v;
    }
}
```

VT входит в `get()`, берёт monitor. Делает `fetchFromDb()` — блокирующий SQL. Не может unmount (pinned). Carrier занят. При тысяче параллельных запросов на один cache — все VT встанут в очередь на monitor, carriers полностью заняты, приложение де-факто работает как singlethreaded.

Обход в Java 21: заменить `synchronized` на `ReentrantLock` — работает нормально с VT (не pin'ит).

```java
private final ReentrantLock lock = new ReentrantLock();

public Value get(String key) {
    lock.lock();
    try {
        // ... то же самое
    } finally {
        lock.unlock();
    }
}
```

**2. Native code (JNI) в стеке**. Если VT в стеке имеет фрейм из native (`method native` или `MethodHandle` в native), unmount невозможен — native code не знает про continuations, не может быть скопирован. Классический пример — old-style JDBC-драйверы использующие JNI для network I/O. Новые Postgres/MySQL драйверы полностью на Java — работают с VT.

**3. Foreign Function & Memory API (FFM)**. Использование FFM (замена JNI, JEP 442) — тот же эффект. Native вызов = pinning.

**4. `Object.wait()` до Java 24**. Устаревший механизм synchronized-based ожидания. Pin'ит. `wait/notify` практически не используется в новом коде — есть `LockSupport.park()`, `Condition`, `CountDownLatch`.

**5. Class initializer**. VT инициализирующий класс (первое обращение к `Foo.class`) — pin'ится на время `<clinit>`. Обычно не проблема (`<clinit>` быстрый), но если внутри init делается блокирующий I/O — pinning.

**6. Некоторые вызовы parking через `synchronized` внутри JDK-классов**. Есть legacy JDK-код (например, старые версии `ConcurrentHashMap` использовали `synchronized` для узлов), где pinning неизбежен. С каждой новой версией JDK количество таких мест уменьшается.

**Как поймать pinning в проде**:

Опция JVM `-Djdk.tracePinnedThreads=full` — при каждом pinning печатает stacktrace в stderr:

```
Thread[#25,ForkJoinPool-1-worker-1,5,CarrierThreads]
    java.base/java.lang.VirtualThread.parkOnCarrierThread(VirtualThread.java:...)
    java.base/java.lang.VirtualThread.park(VirtualThread.java:...)
    java.base/java.lang.System$2.parkVirtualThread(System.java:...)
    java.base/jdk.internal.misc.VirtualThreads.park(VirtualThreads.java:...)
    java.base/java.util.concurrent.locks.LockSupport.park(LockSupport.java:...)
    ...
    - <== monitors:1  ← вот тут pinning
    com.example.LegacyCache.get(LegacyCache.java:42)
```

Строка `<== monitors:N` показывает сколько monitor'ов держит VT в моменте pinning. Стек показывает где.

`-Djdk.tracePinnedThreads=short` — только имя метода без полного стека, меньше шума.

**JFR events** (Java Flight Recorder) — production-friendly способ. Event `jdk.VirtualThreadPinned` фиксирует каждое событие pinning с duration и stack. Собирается через:

```bash
jcmd <pid> JFR.start duration=60s filename=vt.jfr settings=profile
```

Открыть в JMC (Java Mission Control) или через `jfr print vt.jfr`. Топ pinning-мест — кандидаты на переписывание.

## Structured Concurrency (JEP 453, preview в 21, стабильно в 25)

Проблема классической конкурентности: если запустил 10 задач в parallel, одна упала, а остальные — что с ними? Обычный `ExecutorService` не даёт хорошего инструмента: `shutdownNow()` может не остановить, `Future.cancel()` не гарантирует. Утечки задач-«зомби», выполняющих ненужную работу после ошибки.

**Structured Concurrency** переносит концепцию структурного программирования (if/for/try с определёнными границами) на треды. Все sub-tasks одной родительской scope живут в пределах этой scope. При выходе — гарантированно все sub-tasks либо завершены, либо cancelled.

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<String> user = scope.fork(() -> fetchUser(id));
    Subtask<Order> order = scope.fork(() -> fetchOrder(id));
    Subtask<Payment> payment = scope.fork(() -> fetchPayment(id));
    
    scope.join();              // ждём все
    scope.throwIfFailed();     // если хоть одна упала — throw
    
    return combine(user.get(), order.get(), payment.get());
}
```

Три параллельных запроса. Если один упал — остальные cancelled (interrupted). Scope закрывается — гарантированно все VT завершены. Никаких zombie тредов. Никакого ручного tracking futures.

**Варианты scope**:

- **ShutdownOnFailure** — cancel всех при первой ошибке. Классический fail-fast.
- **ShutdownOnSuccess** — cancel всех при первом успехе. Race pattern (первый ответивший выигрывает).
- **Custom** — свой Policy через subclassing.

Для VT structured concurrency — естественный паттерн: миллион дешёвых VT позволяет fork'ать сколько угодно параллельных запросов без беспокойства о ресурсах. С platform threads это было дорого — теперь нет.

## Scoped Values (JEP 464)

`ThreadLocal` был стандартным способом хранить per-thread context (текущий пользователь, request ID для MDC, tenant в multi-tenant SaaS). С миллионом VT возникают проблемы:

- **Память**. Каждый VT × ThreadLocal с непустым значением = миллион копий значения. Может быть гигабайты.
- **Наследование**. `InheritableThreadLocal` копируется при создании дочернего треда — при миллионе parent VT × копии в детей = экспоненциальный рост.
- **Mutability**. ThreadLocal изменяем, что нарушает functional подход и делает reasoning сложнее.

**Scoped Values** (JEP 464 preview в 21, стабильно в 25) — новый механизм передачи per-thread context, специально для VT.

```java
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

// Установка
ScopedValue.where(CURRENT_USER, user)
    .run(() -> {
        // всё что вызывается отсюда видит CURRENT_USER.get() == user
        processRequest();
    });

// Внутри
public void doWork() {
    User u = CURRENT_USER.get();  // тот же самый user
    // ...
}
```

Отличия от ThreadLocal:

- **Immutable per scope**. `.where(...)` создаёт **новый binding**, не меняет существующий. Внутри scope значение не меняется, можно только nested `.where()` с новым значением.
- **Автоматическое очищение**. При выходе из `run()` binding исчезает. Нет утечек как с забытым `ThreadLocal.remove()`.
- **Работает через structured concurrency**. Sub-tasks fork'нутые внутри scope видят те же ScopedValue — но только для structured (не для fire-and-forget executor.submit).
- **Дешевле по памяти**. Один binding shared между всеми VT в scope, не копия на каждый.

Для VT-heavy кода — стоит переносить ThreadLocal → ScopedValue где возможно. Особенно для immutable context (user, request ID, tenant).

## VT и JDBC: connection pool не отменяется

Частое заблуждение: «с VT можно миллион одновременных запросов к БД». Неверно. БД не масштабируется до миллиона connections — каждый connection = PostgreSQL backend process с 5-10 MB памяти, плюс контеншен за диски и WAL. Разумный пул — 10-30 connections (см. файл 107 про connection pools + I/O wait).

Что изменилось с VT? Раньше блокирующий вызов `dataSource.getConnection()` на исчерпанном пуле блокировал platform thread (дорогой). Теперь блокирует VT — carrier освобождается для других VT. Приложение может держать миллион запросов ждущих connection без исчерпания carrier'ов. Но сам пул по-прежнему потолок реальной параллельности БД-операций.

Правильная семантика: **VT для concurrent HTTP requests, connection pool для concurrent DB queries**. Разные уровни, разные потолки.

**JDBC-драйверы и pinning**. Историческая проблема — многие драйверы использовали `synchronized` в hot paths (например, `getConnection`, `executeQuery`). VT входит в synchronized → pinned на время I/O запроса к БД. Одна SQL-транзакция — секунды pinning, carrier недоступен.

PostgreSQL JDBC driver с версии 42.7+ переведён на `ReentrantLock` — работает с VT нормально. HikariCP с 5.1+ тоже избавился от synchronized в hot path. Проверить свою версию — обязательный шаг перед миграцией на VT.

Как проверить в проде: включить `-Djdk.tracePinnedThreads=full` в staging, прогнать нагрузку, посмотреть в stderr нет ли JDBC-стеков с `<== monitors:N`. Если есть — обновить драйвер или искать workaround.

## Spring Boot и VT

Spring Boot 3.2+ поддерживает VT через свойство:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Что это включает: Tomcat/Jetty/Undertow используют VT для обработки HTTP-запросов вместо thread pool. `@Async` использует VT для async-методов. Scheduled tasks — VT для запуска. RabbitTemplate/KafkaTemplate consumer'ы — VT.

Что **не** меняется: `@Transactional` — работает как обычно, поверх VT. Просто connection берётся из HikariCP так же. Если HikariCP версии до 5.1 — pinning на getConnection. Обновить.

**Проверка**: Spring Boot Actuator `/actuator/metrics/jvm.threads.started` показывает total тредов. С VT + нагрузкой должно расти в тысячи (не сотни как раньше).

Одна тонкость: **не смешивать VT и `CompletableFuture` без причины**. Раньше async pattern через `CompletableFuture.supplyAsync(...)` + `thenCompose` был спасением от блокировки. С VT это уже не нужно — обычный синхронный код на VT такой же эффективный. Избавляться от reactive-хвостов в пользу простого кода.

## Debugging thread dump с VT

`jstack` показывает только platform threads (carriers + всё остальное). VT в обычном dump'е не видны. Нужен новый инструмент:

```bash
jcmd <pid> Thread.dump_to_file -format=json /tmp/dump.json
```

JSON-формат содержит все VT с их состоянием, stack traces, current carrier. Просмотр — через специальные tools или `jq`:

```bash
cat dump.json | jq '.threads[] | select(.state == "RUNNABLE") | .name'
```

Также доступен plain text format (`-format=text`) — читабельнее для беглого просмотра, но объёмнее (миллион VT = мегабайты текста).

**JFR** более удобен для systematic анализа:

```bash
jcmd <pid> JFR.start duration=30s filename=recording.jfr settings=profile
```

Собирает всё: pinning events, mount/unmount, GC, blocking calls. Открывать в JMC.

Полезные JFR events для VT:
- `jdk.VirtualThreadStart` / `jdk.VirtualThreadEnd` — жизненный цикл.
- `jdk.VirtualThreadPinned` — pinning с duration и stack.
- `jdk.VirtualThreadSubmitFailed` — не удалось запустить VT (обычно scheduler переполнен).

**Метрики Micrometer/Actuator**. Spring Boot 3.2+ автоматически регистрирует:
- `jvm.threads.started` — общее число created тредов (rate = throughput).
- Есть ли специфичные VT metric — зависит от версии Spring Actuator.

## Когда VT не даёт выигрыша

VT — не серебряная пуля. Три случая когда переход на VT ничего не даст:

**1. CPU-bound задачи**. VT предназначены для I/O-bound: тред большую часть времени ждёт (network, disk, sleep). Если задачи молотят CPU (crypto, image processing, complex calculations) — параллельность ограничена числом ядер, VT сверх этого просто конкурируют за CPU без пользы. Использовать обычный thread pool = число ядер, не VT.

**2. Малое количество concurrent задач**. Если у тебя обычно 10-100 одновременных запросов — platform threads прекрасно справятся, никакого выигрыша от VT нет. VT становятся ощутимо лучше при 10K+ concurrent. Ниже — просто overhead на continuation machinery.

**3. Существующий async-код**. Если ты уже написал приложение на WebFlux/Reactor или Netty callbacks — переход на VT потребует полного переписывания. Выигрыш может быть (проще debugging, синхронный код) — но стоимость миграции высокая. Оценивать trade-off.

Правильный use case VT: **thread-per-request приложение с большим числом одновременных I/O-bound соединений**. HTTP-серверы, gateway-сервисы, chat-серверы, streaming (SSE, long-polling). Именно тут VT дают на порядки лучший scaling.

## Миграция существующего проекта

Чек-лист для перехода приложения на VT в Java 21:

**1. Обновить JDK до 21 LTS**. Проверить что все зависимости совместимы (Spring Boot 3.2+, Hibernate 6.4+, HikariCP 5.1+, PostgreSQL JDBC 42.7+).

**2. Найти все места с `synchronized`**. `grep -rn "synchronized" src/`. Приоритетно проверить те что оборачивают I/O операции (SQL, HTTP, files). Заменить на `ReentrantLock` где обёрнута блокирующая работа. Внутренние fast-path synchronized (короткие критические секции без I/O) — можно оставить, они не создают серьёзного pinning.

**3. Проверить JDBC-драйвер**. Актуальные версии основных драйверов (PostgreSQL 42.7+, MySQL Connector/J 8.3+) уже без synchronized в hot path. Устаревшие версии — обновить.

**4. Проверить кастомные connection pool'ы или JDBC-обёртки**. Часто enterprise-проекты имеют самописные wrappers. Аудит на synchronized внутри.

**5. Включить VT в Spring Boot**:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

**6. Убрать явные `@Async` + custom Executor если он был workaround для thread pool exhaustion**. С VT default behavior уже правильный.

**7. Аудит ThreadLocal**. Если ThreadLocal хранит крупные объекты или используется в hot path — рассмотреть замену на ScopedValue (только для тех мест где семантика immutable per scope работает).

**8. Тесты на staging с realistic нагрузкой**. Включить `-Djdk.tracePinnedThreads=full` (только для staging! в проде даст много stderr шума). Прогнать load test — искать pinning stacks. Устранять.

**9. Мониторинг в проде через JFR**. Постоянный low-overhead JFR recording в prod (профиль `default`), периодический анализ на pinning-heatmap.

**10. Плавный rollout**. Включить VT на одном инстансе (canary через feature flag или отдельный deployment), сравнить метрики: throughput, latency percentiles, CPU/memory. Если всё ок — раскатать на все.

## Практический пример: HTTP-сервер до и после

**До VT (Spring Boot 3.1, Tomcat, обычный thread pool)**:

Tomcat threads = 200 (дефолт). Каждый занят на всё время обработки запроса. Если запрос делает JDBC-вызов на 500 мс — 200 concurrent запросов, все Tomcat threads заняты, 201-й запрос ждёт в очереди. Максимум throughput: 200 requests × (1 / 500ms) = 400 req/s.

**После VT (Spring Boot 3.2+, VT enabled)**:

Tomcat создаёт VT на каждый запрос. Carriers = 8 (ncores). VT делает JDBC-вызов — unmount с carrier (при условии драйвера без synchronized), carrier берёт следующий VT. Максимум throughput ограничен не тредами, а **connection pool'ом**: 20 connections × (1 / 500ms) = 40 SQL query/s. То есть 40 запросов активно делают SQL параллельно, остальные VT висят в `getConnection()`.

Итог: throughput тот же (упирается в БД), но приложение легко держит 10 000+ concurrent соединений без OOM. При росте пула БД до 100 — throughput скалируется линейно.

## Заключение

Virtual Threads в Java 21 — не «magic performance boost», а новая абстракция потоков, оптимизированная под I/O-bound thread-per-request приложения. Дают возможность иметь миллион concurrent «тредов» с обычным синхронным кодом. Основаны на continuations — heap-based структурах сохраняющих стек между mount/unmount на carrier threads (ForkJoinPool под капотом).

**Pinning** — главная ловушка. VT не может unmount если стек содержит synchronized (до Java 24), JNI/FFM, Object.wait, native code. Pinned VT занимает carrier целиком. При 8 carriers и 8 pinned VT — вся система стоит. Обход в Java 21: `ReentrantLock` вместо `synchronized` для блоков вокруг I/O. Обновить JDBC-драйверы (PG 42.7+, HikariCP 5.1+). Диагностика через `-Djdk.tracePinnedThreads=full` в staging + JFR event `jdk.VirtualThreadPinned` в prod.

**Structured Concurrency** (JEP 453) — правильный способ управлять группами VT. `StructuredTaskScope` гарантирует что все sub-tasks завершены при выходе из scope, cancel при первой ошибке (`ShutdownOnFailure`) или первом успехе (`ShutdownOnSuccess`). Никаких zombie тредов.

**Scoped Values** (JEP 464) — замена ThreadLocal для VT. Immutable per scope, автоматическое очищение, дешевле по памяти при миллионе VT.

**Connection pool не отменяется**. VT позволяют миллион concurrent HTTP-запросов, но БД по-прежнему масштабируется через пул connections (обычно 10-30). Правильная семантика: VT для HTTP-параллельности, пул для SQL-параллельности.

**Spring Boot 3.2+** поддерживает VT через `spring.threads.virtual.enabled=true`. Включает VT для HTTP-обработки, @Async, scheduled tasks, message consumers. `@Transactional` работает без изменений.

**Debugging**: `jcmd Thread.dump_to_file -format=json` для дампа VT (jstack их не показывает). JFR для systematic анализа pinning. Метрики Actuator для throughput.

**Когда VT не помогает**: CPU-bound задачи (потолок = ncores), малое число concurrent (<1K, platform threads OK), существующий reactive-код (стоимость миграции высокая).

**Миграция**: JDK 21 → аудит synchronized вокруг I/O → обновить JDBC/HikariCP → включить в Spring Boot → тесты на staging с `-Djdk.tracePinnedThreads` → мониторинг JFR в prod → canary rollout.

**Правильный use case**: thread-per-request с большим числом I/O-bound concurrent соединений (HTTP API, gateway, chat, streaming). VT дают на порядки лучший scaling при простоте синхронного кода. Reactive-подход становится нужен реже — только для реально streaming-cases (backpressure, event sourcing).

Java 24 (JEP 491) уберёт synchronized pinning окончательно — VT станут ещё более transparent. Java 25 сделает Structured Concurrency и Scoped Values стабильными (non-preview). Ecosystem движется в правильную сторону.

Базовое intro в 19. Сравнение platform vs virtual с числами в 83. Connection pool + I/O wait — файл 107. Здесь была глубина по VT: continuations, carrier scheduler, полный список pinning причин, structured concurrency, scoped values, диагностика в проде, миграция существующего кода.
