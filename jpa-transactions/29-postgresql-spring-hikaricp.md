# 29. PostgreSQL плюс Spring Boot: HikariCP, connection pool, timeouts

## JDBC как основа

JDBC (Java Database Connectivity) это стандартный API для работы с реляционными базами данных из Java. Определяет abstract interfaces которые реализуются database-specific драйверами.

Основные объекты JDBC. DataSource это фабрика соединений — logical представление database, инкапсулирующее URL и credentials. Connection представляет одно TCP соединение с базой плюс database session state. Statement и PreparedStatement для выполнения SQL — PreparedStatement предпочтителен для параметризованных queries. ResultSet это курсор для итерации по результатам SELECT.

Типичный код работы с JDBC:
```java
DataSource ds = getDataSource();
try (Connection conn = ds.getConnection()) {
    try (PreparedStatement ps = conn.prepareStatement(
            "SELECT * FROM fno WHERE id = ?")) {
        ps.setLong(1, 123);
        try (ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                // обработка каждой строки
            }
        }
    }
}
```

try-with-resources гарантирует правильное освобождение ресурсов — Connection возвращается в pool, PreparedStatement и ResultSet closes. Пропуск освобождения ведёт к leak connections что deprecates pool eventually.

PostgreSQL JDBC driver в дистрибутиве org.postgresql:postgresql отвечает за низкоуровневое взаимодействие с PostgreSQL. TCP соединение на порт 5432 с сервером. Реализация wire-протокола PostgreSQL для communication с backend. Обработка prepared statements — client-side или server-side. Type conversion между Java и PostgreSQL типами (LocalDateTime к timestamp, boolean к bit, UUID к uuid). Handling SSL negotiation если настроено.

## Зачем connection pool

Без connection pool каждый database вызов включает создание нового TCP соединения. Overhead этого создания значителен и суммируется в latency для высоконагруженных приложений.

Компоненты overhead. TCP handshake занимает несколько миллисекунд (SYN, SYN-ACK, ACK). SSL handshake если используется добавляет 10-50 миллисекунд для key exchange и certificate verification. PostgreSQL authentication включая password check, backend process creation, session initialization занимает 20-50 миллисекунд. Итого 30-100 миллисекунд на каждое новое соединение.

При 1000 requests в секунду создание нового connection на каждый запрос означает 1000 handshakes в секунду что фактически невозможно. Плюс каждый backend process в PostgreSQL занимает 5-10 MB памяти — 1000 concurrent connections потребует 5-10 GB просто на process overhead. PostgreSQL max_connections по default 100 — быстро исчерпывается.

Connection pool решает эти проблемы держа предопределённое количество открытых соединений готовых к использованию:
```
Приложение
    │
    │  getConnection()
    ▼
┌─────────── Pool ────────────┐
│  [idle]   [idle]            │  ← готовые соединения
│  [in-use] [idle]   [idle]   │
└──────────┬──────────────────┘
           │
           ▼
      PostgreSQL
```

Приложение просит соединение из pool через getConnection. Pool возвращает existing idle connection мгновенно. Приложение использует connection для queries. При close (или конце try-with-resources) connection возвращается в pool, не закрывается физически.

Плюсы поразительные. Соединение переиспользуется — нет дорогого init на каждый запрос. Latency getConnection падает с 30-100ms до subMillisecond. Количество PostgreSQL connections ограничено — pool size обычно 10-30, а не сотни. Ресурсы database и приложения используются эффективно.

## HikariCP стандарт

HikariCP это самый быстрый JDBC connection pool. Default в Spring Boot начиная с версии 2.0 как рекомендуемый выбор.

Характеристики. Легковесный — минимум features, максимум performance. Минимум блокировок внутри — использует lock-free structures где возможно. Хорошие defaults — работает out of box для большинства сценариев. Богатые metrics — интеграция с Micrometer для observability.

Альтернативы включают Tomcat JDBC Pool и Apache DBCP2 но оба практически устарели. HikariCP превосходит их по производительности и функциональности. Нет причин использовать альтернативы кроме исторических.

## Ключевые настройки

Пример полной конфигурации HikariCP:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://db-knp:5432/knp
    username: knp
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    
    hikari:
      pool-name: knp-hikari
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 30000
      max-lifetime: 1800000
      keepalive-time: 300000
      validation-timeout: 5000
      leak-detection-threshold: 60000
      auto-commit: false
      transaction-isolation: TRANSACTION_READ_COMMITTED
      data-source-properties:
        reWriteBatchedInserts: true
        prepareThreshold: 5
        cachePrepStmts: true
        useServerPrepStmts: true
