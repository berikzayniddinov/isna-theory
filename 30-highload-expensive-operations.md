# 30. Дорогостоящие операции в highload проектах

Что убивает производительность. Как узнать что дорого, что дешёво. Правила highload.

---

## 1. Иерархия «дороговизны» операций

Ориентировочные времена (для средней машины):

```
Регистр CPU                ~1 нс          × 1
L1 cache                   ~1 нс          × 1
L2 cache                   ~4 нс          × 4
L3 cache                   ~15 нс         × 15
RAM                        ~100 нс        × 100
NVMe SSD (random)          ~100 мкс       × 100 000
Network (local DC)         ~500 мкс       × 500 000
HDD (random)               ~10 мс         × 10 000 000
Network (cross-region)     ~150 мс        × 150 000 000
```

Разница между L1 и HDD — **7 порядков** (10 миллионов раз).

**Правило**: чем ближе к CPU, тем дешевле. Всё что «идёт наружу» (диск, сеть, БД) — дорого.

---

## 2. Что реально дорого в highload

### 2.1 Синхронный HTTP-вызов внутри транзакции ⭐⭐⭐

```java
@Transactional
void submit(Fno f) {
    repo.save(f);
    externalApi.call(f);   // ← 30 сек ждём внешний API, ВСЁ ЭТО ВРЕМЯ transaction ОТКРЫТА
    audit(f);
}
```

Последствия:
- Connection в пуле занят 30 сек.
- Row locks в БД держатся.
- Deadlock'и растут.
- Пул исчерпается → 500 на новых запросах.
- Прод падает.

**Правило**: **никаких сетевых вызовов внутри `@Transactional`**. Только БД.

Fix:
```java
void submit(Fno f) {
    doSave(f);                // короткая tx
    externalApi.call(f);      // снаружи tx
}
@Transactional
void doSave(Fno f) { repo.save(f); }
```

Или **outbox pattern**: сохранить в БД + запись «отправь этому API», отдельный job подхватит.

### 2.2 N+1 запросы ⭐⭐⭐

Обсуждали в `15-jpa-performance.md`. Каждый лишний SQL = round-trip = 1-5 мс. 1000 SELECT вместо одного JOIN = +5 сек latency.

Лечится: JOIN FETCH / EntityGraph / DTO projection / @BatchSize.

### 2.3 Отсутствие индекса (Seq Scan) ⭐⭐⭐

Таблица 10M строк, запрос без индекса → PG читает все 10M строк.
С индексом → 10-100 строк.

Разница 100 000× в latency.

Проверить: `EXPLAIN ANALYZE`. Ищи `Seq Scan` на больших таблицах.

### 2.4 SELECT COUNT(*) на большой таблице

PG вынужден прочитать все строки (для MVCC — нельзя просто взять число из header). На таблице 10M — до нескольких секунд.

Fixes:
- Кэшировать (обновлять периодически).
- `EXPLAIN` — approximate count.
- Использовать `Slice` вместо `Page` (без COUNT).
- Секционировать таблицу.
- В аналитике — материализованный view.

### 2.5 OFFSET для пагинации на больших страницах

`OFFSET 100000 LIMIT 20` → PG читает 100020 строк, отбрасывает 100000. Медленно.

Fix: **keyset pagination** (курсор):
```sql
WHERE created_at < :last_seen ORDER BY created_at DESC LIMIT 20
```

Двигается вперёд без OFFSET.

### 2.6 LIKE '%x%' (leading wildcard)

`WHERE name LIKE '%berik%'` — не может использовать B-Tree индекс.

Fixes:
- **Full-text search** (`tsvector` + GIN).
- **Trigram index** (`pg_trgm` extension).
- Elasticsearch для полного текстового поиска.

### 2.7 Deep JSON parsing ⭐

Парсинг больших JSON (мегабайты) — CPU-intensive.
- Jackson `readValue` на 10 MB = 200-500 мс.
- Многократно = CPU 100%.

Fixes:
- **Streaming API** — `JsonParser`, читать token за token.
- **Схема** — заранее знать структуру.
- Меньше JSON в API (правильные DTO с filters).

