# 77. Kubernetes deep: workloads, network, disruption, graceful shutdown

Продвинутые k8s-концепции которые нужно знать когда у тебя больше пары микросервисов. StatefulSet vs Deployment, Ingress, NetworkPolicy, PDB, PodPriority, graceful shutdown в Boot, init/sidecar containers.

Базовые концепции (Pod, Deployment, Service) — в `10-kubernetes-detailed.md`.

---

## 1. Workloads: Deployment vs StatefulSet vs DaemonSet vs Job

### 1.1 Deployment (стандартное)

- Реплики **взаимозаменяемые**. `pod-a-xyz`, `pod-a-abc` — одинаковые, случайный IP, случайное имя.
- Работает для 90% микросервисов: stateless HTTP-сервис, обрабатывает запрос и забыл.
- Rolling update: убивает старые поды, поднимает новые, по одному.

### 1.2 StatefulSet

Когда **порядок и идентичность** важны. Примеры: Postgres, Kafka, Elasticsearch, Redis Cluster, Zookeeper — всё что имеет **shared state** между репликами.

Что даёт:
- **Stable pod names**: `postgres-0`, `postgres-1`, `postgres-2` (не хеши).
- **Ordered startup/shutdown**: `postgres-1` не стартует пока `postgres-0` не Ready. При shutdown — в обратном порядке.
- **Persistent volume per pod**: каждый pod получает СВОЙ PVC (не shared). Даже после удаления pod'а — том остаётся, привяжется к новому pod'у с тем же именем.
- **Headless Service**: `postgres-0.postgres.default.svc.cluster.local` — прямой DNS до конкретного pod'а.

Пример:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels: {app: postgres}
  template:
    metadata:
      labels: {app: postgres}
    spec:
      containers:
      - name: postgres
        image: postgres:17
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 100Gi
      storageClassName: fast-ssd
```

Каждый pod получит свой PVC `data-postgres-0`, `data-postgres-1`, `data-postgres-2`. При delete pod'а PVC остаётся, при пересоздании тот же PVC монтируется.

Важно: StatefulSet ≠ auto-clustering. Ты только получаешь стабильную идентичность. Кластеризация (Postgres streaming replication, Kafka broker discovery) — надо настраивать в приложении.

### 1.3 DaemonSet

**По одному pod'у на каждой ноде** кластера. Используется для инфраструктурных нужд:
- Fluent Bit (сборка логов).
- Node Exporter (метрики ноды в Prometheus).
- kube-proxy, CNI plugin.

При добавлении новой ноды k8s автоматически шедулит на неё pod из DaemonSet.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
spec:
  selector:
    matchLabels: {app: fluent-bit}
  template:
    metadata:
      labels: {app: fluent-bit}
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.2
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

### 1.4 Job / CronJob

**Job** — одноразовая задача. Запусти pod, дождись exit 0, удали.
- Миграция БД (Liquibase, Flyway).
- Backfill данных.

**CronJob** — Job по расписанию.
- Ротация ключей.
- Ежедневный отчёт.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-old-logs
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cleanup
            image: my-cleanup:1.0
            args: ["--retention-days=30"]
```

---

## 2. Storage: PV, PVC, StorageClass

Абстракция k8s поверх реальных дисков.

### 2.1 Терминология

- **PersistentVolume (PV)** — реальный кусок диска (EBS volume в AWS, disk в Azure/GCP, локальный SSD, NFS). Кластерный ресурс, живёт вне pod'а.
- **PersistentVolumeClaim (PVC)** — request на диск от pod'а: "мне нужно 100Gi RWO, класс fast-ssd". k8s находит подходящий PV и связывает.
- **StorageClass** — шаблон динамического provisioning'а. `fast-ssd`, `slow-hdd`, `nfs`. При создании PVC k8s провизионит PV сам согласно классу.

### 2.2 Access modes

- **ReadWriteOnce (RWO)** — один pod пишет/читает. Стандарт для БД (EBS, standard SSD).
- **ReadOnlyMany (ROX)** — много pod'ов читают. Редко.
- **ReadWriteMany (RWX)** — много pod'ов пишут/читают. Требует NFS/CephFS/EFS. Медленно, дорого. Обычно избегается.

### 2.3 Reclaim policy

- **Retain** — при удалении PVC PV остаётся, надо чистить руками. Для важных данных.
- **Delete** — при удалении PVC удаляется и PV (и реальный диск!). Для эфемерных.

Пример misconfig: `reclaimPolicy: Delete` для базы → удалили deployment → ушёл PVC → drop of database. Классика продовой катастрофы.

### 2.4 emptyDir vs hostPath vs configMap/secret

