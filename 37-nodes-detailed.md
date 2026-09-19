# 37. Nodes: что это в разных системах (детально)

**Node** — универсальный термин в распределённых системах. В разных контекстах означает разное. Разберём каждый случай.

---

## 1. Общее определение

**Node** = «узел» = **один экземпляр** в распределённой системе.

Может быть:
- Физическая машина.
- Виртуальная машина.
- Docker-контейнер.
- K8s Pod.
- Процесс на сервере.

Идея одна: система состоит из **нескольких** node, они общаются между собой и работают как единое целое.

Ключевые свойства распределённых систем:
- **Failure** — один node может упасть, система должна работать.
- **Consensus** — как несколько node договариваются.
- **Split-brain** — сеть разделила node, каждая думает что «главная».
- **Quorum** — большинство node должны согласиться.

---

## 2. Node в Kubernetes

Обсуждалось в `10-kubernetes-detailed.md`, но подробнее.

### 2.1 Что это

**K8s Node** = физическая или виртуальная машина, на которой бегут поды.

```
K8s Cluster
    │
    ├─ Master node 1  ┐
    ├─ Master node 2  ├─ Control Plane (обычно 3 для HA)
    ├─ Master node 3  ┘
    │
    ├─ Worker node 1  ┐
    ├─ Worker node 2  ├─ где реально бегут пользовательские pods
    ├─ Worker node 3  │
    └─ ...             ┘
```

### 2.2 Компоненты worker node

- **kubelet** — агент K8s, запускает контейнеры через container runtime.
- **kube-proxy** — реализует Service через iptables/IPVS.
- **container runtime** — containerd / CRI-O (раньше Docker).
- **cAdvisor** — метрики контейнеров.

### 2.3 Компоненты control-plane node

- **kube-apiserver** — REST API кластера.
- **etcd** — KV-хранилище (Raft).
- **kube-scheduler** — решает на какую ноду ставить под.
- **kube-controller-manager** — контроллеры (Deployment, ReplicaSet, ...).

### 2.4 Как узнать про ноды

```bash
kubectl get nodes                        # список
kubectl describe node <name>             # подробно
kubectl top nodes                        # CPU/memory
kubectl get pods -o wide --all-namespaces   # где какой pod
```

### 2.5 Node lifecycle

- **Ready** — kubelet отвечает, все ОК.
- **NotReady** — kubelet не отвечает, поды эвикнутся через ~5 мин.
- **SchedulingDisabled** — новых подов не пускаем (при maintenance).

### 2.6 Taints & Tolerations

**Taint** на ноде — «не ставьте сюда pods, если у них нет tolerance».
```bash
kubectl taint node worker-1 special=true:NoSchedule
```

Pod с tolerance попадёт, без — нет. Использование: выделенные ноды под GPU / IO-heavy / dedicated tenants.

### 2.7 Node affinity

Правила «где размещать pod»:
- `nodeSelector` — простое совпадение labels.
- `nodeAffinity` — сложные правила (required, preferred, weight).

### 2.8 Реальные IP-адреса нод ИСНА

Из memory: `172.158.0.12` (extprod/intprod host), `172.19.21.66` (etprod-kesh), etc. Реальные VM/физ. серверы.

---

## 3. Node в Consul

Обсуждалось в `11-consul-detailed.md`.

### 3.1 Что это

**Consul Node** = один экземпляр процесса Consul.

Два типа:
- **Server** — участвует в Raft consensus, хранит данные. Обычно 3 или 5 в кластере.
- **Client (agent)** — легковесный агент на каждой ноде приложения, форвардит запросы к серверам.

### 3.2 Раздельно от K8s Node

`Consul node` ≠ `K8s node`. Consul — своя абстракция.

Пример: K8s cluster с 10 worker nodes. На каждом бежит Consul agent (client) как DaemonSet. Плюс отдельные 3 Consul server (в K8s или вне).

### 3.3 Gossip

Consul agents гошипятся между собой (SWIM protocol) — быстро распространяют информацию о состоянии.

- **LAN gossip** — внутри одного DC.
- **WAN gossip** — между DC.

### 3.4 Как посмотреть

```bash
consul members                    # все ноды кластера
consul operator raft list-peers   # какие server в Raft
```

---

## 4. Node в RabbitMQ

### 4.1 Что это

**RabbitMQ Node** = один процесс Erlang broker.

Кластер:
```
Cluster
  ├─ rabbit@node1
  ├─ rabbit@node2
  └─ rabbit@node3
```

### 4.2 Data distribution

- **Classic queues** — на одной ноде, могут mirror'иться (deprecated).
- **Quorum queues** — на 3 нодах (Raft), автоматический failover.
- **Metadata** (exchanges, bindings, users) — синхронно на всех нодах.

