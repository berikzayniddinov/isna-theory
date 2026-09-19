# 78. Прод-диагностика: kubectl workflow, thread dumps, OOMKilled, crash-loop

Практический workflow расследования прод-инцидентов в k8s+Spring Boot. Основано на реальных диагностиках КНП (sync crash-loop, gateway CI fails, liquibase checksum mismatch, deploy timeout).

---

## 1. Первое, что делаешь на алерте

Алгоритм из 5 шагов, работает 80% случаев:

```
1. kubectl get pods -n <ns> -o wide
2. kubectl describe pod <name> -n <ns>
3. kubectl logs <name> -n <ns> --previous --tail=200
4. kubectl get events -n <ns> --sort-by='.lastTimestamp' | tail -30
5. Reproduce локально или в staging (только если критично)
```

Разберём каждый шаг.

---

## 2. kubectl get pods — что смотреть

```
NAME                       READY   STATUS             RESTARTS       AGE
isnaknpsync-abc-xyz        0/1     CrashLoopBackOff   17 (2m ago)    3h
isnaknpuser-def-xyz        1/1     Running            0              7d
```

**READY 1/1** — pod живой и readinessProbe = OK.
**READY 0/1** — либо не готов (readiness fail), либо контейнер не стартовал.

**STATUS**:
- `Running` — pod живой.
- `Pending` — pod ещё не запущен: нет ресурсов (Insufficient CPU/memory), нет ноды с нужными selectors/taints, wait for PVC.
- `ContainerCreating` — образ качается, том монтируется.
- `CrashLoopBackOff` — контейнер запустился, но упал, k8s ждёт увеличивающийся интервал перед следующим restart (10s, 20s, 40s, ..., до 5min).
- `ImagePullBackOff` / `ErrImagePull` — не может скачать образ (нет прав в registry, image не существует, wrong tag).
- `OOMKilled` — контейнер убит cgroup OOM killer.
- `Error` / `Completed` — Job закончился.
- `Terminating` — pod в процессе удаления (или застрял в удалении если >60s).

**RESTARTS** — растёт → контейнер регулярно падает. `17 (2m ago)` = 17 рестартов, последний 2 минуты назад = активный crash-loop.

**AGE** — pod существует столько-то. Свежий pod с 0 рестартов после deploy — обычно ok.

---

## 3. kubectl describe pod — 90% ответов здесь

```bash
kubectl describe pod isnaknpsync-abc-xyz -n knp
```

Скроллим до конца:

```
Events:
  Type     Reason     Age                  From     Message
  ----     ------     ----                 ----     -------
  Normal   Scheduled  20m                  ...      Successfully assigned knp/isnaknpsync-abc-xyz
  Normal   Pulling    20m                  kubelet  Pulling image "registry.1sc.kz/isnaknpsync:ac01514b"
  Normal   Pulled     20m                  kubelet  Image pulled
  Normal   Created    20m (x3 over 3h)     kubelet  Created container isnaknpsync
  Normal   Started    20m (x3 over 3h)     kubelet  Started container isnaknpsync
  Warning  BackOff    2m50s (x1693 over 3h)  kubelet  Back-off restarting failed container
```

**Смотрим Events снизу вверх** — самый свежий внизу. `BackOff x1693` = 1693 попыток за 3 часа перезапустить — сильный crash-loop.

Выше в describe:
```
State:          Waiting
  Reason:       CrashLoopBackOff
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
  Started:      Fri, 12 Sep 2026 10:15:00 +0500
  Finished:     Fri, 12 Sep 2026 10:15:12 +0500
```

Exit Code:
- **0** — ok, программа завершилась чисто.
- **1** — общая ошибка приложения (`throw new RuntimeException`).
- **137** — SIGKILL. Обычно OOMKilled или terminationGracePeriod истёк.
- **139** — SIGSEGV. Native crash (rare).
- **143** — SIGTERM. Штатный shutdown (Boot graceful).

**Секции резюме сверху describe**:
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
      ...
```

Смотри **Image tag** (ту ли версию задеплоили), **Environment** (все ли ENV на месте, не пустые ли ключи), **Requests/Limits** (не задушен ли).

Смотри **Conditions**:
```
Conditions:
  Type              Status
  Initialized       True
  Ready             False    ← ноготок
  ContainersReady   False
  PodScheduled      True