Мимо PV:
- **emptyDir** — временный том на диске ноды. Живёт пока pod живёт. Для scratch/cache.
- **hostPath** — том с ноды host (`/var/log`, `/var/run/docker.sock`). Опасно, обычно только для infra DaemonSet'ов.
- **configMap / secret** — файлы с содержимым из объекта k8s. Читай в контейнере как обычные файлы.

---

## 3. Network: Service, Ingress, NetworkPolicy

### 3.1 Service — cluster-internal LB

Уже разобрано в `10-kubernetes-detailed.md`. Кратко:
- **ClusterIP** — виртуальный IP + DNS `<service>.<ns>.svc.cluster.local`. Пробуют pod'ы по selector.
- **NodePort** — открывает порт на каждой ноде. Debug/dev.
- **LoadBalancer** — просит внешний LB (AWS ELB, MetalLB). Прод-точка входа для одного сервиса.

**Headless Service** (`clusterIP: None`) — без LB, DNS возвращает список pod IP напрямую. Для StatefulSet: `postgres-0.postgres.default.svc.cluster.local`.

### 3.2 Ingress — HTTP L7 роутер

Один Ingress Controller (nginx-ingress, Traefik, HAProxy Ingress) слушает на 80/443 всей cluster, роутит запросы по host/path в Services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: knp-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts: [knp.kgd.gov.kz]
    secretName: knp-tls
  rules:
  - host: knp.kgd.gov.kz
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: isnaknpgateway
            port:
              number: 80
      - path: /api/user
        pathType: Prefix
        backend:
          service:
            name: isnaknpuser
            port:
              number: 80
```

Один IP + один TLS-cert для многих сервисов. С `cert-manager` — автоматическое обновление Let's Encrypt.

**nginx-ingress** — самый популярный. Traefik — легче конфигурируется через labels, поддерживает CRD. Выбор проектный.

### 3.3 NetworkPolicy — L3/L4 firewall

По умолчанию в k8s любой pod может ходить в любой другой pod. Плохо для безопасности: если один сервис взломан → доступ ко всем.

NetworkPolicy — правила "кто с кем может общаться".

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-only-app
  namespace: knp
spec:
  podSelector:
    matchLabels: {app: postgres}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {app: isnaknpuser}
    - podSelector:
        matchLabels: {app: isnaknpsync}
    ports:
    - protocol: TCP
      port: 5432
```

Что говорит: pod'ы с label `app=postgres` принимают входящие соединения на порт 5432 **только** от pod'ов с label `app=isnaknpuser` или `app=isnaknpsync`. Всё остальное — reject.

**Важно**: NetworkPolicy требует **CNI plugin с поддержкой** (Calico, Cilium, Weave). Стандартный kubenet не умеет.

Хорошая практика:
1. Default deny all ingress в namespace.
2. Явные rules для конкретных пар "источник → назначение".

---

## 4. PodDisruptionBudget (PDB) — защита от массового перезапуска

Проблема: node drain (обслуживание, апгрейд). k8s хочет эвакуировать pod'ы с ноды. Если у тебя `replicas: 3` и все на одной ноде — ты потеряешь все 3 разом, downtime.

PDB говорит: "минимум 2 из 3 pod'ов моего приложения должны быть Available всегда, даже во время drain".

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: isnaknpuser-pdb
spec:
  minAvailable: 2      # или maxUnavailable: 1
  selector:
    matchLabels: {app: isnaknpuser}
```

Kubernetes при drain'е ноды спросит: "могу ли я убить этот pod, не нарушив PDB?" — если нет, drain ждёт.

**Правило прода**: PDB для каждого критичного сервиса. Обычно `minAvailable: N-1` где N = replicas.

**Гоча**: если у тебя всего 1 replica → PDB нельзя настроить (drain никогда не пройдёт). Для важных сервисов **всегда 2+ replicas**.

---

## 5. Priority и Preemption

Кластер переполнен, надо запустить критичный сервис. Что вытеснить?

**PriorityClass** — численный приоритет pod'а.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "Для критичных сервисов knp"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: isnaknpgateway
spec:
  template:
    spec:
      priorityClassName: high-priority
```

Если новый pod с priority 1000 не помещается — scheduler ищет pod'ы с меньшим priority на нодах, вытесняет (evicts) их.

**Правило прода**: критичные — high, обычные — medium, batch/cron — low.

Стандартные k8s имеет `system-cluster-critical` (2 000 000 000) для kube-system pod'ов.

---

## 6. QoS classes

Каждый pod автоматически получает QoS class на основе `requests/limits`:

- **Guaranteed** — requests == limits для всех контейнеров, всех ресурсов. При OOM или nodetension — убивают в последнюю очередь.
- **Burstable** — requests < limits хоть где-то. Обычная категория.
- **BestEffort** — requests не указаны нигде. Убивают первыми при пресcении.

Проверить: `kubectl describe pod X | grep QoS`.

Правило прода: **все критичные — Guaranteed**, `requests=limits`. Batch — Burstable ок. Никогда BestEffort в prod.

