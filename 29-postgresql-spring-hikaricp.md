# 29. PostgreSQL + Spring Boot: HikariCP, connection pool, timeouts

Как Java-приложение работает с PG. Что такое пул соединений, его настройки, статусы, timeouts.

---

## 1. JDBC — как устроен

**JDBC** = Java Database Connectivity. Стандартный API для БД.

Основные объекты:
- **`DataSource`** — фабрика соединений.
- **`Connection`** — TCP-соединение с БД + сессия.
- **`Statement` / `PreparedStatement`** — SQL для выполнения.
- **`ResultSet`** — курсор для чтения результатов.

Цепочка:
```java
DataSource ds = getDataSource();
try (Connection conn = ds.getConnection()) {
    try (PreparedStatement ps = conn.prepareStatement("SELECT * FROM fno WHERE id=?")) {
        ps.setLong(1, 123);
        try (ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                // ...
            }
        }
    }
}
```

Реальный **PostgreSQL JDBC driver** — `org.postgresql:postgresql`. Отвечает за:
- TCP-соединение на порт 5432.
- Wire-протокол PostgreSQL (FE/BE протокол).
- Прeparation / execution SQL.
- Type conversion (BIT → boolean, TIMESTAMP → LocalDateTime).

---

## 2. Зачем connection pool

### 2.1 Проблема без пула

```java
Connection conn = DriverManager.getConnection(url, user, pw);
// ...work...
conn.close();
```

Каждый вызов = новое TCP-соединение + новый PG backend process:
- TCP handshake ~ несколько мс.
- SSL handshake (если) ~ 10-50 мс.
- PG аутентификация + init ~ 20-50 мс.
- **Итого: 30-100 мс на каждое соединение**. Для 1000 req/s = абсурд.

Плюс:
- Каждый backend PG = 5-10 MB памяти. 1000 open connections = 5-10 GB.
- PG имеет `max_connections` (default 100), быстро исчерпается.

### 2.2 Пул

Пул держит **N готовых открытых соединений**. Приложение берёт, использует, возвращает.

```
Приложение
    │
    │  getConnection()
    ▼
┌─────── Pool ───────┐
│  [free] [free]     │  ← готовые соединения
│  [in-use] [free]   │
└─────────┬──────────┘
          │
          ▼
    PostgreSQL
```

Плюсы:
- Соединение переиспользуется → нет дорогого init.
- Ограничение количества → PG не перегружен.
- Latency drops from 30ms to <1ms per connection acquire.

---

## 3. HikariCP — стандарт в Spring Boot

**HikariCP** — самый быстрый JDBC-пул. Default в Spring Boot начиная с 2.0.

Особенности:
- Легковесный.
- Минимум блокировок внутри.
- Хорошие defaults.
- Богатые метрики.

Альтернативы: Tomcat JDBC Pool, Apache DBCP2 — устаревшие.

---

## 4. Ключевые настройки

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
      idle-timeout: 300000               # 5 мин
      connection-timeout: 30000          # 30 сек
      max-lifetime: 1800000              # 30 мин
      keepalive-time: 300000             # 5 мин
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

Разберём каждую.

### 4.1 `maximum-pool-size`

Максимум соединений в пуле.

**Правило**: `pool_size = ((CPU_cores × 2) + effective_spindle_count)` для БД. Для приложения — обычно 10-30.

Больше НЕ значит быстрее:
- Больше 20-30 → contention внутри PG.
- Ставить 100+ = обычно ошибка.

Пример: PG с `max_connections=200`. 10 подов приложения × 30 = 300 → PG отказывает в соединениях.

**PgBouncer** решает эту проблему (см. §7).

### 4.2 `minimum-idle`

Сколько соединений всегда держать открытыми (даже без нагрузки). По умолчанию = max.

Установить = max для стабильной latency (нет старта соединения при первом запросе).
Установить меньше = экономия ресурсов, но первый запрос после idle-периода долгий.

### 4.3 `connection-timeout` ⭐

**Максимум сколько приложение ждёт свободного соединения из пула**.

Default: 30 сек.

Если пул полный и все in-use → getConnection() блокируется. Через `connection-timeout` → бросается `SQLTransientConnectionException: HikariPool - Connection is not available, request timed out after 30000ms`.

**Слишком большой** = приложение висит.
**Слишком маленький** = false alarm при коротком пике.

30 сек — разумно для дефолта.

### 4.4 `idle-timeout`

Через сколько миллисекунд простаивающее соединение (сверх `minimum-idle`) закрывается.

