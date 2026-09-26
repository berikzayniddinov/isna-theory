# 107. Connection pool и I/O wait: как приложение реально ждёт базу

## Зачем это знать

В проде 80% инцидентов «приложение висит» упираются в две связанные штуки: пул исчерпан и/или диски (сеть) не отвечают. Симптом одинаковый — CPU почти не грузится, а latency улетает в потолок и HikariCP пишет `Connection is not available, request timed out after 30000ms`. Дальше начинается разбор: пул мал, база тормозит, где-то держится connection слишком долго, оффлайновый HTTP-вызов внутри `@Transactional`, диск в iowait, или всё сразу.

Понять инцидент без модели того, как поток блокируется на I/O, как connection pool раздаёт и возвращает соединения, как postgres backend process выглядит с точки зрения OS — невозможно. Логи покажут «timeout», но не почему. Метрики покажут `hikaricp_connections_pending > 0`, но не кто виноват. Дампы потоков дадут `sun.nio.ch.EPollArrayWrapper.epollWait` в тридцати местах, но без модели I/O это ни о чём не говорит.

Этот файл собирает всю цепочку сверху вниз. Что такое connection pool изнутри — очередь, семафор, housekeeper, как выдаётся connection и что реально происходит когда `getConnection()` блокируется. Как правильно сайзить пул через Little's Law и почему «больше пул = лучше» — миф. Что такое I/O wait на уровне Linux — %iowait в top, iostat, что он реально измеряет и что нет. Как поток JVM блокируется на сокете (park через futex, epoll_wait, kernel-side wait queue). Разница между thread-per-request и virtual threads в контексте JDBC. Разбор боевых сценариев: idle in transaction, HTTP-вызов под транзакцией, thundering herd, deadlock на пуле, wait_event = ClientRead. Как это лечить и как отловить в проде через pg_stat_activity, hikaricp метрики, thread dump, iostat, ss.

## Connection pool изнутри: очередь, семафор, воркер

Concurrent пул — не просто «список соединений». Это трёхуровневая структура: пул объектов (соединения), очередь ожидающих (потоки, вызвавшие `getConnection` пока пул исчерпан), и housekeeper (фоновый тред, который добивает пул до `minimumIdle` и убивает старые соединения).

HikariCP использует специализированный `ConcurrentBag` — свою lock-free структуру вместо `ArrayBlockingQueue`. Идея: каждый поток сначала пытается взять соединение из своей thread-local списка (быстро, без синхронизации). Только если там пусто, идёт в общий пул через CAS-цикл. Если и там пусто — паркуется на `SynchronousQueue`, ожидая пока другой поток вернёт connection (через `borrow` / `requite`). Это даёт очень низкий overhead на fast path (borrow → use → return в одном треде) и корректно масштабируется на десятки тредов.

Схема:

```
     Application Thread              HikariCP internals
     ────────────────────            ──────────────────
     getConnection()  ──────────►    ConcurrentBag.borrow(timeout)
                                     │
                                     ├─► thread-local: есть свободный?
                                     │       ├─► да → выдать (fast path)
                                     │       └─► нет ↓
                                     │
                                     ├─► shared list: есть idle через CAS?
                                     │       ├─► да → выдать
                                     │       └─► нет ↓
                                     │
                                     ├─► пул не достиг max? → создать новое ↓
                                     │       (blocking network I/O!)
                                     │
                                     └─► park на handoffQueue до timeout
                                              │
                                              ├─► кто-то сделал requite() → взять
                                              └─► таймаут → SQLException
```

Ключевая деталь — **создание нового соединения выполняется в вызывающем треде**, синхронно. Handshake, аутентификация, SSL, `SET application_name`, jdbc init — всё это блокирующая сетевая работа, которая занимает 30-100 мс. Пока идёт, вызывающий тред заблокирован. Если пул холодный и пришло 20 запросов одновременно — все 20 будут ждать пока пул наполнится, каждый по 30-100 мс. Это «cold pool» проблема — лечится параметром `minimum-idle`, чтобы после старта уже был warm pool.

