# 30. Дорогостоящие операции в highload проектах

## Иерархия дороговизны операций

Понимание относительной стоимости операций критически важно для проектирования highload систем. Разница между быстрыми и медленными операциями составляет несколько порядков магнитуды, что делает правильный выбор архитектуры решающим для производительности.

Ориентировочные времена типичных операций на современной серверной машине:

```
Регистр CPU                ~1 нс          ×1
L1 cache                   ~1 нс          ×1
L2 cache                   ~4 нс          ×4
L3 cache                   ~15 нс         ×15
RAM                        ~100 нс        ×100
NVMe SSD (random)          ~100 мкс       ×100 000
Network (local DC)         ~500 мкс       ×500 000
HDD (random)               ~10 мс         ×10 000 000
Network (cross-region)     ~150 мс        ×150 000 000
```

Разница между L1 cache и HDD составляет семь порядков магнитуды — десять миллионов раз. Это не просто abstract number — практическое последствие в том что операция кажущаяся быстрой в testing environment может стать bottleneck в production под нагрузкой.

Фундаментальное правило — чем ближе к CPU тем дешевле операция. Всё что идёт «наружу» — диск, сеть, database — принципиально дорого по сравнению с in-memory операциями. Architectural решения должны минимизировать количество таких «внешних» вызовов на critical path.

## Что реально дорого в highload

Ниже разберём наиболее значимые категории дорогостоящих операций встречающиеся в реальных приложениях.

Синхронный HTTP-вызов внутри транзакции это критическая ошибка часто встречающаяся в enterprise коде:
```java
@Transactional
void submit(Fno f) {
    repo.save(f);
    externalApi.call(f);   // ← до 30 секунд ждём внешний API
    audit(f);
}
```

Последствия катастрофические. Connection в pool занят все время внешнего вызова. Row locks в БД держатся всё это время. Deadlocks растут потому что locks дольше живут. Pool исчерпается быстро при нескольких таких запросах. Приложение возвращает 500 на новых запросах когда pool exhausted.

Правило абсолютное — никаких сетевых вызовов внутри @Transactional. Только database операции. Fix путём разбиения на короткую transaction plus внешний вызов вне неё:
```java
void submit(Fno f) {
    doSave(f);                // короткая транзакция
    externalApi.call(f);      // вне транзакции
}

@Transactional
void doSave(Fno f) { 
    repo.save(f); 
}
```

Или через outbox pattern — сохранить в БД запись «отправь этому API», отдельный job подхватит и вызовет вне transaction контекста. Дополнительно даёт guarantee что вызов произойдёт даже при сбое.

N+1 запросы это одна из наиболее частых performance проблем в ORM-based приложениях. Обсуждалось в файле 15. Каждый дополнительный SQL — round-trip к database — 1-5 миллисекунд. При 1000 запросов вместо одного JOIN легко получить 5 секунд дополнительной latency. Лечится через JOIN FETCH, EntityGraph, DTO projection, @BatchSize.

Отсутствие индекса (Seq Scan) убийственно для больших таблиц. Таблица с 10 миллионами строк без индекса — PostgreSQL читает все 10 миллионов при каждом query. С индексом читаются 10-100 строк. Разница в latency 100 000 раз. Диагностика через EXPLAIN ANALYZE ищет Seq Scan на больших таблицах.

SELECT COUNT(*) на большой таблице специфическая проблема PostgreSQL. Из-за MVCC невозможно просто взять число из header — PostgreSQL должен прочитать все строки чтобы определить какие видимы для текущей transaction. На таблице 10 миллионов строк может занять несколько секунд.

Решения включают кэширование count с периодическим обновлением. Approximate count из pg_class.reltuples — приблизительно достаточно для UI. Slice вместо Page в Spring Data — избегает COUNT query. Секционирование таблицы уменьшает scope count. Materialized view для аналитических dashboard.

OFFSET для pagination на больших страницах. OFFSET 100000 LIMIT 20 заставляет PostgreSQL прочитать 100020 строк из индекса и отбросить первые 100000. Extremely slow. Fix через keyset pagination:
```sql
WHERE created_at < :last_seen_at 
ORDER BY created_at DESC 
LIMIT 20
```

Курсор движется вперёд по значению без OFFSET. Полагается на индекс для быстрого seek.

LIKE с leading wildcard не может использовать B-Tree индекс:
```sql
WHERE name LIKE '%berik%'
```

B-Tree индекс работает только для prefix search. Full-text search через tsvector plus GIN индекс правильный подход для полнотекстового поиска. Trigram index через pg_trgm extension для fuzzy matching. Elasticsearch для complex text search сценариев.

