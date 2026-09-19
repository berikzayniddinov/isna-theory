# 80. Kubernetes internals — углублённая подготовка к собесу

Файл про то, **как реально устроен K8s внутри**. Базу (Pod/Deployment/Service, probes, PVC, Helm) смотри в `10-kubernetes-detailed.md`. Продвинутые workloads / graceful shutdown / PDB — в `77-kubernetes-deep-microservices.md`.

Здесь — внутренности control plane (etcd/Raft, apiserver watch, scheduler framework), controller pattern с informers, kube-proxy режимы, CNI на уровне пакета, RBAC/admission, CSI, CRD/operator, HA control plane, debugging сценарии и большой блок собесных вопросов с ответами.

---

## 0. Ментальная модель

Kubernetes — это распределённая **desired-state машина**, работающая по принципу **level-triggered reconciliation**.

- **Пользователь** пишет в apiserver: «хочу N объектов ресурса R с параметрами spec».
- **Apiserver** валидирует, прогоняет через admission, кладёт в etcd.
- **Controllers** слушают изменения (через watch) и приводят реальный мир к spec (reconcile loop).
- **Scheduler** решает где именно запустить Pod (это тоже controller).
- **kubelet** на ноде запускает контейнеры и репортит status обратно в apiserver.

Всё остальное — вариации этих четырёх ролей. Даже HPA, Ingress Controller, cert-manager, ArgoCD — это **контроллеры**. Понимаешь reconcile loop — понимаешь всю систему.

Ключевые свойства модели:

1. **Level-triggered, а не edge-triggered.** Controller не «получает событие один раз», он видит **текущее состояние** каждый раз и приводит его к spec. Пропустил уведомление → следующий resync его увидит.
2. **Eventually consistent.** Между apply и наступлением реального состояния — задержка. Спрашивать «когда все Pod'ы будут Ready» — неправильный вопрос, правильный: «мониторь status».
3. **API — единственная правда.** Всё общение — через apiserver. Никаких RPC между контроллерами.
4. **Etcd — источник правды.** Всё что не в etcd — не существует для K8s.

---

## 1. etcd — распределённое хранилище

### 1.1 Что это

**etcd** — распределённая key-value БД на алгоритме консенсуса **Raft**. Написана на Go в CoreOS (~2013), сейчас — CNCF-проект. K8s использует etcd v3 как единственный persistent store для всех своих объектов.

Свойства:
- **Strong consistency** — read after write гарантирован (для linearizable reads).
- **HA через Raft** — переживает падение (N-1)/2 нод из N (при N=3 → 1 падение, при N=5 → 2 падения).
- **Watch API** — клиент подписывается на изменения ключей.
- **MVCC** — многоверсионная модель, каждое изменение = новая версия.

### 1.2 Raft — базовая идея

Есть N нод etcd. Одна из них — **leader**, остальные — **followers**.

- Клиент пишет → leader принимает → реплицирует в лог followers → как только большинство подтвердило (**quorum**) → leader коммитит и отвечает клиенту.
- Followers применяют коммиченные записи к своему state machine.
- Если leader умер → followers через таймаут начинают выборы (election), голосуют, выбирают нового leader'а.
- Каждый терм (эпоха) leader — уникальный, чтобы отличать «старые» команды от новых.

**Quorum-формула**: `Q = floor(N/2) + 1`.

| Нод | Quorum | Переживает падение |
|-----|--------|--------------------|
| 1   | 1      | 0                  |
| 3   | 2      | 1                  |
| 5   | 3      | 2                  |
| 7   | 4      | 3                  |

Чётное число (2, 4, 6) — **бесполезно**: quorum тот же, что и у нечётного меньше на 1, но переживает меньше падений. **Всегда ставь нечётное** — 3 (стандарт) или 5 (крупный кластер).

### 1.3 MVCC и watch

Каждое изменение в etcd = новая **revision** (монотонно растущий int64). Пример: `revision 100` — установили ключ X; `revision 101` — обновили; `revision 102` — удалили.

**Ключевое**: старые версии не удаляются сразу — они хранятся в **BoltDB** (backend etcd). Это позволяет:
- Читать «на момент revision R».
- Watch «начиная с revision R» — подписка получает все изменения, произошедшие после R.

**Watch mechanism** — фундамент для K8s controllers. Apiserver держит watch на etcd, controllers держат watch на apiserver → любое изменение ресурса моментально доставляется до всех подписчиков.

### 1.4 Compaction

Старые revisions занимают место. Раз в N минут (обычно 5) apiserver вызывает `etcd compact` — удаляет revisions старше порога. После compaction — `etcd defrag`, чтобы физически освободить место в BoltDB.

**Собесная ловушка**: если etcd переполнен (`space quota exceeded`) — весь K8s становится **read-only**. Симптомы: `kubectl apply` возвращает `mvcc: database space exceeded`. Лечение:
1. `etcdctl endpoint status` — проверить занятость.
2. `etcdctl alarm list` — есть ли `NOSPACE` alarm.
3. `etcdctl compact <rev>` + `etcdctl defrag`.
4. `etcdctl alarm disarm`.

### 1.5 Что хранит etcd в K8s

- Все объекты: Pod, Deployment, Service, ConfigMap, Secret, CustomResources.
- Namespaces.
- Node registrations.
- Leases (для leader election контроллеров).
- events (короткоживущие).

**НЕ хранит**: логи, метрики, docker images, container states (это на кубелетах).

### 1.6 Secrets в etcd

По умолчанию Secret хранится в etcd **в base64** — это не шифрование, это кодирование. Любой с доступом к etcd читает пароль в открытом виде.

**Encryption at rest** — включается через `--encryption-provider-config` у apiserver. Форматы:
- `identity` — plain (default).
- `aescbc` — AES-CBC 32-byte key.
- `aesgcm` — AES-GCM, требует ротации ключа.
- `secretbox` — XSalsa20+Poly1305.
- `kms` — интеграция с внешним KMS (AWS KMS, HashiCorp Vault). Правильный prod-выбор.

Ротация ключа — двухфазная: сначала добавляешь новый ключ вторым, пересохраняешь все Secrets (чтобы они зашифровались новым), потом убираешь старый.

### 1.7 Disaster recovery

**Backup**:
```bash
etcdctl --endpoints=... snapshot save backup.db
```

**Restore** (для одной ноды):
```bash
etcdctl snapshot restore backup.db \
  --name m1 \
  --initial-cluster m1=http://host1:2380 \
  --initial-advertise-peer-urls http://host1:2380 \
  --data-dir /var/lib/etcd
```

Правило: **backup etcd ежедневно, храни минимум 7 дней**. Без backup'а etcd → потеряли весь кластер (описание объектов, RBAC, secrets). Реальные Pod'ы продолжат работать пока живы ноды, но управление сломано.

### 1.8 Собесные капканы

- **«Почему нельзя ставить 2 ноды etcd?»** — Quorum = 2, при падении одной ноды кластер read-only. То есть 2 ноды дают меньше доступности, чем 1.
- **«Что произойдёт если etcd упадёт целиком?»** — apiserver возвращает 500, ничего нельзя создавать/обновлять. Но существующие Pod'ы **продолжают работать** — kubelet не общается с etcd напрямую, только через apiserver. Service продолжает балансировать (kube-proxy уже настроил iptables).
- **«Split-brain?»** — Raft гарантирует что split brain невозможен: partitioned минорити не может выбрать leader'а без quorum.
- **«Почему etcd в отдельных нодах?»** — Изоляция от apiserver-нагрузки, отдельные диски (etcd очень чувствителен к latency диска — рекомендуют NVMe/SSD с fsync <10ms).

---

## 2. kube-apiserver

### 2.1 Роль

Единственный компонент, который общается с etcd. Все — kubelet, scheduler, controller-manager, kubectl, HPA, любые операторы — идут через apiserver.

Функции:
- REST-фасад над etcd.
- Auth (аутентификация) + Authz (авторизация RBAC).
- **Admission** — валидация и мутация входящих запросов.
- **Watch** — стриминг изменений подписчикам.
- **API aggregation** — можно подключить сторонние API-серверы под тем же `kubectl`.

Пишется на Go, работает как обычный HTTP-сервер на порту 6443 (TLS).

### 2.2 Полный путь запроса

```
kubectl apply -f pod.yaml
     ↓
[TLS handshake, cert auth или token auth]
     ↓
[Authentication]  ← кто ты? (X.509, token, OIDC, ServiceAccount JWT)
     ↓
[Authorization]   ← можешь ли ты это делать? (RBAC, ABAC, Webhook, Node)
     ↓
[Mutating Admission]   ← модифицируем объект (inject sidecar, добавить label, установить defaults)
     ↓
[Object Schema Validation]  ← OpenAPI-схема ресурса
     ↓
[Validating Admission]  ← можно ли сохранить? (ResourceQuota, PSA, custom webhooks)
     ↓
[etcd write]
     ↓
Response to client
     ↓
Watch notifications to все подписчики
```

**Порядок mutating → validating** — важен: сначала все меняют объект (в любом порядке), потом все валидируют финальный вариант.

### 2.3 Watch — как это работает

Client шлёт GET на `/api/v1/pods?watch=true&resourceVersion=100500`. Apiserver держит соединение открытым (chunked HTTP), стримит события: `ADDED`, `MODIFIED`, `DELETED`.

**Каждое событие содержит resourceVersion** — если клиент разорвал соединение и переподключился, продолжит с последнего RV.

**Что если сервер выкинул старые revisions из кэша?** — Клиент получает `410 Gone`. Он обязан сделать полный **relist** (`GET /pods`), взять новый resourceVersion и начать watch заново. Это стандартный паттерн «list-then-watch» и его реализует client-go **reflector**.

### 2.4 API groups и версии

Все объекты K8s разделены на **API groups**:
- Core (`/api/v1`) — Pod, Service, ConfigMap, Secret, Namespace, Node, PersistentVolume.
- `apps/v1` — Deployment, StatefulSet, DaemonSet, ReplicaSet.
- `batch/v1` — Job, CronJob.
- `networking.k8s.io/v1` — Ingress, NetworkPolicy.
- `rbac.authorization.k8s.io/v1` — Role, ClusterRole, RoleBinding, ClusterRoleBinding.
- `policy/v1` — PodDisruptionBudget.
- `scheduling.k8s.io/v1` — PriorityClass.
- `storage.k8s.io/v1` — StorageClass, CSIDriver, VolumeSnapshotClass.
- `autoscaling/v2` — HPA.
- `apiextensions.k8s.io/v1` — CustomResourceDefinition.

Уровни зрелости: `alpha` (v1alpha1, отключён по умолчанию, обратной совместимости нет) → `beta` (v1beta1, включён, но API может измениться) → `stable` (v1).

### 2.5 API aggregation layer

Позволяет включить кастомный apiserver под тем же `kubectl`. Пример: `metrics-server` — предоставляет `/apis/metrics.k8s.io/v1beta1/pods`. Не через CRD (CRD хранят в основном etcd), а как отдельный HTTP-сервер, к которому проксирует main apiserver.

Регистрация — через `APIService`:
```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  service:
    name: metrics-server
    namespace: kube-system
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```

Разница с CRD: **CRD** — новый тип ресурса, хранится в стандартном etcd. **APIService** — прокси к сторонней реализации apiserver, может хранить где угодно.

### 2.6 Rate limiting и priority

Apiserver — узкое место. При флуде запросов может лечь. Механизм защиты — **APF (API Priority and Fairness)**.

Определяются **FlowSchemas** (по кому фильтровать) → **PriorityLevelConfigurations** (сколько concurrent).

Пример: `kubectl` от `system:masters` — высший приоритет; `system:node:xxx` (кубелеты) — средний; неизвестные ServiceAccount — низкий. При переполнении низкоприоритетные ждут в очереди.

### 2.7 Собесные вопросы