Housekeeper HikariCP работает в отдельном scheduled executor. Раз в секунду проверяет: сколько idle, сколько active, не превышают ли соединения `maxLifetime` (по умолчанию 30 минут — обязательно меньше чем `wait_timeout` PostgreSQL/MySQL, иначе получишь мёртвые соединения), нет ли соединений старше `idleTimeout` (по умолчанию 10 минут — их закрывает, добивая до `minimumIdle`). Плюс раз в `keepaliveTime` (5 минут по умолчанию) шлёт `SELECT 1` на idle соединения, чтобы firewall / L4 балансер не рвал их по inactivity.

## Little's Law и правильный размер пула

Классический вопрос: «сколько ставить `maximum-pool-size`?» Ответ не «100» и не «1000», а тот, который вытекает из **Little's Law**.

Little's Law: `L = λ × W`, где `L` — среднее количество запросов «в работе», `λ` — arrival rate (запросов в секунду), `W` — среднее время в системе (секунд). Если в проде 500 req/s и каждый запрос держит connection 20 мс — среднее количество занятых соединений `L = 500 × 0.02 = 10`. Пул из 15 обычно достаточно, из 30 — с запасом на всплески.

Обратная сторона: **большой пул не ускоряет, а замедляет.** Каждое активное соединение = один backend process в PostgreSQL, каждый жрёт 5-10 MB памяти и **делит одни и те же диски, ту же CPU, тот же WAL**. Если у сервера 8 vCPU, а ты держишь 200 активных connection — они дерутся за 8 ядер, context switch overhead растёт, кэши процессора мимо, latency каждого запроса ухудшается, общий throughput падает.

Официальный HikariCP FAQ (About Pool Sizing) — one of the best writeup'ов на эту тему. Основная формула: `connections = ((cores × 2) + effective_spindle_count)`. Для сервера с 8 vCPU и SSD (effective_spindle ≈ 1): `(8×2)+1 = 17`. Это **на всю базу**. Не на инстанс приложения, а суммарно от всех клиентов. Если микросервисов десять — каждый по 2 соединения, не по 20.

Практический recipe:
- начать с малого пула (`maximum-pool-size = 10-20`)
- снимать метрики `hikaricp_connections_active` и `hikaricp_connections_pending`
- если `active` часто упирается в max и `pending > 0` — расследовать сначала **почему запросы держат connection долго**, а не наращивать пул
- если после расследования всё ок, latency нормальная, а нужен ещё throughput — тогда увеличивать пул шагом 5-10
- держать в уме `max_connections` PostgreSQL (обычно 100-200) — суммарно от всех клиентов должно влезать с запасом на pgadmin, backup, репликацию

**Признак «пул мал» ≠ «увеличить пул».** Признак «пул мал» = «расследовать долгие запросы, транзакции, `getConnection()` перед HTTP-вызовом». Увеличение пула — это последняя линия обороны когда всё остальное честно уже сделано.

## Как поток JVM блокируется на I/O

Чтобы понять «поток ждёт базу», надо понимать что такое `wait` на уровне OS. Поток JVM — это OS thread (обычно pthread на Linux). У него есть три основных состояния: `RUNNABLE` (реально выполняется на CPU или готов), `BLOCKED` (ждёт monitor/synchronized), `WAITING`/`TIMED_WAITING` (припаркован в userspace через `LockSupport.park` или ушёл в syscall).

Когда JDBC делает `preparedStatement.executeQuery()`, оно в конце превращается в вызов сокета: `SocketChannel.write(query)` + `SocketChannel.read(response)`. `read()` на blocking socket превращается в syscall `recv()` в ядро. Ядро смотрит: если данные в receive buffer сокета есть — копирует и возвращает. Если нет — ставит тред в **wait queue** конкретного сокета и переводит его в состояние `TASK_INTERRUPTIBLE`. Всё, тред не выполняется, CPU освобождается для других тредов.

Как только сетевая карта получает пакет от PostgreSQL, драйвер обрабатывает interrupt, кладёт данные в receive buffer сокета, ядро видит что на этом сокете есть тред в wait queue — будит его, тред переходит в `TASK_RUNNING`, scheduler ставит его в очередь на CPU, `recv()` возвращает данные.