Deep JSON parsing на CPU-intensive для больших payloads. Jackson readValue на 10 MB JSON занимает 200-500 миллисекунд CPU. Многократные parse операции нагружают CPU до 100 процентов. Решения включают streaming API через JsonParser чтение token за token без full materialization. Явная схема (POJO) заранее известная — быстрее generic parsing. Меньше JSON в API через правильные DTO с filter'ами полей.

SSL/TLS handshake добавляет 10-50 миллисекунд к каждому новому connection. Без connection pool это становится значительной частью latency. HTTP client keep-alive и JDBC connection pool это ключевые mitigation.

DNS resolution может быть surprisingly slow. Медленный DNS server или сеть вне DC добавляет 10-100 миллисекунд. Local DNS caching через nscd или dnsmasq решает. В JVM параметр -Dnetworkaddress.cache.ttl=60 контролирует DNS cache. Consul service discovery кэширует список инстансов уменьшая DNS lookups.

Full GC в JVM это stop-the-world pause. G1 обычно 50-200 миллисекунд, редко секунды. ZGC меньше 10 миллисекунд что делает его предпочтительным для latency-sensitive приложений. Мониторинг через -Xlog:gc*. Долгие GC часто indicate memory leak, insufficient heap, или бедный tuning.

Логирование в production может неожиданно стоить дорого:
```java
log.debug("Fno: " + heavyToString(fno));   // heavyToString вызовется ВСЕГДА
```

Even если DEBUG level disabled, конкатенация строки происходит перед вызовом log метода. heavyToString производит CPU work бесполезно. Правильный подход через parameterized logging:
```java
log.debug("Fno: {}", fno);   // toString только если DEBUG enabled
```

SLF4J deferred evaluation — toString вызывается лениво только при actual logging. Или explicit guard через isDebugEnabled если требуется сложная preparation:
```java
if (log.isDebugEnabled()) {
    log.debug("Fno: " + heavyToString(fno));
}
```

Reflection на горячем пути в 10-100 раз медленнее прямого вызова метода. Fix через cache Method и Field объектов, использование MethodHandles для faster invocation, кодогенерация через Lombok, MapStruct вместо runtime reflection. Для конфигурации и edge use cases reflection acceptable, но не на critical path.

Открытие и закрытие ресурсов имеет overhead. File open это system call 10-100 микросекунд. Socket open плюс TCP handshake plus SSL если применимо. Всё это должно быть pooled.

Блокировки БД через SELECT FOR UPDATE. Fine-grained locks на одной строке ok. Coarse-grained locks like LOCK TABLE catastrophic — вся таблица блокируется. Долгие locks приводят к timeouts и deadlocks. Правило держать locks максимально коротко. Optimistic locking через version columns часто предпочтительнее — WHERE id=? AND version=?.

Cross-DC вызовы 150+ миллисекунд round-trip между географически distant DCs. Пять таких вызовов equals 750 миллисекунд только на network. Решения через local replicas, кэш, batching, редизайн для локальности данных.

Горячие мьютексы создают contention. synchronized на static field с многими threads означает все ждут одного lock. Решения через ReentrantLock с tryLock для non-blocking attempts, lock-free structures через Atomic и CAS, sharding на разные locks по ключу, immutable data не требующая synchronization.

Много файловых дескрипторов и sockets. Каждый это kernel resource с limit через ulimit -n. Утечка приводит к Too many open files errors. Мониторинг через lsof -p pid.

Encryption/decryption операции на CPU. BCrypt intentionally slow 10-100 миллисекунд для brute force защиты. RSA sign/verify 1-10 миллисекунд. AES миллисекунды на MB. Решения через кэширование результатов (не re-encrypt то же самое), hardware acceleration через AES-NI где доступно.

## Как узнать что дорого

Диагностика performance требует правильных инструментов. Guessing без данных обычно приводит к optimization wrong вещей.

Profilers для JVM. Java Flight Recorder (JFR) встроенный low-overhead. Всегда включать в production на sampling basis. async-profiler даёт sampling для CPU, allocations, locks с flame graph visualization. VisualVM для быстрой GUI диагностики. YourKit и JProfiler коммерческие с богатыми возможностями.

Application Performance Monitoring системы — Datadog, New Relic, Elastic APM. Показывают distributed tracing через все сервисы. Latency каждого HTTP запроса, SQL query, external call. Automatic detection anomalies. OpenTelemetry как open standard для распределённой трассировки.