- **«Как аутентифицируется kubelet при обращении к apiserver?»** — X.509-сертификат, подписанный CA кластера, `CN: system:node:<hostname>`. Node authorizer разрешает только те объекты, которые касаются этой ноды.
- **«Что такое ServiceAccount и как он работает внутри Pod?»** — SA — объект K8s, каждый Pod получает JWT-токен, вмонтированный в `/var/run/secrets/kubernetes.io/serviceaccount/token`. Проекционный volume, ротация через 1 час (bound token, TokenRequest API, K8s 1.22+).
- **«Как ускорить apiserver?»** — Increase `--max-requests-inflight`, разнести read/write на разные endpoints, APF-конфиг, шардировать etcd (etcd events отдельно от основного).

---

## 3. kube-scheduler

### 3.1 Что делает

Слушает через watch: «Pod с `spec.nodeName == ""`». Для каждого такого пода — решает на какую ноду его назначить. Пишет `spec.nodeName = <node>` в apiserver. Дальше kubelet видит и запускает.

**Scheduler НЕ запускает контейнеры.** Он только принимает решение.

### 3.2 Scheduler framework (v1.19+)

Раньше был двухфазный алгоритм: **predicates** (фильтр «нода подходит?») → **priorities** (скор «насколько хорошо?»). Теперь — **framework** с 12 extension points, в каждой из которых работают плагины.

Основные extension points (в порядке выполнения):

```
QueueSort  → PreEnqueue  → PreFilter  → Filter  → PostFilter  →
PreScore   → Score  → NormalizeScore  →
Reserve  → Permit  → PreBind  → Bind  → PostBind
```

- **Filter** — «может ли Pod встать на эту ноду?» (аналог predicates). Плагины: `NodeResourcesFit`, `NodeAffinity`, `PodTopologySpread`, `TaintToleration`, `NodeUnschedulable`, `VolumeBinding`, `NodeName`, `NodePorts`.
- **Score** — «насколько хорошо?». Плагины: `NodeResourcesFit` (spread/binpack), `ImageLocality` (нода уже имеет образ), `InterPodAffinity`, `NodeAffinity`.
- **Bind** — фактическая привязка (обычно `DefaultBinder` пишет `spec.nodeName`).

### 3.3 Как работают предикаты и приоритеты (для интуиции)

**Filter phase**: для каждой ноды прогоняем все filter-плагины. Если хоть один сказал «нет» → нода отсеяна. Оставшиеся — «feasible».

**Score phase**: каждый score-плагин выставляет 0-100. Веса плагинов суммируются. Максимум — назначается.

**Tie-breaker**: если ничьи → случайный выбор среди максимальных.

**Что если нет feasible нод?** → **PostFilter** запускается. Обычно там — **Preemption**: пытаемся вытеснить Pod'ы с меньшим приоритетом на какой-то ноде, чтобы освободить место.

### 3.4 Управление размещением из spec

**nodeSelector** — самое простое, соответствие labels:
```yaml
spec:
  nodeSelector:
    disk: ssd
    zone: us-east-1a
```

**nodeAffinity** — гибче, поддерживает `In`, `NotIn`, `Exists`, `Gt`, `Lt`:
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:  # обязательно
      nodeSelectorTerms:
      - matchExpressions:
        - key: gpu
          operator: In
          values: [nvidia-a100, nvidia-h100]
    preferredDuringSchedulingIgnoredDuringExecution:  # желательно
    - weight: 100
      preference:
        matchExpressions:
        - key: zone
          operator: In
          values: [us-east-1a]
```

**podAffinity / podAntiAffinity** — размещать рядом (или подальше от) других подов:
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: isnaknpuser
      topologyKey: kubernetes.io/hostname  # разные ноды
```

`topologyKey` — какой label ноды считаем «топологической единицей». `hostname` — ноды. `topology.kubernetes.io/zone` — availability zones. Правило: HA-приложения → antiAffinity по zone.

**Taints и tolerations** — обратная логика: нода **отталкивает** поды, если те не имеют соответствующего tolerance:
```bash
kubectl taint node worker1 dedicated=gpu:NoSchedule
```
Теперь только Pod с
```yaml
tolerations:
- key: dedicated
  operator: Equal
  value: gpu
  effect: NoSchedule
```
может встать на `worker1`.

Effects: `NoSchedule` (не шедулим), `PreferNoSchedule` (стараемся не), `NoExecute` (не шедулим И выгоняем существующие без tolerance).

**TopologySpreadConstraints** — равномерное распределение:
```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: isnaknpuser
```
Скажет: «между zone разница по количеству подов не больше 1». Правильнее чем podAntiAffinity для равномерного распределения (antiAffinity даёт «либо один-на-хост, либо всё сломалось»).

### 3.5 Preemption

При `spec.priorityClassName` с высоким value — если Pod не помещается, scheduler ищет ноду, где можно выгнать Pod'ы с меньшим priority, чтобы освободить место.

Алгоритм упрощённо:
1. Filter не нашёл feasible.
2. PostFilter (`DefaultPreemption` плагин) для каждой ноды пытается: какие поды с меньшим priority надо убрать чтобы наш встал?
3. Выбирает ноду с наименьшим «повреждением».
4. Отправляет графс-делит preempted подов (с их terminationGracePeriodSeconds).
5. Наш Pod ждёт в очереди пока освободится, потом шедулится.

### 3.6 Как отладить «Pending forever»

```bash
kubectl describe pod X
# Events:
#   Warning FailedScheduling  0/5 nodes are available: 3 Insufficient memory, 2 node(s) had taint {...}
```

Из Events понятно почему. Причины:
- `Insufficient cpu/memory` — resource requests не влезли.
- `node(s) had taint` — нет tolerance.
- `didn't match Pod's node affinity/selector` — не подошли labels.
- `didn't find available persistent volumes to bind` — PVC не смогли привязаться.
- `too many pods` — max pods per node (default 110).

### 3.7 Custom scheduler

Можно писать свой scheduler или расширить дефолтный через `KubeSchedulerConfiguration` + свои плагины. Практически — редко нужно. Чаще — `nodeAffinity` + taints решают всё.

Пример второго scheduler'а в одном кластере — Pod указывает `spec.schedulerName: my-custom-scheduler`. Оба scheduler'а работают, каждый шедулит свои Pod'ы.

---

## 4. Controller pattern (сердце K8s)

### 4.1 Reconcile loop

Псевдокод любого controller'а в K8s:
```go
for {
    desired := getDesiredState()   // из apiserver, spec
    actual := getActualState()     // из apiserver, status + реальный мир
    if desired != actual {
        act(desired, actual)       // сделать что-то чтобы приблизиться
    }
    sleep(shortWait)
}
```

На практике — не polling, а **event-driven**: controller подписывается на **watch**, event триггерит reconcile. Плюс — периодический resync (default 10 минут) как страховка от пропущенных событий.

**Level vs edge triggering**: K8s controller — **level-triggered**. Смотрит на текущее состояние, а не на «что произошло». Пропустил событие → следующий resync или следующее событие всё равно приведёт к правильному действию.

### 4.2 Informers, Listers, Work queues

Реальный controller (написан обычно на client-go для Go) выглядит так:

```
                                                    ┌─────────────┐
                                                    │  Reflector  │──── watch ────► apiserver
                                                    └──────┬──────┘
                                                           │ raw events
                                                           ▼
                                                    ┌─────────────┐
                                                    │  DeltaFIFO  │
                                                    └──────┬──────┘
                                                           │
                                                           ▼
                                                    ┌─────────────┐
                                                    │   Informer  │
                                                    └──────┬──────┘
                                                     Add / Update / Delete
                                                           │
                            ┌──────────────────────────────┼──────────────────────┐
                            ▼                              ▼                      ▼
                        ┌────────┐                   ┌────────┐              ┌─────────┐
                        │ Cache  │                   │ Handler│──── enqueue─►│WorkQueue│
                        │(local) │                   └────────┘              └────┬────┘
                        └────┬───┘                                               │
                             │                                                   │
                             ▼                                                   ▼
                         ┌──────┐                                          ┌──────────┐
                         │Lister│─── read ─────────────────────────────────│  Worker  │
                         └──────┘                                          │(reconcile)
                                                                           └──────────┘
```

- **Reflector** — стримит watch с apiserver, кладёт события в **DeltaFIFO**.
- **Informer** — читает из DeltaFIFO, обновляет **local cache** (thread-safe in-memory store) и вызывает **event handlers**.
- **Handler** — обычно один: положить `key` (`namespace/name`) в **WorkQueue**.
- **Worker** — берёт key из WorkQueue, читает из **Lister** (обёртка над cache), делает reconcile.
- Если reconcile упал — worker кладёт key обратно в очередь с **exponential backoff**.

Смысл этой архитектуры:
1. **Кэш** — не бомбим apiserver read-запросами, читаем из локальной копии.
2. **Rate limiting** — WorkQueue дедуплицирует (много events на один object → один reconcile) и делает backoff при ошибках.
3. **Идемпотентность** — reconcile всегда идёт по текущему состоянию из cache, а не по конкретному событию. Пропустил event — не страшно.

### 4.3 Что нужно знать при написании controller

- **Idempotency**: reconcile может вызваться 100 раз для одного объекта. Действия должны быть безопасны для повторения.
- **Finalizers**: если controller «владеет» внешним ресурсом (создал что-то в облаке), нужно ставить finalizer на объект. При удалении K8s не удалит объект пока finalizer стоит — controller успеет почистить внешнее и снять finalizer.
- **Status updates**: не мешать status в `Reconcile` без ретрая — apiserver может вернуть 409 Conflict (optimistic concurrency через resourceVersion).
- **Deletion timestamp**: объект в стадии удаления имеет `metadata.deletionTimestamp != nil`. Controller должен уметь это обработать.

### 4.4 Reconcile-паттерн примером — Deployment controller

Что происходит когда ты создал Deployment:

1. Deployment controller видит новый Deployment (watch).
2. Читает — нет ReplicaSet для этой версии → создаёт ReplicaSet (`apps/v1/ReplicaSet`).
3. ReplicaSet controller видит новый RS.
4. Читает — нет N Pod'ов с matching labels → создаёт N Pod'ов.
5. Scheduler видит Pod'ы без nodeName → назначает nodeName.
6. Kubelet на ноде видит Pod со своим nodeName → запускает.

Изменил `spec.template.image`:
1. Deployment controller видит изменение.
2. Считает hash template → отличается от текущего RS → создаёт новый RS с новым hash.
3. Скейлит новый RS вверх, старый вниз согласно `strategy.rollingUpdate`.
4. Ждёт readiness новых Pod'ов перед скейл-даун старых.

### 4.5 Собесные

- **«Почему reconcile-паттерн лучше императивного?»** — Level-triggered => самолечится. Пропущенное событие не ломает всё. Плюс идемпотентность → безопасно перезапускать controller.
- **«Что если два controller'а претендуют на один объект?»** — Конфликт (optimistic locking через resourceVersion). Проигравший переретраится.
- **«Как реализовать leader election для controller'а?»** — Через **Lease** объект в K8s API. `client-go/tools/leaderelection`. Только leader делает reconcile, остальные ждут.

---

## 5. kubelet — что реально запускает контейнеры

### 5.1 Роль

Агент на каждой ноде. Функции:
- Регистрирует ноду в apiserver (`Node` object).
- Слушает watch «Pod'ы с `nodeName == myself`».
- Через **CRI** запускает/останавливает контейнеры.
- Управляет volumes (монтирует CSI).
- Запускает probes.
- Отчитывается статус Pod'ов обратно в apiserver.
- Собирает метрики (cAdvisor встроен).

### 5.2 SyncLoop

Главный цикл kubelet:
```
for {
    changes := merge(
        apiserverPodUpdates,   // watch с apiserver
        pleg.events,           // PLEG сообщает о рантайм-изменениях
        housekeeping.tick,     // раз в 2s
        livenessProbeResults,
    )
    for _, pod := range changes {
        syncPod(pod)   // приводит реальность в соответствие с pod spec
    }
}
```

**PLEG (Pod Lifecycle Event Generator)** — периодически (по умолчанию 1 сек) опрашивает container runtime «какие контейнеры сейчас есть», сравнивает с прошлым состоянием, генерирует events типа `ContainerStarted`, `ContainerDied`.