### 4.3 Split-brain

Если сеть разделила ноды (network partition) — каждая часть может думать «я главная».

RabbitMQ политики:
- `ignore` — каждая часть работает независимо (плохо, data conflict).
- `pause_minority` — часть без большинства блокируется.
- `autoheal` — при восстановлении сети выбирается «главная» часть, минорная перезапускается.

**Правильно**: `pause_minority` для избежания split-brain data loss.

### 4.4 Как посмотреть

```bash
rabbitmqctl cluster_status
rabbitmqctl list_queues name node
```

---

## 5. Node в PostgreSQL

### 5.1 Primary vs Replica

- **Primary (master)** — принимает write. Один в кластере.
- **Replica (standby)** — только read; следит за WAL primary.

```
Application → write → [Primary]
                 read → [Replica 1]
                       [Replica 2]
```

### 5.2 Streaming replication

Primary стримит WAL в реплики. Реплики применяют.

- **Sync replication** — commit только когда реплика подтвердила (медленнее, надёжнее).
- **Async replication** — быстрее, но при падении primary может потерять последние транзакции.

### 5.3 Failover

Primary упал → одна из реплик promoted до primary.

Инструменты:
- **Patroni** — автоматический failover через etcd/Consul + Raft.
- **repmgr**.
- **pg_auto_failover**.

### 5.4 Read replicas в приложении

```yaml
spring:
  datasource:
    write:
      url: jdbc:postgresql://pg-primary:5432/knp
    read:
      url: jdbc:postgresql://pg-replica:5432/knp
```

Кавет **replication lag** — реплика может отстать на 100 мс — 1 сек. Не для read-after-write критики.

---

## 6. Node в Kafka

Kafka node называется **broker**.

### 6.1 Что это

Kafka broker = один процесс. Кластер обычно из 3-9 broker'ов.

Каждый broker:
- Держит части topics (partitions).
- Обслуживает producers/consumers.
- Реплицирует данные с/на другие broker'ы.

### 6.2 Роли

- **Controller** — один broker выбран через Raft, отвечает за metadata (создание topic, перераспределение partition при падении).
- **Leader** — для каждой partition один broker leader (принимает write).
- **Follower** — replica, следит за leader.

### 6.3 ISR

**In-Sync Replicas** — replicas, догоняющие leader (лаг < threshold).

При падении leader — новый leader выбирается из ISR.

---

## 7. Node в Elasticsearch

Тема отдельного файла `45-elasticsearch.md`. Кратко.

Роли:
- **Master node** — управляет кластером (создание индексов, распределение shards).
- **Data node** — хранит данные, обслуживает queries.
- **Coordinating node** — принимает запросы, координирует.
- **Ingest node** — pre-processing документов.

Node может совмещать несколько ролей.

---

## 8. Node в Cassandra / Redis Cluster

**Cassandra**: peer-to-peer, все ноды равноправны. Данные распределяются через consistent hashing (token ring).

**Redis Cluster**: 16384 hash slots распределены между master node. Каждый master имеет replicas.

Общее: нет «главного» master в классическом смысле. Sharding через hash.

---

## 9. Node в Zookeeper

**Zookeeper ensemble** — 3 или 5 node.

Ensemble — координация распределённых систем (используется в Kafka, Hadoop, HBase).

Consensus через **Zab protocol** (похожий на Raft).

**Kafka с 3.x** переходит на KRaft (внутренний Raft), убирая зависимость от Zookeeper.

---

## 10. Node в Neo4j (graph БД)

Здесь **node** имеет другое значение — **вершина графа** (не «сервер»).

```
(a:Person)-[:KNOWS]->(b:Person)
   node A            node B
```

Не путать с cluster node.

---

## 11. Общие принципы работы с nodes

### 11.1 Failure detection

Как понять что node упала?
- **Heartbeat** — периодические keep-alive.
- **Gossip** — ноды сплетничают о состоянии соседей.
- **Health check** — активная проверка.

Порог: сколько failed heartbeats подряд = node dead.

### 11.2 Consensus (Raft, Paxos)

Как несколько nodes договариваются о состоянии?

**Raft** (более простой и популярный):
1. Одна нода — **leader**, остальные — **followers**.
2. Все writes через leader.
3. Leader реплицирует на followers.
4. При падении leader — выборы нового.

Используется в: etcd, Consul, quorum queues, KRaft, Nomad, TiKV.

**Paxos** — оригинальный, сложнее. Использовался в Zookeeper (Zab — вариация).

### 11.3 Quorum

**Кворум** = минимальное большинство для принятия решения.

- 3 nodes → кворум 2 (можно потерять 1).
- 5 nodes → кворум 3 (можно потерять 2).