```

Разберём каждую настройку и её impact.

maximum-pool-size ограничивает количество соединений в pool. Правило sizing pool_size = ((CPU_cores × 2) + effective_spindle_count) для database сервера. Для приложения обычно 10-30. Больше не значит быстрее — при 20-30+ contention внутри PostgreSQL начинает деградировать throughput вместо роста. Ставить 100+ обычно ошибка.

Практический пример проблемы. PostgreSQL с max_connections равным 200. 10 подов приложения каждый с pool 30 равно 300 possible connections. При peak PostgreSQL начинает отказывать в новых соединениях с ошибкой too many connections. Решение либо уменьшить pool per pod, либо PgBouncer для мультиплексирования.

minimum-idle это сколько соединений всегда держать открытыми даже при отсутствии нагрузки. Default равно maximum. Установка равного maximum даёт стабильную latency без старта соединения при первом запросе после idle периода. Меньшее значение экономит ресурсы но первые запросы после idle могут быть медленнее.

connection-timeout критически важный параметр. Задаёт максимум времени ожидания свободного connection из pool. Если pool полный и все in-use, getConnection блокируется до появления free connection или до timeout. По истечении timeout бросается SQLTransientConnectionException с сообщением Connection is not available.

Default 30 секунд разумен для большинства сценариев. Слишком большое значение означает что приложение висит долгое время при exhausted pool. Слишком маленькое создаёт false alarms при коротких пиках нагрузки. 30 секунд компромисс — достаточно чтобы переждать transient spike но не так долго что приложение становится неотзывчивым.

idle-timeout это через сколько миллисекунд простаивающие соединения (сверх minimum-idle) закрываются. По default 10 минут. Обычно 5-10 минут работает хорошо. Помогает освобождать resources когда нагрузка снижается.

max-lifetime это максимальный возраст соединения. По достижении соединение закрывается и создаётся новое. Default 30 минут. Важно для нескольких целей. Избежание stale connections из-за network issues или firewall timeouts. Handling ситуации когда PostgreSQL перезапускается на replica — соединения к старому pod eventually обновляются. Distribution нагрузки при database scale up.

Правило max-lifetime меньше чем PostgreSQL idle_in_transaction_session_timeout минус несколько секунд. Также меньше firewall TCP idle timeout если применимо. Обычно 20-30 минут.

keepalive-time это Hikari 4.0+ возможность. Периодически проверяет idle connections через SELECT 1 для поддержания TCP alive. Полезно когда firewall закрывает idle connections. Default disabled (0). Установка 5 минут (300000) полезна если между app и database есть firewall который может убить idle TCP.

validation-timeout это максимум на выполнение validation query (SELECT 1). Default 5 секунд. При slow database значения могут быть увеличены.

leak-detection-threshold это критически важный параметр для troubleshooting. Если connection не возвращён в pool за N миллисекунд, HikariCP логирует warning со stack trace откуда connection был взят. Помогает находить утечки — методы забывающие close connection или обрабатывающие exceptions некорректно.

Default 0 (disabled). Обязательно включай в production, обычно 60000 (60 секунд). Логи выглядят так:
```
HikariCP - Connection leak detection triggered for ProxyConnection@...
    on thread http-nio-8080-exec-42, stack trace follows
    at ...FnoService.badMethod(FnoService.java:42)
```

Полезно даже когда нет known проблем — periodic warnings указывают на code paths с потенциальными issues.

auto-commit определяет default transaction behavior. true (default) означает каждый statement как отдельная transaction — implicit commit после каждого. false означает необходимость explicit commit или rollback. Spring @Transactional управляет вручную поэтому значение не критично при использовании Spring transactions.

transaction-isolation задаёт default isolation level для соединений. PostgreSQL обычно требует TRANSACTION_READ_COMMITTED. Реальный кейс из КНП memory knp-e2e-runner-hikari-isolation-poisoning — раннер использовал isolation равное -1 (opt-in), пул отравлялся при shared через PgBouncer. Всегда явно задавать явное значение чтобы избежать surprises.

## Статусы соединений в pool

Каждое соединение в one из состояний. idle означает свободно, готово к использованию. active или in-use — выдано приложению, обрабатывает queries. awaiting означает приложение ждёт соединение из full pool. stale или evicted — превысило max-lifetime или idle-timeout, закрывается.

HikariCP экспортирует metrics для каждого состояния через Micrometer. Стандартные метрики:
```
hikaricp.connections.active         текущее in-use
hikaricp.connections.idle           текущее idle
hikaricp.connections.pending        сколько threads ждёт connection
hikaricp.connections.timeout        сколько раз таймаут произошёл
hikaricp.connections.usage          histogram времени использования
hikaricp.connections.acquire        histogram latency getConnection
hikaricp.connections.creation       histogram времени создания нового
```

Метрики экспортируются через Actuator plus Micrometer в Prometheus что позволяет monitoring в Grafana. Обязательный компонент production observability.

Правила мониторинга. Pending regularly больше 0 указывает на недостаточно большой pool — не хватает соединений для peak нагрузки. Timeout больше 0 критическая ситуация — либо утечка соединений либо долгие queries держат pool exhausted. Active приближающийся к maximum означает перегрузку — pool близок к исчерпанию.

## Connection timeout, socket timeout, IO wait

Различные типы timeout часто путают, важно понимать различия.

Connection timeout уже разобран — сколько приложение ждёт соединения из pool. Свойство pool level.

Socket connect timeout это сколько ждать установления TCP соединения с database. Свойство driver level:
```yaml
spring.datasource.hikari.data-source-properties:
  socketTimeout: 30           # secondsмаксимум на выполнение statement
  connectTimeout: 10          # секунды для установления TCP