**Собесная деталь**: `PLEG is not healthy` — известное сообщение в /var/log/kubelet.log. Возникает когда container runtime зависает (обычно containerd). Приводит к тому что kubelet не видит новые контейнеры, всё стопорится.

### 5.3 CRI — Container Runtime Interface

Kubelet общается с runtime через **gRPC-протокол CRI**. Ключевые методы:
- `RunPodSandbox` — создать pause-контейнер + все namespace.
- `CreateContainer` / `StartContainer` — запустить рабочий контейнер в существующем sandbox.
- `RemoveContainer` / `StopPodSandbox` — обратное.
- `ListContainers`, `ContainerStatus` — читать состояние.

Runtimes:
- **containerd** — стандарт, легкий.
- **CRI-O** — совместимый, часто в OpenShift.
- **Docker Engine** — **удалён из K8s 1.24+**. Docker никогда не имел CRI, использовался через **dockershim** (обёртка внутри kubelet), теперь удалён.
- **kata-containers** — VMы вместо контейнеров, полная изоляция.
- **gVisor** — user-space sandbox, компромисс.

### 5.4 OCI — Open Container Initiative

Три спецификации:
1. **OCI Image Spec** — формат образа (слои, манифест). Docker image = OCI image.
2. **OCI Runtime Spec** — как запустить контейнер (JSON конфиг + rootfs).
3. **OCI Distribution Spec** — API реестра (docker registry, harbor).

Реальный runtime под capnd — **runc** (Go, reference-implementation) или **crun** (C, faster). Именно они вызывают `clone()`, `setns()`, `unshare()`, `pivot_root()` в Linux для создания контейнера.

### 5.5 Слой vs слой

Цепочка:
```
kubectl apply
      ↓
apiserver
      ↓
etcd
      ↓ (kubelet узнаёт через watch)
kubelet
      ↓ (CRI, gRPC)
containerd
      ↓ (shim + OCI runtime spec)
containerd-shim  ←── долгоживущий процесс на контейнер
      ↓ (fork/exec)
runc
      ↓ (syscalls)
Linux kernel  →  cgroups + namespaces  →  реальный процесс
```

**Зачем shim?** — Чтобы containerd можно было перезапустить, а контейнеры не умерли. Shim держит stdout/stderr, репортит exit code.

### 5.6 Static Pods

Kubelet может запускать Pod'ы, определённые в `--pod-manifest-path` (обычно `/etc/kubernetes/manifests/`). Файлы YAML в этой директории → kubelet сам их запускает, апиserver ничего не знает (но kubelet создаёт **mirror pod** в apiserver для видимости).

Используется для **самого control plane при bootstrap**: kubeadm ставит kubelet, kubelet читает манифесты apiserver'а/etcd/scheduler'а/controller-manager'а из static pod dir → всё поднимается.

Удалить mirror через `kubectl delete` — не работает (kubelet сразу пересоздаст). Надо удалить файл манифеста на ноде.

### 5.7 Проверки здоровья kubelet

- `/healthz` — жив ли kubelet.
- `/metrics` — Prometheus метрики.
- `/pods` — что kubelet считает своими Pod'ами.
- `/stats/summary` — использование ресурсов.

Node становится `NotReady` если kubelet не пингует apiserver дольше `node-monitor-grace-period` (default 40s). После `pod-eviction-timeout` (default 5m) — controller-manager начинает эвиктить Pod'ы с этой ноды.

---

## 6. kube-proxy и Service internals

### 6.1 Что делает kube-proxy

На каждой ноде. Читает Service+EndpointSlice через watch. Настраивает **правила ядра** так, чтобы трафик на ClusterIP шёл на реальный Pod IP.

**Не проксирует в userspace** (в старых версиях умел, deprecated).

### 6.2 Режимы

#### 6.2.1 iptables (default)

Для каждого Service — цепочка iptables. Для каждого backend — правило с DNAT. **Random selection** через probability модуль (`--mode random`).

Пример правил (упрощённо) для Service `10.96.42.15:8080` с двумя Pod'ами `10.244.1.5:8080`, `10.244.2.6:8080`:

```
-A KUBE-SERVICES  -d 10.96.42.15/32 -p tcp --dport 8080 -j KUBE-SVC-X
-A KUBE-SVC-X  -m statistic --mode random --probability 0.5 -j KUBE-SEP-A
-A KUBE-SVC-X  -j KUBE-SEP-B
-A KUBE-SEP-A  -p tcp -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-B  -p tcp -j DNAT --to-destination 10.244.2.6:8080
```

**Проблемы iptables**:
- Правила линейные — при 1000 Service × 3 replica = 3000 правил в цепи. Iptables оценивает O(n), при новом соединении проходит все.
- Обновление всей цепи атомарно (`iptables-restore`) — при 10000 правил это ~сек. Каждый Service update = блокировка kube-proxy.
- Не даёт настоящий L4 LB — только random.

#### 6.2.2 IPVS

Использует Linux IPVS (in-kernel L4 load balancer, изначально для LVS). Хеш-таблицы вместо линейного поиска → O(1). Поддерживает алгоритмы: `rr` (round robin), `lc` (least connections), `dh` (destination hash), `sh` (source hash), `wrr` (weighted).

Плюсы: масштабируется на 10000+ Service. Стабильная latency.
Минусы: сложнее диагностировать (`ipvsadm -Ln`), меньше документации.

Включается через `--proxy-mode=ipvs` в kube-proxy.

#### 6.2.3 nftables (K8s 1.29+)

Более современная замена iptables. Быстрее, лучше атомарность. Ещё не default.

#### 6.2.4 eBPF (Cilium)

Полная замена kube-proxy. Cilium использует eBPF-программы на сетевом стеке ядра, обходит iptables целиком. Плюсы: гораздо быстрее (миллионы Service без замедления), богатая observability. Минусы: сложнее ставить, требует Linux kernel 4.19+.

### 6.3 EndpointSlices — что заменило Endpoints

Раньше был один `Endpoints` object на Service со списком всех Pod IP. При 5000 Pod'ов — object 200KB, каждое обновление шлёт полный watch → серьёзная нагрузка.

`EndpointSlice` (K8s 1.17+) — разбивка на chunks по 100 Pod'ов. Меньше traffic на watch, лучше scaling.

Смотреть:
```bash
kubectl get endpointslices -n knp
kubectl describe endpointslice isnaknpuser-abc-def
```

### 6.4 Внешний трафик — externalTrafficPolicy

Когда Service типа NodePort или LoadBalancer принимает трафик снаружи, есть выбор:
- **externalTrafficPolicy: Cluster (default)** — трафик, попавший на ноду N, может быть перенаправлен на Pod на ноде M. **Source IP теряется** (SNAT), но балансировка равномерная.
- **externalTrafficPolicy: Local** — трафик остаётся на ноде, где принял (только на Pod'ы этой же ноды). **Source IP сохраняется**. Но если на ноде нет Pod'а — 0 backend → внешний LB должен исключать такие ноды через health check.

Для приложений, которым нужен реальный клиентский IP (rate limiting, audit log) — используй `Local` + внешний LB с healthcheck.

### 6.5 Service topology aware routing

**topologyKeys** (deprecated) и его замена **Topology Aware Routing** (K8s 1.21+, `service.kubernetes.io/topology-mode: Auto`) — kube-proxy предпочитает backend'ы в той же zone/regione, что и клиент. Снижает cross-AZ трафик (в AWS это стоит денег).

### 6.6 headless Services

`clusterIP: None`. Kube-proxy не создаёт правил. CoreDNS отдаёт A-записи всех Pod'ов напрямую.

Клиенту — самому решать балансировку. Используется для:
- StatefulSet DNS: `postgres-0.postgres.default.svc.cluster.local`.
- Client-side load balancing (например Spring Cloud LoadBalancer, gRPC).
- Discovery в приложении.

---

## 7. CNI — Container Networking Interface

### 7.1 Модель K8s network

**Требования модели**:
1. Все Pod'ы могут общаться со всеми Pod'ами без NAT.
2. Все Nodes могут общаться со всеми Pod'ами без NAT.
3. IP, который Pod видит у себя, — это IP, который видят другие.

Не диктует **как**, диктует **что**. Реализуют — CNI plugins.

### 7.2 Как работает CNI

Kubelet при создании Pod'а вызывает CNI plugin (binary в `/opt/cni/bin/`, конфиг в `/etc/cni/net.d/`).

Plugin получает:
- Pod namespace (network namespace path).
- Command: `ADD`, `DEL`, `CHECK`.

Возвращает: IP, routes, DNS.

Что делает plugin (упрощённо для `ADD`):
1. Выделить IP из pool (**IPAM**).
2. Создать veth-pair — один конец в host, другой в pod netns.
3. Назначить IP на pod-стороне.
4. Настроить route в pod netns (default gateway).
5. На хосте настроить bridge/route чтобы трафик доходил.

### 7.3 Основные CNI

#### Flannel — overlay через VXLAN

Каждой ноде — CIDR (например `10.244.1.0/24` для node1, `10.244.2.0/24` для node2). Pod на node1 (`10.244.1.5`) шлёт на Pod на node2 (`10.244.2.6`):
1. IP-пакет отправляется через bridge.
2. Флэннел заворачивает в VXLAN (UDP на порту 4789).
3. Пакет летит по physical сети как UDP до node2.
4. На node2 распаковывается, попадает в bridge, доходит до Pod.

Плюсы: работает в любом окружении, не требует поддержки от сети. Минусы: overhead инкапсуляции (~50 bytes на пакет), MTU нужно уменьшать.

#### Calico — BGP (без overlay)

Каждая нода — маршрутизатор. Настраивают BGP peering с соседями. Pod IP анонсируются как отдельные /32 routes. Никакой инкапсуляции — плоская сеть.

Плюсы: производительность как у native. Минусы: требует чтобы underlying сеть пропускала произвольные IP (в облаке — не всегда).

**IPIP mode** — Calico с инкапсуляцией (для случая когда flat не работает).

#### Cilium — eBPF

Использует eBPF на всём: pod-to-pod, service load balancing, NetworkPolicy, observability. Может полностью заменить kube-proxy.

Плюсы: производительность, богатая observability (Hubble). Минусы: Linux kernel 4.19+.

### 7.4 IPAM

Distributes IP addresses to Pods. Варианты:
- **host-local** — каждая нода имеет свой CIDR, назначает локально. Простой.
- **calico-ipam** — глобальный pool с распределением по нодам с блоков.
- **AWS VPC CNI** — использует ENI'ы AWS, каждый Pod получает реальный VPC IP (можно из appflow security groups, PrivateLink и т.д.).

### 7.5 Собесное

- **«Что такое CNI и почему K8s не решает networking сам?»** — CNI = спецификация плагинов. K8s декларирует модель (плоская сеть, no NAT), реализация — под окружение.
- **«Как выбирать CNI?»** — В облаке: обычно провайдер даёт свой (AWS VPC CNI, Azure CNI, GCP netd). On-prem: Calico/Cilium. Если нужна NetworkPolicy — не используй Flannel.
- **«Разница pod-to-pod и pod-to-service?»** — Pod-to-pod: прямая IP-коммуникация через CNI. Pod-to-service: сначала DNS → ClusterIP → kube-proxy iptables DNAT → реальный Pod IP → CNI.

---

## 8. DNS в кластере

### 8.1 CoreDNS

Стандартный DNS-сервер K8s (заменил kube-dns). Работает как Deployment в `kube-system`. Каждый Pod получает через `dnsPolicy: ClusterFirst` (default) настройку `/etc/resolv.conf`:

```
nameserver 10.96.0.10       # ClusterIP CoreDNS
search knp.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### 8.2 Форматы имён

- **Service**: `<service>.<namespace>.svc.cluster.local` (A-record → ClusterIP).
- **Headless Service**: `<service>.<namespace>.svc.cluster.local` (A-records → все Pod IP).
- **Pod DNS в StatefulSet**: `<pod-name>.<service>.<namespace>.svc.cluster.local`.
- **Pod DNS (обычный)**: не рекомендуется, но есть — `<pod-ip-с-дефисами>.<namespace>.pod.cluster.local`.
- **External via ExternalName Service**: возвращает CNAME.

### 8.3 Search domains и ndots

`ndots:5` = если в запрашиваемом имени меньше 5 точек — сначала пробуй с search domains, потом абсолютно.

Пример: `curl isnaknpuser` (0 точек) →
- пробует `isnaknpuser.knp.svc.cluster.local`
- потом `isnaknpuser.svc.cluster.local`
- потом `isnaknpuser.cluster.local`
- потом `isnaknpuser.` (as-is)

Это работает удобно для internal, но замедляет запросы к внешним доменам без FQDN. Оптимизация: указывать полные FQDN (`external-api.com.` с точкой в конце) или `dnsConfig` в Pod spec с `ndots:2`.

### 8.4 Собесное

- **«Почему `curl external.api.com` в Pod медленный?»** — 4 dots минимум, ndots:5 → 4 попытки в search domains перед реальным запросом. Каждая — negative cache/NXDOMAIN.
- **«Как отладить DNS в Pod?»** — `kubectl exec <pod> -- nslookup <name>`, проверить `/etc/resolv.conf`, `kubectl -n kube-system get pods -l k8s-app=kube-dns`.
- **«NodeLocal DNS Cache?»** — DaemonSet CoreDNS на каждой ноде. Pod'ы кешируют локально, снижает нагрузку на центральный CoreDNS и латентность.

---

## 9. Ingress vs Gateway API

### 9.1 Ingress — ветеран

Стандартный ресурс с K8s 1.1. Ограничения:
- HTTP/HTTPS только.
- Все advanced-фичи через **аннотации** (nginx-ingress своё, Traefik своё). Нестандартизированно.
- Нет разделения ролей (кто admin, кто dev).

### 9.2 Gateway API — новый (v1 GA в K8s 1.29)

Три ресурса:
- **GatewayClass** — тип шлюза (nginx, contour, istio). Admin определяет.
- **Gateway** — экземпляр шлюза с listeners. Обычно admin.
- **HTTPRoute** (или **TCPRoute**, **TLSRoute**, **GRPCRoute**) — правила роутинга. Dev.

Плюсы над Ingress:
- Стандартизованные фичи (traffic split, header manipulation) без аннотаций.
- Роль-oriented: admin владеет Gateway, dev пишет HTTPRoute.
- Поддержка не-HTTP протоколов (TCP, TLS passthrough).

Пример HTTPRoute:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: knp-route
spec:
  parentRefs:
  - name: knp-gateway
  hostnames: ["knp.kgd.gov.kz"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/user
    backendRefs:
    - name: isnaknpuser
      port: 8080
      weight: 90       # canary — 10% на v2
    - name: isnaknpuser-v2
      port: 8080
      weight: 10
```

Ingress vs Gateway на 2026 — новые проекты стоит начинать с Gateway API. Старые — постепенно мигрировать.

---

## 10. RBAC

### 10.1 4 объекта

- **Role** — набор permissions в **namespace**.
- **ClusterRole** — набор permissions на весь кластер (или неspaced ресурсы).
- **RoleBinding** — связь Role с subject (User/Group/ServiceAccount) в namespace.
- **ClusterRoleBinding** — связь ClusterRole с subject на всём кластере.

### 10.2 Пример

Role:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: knp
  name: pod-reader
rules:
- apiGroups: [""]           # core group
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]
```

Binding:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: knp
  name: pod-reader-binding
subjects:
- kind: ServiceAccount
  name: monitoring
  namespace: monitoring
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Смысл: SA `monitoring` из namespace `monitoring` может читать Pod'ы в namespace `knp`.

### 10.3 ServiceAccount и Pod

Каждому Pod'у можно указать SA (`spec.serviceAccountName`). Kubelet вмонтирует JWT-токен этого SA в `/var/run/secrets/kubernetes.io/serviceaccount/token`. Приложение читает и использует для запросов к apiserver.

**Bound Service Account Tokens** (K8s 1.22+): токены короткоживущие (1 час), ротируются автоматически через `TokenRequest` API + projected volume. До 1.22 были долгоживущие Secrets — уязвимость.

### 10.4 Default SA — гоча

Если не указать `serviceAccountName`, Pod получает SA `default` из своего namespace. По умолчанию `default` SA НИКАКИХ прав не имеет. Но многие legacy-приложения предполагают что могут читать apiserver — и падают.

Best practice: **явно указывать SA для каждого Deployment**, минимальные permissions. Никогда не давать `cluster-admin` кроме как для настоящих admin-tools.

### 10.5 aggregate ClusterRole

Можно создать `ClusterRole` с `aggregationRule` — он собирает permissions из других ClusterRole по label. Позволяет operator'ам расширять существующие роли:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
```