**Почему нечётное число**: 4 nodes → кворум 3. Потеря 2 → нельзя работать. То же что и с 3 nodes, но дороже.

### 11.4 Split-brain

Сеть разделила кластер на две части, каждая думает что «главная».

Классический пример:
- 5 nodes, разделены 3 + 2.
- Часть с 3 — кворум есть, работает.
- Часть с 2 — кворума нет, **должна остановиться** (иначе data loss при восстановлении).

Правильно построенные системы (Raft) — не дают split-brain через кворум.

Плохо построенные (некоторые старые MySQL replication setups) — split-brain приводит к data loss.

### 11.5 Cascading failure

Одна нода упала → нагрузка перераспределилась на других → они не справились → тоже упали.

Защита:
- **Circuit breaker**.
- **Rate limiting**.
- **Backpressure**.
- **Auto-scaling**.
- **Bulkheads**.

---

## 12. Reference: node counts

| System | Typical count | Notes |
|---|---|---|
| K8s control plane | 3 или 5 | HA + Raft |
| K8s worker | 3-1000+ | сколько нужно |
| Consul server | 3 или 5 | Raft |
| Consul agent | по одному на каждый узел | |
| RabbitMQ cluster | 3 | quorum queues |
| Kafka broker | 3-9 | replication factor 3 |
| PostgreSQL | 1 primary + 1-N replicas | |
| Elasticsearch | 3-N | 3 master-eligible |
| Zookeeper ensemble | 3 или 5 | |
| Redis Cluster | 3 master + 3 replicas минимум | |

---

## 13. Как ИСНА использует ноды (по memory)

- **K8s nodes**: несколько VM (172.158.0.12 extprod/intprod host, etc). Разные namespaces (knp, fno, tax-report) — все на общем worker pool.
- **Consul servers**: 3 (обычно) для HA.
- **PostgreSQL**: обычно primary + реплика. Через **pgbouncer** (`db-knp` = pgbouncer в pod-сети).
- **RabbitMQ**: кластер.
- **Hazelcast**: distributed cache (memory `taxrep21-hazelcast-kesh66-unreachable` — .66:5702 unreachable = один член кластера недоступен).
- **Elasticsearch**: ELK-стек для логов (memory `knp-prod-historical-logs-elk`).
- **Kubelet + kube-proxy** на каждой worker node.

---

## 14. Собесные вопросы

1. **Что такое node в распределённой системе?** — Один экземпляр (физ. VM, VM, контейнер, процесс).
2. **Что такое K8s node?** — Физ/виртуальная машина где бегут pods; control-plane или worker.
3. **Компоненты worker node?** — kubelet, kube-proxy, container runtime.
4. **Что такое split-brain?** — Сеть разделила cluster, каждая часть думает что «главная»; риск data loss.
5. **Что такое quorum?** — Минимальное большинство для принятия решения; нечётное число nodes.
6. **Почему всегда нечётное число nodes?** — Чётное — тот же fault tolerance что предыдущее нечётное, но дороже.
7. **Что такое Raft?** — Алгоритм консенсуса; leader + followers; используется в etcd, Consul, KRaft.
8. **Разница master и worker node в K8s?** — Master = control-plane (apiserver, etcd, scheduler); worker = где бегут pods.
9. **Что делает kubelet?** — Агент K8s на ноде; запускает/останавливает контейнеры через CRI.
10. **Что такое ISR в Kafka?** — In-Sync Replicas — replicas, догоняющие leader.
11. **PostgreSQL primary vs replica?** — Primary для writes; replicas для reads; sync/async replication.
12. **Как узнать про K8s nodes?** — `kubectl get nodes`, `kubectl describe node`, `kubectl top nodes`.
13. **Что такое taint и toleration?** — Taint на ноде запрещает pods без toleration.
14. **Cassandra vs Cassandra: где master?** — Peer-to-peer, все равноправны, sharding через consistent hashing.
15. **Что такое cascading failure?** — Падение одного node → перегрузка других → они падают тоже.

---

## Итог

- **Node** = один экземпляр в distributed system.
- **K8s**: control-plane + worker; kubelet на каждой.
- **Consul**: server (Raft) + agent (на каждой ноде).
- **RabbitMQ**: broker; quorum queues для HA.
- **PostgreSQL**: primary + replicas; sync/async replication.
- **Kafka**: broker; leader/follower per partition; ISR.
- **Elasticsearch**: master/data/coordinating/ingest роли.
- **Raft** — консенсус для метадаты (etcd, Consul, KRaft, quorum queues).
- **Quorum** — большинство; нечётное число nodes оптимально.
- **Split-brain** — главный враг distributed systems; защита через quorum.

Следующий — `38-logging.md`.
