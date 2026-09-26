# 90. Репликация и высокая доступность PostgreSQL

## Зачем нужна репликация

Одна база на одном сервере — это точка отказа. Умер сервер, умерла база, встал сервис. Диск сгорел, данные потеряны. Датацентр отключили — регион недоступен. Для любой серьёзной системы это неприемлемо. Ответ индустрии — реплицирование данных на несколько машин, желательно в разных датацентрах, желательно с автоматическим переключением при отказе.

Кроме отказоустойчивости репликация решает ещё две задачи. Первая — масштабирование чтения. Одна база выдерживает конечное количество запросов; если добавить реплики только для чтения, суммарная нагрузка распределяется. Вторая — снятие резервных копий и запуск аналитических запросов без нагрузки на prod: делаешь всё это на реплике, primary при этом работает без помех.

PostgreSQL предоставляет два принципиально разных механизма репликации: **streaming replication** (физическая, на уровне WAL) и **logical replication** (логическая, на уровне отдельных изменений строк). Каждый со своими плюсами, минусами, сферой применения. И на этом уровне есть ещё одна тема — HA (high availability), автоматическое переключение при отказе. Тут появляются внешние инструменты: Patroni, repmgr, etcd/Consul для координации.

В этом файле разберём всё это подробно. Как работает streaming replication изнутри, что такое replication slot, чем отличается synchronous от asynchronous, как настраивается cascading. Logical replication на уровне publisher/subscriber, где применима, где нет. Patroni как de facto стандарт для HA, роль etcd/Consul, механика failover, split-brain и как его избежать. Мониторинг лага, специфика чтения с реплики, консистентность.

## Streaming replication — как работает физически

Streaming replication работает на уровне WAL. Напомним из предыдущих файлов: каждое изменение данных в PostgreSQL порождает WAL-запись, которая сначала попадает в wal_buffers, потом в WAL-файл на диске. При COMMIT'е делается fsync WAL — только после этого клиент получает подтверждение.

Идея streaming replication проста: этот же WAL-поток отправлять по сети на другой сервер, и там применять к копии данных. Копия остаётся синхронизированной с исходной базой практически в реальном времени.

Механически это работает так. На primary есть процесс **wal sender**. На standby (реплике) — **wal receiver**. Standby подключается к primary через специальное replication-соединение (обычное соединение к порту 5432, но с параметром `replication=true`). Wal sender стримит WAL начиная с определённой позиции (LSN). Wal receiver принимает, записывает в свой локальный WAL и одновременно передаёт **startup process**, который применяет изменения к data files.

Результат: standby всегда содержит **физически идентичную** копию primary. Каждый байт файлов данных на standby совпадает с primary. Именно поэтому это называется физической репликацией — реплицируется не логика изменений, а физические байты страниц.

Настраивается это в конфиге PostgreSQL. На primary:

```
wal_level = replica
max_wal_senders = 10
wal_keep_size = 2GB
```

`wal_level = replica` включает генерацию WAL достаточной подробности для репликации (в отличие от `minimal`, где WAL — только для crash recovery). `max_wal_senders` — сколько standby могут одновременно подключаться. `wal_keep_size` — сколько WAL держать «на всякий случай», если standby отстанет.

На standby создаётся `standby.signal` файл (пустой) — это триггер, говорящий PostgreSQL запуститься в режиме репликации. Плюс конфиг:

```
primary_conninfo = 'host=primary-host port=5432 user=replicator password=xxx'
primary_slot_name = 'standby1_slot'
```

`primary_conninfo` — как подключаться к primary. `primary_slot_name` — какой replication slot использовать (см. ниже).

## Replication slots

Одна из тонких но важных концепций. Что если standby отстал, а WAL на primary уже переиспользован? Standby больше не сможет догнать — нужных WAL-записей просто нет. Единственный выход — базовый бэкап и синхронизация заново.

