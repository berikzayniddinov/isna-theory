# 69. TCP connection, батчи, flushing

Как связаны буферизация, батчинг и flush. Разные уровни, одна идея.

---

## 1. Общая идея

**Отправить 1000 сообщений по одному — дорого**. Каждый — свой overhead (system call, TCP handshake, БД round-trip).

**Отправить одним пакетом — эффективно**. Overhead один раз.

Отсюда паттерны:
- **TCP buffer** — kernel накапливает данные перед отправкой.
- **Batching** — приложение накапливает операции.
- **Flushing** — принудительная отправка накопленного.

Разберём каждый уровень.

---

## 2. Что такое TCP connection

**TCP (Transmission Control Protocol)** — надёжный byte-stream протокол над IP.

TCP-соединение = **виртуальный канал** между двумя парами (IP, port).

Свойства:
- **Reliable** — гарантия доставки, повторная отправка потерянных пакетов.
- **Ordered** — байты приходят в том же порядке что отправлены.
- **Full-duplex** — оба направления одновременно.
- **Flow control** — receiver говорит sender'у "стоп, я не успеваю".
- **Congestion control** — не забивать сеть.

Байты, не сообщения — приложение само отделяет "конец записи".

### 2.1 3-way handshake (открытие)

```
Client                                  Server
   │                                       │
   │────────── SYN (seq=x) ──────────────►│
   │                                       │
   │◄──────── SYN-ACK (seq=y, ack=x+1) ──│
   │                                       │
   │────────── ACK (ack=y+1) ────────────►│
   │                                       │
Connection established
```

3 пакета. Занимает **1 RTT** (round-trip time).

- **local network**: ~1 ms.
- **same datacenter**: ~0.5 ms.
- **cross-region**: 50-200 ms.

Отсюда — **connection pool** и **keep-alive** обязательны.

### 2.2 4-way handshake (закрытие)

```
Client                                  Server
   │                                       │
   │────────── FIN ─────────────────────►│  "я закончил слать"
   │                                       │
   │◄──────── ACK ───────────────────────│
   │                                       │
   │◄──────── FIN ───────────────────────│  "я тоже закончил"
   │                                       │
   │────────── ACK ─────────────────────►│
   │                                       │
Connection closed
```

Каждая сторона независимо закрывает своё направление.

### 2.3 TIME_WAIT

После close — TCP-порт **не сразу освобождается**. Остаётся в состоянии `TIME_WAIT` (обычно 60 сек — 2 × MSL).

Почему: если пришёл задержавшийся пакет из старого соединения — не должен попасть в новое соединение с тем же portом.

**Проблема**: сервер с высоким RPS открывает и закрывает много TCP → **thousands of TIME_WAIT** → исчерпание портов.

Fix:
- **HTTP keep-alive** — переиспользовать соединение.
- **Connection pool** — не открывать заново.
- `net.ipv4.tcp_tw_reuse=1` (Linux) — переиспользовать TIME_WAIT.

### 2.4 TCP buffer

Каждое TCP-соединение имеет **буферы** в kernel:
- **Send buffer** — данные, ждущие отправки.
- **Receive buffer** — данные, ждущие чтения приложением.

```
Application → write() → Send buffer → TCP-пакеты → network
```

`write()` возвращает сразу — данные в буфере. Kernel сам отправляет.

Настройки Linux:
```
net.core.rmem_max = 16777216       # 16 MB max receive
net.core.wmem_max = 16777216       # 16 MB max send
net.ipv4.tcp_rmem = 4096 87380 16777216   # min default max
net.ipv4.tcp_wmem = 4096 65536 16777216
```

При переполнении receive buffer у receiver'а → sender замедляется (**flow control** через TCP window).

### 2.5 Nagle's algorithm

TCP пытается **склеить** маленькие пакеты, чтобы не отправлять по 1 байту.

Правило: не отправлять новый маленький пакет пока предыдущий не ack'нут.