Все permissions этой роли автоматически попадают в стандартную `view`.

---

## 11. Admission controllers

### 11.1 Что это

Плагины, работающие после auth/authz, до записи в etcd. Два типа:
- **Mutating** — могут менять объект.
- **Validating** — только да/нет.

Порядок: сначала все mutating (в порядке имён), потом все validating.

### 11.2 Built-in

Примеры встроенных (несколько десятков):
- **NamespaceLifecycle** — нельзя создавать в удаляющемся namespace.
- **LimitRanger** — применяет LimitRange (default requests/limits).
- **ServiceAccount** — вмонтирует default SA если не указан.
- **DefaultStorageClass** — для PVC без класса ставит default.
- **PersistentVolumeClaimResize** — контроль изменения PVC size.
- **PodSecurity** — Pod Security Standards (заменил PSP).
- **ResourceQuota** — проверка ResourceQuota в namespace.

### 11.3 Dynamic (webhook) admission

Пишешь свой HTTP-сервис. Регистрируешь через `MutatingWebhookConfiguration` / `ValidatingWebhookConfiguration`. Apiserver для каждого запроса шлёт webhook AdmissionReview, тот отвечает allow/deny (или patched object для mutating).

Пример правила webhook'а:
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: image-policy
webhooks:
- name: images.example.com
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["pods"]
  clientConfig:
    service:
      name: image-policy
      namespace: policy
      path: /validate
    caBundle: <base64>
  failurePolicy: Fail
  sideEffects: None
  timeoutSeconds: 5
  admissionReviewVersions: ["v1"]
```

`failurePolicy: Fail` — если webhook недоступен, запрос **отклоняется**. `Ignore` — пропускается (небезопасно для security webhook'ов).

**Опасность**: если webhook смотрит на все Pod'ы (включая kube-system) и падает — control plane может не подняться после рестарта. Best practice: `namespaceSelector` исключающий системные ns.

### 11.4 OPA / Gatekeeper и Kyverno

Instead писать webhook на Go — используем policy engines:

**OPA Gatekeeper** — на языке Rego. Мощно, но кривая обучения.
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pod-must-have-owner
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    labels: ["owner"]
```

**Kyverno** — на языке YAML, проще:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: enforce
  rules:
  - name: check-owner-label
    match:
      resources:
        kinds: [Pod]
    validate:
      message: "Pod must have owner label"
      pattern:
        metadata:
          labels:
            owner: "?*"
```

Kyverno — легче начать. OPA — если политики шире K8s.

### 11.5 Pod Security Admission (PSA)

Заменил PodSecurityPolicy (deprecated в 1.21, удалён в 1.25). Три уровня:
- **privileged** — всё разрешено.
- **baseline** — минимальные ограничения (нельзя hostNetwork, hostPID).
- **restricted** — строгий (RunAsNonRoot, dropCapabilities ALL, readOnlyRootFilesystem, seccomp).

Включается через label на namespace:
```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

`enforce` — блокирует. `warn` — предупреждение в CLI. `audit` — audit log.

---

## 12. CSI — Container Storage Interface

### 12.1 Архитектура

CSI = стандартный API между K8s и провайдерами хранилищ. Раньше были in-tree drivers (AWS EBS, GCE PD внутри K8s), сейчас — все вынесены в отдельные CSI drivers.

Три компонента CSI driver:
1. **CSI Controller** — Deployment/StatefulSet с 1-2 replicas. Общается с cloud API. Создаёт/удаляет volumes.
2. **CSI Node** — DaemonSet на каждой ноде. Attach/mount volumes к Pod'ам.
3. **Sidecars от K8s**: `external-provisioner`, `external-attacher`, `external-resizer`, `external-snapshotter`, `node-driver-registrar`.

### 12.2 Жизненный цикл PVC

1. `kubectl apply -f pvc.yaml` — создаётся PVC (Pending).
2. **external-provisioner** видит: PVC без PV → зовёт CSI Controller `CreateVolume` → создаётся реальный disk в cloud → создаётся PV → bound с PVC.
3. Pod с этим PVC назначается на ноду.
4. **external-attacher** зовёт CSI Controller `ControllerPublishVolume` → cloud API attach'ит disk к VM ноды.
5. **CSI Node** зовёт `NodeStageVolume` (format, mkfs) → `NodePublishVolume` (mount в Pod).
6. Pod стартует, видит volume.

### 12.3 Volume snapshots

CSI поддерживает snapshots (`VolumeSnapshot` object). Через `external-snapshotter`:
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: pg-backup-2026-01-15
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: pg-data
```

Из snapshot можно создать новый PVC — быстрое восстановление БД.

### 12.4 Practical gotchas

- **RWO + StatefulSet**: PVC привязан к одной ноде. Если нода умерла — Pod pending, пока K8s не decommission старую и attach на новую. Cloud attach/detach — ~2 минуты.
- **fsGroup** в Pod securityContext — chown-ит volume при mount. На больших volumes — очень медленно. Fix: `fsGroupChangePolicy: OnRootMismatch` (K8s 1.23+) — chown только если корень не совпадает.
- **VolumeAttachment stuck**: при NodeNotReady VolumeAttachment висит, новый Pod с PVC — Pending. Помогает `kubectl delete volumeattachment` или ждать TIMEOUT (6 минут).

---

## 13. Autoscaling глубже

### 13.1 HPA — algorithm

Формула масштабирования (для CPU-based):
```
desiredReplicas = ceil(currentReplicas * (currentMetric / targetMetric))
```

Пример: `replicas=3`, CPU utilization 90%, target 60% →
`desired = ceil(3 * 90/60) = ceil(4.5) = 5`.

Скейл делается каждые 15 сек. **Tolerance 10%** — если currentMetric/targetMetric в пределах [0.9, 1.1] — ничего не делаем (защита от флаппинга).

Custom metrics (RPS, queue length) — через `metrics.k8s.io/v1beta1` (resource metrics) или `custom.metrics.k8s.io/v1beta1` (Prometheus adapter).

### 13.2 HPA behavior

С K8s 1.18 — `behavior` для тонкой настройки:
```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0    # мгновенно
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300  # 5 минут откладываем
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

Читается: «scale up — за 15 сек можем удвоить; scale down — за минуту можем убрать 10%, но только после 5 минут стабильно низкой нагрузки».

### 13.3 VPA — Vertical Pod Autoscaler

Меняет requests/limits Pod'ов на основе исторического использования. Компоненты:
- **Recommender** — анализирует metrics, генерирует рекомендации.
- **Updater** — evicts Pod'ы если они сильно отклоняются от рекомендации.
- **Admission Controller** — на новых Pod'ах ставит recommended values.

Modes:
- **Off** — только рекомендации (посмотреть).
- **Initial** — только при создании нового Pod'а.
- **Auto** — evicts и пересоздаёт с новыми requests. Не для критичных! Downtime.

**HPA + VPA on same metric** — конфликт (оба хотят реагировать на CPU). Или разделяй метрики (HPA на custom, VPA на CPU), или используй только один.

### 13.4 Cluster Autoscaler

Добавляет/убирает Node'ы кластера (cloud-специфично).

- Смотрит: есть Pod'ы Pending из-за insufficient resources → добавляет Node (через AWS ASG / GCE MIG / Azure VMSS).
- Смотрит: Node underutilized (long time) → cordon + drain + delete.

Каждый Pod с `nodeSelector` / affinity — на Cluster Autoscaler разбирается: подходит ли новая Node.

### 13.5 Karpenter (AWS)