**Replication slot** решает эту проблему. Это персистентный маркер на primary, указывающий: «эта реплика подключена и должна получить WAL начиная с LSN X». Пока slot существует, primary не переиспользует WAL, которые ещё нужны этой реплике. Даже если реплика отключилась на несколько дней, WAL сохраняется, при повторном подключении она догонит с той точки, где остановилась.

Создание slot:

```sql
SELECT pg_create_physical_replication_slot('standby1_slot');
```

Standby в конфиге указывает `primary_slot_name = 'standby1_slot'`, при подключении заявляет свой slot, primary начинает удерживать WAL для него.

Оборотная сторона: если реплика умерла окончательно, а slot остался, primary будет держать WAL бесконечно, файловая система заполнится и primary сам ляжет. Отсюда правило: если реплика reconstruable (можешь сделать новый бэкап и стартовать заново), можно работать без slot. Если реплика критична и медленно стартует, slot нужен. И слоты забытых реплик надо удалять:

```sql
SELECT pg_drop_replication_slot('old_slot');
```

Мониторинг слотов:

```sql
SELECT slot_name, active, restart_lsn, 
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

Если `retained_wal` растёт — реплика не догоняет или отключена, WAL накапливается.

## Synchronous vs asynchronous

По умолчанию streaming replication асинхронная. Primary не ждёт подтверждения от standby при COMMIT'е — просто fsync WAL локально, отвечает клиенту, а стриминг идёт в фоне. Плюс: latency коммита не зависит от сети до standby. Минус: если primary внезапно упал и потерял WAL до отправки, эти коммиты потеряны.

**Синхронная репликация** — primary дополнительно ждёт подтверждения от standby перед возвратом клиенту. Настраивается через `synchronous_standby_names` в конфиге:

```
synchronous_standby_names = 'standby1'
```

Standby, которое primary объявил синхронным, должно подтвердить, что WAL до commit LSN получен. Только после этого клиент получает подтверждение.

Насколько синхронно — определяется параметром `synchronous_commit`:
- `on` (default) — ждём fsync локально И flush на synchronous standby.
- `remote_write` — локально fsync + просто write (не fsync) на standby. Быстрее, но реплика может потерять последние ms при падении.
- `remote_apply` — самый строгий: ждём пока standby реально применит изменение и оно станет видимо чтения. Дороже всего.
- `local` — только локально, игнорируем sync standby.
- `off` — вообще ничего не ждём.

Trade-off: полная синхронность даёт zero data loss (если primary падает, всё закоммиченное точно есть на standby), но добавляет latency коммитов (одна round-trip до standby). Для приложений с очень строгими требованиями к durability (финансы, регуляторные) — оправдано. Для типичного веб-приложения — обычно избыточно, асинхронная достаточна.

Есть промежуточный вариант — **quorum-based** синхронная репликация. Настраивается:

```
synchronous_standby_names = 'ANY 2 (standby1, standby2, standby3)'
```

Означает: ждём подтверждения от любых двух из трёх standby'ев. Позволяет пережить отказ одной реплики без потери synchronous гарантий и без потери latency (пока хотя бы две standby живы).

## Cascading replication

Стандарт: standby подключаются напрямую к primary. Но primary не бесконечен — 10-20 standby на одного primary уже создают нагрузку. Что если нужно 50 реплик?

**Cascading replication** — standby может сам стать источником для других standby'ев. Строится дерево: primary → standby1 → standby2 → standby3, где стрелка означает «поставщик WAL». Standby1 получает WAL с primary, дальше сам стримит его на standby2 и так далее.

Настраивается идентично обычному standby, только в `primary_conninfo` указывается адрес не primary, а промежуточной standby.

Полезно для географически распределённых деплойментов. Одна standby в удалённом датацентре — источник для всех локальных реплик там же. Между датацентрами один поток вместо N.

Ограничение: cascading standby не может выполнять synchronous replication (потому что не имеет прямого соединения с primary). Только асинхронно.

## Что можно делать на standby

Standby в PostgreSQL работают в режиме **hot standby** (если включено, дефолт с давних версий). Это значит: они принимают read-only запросы. `SELECT` работает, `INSERT`/`UPDATE`/`DELETE`/DDL — ошибка «cannot execute in read-only transaction».

Это позволяет разгрузить primary — направлять аналитические запросы, отчёты, non-critical чтения на реплику. Приложение может использовать пул соединений с primary и другой пул с реплики, разнося нагрузку.

Но есть тонкости. Первая — replication lag. Даже при асинхронной репликации standby отстаёт на секунды, миллисекунды при синхронной. Если приложение делает UPDATE на primary и сразу читает с standby, оно может не увидеть свои изменения (read-after-write inconsistency). Классическая проблема. Fix: для read-after-write операций читать с primary, для чисто аналитических (не нужны последние данные) — с реплики.

Вторая — конфликты между запросом и репликацией. Реплика применяет WAL и одновременно обслуживает read-запросы. Иногда они конфликтуют: например, WAL удаляет старые версии tuple'ов (VACUUM), а read-запрос на standby всё ещё их читает. Решение — либо задержать applying WAL до конца запроса (тогда replication lag растёт), либо отменить запрос.

Настраивается через `max_standby_streaming_delay` — сколько ждать перед отменой запроса. По умолчанию 30 секунд. Если поставить `-1` — ждать бесконечно (реплика может сколь угодно отставать). Если 0 — отменять сразу.

Симптом конфликта: клиент видит `ERROR: canceling statement due to conflict with recovery`. Fix: `hot_standby_feedback = on` на standby — реплика будет сообщать primary о своих открытых транзакциях, primary не будет удалять tuple'ы, нужные реплике. Работает, но замедляет VACUUM на primary. Trade-off.

## Logical replication

Streaming replication — физическая: реплика — байт-к-байту копия primary. Это отлично для HA и read scaling, но имеет ограничения:

- Обе стороны — одна версия PostgreSQL (можно с разными minor версиями, но не major).
- Реплицируется вся база данных, нельзя выборочно.
- Реплика read-only.
- Нельзя менять структуру данных при передаче (например, добавить колонку только на реплике).

Для случаев, где нужна гибкость, есть **logical replication**. Она работает на уровне отдельных изменений строк (INSERT/UPDATE/DELETE), а не байтов страниц. Реплика не обязана быть идентичной primary.

Механика. На publisher (primary в терминах logical replication) объявляется **publication** — какие таблицы публиковать:

```sql
CREATE PUBLICATION my_pub FOR TABLE orders, users;
```

На subscriber (destination) создаётся **subscription**:

```sql
CREATE SUBSCRIPTION my_sub 
CONNECTION 'host=primary port=5432 dbname=knp user=replicator' 
PUBLICATION my_pub;
```

При создании subscription делается initial data copy (все существующие данные копируются на subscriber), потом начинается continuous replication.

Механика continuous replication: на publisher process **wal sender** (тот же что для streaming) декодирует WAL через **logical decoding plugin** (обычно `pgoutput` — встроенный) в поток логических событий: «INSERT в table X values (Y, Z)», «UPDATE table X SET col=A WHERE key=B». Эти события шлются subscriber'у, который применяет их к своим таблицам как обычные SQL операции.

Плюсы logical replication:
- Между разными major версиями PostgreSQL (14 → 16, например). Единственный вменяемый способ upgrade без даунтайма.
- Между разными платформами (Linux → Windows).
- Selective replication — только определённые таблицы.
- Filter'ы (в PG 15+ можно replicate только определённые строки).
- Subscriber writable — можно делать локальные изменения, дополнительные индексы, дополнительные таблицы.
- Cross-database (в одном кластере replicate между databases).

Минусы:
- Медленнее streaming (декодирование + отдельная транзакция на apply).
- Не реплицирует schema changes — надо руками делать DDL и на publisher, и на subscriber.
- Не реплицирует sequence advances корректно во всех случаях.
- Не подходит для large writes (bulk INSERT) — производительность падает.
- До PG 16 — одна transaction reception + apply thread (медленно). PG 16+ — parallel apply, лучше.

Practical use cases для logical replication:

**Zero-downtime major upgrade**. Есть prod на PG 12, надо на PG 16. Ставишь PG 16 на новую машину, делаешь его subscriber от текущего prod (publisher). Ждёшь пока догонится. Приложение переключаешь на новый — всё, ты на PG 16.

**Multi-region replication**. Каждый регион — свой primary, часть данных реплицируется между ними (общие справочники, master data).

**Data warehousing**. OLTP-база пушит изменения в аналитическую (например, ClickHouse через специальный logical replication плагин).

**Migration to different platforms**. С AWS RDS → GCP Cloud SQL или наоборот. Logical replication работает через сеть, не требует access к файловой системе.

## HA и Patroni

Streaming replication даёт данные на нескольких машинах. Но что происходит если primary упал? Нужен способ автоматически переключить приложение на одну из реплик, promote'ить её в новый primary, remaining реплики переключить на нового primary. Всё это называется **failover** и требует специализированного инструментария.

Самый распространённый инструмент — **Patroni** (open-source, Python-based, разработан Zalando). De facto стандарт для HA PostgreSQL. Понимает механику PostgreSQL, streaming replication, replication slots, timeline switches. Работает вместе с distributed consensus system — обычно **etcd** или **Consul** (Zookeeper тоже поддерживается).

Архитектура. На каждом узле PostgreSQL кластера работает Patroni-агент. Он мониторит локальную БД, репортит статус в distributed store (etcd/Consul), читает оттуда решения. Etcd/Consul выступает как «книга правды» — кто сейчас primary, какие узлы в кластере, чей leader lock. Все Patroni-агенты видят одну и ту же информацию.

Один агент в любой момент времени владеет **leader lock** — это ключ в etcd/Consul с TTL. Владелец lock'а — Patroni, работающий на текущем primary. Он периодически (каждые несколько секунд) обновляет lock. Если primary упал и Patroni не может обновить — lock истекает через TTL.

Остальные Patroni'ы это видят: lock истёк, нужны выборы нового primary. Проверяют локальный statuс standby'ев (кто ближе всего к последнему LSN primary'а), договариваются через etcd (только один может взять lock), выбирают победителя. Этот Patroni делает `pg_promote` на своей standby — она становится новым primary. Обновляет lock, теперь принадлежит ему.

Остальные standby'и видят: primary изменился. Они reconnect'ятся к новому primary, продолжают репликацию. Failover занимает секунды.

Есть тонкий момент — что если старый primary «оживёт» через минуту? Он думает, что он primary (никто его не остановил). Одновременно новый primary тоже работает. Оба принимают writes — **split brain**, данные расходятся, потом их не свести. Катастрофа.

Защита — **fencing**. Патрини перед promote'ом убеждается, что старый primary точно не может писать. Способы разные:

- **STONITH** (Shoot The Other Node In The Head) — принудительное выключение старого primary через IPMI, cloud API (AWS/GCP выключение VM), или физический power cut. Гарантированно, но требует API доступа к железу.

- **Watchdog** — на каждом узле работает watchdog-таймер (программный или hardware). Если Patroni не пингует его каждые несколько секунд, watchdog принудительно перезагружает узел. Хороший подход, если конфиг верный.

- **Advisory locks в etcd** — старый primary при потере leader lock'а должен сам себя понизить до read-only. Patroni имеет механизм: если он потерял лид, он останавливает postgres на своей машине. Это надёжно если Patroni работает, но что если он висит?

В production обычно комбинация: Patroni понижает себя + watchdog как страховка + fencing через cloud API.

Приложение подключается к базе не по прямому адресу конкретной машины, а через **виртуальный IP** или **DNS**, который переключается вместе с primary. Обычно через HAProxy или PgBouncer перед Patroni-кластером. HAProxy опрашивает Patroni endpoints через REST API — узнаёт кто сейчас primary, направляет соединения туда. При failover'е обновляется автоматически.

## Patroni настройки на примере

Конфиг Patroni (patroni.yml) — большой, но ключевые части:

```yaml
scope: knp-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.0.1:8008