В thread dump JVM это выглядит как:

```
"http-nio-8080-exec-15" #123 daemon prio=5 os_prio=0 tid=0x... nid=0x...
   java.lang.Thread.State: RUNNABLE
        at sun.nio.ch.SocketDispatcher.read0(Native Method)
        at sun.nio.ch.SocketDispatcher.read(SocketDispatcher.java:47)
        at sun.nio.ch.NioSocketImpl.tryRead(...)
        at sun.nio.ch.NioSocketImpl.implRead(...)
        at ...
        at org.postgresql.core.PGStream.receiveChar(PGStream.java:...)
        at org.postgresql.core.v3.QueryExecutorImpl.processResults(...)
        at ...
        at com.zaxxer.hikari.pool.HikariProxyPreparedStatement.executeQuery(...)
        at ru.yourapp.Repository.findById(Repository.java:42)
```

Важно: `Thread.State: RUNNABLE` вводит в заблуждение. JVM показывает RUNNABLE потому что JVM-уровень не знает про blocking syscall — с точки зрения JVM тред «выполняется в native коде». Реально же тред в kernel wait queue, не тратит CPU. Чтобы узнать что тред реально ждёт — смотри stacktrace: если топ фрейма `SocketDispatcher.read0` — тред ждёт сеть.

`LockSupport.park` (то, что HikariCP использует когда пул исчерпан) — это тоже уход в ядро, но через futex. futex — Fast Userspace muTEX, интерфейс ядра для эффективного ожидания на 32-битном слове в памяти. Тред делает `futex(FUTEX_WAIT, addr, expected_value)`, ядро проверяет что `*addr == expected_value`, паркует тред в wait queue связанную с этим адресом. Разбудить — `futex(FUTEX_WAKE, addr)`. В thread dump это будет `Thread.State: WAITING (parking)` с фреймом `jdk.internal.misc.Unsafe.park`.

## I/O wait: что это на самом деле

Открываешь `top` на сервере, видишь строку CPU:

```
%Cpu(s):  3.2 us,  1.1 sy,  0.0 ni, 45.7 id, 49.8 wa,  0.0 hi,  0.2 si,  0.0 st
```

`49.8 wa` = 49.8% времени CPU был в состоянии I/O wait. Что это значит? Половина CPU где-то тормозит на диске? Не совсем.

`%iowait` — это **процент времени, когда CPU был idle И в системе был хотя бы один процесс, ожидающий disk I/O**. Ключевое: «И» — это одновременное условие. CPU **не занят** этим ожиданием. Он свободен, мог бы что-то делать. Просто линуксовый accounting отмечает: «был бы работать, но никого нет, а вон те процессы ждут диск».

Если `%iowait = 50%` и `%idle = 45%`, суммарно 95% CPU **свободен**. Половину этого времени в системе есть тред, ждущий диск. Если бы этот тред мог что-то сделать — CPU бы делал. Но тред спит в wait queue.

Из этого следуют неочевидные штуки:

1. **Высокий iowait не значит перегруз CPU.** Значит: диски медленные (или их мало), плюс кто-то их ждёт. Добавить CPU не поможет. Ускорить диски (SSD, RAID, кэш) — поможет.

2. **Низкий iowait не значит что диски ок.** Если CPU 100% занят user work — счётчик iowait покажет 0, даже если приложение постоянно ждёт диск. Просто в момент когда тред ждёт — другой тред занимает CPU, состояние «idle И waiting for I/O» не наступает.

3. **iowait учитывает только disk I/O**, не сеть. Ожидание TCP-сокета не даёт iowait. Приложение, полностью упершееся в базу по сети, покажет `%iowait = 0` и высокий `%idle` — CPU просто ничего не делает.

4. На виртуалках `%st` (steal time) — то, что hypervisor забрал у твоей VM на других соседей. Ненулевой steal = соседи по хосту гадят. Тоже часто путают с iowait.