```

Socket timeout это максимум на выполнение statement. Если query занимает более 30 секунд driver прерывает соединение. Обязательно устанавливать иначе runaway query может повесить весь pool. Разумное значение зависит от expected query complexity — 30-60 seconds для transactional, длиннее для аналитических.

IO wait это когда процессор waits for I/O операцию — disk, сеть. Для database запроса типичное. Приложение отправляет SQL по сети (1 мс). PostgreSQL работает над запросом (5-500 мс). Возвращает результат по сети (1 мс). Всё это время Java thread заблокирован ожидая I/O. CPU может быть свободен для других threads.

Мониторинг IO wait. Метрика hikari.connections.acquire.time показывает время получения connection из pool. jdbc.query.time (при использовании APM) показывает время самих queries. В OS через top команду процент wa показывает CPU waiting for I/O. В PostgreSQL pg_stat_activity.wait_event показывает что каждый backend ждёт.

Высокий IO wait в JVM означает много блокирующих I/O операций. Возможные решения. Batch операции для уменьшения количества round-trips. Reactive или Virtual Threads чтобы не блокировать thread на I/O. Cache для избежания I/O. Индексы для ускорения queries.

## Pageable в Spring Data

Pageable это Spring абстракция для pagination. Не про pool напрямую но связан с database queries.

Использование:
```java
Page<Fno> page = repo.findAll(
    PageRequest.of(0, 20, Sort.by("createdAt").descending()));
```

Под капотом Spring генерирует два запроса. SELECT ... LIMIT 20 OFFSET 0 возвращает страницу данных. SELECT COUNT(*) считает total rows.

Page объект содержит content (список), totalElements (общее количество), totalPages (расчётное количество страниц), hasNext, hasPrevious. Полезен для UI показывающих pagination controls.

Slice это alternative без total count — только hasNext. Быстрее чем Page потому что не выполняет COUNT query. Использовать когда total не нужен, только «есть ли следующая страница».

Caveat OFFSET на больших страницах. OFFSET 100000 LIMIT 20 заставляет PostgreSQL прочитать 100020 rows и отбросить первые 100000. Extremely slow на глубоких страницах. Также COUNT(*) на большой таблице сам по себе expensive.

Решение — keyset pagination или курсор пагинация:
```sql
WHERE created_at < :last_seen_at 
ORDER BY created_at DESC 
LIMIT 20
```

Клиент передаёт последний seen timestamp вместо номера страницы. PostgreSQL использует индекс для быстрого seek к правильной позиции без чтения skipped rows. Значительно быстрее OFFSET для глубоких страниц. Ограничение — можно двигаться только forward/backward последовательно, нет random access к произвольной странице.

## PgBouncer

PgBouncer это connection pooler на уровне сети между приложением и PostgreSQL. Работает как proxy принимающий соединения от клиентов и мультиплексирующий их на small pool real PostgreSQL connections.

Зачем нужен. Приложение может держать сотни connections к PgBouncer (дешёвые с его стороны). PgBouncer держит только desktop 20 к настоящему PostgreSQL. Мультиплексирует запросы — когда клиент simult idle между queries, real PostgreSQL connection может быть использован для другого клиента.

Три режима работы. Session mode — одно клиентское соединение биндится к одному PostgreSQL соединению на весь session. Работает как прямой PostgreSQL с точки зрения клиента, включая prepared statements и session state. Не даёт экономии connections — 1:1 mapping.

Transaction mode это рекомендуемый режим для мультиплексирования. Клиентское соединение биндится к PostgreSQL connection только на время транзакции, потом возвращается в pool. Один PostgreSQL connection может обслуживать много клиентов sequentially. Экономия огромна — 100 клиентов могут работать через 10 PostgreSQL connections. Ограничение — prepared statements не сохраняются между транзакциями потому что session меняется. SET session settings теряются между transactions.

Statement mode — соединение возвращается после каждого statement. Не поддерживает transactions клиента. Только для read-only аналитики.

В КНП обычно transaction mode для максимальной экономии. Memory кейс knp-fs-consul-deregister-after-db-flap упоминает db-knp это PgBouncer в pod сети, доступный только внутри podа не с хоста.

Требования JDBC при использовании PgBouncer transaction mode. Отключить prepared statements caching в клиенте:
```yaml
data-source-properties:
  prepareThreshold: 0
  preparedStatementCacheQueries: 0