```

Ready=False → readinessProbe fails → pod не в endpoints Service'а → трафик не идёт.

---

## 4. kubectl logs — фактическая ошибка приложения

```bash
kubectl logs isnaknpsync-abc-xyz -n knp --tail=200
```

Логи **живого** контейнера. Если pod в CrashLoopBackOff — контейнер сейчас не запущен, тут покажет только последние строки последнего запуска.

Для **предыдущего** container'а (упавшего):
```bash
kubectl logs isnaknpsync-abc-xyz -n knp --previous --tail=400
```

Это **золото** — оно покажет что было в моменте краха.

**Что искать**:
- `ERROR` и `Exception` — конкретные исключения.
- `Caused by:` — root cause в цепочке exception'ов.
- Стектрейсы приложения (не Spring internal).

Пример из sync crash-loop:
```
ERROR o.s.boot.SpringApplication : Application run failed
org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean 
  with name 'encryptionBackfillJob' ...
Caused by: java.lang.IllegalStateException: 
  phys_person_capacity_status_history_hash backfill enabled without ENCRYPTION_BLIND_INDEX_KEY
```

Причина ясна: не задана env variable. Идти в Deployment/Secret, добавить.

**Другие полезные флаги logs**:
- `-f` (`--follow`) — стрим live (как tail -f).
- `--since=10m` — только за последние 10 минут.
- `--since-time=2026-09-12T10:00:00Z` — с точной timestamp.
- `-c <container>` — если multi-container pod.
- `--all-containers` — все контейнеры pod'а сразу.

---

## 5. kubectl get events — контекст кластера

```bash
kubectl get events -n knp --sort-by='.lastTimestamp' | tail -30
```

Показывает "что происходило в namespace недавно": schedule, image pull, pod restart, PVC events, Ingress reload.

Полезно когда pod не стартует и в describe pod непонятно:
```
Warning  FailedScheduling  pod/isnaknpuser-xyz  0/5 nodes are available: 
  1 Insufficient memory, 4 node(s) had untolerated taint {dedicated: infra}
```

Причина: нет свободной memory на подходящих нодах.

Или:
```
Warning  FailedMount  pod/postgres-0  Unable to attach or mount volumes: 
  timed out waiting for the condition; unattached volumes=[data]
```

PVC не монтируется — проблема со storage provisioner'ом.

---

## 6. Типовые сценарии

### 6.1 CrashLoopBackOff с exit=1

Приложение падает при старте. Смотри `logs --previous`:

- **ClassNotFoundException / NoSuchMethodError** — dependency conflict. Проверь build (`gradle dependencies`) на конфликтующие версии.
- **BeanCreationException** — Spring не может создать бин. Дальше в стектрейсе Caused by → реальная причина: missing env, missing bean, круговая зависимость.
- **PSQLException: Connection to db... refused** — БД недоступна. Проверь `SPRING_DATASOURCE_URL`, network, БД жив ли.
- **liquibase.exception.ValidationFailedException: checksum** — как в нашей ситуации: файл поменяли после apply. Откатить файл или clearCheckSums.
- **IllegalStateException при @PostConstruct** — env variable не задана. `describe pod → Environment` покажет.

### 6.2 OOMKilled

```bash
kubectl describe pod X | grep -A2 "Last State"
```

Видишь `Reason: OOMKilled`. Диагностика:

1. **Live memory usage**:
   ```bash
   kubectl top pod X -n knp
   ```
   `NAME  CPU  MEMORY`
   `X    500m  1900Mi`
   При лимите 2Gi = близко к пределу.

2. **JVM view**:
   ```bash
   kubectl exec X -- jstat -gc 1 1s 10
   ```
   Смотри S0U/S1U/EU/OU (Survivor, Eden, Old use). Если Old близко к limit и растёт после каждого GC → утечка.

3. **Heap dump**:
   ```bash
   kubectl exec X -- jcmd 1 GC.heap_dump /tmp/heap.hprof
   kubectl cp X:/tmp/heap.hprof ./heap.hprof
   ```
   Открой Eclipse Memory Analyzer (MAT). Dominator Tree → топ big retained objects. Path to GC Roots → почему не собирается.

4. **JVM options проверить**:
   ```bash
   kubectl exec X -- jinfo -flags 1 | grep -E "MaxHeapSize|Xmx"
   ```
   Убедись `-Xmx` < container limit минимум на 512Mi (metaspace + direct + stacks).

5. **Утекшие места**:
   - Session-scoped bean с большим state?
   - CacheManager без evict?
   - Static Map с ростом?
   - JDBC statements не закрываются?

### 6.3 ImagePullBackOff

```
Failed to pull image "registry.1sc.kz/isnaknpsync:abc": rpc error: 
  code = Unknown desc = failed to pull and unpack image ...: 
  failed to resolve reference: not found