Заменяет Cluster Autoscaler на AWS. Плюсы:
- Быстрее (не ждёт ASG).
- Может провизить разные instance types on-demand под потребности.
- Меньше конфигурации.

### 13.6 KEDA — Event-Driven Autoscaling

Расширяет HPA scaling'ом на основе событий из внешних систем:
- Длина очереди в RabbitMQ / Kafka.
- Количество сообщений в SQS.
- Custom Prometheus queries.

Позволяет масштабировать до 0 replicas (HPA — минимум 1).

Пример KEDA ScaledObject для консюмера Kafka:
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer
spec:
  scaleTargetRef:
    name: kafka-consumer
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: my-group
      topic: events
      lagThreshold: "100"
```

При lag>100 сообщений — добавляет Pod'ы.

---

## 14. CRD и Operator Pattern

### 14.1 CRD

Кастомный тип ресурса. Даёт возможность расширять K8s API объектами домена.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.knp.io
spec:
  group: knp.io
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames: [db]
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [engine, size]
            properties:
              engine:
                type: string
                enum: [postgres, mysql]
              version:
                type: string
              size:
                type: string
          status:
            type: object
            properties:
              phase:
                type: string
    subresources:
      status: {}
      scale:
        specReplicasPath: .spec.replicas
        statusReplicasPath: .status.replicas
```

После apply — можно писать `Database` объекты. Но без controller'а они просто лежат в etcd. Обычно рядом с CRD пишут **operator** — controller, знающий что делать.

### 14.2 Operator Pattern

**Operator = CRD + Controller** (обычно один binary). Инкапсулирует expertise по конкретному приложению.

Примеры:
- **postgres-operator** (Zalando, Crunchy) — управляет Postgres-кластерами: HA через Patroni, backup в S3, PIT restore.
- **kafka-operator** (Strimzi) — Kafka+Zookeeper кластеры.
- **elasticsearch-operator** — ES кластеры + snapshot policies.
- **prometheus-operator** — Prometheus + Alertmanager + ServiceMonitor.
- **cert-manager** — сертификаты Let's Encrypt.

### 14.3 Kubebuilder / operator-sdk

Frameworks для написания operator'ов. Дают:
- Scaffolding (генерация boilerplate).
- Обёртка над client-go: работа с типизированными клиентами.
- Отладку через `make run` локально против кластера.
- Deploy как manager Pod.

Реальный код reconciler примерно:
```go
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var db knpv1.Database
    if err := r.Get(ctx, req.NamespacedName, &db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Проверить наличие StatefulSet
    var sts appsv1.StatefulSet
    err := r.Get(ctx, req.NamespacedName, &sts)
    if apierrors.IsNotFound(err) {
        // Создать StatefulSet под spec
        newSts := buildStatefulSet(&db)
        if err := controllerutil.SetControllerReference(&db, newSts, r.Scheme); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{}, r.Create(ctx, newSts)
    }

    // Проверить что spec совпадает — обновить если нет.
    // Обновить status.
    return ctrl.Result{}, nil
}
```

### 14.4 Levels of Operator Maturity

CoreOS определял 5 уровней (пирамида):
1. **Basic install** — Deploy CRD, StatefulSet.
2. **Seamless upgrades** — Rolling update, миграции.
3. **Full lifecycle** — Backup/restore, failover.
4. **Deep insights** — Metrics, alerts, tracing.
5. **Auto pilot** — Self-tuning, healing, capacity planning.

---

## 15. HA control plane

### 15.1 Stacked vs external etcd

**Stacked** (`kubeadm init --control-plane-endpoint=...`): etcd работает на тех же нодах, что и apiserver/scheduler/controller-manager. 3 master нод = 3 etcd + 3 apiserver.

Плюсы: меньше нод, проще.
Минусы: смерть control plane ноды = смерть etcd члена одновременно.

**External etcd**: отдельный кластер из 3-5 нод etcd, control plane отдельно. Промышленный стандарт для крупных кластеров.

### 15.2 Как выглядит HA

```
                    ┌──────── HA LoadBalancer (haproxy/AWS ELB) ────────┐
                    │  vip: control-plane.example.com:6443              │
                    └───┬──────────────┬────────────────┬───────────────┘
                        │              │                │
                    ┌───▼───┐      ┌───▼───┐        ┌───▼───┐
                    │master1│      │master2│        │master3│
                    │apiserv│      │apiserv│        │apiserv│
                    │sched  │      │sched  │        │sched  │
                    │cm     │      │cm     │        │cm     │
                    │etcd   │      │etcd   │        │etcd   │
                    └───────┘      └───────┘        └───────┘
                        ▲              ▲                ▲
                        │              │                │
                        └──────────────┴────────────────┘
                             etcd Raft cluster
```

- **apiserver** — stateless, все три активны. LB балансирует.
- **scheduler** и **controller-manager** — **leader-election**: только один активен в момент времени (через `Lease` object).
- **etcd** — Raft, один leader, все три реплицируют.

### 15.3 Kubelet и control-plane смерть

Что происходит если весь control plane упал:
- Kubelet продолжает работать (у него локальный кэш podspec'ов).
- Существующие Pod'ы бегут, Service работает (kube-proxy уже настроил iptables).
- Умрёт Pod → kubelet **не сможет** его переpull'ить (нет apiserver для чтения image auth) но попытается запустить из локального image.
- Новые деплои не идут.

Вывод: краткий сбой control plane — не катастрофа. Долгий (часы) — проблема.

---

## 16. Обзор pod lifecycle

### 16.1 Фазы

- **Pending** — apiserver принял, но контейнеры не стартовали (Pending schedule, ImagePull, InitContainer running).
- **Running** — хотя бы один контейнер запущен.
- **Succeeded** — все контейнеры завершились с exit 0 (Job).
- **Failed** — все контейнеры завершились, хоть один с non-zero exit.
- **Unknown** — kubelet потерял связь с apiserver.

### 16.2 Container states

Внутри Pod каждый контейнер имеет state:
- **Waiting** — с `reason`: `ContainerCreating`, `ImagePullBackOff`, `CrashLoopBackOff`, `CreateContainerConfigError`.
- **Running**.
- **Terminated** — с `reason`: `Completed`, `Error`, `OOMKilled`.

Проверять:
```bash
kubectl get pod X -o jsonpath='{.status.containerStatuses[*].state}'
```

### 16.3 Restart policy

- **Always** (default для Pod из Deployment) — контейнер всегда перезапускается.
- **OnFailure** — только если non-zero exit.
- **Never** — не перезапускать.

Из Deployment/ReplicaSet — всегда `Always`. Job — `OnFailure` или `Never`. CronJob — `OnFailure`.

### 16.4 CrashLoopBackOff — что это

Kubelet видит: контейнер упал → рестарт → упал → рестарт → упал… Ставит **exponential backoff**: 10s, 20s, 40s, 80s, 160s, потом capped 5 min.

Ты видишь `STATUS: CrashLoopBackOff` — это не отдельное состояние, это способ kubelet'а сказать «я замедляю рестарты».

Отлаживать:
```bash
kubectl logs X --previous  # логи последнего упавшего инстанса
kubectl describe pod X     # события, exit code
```

### 16.5 ImagePullBackOff

Runtime не может скачать image:
- Неверный tag / typo.
- Нет доступа к registry (private registry без `imagePullSecrets`).
- Registry down.
- Rate limiting (Docker Hub).

`kubectl describe pod X | grep -A5 Events` → увидишь конкретную ошибку.

---

## 17. Практический debug

### 17.1 Не поднимается Pod

Алгоритм:
```bash
# 1. Что говорит K8s
kubectl describe pod X -n ns
# Смотри секцию Events внизу.
# Причины: FailedScheduling / FailedCreatePodSandbox / ImagePullBackOff / FailedMount / Error

# 2. Если статус Running но crash — логи
kubectl logs X -n ns
kubectl logs X -n ns --previous  # если рестартнулся

# 3. Exec если ещё жив
kubectl exec -it X -n ns -- sh

# 4. Ephemeral container для крашнутого
kubectl debug -it X --image=busybox --target=app -n ns
```

### 17.2 Ephemeral containers (K8s 1.23 GA)

Раньше — если контейнер упал из distroless-образа, ты не мог `kubectl exec` (нет shell). Теперь можно:
```bash
kubectl debug -it problem-pod --image=busybox --target=app --share-processes
```

Добавляется временный контейнер в существующий Pod, шарит namespace с целевым — можно смотреть процессы, файлы, сеть.

### 17.3 Не идёт трафик до Service

```bash
# 1. Endpoints есть?
kubectl get endpoints svc-name -n ns
# Если пусто — все Pod'ы NotReady или selector не совпадает

# 2. Селектор совпадает?
kubectl get svc svc-name -n ns -o yaml | grep selector
kubectl get pods -n ns --show-labels

# 3. Readiness работает?
kubectl describe pod X -n ns | grep -A5 Readiness

# 4. Проверить с ноды kube-proxy правила
# на ноде:
iptables-save | grep <clusterIP>

# 5. Из другого Pod'а
kubectl run -it --rm debug --image=nicolaka/netshoot -- sh
# curl svc-name.ns.svc.cluster.local
# nslookup svc-name.ns
```

### 17.4 Ноды NotReady

```bash
kubectl get nodes
kubectl describe node worker-3
# Смотри Conditions:
#   MemoryPressure
#   DiskPressure
#   PIDPressure
#   Ready

# На ноде — kubelet живой?
ssh worker-3
systemctl status kubelet
journalctl -u kubelet --since "10 minutes ago"

# containerd жив?
systemctl status containerd
crictl ps  # что-то показывает?
```

### 17.5 Полный кластер не отвечает

- **apiserver умер** — `kubectl` возвращает `connection refused`. Проверь `systemctl status kube-apiserver` на master нодах.
- **etcd упал** — apiserver жив, но каждый запрос возвращает `etcdserver: request timed out`. `etcdctl endpoint status` покажет.
- **DNS сломан** — pod-to-pod через ClusterIP работает, но `curl svc-name` не резолвит. Проверь `kubectl -n kube-system get pods -l k8s-app=kube-dns`.

---

## 18. Cgroups и namespaces — что реально изолирует контейнер

Контейнер — это НЕ виртуалка. Это обычный процесс Linux, с ограничениями через **namespaces** (что видит) и **cgroups** (сколько может).

### 18.1 Namespaces

Ядерные конструкции, каждый процесс принадлежит нескольким namespace'ам одного типа:

- **PID** — свой набор процессов, `ps` внутри контейнера видит только «свои». PID 1 в контейнере — главный процесс.
- **NET** — свой сетевой стек: интерфейсы, IP, iptables, sockets.
- **MNT** — своя иерархия монтирования (файловая система).
- **UTS** — hostname, domain.
- **IPC** — свои очереди сообщений, semaphores.
- **USER** — свой UID/GID mapping. Root в контейнере — не root на хосте (если настроено).
- **CGROUP** — своё представление cgroups.
- **TIME** (Linux 5.6+) — свои clocks.

Pod = процессы разделяющие **NET**, **IPC**, **UTS**, **[USER]** — но не PID и не MNT (у каждого контейнера свои).

### 18.2 Cgroups

Контроллер (v1 или v2 в новых kernel) ограничивает потребление ресурсов:
- **cpu** — CPU shares, CFS quota, throttling.
- **memory** — hard limit, soft limit, OOM.
- **pids** — max processes.
- **io** — throttle block I/O.
- **hugetlb**, **cpuset** — специфические.

`limits.cpu: "2"` → cgroup CFS quota: `cpu.cfs_quota_us=200000`, `cpu.cfs_period_us=100000` → максимум 200% CPU за период (2 полных ядра).

**Throttling** — если превышаешь quota в моменте, приложение **тормозится** (не kill). У Java это болезненно (GC не может доработать). Метрика — `container_cpu_cfs_throttled_periods_total` в Prometheus.

`limits.memory: 1Gi` → cgroup `memory.limit_in_bytes = 1073741824`. При превышении — OOM killer в ядре убивает процесс из этого cgroup.

### 18.3 Container image — что это на диске

OCI image = набор **слоёв** (tar.gz), плюс манифест (JSON). Каждый слой — diff файлов относительно предыдущего.

При запуске containerd:
1. Раскладывает слои в overlay filesystem (`/var/lib/containerd/...`).
2. Создаёт writable layer сверху (`upperdir` в overlay).
3. Монтирует в mount namespace контейнера.

Копирование файла в контейнер — не копирует image, копирует только diff в upperdir. Удаление файла из image — создаёт **whiteout** в upperdir.

### 18.4 Секции безопасности в Pod

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 2000
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]  # если нужен порт <1024
  seccompProfile:
    type: RuntimeDefault