Метрики через Micrometer plus Prometheus plus Grafana это стандартный observability stack. Что мониторить. HTTP метрики через http.server.requests — rate, latency percentiles, error rates. JDBC через hikaricp.* — pool usage, wait times. JPA через hibernate.* — queries per session, cache hits. JVM — heap usage, GC pauses, thread counts. Custom бизнес метрики специфичные для application.

Логи с correlation ID для распределённой трассировки. Каждому входящему запросу — уникальный trace ID пропускаемый через все downstream calls. Легко найти всю chain обработки одного запроса. Spring Cloud Sleuth или Micrometer Tracing.

Load testing критически важен для highload систем. Инструменты — JMeter, Gatling, k6, wrk. Метрики важные — p50, p95, p99, p99.9 latency. p99 значительно больше p95 указывает на «fat tail» — occasional slow requests которые могут быть unacceptable для UX.

## Стратегии оптимизации

Кэш это одна из самых мощных техник — самая быстрая операция это та которую не сделали. Уровни кэша. Redis, Memcached, Hazelcast для distributed cache. Caffeine для in-memory JVM cache. HTTP cache через nginx или CDN для static content. Hibernate second-level cache для JPA entities.

Правила эффективного кэширования. Знать TTL или условия invalidation — устаревший cache хуже отсутствия cache. Stale-while-revalidate pattern — отдавать старое пока обновление на подходе. Cache stampede protection — когда expiration приводит к thundering herd (все clients simultaneously miss и лезут за данными), решается через mutex или debounce на regeneration.

Async и очереди для non-blocking обработки. Не блокировать HTTP запрос долгими операциями. Положить сообщение в Rabbit или Kafka, ответить клиенту сразу, обработать в background. Улучшает UX (быстрый response) и throughput (нет blocking).

Batch операции для reducing round-trips. Много одинаковых операций объединяются в одну. INSERT/UPDATE batch через JDBC. HTTP requests к batch API endpoint. Bulk Elasticsearch indexing. Общий принцип — если хотите быструю систему делайте меньше operations.

Sharding и partitioning разделение данных по ключу. Database partitions для больших таблиц. Kafka partitions для parallel consumption. Shards в Elasticsearch. Parallel обработка ускоряет операции пропорционально количеству shards.

Скэйлинг vertical или horizontal. Vertical scaling увеличение CPU/RAM одного instance — простая но ограниченная максимальным hardware. Horizontal scaling больше instances — theoretically unlimited если приложение stateless. Правильно спроектированные микросервисы легко scale horizontally.

Precompute и materialized views. Тяжёлые аггрегаты рассчитываются заранее раз в час или раз в 5 минут вместо каждого запроса. Пример dashboard с counts по 20 категориям — materialized view refresh раз в 5 минут вместо 20 SELECT COUNT на каждый view page.

## Highload принципы

Fail fast принцип критичен для системной стабильности. Не ждать 30 секунд если downstream упал — circuit breaker размыкается быстро после нескольких failures, дальнейшие вызовы возвращают immediate error без attempt соединения. Пользователь получает 503 быстро вместо hanging endpoint. Resilience4j стандартный инструмент, Hystrix устарел но были подобные концепции.

Bulkhead pattern разделяет пулы ресурсов для разных зависимостей. Downstream A упал — его pool exhausted, но pool B продолжает работать. Failure одной dependency не каскадирует на всю систему. Реализуется через отдельные thread pools, connection pools per dependency.

Timeouts на всех уровнях. HTTP connect и read timeout. Database connection и socket timeout. RabbitMQ publish timeout. Redis command timeout. Kafka consumer poll timeout. Никогда не полагаться на defaults — часто бесконечные что приводит к hanging при проблемах.

Retry с exponential backoff. Не сразу повторять после failure — только усугубит перегрузку downstream. Ждать увеличивающееся время между попытками — 1 секунду, 2 секунды, 4 секунды, 8 секунд. Даёт time для recovery downstream.

Идемпотентность при retry обязательна. Если retry может повторить операцию, она должна быть безопасна при повторе. Все write operations должны быть идемпотентны через unique keys или conditional updates.

Rate limiting защищает от abuse и перегрузки. Ограничение rate от одного клиента или IP. Token bucket или sliding window algorithms. Обычно реализуется через Redis with atomic scripts.

Graceful degradation — часть функционала недоступна, отдаём что можем. Страница блога — comments сервис упал, показываем пост без comments с «comments temporarily unavailable» notice. Better than showing full error page.

Observability first — метрики, логи, distributed traces обязательны. Без них troubleshooting production issues становится guessing. Investment в observability окупается многократно при первом же сложном bugе.