### 2.8 SSL/TLS handshake

10-50 мс на каждый новый connection. При частых новых соединениях (без пула) — большая доля latency.

Fix: connection pooling (JDBC pool, HTTP client keep-alive).

### 2.9 DNS resolution

Разрешение имени → 10-100 мс если DNS медленный / уходит наружу.

Fix:
- Локальный DNS caching (`/etc/nscd.conf`, `dnsmasq`).
- В JVM: `-Dnetworkaddress.cache.ttl=60`.
- Consul service discovery кэширует список инстансов.

### 2.10 Full GC ⭐

Stop-the-world пауза в JVM.
- G1: обычно 50-200 мс, изредка секунды.
- ZGC: <10 мс.

Мониторь `-Xlog:gc*`. Долгие Full GC = утечка / мало heap / плохой tuning.

### 2.11 Логирование в проде

`log.debug(...)` со сложной строкой — если DEBUG выключен, всё равно вычисляется:
```java
log.debug("Fno: " + heavyToString(fno));   // heavyToString вызовется всегда!
```

Правильно:
```java
log.debug("Fno: {}", fno);                 // toString только если DEBUG on
```

Или guard:
```java
if (log.isDebugEnabled()) {
    log.debug("Fno: " + heavyToString(fno));
}
```

### 2.12 Reflection

Reflection на **горячем пути** — 10-100× медленнее прямого вызова.

Fixes:
- Кэшировать `Method`/`Field`.
- `MethodHandles`.
- Кодогенерация (Lombok, MapStruct).

Для конфигурации / edge — приемлемо.

### 2.13 Открытие/закрытие ресурсов

- File open — sys call, ~10-100 мкс.
- Socket open — TCP handshake, +SSL если есть.

Пуливать.

### 2.14 Блокировки БД (SELECT FOR UPDATE)

- Тонкая гранула: одна строка — ок.
- Крупная гранула: `LOCK TABLE` — плохо, всё встало.
- Долгая блокировка → deadlock, timeout.

Правило: **держать локи как можно короче**. Обновление через `WHERE id=? AND version=?` (optimistic).

### 2.15 Cross-DC вызовы

150+ мс round-trip между Almaty и Amsterdam. Если делаешь 5 таких — уже 750 мс.

Fixes:
- Локальные реплики.
- Кэш.
- Batch.

### 2.16 Горячие мьютексы

`synchronized` на static field, много потоков → contention → все ждут.

Fixes:
- `ReentrantLock` с TryLock.
- Lock-free (Atomic, CAS).
- Sharding — разные потоки на разные locks.
- Immutable data.

### 2.17 Много дескрипторов файлов / сокетов

Каждый — kernel resource. Лимит `ulimit -n`. Утечка → «Too many open files».

Мониторить `lsof -p <pid> | wc -l`.

### 2.18 Encryption / decryption на CPU

BCrypt: 10-100 мс (специально, для brute-force защиты).
RSA sign / verify: 1-10 мс.
AES: миллисекунды на MB.

Fixes:
- Кэшировать результаты (не пере-шифровывать одно и то же).
- Hardware acceleration (AES-NI).

---

## 3. Как узнать что дорого

### 3.1 Profilers

- **JFR (Java Flight Recorder)** — встроенный, low-overhead. Bсегда включай в проде на семпле.
- **async-profiler** — sampling для CPU, alloc, lock. Flame graph.
- **VisualVM** — GUI для быстрой диагностики.
- **YourKit, JProfiler** — коммерческие, мощные.

### 3.2 APM (Application Performance Monitoring)

- **Datadog**, **New Relic**, **Elastic APM** — трассировка запросов через сервисы.
- Показывает latency каждого HTTP-запроса, SQL, external call.
- Distributed tracing (OpenTelemetry).

### 3.3 Метрики

- **Micrometer** + **Prometheus** + **Grafana**.
- Что мониторить:
  - HTTP: `http.server.requests` (rate, latency percentiles, errors).
  - JDBC: `hikaricp.*` (см. предыдущий файл).
  - JPA: `hibernate.*`.
  - JVM: heap, GC, threads.
  - Кастомные бизнес-метрики.