```

- `runAsUser` — UID процесса.
- `readOnlyRootFilesystem` — rootfs mount read-only, только `/tmp` и volumes writable.
- `capabilities.drop: [ALL]` — снять все Linux capabilities (network admin, mknod, ptrace...). Обычно оставляют только `NET_BIND_SERVICE` если нужен privileged port.
- `seccomp` — фильтр системных вызовов. `RuntimeDefault` — containerd имеет default seccomp profile.
- `allowPrivilegeEscalation: false` — `setuid` binary не сможет получить root.

Best practice: **всё вышеперечисленное в prod**. PSA `restricted` требует именно этих настроек.

---

## 19. Deployment strategies подробно

### 19.1 RollingUpdate

Уже разобрано в 10 и 77. Ключевые параметры:
- `maxSurge`: сколько сверху можно поднять (по умолчанию 25%).
- `maxUnavailable`: сколько может быть недоступно (по умолчанию 25%).

**Для zero-downtime prod**: `maxSurge: 1`, `maxUnavailable: 0`. Никогда не убьём Pod, не подняв нового.

**Progressive rollout** — `.spec.progressDeadlineSeconds` (default 600s). Если за это время новая ревизия не стала Ready — Deployment помечается как `Progressing=False, Reason=ProgressDeadlineExceeded`. Автоматически НЕ откатывается — нужно руками `kubectl rollout undo`.

### 19.2 Recreate

```yaml
strategy:
  type: Recreate
```

Убивает все старые, потом поднимает новые. **Downtime гарантирован**. Используется:
- Когда старая и новая версия не могут работать одновременно (изменение схемы БД в inline-режиме).
- Job-like приложения.
- Dev/staging.

### 19.3 Blue/Green (нативно не поддерживается)

Реализация вручную:
1. Deployment `app-blue` — v1, Service `app` селектит `version: blue`.
2. Создать Deployment `app-green` — v2.
3. Ждать пока `app-green` полностью Ready.
4. Переключить Service selector на `version: green`.
5. Понаблюдать. Если проблемы — сменить обратно (мгновенный rollback).
6. Удалить `app-blue`.

Плюсы: мгновенное переключение, легко откатить.
Минусы: 2× ресурсов в момент переключения, скачок в БД миграциях сложен.

### 19.4 Canary

Реализация вручную:
1. Deployment `app-stable` с 9 replicas.
2. Deployment `app-canary` с 1 replica, тот же label `app: app`.
3. Один Service селектит по `app: app` → трафик автоматически 10% canary / 90% stable (пропорционально replicas).
4. Мониторим error rate. Если ОК — увеличиваем canary до 100%, убираем stable.

**Проблема**: балансировка на уровне replicas — грубо. Нельзя дать 5% канарею.

### 19.5 Argo Rollouts / Flagger

Продвинутые CRD, дающие настоящие blue/green и canary с автоанализом.

**Argo Rollouts** (замена Deployment):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 5
      - pause: {duration: 5m}
      - setWeight: 20
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {}     # ждать ручного promote
      canaryService: app-canary
      stableService: app-stable
      trafficRouting:
        istio:
          virtualService: {name: app-vs}
      analysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: app-canary
```

Плюсы: 5% → 20% → 50% → 100% с pause и автоанализом Prometheus queries. Автоматический rollback при деградации метрик.

**Flagger** — конкурент с похожими фичами, интегрируется с Istio/Linkerd/nginx-ingress.

Промышленно — стандарт для serious canary.

---

## 20. Многокластерные паттерны (кратко)

### 20.1 Зачем много кластеров

- Изоляция по регионам (compliance, latency).
- Blast radius: один упал — другой жив.
- Разные версии K8s на разных стадиях.
- Prod / staging / dev как отдельные кластеры.

### 20.2 Инструменты

- **Kubefed** (deprecated) — федерация ресурсов.
- **Karmada** — новый CNCF-проект, federation with multi-cluster scheduling.
- **Cluster API (CAPI)** — declarative provisioning кластеров как CRD.
- **ArgoCD ApplicationSet** — деплой одного приложения в N кластеров из одного git repo.
- **Istio multi-cluster mesh** — Service Mesh поверх кластеров.

Для большинства команд — **не начинай с multi-cluster**. Один хороший кластер лучше чем два плохих.

---

## 21. Секреты и Vault

### 21.1 Проблема K8s Secret

- Base64 в etcd.
- Cluster-admin читает всё.
- Нет ротации.
- Нет аудита кто читал.

### 21.2 Решения

**Encryption at rest** — уже разобрано (KMS integration).

**Sealed Secrets** (Bitnami) — шифруешь Secret в git через public key, кластер расшифровывает при apply приватным. Плюс — можно коммитить в git. Минус — ротация ключа тяжёлая.

**External Secrets Operator** — CRD `ExternalSecret`, синхронизирует из внешних хранилищ (Vault, AWS Secrets Manager, GCP Secret Manager) в K8s Secret. Актуальный best practice.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault
    kind: SecretStore
  target:
    name: db-secret
  data:
  - secretKey: password
    remoteRef:
      key: secret/data/knp/db
      property: password
```

**Vault Agent Injector** — sidecar от HashiCorp, инжектит секреты в Pod'ы напрямую без создания K8s Secret. Через init container fetch, mount на emptyDir.

### 21.3 K8s auth в Vault

Vault знает про K8s: SA JWT-токен Pod'а → Vault валидирует у K8s TokenReview API → выдаёт Vault-токен. Pod читает секреты по SA identity, не по паролю.

Конфиг в Vault:
```
vault write auth/kubernetes/config \
    token_reviewer_jwt="$SA_JWT" \
    kubernetes_host=https://apiserver \
    kubernetes_ca_cert=@ca.crt

vault write auth/kubernetes/role/knp-app \
    bound_service_account_names=isnaknpuser \
    bound_service_account_namespaces=knp \
    policies=knp-app-policy \
    ttl=1h
```

---

## 22. Observability essentials

### 22.1 Что мониторить

- **Node** — CPU/mem/disk/network per node.
- **Pod** — CPU/mem/restart count.
- **Container** — throttling, OOMKilled.
- **kubelet** — pleg latency, sync duration.
- **apiserver** — request rate/latency/errors.
- **etcd** — WAL fsync latency, DB size, leader elections.
- **Scheduler** — scheduling latency, pending pods.

### 22.2 Метрики

- **metrics-server** — базовые CPU/mem, `kubectl top` использует.
- **kube-state-metrics** — состояние объектов (Deployment replicas, Pod phase, PVC size).
- **cAdvisor** — встроен в kubelet, метрики контейнеров.
- **node-exporter** — метрики Linux (system).

Всё → **Prometheus** → **Grafana**.

### 22.3 Логи

Стандарт: log stdout/stderr → kubelet перекладывает в файлы (`/var/log/pods/...`) → DaemonSet агент собирает.

Агенты:
- **Fluent Bit** — лёгкий, C, стандарт.
- **Fluentd** — Ruby, старше, тяжелее.
- **Vector** — новый, Rust.
- **Promtail** — для Loki (легковесный log store).

Отправляют → **Elasticsearch/Loki/CloudWatch/BigQuery** → **Kibana/Grafana**.

### 22.4 Трейсинг

- **OpenTelemetry** — стандарт SDK и Collector.
- **Jaeger** / **Tempo** / **Zipkin** — trace store + UI.

Distributed traces важны для debugging «где тормозит» в микросервисах.

### 22.5 Events

К8s events — короткоживущие (1 час по умолчанию). Полезны для дебага в моменте:
```bash
kubectl get events -n ns --sort-by='.lastTimestamp'
kubectl get events --field-selector involvedObject.name=X -n ns
```

Долгосрочное хранение — через **kube-events-exporter** → Loki/Elasticsearch.

---

## 23. Best practices на прод

### 23.1 Обязательный минимум для каждого Deployment

- `replicas ≥ 2` (лучше 3).
- `resources.requests` + `resources.limits` (Guaranteed QoS).
- `livenessProbe` + `readinessProbe` (правильно разделённые!).
- `startupProbe` для медленных Spring Boot.
- `terminationGracePeriodSeconds` > `spring.lifecycle.timeout-per-shutdown-phase`.
- `preStop` hook: `sleep 5-10` для race-free graceful.
- `PodDisruptionBudget` с `minAvailable: replicas - 1`.
- `podAntiAffinity` по `hostname` или лучше `zone`.
- `priorityClassName`.
- `nodeSelector` или tolerations (если специфичные ноды).
- `securityContext`: `runAsNonRoot`, `readOnlyRootFilesystem`, `capabilities.drop: [ALL]`.
- `imagePullPolicy: IfNotPresent` (не Always — пылит рестарт).
- Метки на всё для `kubectl get -l ...` и для NetworkPolicy.
- `annotations` для prometheus scraping.

### 23.2 Nice-to-have

- Sidecar Envoy/Istio для mTLS и observability.
- NetworkPolicy для namespace default-deny.
- ImagePullSecrets из ExternalSecrets.
- ArgoCD/Flux для GitOps deploy.
- Argo Rollouts для canary.

### 23.3 Anti-patterns

- **`latest` tag** в image. Каждый Pod может подтянуть разное. Пиши immutable теги.
- **Один Pod = много контейнеров без причины**. Всё что можно разделить — разделяй.
- **hostNetwork / hostPID / privileged** в prod — только для инфры (kube-proxy, node-exporter).
- **BestEffort QoS** — первые кандидаты на eviction под давлением.
- **replicas: 1** для критичных сервисов.
- **Отсутствие resource limits** — один Pod съест ноду.
- **Отсутствие readiness** — rolling update отправит трафик в непрогретое приложение.

---

## 24. Собесные вопросы с полными ответами

### 24.1 Базовое (обязано отскакивать от зубов)

**Q1: Что такое Pod и почему это не контейнер?**

Pod — минимальная единица планирования в K8s. Обычно 1 контейнер, но может быть несколько (sidecar). Контейнеры в Pod'е шарят network namespace (один IP, доступ по localhost), IPC, volumes; **не шарят** MNT (свои файловые системы) и PID (по умолчанию).

Не контейнер, потому что K8s часто нужно запускать связанные процессы вместе (application + envoy proxy, application + log-forwarder). Уровень абстракции выше контейнера.

**Q2: Разница Deployment и StatefulSet?**

Deployment — для stateless приложений: replicas взаимозаменяемы, случайные имена (`app-abc123-xyz`), любой Pod может обрабатывать любой запрос. Rolling update — параллельный.

StatefulSet — для stateful: стабильные имена (`app-0`, `app-1`, `app-2`), стабильный DNS (`app-0.svc.ns.svc.cluster.local`), стабильный PVC на каждый Pod, порядковый старт/остановка. Используется для БД, брокеров, кластерных приложений.

**Q3: Разница liveness и readiness probe?**

Liveness: «жив ли контейнер». Fail → kubelet **перезапускает** контейнер. Используется когда приложение может залипнуть (deadlock, зациклилось).

Readiness: «готов ли принимать трафик». Fail → Pod исключается из **endpoints Service'а**, трафик не идёт, но Pod не рестартится. Используется для «БД временно недоступна», «прогрев кэша».

Общее правило: liveness не должен зависеть от внешних систем (иначе флап БД → cascading restart). Readiness может.

**Q4: Что такое Service и как он работает?**

Service — стабильный VIP + балансировка на группу Pod'ов по label selector.

Работает через **kube-proxy** на каждой ноде. Kube-proxy читает Service и EndpointSlices через watch. Настраивает iptables (или IPVS) правила: пакет на ClusterIP:port → DNAT на один из Pod IP. Балансировка round-robin/random.

DNS: CoreDNS резолвит `<service>.<namespace>.svc.cluster.local` в ClusterIP.

### 24.2 Средне

**Q5: Что происходит при `kubectl apply -f deploy.yaml`?**

1. kubectl вычисляет diff с текущим объектом в etcd (через 3-way merge).
2. Отправляет PATCH на apiserver.
3. Apiserver: auth → authz (RBAC) → mutating admission → validation → validating admission → etcd write.
4. Возвращает результат клиенту.
5. Deployment controller видит новую версию через watch → reconcile:
   - Если spec.template отличается — создаёт новый ReplicaSet, начинает rolling update.
   - Скейлит старый RS вниз, новый вверх согласно `strategy.rollingUpdate`.
6. ReplicaSet controller создаёт Pod'ы.
7. Scheduler назначает nodeName.
8. Kubelet на ноде запускает через CRI.

**Q6: Что произойдёт если Pod превысит memory limit?**

**OOMKilled** — Linux OOM killer в ядре убивает процесс из cgroup. Exit code 137 (128 + 9 SIGKILL). Kubelet видит exit → рестартит согласно restart policy (обычно Always). Если рестарты частые — CrashLoopBackOff, exponential backoff.

Проверить: `kubectl describe pod X | grep -A5 "Last State"` — увидишь `Reason: OOMKilled`.

Отладить: heap dump через `jcmd 1 GC.heap_dump /tmp/heap.hprof`, скопировать через `kubectl cp`, анализировать в Eclipse MAT.

Fix: увеличить limit + правильно настроить `-Xmx` (~70% от limit).

**Q7: Что произойдёт если Pod превысит CPU limit?**

**Throttling**, не kill. Linux CFS scheduler ограничивает CPU time. Приложение работает медленнее, но живёт.

Особенно болезненно для JVM: GC не может отработать в отведённом окне → длинные паузы. Метрика: `container_cpu_cfs_throttled_periods_total`.

Часто рекомендуется **не ставить CPU limit вообще**, только request (гарантия). Пусть Pod жрёт свободные CPU когда они есть.

**Q8: Как работает Rolling Update?**

Deployment имеет `strategy.type: RollingUpdate` с `maxSurge` и `maxUnavailable`.

1. При смене `spec.template` (например image) — Deployment controller создаёт новый ReplicaSet (RS) с новым template hash.
2. Начинает: скейлить новый RS вверх, старый вниз.
3. Балансирует так, чтобы `Ready pods ≥ replicas - maxUnavailable` и `Total pods ≤ replicas + maxSurge`.
4. Ждёт readiness каждого нового Pod'а перед скейл-даун старого.
5. Продолжает пока новый RS не имеет `replicas` подов.

Rollback: `kubectl rollout undo` — Deployment восстанавливает старый RS (он оставался с 0 replicas), скейлит вверх, новый вниз.

**Q9: Как безопасно катить breaking-change API?**

1. **Deploy backward-compatible изменения** в существующий сервис. Например: добавить новое поле, но старое ещё поддерживать.
2. Мигрировать всех потребителей на новое API.
3. **Deploy financial change** — убрать старую совместимость.

Это классический **expand-contract pattern** (или **parallel change**).

С БД:
1. Добавить новую колонку/таблицу.
2. Deploy код который пишет и в старое и в новое.
3. Backfill исторических данных.
4. Deploy код который читает из нового.
5. Deploy код который не пишет в старое.
6. Удалить старую колонку.

Никогда одним PR: `изменил схему` → `изменил код` в одном деплое.

**Q10: Что такое ConfigMap и как его подхватывает Spring Boot?**

ConfigMap — объект K8s с key/value конфигами. Можно смонтировать как volume (файлы) или env-variables.

Как volume:
```yaml
volumeMounts:
- name: config
  mountPath: /app/config