Правильный инструмент для диагностики дисков — не `top`, а **`iostat -xz 1`**:

```
Device  r/s  w/s  rkB/s  wkB/s  await  r_await  w_await  %util
nvme0n1 120  340  1800   5100   12.4   3.1      15.8     78.2
```

- `r/s`, `w/s` — IOPS чтения/записи
- `rkB/s`, `wkB/s` — throughput
- `await` — среднее время (мс) от отправки запроса в диск до его завершения, включая очередь. Норма для SSD — <5 мс, для NVMe — <1 мс, для HDD — 5-20 мс
- `%util` — процент времени, когда диск был занят. Для HDD >80% = насыщение. Для NVMe со внутренним параллелизмом даже 100% не всегда насыщение — смотри `await`

`await` — самый честный сигнал. Если он взлетел с 2 мс до 200 мс — диск задыхается. И вот это уже реально бьёт по приложению: каждый запрос к PostgreSQL, требующий чтения страницы с диска (`Buffers: shared read=`, см. файл 88), будет ждать эти 200 мс.

## Как высокий iowait превращается в исчерпанный пул

Цепочка:

1. Диск сервера базы задыхается (`await = 200ms`).
2. PostgreSQL backend, обрабатывающий запрос, вместо 5 мс тратит 300 мс — он ждёт чтения heap-страницы с диска (`wait_event = 'DataFileRead'`).
3. JDBC connection, через который шёл этот запрос, занят все эти 300 мс.
4. С клиента прилетают ещё запросы. Приложение вызывает `hikari.getConnection()` — свободных нет.
5. Пул из 20 соединений занят все, `pending` растёт.
6. Через 30 сек (`connection-timeout`) вызывающий тред получает `SQLException: Connection is not available`.
7. HTTP-запрос возвращает 500. Пользователь видит ошибку.

При этом на самом сервере приложения `%cpu = 5%`, `%iowait = 0` (там нет disk I/O). На сервере базы `%cpu = 10%`, `%iowait = 60%`. Метрики JVM показывают: 20 тредов Tomcat в состоянии RUNNABLE с топ-фреймом `SocketDispatcher.read0` — то есть все ждут сеть от базы. Метрики HikariCP: `active = 20`, `pending = 40`, `usage = 100%`.

Диагноз по метрикам ясный: **пул исчерпан, соединения зависли в ожидании ответа базы**. Причина — не в пуле, а в базе, там диск. Увеличение пула сделает только хуже: 40 backend процессов вместо 20 будут драться за тот же диск, каждый ещё медленнее.

Лечение — в базе: разобрать что читается с диска, добавить индексы (см. 88), покрутить `shared_buffers` чтобы hot данные не вытеснялись, при необходимости — быстрее диски.

## Второй сценарий: HTTP-вызов под транзакцией

Ещё более частая причина исчерпанного пула — код вида:

```java
@Transactional
public void processFno(Long id) {
    Fno fno = fnoRepository.findById(id).orElseThrow();
    fno.setStatus(PROCESSING);
    fnoRepository.save(fno);

    // а теперь идём в внешний сервис
    ExternalResponse resp = externalClient.callSomeSlowApi(fno);  // 5 секунд

    fno.setStatus(DONE);
    fno.setExternalCode(resp.getCode());
    fnoRepository.save(fno);
}
```

Что происходит: `@Transactional` открыл транзакцию → взял connection из HikariCP → выполнил SELECT → выполнил UPDATE → **держит connection все 5 секунд HTTP-вызова** → второй UPDATE → COMMIT → connection возвращается.

Если этот метод вызывается 50 раз в секунду, а `maximum-pool-size = 20` — Little's Law: `L = 50 × 5 = 250`. Нужно 250 соединений, есть 20. Пул мгновенно забит, всё висит.

Симптомы: `hikaricp_connections_active` пилообразный, упирается в max. Thread dump: 20 тредов в `SocketDispatcher.read0` где стек ведёт не в JDBC, а в HTTP-клиент (`HttpURLConnection`, `RestTemplate`, `WebClient` в blocking режиме). Это ключевой отличающий признак: соединение JDBC не работает с базой, но занято.