### 3.4 Логи с correlation ID

Каждому запросу — уникальный ID (traceId). Пропускать через все сервисы. Легко найти цепочку.

Spring Cloud Sleuth / Micrometer Tracing.

### 3.5 Load testing

- **JMeter**, **Gatling**, **k6**, **wrk** — стрельба нагрузкой.
- Ищи deltas: p50, p95, p99, p99.9 latency.
- p99 << p95 = fat tail (иногда очень плохо).

---

## 4. Стратегии оптимизации

### 4.1 Кэш

Правило: **самая быстрая операция — та, которую не сделали**.

Кэшируй:
- Redis / Memcached / Hazelcast — распределённый.
- Caffeine — in-memory JVM.
- HTTP-кэш (nginx / CDN).
- Кэш второго уровня Hibernate.

Правила:
- Знать TTL / когда инвалидировать.
- Stale-while-revalidate — отдавать старое, пока обновляется.
- Cache stampede (все сразу лезут за expired) → mutex / debounce.

### 4.2 Async / очереди

Не блокировать HTTP-запрос долгими операциями. Положить в Rabbit/Kafka, ответить сразу, обработать асинхронно.

### 4.3 Batch

Много одинаковых операций → одной пачкой.
- INSERT/UPDATE batch (JDBC).
- HTTP-запросы к batch API.
- Bulk Elastic index.

### 4.4 Sharding / Partitioning

Разделить данные по ключу → параллельная обработка.
- БД партиции.
- Kafka partitions.
- Shards в Elastic.

### 4.5 Скэйлинг

- **Vertical** — больше CPU/RAM для одного инстанса.
- **Horizontal** — больше инстансов.

Горизонтально масштабируется stateless + правильная балансировка.

### 4.6 Precompute / materialized views

Тяжёлые агрегаты — считать заранее (job раз в час), хранить.

Пример: dashboard с count по 20 категориям → материализованный view + refresh раз в 5 мин.

---

## 5. Highload principles

### 5.1 Fail fast

Не ждать 30 сек: если downstream упал → **circuit breaker** размыкается → fast fail → пользователь получает 503 сразу.

Resilience4j, Hystrix (устарел).

### 5.2 Bulkhead

Разделять пулы для разных зависимостей. Downstream A упал → его пул исчерпан, но пул B работает.

### 5.3 Timeouts everywhere

- HTTP connect + read timeout.
- DB connection timeout.
- DB socket timeout.
- Rabbit publish timeout.
- Redis command timeout.

Никогда без явного timeout (default может быть бесконечный).

### 5.4 Retry с exponential backoff

Не сразу retry (только усугубит перегрузку). Ждать: 1с, 2с, 4с, 8с.

### 5.5 Идемпотентность

При retry можешь повторить операцию → должна быть безопасна.

### 5.6 Rate limiting

Ограничение rate от одного клиента. Защита от abuse.

Token bucket, sliding window. Reddis + Lua скрипт.

### 5.7 Graceful degradation

Часть функционала упала → отдать что можешь.

Пример: страница блога. Комментарии упали → показать пост без комментариев (с текстом «комментарии временно недоступны»).

### 5.8 Observability first

Метрики + логи + trace = must. Без них — тыкаешь пальцем в небо.

---

## 6. Правила для JVM/Java highload

- **Не new-ить много объектов на горячем пути** — allocation pressure → GC.
- **Кэшировать** immutable-объекты, use of `String.intern`.
- **Reuse buffers** (`ByteBuffer`, `char[]`).
- **Streaming вместо full-load** (`InputStream` вместо `readAllBytes`).
- **Avoid autoboxing** — `List<Long>` boxes каждый `long`; для hot path — примитивные коллекции (Eclipse Collections, Koloboke).
- **Async I/O** — CompletableFuture, Reactor, Virtual Threads.
- **Prefer immutable** — потокобезопасно, GC-friendly.

---

## 7. Правила для БД highload