```

Не использовать session-level SET — теряются между transactions. Не полагаться на session state как temp tables — не сохраняются.

## SSL

По default JDBC connections к PostgreSQL не encrypted. Для production обычно включают SSL особенно для connections через untrusted networks:
```yaml
spring.datasource.url: jdbc:postgresql://db:5432/knp?sslmode=require
```

Режимы sslmode. disable совсем без SSL. allow предпочитает без но принимает если сервер требует. prefer предпочитает с SSL, fallback на без. require только с SSL, отклоняет без. verify-ca плюс require добавляет проверку certificate authority. verify-full плюс verify-ca добавляет проверку hostname в сертификате.

Production обычно require минимум, verify-full для максимальной безопасности. verify-full требует правильно настроенных certificates соответствующих hostnames что может быть сложнее в dynamic environments.

SSL handshake добавляет 10-50 миллисекунд к open connection. С pool этот cost платится только при первом установлении, потом соединения переиспользуются — impact minimal.

## Timezone

Классическая проблема JDBC плюс PostgreSQL. Разные timezones на JVM, database, PostgreSQL server могут привести к unexpected timestamp values.

Стандартная рекомендация — всё в UTC внутри системы. Отображение в local timezone только на UI уровне через explicit conversion. Настройка:
```yaml
spring.jpa.properties.hibernate:
  jdbc.time_zone: UTC
```

Плюс JVM argument -Duser.timezone=UTC для установки JVM default timezone.

Без правильной настройки LocalDateTime интерпретируется в JVM default timezone что может отличаться между podами (например если один в UTC, другой в local timezone). Приводит к inconsistent storage дат и потенциальным off-by-timezone багам в бизнес-логике.

Правило always timestamptz в PostgreSQL columns вместо timestamp. timestamptz сохраняет explicit timezone information избегая ambiguity. timestamp без timezone это naive datetime потенциально проблемный при conversion.

## Prepared statements

PreparedStatement основа безопасного и эффективного query execution:
```java
PreparedStatement ps = conn.prepareStatement("SELECT * FROM fno WHERE reg_num = ?");
ps.setString(1, "12345");
ResultSet rs = ps.executeQuery();
```

Vs regular Statement:
```java
Statement st = conn.createStatement();
ResultSet rs = st.executeQuery("SELECT * FROM fno WHERE reg_num = '12345'");
```

Плюсы PreparedStatement. SQL injection невозможен — параметры передаются отдельно и правильно escaped драйвером. План запроса кэшируется на сервере — повторные execute быстрее потому что PostgreSQL не парсит и планирует каждый раз. Batch operations возможны через addBatch и executeBatch.

Различие client-side vs server-side prepared statements. Client-side — JDBC формирует финальный SQL с подставленными значениями и шлёт как regular statement. Работает всегда, но не даёт server-side plan caching. Server-side — JDBC шлёт explicit PREPARE и EXECUTE команды, PostgreSQL хранит план. Дают maximum performance для повторяющихся queries.

Настройка через свойства:
```yaml
data-source-properties:
  prepareThreshold: 5              # начать server-side prepare после 5-го использования
  preparedStatementCacheQueries: 256
  preparedStatementCacheSizeMiB: 5