```

Причины:
- **Tag не существует** — сборка не прошла или деплой указал не тот tag.
- **Нет доступа к registry** — imagePullSecret не настроен.
- **Registry unreachable** — network policy блокирует, DNS не резолвит.

Проверка:
```bash
kubectl get pod X -o yaml | grep -A2 imagePullSecrets
kubectl get secret <secret-name> -n knp -o yaml
```

### 6.4 Deployment зависает (rollout не завершается)

```bash
kubectl rollout status deployment/X -n knp
```

Ждёт, ждёт, timeout.

```bash
kubectl rollout status deployment/X --timeout=5m
```

Что делать:
```bash
kubectl get pods -n knp -l app=X
```
Смотри какие новые pod'ы Ready, какие нет. Если новые не Ready → они не проходят readiness. Смотри их логи, describe.

Rollback:
```bash
kubectl rollout undo deployment/X -n knp
```

### 6.5 Приложение отвечает медленно, 5xx растут

1. **Метрики Prometheus/Grafana**:
   - `rate(http_server_requests_seconds_count{status="5xx"}[5m])` — растёт где?
   - `histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m]))` — p99 latency.
   - `jvm_memory_used_bytes / jvm_memory_max_bytes` — heap.
   - `pg_stat_activity_count` — соединения к БД.

2. **Логи по correlation_id**:
   ```
   {app="isnaknpuser"} |= "ERROR" | json | correlation_id=""
   ```
   Grafana Loki / Kibana.

3. **Thread dump — узкое место в коде**:
   ```bash
   kubectl exec pod-x -- jstack 1 > threads.txt
   ```
   Или через jcmd:
   ```bash
   kubectl exec pod-x -- jcmd 1 Thread.print > threads.txt
   ```
   Читай: много ли threads на одной строке? Blocked на synchronized? Wait на HikariPool? На Rabbit? Fast Thread Analyzer поможет (fastthread.io).

4. **DB slow queries**:
   ```sql
   -- В Postgres
   SELECT pid, state, query, now() - query_start AS duration
   FROM pg_stat_activity
   WHERE state != 'idle' AND now() - query_start > interval '1 minute'
   ORDER BY duration DESC;
   ```

### 6.6 Deploy job'а k8s зависает (Progressing timeout)

Из моего опыта: sync deployment после MR:
```
error: deployment "isnaknpsync" exceeded its progress deadline
```

Означает `spec.progressDeadlineSeconds` истёк (по умолчанию 600s). Новый ReplicaSet не смог поднять replicas в Ready.

Диагностика:
```bash
kubectl describe deploy isnaknpsync -n knp
kubectl get rs -n knp -l app=isnaknpsync  # старая и новая RS
kubectl get pods -n knp -l app=isnaknpsync
```

Новые pod'ы (свежий hash в имени) — почему не Ready? Смотри их логи.

---

## 7. Thread dump: как читать

```bash
kubectl exec pod-x -- jcmd 1 Thread.print > threads.txt
```

Формат:
```
"http-nio-8080-exec-42" #123 daemon prio=5 os_prio=0 tid=0x... nid=0x... waiting for monitor entry
   java.lang.Thread.State: BLOCKED (on object monitor)
    at kz.example.Service.doWork(Service.java:42)
    - waiting to lock <0x00000007f8a0cd28> (a java.lang.Object)
    at kz.example.Controller.endpoint(Controller.java:15)
    ...

"http-nio-8080-exec-43" #124 daemon prio=5 os_prio=0 tid=0x... nid=0x... runnable
   java.lang.Thread.State: RUNNABLE
    at kz.example.Service.compute(Service.java:88)
    ...
