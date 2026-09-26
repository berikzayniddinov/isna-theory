# 77. Kubernetes для микросервисов: workloads, network, disruption, graceful shutdown

## Зачем это знать

Kubernetes для одного сервиса — это `kubectl apply -f deployment.yaml` и всё. Kubernetes для десяти микросервисов — это уже вопрос: как их правильно раскладывать по типам workload'ов, как контролировать сетевые связи, как переживать node drain без даунтайма, как правильно останавливаться, как раздавать приоритеты когда кластер тесно. Ответы на эти вопросы формируют разницу между «прод пятилетней давности» (руками кубик подпираем) и «прод который перезапускается сам и о котором не думаешь ночью».

Базовые концепции (Pod, Deployment, Service) — в `10-kubernetes-detailed.md`. Здесь — продвинутый пласт, который начинает быть нужен когда сервисов больше пары и SRE-практики становятся обязательными.

Разберём workloads: почему `Deployment` не подходит базе, зачем `StatefulSet` даёт стабильные имена, чем отличается `DaemonSet` и когда нужен `Job` — со сценариями где выбор неправильного типа стоит инцидента. Пройдёмся по storage: PV, PVC, StorageClass, access modes, reclaim policy — с историей как `reclaimPolicy: Delete` на базе стоил компаниям данных. Сеть: Service, Ingress, NetworkPolicy — как ограничивать pod-to-pod трафик так, чтобы взлом одного сервиса не превращался во взлом всего кластера. PodDisruptionBudget и Priority — про переживание обслуживания нод и приоритезацию при перегрузке. QoS classes — почему `requests == limits` для критичных сервисов. Graceful shutdown — где именно race condition, как правильно ставить preStop, как согласовывать `terminationGracePeriodSeconds` с Spring Boot `shutdown-timeout`. Probes: разница liveness / readiness / startup, типичный анти-паттерн когда liveness тянет за собой каскадный отказ. И полный шаблон deployment'а, собирающий всё.

## Workloads: почему тип имеет значение

Kubernetes даёт четыре основных типа workload'ов, каждый под свой класс приложений. Ошибка на этом уровне не всегда сразу бьёт по проду, но становится больной когда доходит до масштабирования или инцидента.

**Deployment** — про stateless. Реплики полностью взаимозаменяемы: `pod-a-xyz`, `pod-a-abc`, IP меняются, имена — хеши. Rolling update убивает старые поды и поднимает новые в любом порядке, потому что «первый» и «второй» ничем не отличаются. Работает для 90% микросервисов: обработал HTTP-запрос, забыл всё, следующий может уйти в другой pod. Никаких persistent данных на самом pod'е — всё что нужно, хранится в базе или объектном хранилище снаружи.

**StatefulSet** — когда identity и порядок важны. Классические примеры: PostgreSQL, Kafka, Elasticsearch, Redis Cluster, Zookeeper. Не потому что «это базы», а потому что реплики этих систем **не взаимозаменяемы** — они делят состояние, у каждой своя роль (лидер/реплика), у каждой свой отдельный кусок данных. StatefulSet даёт стабильные имена (`postgres-0`, `postgres-1`, `postgres-2`), упорядоченный старт (postgres-1 не запустится пока postgres-0 не Ready) и упорядоченный shutdown (в обратном порядке). Каждый pod получает свой персональный PVC через `volumeClaimTemplates` — при удалении pod'а том остаётся, при пересоздании pod'а с тем же именем прицепится тот же том. Плюс headless Service даёт DNS до конкретного pod'а: `postgres-0.postgres.default.svc.cluster.local`.

Что критично понимать: **StatefulSet ≠ автоматическая кластеризация**. Ты получаешь только стабильную идентичность и приватные тома. Streaming replication в PostgreSQL, broker discovery в Kafka, cluster.conf в Redis — это всё ты сам настраиваешь в приложении/конфигурации. StatefulSet — это лишь фундамент, поверх которого твоё приложение может опираться на факт «я знаю кто я, у меня стабильное имя, мой диск не убежит».

**DaemonSet** — по одному pod'у на каждой ноде. Инфраструктурная штука: Fluent Bit собирает логи со всех нод, node-exporter снимает метрики железа, kube-proxy и CNI-плагин обеспечивают сеть. Когда добавляется новая нода — DaemonSet автоматически шедулит на неё pod. Это единственный корректный способ развернуть «агента на каждой ноде», потому что Deployment не знает про топологию, а вручную считать реплики под каждый scale-up кластера — путь в никуда.