Default: 10 мин. Обычно 5-10 мин ок.

### 4.5 `max-lifetime` ⭐

**Максимальный возраст соединения**. По достижении — закрывается и открывается новое.

Default: 30 мин.

**Зачем**: избежать проблем с "stale" соединениями (сеть, firewall, PG перезапуск на реплике). PG сам может убить долгоживущие.

**Правило**: `max-lifetime` < `pg-idle-timeout` минус несколько секунд. И меньше firewall's TCP timeout.

### 4.6 `keepalive-time` (Hikari 4.0+)

Периодически проверяет idle соединения (`SELECT 1`), чтобы держать TCP живым.

Default: disabled (0). Устанавливай в 5 минут если сеть закрывает idle-соединения (firewall).

### 4.7 `validation-timeout`

Максимум на выполнение validation query.

### 4.8 `leak-detection-threshold` ⭐

Если соединение не возвращено в пул за N мс → log warning со stack trace `откуда взяли`.

Default: 0 (выключено). **Включай в проде** = 60000 (60 сек). Помогает найти утечки:
```
HikariCP - Connection leak detection triggered for ... on thread ..., stack trace follows
    at ...FnoService.badMethod(FnoService.java:42)
```

### 4.9 `auto-commit`

`true` (default) — каждый statement = отдельная tx.
`false` — надо явно commit/rollback.

Spring `@Transactional` управляет вручную → значение не критично.

### 4.10 `transaction-isolation`

Уровень изоляции по умолчанию. Для PG обычно `TRANSACTION_READ_COMMITTED`.

**Реальный ИСНА-кейс** memory `knp-e2e-runner-hikari-isolation-poisoning`: раннер использовал `isolation=-1` (opt-in), пул отравлялся. Всегда явно задавай.

---

## 5. Статусы соединений в пуле

Каждое соединение в одном из состояний:

- **idle** — свободно, ждёт использования.
- **active / in-use** — выдано приложению, используется.
- **awaiting** — приложение ждёт соединение (пул исчерпан).
- **stale / evicted** — превысило `max-lifetime` или `idle-timeout`, закрывается.

Метрики:
```
hikaricp.connections.active
hikaricp.connections.idle
hikaricp.connections.pending    ← сколько ждёт
hikaricp.connections.timeout    ← сколько раз таймаутнули
hikaricp.connections.usage
hikaricp.connections.acquire     ← latency getConnection
```

Экспорируются через Actuator + Micrometer → Prometheus.

**Правило мониторинга**:
- `pending > 0` часто → пул мал, увеличить.
- `timeout > 0` → критично, что-то не так (утечка / долгие tx / БД тормозит).
- `active / max_pool_size` близко к 1 → перегрузка.

---

## 6. Что такое connection timeout, io wait

### 6.1 Connection timeout

Уже разобрано (§4.3) — сколько приложение ждёт **соединение из пула**.

Отличать от **socket connect timeout** (соединение с БД на TCP уровне):
```yaml
spring.datasource.hikari.data-source-properties:
  socketTimeout: 30                  # секунды
  connectTimeout: 10                 # для установления TCP
```

Первый — очередь в пуле. Второй — сеть до БД.

### 6.2 Socket timeout

Максимум на **выполнение statement**. Если запрос идёт >30 сек — driver прерывает.

Установи! Иначе злой запрос повесит весь пул.

### 6.3 IO wait

**IO wait** = процессор ждёт I/O операцию (диск, сеть).

Для БД-запроса типичное:
1. Приложение отправляет SQL по сети (~1 мс).
2. **PG работает над запросом** (может 5-500 мс).
3. Возвращает результат по сети (~1 мс).

Всё это время Java-поток **заблокирован** — ждёт I/O. CPU простаивает (может быть занят другими потоками).

Мониторить:
- В приложении — метрика `hikari.connections.acquire.time` + `jdbc.query.time` (если есть APM).
- В OS — `top` показывает `%wa` (CPU waiting for I/O).
- В PG — `pg_stat_activity.wait_event`.

Высокий IO wait в JVM = много блокирующих I/O операций. Решения:
- **Batch** — уменьшить количество round-trip.
- **Reactive / Virtual Threads** — не блокировать поток на I/O.
- **Кэш** — избежать I/O.
- **Индексы** — ускорить сам запрос.

### 6.4 Что такое Pageable

Не про пул, но пользователь спросил. Разбирал в файле `14-spring-data-jpa.md`, повторим кратко.

**`Pageable`** — Spring абстракция для пагинации:
```java
Page<Fno> page = repo.findAll(PageRequest.of(0, 20, Sort.by("createdAt").descending()));
```