etcd3:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30                       # leader lock TTL секунд
    loop_wait: 10                 # как часто Patroni просыпается
    retry_timeout: 10             # таймауты на etcd операции
    maximum_lag_on_failover: 1048576  # 1MB - не failover если standby отстал больше
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 200
        shared_buffers: 8GB
        max_wal_senders: 10
        wal_keep_size: 2GB

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.0.1:5432
  data_dir: /var/lib/postgresql/data
  authentication:
    replication:
      username: replicator
      password: xxx
```

`ttl` и `loop_wait` — компромисс. Меньше значения — быстрее failover, но выше риск false-positive failover при временных сетевых проблемах. Больше — надёжнее, но медленнее failover. Обычно ttl=30, loop_wait=10 — failover 15-30 секунд.

`maximum_lag_on_failover` — если у candidate standby lag больше этого, Patroni не будет её promote (потеряется много данных). Компромисс между availability и durability.

`use_pg_rewind` — при переключении старого primary в standby после failover'а `pg_rewind` синхронизирует его без полной ре-инициализации. Быстрее чем pg_basebackup, но требует `wal_log_hints = on` или checksums.

## Мониторинг репликации

Что смотреть в проде.

Replication lag — насколько standby отстаёт от primary. Три метрики:

```sql
SELECT client_addr, state, 
       sent_lsn, write_lsn, flush_lsn, replay_lsn,
       pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS sent_lag,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag
FROM pg_stat_replication;
```

- `sent_lsn` — что уже отправлено standby.
- `write_lsn` — что standby записал в свой WAL.
- `flush_lsn` — что standby зафлашнил fsync'ом.
- `replay_lsn` — что уже применено к data files.

`sent_lag` в байтах — обычно очень маленький (сеть быстрая). `replay_lag` может быть больше — apply медленнее чем receive при большой нагрузке.

Более понятная метрика — время:

```sql
SELECT client_addr, replay_lag, write_lag, flush_lag
FROM pg_stat_replication;
```

`replay_lag` в интервале — «стандby отстаёт по времени на X». Например, 500ms — значит если primary умрёт сейчас, стандby переваривает WAL до момента 500ms назад.

Alerting: если replay_lag больше нескольких секунд систематически — что-то не так, либо стандby перегружена, либо большие транзакции на primary.

Второе — размер удерживаемого WAL:

```sql
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

Если retained_wal растёт и растёт — где-то slot забыт или standby не догоняет. Может привести к переполнению диска на primary.

Третье — synchronous состояние (если используется):

```sql
SELECT application_name, sync_state, sync_priority
FROM pg_stat_replication;
```

`sync_state` — `async` (обычный), `sync` (текущий синхронный), `potential` (может стать синхронным если текущий упадёт), `quorum`. Если ожидается синхронный, а показывает async — что-то не так с настройкой.