**Job и CronJob** — для одноразовых или расписанных задач. Job запускает pod, ждёт `exit 0`, считает готово. Классика: миграции Liquibase/Flyway перед деплоем, backfill данных, ротация ключей. CronJob — тот же Job, но по крону. Важная деталь: Job без `restartPolicy: OnFailure` не будет перезапускать pod при падении, а `backoffLimit` (по умолчанию 6) ограничивает попытки. `activeDeadlineSeconds` защищает от зависших Job'ов, которые могут крутиться днями.

Типовая ошибка: разворачивать миграции как `initContainer` в основном Deployment'е. При replicas > 1 все поды параллельно попытаются применить миграции — Liquibase возьмёт advisory lock, но остальные всё равно будут ждать, старт растянется. Правильно — вынести миграции в отдельный Job, а Deployment запускать после успешного завершения (через Argo Workflows, Helm hooks или CI-пайплайн).

## Storage: PV, PVC, StorageClass и цена ошибки

Kubernetes абстрагирует диски через три слоя. **PersistentVolume (PV)** — реальный кусок диска: EBS-том в AWS, disk в Azure/GCP, локальный SSD ноды, NFS-шара. Живёт вне pod'а, ресурс кластерного уровня. **PersistentVolumeClaim (PVC)** — запрос от pod'а: «мне 100Gi, RWO, класс fast-ssd». Kubernetes находит подходящий PV и связывает. **StorageClass** — шаблон для динамического provisioning: не заранее выделенный PV, а параметры «как создать» — тип диска в облаке, репликация, шифрование. При создании PVC под StorageClass кластер сам провизионит PV нужного размера.

Access modes определяют, кто может писать. **ReadWriteOnce (RWO)** — один pod на конкретной ноде. Это тот случай, когда AWS EBS или Azure Disk может быть примонтирован только к одному инстансу. Стандарт для баз. **ReadOnlyMany (ROX)** — много pod'ов могут читать. Редкий кейс, например shared reference data. **ReadWriteMany (RWX)** — много pod'ов пишут и читают. Требует shared filesystem: NFS, CephFS, EFS. Медленно, дорого, эксплуатационно сложно. Обычно если хочется RWX — надо задать вопрос «а точно нельзя переложить на объектное хранилище S3/MinIO?».

Reclaim policy определяет что случится с реальным диском при удалении PVC. **Retain** — PV и диск остаются, надо чистить руками. Для важных данных. **Delete** — PV и реальный диск исчезают. Для эфемерных.

И вот здесь — классика продовой катастрофы. Разработчик написал Helm chart для PostgreSQL, поставил StorageClass с `reclaimPolicy: Delete` (потому что дефолт AWS — Delete). Год работы, база наполнилась. Кто-то делает `helm uninstall` (например, чтобы переустановить с новой конфигурацией). Helm сносит все ресурсы, включая PVC. Kubernetes видит `reclaimPolicy: Delete` и удаляет и PV, и реальный EBS-том. Данные ушли. Backup был вчера, потерян день.

Правило: **для любых persistent данных — `reclaimPolicy: Retain`**. Всегда. Даже если это стейджинг — привычка одна. Восстановление тома при неправильной policy — не всегда возможно даже через AWS support.

Помимо PV есть тома, не требующие persistent хранения. `emptyDir` — временный диск на ноде, живёт пока pod живёт, используется для scratch и кэшей. `hostPath` — том прямо с файловой системы ноды (`/var/log`, `/var/run/docker.sock`). Опасен: pod получает доступ к хосту, риск escape'а. Использовать только в infra DaemonSet'ах. `configMap` и `secret` монтируются как файлы с содержимым из соответствующего объекта Kubernetes — конфиги и секреты приложение читает как обычные файлы, без специального SDK.

## Network: Service, Ingress, NetworkPolicy

Внутрикластерная связность — через Service. `ClusterIP` даёт виртуальный IP + DNS `<name>.<ns>.svc.cluster.local`, балансирует запросы между pod'ами по selector. `NodePort` открывает порт на каждой ноде — для debug/dev, в проде обычно избегается. `LoadBalancer` заказывает внешний балансировщик у облака (AWS ELB, GCP LB, MetalLB в on-prem). `Headless Service` (`clusterIP: None`) — не даёт виртуального IP, DNS-запрос возвращает список IP всех pod'ов напрямую; нужен для StatefulSet, где приложение хочет обращаться к конкретному pod'у.