- **Правильные индексы** — must.
- **EXPLAIN ANALYZE** для медленных.
- **Небольшие транзакции**.
- **Никаких сетевых вызовов** в tx.
- **Keyset pagination** для больших наборов.
- **Batch insert/update**.
- **Read replicas** для тяжёлого чтения.
- **Партиционирование** для очень больших таблиц.
- **Материализованные views** для аналитики.
- **`pg_stat_statements`** для профилирования запросов.

---

## 8. Правила для микросервисов

- **Не синхронно** там где можно async.
- **Timeouts + retry + circuit breaker** на любом внешнем вызове.
- **Idempotency** для всех write-операций.
- **Кэшировать service discovery lookups**.
- **Не логировать чувствительное**.
- **Trace ID** через все сервисы.
- **Метрики per API** (rate, latency percentiles, errors).

---

## 9. Реальные ИСНА-кейсы

Из memory:
- `knp-filter-sent-documents-perf`: синхронный RestTemplate на АРМ → блокирует downstream + новый HTTPS-connect на запрос. **Fix**: connection pool + async.
- `knp-fno21-shedlock-stale-image-dup-regnum`: без ShedLock scheduled-job лупился параллельно на всех репликах → duplicate INSERT. **Fix**: distributed lock.
- `knp-fo-sync-notification-bugs`: @Transactional мёртв из-за self-invocation + `printStackTrace` вместо log.error → ELK-слепая зона. **Fix**: правильные транзакции + structured logging.
- `knp-e2e-runner-hikari-isolation-poisoning`: opt-in `isolation=-1` отравлял пул → gate краснел 18 мин. **Fix**: явный `transactionIsolation`.

---

## 10. Собесные вопросы

1. **Что дорого в БД?** — Seq scan (нет индекса), N+1, COUNT(*), OFFSET на больших, LIKE '%x%', долгие транзакции.
2. **Что дорого в JVM?** — Full GC, много аллокаций, reflection на горячем пути, blocking I/O в мало потоках.
3. **Почему нельзя внешний API внутри `@Transactional`?** — Держит connection БД и row locks → пул истощается / deadlock.
4. **Как избежать N+1?** — JOIN FETCH, @EntityGraph, @BatchSize, DTO projection.
5. **Что такое keyset pagination?** — Курсор по значению (WHERE created_at < ?), без OFFSET; для больших страниц.
6. **Как ускорить COUNT(*)?** — Кэш, approximate из pg_class, материализованный view, Slice вместо Page.
7. **Как ускорить старт (cold start)?** — Меньше auto-config, CDS, AppCDS, GraalVM Native.
8. **Что такое circuit breaker?** — Разомкнутая цепь при повторных отказах downstream → fast fail.
9. **Что такое bulkhead?** — Изоляция пулов ресурсов для разных зависимостей.
10. **Что такое graceful degradation?** — Часть функционала упала → отдаём что можем.
11. **Зачем connection pool?** — Reuse TCP + auth (30-100 мс saved на acquire).
12. **Что такое retry backoff?** — Ждать увеличивающееся время между повторами (1s, 2s, 4s, ...).
13. **Как найти узкое место в проде?** — Метрики (Prometheus/Grafana) + APM (Datadog/etc) + JFR profiles.
14. **Что дороже: HTTP или БД?** — Зависит: локальная БД (~1 мс), локальный HTTP (~1-10 мс), remote HTTP (10-500 мс).
15. **Как измерить перформанс метода?** — JFR / async-profiler flame graph; или Micrometer Timer.

---

## Итог

- **7 порядков** разница между CPU cache и HDD → всё что «наружу» дорого.
- **N+1, sync-в-tx, Seq scan, printStackTrace** — топ-4 убийцы прода.
- **Кэш, async, batch, sharding** — стандартные способы масштабирования.
- **Timeouts, retry, circuit breaker, bulkhead** — паттерны надёжности.
- **Метрики + APM + JFR** — must для диагностики.
- **Правила**: не блокировать поток на I/O надолго, не держать транзакцию, всегда явные timeouts.

Следующий — `31-load-balancer.md`.