```

### Ключевые состояния:

- **RUNNABLE** — работает или в native (не всегда on-CPU).
- **BLOCKED** — ждёт monitor lock, кто-то другой в synchronized блоке.
- **WAITING / TIMED_WAITING** — ждёт условие (`Object.wait()`, `Thread.sleep()`, `LockSupport.park()`).

### Что искать:

**1) Много threads в BLOCKED на одном lock**:
```
"exec-42" BLOCKED
  - waiting to lock <0x00000007f8a0cd28>
"exec-43" BLOCKED
  - waiting to lock <0x00000007f8a0cd28>
"exec-50" RUNNABLE
  - locked <0x00000007f8a0cd28>
    at kz.example.LegacyService.slowMethod(LegacyService.java:100)
```
Причина: `synchronized` bottleneck. `exec-50` держит monitor, все ждут его.

**2) Тред застрял на HikariPool**:
```
"exec-42" TIMED_WAITING
  at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:80)
```
Пул connection'ов исчерпан. Увеличь `spring.datasource.hikari.maximum-pool-size` или ищи транзакции которые долго не закрываются.

**3) Тред в Feign/RestTemplate call, waiting on socket**:
```
"exec-42" RUNNABLE
  at java.net.SocketInputStream.socketRead0
  ...
  at RestTemplate.execute...
```
Remote-сервис отвечает медленно. Смотри timeout настройки.

**4) Rabbit consumer в receive**:
```
"rabbit-consumer" WAITING (parking)
  at com.rabbitmq.client.impl.recovery.RecoveryAwareChannelN.basicGet
```
Норм состояние, ждёт сообщения.

### Инструменты:

- **fastthread.io** — залей dump, красивая визуализация групп потоков.
- **VisualVM** — offline анализ (Load → Thread dump).
- **jstack** — простая версия jcmd.

---

## 8. Postgres — pg_stat_activity, locks, slow queries

БД часто узкое место. Инструменты Postgres:

### 8.1 Кто сейчас что-то делает

```sql
SELECT pid, usename, application_name, state,
       now() - query_start AS duration,
       query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC
LIMIT 20;
```

### 8.2 Locks

```sql
SELECT blocked_locks.pid AS blocked_pid,
       blocked_activity.usename AS blocked_user,
       blocking_locks.pid AS blocking_pid,
       blocking_activity.usename AS blocking_user,
       blocked_activity.query AS blocked_query,
       blocking_activity.query AS blocking_query
FROM pg_locks blocked_locks
JOIN pg_stat_activity blocked_activity ON blocked_locks.pid = blocked_activity.pid
JOIN pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.relation = blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_stat_activity blocking_activity ON blocking_locks.pid = blocking_activity.pid
WHERE NOT blocked_locks.granted;
```

Показывает "кто кого блокирует".

### 8.3 Idle in transaction

Классика прод-проблем — тред начал транзакцию и **не закрыл**. Держит locks, connection pool забивается.

```sql
SELECT pid, state, now() - state_change AS idle_duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY idle_duration DESC;
```

Kill sesscsию:
```sql
SELECT pg_terminate_backend(pid);
```

Причина в коде: `@Transactional` метод бросает exception → транзакция rollback. Но если проглотить exception и продолжать — HikariCP не возвращает connection в пул.

### 8.4 Медленные запросы

Включи `pg_stat_statements`:
```sql
CREATE EXTENSION pg_stat_statements;

SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;
```

Топ по времени.

---

## 9. Actuator endpoints для диагностики

Boot Actuator в живом pod'е — быстрые ответы:

```bash
kubectl port-forward pod/isnaknpuser-xyz -n knp 8081:8080
# в другом терминале
curl localhost:8081/actuator/health
curl localhost:8081/actuator/metrics
curl localhost:8081/actuator/prometheus
curl localhost:8081/actuator/env
curl localhost:8081/actuator/threaddump
curl localhost:8081/actuator/heapdump > heap.hprof
```

**`/actuator/health`** — общий статус (UP/DOWN) и детали компонентов (db, disk, redis).

**`/actuator/threaddump`** — то же самое что jstack, но через HTTP.

**`/actuator/heapdump`** — heap dump в hprof формате. Открывать в MAT.

**`/actuator/env`** — все прописанные env, propertysources, effective values. Огромный вывод, ищи gruop-by:
```
curl localhost:8081/actuator/env/spring.datasource.url
```

**`/actuator/loggers`** — можно **на живую** менять log level:
```
curl -X POST -H "Content-Type: application/json" \
  -d '{"configuredLevel":"DEBUG"}' \
  localhost:8081/actuator/loggers/kz.example.MyService