Внешний вход в кластер идёт через **Ingress** — L7 роутер поверх nginx/Traefik/HAProxy. Один Ingress Controller висит на 80/443 всей ноды или в LoadBalancer'е, роутит запросы по host/path в нужные Services. Так один IP и один TLS-сертификат обслуживают десятки сервисов через разные хосты. С cert-manager сертификаты Let's Encrypt обновляются автоматически. У nginx-ingress традиционно больше настроек через аннотации; у Traefik — конфигурация ближе к декларативной и удобнее с CRD; выбор проектный, оба production-ready.

**NetworkPolicy** — L3/L4 firewall между pod'ами. По умолчанию Kubernetes даёт полный mesh: любой pod может ходить в любой pod. Это удобно для разработки и катастрофа для безопасности. Взломали один сервис — атакующий сразу может сканировать всю сеть кластера. NetworkPolicy позволяет описать: «pod'ы с меткой app=postgres принимают TCP:5432 только от pod'ов с метками app=isnaknpuser или app=isnaknpsync, всё остальное drop». Аналогично для egress.

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
    - podSelector: {matchLabels: {app: isnaknpuser}}
    - podSelector: {matchLabels: {app: isnaknpsync}}
    ports:
    - protocol: TCP
      port: 5432
```

Критическое ограничение: NetworkPolicy требует CNI-плагина с поддержкой. Calico, Cilium, Weave — работают. Стандартный kubenet или flannel в базовом режиме — игнорируют NetworkPolicy молча, правила «применяются» но ничего не блокируют. Первое что проверить в новом кластере: `kubectl get pods -n kube-system | grep -E 'calico|cilium|weave'`.

Правильная security-практика: **default-deny** в каждом namespace (пустой ingress = никто не может войти), плюс явные разрешения. Так каждый новый сервис обязан объявить кому он даёт доступ, и любой пропущенный default-allow становится видимым.

## PodDisruptionBudget: пережить node drain

Ноды кластера периодически надо обслуживать: обновление kernel, апгрейд kubelet, замена железа. Kubernetes для этого делает `drain` — эвакуирует все pod'ы с ноды на другие. Если у тебя `replicas: 3` и все три случайно оказались на одной ноде — drain убьёт все три одновременно, downtime гарантирован.

**PodDisruptionBudget (PDB)** — контракт «минимум N pod'ов моего приложения должны быть Available всегда, даже во время voluntary disruption». Когда kube-controller хочет эвакуировать pod, он спрашивает у PDB: «можно?». Если убийство pod'а нарушит PDB — drain ждёт.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: isnaknpuser-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels: {app: isnaknpuser}
```

Стандартная формула — `minAvailable: N-1` где N = replicas. Три реплики = минимум две доступны. Одна реплика — PDB невозможен: `minAvailable: 0` разрешает всё, `minAvailable: 1` навсегда блокирует drain. Отсюда правило: **для критичных сервисов всегда 2+ replicas**, иначе PDB не защитит.

Есть ещё «involuntary» disruptions — падение ноды, OOM ядра, hardware failure. PDB против них бессилен, потому что pod уже мёртв к моменту принятия решения. Защита от них — anti-affinity (см. дальше) и запас реплик.

## Priority и preemption: кого вытеснить

