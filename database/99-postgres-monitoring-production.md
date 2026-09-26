# 99. Мониторинг PostgreSQL в production

## Зачем и что мониторить

Есть старое правило системного администрирования: если ты не мониторишь систему, ты не знаешь как она работает. Ты знаешь только как она *выглядит* когда ты смотришь. Между наблюдениями может происходить всё что угодно. Реальные production проблемы часто проявляются в неудобное время — ночью, в выходные, во время maintenance у соседних систем. Без continuous monitoring ты узнаёшь про них когда пользователи начинают жаловаться, и то не сразу.

Правильный мониторинг PostgreSQL решает три задачи. Первая — **проактивное обнаружение проблем**. Метрики показывают тренды: cache hit ratio медленно падает, replication lag растёт, disk usage приближается к пределу. Alerts срабатывают до того как проблема станет критичной. Вторая — **диагностика во время инцидента**. Когда что-то падает, dashboards показывают текущее состояние всех компонентов, история — что предшествовало. Не нужно догадываться, вот данные. Третья — **capacity planning**. Тренды за месяцы говорят о том, что через два месяца понадобится upgrade, что определённая нагрузка растёт линейно, что можно оптимизировать.

В этом файле разберём практическую сторону monitoring PostgreSQL. Какие метрики собирать (их десятки, но реально важных — двадцать). Как их собирать через postgres_exporter и Prometheus. Как визуализировать в Grafana. Как настроить alerting так, чтобы получать нужные уведомления вовремя без spam'а от ложных срабатываний. SLI/SLO подход. Полезные dashboards как starting points.

## Категории метрик

Метрики PostgreSQL логически делятся на несколько групп. Каждая говорит о своём аспекте здоровья системы.

**Availability и connections**. Работает ли база вообще, сколько соединений открыто, есть ли ошибки подключения. Первый уровень: если это не в порядке — ничего другое не важно.

**Performance**. Queries per second, latency, transactions per second, deadlocks. Показывает как быстро система обслуживает нагрузку.

**Resource usage**. CPU, memory (shared_buffers, work_mem usage), disk (space, IOPS), network. Насколько эффективно используются ресурсы, есть ли запас.

**Cache и I/O**. Cache hit ratio (в shared_buffers и OS page cache), количество disk reads, temp files. Указывает где узкое место — CPU/RAM или диск.

**Vacuum и bloat**. Как autovacuum работает, растёт ли bloat, есть ли отстающие таблицы. Долгосрочное здоровье БД зависит от этого.

**Replication**. Replication lag, статус реплик, WAL generation. Если репликация настроена — критически важно.

**Locks и blocking**. Активные locks, blocked sessions, deadlock count. Показывает конкурентные проблемы.

**Errors и logs**. Fatal errors, connection failures, disk errors. Ранние признаки серьёзных проблем.

## postgres_exporter

Стандартный инструмент для сбора метрик PostgreSQL — **postgres_exporter**. Написан на Go, работает как отдельный процесс (или Kubernetes sidecar). Подключается к PostgreSQL, периодически (обычно каждые 15-60 секунд) выполняет queries к системным views (pg_stat_activity, pg_stat_database, pg_stat_replication и другие), преобразует результаты в Prometheus-compatible метрики, exposes их через HTTP endpoint.

Prometheus периодически scrape'ит exporter, сохраняет метрики в time-series storage. Grafana подключается к Prometheus для визуализации.

Простая настройка postgres_exporter (Docker):

```yaml
version: '3'
services:
  postgres_exporter:
    image: quay.io/prometheuscommunity/postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://exporter:password@postgres:5432/postgres?sslmode=disable"
    ports:
      - "9187:9187"
```

Роль в PostgreSQL для exporter'а:

```sql
CREATE USER exporter WITH PASSWORD '...';
GRANT pg_monitor TO exporter;
-- pg_monitor — встроенная роль с правами на все stats views
```

Не давай exporter'у full SELECT privileges на data — только на statistics.

## Ключевые метрики: что реально важно

Не все метрики одинаково важны. Есть десятки, реально критичны — двадцать.

**pg_up** — работает ли сервер. Простая, но важнейшая метрика. Если 0 — сервер недоступен, всё остальное неважно. Alert immediate.

**pg_stat_database_numbackends** — количество активных соединений на database. Приближение к `max_connections` — плохо (новые соединения будут получать ошибку). Alert при 80% от max_connections.