## Failover в реальной жизни

Что реально происходит при failover'е.

Момент T=0: primary внезапно недоступен. Может умер сервер, порвалась сеть, ядро паникнуло, всё что угодно.

T=0 до T=10 сек: Patroni на других узлах продолжает попытки достучаться до primary. Etcd/Consul видят, что primary не обновляет leader lock. Standby'и продолжают work, но перестают получать новые WAL.

T=10-30 сек: leader lock в etcd истёк. Patroni'и на standby'ях видят это, начинают elections. Каждый Patroni сообщает свой replay_lsn, договариваются — кто наиболее up-to-date, тот становится кандидатом. Обычно тот, кто был synchronous standby или физически ближе.

T=30-45 сек: выбранный кандидат делает `pg_promote`. Postgres становится writable. Patroni берёт leader lock в etcd. HAProxy опрашивает Patroni REST API, видит новый primary, переключает direction соединений. Остальные стандby'и получают signal reconnect, начинают стриминг с нового primary. Приложение продолжает работу — но соединения теперь идут в новый primary.

За failover теряется, в асинхронном режиме, всё что было в WAL primary'а но не долетело до standby'ев. Обычно это последние несколько миллисекунд commit'ов. В synchronous режиме — ничего.

Что если старый primary очнётся? С включенным fencing — он окажется убит (STONITH сработает), Patroni при возвращении определит его как «отстал», делает `pg_rewind` для синхронизации, регистрирует как standby. Приложение перенастраивается автоматически.

