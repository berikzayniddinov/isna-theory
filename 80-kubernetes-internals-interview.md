# 80. Kubernetes internals: как реально устроен изнутри

## Зачем это знать

Kubernetes для большинства команд — это «применил yaml, работает». Пока работает — прекрасно. Как только начинается инцидент, миграция, сложная production-задача — эта модель перестаёт помогать. Почему `kubectl apply` иногда не применяется. Почему pod пропал но приложение продолжает работать. Почему CRD появился в API но controller не срабатывает. Почему `kubectl exec` вдруг стал недоступен на всём кластере. Ответы находятся не в yaml-манифестах, а в устройстве control plane, в reconcile-модели контроллеров, в том как apiserver прокидывает watch до kubelet, как etcd переживает split brain, как scheduler фильтрует ноды.

Базу (Pod / Deployment / Service, probes, PVC, Helm) смотри в 10-kubernetes-detailed.md. Продвинутые workloads, graceful shutdown, PDB — в 77-kubernetes-deep-microservices.md. Диагностику типовых инцидентов — в 78-prod-diagnostics-workflow.md. Здесь — внутренности: etcd и Raft, apiserver и admission, scheduler framework, controller pattern с informers/workqueue, kubelet и CRI, kube-proxy режимы, CNI, DNS, RBAC, CSI, autoscaling, CRD/operator, HA. Плюс — большой блок собеседных вопросов с осмысленными ответами.

Одна ментальная модель, из которой вырастает всё остальное: **Kubernetes — это распределённая desired-state машина, работающая по level-triggered reconciliation**. Пользователь пишет в apiserver «хочу N объектов ресурса R с параметрами spec». Apiserver валидирует, прогоняет через admission, кладёт в etcd. Controllers слушают изменения через watch и приводят реальный мир к spec — этот цикл называется reconcile loop. Scheduler — тоже controller, решает где именно запустить Pod. Kubelet на ноде запускает контейнеры и репортит status обратно в apiserver. Всё остальное — HPA, Ingress Controller, cert-manager, ArgoCD — это ещё контроллеры. Понял reconcile loop — понял всю систему.

Ключевые свойства этой модели, которые надо усвоить в кости:

- **Level-triggered, а не edge-triggered.** Controller не «получает событие один раз». Он смотрит на текущее состояние каждый раз и приводит его к желаемому. Пропустил уведомление — следующий resync (каждые 10 минут по умолчанию) всё равно приведёт к правильному действию. Это делает систему устойчивой к сетевым сбоям.
- **Eventually consistent.** Между apply и наступлением реального состояния — задержка (обычно секунды, иногда минуты). Спрашивать «когда все Pod'ы будут Ready» — неправильный вопрос, правильный: «мониторь status и жди».
- **API — единственная правда.** Все компоненты общаются только через apiserver, нет прямых RPC между контроллерами.
- **Etcd — источник правды.** Всё что не в etcd — не существует для K8s. Restart компонента = чтение состояния из etcd.

## etcd — распределённое сердце кластера

etcd — распределённая key-value БД на алгоритме консенсуса Raft. Написана в CoreOS в 2013, сейчас CNCF-проект. Kubernetes использует etcd v3 как единственный persistent store всех объектов. Ключевые свойства: strong consistency (linearizable reads: read after write гарантирован), HA через Raft (переживает падение `(N-1)/2` нод из N), watch API (клиенты подписываются на изменения), MVCC (каждое изменение — новая ревизия).

**Raft** в двух словах. Есть N нод etcd. Одна — leader, остальные — followers. Клиент пишет → leader принимает → реплицирует в лог followers → как только большинство подтвердило (**quorum**) → leader коммитит и отвечает клиенту. Followers применяют коммиченные записи к своему state machine. Если leader умер — followers через таймаут начинают выборы, голосуют, выбирают нового leader'а. Каждый терм (эпоха) leader'а уникален, что не даёт «старым» leader'ам путать порядок команд после раскола сети.

Quorum-формула: `Q = floor(N/2) + 1`. Три ноды — quorum 2, переживает падение одной. Пять — quorum 3, переживает две. Семь — четыре, переживает три. Чётное число (2, 4, 6) **бесполезно**: quorum тот же что и у нечётного меньше на 1, но переживает меньше падений. Всегда ставь нечётное — 3 (стандарт) или 5 (крупный кластер). Одна нода etcd — не HA, любая проблема кладёт весь кластер.

Внутри etcd — MVCC. Каждое изменение = новая **revision** (монотонно растущий int64). Старые версии не удаляются сразу, хранятся в BoltDB (backend). Это позволяет читать «на момент revision R» и главное — **watch «начиная с revision R»**. Подписчик получает все изменения после R. Именно эта возможность — фундамент для K8s controllers. Apiserver держит watch на etcd, controllers держат watch на apiserver — любое изменение ресурса моментально доставляется до всех подписчиков.

Старые revisions занимают место. Раз в 5 минут apiserver вызывает `etcd compact` — удаляет revisions старше порога. После compaction нужен `etcd defrag`, чтобы физически освободить место в BoltDB. Если этого не делать — этcd растёт, в какой-то момент упирается в space quota (`--quota-backend-bytes`, дефолт 2GB) и **весь кластер K8s становится read-only**. Симптомы: `kubectl apply` возвращает `mvcc: database space exceeded`. Лечение: `etcdctl endpoint status` (проверить занятость), `etcdctl alarm list` (есть ли `NOSPACE` alarm), `etcdctl compact <rev>` + `etcdctl defrag`, `etcdctl alarm disarm`.

Что хранит etcd: все объекты (Pod, Deployment, Service, ConfigMap, Secret, CustomResources), namespaces, node registrations, leases для leader election контроллеров, короткоживущие events. **Не хранит**: логи приложений, метрики, docker images, container runtime состояние — это всё на kubelet'ах и registry.

**Секреты в etcd** — важная тема. По умолчанию Secret хранится в etcd в base64 — это кодирование, не шифрование. Любой с доступом к etcd читает пароль в открытом виде. Encryption at rest включается через `--encryption-provider-config` у apiserver. Форматы: identity (plain, дефолт), aescbc, aesgcm, secretbox, **kms** (интеграция с внешним KMS типа AWS KMS, HashiCorp Vault — правильный prod-выбор). Ротация ключа двухфазная: добавляешь новый ключ вторым, пересохраняешь все Secrets (чтобы они зашифровались новым), убираешь старый.

Backup etcd — обязательная операция, минимум ежедневно, хранить неделю. `etcdctl snapshot save backup.db`. Без backup'а полная потеря etcd = потеря кластера (все объекты, RBAC, secrets). Реальные Pod'ы продолжат работать пока живы ноды, но управление сломано. Restore — `etcdctl snapshot restore` с параметрами cluster'а, стартовать etcd с новым data-dir.

Собесные ловушки: **«Почему нельзя 2 ноды etcd?»** — quorum = 2, падение одной = read-only. **«Что произойдёт если etcd упадёт полностью?»** — apiserver возвращает 500 на write, ничего нельзя создавать. Но существующие Pod'ы продолжают работать: kubelet не общается с etcd напрямую, только через apiserver, а kube-proxy уже настроил iptables. **«Split-brain?»** — Raft гарантирует что split brain невозможен: partitioned minority не может выбрать leader'а без quorum.

## kube-apiserver: единственная дверь

Apiserver — единственный компонент, который общается с etcd. Все остальные — kubelet, scheduler, controller-manager, kubectl, HPA, любые операторы — ходят через apiserver. Это гарантирует консистентность (все видят одну и ту же картину) и позволяет ставить security-политики (RBAC, admission) в одном месте.