```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "500m"
```

Такой pod = Guaranteed. При OOM ноды — убивают в последнюю очередь.

---

## 7. Graceful shutdown: SIGTERM → preStop → terminationGracePeriod

Что происходит когда k8s хочет убить pod:

```
1. kubectl delete pod X (или rolling update / eviction)
2. k8s помечает pod как "Terminating"
3. Pod удаляется из endpoints Service'а
   → Новые запросы не идут больше в этот pod
4. Одновременно:
   - Выполняется preStop hook (если настроен)
   - SIGTERM летит в контейнер
5. terminationGracePeriodSeconds (default 30s) отсчитывается
6. Если контейнер не завершился — SIGKILL
```

### 7.1 Правильный Spring Boot graceful shutdown

Boot 2.3+ поддерживает graceful shutdown нативно:

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
```

При SIGTERM:
1. Boot прекращает принимать новые HTTP-запросы.
2. Ждёт до `timeout-per-shutdown-phase` пока in-flight запросы завершатся.
3. Закрывает контекст (destroys beans в правильном порядке).
4. Exits 0.

Важно: `terminationGracePeriodSeconds` в k8s **больше** чем `timeout-per-shutdown-phase` в Boot. Иначе SIGKILL прибьёт middle-shutdown.

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 45  # > 25 из Boot config
      containers:
      - name: app
        # ...
```

### 7.2 preStop hook — задержка перед SIGTERM

Race condition в k8s: между "pod удалён из endpoints" и "kube-proxy на всех нодах обновил iptables" — 1-5 секунд. В это время новый запрос всё ещё может прилететь, но Boot уже закрылся.

preStop `sleep 5` даёт время iptables обновиться, ПОТОМ SIGTERM:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
```

Порядок:
1. `pod → Terminating`, удаляется из endpoints.
2. **preStop `sleep 5`** — окно чтобы kube-proxy на всех нодах обновил правила.
3. SIGTERM в контейнер.
4. Boot начинает graceful shutdown.
5. Boot exits, контейнер done.

---

## 8. Health probes: liveness, readiness, startup

Boot Actuator даёт эндпоинты, k8s опрашивает.

### 8.1 Liveness

"Приложение живое или зависло?". Если fail → k8s **перезапускает** pod.

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

**Правило**: liveness никогда не должен зависеть от внешних систем (БД, других сервисов). Иначе временный сбой БД → k8s начинает перезапускать все pod'ы → каскадный отказ.

### 8.2 Readiness

"Готов ли pod принимать трафик?". Если fail → pod остаётся Running, но **удаляется из endpoints Service'а**.

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

**Правило**: readiness МОЖЕТ зависеть от внешних систем (БД). Если БД недоступна — pod удаляется из балансировки, трафик идёт на другие replicas.

Boot Actuator по умолчанию имеет `ReadinessStateHealthIndicator` — с БД в составе.

### 8.3 Startup probe

Для медленно стартующих приложений (JVM warmup, cache initialization).

```yaml
startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

Пока startup не Ok — liveness/readiness не проверяются. Даёт 30 x 10 = 5 минут на старт.

---

## 9. Init containers и sidecars

### 9.1 Init container

Контейнер, который **запускается ДО главного**, должен exit 0 чтобы главный стартовал. Sequential.

Применения:
- Ждать пока БД доступна (`while ! nc -z db 5432; do sleep 1; done`).
- Скачать конфиг/секрет из Vault и положить на shared volume.
- Прогнать миграции Liquibase перед стартом сервиса.

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 1; done']
  - name: run-migrations
    image: my-liquibase:1.0
  containers:
  - name: app
    image: my-app:1.0
```

### 9.2 Sidecar

Второй контейнер в том же pod'е, **параллельно** с главным. Общий network namespace (localhost) и volumes.

Примеры:
- **Log-forwarder**: контейнер читает логи главного, отправляет в удалённое место. Устарел с DaemonSet Fluent Bit.
- **Proxy**: Envoy / Istio-sidecar перехватывает исходящий трафик, добавляет mTLS, retry, tracing.
- **Config-reloader**: следит за ConfigMap, перегружает главный при изменении.
- **Backup-agent**: раз в час снимает snapshot тома главного.

Anti-pattern: **всё в один pod через sidecar**. Разделяй по границе жизненного цикла: если два процесса должны стартовать/умирать вместе → один pod. Если независимо → разные Deployment'ы.

---

## 10. Rolling update: как безопасно катить новую версию

Default стратегия Deployment:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%          # сколько новых можно поднять сверху (round up)
    maxUnavailable: 25%    # сколько старых можно убить (round down)
```

Пример при `replicas: 4`:
- `maxSurge: 25%` = 1 → на пике = 5 pod'ов.
- `maxUnavailable: 25%` = 1 → в моменте не меньше 3 Ready.