Под капотом:
- `SELECT ... LIMIT 20 OFFSET 0` — данные.
- `SELECT COUNT(*) ...` — total count.

**`Page`** содержит:
- `content` — список.
- `totalElements` — общее количество.
- `totalPages`.
- `hasNext`, `hasPrevious`.

**`Slice`** — без COUNT, только `hasNext` (быстрее, если total не нужен).

**Кавет OFFSET на больших страницах**: `OFFSET 100000 LIMIT 20` заставит PG прочитать 100020 строк, отбросить 100000. Медленно! Решение — **keyset pagination**:
```sql
WHERE created_at < :last_seen_at ORDER BY created_at DESC LIMIT 20
```

Курсор двигается вперёд по значению, без OFFSET.

---

## 7. PgBouncer

Между приложением и PG часто ставят **PgBouncer** — connection pooler на уровне сети.

### 7.1 Зачем

- Приложение может держать 100 connections к PgBouncer.
- PgBouncer держит только 20 к настоящему PG.
- Мультиплексирует запросы.

### 7.2 Режимы

**Session** — одно соединение приложения = одно соединение PG на весь сеанс.
- Плюс: как обычный PG.
- Минус: не помогает уменьшить количество PG connections.

**Transaction** ⭐ — соединение возвращается в пул после каждой транзакции.
- Плюс: экономия соединений PG (100 приложений → 10 PG).
- **Минус**: **prepared statements не работают** (кэш на уровне session).

**Statement** — после каждого statement.
- Не поддерживает транзакции клиента.
- Только для аналитики.

### 7.3 В ИСНА

Memory `knp-fs-consul-deregister-after-db-flap`: `db-knp` = PgBouncer в pod-сети (с хоста не пинганеть, exec из пода).

Обычно **transaction mode**. Отсюда:
- Отключить prepared statements caching в JDBC:
  ```yaml
  data-source-properties:
    prepareThreshold: 0
    preparedStatementCacheQueries: 0
  ```
- Не использовать `SET ...` (session-level settings) — теряются между транзакциями.

---

## 8. SSL

По умолчанию JDBC PG не шифрует. Для production обычно включают:
```yaml
spring.datasource.url: jdbc:postgresql://db:5432/knp?sslmode=require
```

Режимы `sslmode`:
- `disable` — нет SSL.
- `allow` — предпочтителен без.
- `prefer` — предпочтителен с, fallback без.
- `require` — только SSL.
- `verify-ca` — + проверка cert authority.
- `verify-full` — + проверка hostname.

SSL handshake ~ 10-50 мс — оправдан только один раз при open connection (пул спасает).

---

## 9. Timezone

Классическая проблема JDBC + PG.

```yaml
spring.jpa.properties.hibernate:
  jdbc.time_zone: UTC
```

Плюс `-Duser.timezone=UTC` в JVM args.

Иначе `LocalDateTime` может интерпретироваться в разных TZ на разных нодах → путаница.

**Правило**: **всё в UTC внутри**. Отображение в UI — конвертация к timezone пользователя.

---

## 10. Prepared statements

### 10.1 Что это

```java
PreparedStatement ps = conn.prepareStatement("SELECT * FROM fno WHERE reg_num = ?");
ps.setString(1, "12345");
ResultSet rs = ps.executeQuery();
```

vs regular statement:
```java
Statement st = conn.createStatement();
ResultSet rs = st.executeQuery("SELECT * FROM fno WHERE reg_num = '12345'");
```

Плюсы prepared:
- **SQL injection** невозможен (параметры отдельно).
- **План запроса cache**ится на сервере — быстрее следующие вызовы.
- **Batch** возможен.

### 10.2 Client vs server side

**Client side**: JDBC формирует SQL с подставленными значениями, шлёт как обычный statement.

**Server side** ⭐: JDBC шлёт `PREPARE` + `EXECUTE` — PG хранит план.

Настройка PG JDBC:
```yaml
data-source-properties:
  prepareThreshold: 5              # начать серверный prepare после 5-го использования
  preparedStatementCacheQueries: 256
  preparedStatementCacheSizeMiB: 5
```

Хороший тюнинг для повторяющихся запросов.

### 10.3 PgBouncer transaction mode ломает

Как упоминалось (§7.2) — сервер prepared statements не сохраняются между транзакциями в PgBouncer transaction. Отключай.

---

## 11. Реальные проблемы и диагностика

### 11.1 `Connection is not available, request timed out after 30000ms`