На стороне PostgreSQL эти соединения видны как `state = 'idle in transaction'` в `pg_stat_activity`, `xact_start` был секунды назад, но `query_start` тоже секунды назад — то есть транзакция открыта, последний запрос давно закончился, база тупо ждёт следующего команды от клиента. `wait_event = 'ClientRead'` — база ждёт чтения от клиента.

**ClientRead в `pg_stat_activity` = приложение держит транзакцию открытой и что-то делает вне базы.** Это красный флаг всегда.

Лечение — код:

```java
public void processFno(Long id) {
    // короткая транзакция №1 — только чтение и mark as PROCESSING
    Fno fno = markProcessing(id);

    // HTTP вне транзакции, connection свободен для других
    ExternalResponse resp = externalClient.callSomeSlowApi(fno);

    // короткая транзакция №2 — записать результат
    markDone(id, resp);
}

@Transactional
protected Fno markProcessing(Long id) {
    Fno fno = fnoRepository.findById(id).orElseThrow();
    fno.setStatus(PROCESSING);
    return fnoRepository.save(fno);
}

@Transactional
protected void markDone(Long id, ExternalResponse resp) {
    Fno fno = fnoRepository.findById(id).orElseThrow();
    fno.setStatus(DONE);
    fno.setExternalCode(resp.getCode());
    fnoRepository.save(fno);
}
```

Правило: **никогда не делать блокирующие вызовы (HTTP, файлы, RabbitMQ publish в persistent режиме) внутри `@Transactional`**. Транзакция должна брать connection, делать SQL, освобождать. Всё остальное — снаружи.

## Deadlock на пуле

Хитрый случай: приложение делает вложенные транзакции с разными datasource'ами или с одним datasource через раз, и упирается в deadlock на самом пуле.

Пример: сервис берёт connection, потом внутри дёргает другой сервис, который тоже берёт connection. Если пул из 10 и 10 запросов одновременно взяли по одному connection и все ждут второго — deadlock. Ни один не отпустит первый, пока не получит второй, а второго не будет пока кто-то не отпустит первый.

Симптом: пул исчерпан, все треды в `getConnection()`, `hikaricp_connections_pending` = число тредов. Через 30 сек — timeout у всех. Через минуту — то же самое.

Правило: **один HTTP-запрос = максимум одно соединение одновременно**. Если нужно два — плохой дизайн, надо переписать (обычно вызов вложенного метода в том же треде на том же connection через `@Transactional(propagation = REQUIRED)` — Spring не будет брать второй connection, а переиспользует).

## thread-per-request vs virtual threads

До Java 21 модель была строгая: один HTTP-запрос обрабатывается одним OS thread. Tomcat/Undertow держат пул тредов (обычно 100-200), каждый запрос садится на один, все blocking I/O внутри блокируют этот тред. Максимум concurrent запросов = размер пула тредов. Дальше очередь.

С JDBC это работает предсказуемо: `hikari.maximum-pool-size = 20`, tomcat threads = 200 → одновременно 20 запросов реально выполняют SQL, ещё 180 могут ждать в очереди на пул или заниматься чем-то не связанным с базой. Соотношение чётко видно.

Virtual threads (JEP 444, Java 21, см. файл 19) меняют картину. Виртуальный тред — не OS thread, а объект в куче JVM, шедулится на пул платформенных тредов (carrier threads, обычно = число CPU). Виртуальных тредов может быть миллион. Когда виртуальный тред делает blocking I/O, JVM **отпаркует его с carrier thread** (unmount), carrier переходит к другому виртуальному треду. Когда I/O завершилось — виртуальный тред снова маунтится, продолжает выполнение.

С сокетами это работает автоматически — JDK перевёл `SocketChannel` под virtual-thread-aware механизм: `read()` на blocking socket из виртуального треда → JVM внутри делает non-blocking read + регистрирует интерес в `epoll`, паркует виртуальный тред, освобождает carrier. Приложение видит обычный blocking API, а под капотом асинхронщина.