Шаги:
1. Создать 1 новый pod (v2). Ждать Ready.
2. Убить 1 старый (v1).
3. Повторять пока все не v2.

Если новый pod не становится Ready — rollout зависает. `kubectl rollout status deployment/X` покажет.

**Rollback**:
```bash
kubectl rollout undo deployment/isnaknpuser
kubectl rollout undo deployment/isnaknpuser --to-revision=3
```

**История**:
```bash
kubectl rollout history deployment/isnaknpuser
```

### 10.1 Recreate strategy — не для прод

```yaml
strategy:
  type: Recreate
```

Убьёт ВСЕ старые, потом создаст новые. Даунтайм гарантирован. Только для dev/staging.

---

## 11. Resource management — что происходит при OOM

Когда pod превышает memory limit → **OOMKilled**.

`kubectl describe pod X | grep -A5 "Last State"`:
```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

- Exit 137 = 128 + 9 (SIGKILL).
- Убивает **node kernel** (cgroup OOM), не k8s.
- Kubelet видит exit, применяет restart policy (обычно Always → перезапуск).

Как расследовать:
1. `kubectl top pod X` — сколько ест сейчас.
2. `kubectl describe pod X` — есть ли `Reason: OOMKilled` в last state.
3. Heap dump через `jcmd`:
   ```
   kubectl exec pod-x -- jcmd 1 GC.heap_dump /tmp/heap.hprof
   kubectl cp pod-x:/tmp/heap.hprof ./heap.hprof
   ```
   Открывать Eclipse MAT — искать leaked objects.
4. Разобрать в MAT: "Dominator Tree", ищи очень большие retained sizes.

Правило: **JVM heap < container memory limit**. Обычно `-Xmx` = 70-80% от `limit`. Пример: `limits.memory: 2Gi`, `-Xmx1500m`. Оставь запас для metaspace, direct memory, stacks.

---

## 12. Как всё это связано на реальном проде

Полный шаблон deployment для типичного микросервиса на КНП:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: isnaknpuser
  namespace: knp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels: {app: isnaknpuser}
  template:
    metadata:
      labels: {app: isnaknpuser, version: v1.2.3}
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      priorityClassName: high-priority
      terminationGracePeriodSeconds: 45
      containers:
      - name: app
        image: registry.1sc.kz/isnaknpuser:v1.2.3
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: prod
        - name: JAVA_TOOL_OPTIONS
          value: "-Xmx1500m -XX:+ExitOnOutOfMemoryError"
        resources:
          requests: {memory: 2Gi, cpu: 500m}
          limits: {memory: 2Gi, cpu: 2000m}
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
        startupProbe:
          httpGet: {path: /actuator/health/liveness, port: 8080}
          failureThreshold: 30
          periodSeconds: 10
        livenessProbe:
          httpGet: {path: /actuator/health/liveness, port: 8080}
          periodSeconds: 15
          failureThreshold: 3
        readinessProbe:
          httpGet: {path: /actuator/health/readiness, port: 8080}
          periodSeconds: 5
          failureThreshold: 2
---
apiVersion: v1
kind: Service
metadata:
  name: isnaknpuser
  namespace: knp
spec:
  selector: {app: isnaknpuser}
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: isnaknpuser-pdb
  namespace: knp
spec:
  minAvailable: 2
  selector:
    matchLabels: {app: isnaknpuser}
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isnaknpuser-policy
  namespace: knp
spec:
  podSelector: {matchLabels: {app: isnaknpuser}}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: {matchLabels: {app: isnaknpgateway}}
    ports:
    - protocol: TCP
      port: 8080
```

---

## 13. Кратко

- **Workloads**: Deployment (stateless), StatefulSet (identity+PV), DaemonSet (per-node), Job/CronJob (batch).
- **Storage**: PV/PVC/StorageClass. RWO для БД, reclaim=Retain для важного.
- **Network**: Service (internal), Ingress (external L7), NetworkPolicy (firewall).
- **PDB**: минимум N-1 pod доступен при drain. Обязательно для критичных.
- **QoS**: Guaranteed для критичных (requests==limits).
- **Priority**: high для критичных, чтобы вытеснять при переполнении.
- **Graceful shutdown**: preStop sleep 5 → SIGTERM → Boot shutdown → exit. `terminationGracePeriodSeconds` больше чем shutdown-timeout Boot.
- **Probes**: startup (для медленного старта), liveness (авторестарт), readiness (LB кнопка).
- **Init containers**: миграции, wait-for-db, config-fetch.
- **Sidecars**: envoy proxy, config-reloader. Не пихай всё в один pod.
- **Rolling update**: `maxUnavailable: 0` для zero-downtime. Rollback: `kubectl rollout undo`.
- **OOM**: -Xmx < limit. Heap dump через jcmd + MAT.