**pg_stat_database_xact_commit** и **pg_stat_database_xact_rollback** — количество коммитов и rollback'ов. Ratio (rollback / total) показывает "health" транзакций: высокий rollback rate = приложение делает что-то неправильно.

**pg_stat_database_blks_hit** и **pg_stat_database_blks_read** — cache hits и reads с диска. `blks_hit / (blks_hit + blks_read)` = cache hit ratio. Должно быть >99% для OLTP. Alert при <95%.

**pg_stat_database_deadlocks** — количество deadlocks. Any deadlock — прикинуть; регулярные (несколько в час) — проблема.

**pg_stat_database_temp_bytes** — сколько данных писалось во временные файлы (spill сортировок). Много — work_mem мал или plans plohye.

**pg_stat_database_conflicts** — конфликты между запросами на standby и replication. Много — надо разобраться с `max_standby_streaming_delay` или `hot_standby_feedback`.

**pg_stat_activity_max_tx_duration** — самая долгая активная транзакция. Растёт — где-то забыт COMMIT, что приведёт к bloat.

**pg_stat_activity idle_in_transaction** count — сколько соединений в состоянии `idle in transaction`. Растёт — приложение делает BEGIN но забывает COMMIT.

**pg_stat_replication_replay_lag_bytes** и **replay_lag_seconds** — на сколько standby отстаёт. Растёт — что-то не так с сетью, диском standby или большие транзакции.

**pg_stat_bgwriter_buffers_checkpoint** — сколько pages пишется по checkpoint. Большой скачок — много writes. Плюс `checkpoints_req` — checkpoint по требованию (плохо, должен быть по времени).

**pg_stat_user_tables_n_dead_tup** — dead tuples по таблицам. Растёт быстрее чем vacuum убирает — autovacuum отстаёт.

**pg_stat_user_tables_seq_scan / idx_scan** — соотношение sequential scan'ов к index scan'ам. Много seq scan — проблема с индексами.

**pg_locks_count** — количество locks сейчас. Резкий скачок — что-то блокирует много.

**pg_settings_max_connections** vs **pg_stat_activity count** — сколько connections из максимума занято.

**Custom метрики**. postgres_exporter поддерживает custom queries через yaml файл — можешь добавить любые бизнес-метрики.

## Prometheus и retention

Prometheus сохраняет time-series данные локально (по умолчанию 15 дней). Для более долгого retention — remote storage (VictoriaMetrics, Thanos, Cortex).

Настройка scrape'а PostgreSQL exporter'а:

```yaml
scrape_configs:
  - job_name: postgres
    scrape_interval: 30s
    static_configs:
      - targets: ['postgres-exporter:9187']
        labels:
          instance: postgres-primary
```

30-секундный interval — компромисс. Меньше — детализация выше, storage растёт. Больше — можешь пропустить короткие спайки.

Для КНП разумно: scrape каждые 15-30 секунд, retention в Prometheus 15-30 дней, remote storage (VictoriaMetrics) для года истории. Не нужны миллисекундные детализации metric'ов уровня цикла CPU, важны минутные тренды.

## Alerting

Метрики без алертов — просто картинки. Alerts переводят observability в actionable — знать когда что-то не так, до того как users сообщат.

Prometheus Alertmanager — стандартный компонент. Определяешь правила: если metric X удовлетворяет условию Y течение времени Z — send alert.

Простые правила:

```yaml
groups:
  - name: postgres_critical
    rules:
      - alert: PostgresDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is down on {{ $labels.instance }}"

      - alert: HighConnections
        expr: pg_stat_database_numbackends / pg_settings_max_connections > 0.8
        for: 5m
        labels:
          severity: warning

      - alert: ReplicationLagHigh
        expr: pg_stat_replication_replay_lag_seconds > 30
        for: 5m
        labels:
          severity: critical

      - alert: LowCacheHitRatio
        expr: pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read + 1) < 0.95
        for: 15m
        labels:
          severity: warning
```

Ключевые практики alerting.

**`for` clause обязательно**. Значение должно превышать порог **в течение времени**, не мгновенно. Иначе flip-flop alerts на каждый flap. Обычно 5-15 минут.

**Severity levels**. `critical` — pager (кого-то будят ночью). `warning` — Slack/email в business hours. `info` — просто в лог для history.

**Actionable alerts only**. Каждый alert должен иметь runbook: что делать когда сработал. Иначе получатели не знают что делать, alerts просто игнорируются.