Но есть **синхронизация pinning**: если виртуальный тред блокируется внутри `synchronized` блока или в JNI-вызове — он **не может отпарковаться**, carrier застревает. Для JDBC это критично: `HikariCP`, `postgres jdbc driver` — исторически используют `synchronized` в некоторых hot paths (в новых версиях частично переведены на ReentrantLock). Если виртуальный тред делает `preparedStatement.executeQuery()` внутри synchronized (например в самом драйвере) — carrier застрянет на время I/O, вся идея virtual threads разваливается.

Практически на 2026 год: с драйверами PostgreSQL 42.7+ и HikariCP 5.1+ большинство pinning'ов убрано, virtual threads с JDBC работают. Но connection pool всё ещё нужен — база не масштабируется до миллиона connections. Схема становится: миллион виртуальных тредов, 20 physical соединений в пуле. Виртуальный тред, который не может получить connection — паркуется, carrier освобождается для других. HikariCP `getConnection` с виртуальным тредом уже не блокирует carrier (использует `LockSupport.park`, который virtual-thread aware).

Итог: **connection pool не отменяется virtual threads**. Отменяется только огромный tomcat thread pool — вместо 200 platform threads можно иметь миллион virtual threads, но пул к базе всё равно 20-30, и всё равно надо думать про Little's Law.

## Диагностика в проде: чек-лист

**Симптом:** HTTP-latency улетела, приложение отвечает 500. Начинаем.

1. **HikariCP метрики** (Prometheus / Micrometer):
   - `hikaricp_connections_active` — сколько занято сейчас
   - `hikaricp_connections_idle` — сколько свободно
   - `hikaricp_connections_pending` — сколько тредов ждут пул
   - `hikaricp_connections_timeout_total` — счётчик таймаутов getConnection
   - `hikaricp_connections_usage_seconds` — сколько connection живёт от borrow до return

   Если `active == max` и `pending > 0` — пул исчерпан. Если `timeout_total` растёт — треды не дожидаются.

2. **Thread dump** приложения: `jstack <pid> > dump.txt`. Найти в дампе:
   - все треды в состоянии `WAITING (parking)` с фреймом `com.zaxxer.hikari.pool.HikariPool.createTimeoutException` или `getConnection` — ждут пул
   - все треды в `RUNNABLE` с `SocketDispatcher.read0` в топе — ждут сеть; смотри стек ниже: если ведёт в JDBC → ждут базу; если в HTTP-клиент → ждут внешний сервис под транзакцией

3. **PostgreSQL `pg_stat_activity`** на стороне базы:
   ```sql
   SELECT pid, state, wait_event_type, wait_event,
          NOW() - xact_start AS xact_age,
          NOW() - query_start AS query_age,
          left(query, 100) AS q
   FROM pg_stat_activity
   WHERE datname = 'your_db' AND state != 'idle'
   ORDER BY xact_start;
   ```
   - Много `state = 'idle in transaction'` с большим `xact_age` — приложение держит транзакции. Красный флаг.
   - `wait_event = 'ClientRead'` — база ждёт клиента (приложение занято вне базы под транзакцией).
   - `wait_event_type = 'IO'`, `wait_event = 'DataFileRead'` — база ждёт диск. Проверь iostat на сервере базы.
   - `wait_event_type = 'Lock'` — блокировка (см. файл 88, `pg_blocking_pids`).

4. **iostat -xz 1** на сервере базы: `await` > 20 мс = диск задыхается. Проверь `%util` (>80% на одном диске = насыщение).

5. **ss -tan | grep :5432 | wc -l** на сервере приложения: сколько реально TCP-соединений к базе. Должно совпадать с `hikaricp_connections_active + idle`. Если сильно меньше — драйвер рвёт соединения. Если больше — leak.

6. **top / htop** на приложении: если `%us` низкий (<20%), а latency плохая — приложение ждёт I/O, а не считает. Если `%us` высокий — CPU-bound, дело не в пуле.

Комбинируя эти шесть источников, картина восстанавливается быстро. Пример реального разбора:

- `hikaricp_connections_active = 20`, `pending = 60`, timeout'ы летят.
- Thread dump: 20 тредов в `SocketDispatcher.read0`, стек ведёт в JDBC → `PgResultSet.next`.
- `pg_stat_activity`: 20 сессий в `active`, `wait_event = 'DataFileRead'`.
- `iostat`: `await = 380 мс`, `%util = 99%` на data-диске.
- Диагноз: **диск задыхается**. Пул тут не при чём.
- Дальнейшее расследование: `pg_stat_statements` по `shared_blks_read` → топ-запрос делает Seq Scan по большой таблице без индекса → `EXPLAIN ANALYZE` → добавить индекс → диск успокаивается → пул освобождается → latency восстанавливается.

## Timeouts: что стоит по чему

Настройки timeout'ов формируют цепочку — каждый уровень должен быть меньше следующего, иначе получится каскад висящих запросов.

HikariCP:
- `connection-timeout` (default 30s) — сколько ждать `getConnection()` пока пул свободный не появится. **Уменьшать до 3-5s** для fail-fast семантики. 30 секунд — это очень много: пользователь давно уже ушёл, а тред всё ждёт.
- `validation-timeout` (default 5s) — сколько ждать `SELECT 1` при валидации connection.
- `max-lifetime` (default 30 min) — сколько живёт connection. Должен быть **меньше** любых upstream timeout'ов: `wait_timeout` PostgreSQL, idle-timeout L4 балансера, firewall session timeout. Иначе получишь мёртвый connection в пуле, первый запрос упадёт с `Connection reset`.
- `keepalive-time` (default 5 min) — период `SELECT 1` для idle connection. Должен быть **меньше** idle-timeout балансера/firewall (типично 10-15 мин на AWS ELB, 60s на некоторых enterprise firewall).