Плюсы: меньше overhead.
Минусы: **задержка** (сотни ms).

Для интерактивных приложений (SSH, gaming) — плохо.

### 2.6 TCP_NODELAY

Отключить Nagle:
```java
Socket socket = ...;
socket.setTcpNoDelay(true);
```

Пакет отправляется **сразу**. Больше пакетов, но меньше latency.

Обычно включают для:
- HTTP-серверов.
- Real-time (WebSocket, gRPC).
- Приложений с низкой latency.

### 2.7 TCP keep-alive

Периодические пробы что соединение живо (даже без трафика).

```java
socket.setKeepAlive(true);
```

Настройки Linux:
```
net.ipv4.tcp_keepalive_time = 7200   # начать пробить через 2 часа idle
net.ipv4.tcp_keepalive_intvl = 75    # каждые 75 сек
net.ipv4.tcp_keepalive_probes = 9    # 9 попыток
```

Полезно для длительных соединений (JDBC, gRPC streaming), чтобы обнаружить обрыв.

### 2.8 HTTP Keep-Alive (не путать)

HTTP/1.1 — **connection: keep-alive** header. TCP-соединение **не закрывается** после ответа.

Следующий HTTP-запрос переиспользует TCP → **нет handshake** → быстрее.

Экономит 1 RTT на запрос. Big deal при cross-region.

HTTP/2 — multiplexing (много запросов через одно соединение параллельно).

---

## 3. Что такое батчи

**Batch (batching)** — объединение **множества операций в одну**.

Идея: overhead per-operation фиксированный. Много операций пачкой = разделить overhead.

### 3.1 Универсальный принцип

```
1000 ops по одной:
  op1: overhead + work
  op2: overhead + work
  ...
  op1000: overhead + work
  = 1000 × overhead + 1000 × work

1000 ops батчем:
  batch: overhead + 1000 × work
  = 1 × overhead + 1000 × work
```

Overhead может быть:
- Round-trip к серверу.
- Открытие соединения.
- System call.
- Lock.

---

## 4. JDBC / Hibernate batch

### 4.1 Без batch

```java
for (Order o : list) {
    em.persist(o);
    em.flush();
}
```

SQL:
```sql
INSERT INTO orders VALUES (...);   -- round-trip
INSERT INTO orders VALUES (...);   -- round-trip
...
```

Каждый INSERT — round-trip к БД (1-10 мс). 1000 orders = 10 секунд.

### 4.2 С batch

Hibernate:
```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
```

Внутри `PreparedStatement.addBatch()`:
```java
PreparedStatement ps = conn.prepareStatement("INSERT INTO orders VALUES (?, ?, ?)");
for (Order o : list) {
    ps.setLong(1, o.id);
    ps.setString(2, o.customerId);
    ps.setBigDecimal(3, o.amount);
    ps.addBatch();

    if (++count % 50 == 0) {
        ps.executeBatch();   // одним round-trip
        ps.clearBatch();
    }
}
ps.executeBatch();   // остатки
```

Ускорение: 1000 INSERT'ов → 20 batch'ей → ~200 ms вместо 10 sec. **50× быстрее**.

### 4.3 Кавет IDENTITY vs SEQUENCE

Уже разбирали в `12-jpa-basics.md`, `13-hibernate-internals.md`.

- **IDENTITY** — Hibernate вынужден INSERT сразу для получения id → **batch не работает**.
- **SEQUENCE** — можно взять id заранее → batch OK.

**В PG обычно SEQUENCE** — batch работает.

### 4.4 rewriteBatchedInserts (PG-специфичное)

```yaml
spring.datasource.hikari.data-source-properties:
  reWriteBatchedInserts: true
```

PG JDBC-драйвер объединяет:
```sql
INSERT INTO orders VALUES (v1), (v2), (v3), ..., (v50);
```

Ещё быстрее.

---