**Avoid alert fatigue**. Слишком много alerts = игнорирование. Лучше меньше но точные alerts. Регулярно review — какие срабатывали часто без действий, отключить.

## SLI и SLO

Правильный подход к monitoring — построение через SLI (Service Level Indicators) и SLO (Service Level Objectives).

**SLI** — конкретная метрика, отражающая user experience. Например: доля запросов ответивших быстрее 500 ms; доля транзакций закоммиченных успешно; uptime базы.

**SLO** — target для SLI. Например: 99.9% запросов быстрее 500 ms; 99.99% транзакций успешны; 99.95% uptime за месяц.

Разница с обычными метриками — SLO ориентированы на user, не на систему. Cache hit ratio 99% — техническая метрика. "P99 latency < 100ms" — user-facing.

Стандартная SLO модель:

- **Availability SLO**: `uptime = 100% - downtime_percent`.
- **Latency SLO**: `latency_p95 <= threshold`.
- **Success rate SLO**: `error_rate <= threshold`.

Для PostgreSQL SLO могут быть:

- Availability: 99.95% uptime за месяц (около 22 минут допустимого downtime).
- Latency: 99% транзакций коммитятся быстрее 100 ms.
- Success rate: <0.1% transactions rolled back (не по бизнес-причинам).

**Error budget**: доля времени/запросов, в которой можно нарушать SLO без последствий. Если SLO 99.9% availability, error budget = 0.1% времени в месяц (~43 минуты). Если использовали больше — freeze deployments, focus на reliability.

Google SRE книга — основа этого подхода, стоит прочитать. Для критичных систем внедрение SLO дисциплинирует команду: не «система работает или падает», а «мы обещали X, соблюдаем ли».

## Полезные Grafana dashboards

Готовые dashboards экономят часы конфигурации. Ключевые для PostgreSQL:

**PostgreSQL Database dashboard (Percona)**. Комплексный обзор: connections, transactions, performance, WAL, replication. Один из самых популярных.

**PostgreSQL Exporter Quickstart and Dashboard**. От авторов postgres_exporter, покрывает базовые метрики.

**PMM Database Dashboards**. Percona Monitoring & Management — набор dashboards для разных типов баз.

Кастомизировать под свой контекст. Для КНП: dashboard'ы для каждого сервиса-БД с business метриками (транзакции по типу операций, конкретные бизнес-запросы, специфические SLO).

Отдельный dashboard для incident response: cheat sheet с ключевыми запросами (top slow queries сейчас, blocking queries, stuck transactions). Открывается при инциденте, показывает всё сразу.

## Логирование

Метрики — это агрегаты. Иногда нужны детали. Логи PostgreSQL — источник детальной информации.

Ключевые настройки логирования:

```
logging_collector = on
log_destination = 'csvlog'
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_size = 100MB

log_min_duration_statement = 1000  -- логировать statements >1сек
log_line_prefix = '%m [%p] %q%u@%d '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0  -- любые temp files
log_autovacuum_min_duration = 250ms
```

Что логируется:

- Slow queries (log_min_duration_statement).
- Checkpoints — timing и размер.
- Connections/disconnections.
- Lock waits — когда query висит на locke дольше `deadlock_timeout`.
- Temp files — когда operations spill'ают на диск.
- Autovacuum activity.

Логи в CSV формате легко parseable. Отправлять централизованно (Loki, Elasticsearch/Kibana) для search и analysis.

Плюс **pg_stat_statements** — не логи, а таблица со статистикой всех выполненных queries. Мы обсуждали в 89. Обязательный extension для мониторинга запросов.

**auto_explain** — extension который логирует EXPLAIN plans для медленных queries автоматически. Без ручного EXPLAIN ANALYZE. Просто по log:

```
auto_explain.log_min_duration = 5000
auto_explain.log_analyze = on
auto_explain.log_buffers = on
```

Медленные (>5с) queries автоматически получают plan в лог. Ценно для post-mortem анализа.

## Что мониторить на OS уровне

PostgreSQL — процесс в OS, метрики Linux тоже важны.

**CPU**. `node_cpu_seconds_total` — utilization по user/system/idle/iowait. Высокий iowait = disk bottleneck. Высокий user = CPU-bound.

**Memory**. `node_memory_MemAvailable_bytes` — сколько реально доступно. `node_memory_SwapUsed` — если swap активно используется, катастрофа.