JDBC (setter'ы на драйвере или через url params):
- `loginTimeout` — сколько ждать TCP+SSL+auth. Ставь 5-10 сек. Иначе при недоступной базе тред застрянет на десятки секунд ядерного `connect()`.
- `socketTimeout` — SO_TIMEOUT на сокете, максимальное время одного `recv()`. Если запрос выполняется дольше — SQLException. **Ставить обязательно** (например 30-60 сек), иначе завис базы = навсегда висящий тред.
- `tcpKeepAlive = true` — включает TCP-уровневый keepalive. Дополнительная защита от «мёртвых» соединений при обрыве сети посередине.

PostgreSQL:
- `statement_timeout` — прибивает запрос дольше N мс. Ставится на сессию (`SET statement_timeout = '30s'` в connection-init-sql HikariCP) или в `postgresql.conf` глобально. Защита от runaway-запросов.
- `idle_in_transaction_session_timeout` — прибивает сессию, которая держит транзакцию открытой без активности. **Обязательно ставить** (60-120 сек) — защита от кода, забывающего COMMIT.
- `lock_timeout` — сколько ждать lock перед фейлом. Ставь на миграциях (см. файл 88).

Цепочка должна быть согласованной: `HTTP-клиент timeout > socketTimeout > statement_timeout > обычный запрос`. Если socketTimeout меньше statement_timeout — драйвер разрывает связь, а база продолжает выполнять запрос впустую (потом откатывает, теряет работу). Если больше — база отменит, а драйвер ещё сидит и ждёт «может ответит».

## Метрики, за которыми смотреть постоянно

В стабильном dashboard приложения по connection pool должно быть:

- **`hikaricp_connections_active` (gauge)** — как далеко от max. Норма: <50%. Устойчивый 100% = пул мал ИЛИ проблема в базе/коде.
- **`hikaricp_connections_pending` (gauge)** — очередь ожидания. Норма: 0. Любое ненулевое значение — сигнал.
- **`hikaricp_connections_usage_seconds` (histogram)** — сколько connection «в руках» приложения. Норма: p95 < 100 мс. Большие значения = длинные транзакции или блокирующие вызовы под транзакцией.
- **`hikaricp_connections_acquire_seconds` (histogram)** — сколько ждали `getConnection`. Норма: p99 < 10 мс. Если растёт — пул начинает быть узким местом.
- **`hikaricp_connections_timeout_total` (counter)** — счётчик SQLException по таймауту. Норма: 0. Любые тикающие таймауты = ЧП.

На стороне PostgreSQL:
- Количество connection: `SELECT count(*) FROM pg_stat_activity` — не должно приближаться к `max_connections`.
- Активные транзакции по возрасту: `SELECT max(EXTRACT(EPOCH FROM NOW() - xact_start)) FROM pg_stat_activity WHERE state != 'idle'` — не должно расти неограниченно.
- Долгие `idle in transaction`: `SELECT count(*) FROM pg_stat_activity WHERE state = 'idle in transaction' AND NOW() - state_change > interval '30 seconds'` — норма 0.

На уровне OS (сервер базы):
- `node_disk_io_time_seconds_total` (Prometheus node_exporter) — util дисков.
- `node_disk_await` (или считать из read/write time + IOPS) — latency I/O.
- `node_cpu_seconds_total{mode="iowait"}` — тот самый iowait из top.

Алерты:
- `hikaricp_connections_timeout_total > 0` за 5 минут → paging on-call.
- `hikaricp_connections_pending > 5` устойчиво минуту → warning.
- `pg_stat_activity` count > 80% от max_connections → warning.
- `node_disk_await > 50ms` устойчиво 3 минуты → warning на DBA.

## Заключение

Connection pool — не «магический ящик который делает быстрее», а очередь с семафором, где потоки блокируются когда объектов не хватает. Правильный размер пула определяется Little's Law и особенностями базы (число ядер, число дисков), обычно намного меньше чем интуитивно кажется — 10-30 хватает подавляющему большинству приложений. «Больше пул = лучше» — миф; больше активных connection = больше борьбы в базе за одни и те же ресурсы, latency растёт.

I/O wait в top — не мера тормозов, а мера ситуации «CPU idle, но кто-то ждёт диск». Ноль iowait не означает что диск ок, если CPU полностью нагружен user work. Ненулевой iowait не означает CPU-проблему — CPU свободен, диск медленный. Настоящая диагностика дисков — `iostat -xz 1` с колонкой `await` (норма для SSD < 5 мс).

Поток JVM «ждущий базу» — это OS-тред в kernel wait queue сокета, `Thread.State: RUNNABLE` с топ-фреймом `SocketDispatcher.read0`. Не занимает CPU. Через futex-парковку связаны `LockSupport.park`, `synchronized`, ожидание на HikariCP handoff queue. Virtual threads с Java 21 не отменяют пул — база всё равно не масштабируется до миллиона connection, — но убирают необходимость большого пула Tomcat-тредов.

Инцидент «пул исчерпан» лечится **расследованием** а не увеличением пула. Смотри цепочку: HikariCP метрики → thread dump приложения → `pg_stat_activity` → `iostat` на сервере базы. Найди где реально ждёт: в HTTP-клиенте под транзакцией, в блокировке (`pg_blocking_pids`), в диске (`wait_event = DataFileRead`), в сети между apps и базой. Каждая причина лечится своим способом: код (короткие транзакции, HTTP вне `@Transactional`), индексы, `shared_buffers`, диски, миграция.

Timeout'ы — обязательные, каскадные, короткие. Дефолтные 30 секунд HikariCP `connection-timeout` — это боль, ставь 3-5 сек. `socketTimeout` в JDBC — обязателен, иначе один зависший запрос = навсегда мёртвый тред. `idle_in_transaction_session_timeout` в PostgreSQL — обязателен, страховка от забытого COMMIT. Правило: любой блокирующий вызов должен уметь фейлиться быстро, а не висеть надеждой.

Метрики HikariCP и `pg_stat_activity` — базовая observability. Без них разбор инцидента становится гаданием по кофейной гуще. С ними — за пять минут понимаешь: пул мал, база стонет, диск лёг, кто-то забыл COMMIT, кто-то дёргает HTTP под транзакцией. Дальше — конкретный фикс.