Функции: REST-фасад над etcd, аутентификация + авторизация, admission (валидация и мутация), watch (стриминг изменений подписчикам), API aggregation (можно подключить сторонние API-серверы под тем же `kubectl`).

**Полный путь запроса** — то, что стоит запомнить наизусть, потому что почти любая проблема с applyем расположена в одном из этих шагов:

```
kubectl apply -f pod.yaml
      ↓
[TLS handshake, cert auth или token auth]
      ↓
[Authentication]   ← кто ты? (X.509, token, OIDC, ServiceAccount JWT)
      ↓
[Authorization]    ← можешь ли? (RBAC, ABAC, Webhook, Node)
      ↓
[Mutating Admission]    ← модифицируем объект (inject sidecar, defaults)
      ↓
[Object Schema Validation]  ← OpenAPI-схема
      ↓
[Validating Admission]  ← можно ли сохранить? (ResourceQuota, PSA, custom)
      ↓
[etcd write]
      ↓
Response to client
      ↓
Watch notifications всем подписчикам
```

Порядок mutating → validating важен: сначала все меняют объект (в любом порядке между собой), потом все валидируют финальный вариант. Это гарантирует что admission-плагины не увидят друг друга наполовину применёнными.

**Watch как работает.** Клиент шлёт `GET /api/v1/pods?watch=true&resourceVersion=100500`. Apiserver держит соединение открытым (chunked HTTP), стримит события: `ADDED`, `MODIFIED`, `DELETED`. Каждое событие содержит `resourceVersion` — если клиент разорвал соединение и переподключился, продолжит с последнего RV. Если сервер выкинул старые revisions из кэша — клиент получает `410 Gone` и обязан сделать полный **relist** (`GET /pods`), взять новый RV и начать watch заново. Это стандартный паттерн «list-then-watch», его реализует client-go **reflector**.

API groups: Core (`/api/v1` — Pod, Service, ConfigMap, Secret, Namespace, Node, PersistentVolume), `apps/v1` (Deployment, StatefulSet, DaemonSet), `batch/v1` (Job, CronJob), `networking.k8s.io/v1` (Ingress, NetworkPolicy), `rbac.authorization.k8s.io/v1`, `policy/v1`, `scheduling.k8s.io/v1`, `storage.k8s.io/v1`, `autoscaling/v2`, `apiextensions.k8s.io/v1` (CRD). Уровни зрелости: alpha (v1alpha1, отключён по умолчанию, обратной совместимости нет) → beta (v1beta1, включён, но API может измениться) → stable (v1).

**API aggregation layer** позволяет включить кастомный apiserver под тем же `kubectl`. Пример — metrics-server, предоставляющий `/apis/metrics.k8s.io/v1beta1/pods`. Не через CRD (CRD хранят в основном etcd), а как отдельный HTTP-сервер, к которому проксирует main apiserver через `APIService`. Разница: CRD — новый тип ресурса в стандартном etcd; APIService — прокси к сторонней реализации, может хранить где угодно.