## 5. Kafka producer batch

### 5.1 Настройки

```
linger.ms=10          # ждать до 10 мс, чтобы накопить batch
batch.size=32768      # max размер batch (32 KB)
compression.type=zstd # сжать batch
```

### 5.2 Как работает

Producer:
1. `send()` — сообщение в буфер (не сразу отправляется).
2. Buffer наполняется до `batch.size` **или** прошло `linger.ms`.
3. Batch отправляется broker'у одним запросом.
4. Broker записывает.

Ускорение: 100k msg/sec без batch → 500k+ с batch. **5-10× rps**.

Плюс compression — batch сжимается лучше чем отдельные сообщения.

### 5.3 Trade-off

- **linger.ms=0** — минимум latency (send сразу), плохой throughput.
- **linger.ms=100** — max throughput, +100 мс latency.

Обычный компромисс: **linger.ms=5-20**.

### 5.4 Кейсы ИСНА

Kafka не массово используется, но принцип тот же для Rabbit publisher batching.

---

## 6. HTTP batch

### 6.1 Проблема

REST API часто позволяет вернуть один объект: `GET /users/1`. Для 1000 users — 1000 запросов.

### 6.2 Batch endpoints

Дизайн:
```
POST /users/batch
Body: [{"id": 1}, {"id": 2}, ..., {"id": 1000}]

Response: [{...}, {...}, ..., {...}]
```

Один запрос, 1000 объектов.

### 6.3 GraphQL

Аналог: один запрос — много ресурсов.

### 6.4 gRPC streaming

`server streaming` — клиент шлёт 1 запрос, сервер отвечает stream ответов.

---

## 7. Bulk operations в БД

Похоже на batch, но one SQL для многих строк.

### 7.1 Bulk INSERT

```sql
INSERT INTO orders (id, customer_id) VALUES
    (1, 'c1'),
    (2, 'c2'),
    ...
    (1000, 'c1000');
```

Один SQL, 1000 строк.

### 7.2 Bulk UPDATE

```sql
UPDATE orders SET status = 'PROCESSED'
WHERE id IN (1, 2, 3, ..., 1000);
```

### 7.3 COPY (PG-специфичное)

```
COPY orders FROM STDIN CSV;
1,c1,100
2,c2,200
...
\.
```

Самый быстрый способ загрузить много строк. В **сотни раз** быстрее INSERT.

Для reports / ETL / initial data load.

---

## 8. Что такое flushing

**Flush** — **принудительная отправка** накопленного (в buffer / batch) сейчас.

Идея: buffer накапливает; flush отправляет; после flush — buffer пуст.

### 8.1 Зачем нужен

- Гарантировать что данные ушли.
- До закрытия ресурса.
- Периодически при долгой работе.

---

## 9. Flushing на разных уровнях

### 9.1 Java OutputStream.flush()

```java
OutputStream out = ...;
out.write(data);
out.flush();   // отправить сразу
```

`BufferedOutputStream` буферизует. Без `flush()` — данные могут не уйти пока buffer не наполнен.

`close()` тоже flush'ит.

### 9.2 Java Writer.flush()

Аналогично для символьных потоков.

### 9.3 Logback flush

Async appender имеет очередь:
```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>512</queueSize>
    <neverBlock>false</neverBlock>
</appender>
```

При shutdown — flush остатков. При падении JVM — можно потерять.

### 9.4 TCP flush

TCP имеет флаг **PSH (push)** — receiver должен передать данные приложению сразу.

Обычно kernel сам решает. Приложение через `TCP_NODELAY` может форсировать.

### 9.5 fsync (диск)

При записи на диск kernel буферизует (**page cache**). `write()` возвращается быстро — данные ещё в RAM.

`fsync(fd)` — принудительная запись на диск. Медленно (миллисекунды на HDD, микросекунды на SSD), но гарантия durability.

БД commits вызывают `fsync` на WAL — отсюда PG commit не мгновенный.