**Disk**. `node_disk_read/write_bytes` — bandwidth. `node_disk_iops` — IOPS. `node_filesystem_avail_bytes` — свободное место (alert <15%).

**Network**. `node_network_receive/transmit_bytes` — трафик. `node_network_receive_errs` — ошибки.

node_exporter — Prometheus exporter для Linux OS метрик. Устанавливается на каждом сервере.

Grafana dashboard "Node Exporter Full" — стандартный, показывает всё.

## Мониторинг репликации

Отдельная тема. Replication должна быть separately monitored, потому что failure там не всегда видна на основных метриках БД.

Ключевое:

**pg_stat_replication** — состояние соединений с standby'ями. Sent_lsn, write_lsn, flush_lsn, replay_lsn.

**pg_replication_slots** — replication slots. Retained WAL size — если растёт, standby не догоняет или мертв, скоро диск заполнится.

**pg_stat_wal_receiver** (на standby) — статус приёма WAL.

Alerts:

- Replay lag > 30 секунд — warning.
- Replay lag > 5 минут — critical.
- Replication slot retained WAL > 10 GB — critical (скоро диск).
- Standby down — critical.

## Инцидент response через мониторинг

Ситуация: пользователи жалуются, что кабинет медленный. Ты открываешь мониторинг.

Первый шаг — общее здоровье. PostgreSQL up? Да. Все реплики up? Да. Connections в норме? Да. CPU/RAM/Disk на серверах? В норме.

Второй — недавние изменения. График latency за последние часы. Есть скачок в 14:30. Что было в 14:30? Смотришь deployment log — deployed новая версия. Ага, возможно новая версия имеет плохой запрос.

Третий — конкретика. pg_stat_statements — какие запросы стали дольше или чаще после 14:30? Находишь новый query, который вызывается тысячи раз в минуту, каждый по 500 мс.

Четвёртый — план. EXPLAIN ANALYZE этого query. Nested Loop на большом результате. Estimated rows 100, actual rows 1000000. Missing статистика или отсутствующий индекс.

Пятый — fix. Добавляешь индекс через CREATE INDEX CONCURRENTLY, ANALYZE. Latency восстанавливается.

Весь этот workflow занимает 15-30 минут потому что мониторинг показывает нужное сразу. Без него — часы блужданий.

## Заключение

Мониторинг PostgreSQL — не опция, а необходимость для production. Три задачи: proactive detection, incident diagnosis, capacity planning. Метрики (агрегаты) плюс логи (детали) плюс traces (opcional, для complex distributed cases).

postgres_exporter + Prometheus + Grafana — стандартный стек. Роль pg_monitor для exporter. Scrape 15-30 сек. Retention 15-30 дней в Prometheus, дольше в remote storage (VictoriaMetrics/Thanos).

Ключевые метрики: pg_up, connections, transactions rate, cache hit ratio, deadlocks, temp bytes, replication lag, dead tuples, checkpoints, locks. Двадцать реально важных из десятков доступных.

Alerts через Alertmanager с `for` clause, severity levels, actionable definitions с runbooks. Регулярно review чтобы избежать alert fatigue.

SLI/SLO подход — user-oriented метрики, targets, error budget. 99.9% availability, 99% latency<100ms — типичные SLO для критичной OLTP базы.

Логирование: log_min_duration_statement, log_lock_waits, log_checkpoints, log_autovacuum. auto_explain для automatic plans на slow. pg_stat_statements обязательно.

OS-level мониторинг (node_exporter) — CPU, memory, disk, network. iowait для disk bottleneck detection. Swap usage как критический alert.

Replication — separate monitoring: replay_lag, replication slot retention, standby availability.

Для КНП правильно: prometheus + grafana + alertmanager стандартный setup. postgres_exporter + node_exporter на каждой БД. Отдельный dashboard per service. Alertов не больше 20, каждый actionable. SLOs определены (99.9% availability, 99% latency P95 < 100ms для типичных queries). Log в централизованный ELK. Regular review dashboards и alerts.

Дальше — практика. Разверни postgres_exporter локально, подключи Prometheus, увидь метрики. Настрой alertmanager с одним test alert. Импортируй Percona dashboard в Grafana, посмотри что показывает. Симулируй проблемы (SELECT pg_sleep(100), forget COMMIT, kill процесса) и смотри что видно в мониторинге. Практика monitoring — единственный способ научиться.