**APF (API Priority and Fairness)** защищает apiserver от флуда. Определяются FlowSchemas (по кому фильтровать: `kubectl` от system:masters, kubelet'ы, случайный SA) и PriorityLevelConfigurations (сколько concurrent-запросов на каждый уровень). При переполнении низкоприоритетные ждут в очереди, высокоприоритетные (системные) продолжают работать.

Собесные: **«Как аутентифицируется kubelet при обращении к apiserver?»** — X.509-сертификат, подписанный CA кластера, `CN: system:node:<hostname>`. Node authorizer разрешает только те объекты, которые касаются этой ноды. **«Что такое ServiceAccount внутри Pod?»** — SA — объект K8s, каждый Pod получает JWT-токен, вмонтированный в `/var/run/secrets/kubernetes.io/serviceaccount/token`. Проекционный volume, ротация через 1 час (bound token, TokenRequest API, K8s 1.22+ вместо старых долгоживущих Secret-based токенов).

## kube-scheduler: как pod'ы попадают на ноды

Scheduler — специализированный controller. Слушает через watch: «Pod с `spec.nodeName == ""`». Для каждого такого пода — решает на какую ноду его назначить, пишет `spec.nodeName = <node>` в apiserver. Дальше kubelet той ноды видит через watch «Pod, назначенный мне» и запускает. **Scheduler не запускает контейнеры**, только принимает решение о размещении.

**Scheduler framework** (K8s 1.19+) заменил старый двухфазный алгоритм (predicates → priorities) на систему **12 extension points** с плагинами:

```
QueueSort → PreEnqueue → PreFilter → Filter → PostFilter →
PreScore → Score → NormalizeScore →
Reserve → Permit → PreBind → Bind → PostBind
```

Основные для понимания — Filter, Score, Bind. **Filter phase**: для каждой ноды прогоняем все filter-плагины (`NodeResourcesFit`, `NodeAffinity`, `PodTopologySpread`, `TaintToleration`, `NodeUnschedulable`, `VolumeBinding`, `NodeName`, `NodePorts`). Если хоть один сказал «нет» → нода отсеяна. Оставшиеся — «feasible». **Score phase**: каждый score-плагин выставляет 0-100 (`NodeResourcesFit` для spread/binpack, `ImageLocality` для нод где уже есть образ, `InterPodAffinity`). Веса плагинов суммируются, максимум назначается. При ничьей — случайный выбор.

**Если нет feasible нод** — запускается PostFilter. Обычно там **Preemption**: пытаемся вытеснить Pod'ы с меньшим приоритетом на какой-то ноде, чтобы освободить место. Алгоритм: для каждой ноды считаем какие Pod'ы с меньшим priority надо убрать, выбираем ноду с наименьшим «повреждением», отправляем graceful delete preempted pod'ов, наш Pod ждёт в очереди пока освободится место, потом шедулится.

Управление размещением из spec — семейство инструментов, каждый со своими use-cases.

**nodeSelector** — самое простое, простое соответствие labels: `spec.nodeSelector: {disk: ssd, zone: us-east-1a}`. Nodes с этими labels — единственные кандидаты.

**nodeAffinity** — гибче, поддерживает выражения `In`, `NotIn`, `Exists`, `Gt`, `Lt`. Разделяется на `requiredDuringSchedulingIgnoredDuringExecution` (жёсткое требование при schedule, но игнорируется потом — не эвиктит Pod если label изменился) и `preferredDuringSchedulingIgnoredDuringExecution` (мягкое предпочтение, weight 1-100).

**podAffinity / podAntiAffinity** — размещение рядом или подальше от других Pod'ов по labels. `topologyKey` определяет что считаем топологической единицей: `kubernetes.io/hostname` = ноды, `topology.kubernetes.io/zone` = availability zones. Для HA-приложений — antiAffinity по zone (три реплики в трёх зонах, падение зоны не убивает все).

**Taints и tolerations** — обратная логика: нода **отталкивает** Pod'ы, если у них нет соответствующего tolerance. `kubectl taint node worker1 dedicated=gpu:NoSchedule` делает worker1 доступной только для Pod'ов с `tolerations: [{key: dedicated, value: gpu, effect: NoSchedule}]`. Effects: `NoSchedule` (не шедулим), `PreferNoSchedule` (стараемся не), `NoExecute` (не шедулим и выгоняем существующие без tolerance).

**TopologySpreadConstraints** — равномерное распределение по топологии. `maxSkew: 1, topologyKey: zone` = между зонами разница по количеству Pod'ов не больше 1. Правильнее чем podAntiAffinity для равномерного распределения, потому что antiAffinity даёт «либо один-на-хост, либо всё сломалось» — жёсткое требование, при 3 репликах и 2 нодах schedule зависает.

**Отладка «Pending forever»** — `kubectl describe pod X` → Events:

```
Warning FailedScheduling  0/5 nodes are available: 
  3 Insufficient memory, 2 node(s) had taint {...}
```

Причины: `Insufficient cpu/memory` (requests не влезли), `node(s) had taint` (нет tolerance), `didn't match Pod's node affinity/selector` (не подошли labels), `didn't find available persistent volumes to bind` (PVC не привязался), `too many pods` (max pods per node, дефолт 110).

## Controller pattern: сердце Kubernetes

Reconcile loop — псевдокод любого controller'а:

```go
for {
    desired := getDesiredState()   // из apiserver, spec
    actual := getActualState()     // из apiserver status + внешний мир
    if desired != actual {
        act(desired, actual)       // приблизить
    }
    sleep(shortWait)
}
```

На практике — не polling, а event-driven через watch. Плюс периодический resync (default 10 минут) как страховка от пропущенных событий.

**Level vs edge triggering** — концепция из электроники, применённая здесь. Edge-triggered: реагируем на конкретное событие (переход из 0 в 1). Пропустил событие — потерял действие. Level-triggered: смотрим на текущее состояние (уровень сигнала). Пропустил событие — следующий взгляд всё равно приведёт к правильному действию. Kubernetes controllers — level-triggered, что делает систему устойчивой к любым сетевым сбоям и рестартам.

**Informers, Listers, Work queues** — стандартная архитектура из client-go. Reflector стримит watch с apiserver в DeltaFIFO. Informer читает DeltaFIFO, обновляет local cache (thread-safe in-memory store) и вызывает event handlers. Handler кладёт `key` (`namespace/name`) в WorkQueue. Worker берёт key из WorkQueue, читает объект из Lister (обёртка над cache), делает reconcile. Если reconcile упал — worker кладёт key обратно в очередь с exponential backoff.

Три свойства, которые эта архитектура даёт:

- **Кэш**: не бомбим apiserver read-запросами, читаем из локальной копии, обновляемой через watch.
- **Rate limiting**: WorkQueue дедуплицирует (много events на один object → один reconcile) и делает backoff при ошибках.
- **Идемпотентность**: reconcile всегда идёт по текущему состоянию из cache, а не по конкретному событию. Пропустил event — не страшно.

**Что нужно знать при написании controller'а**. Идемпотентность — reconcile может вызваться 100 раз для одного объекта, действия должны быть безопасны для повторения. Finalizers — если controller «владеет» внешним ресурсом (создал что-то в облаке), нужно ставить finalizer на объект. При удалении K8s не удалит объект пока finalizer стоит — controller успеет почистить внешнее и снять finalizer. Status updates — не мешать status в Reconcile без ретрая, apiserver может вернуть 409 Conflict (optimistic concurrency через resourceVersion). Deletion timestamp — объект в стадии удаления имеет `metadata.deletionTimestamp != nil`, controller должен уметь это обработать.

**Пример каскада** — что происходит когда ты создал Deployment. Deployment controller видит новый Deployment через watch → создаёт ReplicaSet с template hash. ReplicaSet controller видит новый RS → создаёт N Pod'ов с matching labels. Scheduler видит Pod'ы без nodeName → назначает nodeName. Kubelet на ноде видит Pod со своим nodeName → запускает через CRI. Все эти шаги — независимые reconcile loops разных controllers, каждый работает на своём уровне абстракции.

Изменил `spec.template.image`: Deployment controller видит изменение → считает hash template → отличается от текущего RS → создаёт новый RS с новым hash. Скейлит новый RS вверх, старый вниз согласно `strategy.rollingUpdate`. Ждёт readiness новых Pod'ов перед скейл-даун старых. Всё это — тот же level-triggered reconcile, просто в цикле пока текущее состояние не сравняется с желаемым.

Собесные: **«Почему reconcile-паттерн лучше императивного?»** — level-triggered даёт самолечение, пропущенное событие не ломает всё, идемпотентность делает безопасным перезапуск controller'а. **«Что если два controller'а претендуют на один объект?»** — конфликт через optimistic locking (resourceVersion), проигравший переретраится. **«Как реализовать leader election?»** — через **Lease** object в K8s API. `client-go/tools/leaderelection`. Только leader делает reconcile, остальные ждут. Если leader умер — оставшиеся через `leaseDurationSeconds` начинают выборы.

## kubelet: что реально запускает контейнеры

Kubelet — агент на каждой ноде. Регистрирует ноду в apiserver, слушает watch «Pod'ы с nodeName = myself», через CRI запускает/останавливает контейнеры, управляет volumes (монтирует CSI), запускает probes, отчитывается статус Pod'ов, собирает метрики (cAdvisor встроен).

**SyncLoop** — главный цикл. Merge событий из четырёх источников: apiserver Pod updates (watch), PLEG events (изменения в container runtime), housekeeping tick (каждые 2 сек), liveness probe results. Для каждого события — `syncPod(pod)` приводит реальность в соответствие с pod spec.

**PLEG** (Pod Lifecycle Event Generator) — периодически (по умолчанию 1 сек) опрашивает container runtime «какие контейнеры сейчас есть», сравнивает с прошлым состоянием, генерирует events типа `ContainerStarted`, `ContainerDied`. Известное сообщение `PLEG is not healthy` в `/var/log/kubelet.log` — возникает когда container runtime (обычно containerd) зависает. Приводит к тому что kubelet не видит новые контейнеры, всё стопорится, Node становится NotReady.

**CRI** — Container Runtime Interface. Kubelet общается с runtime через gRPC-протокол CRI. Ключевые методы: `RunPodSandbox` (создать pause-контейнер + все namespace), `CreateContainer` / `StartContainer` (запустить рабочий контейнер в существующем sandbox), `RemoveContainer` / `StopPodSandbox` (обратное), `ListContainers`, `ContainerStatus` (читать состояние). Runtimes: **containerd** (стандарт, легковесный), CRI-O (в OpenShift), Docker Engine — **удалён из K8s 1.24+** (Docker никогда не имел CRI напрямую, использовался через dockershim — обёртку внутри kubelet, теперь удалена). Специализированные: kata-containers (VM вместо контейнеров, полная изоляция), gVisor (user-space sandbox, компромисс).

**OCI** — Open Container Initiative — три спецификации: OCI Image Spec (формат образа — слои, манифест), OCI Runtime Spec (как запустить контейнер), OCI Distribution Spec (API реестра). Реальный runtime под containerd — **runc** (Go, reference implementation) или **crun** (C, быстрее). Именно они вызывают `clone()`, `setns()`, `unshare()`, `pivot_root()` в Linux для создания контейнера.

Полный слой запуска:

```
kubectl apply → apiserver → etcd
                   ↓ (kubelet узнаёт через watch)
                kubelet
                   ↓ (CRI, gRPC)
                containerd
                   ↓ (shim + OCI runtime spec)
             containerd-shim  ←── долгоживущий процесс на контейнер
                   ↓ (fork/exec)
                 runc
                   ↓ (syscalls)
              Linux kernel → cgroups + namespaces → реальный процесс
```

Зачем shim: чтобы containerd можно было перезапустить, а контейнеры не умерли. Shim держит stdout/stderr, репортит exit code.

**Static Pods** — kubelet может запускать Pod'ы, определённые в `--pod-manifest-path` (обычно `/etc/kubernetes/manifests/`). Файлы YAML в этой директории → kubelet сам запускает, apiserver ничего не знает (но kubelet создаёт **mirror pod** в apiserver для видимости). Используется для **самого control plane при bootstrap**: kubeadm ставит kubelet, kubelet читает манифесты apiserver'а/etcd/scheduler'а/controller-manager'а из static pod dir → всё поднимается. Удалить mirror через `kubectl delete` не работает (kubelet сразу пересоздаёт), надо удалить файл манифеста на ноде.

**Node lifecycle**. Node становится `NotReady` если kubelet не пингует apiserver дольше `node-monitor-grace-period` (дефолт 40s). После `pod-eviction-timeout` (дефолт 5m) controller-manager начинает эвиктить Pod'ы с этой ноды. Существующие Pod'ы на реально живой ноде продолжают работать даже если node NotReady — kubelet сам решает.

## kube-proxy и Service internals

Kube-proxy на каждой ноде. Читает Service+EndpointSlice через watch. Настраивает **правила ядра** так, чтобы трафик на ClusterIP шёл на реальный Pod IP. **Не проксирует в userspace** (в старых версиях умел, deprecated).

Три основных режима, каждый со своими trade-off:

**iptables (default)**. Для каждого Service — цепочка iptables. Для каждого backend — правило с DNAT. Random selection через probability модуль. Пример правил (упрощённо) для Service `10.96.42.15:8080` с двумя Pod'ами:

```
-A KUBE-SERVICES  -d 10.96.42.15/32 -p tcp --dport 8080 -j KUBE-SVC-X
-A KUBE-SVC-X  -m statistic --mode random --probability 0.5 -j KUBE-SEP-A
-A KUBE-SVC-X  -j KUBE-SEP-B
-A KUBE-SEP-A  -p tcp -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-B  -p tcp -j DNAT --to-destination 10.244.2.6:8080
```

Проблемы iptables: правила линейные, при 1000 Service × 3 replica = 3000 правил в цепи, iptables оценивает O(n). Обновление всей цепи атомарно (`iptables-restore`) — при 10000 правил это ~сек, каждый Service update = блокировка kube-proxy. Не даёт настоящий L4 LB, только random.

**IPVS**. Использует Linux IPVS (in-kernel L4 load balancer, изначально для LVS). Хеш-таблицы вместо линейного поиска → O(1). Поддерживает алгоритмы: `rr` (round robin), `lc` (least connections), `dh` (destination hash), `sh` (source hash), `wrr` (weighted). Плюсы: масштабируется на 10000+ Service, стабильная latency. Минусы: сложнее диагностировать (`ipvsadm -Ln`), меньше документации. Включается через `--proxy-mode=ipvs`.

**eBPF (Cilium)**. Полная замена kube-proxy. Cilium использует eBPF-программы на сетевом стеке ядра, обходит iptables целиком. Гораздо быстрее (миллионы Service без замедления), богатая observability. Минусы: сложнее ставить, требует Linux kernel 4.19+. Для новых больших кластеров — обычно правильный выбор.

**nftables** (K8s 1.29+) — современная замена iptables, быстрее и с лучшей атомарностью. Ещё не default.

**EndpointSlices** заменили монолитный Endpoints. Раньше был один `Endpoints` object на Service со списком всех Pod IP. При 5000 Pod'ов — object 200KB, каждое обновление шлёт полный watch → нагрузка. EndpointSlice (K8s 1.17+, GA 1.21) — разбивка на chunks по ~100 Pod'ов. Меньше traffic на watch, лучше scaling. Также поддерживает topology hints (zone), dual-stack IPv4/IPv6.

**externalTrafficPolicy** — важный параметр Service типа NodePort/LoadBalancer:

- `Cluster` (default) — трафик, попавший на ноду N, может быть перенаправлен на Pod на ноде M. Source IP теряется (SNAT), балансировка равномерная.
- `Local` — трафик остаётся на ноде, где принял, только на Pod'ы этой же ноды. Source IP сохраняется. Но если на ноде нет Pod'а — 0 backend → внешний LB должен исключать такие ноды через health check.

Для приложений, которым нужен реальный клиентский IP (rate limiting, audit log) — используй `Local` + внешний LB с healthcheck.

**Topology Aware Routing** (K8s 1.21+, `service.kubernetes.io/topology-mode: Auto`) — kube-proxy предпочитает backend'ы в той же zone/regionе, что и клиент. Снижает cross-AZ трафик (в AWS это стоит денег).

**Headless Services** (`clusterIP: None`) — kube-proxy не создаёт правил. CoreDNS отдаёт A-записи всех Pod'ов напрямую. Клиенту самому решать балансировку. Используется для StatefulSet DNS (`postgres-0.postgres.default.svc.cluster.local`), client-side load balancing (Spring Cloud LoadBalancer, gRPC), discovery в приложении.

## CNI: сеть между Pod'ами

**Модель сети K8s** декларирует три требования: все Pod'ы могут общаться со всеми Pod'ами без NAT, все Nodes могут общаться со всеми Pod'ами без NAT, IP который Pod видит у себя — это IP который видят другие. Не диктует **как**, диктует **что**. Реализуют CNI plugins.

Kubelet при создании Pod'а вызывает CNI plugin (binary в `/opt/cni/bin/`, конфиг в `/etc/cni/net.d/`). Plugin получает Pod namespace (network namespace path) и команду `ADD`/`DEL`/`CHECK`, возвращает IP, routes, DNS. Что делает plugin для `ADD`: выделяет IP из пула (IPAM), создаёт veth-pair (один конец в host, другой в pod netns), назначает IP на pod-стороне, настраивает route в pod netns (default gateway), на хосте настраивает bridge/route чтобы трафик доходил.

**Flannel** — overlay через VXLAN. Каждой ноде свой CIDR (`10.244.1.0/24` для node1, `10.244.2.0/24` для node2). Pod на node1 шлёт пакет на Pod на node2: пакет отправляется через bridge, флэннел заворачивает в VXLAN (UDP порт 4789), пакет летит по physical сети как UDP до node2, там распаковывается, попадает в bridge, доходит до Pod. Плюсы: работает в любом окружении, не требует поддержки от сети. Минусы: overhead инкапсуляции (~50 bytes на пакет), MTU нужно уменьшать, NetworkPolicy не поддерживает.

**Calico** — BGP (без overlay). Каждая нода — маршрутизатор, настраивают BGP peering с соседями. Pod IP анонсируются как отдельные /32 routes. Никакой инкапсуляции — плоская сеть. Плюсы: производительность как у native, поддержка NetworkPolicy. Минусы: требует чтобы underlying сеть пропускала произвольные IP (в облаке — не всегда). Есть IPIP mode — Calico с инкапсуляцией для случая когда flat не работает.

**Cilium** — eBPF на всём: pod-to-pod, service load balancing, NetworkPolicy, observability. Может полностью заменить kube-proxy. Плюсы: производительность, богатая observability (Hubble даёт flow logs, service map). Минусы: Linux kernel 4.19+, сложнее в первичной настройке.

**IPAM** (IP Address Management): host-local (каждая нода имеет свой CIDR, назначает локально, простой), calico-ipam (глобальный пул с распределением по нодам блоков), AWS VPC CNI (использует ENI'ы AWS, каждый Pod получает реальный VPC IP — можно использовать security groups, PrivateLink).

Собесное: **«Как выбирать CNI?»** — в облаке провайдер обычно даёт свой (AWS VPC CNI, Azure CNI, GCP netd). On-prem: Calico или Cilium. Если нужна NetworkPolicy — не используй Flannel. **«Разница pod-to-pod и pod-to-service?»** — pod-to-pod: прямая IP-коммуникация через CNI. Pod-to-service: сначала DNS → ClusterIP → kube-proxy iptables DNAT → реальный Pod IP → CNI.

## CoreDNS и разрешение имён

CoreDNS — стандартный DNS-сервер K8s (заменил kube-dns). Работает как Deployment в `kube-system`. Каждый Pod через `dnsPolicy: ClusterFirst` (default) получает `/etc/resolv.conf`:

```
nameserver 10.96.0.10       # ClusterIP CoreDNS
search knp.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Форматы имён: Service — `<service>.<namespace>.svc.cluster.local` (A-record → ClusterIP). Headless Service — то же имя, но возвращает все Pod IP. Pod DNS в StatefulSet — `<pod-name>.<service>.<namespace>.svc.cluster.local`. External через ExternalName Service — возвращает CNAME.

**ndots:5 и search domains** — важная деталь производительности. `ndots:5` означает: если в запрашиваемом имени меньше 5 точек — сначала пробуем с search domains, потом абсолютно. Пример: `curl isnaknpuser` (0 точек) пробует последовательно `isnaknpuser.knp.svc.cluster.local` → `isnaknpuser.svc.cluster.local` → `isnaknpuser.cluster.local` → `isnaknpuser.` (as-is). Это удобно для internal, но замедляет запросы к внешним доменам без FQDN. Оптимизация: указывать полные FQDN с точкой в конце (`external-api.com.`) или `dnsConfig` в Pod spec с `ndots:2`.

Собесное: **«Почему `curl external.api.com` в Pod медленный?»** — 4 dots, при ndots:5 нужно ещё делать попытки через search domains перед реальным запросом. Каждая — negative cache/NXDOMAIN, тормозит.

**NodeLocal DNS Cache** — DaemonSet CoreDNS на каждой ноде. Pod'ы шлют DNS в локальный кэш, снижается нагрузка на центральный CoreDNS и латентность. Best practice для крупных кластеров.

## RBAC: кто может что

Четыре объекта. **Role** — permissions в namespace. **ClusterRole** — permissions на весь кластер или non-namespaced ресурсы. **RoleBinding** — связь Role с subject (User/Group/ServiceAccount) в namespace. **ClusterRoleBinding** — связь ClusterRole с subject на всём кластере.

Пример: SA `monitoring` из namespace `monitoring` должен читать Pod'ы в namespace `knp`. Создаём Role `pod-reader` в `knp`, RoleBinding связывает `monitoring:monitoring` (ServiceAccount в namespace) с `pod-reader`. SA получает права только в `knp`, никаких других namespace'ов не видит.

**ServiceAccount и Pod**: каждому Pod'у можно указать SA (`spec.serviceAccountName`). Kubelet вмонтирует JWT-токен этого SA в `/var/run/secrets/kubernetes.io/serviceaccount/token`. Приложение читает и использует для запросов к apiserver.

**Bound Service Account Tokens** (K8s 1.22+) — токены короткоживущие (1 час), ротируются автоматически через `TokenRequest` API + projected volume. До 1.22 были долгоживущие Secrets — уязвимость, если утёк — навсегда действителен.

**Default SA — типовая гоча**. Если не указать `serviceAccountName`, Pod получает SA `default` из своего namespace. По умолчанию `default` SA никаких прав не имеет. Но многие legacy-приложения предполагают что могут читать apiserver — и падают. Best practice: явно указывать SA для каждого Deployment, минимальные permissions. Никогда не давать `cluster-admin` кроме как для настоящих admin-tools.

## Admission controllers: последняя защита перед etcd

Admission — плагины, работающие после auth/authz, до записи в etcd. Два типа: **Mutating** (могут менять объект) и **Validating** (только да/нет). Порядок: сначала все mutating (в порядке имён), потом все validating.

Built-in примеры: **NamespaceLifecycle** (нельзя создавать в удаляющемся namespace), **LimitRanger** (применяет LimitRange — default requests/limits), **ServiceAccount** (вмонтирует default SA если не указан), **DefaultStorageClass** (для PVC без класса ставит default), **PodSecurity** (Pod Security Standards), **ResourceQuota** (проверка ResourceQuota в namespace).

**Dynamic (webhook) admission** — пишешь свой HTTP-сервис, регистрируешь через `MutatingWebhookConfiguration` / `ValidatingWebhookConfiguration`. Apiserver для каждого запроса шлёт webhook AdmissionReview, тот отвечает allow/deny (или patched object для mutating).

`failurePolicy: Fail` — если webhook недоступен, запрос отклоняется. `Ignore` — пропускается (небезопасно для security webhook'ов).

**Опасность**: если webhook смотрит на все Pod'ы (включая kube-system) и падает — control plane может не подняться после рестарта (кубелеты создают mirror pods, apiserver вызывает webhook, тот недоступен, mirror pod не создастся). Best practice: `namespaceSelector` исключающий системные ns:

```yaml
namespaceSelector:
  matchExpressions:
  - key: kubernetes.io/metadata.name
    operator: NotIn
    values: [kube-system, kube-public]
```

**OPA Gatekeeper и Kyverno** — policy engines поверх admission. Вместо писать webhook на Go, описываешь политики декларативно. Gatekeeper на языке Rego (мощно, крутая обучения). Kyverno на YAML — проще, встроенная валидация типа «Pod должен иметь label owner».

**Pod Security Admission (PSA)** заменил PodSecurityPolicy (deprecated 1.21, удалён 1.25). Три уровня:

- **privileged** — всё разрешено. Для инфры типа kube-proxy.
- **baseline** — минимальные ограничения (нельзя hostNetwork, hostPID, hostIPC, privileged).
- **restricted** — строгий (RunAsNonRoot, dropCapabilities ALL, readOnlyRootFilesystem, seccomp).

Включается через label на namespace:

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

`enforce` блокирует, `warn` предупреждает в CLI, `audit` пишет в audit log. Best practice в prod: `baseline enforce, restricted warn+audit`, постепенно перевести на `restricted enforce`.

## CSI: как diski доезжают до Pod'ов

CSI (Container Storage Interface) — стандартный gRPC API между K8s и провайдерами хранилищ. Заменил in-tree drivers (AWS EBS, GCE PD внутри K8s) с версии 1.13.

Три компонента CSI driver: **Controller** (Deployment/StatefulSet 1-2 replicas, общается с cloud API, создаёт/удаляет volumes), **Node** (DaemonSet на каждой ноде, attach/mount volumes к Pod'ам), плюс sidecars от K8s: `external-provisioner`, `external-attacher`, `external-resizer`, `external-snapshotter`, `node-driver-registrar`.

**Жизненный цикл PVC**:

1. `kubectl apply -f pvc.yaml` — создаётся PVC (Pending).
2. external-provisioner видит: PVC без PV → зовёт CSI Controller `CreateVolume` → создаётся реальный disk в cloud → создаётся PV → bound с PVC.
3. Pod с этим PVC назначается на ноду.
4. external-attacher зовёт CSI Controller `ControllerPublishVolume` → cloud API attach'ит disk к VM ноды.
5. CSI Node зовёт `NodeStageVolume` (format, mkfs) → `NodePublishVolume` (bind mount в Pod).
6. Pod стартует, видит volume.

**Volume snapshots** — CSI поддерживает через `VolumeSnapshot`. Из snapshot можно создать новый PVC — быстрое восстановление БД.

**Practical gotchas**. RWO + StatefulSet: PVC привязан к одной ноде, если нода умерла — Pod pending пока K8s не decommission старую и attach на новую. Cloud attach/detach ~2 минуты. **fsGroup** в Pod securityContext делает chown на volume при mount — на больших volume очень медленно. Fix: `fsGroupChangePolicy: OnRootMismatch` (K8s 1.23+), chown только если корень не совпадает. **VolumeAttachment stuck**: при NodeNotReady VolumeAttachment висит, новый Pod с PVC — Pending. Помогает `kubectl delete volumeattachment` или ждать TIMEOUT (6 минут).

## Autoscaling: HPA, VPA, Cluster Autoscaler, KEDA

**HPA** (Horizontal Pod Autoscaler) — масштабирует replicas Deployment/StatefulSet по метрикам. Формула:

```
desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))
```

При `replicas=3`, CPU utilization 90%, target 60%: `desired = ceil(3 × 90/60) = ceil(4.5) = 5`. Скейл проверяется каждые 15 сек. **Tolerance 10%** — если ratio в [0.9, 1.1] ничего не делаем (защита от флаппинга).

Custom metrics (RPS, queue length) через `metrics.k8s.io/v1beta1` (resource metrics) или `custom.metrics.k8s.io/v1beta1` (обычно Prometheus adapter).

**HPA behavior** (K8s 1.18+) — тонкая настройка cooldown и rate:

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100
      periodSeconds: 15
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60
```