### 9.6 Hibernate flush

Уже разбирали в `13-hibernate-internals.md`.

```java
em.persist(o1);
em.persist(o2);
// изменения в PersistenceContext, ещё не в БД

em.flush();
// SQL INSERTs летят в БД, но tx ещё не commit
```

Flush ≠ commit. Flush отправляет SQL, commit фиксирует.

Auto-flush происходит:
- Перед commit.
- Перед query (чтобы query видел изменения).

Manual — для контроля момента отправки SQL.

### 9.7 Kafka producer flush

```java
producer.send(record1);
producer.send(record2);
producer.send(record3);
// ушли в buffer, не обязательно на broker

producer.flush();   // ждать пока все batch'и отправятся
producer.close();
```

**Всегда** flush + close перед завершением приложения — иначе последние сообщения могут потеряться.

### 9.8 RabbitMQ

Не совсем flush, но `waitForConfirms()` — ждать подтверждения publish'ей.

---

## 10. Связь buffer / batch / flush

Одна идея на разных уровнях:

```
Application

   ┌── Batch (Hibernate/Kafka/JDBC) ──┐
   │                                   │
   │   ┌── Application buffer ─────┐   │
   │   │                            │   │
   │   │   ┌── TCP buffer ─────┐   │   │
   │   │   │                    │   │   │
   │   │   │   Network         │   │   │
   │   │   │                    │   │   │
   │   │   └────────────────────┘   │   │
   │   │                            │   │
   │   └────────────────────────────┘   │
   │                                   │
   └───────────────────────────────────┘
```

- **Buffer** — временное хранилище.
- **Batch** — операции накапливаются в buffer.
- **Flush** — buffer отправляется.

Аналогия: почтовый ящик, конверты, забор почты.

---

## 11. Flush батчей

Отдельно проговорим — специфичный термин.

**Flush batch** = **отправить накопленный batch сейчас, не ждать заполнения**.

### 11.1 JDBC

```java
PreparedStatement ps = conn.prepareStatement("...");
for (int i = 0; i < 100; i++) {
    ps.setLong(1, i);
    ps.addBatch();
}
int[] results = ps.executeBatch();   // flush batch
```

`executeBatch()` = flush.

### 11.2 Hibernate

```java
for (int i = 0; i < 1000; i++) {
    em.persist(new Order(...));
    if (i % 50 == 0) {
        em.flush();   // отправить batch
        em.clear();   // освободить memory
    }
}
```

Периодический flush + clear в batch processing:
- **flush**: отправить SQL накопленных изменений.
- **clear**: очистить PersistenceContext (иначе OOM).

Обязательный паттерн для больших batches (см. `13-hibernate-internals.md`, `15-jpa-performance.md`).

### 11.3 Kafka

```java
for (int i = 0; i < 10000; i++) {
    producer.send(new ProducerRecord<>("topic", "msg-" + i));
    if (i % 1000 == 0) {
        producer.flush();
    }
}
producer.flush();
producer.close();
```

Периодический flush гарантирует что все accumulated batches отправлены.

---

## 12. Trade-offs

Батчинг + буферизация ускоряют, но:

### 12.1 Latency vs Throughput

- **Больше batch** → **больше throughput**, но **больше latency** (ждём наполнения).
- **Меньше batch** → быстрее реакция, но меньше throughput.

Компромисс: `linger.ms` для Kafka, `batch_size` для Hibernate.

### 12.2 Memory

- Больше buffer = больше памяти.
- Kafka producer: `buffer.memory=33554432` (32 MB).
- Hibernate: `PersistenceContext` растёт → OOM.

### 12.3 Durability

- Batching откладывает отправку → при crash **данные в buffer теряются**.
- Async logger — падение JVM теряет queue.

**Правило**: критичные data — flush чаще (или sync). Некритичные — batch агрессивнее.

### 12.4 Ordering