```

Не надо рестартить pod чтобы включить DEBUG. Не забудь обратно на INFO.

Важно: `/actuator` endpoints обычно НЕ выставляются наружу через Ingress. Только localhost port-forward или отдельный internal service.

---

## 10. Пример реального инцидента (реконструкция из КНП)

**Alert**: `deployment/isnaknpsync exceeded progress deadline`.

**Шаг 1**: `kubectl get pods -n knp | grep sync`
```
isnaknpsync-cc5bd64d5-ds5km   1/1  Running            0    46h
isnaknpsync-5d9f56c665-n2sbf  0/1  CrashLoopBackOff   75   6h
```
Старый pod живой, новый не поднимается.

**Шаг 2**: `kubectl describe pod isnaknpsync-5d9f56c665-n2sbf -n knp`
```
Events:
  Pulling ... image "registry.1sc.kz/isnaknpsync:ac01514b"
  Back-off restarting failed container
```
Образ скачан ok, контейнер стартует и падает.

**Шаг 3**: `kubectl logs isnaknpsync-5d9f56c665-n2sbf --previous --tail=200`
```
ERROR SpringApplication : Application run failed
Caused by: IllegalStateException: 
  phys_person_capacity_status_history_hash backfill enabled without ENCRYPTION_BLIND_INDEX_KEY
```
Причина найдена: env var не задана в Secret.

**Шаг 4**: `kubectl get secret isna-secret -n knp -o yaml | grep ENCRYPTION`
Ключа нет.

**Действие**: DevOps добавляет ключ в Secret, restart pod. Deploy проходит.

Итог: диагностика заняла 5 минут. Без workflow — часы.

---

## 11. Пре-написанные bash scripts, которые полезно иметь

**`k8s-diag.sh` — быстрый обзор одного pod'а**:
```bash
#!/bin/bash
POD=$1
NS=${2:-knp}
echo "=== DESCRIBE ==="
kubectl describe pod $POD -n $NS | tail -60
echo "=== LOGS (previous) ==="
kubectl logs $POD -n $NS --previous --tail=100 2>/dev/null || echo "no previous"
echo "=== LOGS (current) ==="
kubectl logs $POD -n $NS --tail=50
echo "=== EVENTS ==="
kubectl get events -n $NS --sort-by='.lastTimestamp' --field-selector involvedObject.name=$POD | tail -10
```

**`k8s-top.sh` — top pod'ов по CPU/memory**:
```bash
kubectl top pods -n knp --sort-by=memory | head -20
```

**Alias'ы**:
```bash
alias k=kubectl
alias kdp='kubectl describe pod'
alias kgpo='kubectl get pods -o wide'
alias klf='kubectl logs -f'
alias klp='kubectl logs --previous --tail=200'
```

---

## 12. Что не делать в проде

- **`kubectl edit` для быстрого fix'а конфигурации**. Изменения потеряются при следующем deploy. Меняй в Helm/Kustomize/gitops.
- **`kubectl delete pod X --force --grace-period=0`** без понимания последствий. Force delete не даёт shutdown, может остаться повреждённое состояние на диске (для БД — жирный минус).
- **`kubectl scale` для quick fix**. Так же не в git → откат при следующем deploy.
- **Локально работающий `kubectl exec ... -- rm -rf ...`** — я видел как убирали "лишние" файлы у Postgres.
- **`kubectl cp` больших файлов** — забьёт network, timeout. Для heap dumps — маленькие или через objectstorage.

---

## 13. Кратко: чек-лист расследования

Любой прод-алерт:
1. `kubectl get pods` → есть ли новые pod'ы, какой статус.
2. `kubectl describe pod` → Events, Last State, Reason.
3. `kubectl logs --previous` → фактическая ошибка.
4. `kubectl get events` → контекст кластера.
5. **Метрики** (Grafana): нагрузка, memory, CPU trend.
6. **Логи по correlation_id** — если функциональная проблема.
7. **Thread dump** — если приложение висит.
8. **Postgres pg_stat_activity** — если БД узкое место.
9. **Actuator /env** — проверить пропущенные env.
10. Fix → deploy → мониторь метрики.

Держи это чек-листом рядом. Первые 5 минут инцидента — самое дорогое время, привычка выработать заранее.