«Scale up — за 15 сек можем удвоить, немедленно. Scale down — за минуту можем убрать 10%, но только после 5 минут стабильно низкой нагрузки». Защита от резких качелей.

**VPA** (Vertical Pod Autoscaler) — меняет requests/limits Pod'ов на основе исторического использования. Компоненты: Recommender (анализирует метрики), Updater (evicts Pod'ы если сильно отклоняются), Admission Controller (на новых Pod'ах ставит recommended values). Modes: Off (только рекомендации, посмотреть), Initial (только при создании нового Pod'а), Auto (evicts и пересоздаёт — не для критичных, downtime).

**HPA + VPA на одной метрике** — конфликт (оба хотят реагировать на CPU). Либо разделяй метрики (HPA на custom, VPA на CPU), либо используй только один.

**Cluster Autoscaler** — добавляет/убирает Nodes кластера. Смотрит: Pod'ы Pending из-за insufficient resources → добавляет Node (через AWS ASG / GCE MIG / Azure VMSS). Смотрит: Node underutilized долго → cordon + drain + delete.

**Karpenter** (AWS) — замена Cluster Autoscaler. Быстрее (не ждёт ASG), может провизить разные instance types on-demand под потребности, меньше конфигурации.

**KEDA** — Event-Driven Autoscaling. Расширяет HPA scaling'ом на основе событий из внешних систем: длина очереди RabbitMQ/Kafka, количество сообщений в SQS, custom Prometheus queries. Позволяет масштабировать до 0 replicas (HPA — минимум 1). Пример для Kafka consumer:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: {name: kafka-consumer}
spec:
  scaleTargetRef: {name: kafka-consumer}
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