Кластер переполнен, новый critical pod не помещается. Что делает scheduler? По умолчанию — не шедулит, pod остаётся в Pending. С PriorityClass — может вытеснить (preempt) pod'ы с меньшим приоритетом, освободить место.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
```

Pod с высоким priority, для которого нет места, заставляет scheduler искать pod'ы с меньшим priority на нодах, evict'ить их (по факту убивать через SIGTERM с graceful period, а не SIGKILL) и запускать высокоприоритетный на освобождённое место. Вытесненные pod'ы попадают обратно в очередь scheduler'а и шедулятся туда где есть место — или ждут если места нет.

Стандартная иерархия: `system-cluster-critical` (2 000 000 000, для kube-system pod'ов), `high` (1000, критичные бизнес-сервисы), `medium` (500, обычные сервисы), `low` (100, batch/cron). Никогда не давать высокий приоритет всем — тогда система деградирует до состояния «никого нельзя вытеснить».

## QoS classes: кого убивать при OOM ноды

Каждый pod автоматически получает QoS class на основе того, как описаны его requests и limits. Kubernetes не спрашивает — вычисляет.

**Guaranteed** — `requests == limits` для всех контейнеров, всех ресурсов (memory и cpu). При OOM ноды (когда суммарное потребление pod'ов превысило доступную память) — таких убивают в последнюю очередь.

**Burstable** — где-то `requests < limits`. Обычная категория для сервисов, у которых пиковое потребление выше среднего. При OOM убивают после BestEffort, до Guaranteed.

**BestEffort** — requests не указаны нигде. Убивают первыми. По сути «я не гарантирую своих потребностей, съем что дадут».

Правило для прода: **все критичные сервисы — Guaranteed** с `requests == limits`. Так kubelet не будет их трогать при пресcении памяти ноды. Batch-джобы — Burstable нормально. BestEffort в проде — почти всегда неправильно, разве что для настоящих fire-and-forget задач которых не жалко.

```yaml
resources:
  requests:
    memory: "2Gi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "500m"
```

Отдельный tricky момент — CPU limits. `cpu: 500m` = 0.5 vCPU. Но `limits.cpu` реализуется через CFS quota в Linux и вводит throttling: если сервис попытался использовать больше — его ядро притормаживает даже если CPU свободен. Для латентно-чувствительных JVM-приложений это плохо: GC-паузы и warmup могут не влезать в quota. Многие в проде оставляют `requests.cpu` и **убирают `limits.cpu` вообще**, доверяя PriorityClass'у и affinity'ю распределять нагрузку.

## Graceful shutdown: где именно race condition

Когда k8s хочет убить pod, происходит примерно следующее. Kubelet помечает pod как `Terminating`. Endpoints-controller удаляет pod из endpoints Service — но это распространяется асинхронно, kube-proxy на каждой ноде обновит iptables/IPVS через 1-5 секунд. Одновременно kubelet выполняет `preStop` hook (если настроен) и посылает SIGTERM основному процессу. Дальше идёт отсчёт `terminationGracePeriodSeconds` (по умолчанию 30 сек). Если контейнер за это время не завершился — SIGKILL, никаких вопросов.

Race condition — между «удалили из endpoints» и «kube-proxy обновил правила». В этом окне (пара секунд) новые запросы всё ещё могут прилететь на умирающий pod, а он уже начал graceful shutdown, отказывается принимать. Клиент получает `Connection refused` или зависание, retry-логика балансировщика может перевести на другой pod, но лишний хвост ошибок в логах и пятисоток на графике обеспечен.

Правильный паттерн: **preStop hook с `sleep 5`**, потом SIGTERM. Порядок такой:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
```

Kubelet сначала полностью выполнит preStop (5 секунд простоя), только потом пошлёт SIGTERM. За эти 5 секунд kube-proxy на всех нодах успевает обновить правила, новые запросы перестают попадать на этот pod. Дальше идёт честный graceful shutdown.