Batch может изменить порядок отправки (если несколько потоков пишут).

Для strict ordering — single-threaded + FIFO batch.

---

## 13. Реальные кейсы ИСНА

Из memory:

- **`knp-fno21-shedlock-stale-image-dup-regnum`** — дубли reporting_incoming_number, потому что без ShedLock scheduled job лупился параллельно и делал дублирующие INSERT'ы. Batch не помог бы — проблема координации.
- **`knp-filter-sent-documents-perf`** — sync RestTemplate блокирует downstream + новый HTTPS-connect на запрос. Если бы был connection pool + keep-alive → не пришлось бы делать TCP handshake каждый раз.
- **Fno328 регенерация** (`Fno328PdfRegenerationJob`) — типичный batch job где нужен flush + clear в цикле.

---

## 14. Best practices

1. **Connection pool** для БД / HTTP клиентов — избегать TCP overhead.
2. **HTTP Keep-Alive** — переиспользовать TCP.
3. **TCP_NODELAY** для low-latency HTTP.
4. **JDBC batch_size = 50** обычно оптимально.
5. **SEQUENCE PK** (не IDENTITY) — для JPA batch.
6. **Kafka linger.ms=5-20** — компромисс latency/throughput.
7. **Kafka batch.size = 32 KB** default норм.
8. **Compression zstd** для Kafka batch.
9. **Hibernate flush + clear** каждые 50-100 в batch job.
10. **producer.flush() + close()** перед shutdown.
11. **fsync** осмысленно — не всегда нужен.
12. **Мониторить** connection count / buffer utilization.
13. **Graceful shutdown** — flush всех async буферов.

---

## 15. Собесные вопросы

1. **Что такое TCP connection?** — Виртуальный канал (IP, port) → (IP, port); reliable byte-stream.
2. **3-way handshake?** — SYN → SYN-ACK → ACK; 1 RTT.
3. **Что такое TIME_WAIT?** — Состояние после close (~60 сек); защита от задержавшихся пакетов.
4. **Что такое TCP buffer?** — Kernel-level буфер send/receive per-connection.
5. **Nagle's algorithm?** — Склеивает маленькие пакеты для эффективности; TCP_NODELAY отключает.
6. **HTTP Keep-Alive?** — Переиспользовать TCP-соединение между HTTP-запросами; экономит handshake.
7. **Что такое batching?** — Объединение множества операций в одну; разделяет overhead.
8. **Как настроить Hibernate batch?** — `hibernate.jdbc.batch_size=50` + `order_inserts=true` + SEQUENCE PK.
9. **Почему IDENTITY ломает JDBC batch?** — Hibernate вынужден INSERT сразу для получения id.
10. **Что такое flushing?** — Принудительная отправка накопленного buffer / batch.
11. **flush vs commit в JDBC?** — flush отправляет SQL; commit фиксирует tx.
12. **Что такое Kafka linger.ms?** — Ждать до N ms для наполнения batch; larger = более throughput, менее latency.
13. **fsync — что делает?** — Форсированная запись page cache на диск; медленно, но durable.
14. **Trade-off batching?** — Throughput vs latency vs memory vs durability при crash.
15. **Что делать перед shutdown Kafka producer?** — `producer.flush()` + `producer.close()` — иначе потеря сообщений в buffer.

---

## Итог

- **TCP connection**: 3-way handshake, buffer, keep-alive, TIME_WAIT.
- **HTTP Keep-Alive + connection pool** — must для performance.
- **Batching** — универсальный принцип: разделить overhead на N операций.
- **JDBC batch**, **Kafka batch**, **HTTP batch endpoints**.
- **Flushing** — принудительная отправка накопленного.
- **Flush batches** периодически в long-running jobs.
- **Trade-offs**: latency vs throughput vs memory vs durability.
- Правило: **batch для throughput, flush для гарантии + memory control**.

Итого **69 файлов** в `isna-theory\`.