volumes:
- name: config
  configMap:
    name: knp-config
```

Spring Boot по умолчанию читает `/app/config/application.yml` (или `application-<profile>.yml` для активного profile). Также `spring-cloud-kubernetes` умеет читать ConfigMap напрямую через K8s API.

**Reload при изменении**: если ConfigMap смонтирован как volume — kubelet обновляет файлы автоматически (задержка ~60s). Но Spring Boot **не перечитывает** — нужен либо `@RefreshScope` (Spring Cloud), либо рестарт Pod'а.

Distinct: если ConfigMap подмонтирован через `subPath` — обновления **не подхватываются** (bind mount).

**Q11: Что делать если Pod stuck в Pending?**

Первый шаг: `kubectl describe pod X` → секция Events. Причины:
- **FailedScheduling: Insufficient cpu/memory** — нет ноды с достаточно свободных ресурсов. Fix: подождать autoscaler, вручную скейлить, снизить requests.
- **FailedScheduling: didn't match Pod's node affinity** — labels на нодах не соответствуют.
- **FailedScheduling: node(s) had taint** — нет tolerations.
- **FailedScheduling: didn't find available persistent volumes** — PVC не мог bound. Проверь StorageClass.
- **FailedCreatePodSandbox** — проблема с runtime (containerd), обычно с CNI. Проверь `journalctl -u kubelet` на ноде.
- **ImagePullBackOff** — образ не скачивается. Проверь имя, registry auth.

**Q12: Как работает Ingress?**

Ingress — declarative rules для HTTP-роутинга (host+path → Service). Сам по себе — только манифест в etcd, работает через **Ingress Controller**.

Controller — Pod (обычно Deployment nginx-ingress или Traefik), который:
1. Watch'ит Ingress-объекты через apiserver.
2. Генерирует конфиг соответствующего HTTP-сервера (nginx.conf, Traefik dynamic config).
3. Хостит сам HTTP-сервер, принимает трафик из внешки, роутит согласно правилам.
4. Экспозится через Service type LoadBalancer (в облаке) или hostPort.

TLS: Ingress с `spec.tls` берёт сертификат из K8s Secret. cert-manager может автоматически выпускать Let's Encrypt.

### 24.3 Продвинутое

**Q13: Что происходит при разрыве network partition между двумя частями кластера?**

Etcd (Raft) — partitioned минорити становится read-only, majority продолжает работать. Всё что писалось в майорити — сохраняется.

Apiserver — работают все, но пишут только те, кто с majority etcd. Читать могут все (если stale reads ОК).

Kubelet — если потеряло связь с apiserver → продолжает поддерживать существующие Pod'ы, но не может сходить за новыми. Через `node-monitor-grace-period` (40s) — Node помечается NotReady в apiserver-майорити. Через `pod-eviction-timeout` (5m) — controller-manager начинает эвиктить Pod'ы (с точки зрения майорити) — но реально Pod'ы продолжают работать на потерянной ноде.

Результат: split brain на уровне видимости, но не на данных. Etcd Raft гарантирует consistency.

**Q14: Разница между Endpoints и EndpointSlices?**

Endpoints — старый объект, один на Service, содержит все IPы всех Pod'ов. При 5000 Pod'ов — object в сотни KB, каждое обновление шлёт полный watch → тяжёлая нагрузка на apiserver/kube-proxy.

EndpointSlice (v1.17+, GA v1.21) — разбивка на chunks по ~100 endpoints. Меньше per-update traffic. Также поддерживает topology hints (zone), dual-stack IPv4/IPv6.

Kube-proxy предпочитает EndpointSlice если доступны.

**Q15: Что такое init container и когда его использовать?**

Init container — контейнер, запускающийся **до** основных, sequential. Должен exit 0 чтобы основные стартовали.

Использования:
- Wait for dependency: `until nc -z db 5432; do sleep 1; done`.
- Скачать конфиг из Vault, положить в emptyDir.
- Прогнать миграции БД (Liquibase/Flyway) — но обычно лучше в отдельный Job перед Deployment.
- Установить permissions на volume (`chown 1000 /data`).

Особенности:
- Init container имеет отдельные образы и ресурсы.
- Effective request Pod'а = max(sum init containers, sum regular containers).
- Восстановление после failure зависит от restartPolicy Pod.

**Q16: Как реализован leader election в K8s?**

Через **Lease** object (`coordination.k8s.io/v1`). Клиенты (например controller-manager replicas) пытаются:
1. `Get` текущего Lease.
2. Если `holderIdentity` устарел (`renewTime + leaseDurationSeconds < now`) — сделать `Update` с собой как holder.
3. Если `Update` успешен (optimistic concurrency через resourceVersion) — ты leader.
4. Периодически renew (`Update`) чтобы не потерять.
5. При shutdown — clear `holderIdentity` для быстрого transfer.

Один Lease per controller-set. Non-leader'ы ждут loop'ом.

client-go предоставляет `LeaderElector`:
```go
leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
    Lock: resourceLock,
    LeaseDuration: 15 * time.Second,
    RenewDeadline: 10 * time.Second,
    RetryPeriod:   2 * time.Second,
    Callbacks: leaderelection.LeaderCallbacks{
        OnStartedLeading: func(ctx context.Context) { run() },
        OnStoppedLeading: func() { os.Exit(0) },
    },
})
```

**Q17: Что такое finalizers?**

Finalizer — строка в `metadata.finalizers`. Когда delete приходит на объект — apiserver ставит `metadata.deletionTimestamp` (**soft delete**), но объект не удаляется пока finalizers не пусты.

Пример: PVC имеет finalizer `kubernetes.io/pvc-protection`. Пока Pod использует PVC — finalizer стоит, delete PVC не завершается. Как только Pod удалён — controller снимает finalizer, PVC удаляется.

Свои controllers могут использовать finalizers, чтобы **гарантированно почистить внешние ресурсы** перед удалением объекта. Reconcile при `deletionTimestamp != nil`:
1. Почистить внешнее (S3 bucket, Route53 запись).
2. Снять свой finalizer из списка.
3. Apiserver видит пустой список → окончательно удаляет объект.

**Q18: Как работают mutating и validating admission webhooks?**

Apiserver при получении запроса, после auth/authz, идёт по цепочке admission:
1. Все built-in mutating (LimitRanger, ServiceAccount, DefaultStorageClass, MutatingAdmissionWebhook).
2. Все зарегистрированные mutating webhooks (в лексиграфическом порядке или как настроено).
3. Валидация OpenAPI-схемы объекта.
4. Все built-in validating.
5. Все зарегистрированные validating webhooks.

Webhook — HTTPS-сервис. Apiserver шлёт POST с `AdmissionReview` (объект + метаданные). Ответ: `allowed: true/false`, для mutating — `patch` в JSON Patch формате.

`failurePolicy`:
- `Fail` — если webhook недоступен → запрос отклоняется.
- `Ignore` — → пропускается. Опасно для security-webhook'ов.

**Timeout**: default 10s, максимум 30s.

**Ловушка**: webhook который смотрит на все Pod'ы и падает — кластер не поднимется после reboot (кубелеты создают mirror pods, apiserver вызывает webhook, тот недоступен, mirror pod не создастся). Правильно: `namespaceSelector: matchExpressions: [{key: kubernetes.io/metadata.name, operator: NotIn, values: [kube-system]}]`.

**Q19: Что такое CSI и как он работает?**

CSI (Container Storage Interface) — стандартный gRPC API между K8s и storage-провайдерами. Заменил in-tree drivers (AWS EBS, GCE PD и т.д.) с K8s 1.13+.

Каждый CSI driver содержит:
- **Controller** (Deployment/StatefulSet) — общается с cloud API. Создаёт volumes, attach'ит к ноде.
- **Node** (DaemonSet) — на каждой ноде. Mount volume в Pod.

Sidecars от K8s (не от driver'а):
- `external-provisioner` — watch PVC, зовёт `CreateVolume`.
- `external-attacher` — watch VolumeAttachment, зовёт `ControllerPublishVolume`.
- `external-resizer` — resize PVC.
- `external-snapshotter` — VolumeSnapshot.
- `node-driver-registrar` — регистрирует driver в kubelet.

Жизненный цикл PVC:
1. PVC создан → external-provisioner видит → зовёт `CreateVolume` → появляется PV → bound.
2. Pod с PVC назначен на ноду → external-attacher зовёт `ControllerPublishVolume` → cloud API attach'ит disk к VM.
3. kubelet на ноде через CSI Node зовёт `NodeStageVolume` (format, mkfs) → `NodePublishVolume` (bind mount в Pod).

**Q20: Как работает Service Mesh (Istio, Linkerd)?**

Service Mesh — layer поверх K8s network, обеспечивающий:
- **mTLS** между всеми сервисами.
- **L7 роутинг** (по headers, weight-based canary).
- **Retries**, **timeouts**, **circuit breakers**.
- **Observability** (traces, metrics, logs).
- **Policy** (кто может звать кого).

Архитектура:
- **Data plane** — sidecar-контейнер Envoy (Istio) или Linkerd-proxy в каждом Pod'е. Перехватывает весь входящий/исходящий трафик Pod'а через iptables redirect.
- **Control plane** — управляет всеми sidecar'ами: конфиг, сертификаты, discovery.

Sidecar injection — через MutatingAdmissionWebhook. При создании Pod'а в labeled namespace webhook добавляет envoy-контейнер.

Тред-off: latency (+1-3ms), CPU/mem overhead (+50-200 MB на Pod из-за sidecar), сложность.

Для среднего проекта — обычно **не нужен**. Простой Ingress + внутренний mTLS через cert-manager для критичных путей достаточно.

**Q21: Как работает horizontal Pod autoscaler math?**

Формула:
```
desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))
```

При `replicas=4`, CPU utilization 80%, target 50%:
`desired = ceil(4 × 80/50) = ceil(6.4) = 7`.

Обновление каждые 15 сек. **Tolerance 10%** — не скейлим если ratio в [0.9, 1.1] (защита от flapping).

Для multiple metrics — считает `desired` по каждой метрике, берёт **максимум**.

Custom metrics — через **metrics.k8s.io** (resource) или **external.metrics.k8s.io** / **custom.metrics.k8s.io**. Обычно реализуется через **prometheus-adapter**.

Behavior (v1.18+) — можно тонко настраивать: cooldown, max scaling rate per period.

**Q22: Что такое Pod Security Standards и как их применить?**

PSS (Pod Security Standards) — 3 уровня политик:
- **privileged** — всё разрешено. Для инфры типа kube-proxy.
- **baseline** — минимальные ограничения. Нельзя hostNetwork, hostPID, hostIPC, privileged.
- **restricted** — строгий. Требует runAsNonRoot, dropCapabilities=[ALL], readOnlyRootFilesystem, seccomp=RuntimeDefault.

Применение через **Pod Security Admission** (built-in admission controller, замена PodSecurityPolicy которая deprecated с v1.25).

Namespace labels:
```yaml
pod-security.kubernetes.io/enforce: baseline
pod-security.kubernetes.io/enforce-version: v1.28
pod-security.kubernetes.io/warn: restricted
pod-security.kubernetes.io/audit: restricted
```

`enforce` — блокирует создание Pod'ов не проходящих политику. `warn` — предупреждение в kubectl. `audit` — только в audit log.

Best practice в prod: `baseline enforce, restricted warn+audit`. Постепенно перевести на `restricted enforce`.

**Q23: Как правильно сделать zero-downtime rolling update Spring Boot?**

Полный чек-лист:

1. **Spring Boot config**:
   ```yaml
   server:
     shutdown: graceful
   spring:
     lifecycle:
       timeout-per-shutdown-phase: 25s
   management:
     endpoint:
       health:
         probes:
           enabled: true
     endpoints:
       web:
         exposure:
           include: health,info,prometheus
   ```

2. **K8s Deployment**:
   - `strategy.rollingUpdate.maxSurge: 1`, `maxUnavailable: 0`.
   - `terminationGracePeriodSeconds: 45` (больше чем Spring timeout).
   - `preStop` hook `sleep 5-10` (дать kube-proxy обновить iptables).
   - `readinessProbe` на `/actuator/health/readiness`.
   - `livenessProbe` на `/actuator/health/liveness` (не readiness — не должна зависеть от БД).
   - `startupProbe` для медленного старта (`failureThreshold × periodSeconds` = достаточно на прогрев).

3. **PodDisruptionBudget** `minAvailable: replicas - 1`.

4. **Клиенты**:
   - HTTP клиенты должны иметь retry (idempotent) на 5xx / connection reset.
   - `Keep-Alive` timeout меньше `terminationGracePeriodSeconds`.

Что происходит step-by-step:
- New Pod поднимается, startup passes, readiness Ready → трафик идёт.
- Old Pod: маркируется Terminating → удаляется из EndpointSlice → kube-proxy обновляет iptables (гдие-то по кластеру 1-5s).
- preStop `sleep 10` — дожидаемся распространения обновления.
- SIGTERM → Boot прекращает принимать новые запросы, ждёт in-flight завершения.
- Boot exits → контейнер done → Pod удалён.

**Q24: Как отладить «Pod не принимает трафик хотя Running»?**

Пошагово:

1. **Pod действительно Ready?**
   ```bash
   kubectl get pod X -n ns
   # STATUS: Running, READY: 1/1
   ```
   Если `READY: 0/1` — readiness fail. Смотри `kubectl describe pod X`, секцию Events и Readiness probe results.

2. **Pod в endpoints Service?**
   ```bash
   kubectl get endpoints svc-name -n ns
   kubectl get endpointslices -n ns -l kubernetes.io/service-name=svc-name
   ```
   Если IP Pod'а не в списке — не Ready или label mismatch.

3. **Selector совпадает?**
   ```bash
   kubectl get svc svc-name -n ns -o jsonpath='{.spec.selector}'
   kubectl get pod X -n ns --show-labels
   ```

4. **NetworkPolicy не блокирует?**
   ```bash
   kubectl get networkpolicy -n ns
   ```
   Проверь rules — может у target Pod'а deny-all ingress без исключения для клиента.

5. **iptables rules на ноде клиента?**
   ```bash
   ssh <client-node>
   iptables-save | grep <service-cluster-ip>
   ```
   Должны быть правила KUBE-SVC-... → KUBE-SEP-... → DNAT.

6. **CoreDNS резолвит?**
   ```bash
   kubectl run debug --rm -it --image=nicolaka/netshoot -- nslookup svc-name.ns
   ```

7. **kube-proxy живой?**
   ```bash
   kubectl -n kube-system get pods -l k8s-app=kube-proxy
   kubectl -n kube-system logs kube-proxy-XXX --tail 50
   ```

**Q25: Как сделать безопасное graceful shutdown Kafka-консюмера в Pod?**

Kafka consumer имеет свой lifecycle (poll loop). При SIGTERM:

1. Preferred: приложение слушает SIGTERM, флажит `running=false` в poll loop:
   ```java
   Runtime.getRuntime().addShutdownHook(new Thread(() -> {
       running = false;
       consumer.wakeup();  // прервать блокирующий poll
   }));

   while (running) {
       ConsumerRecords records = consumer.poll(Duration.ofMillis(100));
       process(records);
       consumer.commitSync();
   }
   consumer.close();  // финальный commit + leave group gracefully
   ```

2. K8s:
   - `terminationGracePeriodSeconds` > max processing time одного batch'а + margin (например 60-120s).
   - Без preStop sleep (нет endpoints для консюмера — не нужно).
   - Liveness через `/actuator/health/liveness` — не на consumer status (при rebalancing может флэпать).

3. Kafka consumer group:
   - `session.timeout.ms: 10s` — rebalance быстро после leave.
   - `max.poll.interval.ms > processing time одного batch'а`.
   - Manual commit после обработки — обязательно.