Failover занимает 30-60 секунд typically. Это заметный downtime для end users. Приложение должно уметь переживать: retry на connection errors, короткие timeouts на JDBC, exponential backoff в client'ах. Обычно за минуту все запросы восстанавливаются.

## Practical trade-offs

Как выбирать между режимами.

**Simple async streaming replication** без Patroni. Для dev, staging, некритичных прод. Нет автофейловера — приходится вручную промоутить standby при отказе primary. Плюс: просто настроить, минимум moving parts. Минус: downtime при отказе — минуты-часы (пока не заметят и не отреагируют).

**Async streaming + Patroni**. Стандарт для production. 30-60 секунд failover, приложение переживает. Некоторая потеря данных возможна (последние миллисекунды commit'ов). Для 90% приложений — правильный выбор.

**Sync streaming + Patroni**. Для критичных систем (финансы, healthcare, регуляторные). Zero data loss guarantee. Ценой — 1-3ms latency на каждый commit (round-trip до standby). При отказе standby'я — writes останавливаются пока не найдётся другой sync standby. Требует минимум 2 standby'ев, чтобы одна могла упасть без остановки primary.

**Quorum-based sync (ANY N)**. Middle ground. Ждём подтверждения от любых N из M standby'ев. Даёт zero data loss + tolerance к отказу одиночного standby.

**Logical replication** для специальных случаев: cross-version, cross-platform, selective replication, migration paths.

## Заключение

Репликация — фундамент отказоустойчивости PostgreSQL. Streaming replication — физическая, на уровне WAL, стандартный механизм для HA и read scaling. Реплики читают WAL от primary через wal sender/receiver и применяют к идентичной копии данных. Replication slots предотвращают потерю WAL если реплика отстала, но требуют мониторинга (забытый slot заполняет диск). Synchronous vs asynchronous — trade-off между latency коммитов и durability guarantee. Cascading позволяет масштабировать количество реплик.

Что можно делать на standby — только read. Hot standby принимает SELECT'ы, но с оговорками: replication lag может быть заметен (read-after-write не гарантирован), conflict resolution через `max_standby_streaming_delay` или `hot_standby_feedback`. Направление аналитической нагрузки на реплики — стандартный способ разгрузить primary.

Logical replication — вторая, гибкая форма. Работает на уровне row changes, не байтов. Между разными major versions, разными платформами, selective таблицы, subscriber writable. Медленнее streaming, не реплицирует DDL, но незаменима для zero-downtime upgrades и cross-system integration.

HA (автоматический failover) — не встроен в PostgreSQL, требует external tools. Patroni — de facto стандарт: агенты на каждом узле, координация через etcd/Consul, автоматический promote при отказе primary, integration с HAProxy для транспарентного переключения приложений. Split-brain prevention через fencing (STONITH или watchdog). Failover 30-60 секунд.

Мониторинг — replay_lag (насколько standby отстаёт), retained_wal (не забыт ли slot), synchronous state (работает ли sync репликация как ожидается). Alerting на любое отклонение.

Практический выбор: async streaming + Patroni для 90% случаев, sync или quorum sync для finance/healthcare, logical replication для миграций и специальных сценариев. В КНП должен быть настроен sync или quorum sync хотя бы на две standby в разных стойках, plus DR-replica в другом датацентре async — стандартная enterprise-конфигурация.

Дальше — практика. Разверни двухузловой кластер локально через Docker (primary + standby), настрой streaming replication руками, симулируй отказ primary, promote standby. Установи Patroni + etcd, увидь как автоматизация работает. Каждый шаг даёт больше понимания, чем чтение доков.