Spring Boot 2.3+ поддерживает graceful shutdown нативно:

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
```

При SIGTERM Boot прекращает принимать новые HTTP-запросы, ждёт до 25 секунд пока текущие завершатся, потом закрывает контекст в правильном порядке (сначала веб, потом DataSource, потом остальное) и выходит с кодом 0.

Критическая арифметика: `terminationGracePeriodSeconds` в манифесте должен быть **больше** `timeout-per-shutdown-phase` в Boot + preStop sleep. Иначе SIGKILL прибьёт Boot посередине shutdown'а. Формула: `terminationGracePeriodSeconds ≥ preStop_sleep + Boot_shutdown_timeout + запас`. Для нашего примера: 5 + 25 + 15 = 45 секунд.

Что реально делать во время graceful shutdown в приложении: дождаться in-flight HTTP-запросов, отменить или дожать consumer'ы RabbitMQ/Kafka (ack то что успело, не подхватывать новое), закрыть connection pool JDBC (см. файл 107), закрыть коннекты к другим сервисам. Всё это Boot делает сам через SmartLifecycle-порядок бинов, но если у тебя своя логика (свой executor, свой WebSocket-сервер) — надо руками имплементить `SmartLifecycle`.

## Probes: liveness, readiness, startup — тонкости

Три probe с разными семантиками, и типовой анти-паттерн когда путают liveness и readiness — стоит инцидентов.

**Liveness** отвечает на вопрос «приложение живое или намертво зависло?». Если fail — kubelet **перезапускает контейнер**. Не pod, а именно контейнер (pod остаётся, restart count растёт). Правильный liveness проверяет **только** внутреннее состояние: JVM отвечает, event loop не заблокирован, deadlock не наступил.

Классическая ошибка — включить в liveness проверку зависимостей (БД, других сервисов). Что происходит при кратковременном сбое БД: readiness на всех pod'ах приложения красный (это правильно), liveness тоже красный (это неправильно) — kubelet начинает перезапускать все pod'ы. БД восстанавливается, но приложения все Restarting, каскадный отказ длиной в минуты. Правило: **liveness не зависит от внешних систем**. Только «сам процесс жив и отвечает».

**Readiness** — «готов ли pod принимать трафик?». Если fail — pod остаётся Running (не перезапускается!), но **удаляется из endpoints Service'а**. Балансировщик не шлёт на него новых запросов до восстановления. Readiness **может и должен** зависеть от внешних систем: БД недоступна — pod не готов, LB перенаправляет на другие реплики. Как только БД вернулась — readiness зелёный, pod снова в endpoints.

Spring Boot Actuator из коробки даёт `/actuator/health/liveness` и `/actuator/health/readiness`. Liveness по умолчанию отдаёт `LivenessState` (простой enum «жив/мёртв»). Readiness тянет `ReadinessStateHealthIndicator` + внешние indicators (по дефолту `DataSourceHealthIndicator` — проверка БД, `RedisHealthIndicator` и т.д.).

**Startup** — для медленно стартующих приложений. Пока startup не Ok — liveness и readiness не проверяются вообще. Даёт JVM время прогреться, кэши прогрузиться, JIT скомпилиться. `failureThreshold: 30, periodSeconds: 10` = 5 минут на старт. Без startup probe пришлось бы ставить огромный `initialDelaySeconds` на liveness — плохо, потому что если приложение реально зависло через минуту работы, liveness не сработает пока не пройдёт initial delay.

```yaml
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
```

## Init containers и sidecars

**Init container** — специальный контейнер в pod'е, который запускается **до** основного и должен выйти с exit 0, иначе основной не стартует. Init'ы выполняются последовательно один за другим. Классические применения:

- **wait-for-dependency**: `until nc -z postgres 5432; do sleep 1; done`. Не пускать приложение пока БД не отвечает. Полезно при холодном старте кластера.
- **fetch-secret**: сходить в Vault/AWS Secrets Manager, положить секрет на shared emptyDir, приложение потом читает файл.
- **run-migrations**: применить Liquibase/Flyway до того как приложение попытается работать со схемой.

Init для миграций проблематичен при replicas > 1: все pod'ы параллельно попытаются запустить миграции. Liquibase возьмёт advisory lock, но старт всё равно растянется, а если миграция сломается — все pod'ы в Init crashloop. Правильный подход — миграции отдельным Job'ом до раскатки Deployment'а.

**Sidecar** — второй (третий, четвёртый) контейнер в том же pod'е, работающий параллельно. Общий network namespace (localhost между контейнерами), общие volumes. Классические примеры:

- **Service mesh proxy** — Envoy/Istio-sidecar перехватывает исходящий трафик приложения, добавляет mTLS, retry, timeout, distributed tracing. Приложение шлёт plain HTTP на localhost, sidecar шифрует и отправляет дальше.
- **Config-reloader** — следит за ConfigMap через shared volume, шлёт SIGHUP главному контейнеру когда конфиг изменился.
- **Backup-agent** — раз в час снимает snapshot тома главного, льёт в S3.

Анти-паттерн: пихать всё в один pod через sidecar'ы «чтобы вместе жили». Sidecar — это когда два процесса **логически неразделимы**, должны стартовать и умирать вместе, делят локальные ресурсы. Если два процесса могут работать независимо — это два разных Deployment'а. Sidecar'ы усложняют graceful shutdown (все контейнеры получают SIGTERM одновременно, порядок остановки — с ловушками), увеличивают memory footprint и добавляют возможности отказов.

Kubernetes 1.28+ добавил `restartPolicy: Always` для init container'ов — это «нативные» sidecar'ы: запускаются до основных, живут параллельно, ждут окончания основных при shutdown. До этого приходилось эмулировать через обычные containers с ручной синхронизацией.

## Rolling update: как безопасно катить

Дефолтная стратегия Deployment — RollingUpdate: постепенное замещение старых pod'ов новыми, без даунтайма.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
```