При lag > 100 сообщений добавляет Pod'ы, при малом lag'е убирает вплоть до нуля.

## CRD и Operator pattern

**CRD** (CustomResourceDefinition) — кастомный тип ресурса, расширяет K8s API объектами домена. После apply CRD с `kind: Database` — можно писать `Database` объекты. Но без controller'а они просто лежат в etcd. Обычно рядом с CRD пишут **operator** — controller, знающий что делать с этим типом.

**Operator = CRD + Controller** (обычно один binary). Инкапсулирует expertise по конкретному приложению. Примеры: postgres-operator (Zalando, Crunchy) — управляет Postgres-кластерами через Patroni, backup в S3, PIT restore. kafka-operator (Strimzi) — Kafka+Zookeeper кластеры. elasticsearch-operator — ES кластеры + snapshot policies. prometheus-operator — Prometheus + Alertmanager + ServiceMonitor. cert-manager — сертификаты Let's Encrypt.

**Kubebuilder / operator-sdk** — фреймворки для написания operator'ов. Дают scaffolding (генерация boilerplate), обёртку над client-go, дебаг через `make run` локально против кластера, deploy как manager Pod.

**Levels of Operator Maturity** (CoreOS) — пирамида: Basic install (Deploy CRD, StatefulSet) → Seamless upgrades (Rolling update, миграции) → Full lifecycle (Backup/restore, failover) → Deep insights (Metrics, alerts, tracing) → Auto pilot (Self-tuning, healing, capacity planning).