## Правила для JVM/Java highload

Минимизировать object allocations на горячем пути. Больше allocations означает больше GC pressure, потенциально дольше pauses. Reuse buffers where possible.

Кэшировать immutable objects. String.intern для repeatedly used strings. Autoboxed Integer values -128 до 127 автоматически cached JVM. Reuse ThreadLocal buffers.

Streaming вместо full-load для больших data. InputStream reading в chunks вместо readAllBytes которое materializes весь file в memory. Одинаково для network data, database results.

Avoid autoboxing на hot path. List<Long> boxes каждый long значение — creates garbage. Для performance-critical кода primitive collections через Eclipse Collections, Koloboke, Trove.

Async I/O для reducing thread blocking. CompletableFuture, Reactor, Virtual Threads в Java 21+. Позволяет много concurrent operations на limited количестве threads.

Prefer immutable objects. Thread-safe без synchronization. GC-friendly в некоторых implementations. Проще reason about.

## Правила для БД highload

Правильные индексы обязательны. Каждый query на большой таблице должен использовать индекс. EXPLAIN ANALYZE для medium/slow queries — regular exercise not just when problems arise.

Небольшие транзакции. Никогда external network calls внутри @Transactional. Держать transactions максимально короткими для minimize lock contention.

Keyset pagination для больших наборов данных вместо OFFSET-based. Batch INSERT/UPDATE вместо individual queries. Read replicas для тяжёлого read traffic — уменьшает нагрузку на master.

Партиционирование для очень больших таблиц. Материализованные views для аналитики. pg_stat_statements обязательный для profiling queries в production.

## Правила для микросервисов

Не синхронно там где можно async. Long-running operations через message queues.

Timeouts, retry, circuit breaker обязательны на любом external вызове. Idempotency для всех write operations. Cache service discovery lookups — не resolving на каждый call.

Не логировать sensitive data — tokens, passwords, PII. Trace IDs через все services для correlated logs.

Метрики per API — rate, latency percentiles, error rates. Alerting на нарушение SLO.

## Реальные кейсы КНП

Memory knp-filter-sent-documents-perf — синхронный RestTemplate на АРМ создавал новый HTTPS connection на каждый запрос. Блокирует downstream, expensive SSL handshake каждый раз. Fix через connection pool в HTTP client plus async обработка где applicable.

Memory knp-fno21-shedlock-stale-image-dup-regnum — без ShedLock scheduled job лупился параллельно на всех репликах. Приводил к duplicate INSERT операциям с одинаковыми регистрационными номерами. Fix через ShedLock как distributed coordination — только одна replica выполняет job at a time.

Memory knp-fo-sync-notification-bugs — @Transactional dead из-за self-invocation plus printStackTrace вместо log.error. Ошибки не попадали в ELK создавая слепую зону мониторинга. Fix через правильные transactions без self-invocation plus structured logging через SLF4J.

Memory knp-e2e-runner-hikari-isolation-poisoning — opt-in isolation равное -1 отравлял shared PgBouncer pool. gate-knp краснел 18 минут. Fix через explicit transactionIsolation равное TRANSACTION_READ_COMMITTED.

## Ключевые метрики

Что должно быть мониторено в production. Response time percentiles — p50, p95, p99 для user-facing endpoints. Throughput — requests per second на разные endpoints. Error rate — процент 4xx и 5xx responses. Saturation — CPU, memory, disk, network utilization.

Специфичные для JVM метрики. Heap usage — total и per generation. GC frequency и pause times. Thread count и state distribution. Class loading counts.

Database специфичные метрики. Connection pool usage через hikaricp.connections.*. Query execution times через APM или pg_stat_statements. Lock waits и deadlocks. Replication lag для replicas.

Message broker метрики. Queue depth per queue. Publish и consume rates. Redelivery rates. Consumer lag для Kafka.

Business метрики. Domain-specific counts — например количество отправленных ФНО в час, количество новых пользователей, количество failed authentications. Показатели health бизнеса не только technical health.

## Alerting стратегии

Alert должен быть actionable. Каждый alert должен требовать human action или он должен быть suppressed. Alerts которые regularly ignore приводят к alert fatigue где critical alerts miss.

Тиеринг alerts. Critical — page кого-то немедленно, ночью, в любое время. Warning — notify team в working hours для investigation. Info — dashboard indicators без active notification.

Пороги должны быть tunated к baseline. Static thresholds типа CPU больше 80% часто false positive при нормальных spikes. Alerting on trend changes или SLO violations более meaningful.