`maxSurge` — сколько pod'ов сверх желаемого количества можно поднять во время update. `maxUnavailable` — сколько pod'ов ниже желаемого можно опустить. При `replicas: 4`, `maxSurge: 25%` = 1, `maxUnavailable: 25%` = 1: на пике 5 pod'ов, в моменте минимум 3 Ready.

Для zero-downtime часто ставят `maxUnavailable: 0, maxSurge: 1`: никогда не опускаемся ниже желаемого количества, поднимаем по одному новому, ждём Ready, только потом убиваем старый. Медленнее, но безопаснее для критичных сервисов.

Если новый pod не становится Ready (сломанная readiness probe, ошибка старта) — rollout зависает. `kubectl rollout status deployment/X` покажет. `kubectl rollout undo deployment/X` откатывает на предыдущую версию (или `--to-revision=N` на конкретную). `kubectl rollout history deployment/X` показывает историю.

Есть ещё стратегия `Recreate`: убить все старые pod'ы, потом запустить все новые. Даунтайм гарантирован. Используется только когда версии несовместимы одновременно (например, изменения схемы БД, требующие остановки всех старых). В обычной работе — не для прода.

Более продвинутые схемы (blue/green, canary) реализуются через Argo Rollouts, Flagger, или руками через два Deployment'а + переключение Service'а. Дают возможность катать 5% трафика на новую версию, смотреть метрики, потом 20%, 50%, 100%. Или держать два полных парка (blue/green) и переключаться атомарно.

## OOMKilled: расследование

Pod вышел с exit code 137. `kubectl describe pod X`:

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

Exit 137 = 128 + 9 (SIGKILL). Убил не Kubernetes, а Linux kernel через cgroup OOM killer — pod превысил `limits.memory`, kernel убил самый жирный процесс в cgroup'е. Kubelet видит exit, применяет restart policy (Always по умолчанию), pod перезапускается.

Первые шаги диагностики:

`kubectl top pod X` — сколько ест сейчас (после рестарта, свежая копия). Если после старта уже под limit — приложение растёт быстро. Если ест мало — рост был постепенный, ищи memory leak.

`kubectl describe pod X | grep -A5 "Last State"` — подтверждение что причина именно OOM, а не что-то другое (exit code 143 — SIGTERM, значит graceful shutdown; exit 1 — приложение упало само).

Heap dump в JVM — самая ценная диагностика. Пока pod жив:

```bash
kubectl exec pod-x -- jcmd 1 GC.heap_dump /tmp/heap.hprof
kubectl cp pod-x:/tmp/heap.hprof ./heap.hprof
```

Открывать в Eclipse MAT. Смотреть Dominator Tree — какой объект удерживает больше всего памяти. Классика: HashMap-кэш без TTL, ThreadLocal без cleanup, connection pool с открытыми ResultSet'ами, EhCache/Caffeine с неограниченной емкостью.

Правило по heap: **`-Xmx` должен быть меньше `limits.memory`**, обычно 70-80%. Пример: `limits.memory: 2Gi`, `-Xmx1500m`. Разница уходит на metaspace, direct memory (Netty, JDBC-драйверы), thread stacks (по 1MB на тред), JIT code cache, native libraries. Если поставить `-Xmx2G` = `limits.memory: 2Gi` — heap заполнится, JVM попытается взять ещё на metaspace, cgroup убьёт.

Полезная опция: `-XX:+ExitOnOutOfMemoryError` — при OOM внутри JVM (heap exhausted, не cgroup) выйти сразу с ошибкой вместо попыток продолжить с полумёртвой памятью. Kubernetes перезапустит, а не будет сервить сломанные ответы.

`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/oom.hprof` — писать heap dump при OOM автоматически. Часто ставят на shared volume чтобы после рестарта dump остался и его можно было забрать для анализа.

## Полный шаблон для прод-сервиса

Собранный воедино манифест, отражающий все обсуждённые практики:

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
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels: {app: isnaknpuser}
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: registry.1sc.kz/isnaknpuser:v1.2.3
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: prod
        - name: JAVA_TOOL_OPTIONS
          value: "-Xmx1500m -XX:+ExitOnOutOfMemoryError -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/oom.hprof"
        resources:
          requests: {memory: 2Gi, cpu: 500m}
          limits:   {memory: 2Gi, cpu: 2000m}
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

