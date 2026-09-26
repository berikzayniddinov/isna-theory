# 78. Прод-диагностика: kubectl workflow, thread dumps, OOMKilled, crash-loop

## Зачем это знать

Инцидент в проде — это гонка со временем. Пользователи получают 5xx, бизнес считает потери, команда в чате ждёт ответа «что происходит и когда починим». В эти первые 5-10 минут работает или не работает натренированный алгоритм: где смотреть, что читать, в каком порядке. Без алгоритма — паника, случайные `kubectl` вслепую, шумные подсказки в чате, тратятся часы.

С алгоритмом 80% инцидентов диагностируются за 5-15 минут. Не потому что типовые (они разные), а потому что первые шаги одинаковые: понять статус pod'а, увидеть Events, прочитать логи предыдущего инстанса, глянуть события кластера. Остальные 20% — глубокие: thread dump, heap dump, `pg_stat_activity`, метрики за последние часы, разбор корреляции по логам. Тоже алгоритмичны, просто чуть длиннее.

Этот файл — практический workflow, собранный из реальных инцидентов на КНП: sync crash-loop от отсутствующего Secret, gateway rollout timeout, liquibase checksum mismatch после отредактированного changelog, OOM tax-rep-report из-за неограниченного кэша PDF-шаблонов. По каждому — как отличить от других, что смотреть, как чинить. Плюс инструменты, которые надо знать: `kubectl describe`, `kubectl logs --previous`, thread dumps через jcmd, heap dumps через MAT, `pg_stat_activity` и `pg_blocking_pids`, Boot Actuator для быстрого поворота log level на живую. И анти-паттерны — что **не** делать в проде, чтобы инцидент не превратился в катастрофу.

## Первые пять шагов на любом алерте

Есть алгоритм, который работает в 80% случаев, вне зависимости от природы инцидента. Пять шагов, в жёстком порядке. Каждый следующий отвечает на вопрос, который поднял предыдущий.

Шаг первый — увидеть pod'ы: `kubectl get pods -n <ns> -o wide`. Кто Running, кто Pending, у кого рестарты, где какие возрасты. Флаг `-o wide` добавляет колонку с IP pod'а и именем ноды — полезно если проблема на конкретной ноде.

Шаг второй — вытянуть детали по проблемному pod'у: `kubectl describe pod <name> -n <ns>`. Смотреть три раздела снизу вверх — Events, Last State, Containers.

Шаг третий — прочитать логи упавшего инстанса: `kubectl logs <name> -n <ns> --previous --tail=200`. Флаг `--previous` — золото, показывает логи того контейнера, который упал, а не текущего попытавшегося запуск.

Шаг четвёртый — контекст namespace'а: `kubectl get events -n <ns> --sort-by='.lastTimestamp' | tail -30`. События уровня выше pod'а — schedule, PVC, network, controller manager.

Шаг пятый — воспроизвести локально или в staging **только если критично**. Быстрый fix прод → расследование позже. Затягивать инцидент ради «сначала понять до конца» — против бизнеса.

Дальше разберём каждый шаг подробно с типовыми сигналами.

## kubectl get pods: чтение статусов

Вывод состоит из колонок READY, STATUS, RESTARTS, AGE — за каждой стоит своя семантика.

**READY** — соотношение containers-Ready / containers-total. `1/1` — единственный контейнер прошёл readiness. `0/1` — либо readiness fails, либо контейнер не стартовал вообще. Для multi-container pod'ов `2/3` означает что один из трёх не Ready — его надо искать через describe.

**STATUS** — состояние pod'а в целом:

- `Running` — pod живёт, все контейнеры запущены.
- `Pending` — pod ещё не запущен. Причины: `Insufficient CPU/memory` на всех подходящих нодах, taints без соответствующих tolerations, PVC ещё не смонтирован (`ContainerCreating` на самом деле состоит из фаз, включая pull image + volume mount).
- `ContainerCreating` — kubelet работает: качает образ, монтирует тома, готовит cgroups. Длится обычно секунды-минуты; если висит десятками минут — искать причину в events (image pull failed, PVC issue).
- `CrashLoopBackOff` — контейнер стартовал, упал, kubelet ждёт с увеличивающимся backoff (10s, 20s, 40s, 80s, ..., cap 5min) перед следующим restart. Классика для application-level ошибок.
- `ImagePullBackOff` или `ErrImagePull` — не удалось скачать образ. Причины: неверный tag, нет прав в registry (imagePullSecret не настроен или битый), registry unreachable.
- `OOMKilled` — контейнер убит cgroup OOM killer. Технически это состояние lastState, но `kubectl get pods` иногда показывает как short-lived reason.
- `Error`, `Completed` — терминальные для Job'ов. Error = exit != 0, Completed = exit 0.
- `Terminating` — pod удаляется. Нормально длится до `terminationGracePeriodSeconds`. Если висит дольше — застрял в удалении (обычно из-за не завершившегося finalizer'а).

**RESTARTS** — сколько раз контейнер перезапускался. `17 (2m ago)` — 17 рестартов, последний 2 минуты назад. Активный crash-loop. `0` на pod'е возрастом день — стабильно работает.

**AGE** — сколько существует. Свежий pod с 0 рестартов после deploy — обычно ok, но `describe` подтвердит.

Ключевой навык — по первому взгляду на `kubectl get pods` понимать: массовая проблема (много pod'ов не Ready, cluster-level инцидент) или локальная (один pod дурит, application-level).

## kubectl describe pod: 90% ответов здесь

Describe — самая информативная команда, выдаёт целую страницу текста. Читать надо сверху вниз, но начинать с конца.

**Events в самом низу**:

```
Events:
  Type     Reason     Age                     From     Message
  Normal   Scheduled  20m                     ...      Successfully assigned knp/isnaknpsync-abc-xyz
  Normal   Pulling    20m                     kubelet  Pulling image "registry.1sc.kz/isnaknpsync:ac01514b"
  Normal   Pulled     20m                     kubelet  Image pulled
  Normal   Created    20m (x3 over 3h)        kubelet  Created container isnaknpsync
  Normal   Started    20m (x3 over 3h)        kubelet  Started container isnaknpsync
  Warning  BackOff    2m50s (x1693 over 3h)   kubelet  Back-off restarting failed container
```

Свежие события внизу. `BackOff x1693 over 3h` — 1693 попытки за 3 часа. Сильный crash-loop. `Created (x3 over 3h)` — контейнер создавали 3 раза за 3 часа, значит между попытками был большой backoff, каждая попытка длилась минуты.

**Last State и State посередине**:

```
State:          Waiting
  Reason:       CrashLoopBackOff
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
  Started:      Fri, 12 Sep 2026 10:15:00 +0500
  Finished:     Fri, 12 Sep 2026 10:15:12 +0500
```

Exit code — важная подсказка о причине падения. **0** — чистый выход, программа завершилась сама. **1** — общая application ошибка (например, `throw new RuntimeException` в main). **137** = 128 + 9 (SIGKILL). Обычно OOMKilled (cgroup убил) или terminationGracePeriod истёк. **139** = 128 + 11 (SIGSEGV) — native crash, редко, обычно JNI-код. **143** = 128 + 15 (SIGTERM) — штатный shutdown, если exit 143 без OOM — Boot корректно принял сигнал и вышел.

Если Started и Finished рядом (12 секунд) — приложение падает при старте, не успевает даже прогреться. Смотреть логи `--previous`.

**Containers сверху** — образ, ресурсы, environment:

```
Containers:
  isnaknpsync:
    Image:      registry.1sc.kz/isnaknpsync:ac01514b
    Ports:      8080/TCP
    Requests:
      cpu:      500m
      memory:   1Gi
    Limits:
      cpu:      2
      memory:   2Gi
    Environment:
      SPRING_PROFILES_ACTIVE:  prod
      DB_HOST:                 postgres-primary
      ...
```

Проверять: тот ли Image tag (мог задеплоиться не тот SHA), все ли Environment на месте (пустой ключ = не задан), не задушен ли по ресурсам (Requests/Limits ниже реальных потребностей).

**Conditions**:

```
Conditions:
  Type              Status
  Initialized       True
  Ready             False    ← вот проблема
  ContainersReady   False
  PodScheduled      True
```

`Ready: False` = readiness probe не проходит = pod не в endpoints = трафик не идёт. Дальше смотреть в logs — почему readiness падает.

## kubectl logs: фактическая ошибка приложения

Логи — единственный источник, где видно что реально происходит внутри JVM. `kubectl logs <pod>` показывает stdout/stderr **текущего** живого контейнера. Если pod в CrashLoopBackOff — контейнер сейчас лежит, логи покажут только последнюю попытку.

Ключевой флаг — **`--previous`**. Показывает логи предыдущего инстанса контейнера, того самого который упал. Без него разбор crash-loop невозможен — актуальные логи будут пустые или обрезанные до момента падения.

```bash
kubectl logs isnaknpsync-abc-xyz -n knp --previous --tail=400
```

Что искать в логах:

Первым делом — `ERROR` и `Exception` в конце (перед падением). Не самая первая ошибка, а последняя перед выходом — обычно она и есть причина. Дальше — `Caused by:` в цепочке — root cause. Spring обёрнутый в свои исключения (BeanCreationException, UnsatisfiedDependencyException) — надо докопать до реального `Caused by`, там будет содержательная причина.

Пример из реального sync crash-loop:

```
ERROR o.s.boot.SpringApplication : Application run failed
org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean 
  with name 'encryptionBackfillJob' ...
Caused by: java.lang.IllegalStateException: 
  phys_person_capacity_status_history_hash backfill enabled without ENCRYPTION_BLIND_INDEX_KEY
```

Причина сразу ясна: не задан `ENCRYPTION_BLIND_INDEX_KEY` в environment. Идти в Deployment/Secret, добавлять. Разбор занял секунды после того как открыл логи с `--previous`.

Полезные флаги logs, за которые новички забывают:

- `-f` (`--follow`) — стрим live как `tail -f`.
- `--since=10m` — только за последние 10 минут (очень полезно на нагруженных сервисах).
- `--since-time=2026-09-12T10:00:00Z` — с точной timestamp.
- `-c <container>` — если pod multi-container.
- `--all-containers` — все контейнеры pod'а сразу (init + main + sidecars).
- `--timestamps` — добавить timestamp к каждой строке (нужно когда логи сами не пишут время).

## kubectl get events: контекст кластера

События — то, что происходило в namespace на уровне выше pod'ов. Schedule решения, pull образов, восстановление PVC, изменения Deployment. Часто причина того что pod не стартует — не в самом pod'е, а в scheduling или volume.

```bash
kubectl get events -n knp --sort-by='.lastTimestamp' | tail -30
```

Типичные полезные события:

```
Warning  FailedScheduling  pod/isnaknpuser-xyz  
  0/5 nodes are available: 1 Insufficient memory, 
  4 node(s) had untolerated taint {dedicated: infra}
```

Scheduler не нашёл ноду. Один узел с достаточно памяти есть, но там taint без соответствующего toleration; на остальных нодах памяти не хватает. Fix: либо освободить память (может кто-то раздутый живёт), либо расширить кластер, либо добавить toleration.

```
Warning  FailedMount  pod/postgres-0  
  Unable to attach or mount volumes: timed out waiting for the condition; 
  unattached volumes=[data]
```

PVC не монтируется. Storage provisioner не смог создать/присоединить том. Причины: проблема с CSI-драйвером, том занят другим pod'ом (RWO конфликт), исчерпан лимит EBS attach на инстансе.

```
Warning  Unhealthy  pod/isnaknpuser-xyz  
  Readiness probe failed: HTTP probe failed with statuscode: 503
```

Readiness пробу kubelet бьёт, приложение возвращает 503. Смотреть логи приложения — почему health/readiness падает (обычно БД недоступна, зависимость не поднялась).

## Типовые сценарии

Каждый сценарий имеет свою характерную сигнатуру и своё лечение.

### CrashLoopBackOff с exit=1

Приложение падает при старте. Логи через `--previous`:

- **ClassNotFoundException / NoSuchMethodError** — конфликт версий зависимостей. Проверять `./gradlew dependencies` на конфликтующие версии, часто причина в том что fat jar собрался с не той версией библиотеки.
- **BeanCreationException** — Spring не может создать бин. Копать глубже в `Caused by`: обычно missing env variable, отсутствующий бин зависимости, circular dependency.
- **PSQLException: Connection to db... refused** — БД недоступна на этапе создания DataSource. Проверить `SPRING_DATASOURCE_URL`, network policies, живёт ли БД (`kubectl get pods -l app=postgres`).
- **liquibase.exception.ValidationFailedException: checksum** — типичная история: изменили уже применённый changeset. Fix — откатить файл к оригиналу или запустить `liquibase clearCheckSums` (осторожно, только если понимаешь последствия).
- **IllegalStateException при @PostConstruct** — обычно проверка env variable в bean init. Смотреть `describe pod → Environment`.

### OOMKilled

Признак: `Reason: OOMKilled` в Last State, exit 137. Диагностика:

Первое — live memory usage: `kubectl top pod X -n knp`. Показывает сколько ест сейчас (уже после рестарта, свежая копия). Если сразу близко к limit — приложение растёт быстро. Если мало — рост постепенный, ищем memory leak.

Второе — JVM view: `kubectl exec X -- jstat -gc 1 1s 10`. Показывает состояние heap регионов раз в секунду 10 секунд. Смотреть S0U/S1U (Survivor), EU (Eden), OU (Old use). Если Old близко к максимуму и растёт после каждого GC — утечка. Если Old стабильный, но Eden часто заполняется — просто высокая аллокация, может быть нормально.

Третье — heap dump:

```bash
kubectl exec X -- jcmd 1 GC.heap_dump /tmp/heap.hprof
kubectl cp X:/tmp/heap.hprof ./heap.hprof
```

Открывать в Eclipse Memory Analyzer (MAT). Dominator Tree — топ по retained size. Path to GC Roots — почему объект не собирается. Классика утечек: HashMap-кэш без TTL, ThreadLocal без cleanup, статические поля с ростом, EhCache/Caffeine с неограниченной ёмкостью, JDBC statements без close.

Четвёртое — проверить JVM options: `kubectl exec X -- jinfo -flags 1 | grep -E "MaxHeapSize|Xmx"`. Убедиться что `-Xmx` меньше container `limits.memory` минимум на 512Mi (metaspace + direct memory + thread stacks + JIT code cache). Классика ошибки: `limits.memory: 2Gi`, `-Xmx2G` — heap сжирает всё, cgroup убивает при первой попытке взять metaspace.

Полезные JVM опции: `-XX:+ExitOnOutOfMemoryError` — выйти сразу при OOM в heap, вместо попыток продолжить с полумёртвой памятью. `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/oom.hprof` — писать дамп автоматически при OOM (не забыть shared volume, иначе после рестарта потеряется).

### ImagePullBackOff

```
Failed to pull image "registry.1sc.kz/isnaknpsync:abc": rpc error: 
  code = Unknown desc = failed to pull and unpack image: 
  failed to resolve reference: not found
```

Возможные причины:

- **Tag не существует в registry** — сборка не прошла или CI указал не тот tag. Проверить: `docker manifest inspect registry.1sc.kz/isnaknpsync:abc` (если есть доступ).
- **Нет прав в registry** — imagePullSecret не настроен или невалидный. `kubectl get pod X -o yaml | grep -A2 imagePullSecrets` — есть ли секрет. `kubectl get secret <secret-name> -o yaml` — что внутри (обычно docker-config JSON).
- **Registry unreachable** — network policy блокирует, DNS не резолвит, registry реально лежит.

### Deployment rollout зависает

```bash
kubectl rollout status deployment/X -n knp
```

Просто висит, ждёт. Через `--timeout=5m` вылетает с ошибкой.

Что делать: посмотреть какие pod'ы образовались:

```bash
kubectl get pods -n knp -l app=X --sort-by=.metadata.creationTimestamp
```

Новые pod'ы (свежий hash в имени) внизу. Если они не Ready — они и являются причиной. Смотреть их `describe` и `logs`. Обычно причина — readiness fails по новой конфигурации, миссинг env, сломанная зависимость.

Если rollout зафейлился и надо срочно вернуть работу:

```bash
kubectl rollout undo deployment/X -n knp
```

Откатывает на предыдущую версию. Если хочешь на конкретную ревизию:

```bash
kubectl rollout history deployment/X -n knp
kubectl rollout undo deployment/X --to-revision=3
```

### Приложение отвечает медленно, 5xx растут

Более сложный случай — pod'ы Running, ready, но качество работы деградировало. Работать нужно не только с kubectl, но с метриками, логами и внутренностями JVM.

Первое — метрики Prometheus/Grafana. Стандартный набор для JVM+Spring Boot:

- `rate(http_server_requests_seconds_count{status=~"5..",app="X"}[5m])` — темп 5xx. Где именно растёт (какой endpoint).
- `histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le))` — p99 latency.
- `jvm_memory_used_bytes / jvm_memory_max_bytes` по регионам — heap growth.
- `hikaricp_connections_active`, `hikaricp_connections_pending` — пул к БД (см. файл 107).
- `process_cpu_usage` — реальный CPU utilization JVM.

Второе — логи по correlation_id, если проблема с конкретными запросами:

```
{app="isnaknpuser"} |= "ERROR" | json | correlation_id="..."
```

Grafana Loki / Kibana. Correlation ID должен пробрасываться через MDC в SLF4J — это стандартная практика для микросервисов.

Третье — thread dump, если приложение висит или тормозит:

```bash
kubectl exec pod-x -- jcmd 1 Thread.print > threads.txt
```

Читать (подробнее ниже): много ли тредов на одной строке, blocked на synchronized, wait на HikariPool, wait на socket read. Загрузить в fastthread.io — визуализация групп тредов по состоянию/стеку.

Четвёртое — БД, если по метрикам HikariCP пул проседает:

```sql
SELECT pid, state, wait_event_type, wait_event, 
       now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle' AND now() - query_start > interval '1 minute'
ORDER BY duration DESC;
```

Длинные активные запросы, ожидания блокировок (`wait_event_type = 'Lock'`), диск (`DataFileRead`), клиент (`ClientRead` — приложение держит транзакцию открытой без активности).

## Thread dump: как читать

Thread dump в JVM снимается за миллисекунды и не аффектит приложение. Может быть жизненно важен для расследования зависших/медленных ситуаций.

Снятие:

```bash
kubectl exec pod-x -- jcmd 1 Thread.print > threads.txt
```

Альтернатива — `jstack 1` (простая версия) или через Actuator: `curl localhost:8080/actuator/threaddump`.

Формат для каждого треда:

```
"http-nio-8080-exec-42" #123 daemon prio=5 os_prio=0 tid=0x... nid=0x... waiting for monitor entry
   java.lang.Thread.State: BLOCKED (on object monitor)
	at kz.example.Service.doWork(Service.java:42)
	- waiting to lock <0x00000007f8a0cd28> (a java.lang.Object)
	at kz.example.Controller.endpoint(Controller.java:15)
```

Ключевые состояния:

- **RUNNABLE** — тред выполняется или готов выполняться. Может быть в userspace или в native syscall (blocking I/O). Топ фрейма покажет что делает — если `SocketDispatcher.read0` — ждёт сеть (не CPU!).
- **BLOCKED (on object monitor)** — тред ждёт `synchronized` монитор, кто-то другой держит.
- **WAITING** и **TIMED_WAITING** — ждёт условие (`Object.wait()`, `Thread.sleep()`, `LockSupport.park()`). У park будет фрейм `jdk.internal.misc.Unsafe.park`.

Что искать:

**Много тредов BLOCKED на одном lock**. Ищи одинаковый идентификатор `<0x...>` в разных тредах:

```
"exec-42" BLOCKED
  - waiting to lock <0x00000007f8a0cd28>
"exec-43" BLOCKED
  - waiting to lock <0x00000007f8a0cd28>
"exec-50" RUNNABLE
  - locked <0x00000007f8a0cd28>
    at kz.example.LegacyService.slowMethod(LegacyService.java:100)
```

Причина: `synchronized` бутылочное горлышко. `exec-50` держит монитор внутри slowMethod, остальные ждут. Fix — либо переписать без глобального synchronized (использовать ConcurrentHashMap вместо HashMap+synchronized, ReentrantLock с fine-grained locking), либо ускорить slowMethod.

**Тред застрял в HikariPool.getConnection**:

```
"exec-42" TIMED_WAITING
  at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:80)
```

Пул исчерпан (см. файл 107). Либо реально мал, либо кто-то держит connection долго (транзакция с HTTP-вызовом внутри, забытый rollback).

**Тред в HTTP-клиенте, ожидает socket read**:

```
"exec-42" RUNNABLE
  at sun.nio.ch.SocketDispatcher.read0(Native Method)
  ...
  at org.apache.http.impl.io.SessionInputBufferImpl.streamRead(...)
  at ...
  at org.springframework.web.client.RestTemplate.doExecute(...)
```

Внешний сервис отвечает медленно. Проверять timeout настройки клиента (по умолчанию у RestTemplate без явных настроек — бесконечные timeouts, что плохо). Смотреть кому идёт вызов, реально ли тот сервис задыхается.

**Rabbit consumer в receive**:

```
"rabbit-consumer" WAITING (parking)
  at com.rabbitmq.client.impl.recovery.RecoveryAwareChannelN.basicGet
```

Норма. Тред-consumer припарковался в ожидании нового сообщения. Не путать с проблемой.

**Много тредов в pool executor**:

```
"pool-2-thread-15" WAITING (parking)
  at java.util.concurrent.locks.LockSupport.park
  at java.util.concurrent.LinkedBlockingQueue.take
  at java.util.concurrent.ThreadPoolExecutor.getTask
```

Норма. ThreadPoolExecutor держит idle тредов, они парятся на take() из queue пока не появится задача.

Инструменты для читабельности:

- **fastthread.io** — загружаешь dump, получаешь визуализацию: группировка по типу состояния, топ blocked locks, deadlocks. Бесплатно, часто хватает.
- **VisualVM** — offline анализ (File → Load → выбрать dump). Хорош для глубокого копания.
- **jstack** — простая CLI-версия, работает как `jcmd Thread.print`.

## PostgreSQL: pg_stat_activity, locks, slow queries

БД часто оказывается узким местом. Основные представления Postgres:

**pg_stat_activity** — что происходит прямо сейчас:

```sql
SELECT pid, usename, application_name, state, wait_event_type, wait_event,
       now() - xact_start AS xact_age,
       now() - query_start AS query_age,
       left(query, 100) AS q
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY xact_start;
```

Ключевые сигналы (см. также файл 88):

- `state = 'active'` + большой `query_age` — долгий запрос идёт.
- `state = 'idle in transaction'` + большой `xact_age` — приложение начало транзакцию и не закрывает. Локи держатся, connection pool забивается.
- `wait_event_type = 'Lock'` — ждёт блокировку. Кого — смотреть через `pg_blocking_pids`.
- `wait_event_type = 'IO', wait_event = 'DataFileRead'` — читает с диска. Диск задыхается или запрос читает много.
- `wait_event = 'ClientRead'` — база ждёт следующей команды от приложения. Признак что приложение занимается чем-то вне БД под транзакцией.

**Кто кого блокирует**:

```sql
SELECT blocked.pid AS blocked_pid,
       blocking.pid AS blocking_pid,
       blocked.query AS blocked_query,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking 
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';
```

`pg_blocking_pids(pid)` возвращает массив PID'ов, блокирующих данную сессию. Классическая цепочка расследования: находишь длинную транзакцию, идёшь по её `pg_blocking_pids` в глубину — обычно на дне цепочки одна старая транзакция, которую все ждут.

**Idle in transaction** — отдельный подход:

```sql
SELECT pid, state, now() - state_change AS idle_duration, 
       left(query, 100) AS last_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY idle_duration DESC;
```

Убить конкретную сессию:

```sql
SELECT pg_terminate_backend(pid);
```

Причина в коде обычно одна из двух: транзакционный метод не завершается (HTTP-вызов внутри @Transactional, ждущий 30 сек), либо exception в @Transactional был проглочен и rollback не сделан. Долгосрочное решение — `idle_in_transaction_session_timeout = 60s` в PostgreSQL, чтобы такие транзакции убивались автоматически.

**Медленные запросы через pg_stat_statements**:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT calls, mean_exec_time, total_exec_time, rows,
       left(query, 200) AS q
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

Топ запросов по суммарному времени — обычно оптимизация одного из них даёт больший эффект чем оптимизация десятка редких. Дальше — `EXPLAIN ANALYZE` каждого, добавление индексов (см. файл 88).

## Actuator endpoints: живая диагностика Spring Boot

Boot Actuator даёт HTTP-эндпоинты в живом pod'е. Часто быстрее чем ходить в JVM через jcmd. Port-forward:

```bash
kubectl port-forward pod/isnaknpuser-xyz -n knp 8081:8080
# в другом терминале
curl localhost:8081/actuator/health
curl localhost:8081/actuator/threaddump
curl localhost:8081/actuator/heapdump > heap.hprof
```

**`/actuator/health`** — общий статус (UP/DOWN) и детали компонентов (db, disk, redis, rabbit). Ключ к пониманию почему readiness падает.

**`/actuator/health/liveness`** и **`/actuator/health/readiness`** — то, что бьёт kubelet.

**`/actuator/threaddump`** — то же что jstack, но через HTTP. Формат JSON, можно парсить программно.

**`/actuator/heapdump`** — heap dump в hprof, скачивается сразу. Открывать в MAT.

**`/actuator/env`** — все прописанные env, propertysources, effective values. Огромный вывод (десятки KB), но можно фильтровать:

```bash
curl localhost:8081/actuator/env/spring.datasource.url
```

Возвращает конкретный ключ с указанием какой PropertySource его дал (application.yml, environment, command line). Полезно когда «почему у меня в приложении не тот URL к БД» — сразу видно кто перебивает.

**`/actuator/loggers`** — можно менять log level **на живую**, без рестарта:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"configuredLevel":"DEBUG"}' \
  localhost:8081/actuator/loggers/kz.example.MyService
```

Проблема воспроизводится редко, нужны debug-логи — включил на конкретный package, воспроизвёл, выключил обратно. Огромная экономия времени по сравнению с rebuild + redeploy ради DEBUG-логов.

Важное правило безопасности: `/actuator` эндпоинты **не выставляются наружу через Ingress**. Только localhost port-forward или отдельный internal service с NetworkPolicy. `/actuator/env` содержит секреты, `/actuator/heapdump` — все данные памяти. Выставлять наружу — прямой путь к утечке всех секретов приложения.

## Реальный инцидент: реконструкция

**Alert**: `deployment/isnaknpsync exceeded progress deadline`. Deploy зависает уже 40 минут после MR.

**Шаг 1**: 

```bash
kubectl get pods -n knp | grep sync
```

```
isnaknpsync-cc5bd64d5-ds5km   1/1  Running            0    46h
isnaknpsync-5d9f56c665-n2sbf  0/1  CrashLoopBackOff   75   6h
```

Старый pod живой (46 часов), новый не поднимается (75 рестартов за 6 часов). Классический rollout stuck.

**Шаг 2**: 

```bash
kubectl describe pod isnaknpsync-5d9f56c665-n2sbf -n knp
```

Events:
```
Pulling ... image "registry.1sc.kz/isnaknpsync:ac01514b"
Pulled ... image
Created container
Started container
Back-off restarting failed container
```

Образ скачан ок, контейнер стартовал и упал. Не image issue, не resource issue. Application-level падение.

**Шаг 3**: 

```bash
kubectl logs isnaknpsync-5d9f56c665-n2sbf -n knp --previous --tail=200
```

```
ERROR SpringApplication : Application run failed
Caused by: IllegalStateException: 
  phys_person_capacity_status_history_hash backfill enabled without ENCRYPTION_BLIND_INDEX_KEY
```

Причина найдена: env variable не задана. В коде — новая функциональность, требующая ключа шифрования, которого нет в проде.

**Шаг 4**: подтвердить в Secret:

```bash
kubectl get secret isna-secret -n knp -o yaml | grep -i ENCRYPTION
```

Ключа нет.

**Действие**: DevOps добавляет `ENCRYPTION_BLIND_INDEX_KEY` в Secret, `kubectl rollout restart deployment/isnaknpsync -n knp`. Через минуту новый pod Ready.

Итог: диагностика 5 минут. Ключевой момент — `--previous` в logs, без него ничего бы не увидели, потому что контейнер уже упал к моменту команды.

## Полезные bash-обёртки

**`k8s-diag.sh` — быстрый обзор одного pod'а**:

```bash
#!/bin/bash
POD=$1
NS=${2:-knp}
echo "=== DESCRIBE (tail) ==="
kubectl describe pod $POD -n $NS | tail -60
echo "=== LOGS (previous) ==="
kubectl logs $POD -n $NS --previous --tail=100 2>/dev/null || echo "no previous"
echo "=== LOGS (current) ==="
kubectl logs $POD -n $NS --tail=50
echo "=== POD EVENTS ==="
kubectl get events -n $NS --sort-by='.lastTimestamp' \
  --field-selector involvedObject.name=$POD | tail -10
```

Одна команда — всё что нужно для первого взгляда.

**Полезные alias'ы в `.bashrc`/`.zshrc`**:

```bash
alias k=kubectl
alias kdp='kubectl describe pod'
alias kgpo='kubectl get pods -o wide'
alias klf='kubectl logs -f'
alias klp='kubectl logs --previous --tail=200'
alias ktp='kubectl top pods --sort-by=memory'
```

Секунды экономятся, а на инциденте секунды складываются в минуты.

## Чего не делать в проде

**`kubectl edit` для быстрого fix'а конфигурации.** Изменения теряются при следующем deploy из CI. Правильно — менять в Helm/Kustomize/GitOps, катать через pipeline. Единственное исключение — экстренный workaround, но с обязательным follow-up тикетом «зафиксировать в git».

**`kubectl delete pod X --force --grace-period=0`** без понимания последствий. Force delete не даёт SIGTERM и graceful period, kubelet просто убивает и удаляет запись из API. Для БД или stateful сервиса — риск повреждения данных. Для application pod'а — потеря in-flight requests. Использовать только когда pod застрял в Terminating часами (обычно finalizer issue) и осознанно.

**`kubectl scale`** для quick fix. Так же не в git, откатится при следующем deploy. Правильно — менять в манифесте, катать через CI. Если срочно надо больше реплик — сначала `kubectl scale`, потом сразу коммит в git.

**Локально работающий `kubectl exec ... -- rm -rf ...`**. Классика падений: удалили «лишние» файлы у Postgres pod'а и получили corrupted database. Любые изменения в pod через exec — временные и опасные.

**`kubectl cp` больших файлов** (десятки MB и больше). Может забить network, timeout, оставить частично скопированные файлы. Для heap dumps лучше сначала архивировать (`gzip`) внутри pod'а, потом cp, или использовать S3/MinIO как intermediate storage.

**Апгрейд production namespace без backup.** Даже helm upgrade может неожиданно снести PVC при нестандартных манифестах. Правило — перед любым upgrade production снять backup БД и убедиться что PV/PVC не удалятся.

**Изменения через `kubectl patch` без записи в git.** То же правило — теряется при deploy.

## Чек-лист расследования

Стандартный алгоритм для любого прод-алерта:

1. `kubectl get pods` — состояние pod'ов, кто не Ready.
2. `kubectl describe pod` — Events, Last State, Reason, Conditions.
3. `kubectl logs --previous` — фактическая ошибка приложения.
4. `kubectl get events` — контекст namespace.
5. Метрики (Grafana) — тренды нагрузки, memory, CPU, latency за последние часы.
6. Логи по correlation_id — если проблема функциональная.
7. Thread dump — если приложение висит.
8. `pg_stat_activity` — если БД узкое место (см. также файлы 88, 107).
9. Actuator `/env`, `/health` — проверка конфигурации в живую.
10. Fix → deploy → мониторить метрики после.

Первые 5 минут инцидента — самое дорогое время. Держать этот чек-лист под рукой и тренироваться на не-инцидентах, чтобы в стрессовой ситуации руки шли по алгоритму автоматически.

## Заключение

Прод-диагностика — это набор навыков, а не одна волшебная команда. `kubectl get pods` даёт первый взгляд, `describe` — 90% ответов, `logs --previous` — фактическая причина падения в crash-loop. Events показывают контекст выше pod'а: scheduling, storage, network. Exit codes несут смысл (137 = OOMKilled/SIGKILL, 143 = SIGTERM/graceful, 1 = application error).

Типовые сценарии знакомы: CrashLoopBackOff с exit=1 идёт в logs; OOMKilled — в top + heap dump + анализ Xmx vs limits; ImagePullBackOff — в registry/secrets; deployment stuck — в pod'ы новой ReplicaSet, почему не Ready.

Thread dump снимается за миллисекунды и открывает внутренности JVM: BLOCKED на synchronized (bottleneck), WAITING в getConnection (пул исчерпан), RUNNABLE в SocketDispatcher.read0 (ждёт сеть, не CPU). fastthread.io ускоряет визуальный анализ.

PostgreSQL диагностируется через `pg_stat_activity` + `pg_blocking_pids`: находятся долгие запросы, ожидания блокировок, `idle in transaction`, `wait_event` для понимания природы (диск, сеть, лок). Медленные запросы через `pg_stat_statements`, дальше — `EXPLAIN ANALYZE` и индексы (файл 88).

Actuator в живом pod'е даёт `/health` для причины readiness fail, `/env` для проверки конфигурации, `/loggers` для включения DEBUG без рестарта, `/threaddump` и `/heapdump` через HTTP. Не выставлять наружу — секреты и полная память в открытом доступе.

Не делать в проде: `kubectl edit`/`scale` без git, `--force --grace-period=0` для БД, `kubectl exec ... -- rm -rf`, `kubectl cp` мегабайтами. Все изменения через CI/GitOps, экстренные обходы — с follow-up на записать в git.

Первые 5 минут инцидента — самые дорогие. Алгоритм из 10 шагов, наработанный на нескольких десятках инцидентов, превращает панику в методичную работу и сокращает MTTR в разы.