Symptoms alerts не causes. Alert на «response time p99 больше 500ms» полезнее чем «CPU больше 80%». User-facing symptoms directly relevant. Root cause определяется after alert firing через investigation.

Silencing и dampening. При known maintenance или incidents дважды не alert. Grouping связанных alerts чтобы не флудить channel.

## Load testing patterns

Test что реально критично. Не все endpoints одинаково важны. Load test самые user-facing и critical paths.

Realistic traffic pattern. Не constant rate — реальный traffic имеет peaks и troughs. Simulate diurnal patterns, weekend variations, seasonal spikes.

Gradual ramp up. Start низкая нагрузка, increase gradually. Позволяет observe degradation points before catastrophic failure. Suddenly hitting peak load часто отличается по characteristics.

Chaos engineering complement to load testing. Что случается когда downstream упал во время нагрузки. Что если 50% Kafka partitions unavailable. Что если replica lag increases. Real failures happen — тест readiness системы к ним.

Regular exercise не one-off. System changes over time — new features, dependencies, data growth. Regular load testing выявляет regressions before они становятся production issues.

## Собеседные вопросы часто задают

Что дорого в БД — Seq Scan без индекса, N+1, COUNT star на большой таблице, OFFSET на больших pages, LIKE с leading wildcard, долгие transactions.

Что дорого в JVM — Full GC, много allocations на hot path, reflection на hot path, blocking I/O в много threads.

Почему нельзя внешний API в @Transactional — держит connection pool и row locks, pool исчерпается, deadlocks возможны, всё встанет при downstream slow.

Как избежать N+1 — JOIN FETCH, @EntityGraph, @BatchSize, DTO projection through Spring Data query methods.

Что такое keyset pagination — cursor-based pagination через WHERE created_at < last_seen вместо OFFSET. Для больших pages где OFFSET slow.

Как ускорить COUNT star — кэш, approximate через pg_class, materialized view, Slice вместо Page, партиционирование.

Как ускорить cold start — меньше auto-configuration, CDS/AppCDS, GraalVM Native Image для extreme cases.

Что такое circuit breaker — разомкнутая цепь при повторных failures downstream, fast fail без attempt соединения.

Что такое bulkhead — изоляция resource pools для разных зависимостей, failure одной не каскадирует.

Что такое graceful degradation — часть функционала недоступна, отдаём остальное с acknowledgment.

Зачем connection pool — reuse TCP plus authentication, saves 30-100ms на each connection acquire.

Что такое retry backoff — waiting increasing time между attempts, 1s, 2s, 4s, exponentially.

Как найти узкое место — metrics через Prometheus/Grafana, APM через Datadog/etc, JFR profiles для detailed analysis.

Что дороже HTTP или БД — depends. Local database ~1ms, local HTTP ~1-10ms, remote HTTP 10-500ms. Network location matters greatly.

Как измерить performance метода — JFR или async-profiler flame graph. Или Micrometer Timer для explicit measurements.

## Итоги

Семь порядков магнитуды разница между CPU cache и HDD. Всё что идёт «наружу» — диск, сеть, database — фундаментально дорого. Architecture должна минимизировать external calls на critical path.

Топ 4 убийцы production. N+1 queries — 1000 SQL вместо одного JOIN. Sync HTTP в transaction — pool exhaustion. Seq Scan на большой таблице — missing index. printStackTrace вместо structured logging — blind spot в мониторинге.

Стандартные способы масштабирования. Кэш для reducing operations. Async для non-blocking. Batch для reducing round-trips. Sharding для parallelism.

Паттерны надёжности. Timeouts везде. Retry с backoff. Circuit breaker для fast fail. Bulkhead для изоляции. Graceful degradation для partial availability.

Observability обязательна. Metrics plus APM plus JFR profiles plus distributed tracing. Без них troubleshooting production становится guessing что unacceptable.

Правила для JVM. Меньше allocations на hot path. Streaming вместо full-load. Async I/O через CompletableFuture или Virtual Threads. Prefer immutable objects.

Правила для БД. Правильные индексы. Небольшие transactions. Keyset pagination. Batch operations. Read replicas для scaling reads. pg_stat_statements включён.

Правила для микросервисов. Async where possible. Timeouts, retry, circuit breaker on external calls. Idempotency for writes. Cached service discovery. Не логировать sensitive. Trace IDs everywhere. Metrics per API.

Реальные кейсы КНП подтверждают что теоретически известные проблемы регулярно встречаются в production. Каждый должен быть explicitly avoided через discipline и code review.

Дальше — load balancer как ключевая инфраструктурная компонента для scaling микросервисов.