```

prepareThreshold контролирует когда переключаться на server-side. 5 означает первые 5 executions client-side, потом server-side. 0 отключает server-side полностью.

Caveat PgBouncer transaction mode. Server prepared statements хранятся в session PostgreSQL. При использовании PgBouncer transaction mode session между transactions меняется. Prepared statement подготовленное в одной transaction невидимо в следующей. Ошибки при execute. Отключить server-side prepare через prepareThreshold равное 0 при использовании PgBouncer в transaction mode.

## Реальные проблемы и диагностика

Connection is not available request timed out after 30000ms — classical сообщение при исчерпании pool. Возможные причины утечка соединений (кто-то взял, не вернул), долгие транзакции с external API вызовами внутри, pool просто мал для нагрузки.

Диагностика. Включить leak-detection-threshold — получить stack trace откуда connection не возвращается. pg_stat_activity показывает что делают все backends database — можно найти долго висящие queries. Метрики hikaricp.connections.pending показывают частоту waits.

Прод тормозит после нескольких часов работы. Возможно max-lifetime не установлен и stale connections накопились. Или firewall убивает idle connections без обнаружения приложением. Fix — установить keepalive-time для регулярной проверки idle connections.

После БД maintenance приложение мертво. Connections не пересоздались после перезапуска database. Fix — HikariCP has broken connection detection, но требует настройки. keepalive-time помогает обнаружить проблемы быстро. connection-test-query для older pools эквивалент.

PgBouncer transaction mode plus prepared statements crash. Классическая проблема. Fix — отключить client-side prepared caching через prepareThreshold равное 0. Или переход на session mode если экономия connections не критична.

Реальный кейс knp-e2e-runner-hikari-isolation-poisoning из КНП memory. Раннер использовал opt-in isolation равное -1 что при shared через PgBouncer «отравляло» pool — subsequent transactions получали wrong isolation level. gate-knp краснел около 18 минут. Fix — явно установить transactionIsolation равное TRANSACTION_READ_COMMITTED вместо opt-in default.

## Production конфигурация пример

Полная production configuration для КНП микросервиса:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://db-knp:5432/knp?ApplicationName=isnaknpintegration
    username: ${DB_USER:knp}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    
    hikari:
      pool-name: knp-hikari
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 10000
      max-lifetime: 1200000              # меньше PG idle_in_transaction_session_timeout
      keepalive-time: 300000             # firewall keeper
      leak-detection-threshold: 60000
      auto-commit: false
      transaction-isolation: TRANSACTION_READ_COMMITTED
      data-source-properties:
        socketTimeout: 60                # секунды
        connectTimeout: 10
        # для PgBouncer transaction mode:
        prepareThreshold: 0
        preparedStatementCacheQueries: 0
        # для обычного PostgreSQL без PgBouncer:
        # prepareThreshold: 5
        # cachePrepStmts: true
        # useServerPrepStmts: true
        reWriteBatchedInserts: true      # batch INSERT переписываются в multi-value
```

Actuator для метрик:
```yaml
management:
  metrics:
    export:
      prometheus.enabled: true
  endpoints:
    web.exposure.include: health,metrics,prometheus,hikaricp
```

ApplicationName в URL позволяет identify connections в pg_stat_activity — очень полезно для debugging когда несколько микросервисов работают с одной database.

## Итоги

JDBC базовый API для работы с PostgreSQL из Java. PostgreSQL JDBC driver реализует TCP протокол, prepared statements, type conversion. try-with-resources для безопасного освобождения ресурсов.

Connection pool необходим для performance. Overhead создания соединения 30-100 миллисекунд — недопустимо на каждый запрос. Pool держит открытые соединения для reuse.

HikariCP default в Spring Boot. Самый быстрый и легковесный. Rich metrics через Micrometer.

Ключевые настройки. maximum-pool-size 10-30 обычно. connection-timeout 30 секунд default. max-lifetime 20-30 минут. leak-detection-threshold 60000 обязательно в production. transaction-isolation explicit значение чтобы избежать surprises.

connection-timeout vs socketTimeout vs IO wait разные концепции. Первый — ожидание из pool. Второй — timeout выполнения statement. Третий — waiting for I/O в OS.

Pageable в Spring Data. Page vs Slice. OFFSET expensive на глубоких страницах. Keyset pagination предпочтительна для big data.

PgBouncer connection pooler между приложением и PostgreSQL. Transaction mode даёт максимальную экономию но ограничивает prepared statements и session state.

SSL для production. sslmode require минимум, verify-full максимум. Overhead handshake амортизируется через pool.

Timezone всё в UTC. hibernate.jdbc.time_zone UTC. JVM -Duser.timezone UTC. timestamptz в PostgreSQL всегда.

Prepared statements обязательны для SQL injection prevention plus performance. Server-side prepare для повторяющихся queries. Отключить при PgBouncer transaction mode.

Реальные проблемы — pool exhaustion, stale connections, PgBouncer plus prepared statements skew, isolation poisoning. Каждая имеет known cause и standard fix.

Дальше — highload considerations и expensive operations как fokus на performance optimization в production системах.