## HA control plane

**Stacked vs external etcd**. Stacked — etcd работает на тех же нодах, что и apiserver/scheduler/controller-manager. 3 master нод = 3 etcd + 3 apiserver. Плюсы: меньше нод, проще. Минусы: смерть control plane ноды = смерть etcd члена одновременно. External etcd — отдельный кластер из 3-5 нод etcd, control plane отдельно. Промышленный стандарт для крупных кластеров.

Как выглядит HA:

```
        ┌─── HA LoadBalancer (haproxy/AWS ELB) ────┐
        │  vip: control-plane.example.com:6443     │
        └───┬────────────┬──────────────┬──────────┘
            │            │              │
        ┌───▼───┐    ┌───▼───┐      ┌───▼───┐
        │master1│    │master2│      │master3│
        │apiserv│    │apiserv│      │apiserv│
        │sched  │    │sched  │      │sched  │
        │cm     │    │cm     │      │cm     │
        │etcd   │    │etcd   │      │etcd   │
        └───────┘    └───────┘      └───────┘
```

- **apiserver** — stateless, все три активны, LB балансирует.
- **scheduler** и **controller-manager** — leader-election через `Lease` object, только один активен.
- **etcd** — Raft, один leader, все три реплицируют.

**Kubelet и control-plane смерть**. Что происходит если весь control plane упал: kubelet продолжает работать (у него локальный кэш podspec'ов). Существующие Pod'ы бегут, Service работает (kube-proxy уже настроил iptables). Умрёт Pod → kubelet **не сможет** его переpull'ить (нет apiserver для чтения image auth), но попытается запустить из локального image. Новые деплои не идут. Вывод: краткий сбой control plane — не катастрофа. Долгий (часы) — проблема, но данные не теряются.

## Cgroups и namespaces: что реально изолирует контейнер

Контейнер — не виртуалка, а обычный процесс Linux с ограничениями через **namespaces** (что видит) и **cgroups** (сколько может).

**Namespaces** — ядерные конструкции, каждый процесс принадлежит нескольким namespace'ам одного типа. PID — свой набор процессов, `ps` внутри контейнера видит только «свои», PID 1 — главный процесс. NET — свой сетевой стек: интерфейсы, IP, iptables, sockets. MNT — своя иерархия монтирования. UTS — hostname. IPC — свои очереди сообщений, semaphores. USER — свой UID/GID mapping (root в контейнере — не root на хосте если настроено). CGROUP — своё представление cgroups. TIME (Linux 5.6+) — свои clocks.

Pod = процессы, разделяющие NET, IPC, UTS, USER — но не PID и не MNT (у каждого контейнера свои).

**Cgroups** (v1 или v2 в новых kernel) ограничивают потребление: cpu (CPU shares, CFS quota, throttling), memory (hard limit, soft limit, OOM), pids (max processes), io (throttle block I/O), hugetlb, cpuset.

`limits.cpu: "2"` → cgroup CFS quota: `cpu.cfs_quota_us=200000, cpu.cfs_period_us=100000` → максимум 200% CPU за период (2 полных ядра). **Throttling**: если превышаешь quota в моменте, приложение тормозится (не kill). У Java это болезненно (GC не может доработать). Метрика — `container_cpu_cfs_throttled_periods_total`. Часто рекомендуется **не ставить CPU limit вообще**, только request (гарантия) — пусть Pod жрёт свободные CPU когда они есть.

`limits.memory: 1Gi` → cgroup `memory.limit_in_bytes = 1073741824`. При превышении — OOM killer в ядре убивает процесс из этого cgroup. Разбор в файле 78.

**SecurityContext в Pod** — минимум для prod:

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
    add: ["NET_BIND_SERVICE"]
  seccompProfile:
    type: RuntimeDefault
```

`readOnlyRootFilesystem` — rootfs mount read-only, только `/tmp` и volumes writable. `capabilities.drop: [ALL]` — снять все Linux capabilities. Обычно оставляют только `NET_BIND_SERVICE` если нужен привилегированный порт < 1024. `seccomp RuntimeDefault` — containerd имеет default seccomp profile, ограничивает набор допустимых syscalls. `allowPrivilegeEscalation: false` — setuid binary не сможет получить root. PSA `restricted` требует именно эти настройки.

## Секреты и Vault

Проблемы дефолтных K8s Secrets: base64 в etcd (не шифрование), cluster-admin читает всё, нет ротации, нет аудита кто читал.

Решения. **Encryption at rest** — уже разобрано (KMS integration через `EncryptionConfiguration`). Обязательный минимум для prod.

**Sealed Secrets** (Bitnami) — шифруешь Secret в git через public key, кластер расшифровывает при apply приватным. Плюс: можно коммитить в git безопасно. Минус: ротация ключа тяжёлая, потеря приватного ключа = потеря секретов.

**External Secrets Operator** — CRD `ExternalSecret` синхронизирует из внешних хранилищ (Vault, AWS Secrets Manager, GCP Secret Manager) в K8s Secret. Актуальный best practice.

**Vault Agent Injector** — sidecar от HashiCorp, инжектит секреты в Pod'ы напрямую без создания K8s Secret. Через init container fetch, mount на emptyDir.

**K8s auth в Vault** — Vault знает про K8s: SA JWT-токен Pod'а → Vault валидирует у K8s TokenReview API → выдаёт Vault-токен. Pod читает секреты по SA identity, не по паролю. Ротация автоматически через bound tokens.

## Observability essentials

Что мониторить: Node (CPU/mem/disk/network per node), Pod (CPU/mem/restart count), Container (throttling, OOMKilled), kubelet (pleg latency, sync duration), apiserver (request rate/latency/errors), etcd (WAL fsync latency, DB size, leader elections), Scheduler (scheduling latency, pending pods).

**Метрики**:
- **metrics-server** — базовые CPU/mem, `kubectl top` использует.
- **kube-state-metrics** — состояние объектов (Deployment replicas, Pod phase, PVC size).
- **cAdvisor** — встроен в kubelet, метрики контейнеров.
- **node-exporter** — метрики Linux (system).

Всё → **Prometheus** → **Grafana**.

**Логи**: стандарт — log stdout/stderr → kubelet перекладывает в `/var/log/pods/...` → DaemonSet агент собирает. Агенты: Fluent Bit (лёгкий, C, стандарт), Fluentd (Ruby, старше), Vector (новый, Rust), Promtail (для Loki). Отправляют → Elasticsearch/Loki/CloudWatch → Kibana/Grafana.

**Трейсинг**: OpenTelemetry — стандарт SDK и Collector. Jaeger / Tempo / Zipkin — trace store + UI. Важны для debugging «где тормозит» в микросервисах.

**Events** — короткоживущие (1 час по умолчанию). `kubectl get events -n ns --sort-by='.lastTimestamp'`. Долгосрочное хранение — через `kube-events-exporter` → Loki/Elasticsearch.

## Практический debug: типовые сценарии

**Не поднимается Pod** — алгоритм из файла 78: `kubectl describe pod` → Events (FailedScheduling / FailedCreatePodSandbox / ImagePullBackOff / FailedMount / Error), `kubectl logs --previous` если статус crash, `kubectl exec` если жив, `kubectl debug -it X --image=busybox --target=app` через ephemeral containers если основной контейнер distroless без shell.

**Не идёт трафик до Service**:

```bash
# Endpoints есть?
kubectl get endpoints svc-name -n ns
# Если пусто — все Pod'ы NotReady или selector не совпадает.

# Селектор совпадает?
kubectl get svc svc-name -n ns -o yaml | grep -A5 selector
kubectl get pods -n ns --show-labels

# Readiness работает?
kubectl describe pod X -n ns | grep -A5 Readiness

# С другой Pod
kubectl run -it --rm debug --image=nicolaka/netshoot -- sh
# внутри: curl svc-name.ns.svc.cluster.local
# внутри: nslookup svc-name.ns

# На ноде kube-proxy правила
iptables-save | grep <clusterIP>
```

**Ноды NotReady**:

```bash
kubectl describe node worker-3
# Смотри Conditions: MemoryPressure, DiskPressure, PIDPressure, Ready

# На ноде — kubelet живой?
systemctl status kubelet
journalctl -u kubelet --since "10 minutes ago"

# containerd жив?
systemctl status containerd
crictl ps
```

**Полный кластер не отвечает**. `kubectl` возвращает `connection refused` → apiserver умер, `systemctl status kube-apiserver` на master. Apiserver жив, `etcdserver: request timed out` → etcd упал, `etcdctl endpoint status`. Pod-to-pod через ClusterIP работает, `curl svc-name` не резолвит → DNS сломан, `kubectl -n kube-system get pods -l k8s-app=kube-dns`.

## Собесные вопросы с осмысленными ответами

**Что такое Pod и почему это не контейнер?** Pod — минимальная единица планирования в K8s. Обычно 1 контейнер, но может быть несколько (sidecar). Контейнеры в Pod'е шарят network namespace (один IP, доступ по localhost), IPC, volumes; не шарят MNT (свои файловые системы) и PID (по умолчанию). Не контейнер, потому что K8s часто нужно запускать связанные процессы вместе (application + envoy proxy) — уровень абстракции выше.

**Разница liveness и readiness probe?** Liveness: «жив ли контейнер». Fail → kubelet перезапускает контейнер. Используется когда приложение может залипнуть. Readiness: «готов ли принимать трафик». Fail → Pod исключается из endpoints Service'а, трафик не идёт, но Pod не рестартится. Общее правило: **liveness не должен зависеть от внешних систем** (иначе флап БД → каскадный restart всех pod'ов). Readiness — может.

**Что происходит при `kubectl apply -f deploy.yaml`?** kubectl вычисляет diff с текущим объектом (3-way merge), отправляет PATCH на apiserver. Apiserver: auth → authz (RBAC) → mutating admission → validation → validating admission → etcd write. Deployment controller видит через watch → reconcile: если spec.template отличается — создаёт новый ReplicaSet, начинает rolling update, скейлит старый вниз, новый вверх согласно strategy. ReplicaSet controller создаёт Pod'ы. Scheduler назначает nodeName. Kubelet на ноде запускает через CRI.

**Что произойдёт если Pod превысит memory limit?** OOMKilled: Linux OOM killer в ядре убивает процесс из cgroup, exit 137. Kubelet видит exit → рестартит. Если рестарты частые — CrashLoopBackOff, exponential backoff. Fix: увеличить limit + правильно настроить `-Xmx` (~70% от limit), см. файлы 77, 78.

**Что произойдёт если Pod превысит CPU limit?** **Throttling**, не kill. Linux CFS scheduler ограничивает CPU time, приложение работает медленнее но живёт. Особенно болезненно для JVM: GC не может отработать в отведённом окне → длинные паузы. Метрика: `container_cpu_cfs_throttled_periods_total`. Часто рекомендуется **не ставить CPU limit вообще**, только request.

**Как безопасно катить breaking-change API?** Expand-contract pattern (parallel change). Deploy backward-compatible изменения → мигрировать всех потребителей → deploy final change (убрать старую совместимость). Для БД: добавить колонку → deploy код пишущий и в старое и в новое → backfill → deploy код читающий из нового → deploy код не пишущий в старое → удалить старую колонку. Никогда одним PR: «изменил схему» + «изменил код» в одном деплое.

**Как реализован leader election в K8s?** Через **Lease** object (`coordination.k8s.io/v1`). Клиенты пытаются `Get` текущего Lease, если `holderIdentity` устарел (`renewTime + leaseDurationSeconds < now`) — делают `Update` с собой как holder. Если Update успешен (optimistic concurrency через resourceVersion) — ты leader. Периодически renew чтобы не потерять. При shutdown — clear `holderIdentity` для быстрого transfer.

**Что такое finalizers?** Строка в `metadata.finalizers`. Когда delete приходит на объект — apiserver ставит `metadata.deletionTimestamp` (soft delete), но объект не удаляется пока finalizers не пусты. Пример: PVC имеет finalizer `kubernetes.io/pvc-protection` — пока Pod использует PVC, delete не завершается. Свои controllers могут использовать finalizers для гарантированной очистки внешних ресурсов перед удалением объекта.

**Как работают mutating и validating admission webhooks?** Apiserver после auth/authz идёт по цепочке: все built-in mutating → зарегистрированные mutating webhooks → OpenAPI-схема → built-in validating → зарегистрированные validating webhooks. Webhook — HTTPS-сервис, apiserver шлёт POST с AdmissionReview, ответ: allowed true/false + patch для mutating. `failurePolicy: Fail` = webhook недоступен → отклонить (безопасно для validation), `Ignore` — пропустить (опасно для security). Timeout default 10s, max 30s. Ловушка: webhook на все Pod'ы + падение = кластер не поднимется после reboot, лечится `namespaceSelector` исключающим kube-system.

**Как правильно zero-downtime rolling update Spring Boot?** Полный чек-лист был в файле 77: `server.shutdown: graceful` + `spring.lifecycle.timeout-per-shutdown-phase: 25s` в Boot; в K8s Deployment `maxSurge: 1, maxUnavailable: 0`, `terminationGracePeriodSeconds: 45`, preStop `sleep 10` (даёт kube-proxy обновить iptables), readiness на `/actuator/health/readiness`, liveness на `/actuator/health/liveness` (не readiness — не должна зависеть от БД), startupProbe для медленного старта. PDB `minAvailable: replicas - 1`.

**Как отладить «Pod не принимает трафик хотя Running»?** Пошагово: (1) Pod Ready? `kubectl get pod X`, если 0/1 — readiness fail. (2) Pod в endpoints? `kubectl get endpoints svc`. (3) Selector совпадает? `kubectl get svc -o jsonpath='{.spec.selector}'` vs `kubectl get pod --show-labels`. (4) NetworkPolicy не блокирует? `kubectl get networkpolicy`. (5) iptables rules на ноде клиента? `iptables-save | grep <cluster-ip>`. (6) CoreDNS резолвит? `kubectl run debug --rm -it --image=nicolaka/netshoot -- nslookup svc-name.ns`. (7) kube-proxy живой?

## Best practices — минимум для prod

Для каждого Deployment: replicas ≥ 2 (лучше 3), requests + limits (Guaranteed QoS), liveness + readiness probes (правильно разделённые), startupProbe для медленных, terminationGracePeriodSeconds > shutdown-timeout, preStop hook `sleep 5-10`, PodDisruptionBudget с `minAvailable: replicas - 1`, podAntiAffinity по hostname/zone, priorityClassName, securityContext (runAsNonRoot, readOnlyRootFilesystem, capabilities.drop [ALL]), imagePullPolicy IfNotPresent (не Always — пылит рестарт), метки для kubectl get и NetworkPolicy, annotations для prometheus scrape.

Anti-patterns, за которые платят: **`latest` tag** в image (каждый Pod может подтянуть разное), один Pod с многими контейнерами без причины, hostNetwork/hostPID/privileged в prod (только для инфры), BestEffort QoS (первые кандидаты на eviction), replicas: 1 для критичных сервисов, отсутствие resource limits (один Pod съест ноду), отсутствие readiness (rolling update отправит трафик в непрогретое).

## Заключение

Kubernetes — распределённая desired-state машина, работающая через четыре роли: **etcd (истина) → apiserver (единственная дверь) → controllers (приводят реальность к spec) → kubelet (запускает)**. Всё остальное — вариации: scheduler = controller, HPA = controller, Ingress controller = controller, operator = controller. Понял reconcile loop с informer/workqueue — понял 80% K8s.

Etcd — Raft, MVCC, watch, encryption at rest, обязательный backup. Три ноды переживают одну потерю, пять — две, чётное бесполезно. Compaction обязателен, иначе space quota убьёт кластер.

Apiserver — единственный who talks to etcd, path запроса: auth → authz → mutating admission → validation → validating admission → etcd write → watch notifications. APF защищает от флуда.

Scheduler — framework с 12 extension points, Filter + Score + Bind основные. Управление размещением через nodeSelector, nodeAffinity, podAntiAffinity, taints/tolerations, TopologySpreadConstraints. Preemption для priority.

Controller pattern — informer → workqueue → worker → reconcile. Level-triggered, идемпотентно, устойчиво к сбоям. Finalizers для внешних ресурсов, leader election через Lease.

Kubelet — syncLoop с PLEG, CRI-протокол к containerd, containerd-shim + runc → cgroups + namespaces. Static Pods для bootstrap control plane.

Kube-proxy — iptables (дефолт, линейный поиск) / IPVS (O(1) хеш) / nftables (новый) / eBPF (Cilium — полная замена). EndpointSlices заменили Endpoints для scaling. externalTrafficPolicy Local сохраняет source IP, но требует health check в внешнем LB.

CNI — модель flat сеть без NAT, реализация через Flannel (VXLAN, простой), Calico (BGP, производительный), Cilium (eBPF, современный). CoreDNS с ndots:5 и search domains — источник частых DNS-тормозов, оптимизация через FQDN с точкой или `dnsConfig`.

RBAC: Role/ClusterRole + Binding, ServiceAccount с bound tokens (1 час, K8s 1.22+). Admission — mutating + validating, webhooks с failurePolicy, Pod Security Admission (baseline enforce + restricted warn — стандарт для prod).

CSI — controller (cloud API) + node (per-node), sidecars от K8s для provision/attach/resize/snapshot. Autoscaling: HPA (formula с 10% tolerance + behavior), VPA (три mode, не с HPA на одной метрике), Cluster Autoscaler / Karpenter, KEDA (scaling to zero по внешним событиям).

CRD + operator — expertise по приложению как код. Kubebuilder / operator-sdk для написания.

HA — external etcd для крупных кластеров, apiserver stateless за LB, scheduler+CM с leader election. Kratkij сбой control plane не убивает работающие Pod'ы.

Диагностика — describe/logs/events для 90% случаев, ephemeral containers для distroless, `pg_stat_activity` и thread dumps для более глубоких. Все инциденты алгоритмичны.

Для дальнейшего погружения — читать код `kubernetes/kubernetes` на GitHub (компоненты в `cmd/`), *Kubernetes in Action* (Marko Lukša, 2nd ed 2023), блоги Kelsey Hightower и Brendan Burns. Для практики — `kubernetes-the-hard-way` от Kelsey Hightower: собрать K8s руками из компонентов, без kubeadm. После этого «внутренности» перестают быть абстракцией.