Причины:
- Утечка соединения (кто-то взял, не отдал).
- Долгие транзакции (внешний API внутри `@Transactional`).
- Пул мал для нагрузки.

Диагностика:
- Включить `leak-detection-threshold` — stack trace где утечка.
- `pg_stat_activity` — что делают backend'ы.
- Метрики `hikaricp.connections.pending`.

### 11.2 «Прод тормозит после нескольких часов»

- Возможно `max-lifetime` не установлен → stale connections накопились.
- Или firewall убивает idle → нужен `keepalive-time`.

### 11.3 «После БД maintenance приложение мертво»

- Соединения не пересоздались.
- Fix: `keepalive-time` + `connection-test-query` (для старых пулов).
- HikariCP умеет detect broken → пересоздать.

### 11.4 «PgBouncer, но prepared statements крашатся»

`transaction` mode → отключай client-side prep cache:
```
prepareThreshold=0
preparedStatementCacheQueries=0
```

Или переходи на `session` mode (теряя экономию соединений).

### 11.5 Реальный ИСНА: HikariCP #2269 isolation=-1

Memory `knp-e2e-runner-hikari-isolation-poisoning`: raннер брал с opt-in isolation → пул на pp-pgbouncer отравлялся → gate-knp краснел ~18 мин. Фикс = явный `transactionIsolation: TRANSACTION_READ_COMMITTED`.

---

## 12. Прод-конфигурация (пример)

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
      max-lifetime: 1200000               # < PG idle_in_transaction_session_timeout
      keepalive-time: 300000              # firewall keeper
      leak-detection-threshold: 60000
      auto-commit: false
      transaction-isolation: TRANSACTION_READ_COMMITTED
      data-source-properties:
        socketTimeout: 60                  # секунды
        connectTimeout: 10
        # для PgBouncer transaction mode:
        prepareThreshold: 0
        preparedStatementCacheQueries: 0
        # для обычного PG (не через PgBouncer):
        # prepareThreshold: 5
        # cachePrepStmts: true
        # useServerPrepStmts: true
        reWriteBatchedInserts: true
```

```yaml
management:
  metrics:
    export:
      prometheus.enabled: true
  endpoints:
    web.exposure.include: health,metrics,prometheus,hikaricp
```

---

## 13. Собесные вопросы

1. **Зачем connection pool?** — Избежать overhead создания соединения (TCP + auth) + ограничить количество к БД.
2. **Что такое HikariCP?** — Самый быстрый JDBC-пул; default в Spring Boot.
3. **Что такое `maximum-pool-size`?** — Максимум соединений; правило `(CPU × 2) + spindle`, обычно 10-30.
4. **Что такое `connection-timeout`?** — Максимум ожидания соединения из пула.
5. **Что такое `max-lifetime`?** — Максимальный возраст соединения; после — закрывается.
6. **Разница `connection-timeout` и `socketTimeout`?** — Первый = ожидание в пуле; второй = ожидание ответа от БД.
7. **Что такое `leak-detection-threshold`?** — Log warning со stack trace если соединение не возвращено N мс.
8. **Статусы соединений в пуле?** — idle, active/in-use, awaiting, evicted.
9. **Что такое PgBouncer?** — Connection pooler между приложением и PG; режимы session/transaction/statement.
10. **PgBouncer transaction mode и prepared statements?** — Не работают (сохраняются в session, а session между tx меняется).
11. **Что такое `Pageable` в Spring Data?** — Абстракция пагинации (page + size + sort); Page/Slice.
12. **Проблема OFFSET на больших страницах?** — PG читает все N+offset строк; лучше keyset pagination.
13. **Что такое IO wait?** — Процесс/поток блокирован ожидая I/O (сеть, диск).
14. **Prepared statements — зачем?** — SQL injection prevention + план кэшируется + batch.
15. **Timezone в JDBC — как правильно?** — Всё в UTC (JVM + Hibernate `jdbc.time_zone: UTC`).

---

## Итог

- **JDBC** → PostgreSQL через `postgresql` driver.
- **HikariCP** = default пул, легко настраивается.
- **Ключевые настройки**: `maximum-pool-size`, `connection-timeout`, `max-lifetime`, `leak-detection-threshold`.
- **PgBouncer** — экономит PG connections, но требует настроек в JDBC.
- **Prepared statements** = must (безопасность + производительность).
- **Timezone** = UTC везде.
- **Мониторинг**: Hikari metrics в Prometheus, `pg_stat_activity`.

Следующий — `30-highload-expensive-operations.md`.