При `consumer.close()` — leave group cleanly, брокер сразу инициирует rebalance, никакого дублирования при следующем поднятии.

---

## 25. Мини-чеклист «прочитал — знаю»

Просмотри пункты, ответ должен всплывать за 3 секунды:

- [ ] Raft, quorum, почему 3/5 нод etcd
- [ ] Что происходит при space quota etcd
- [ ] Encryption at rest, KMS integration
- [ ] Порядок admission в apiserver
- [ ] `list-then-watch` паттерн, что такое `410 Gone`
- [ ] Scheduler framework, extension points
- [ ] podAntiAffinity vs topologySpreadConstraints
- [ ] Preemption и priorityClass
- [ ] Reconcile loop, informer, workqueue, exponential backoff
- [ ] Finalizers и deletionTimestamp
- [ ] kubelet syncLoop, PLEG
- [ ] CRI и OCI, containerd/runc
- [ ] iptables vs IPVS vs eBPF в kube-proxy
- [ ] EndpointSlices vs Endpoints
- [ ] externalTrafficPolicy: Cluster vs Local
- [ ] CNI: Flannel VXLAN, Calico BGP, Cilium eBPF
- [ ] CoreDNS, ndots:5, search domains
- [ ] RBAC: Role/ClusterRole/RoleBinding/ClusterRoleBinding/ServiceAccount
- [ ] Bound tokens (K8s 1.22+), projected volumes
- [ ] Mutating vs Validating admission, failurePolicy
- [ ] Pod Security Admission — privileged/baseline/restricted
- [ ] OPA Gatekeeper vs Kyverno
- [ ] CSI архитектура, external-provisioner/attacher
- [ ] Volume snapshots
- [ ] HPA formula, tolerance 10%, behavior
- [ ] VPA modes, HPA+VPA конфликт
- [ ] Cluster Autoscaler vs Karpenter
- [ ] KEDA — scaling to zero
- [ ] CRD + Operator, kubebuilder
- [ ] Level-triggered reconciliation
- [ ] Stacked vs external etcd
- [ ] Leader election через Lease
- [ ] Pod phases и Container states
- [ ] CrashLoopBackOff mechanics, exponential backoff
- [ ] Cgroups (CPU throttling, memory OOM) и namespaces
- [ ] SecurityContext: runAsNonRoot, readOnlyRootFilesystem, capabilities, seccomp
- [ ] Rolling update: maxSurge/maxUnavailable, progressDeadlineSeconds
- [ ] Blue/Green и Canary через Deployment
- [ ] Argo Rollouts, Flagger
- [ ] Sealed Secrets, External Secrets Operator, Vault Agent Injector
- [ ] Vault K8s auth через SA JWT
- [ ] Prometheus + kube-state-metrics + cAdvisor + node-exporter
- [ ] Ephemeral containers, kubectl debug
- [ ] Preferred prod-set: replicas ≥ 2, PDB, probes, resources, security, antiAffinity

Если по каждому пункту можешь за 3-5 предложений объяснить — на middle+ уровне ты в K8s.

---

## Итог

Kubernetes — это **desired-state машина** через 4 роли:
1. **etcd** (истина) → **apiserver** (единственная дверь) → **controllers** (приводят реальность к spec) → **kubelet** (запускает).

Всё остальное — вариации: scheduler = controller, HPA = controller, Ingress controller = controller, operator = controller. Понял reconcile loop с informer/workqueue — понял 80% K8s.

Основные линии углубления:
- **etcd** (Raft, MVCC, watch, encryption).
- **apiserver** (admission chain, aggregation, APF).
- **scheduler** (framework, extension points, preemption).
- **controller** (informer, lister, workqueue, finalizers, leader election).
- **kubelet** (syncLoop, PLEG, CRI, static pods).
- **networking** (kube-proxy режимы, CNI детально, DNS, Ingress vs Gateway).
- **security** (RBAC, admission, PSS, encryption).
- **storage** (CSI, snapshots).
- **autoscaling** (HPA math, VPA modes, Cluster Autoscaler, Karpenter, KEDA).
- **CRD/Operator pattern**.
- **HA** (stacked/external etcd, leader election).
- **Debugging** (events, describe, logs, ephemeral containers).

Дальше в теме — читай кодовую базу kubernetes/kubernetes на GitHub (компоненты в `cmd/`), книгу *Kubernetes in Action* (Marko Lukša, 2nd ed 2023) и блог posts от Kelsey Hightower / Brendan Burns.

Для practice — **kubernetes-the-hard-way** (Kelsey Hightower на GitHub): собери K8s руками из компонентов, без kubeadm. После этого «внутренности» перестают быть абстракцией.