Здесь всё что мы обсудили: 3 replicas + PDB(minAvailable=2) для zero-downtime при drain, priorityClass high, memory Guaranteed, cpu Burstable, anti-affinity для распределения по разным нодам, preStop sleep + большой terminationGracePeriod, все три probe, метки для Prometheus scrape, NetworkPolicy разрешает трафик только от gateway.

## Как это всё диагностировать в проде

Симптомы часто одинаковые (сервис отвечает 5xx, задержка растёт), а причины разные. Быстрый чек-лист:

**Pod'ы не Ready.** `kubectl get pods -n knp -l app=isnaknpuser` — смотрим READY колонку. `0/1` — контейнер не Ready. `kubectl describe pod X` — раздел Events покажет причину: pull error, probe failure, OOMKilled. `kubectl logs X --previous` — логи предыдущего инстанса перед рестартом (часто там причина).

**Rolling update завис.** `kubectl rollout status deployment/X` — покажет «waiting for rollout to finish: N of M new replicas have been updated». Скорее всего новый pod не становится Ready. `kubectl get pods -l app=X --sort-by=.metadata.creationTimestamp` — новый pod внизу списка, смотреть его describe и logs.

**Один pod ест 100% CPU, остальные простаивают.** Проверить какой pod: `kubectl top pods -n knp --sort-by=cpu`. Стандартная причина — залипший процесс/deadlock/бесконечный цикл. `kubectl exec pod-x -- jstack 1 > threads.txt` — thread dump, искать `RUNNABLE` треды в цикле.

**Все pod'ы Ready, но 5xx летят.** Смотреть с точки зрения балансировщика. `kubectl get endpoints isnaknpuser` — какие IP входят в endpoints. Если pod Terminating но ещё в endpoints — race condition, нужен preStop sleep. Если pod Running но не в endpoints — что-то сломано с Service selector.

**Node drain не проходит.** `kubectl describe node X` — покажет pod'ы, которые нельзя эвакуировать. Обычно причина — `Cannot evict pod as it would violate the pod's disruption budget`. Проверить PDB: `kubectl get pdb -A`. Если PDB требует больше available чем есть реплик — либо увеличить replicas, либо ослабить PDB.

**Pod'ы не могут ходить к базе после включения NetworkPolicy.** `kubectl run tmp --rm -it --image=busybox -- nc -zv postgres 5432`. Если timeout — NetworkPolicy режет. `kubectl get networkpolicy -A` — что применено. Часто причина — default deny, но забыли добавить разрешение конкретному сервису.

## Заключение

Kubernetes для микросервисов — это набор примитивов, которые надо собирать в правильные комбинации. Workload'ы выбираются по природе приложения (Deployment для stateless, StatefulSet для identity, DaemonSet для per-node, Job для одноразовых). Storage требует внимания к reclaim policy — Delete на persistent томе стоит данных. NetworkPolicy обязательна в проде, но требует поддерживающего CNI. PDB защищает от voluntary disruptions, но только если реплик хотя бы две. QoS Guaranteed для критичных сервисов через `requests == limits`. Priority — иерархия для случаев переполнения.

Graceful shutdown — это цепочка, где ошибка на любом этапе даёт хвост ошибок клиентам: preStop sleep для окна iptables, Spring Boot graceful mode для ожидания in-flight запросов, `terminationGracePeriodSeconds` больше суммы. Probes с чётким разделением: liveness не зависит от внешних систем (иначе каскад), readiness зависит и служит кнопкой балансировщика, startup для медленного прогрева.

Rolling update с `maxUnavailable: 0` для zero-downtime; blue/green и canary через специализированные инструменты. OOMKilled лечится анализом heap dump в MAT + правильным соотношением `-Xmx` к `limits.memory`.

Полный шаблон deployment'а собирает всё: три реплики, PDB, priority, anti-affinity, ресурсы, preStop, три probe, метки Prometheus, NetworkPolicy. Такой манифест — базовая единица для любого микросервиса в проде, дальше только доработки под специфику.

Диагностика в проде идёт через несколько инструментов: `kubectl get`/`describe`/`logs`, `kubectl rollout status`, `kubectl top`, `kubectl exec` с jstack/jcmd для JVM. Каждый симптом — своя цепочка команд, набирается практикой и повторяется от инцидента к инциденту.
