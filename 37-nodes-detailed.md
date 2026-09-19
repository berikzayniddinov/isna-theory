# 37. Nodes: что это в разных системах

## Общее определение

Node это универсальный термин в распределённых системах означающий один экземпляр в distributed system. Может представлять разные физические сущности в зависимости от контекста. Физическую машину. Виртуальную машину. Docker контейнер. Kubernetes Pod. Процесс на сервере.

Идея одна — система состоит из нескольких node которые общаются между собой и работают как единое целое. Каждый node имеет свои ресурсы (CPU, memory, storage), может независимо принимать запросы и выполнять работу.

Ключевые свойства распределённых систем определяют почему nodes существуют и как они взаимодействуют. Failure tolerance — один node может упасть, система должна продолжать работать. Consensus — множество nodes должны договариваться о общем состоянии. Split-brain — сеть может разделить nodes на несколько частей, каждая может думать что «главная». Quorum — большинство nodes должно согласиться для принятия критических решений.

Понимание концепций nodes в разных technologies помогает правильно проектировать и troubleshoot distributed systems. Ниже разбор специфики для каждой основной технологии используемой в КНП инфраструктуре.

## Node в Kubernetes

Kubernetes node это физическая или виртуальная машина где бегут pods. Kubernetes cluster состоит из control-plane nodes управляющих кластером и worker nodes где размещаются user workloads.

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

Компоненты worker node. kubelet это агент K8s запускающий контейнеры через container runtime. kube-proxy реализует Service через iptables или IPVS. Container runtime это containerd или CRI-O (раньше Docker но устарел как runtime). cAdvisor собирает метрики контейнеров.

Компоненты control-plane node. kube-apiserver предоставляет REST API кластера. etcd это KV-хранилище используемое как persistent storage через Raft consensus. kube-scheduler решает на какой worker node размещать новые pods. kube-controller-manager запускает контроллеры для Deployment, ReplicaSet и других ресурсов.

Управление nodes через kubectl. `kubectl get nodes` показывает список всех nodes. `kubectl describe node <name>` даёт детальную информацию о конкретном node. `kubectl top nodes` показывает CPU и memory usage. `kubectl get pods -o wide --all-namespaces` показывает где какой pod размещён.

Node lifecycle. Ready означает kubelet отвечает, всё работает нормально. NotReady kubelet не отвечает — pods будут evicted через около 5 минут. SchedulingDisabled это состояние при maintenance — новые pods не размещаются но существующие продолжают работать.

Taints и Tolerations управляют размещением pods. Taint на node говорит «не размещайте здесь pods без соответствующего toleration». Pod с tolerance подходящим к taint может быть размещён, без tolerance — нет. Использование для выделенных nodes под GPU workloads, IO-heavy applications, dedicated tenants requiring isolation.

Node affinity предоставляет более гибкие правила размещения. nodeSelector это простое совпадение labels — pod идёт только на nodes с matching labels. nodeAffinity позволяет сложные правила — required (обязательные) plus preferred (желательные) с весами определяющими priority.

В КНП K8s cluster состоит из нескольких VM на реальных IP addresses. Из memory examples — 172.158.0.12 как extprod/intprod host, 172.19.21.66 как etprod-kesh, и другие. Разные namespaces (knp, fno, tax-report) все размещаются на общем worker pool.

## Node в Consul

Consul node это один экземпляр Consul процесса. Consul cluster имеет два типа nodes для разных ролей.

Server nodes участвуют в Raft consensus, хранят все данные (services registry, KV store, ACLs). Обычно 3 или 5 servers в cluster для HA plus consensus quorum. Servers это критическая инфраструктура — их падение break service discovery.

Client agents это легковесные agents на каждой node где работают applications. Они не хранят данные — только forward requests to servers. Обычно deployed как DaemonSet в Kubernetes — по одному на каждый worker node. Applications обращаются к local agent через localhost который forwards to servers.

Важно понимать что Consul node не то же самое что K8s node. Consul это своя абстракция с собственными nodes. Пример реальной architecture — K8s cluster с 10 worker nodes. На каждом бежит Consul agent (client) как DaemonSet. Плюс 3 отдельных Consul server (могут быть в K8s или на отдельных VMs).

Gossip protocol используется для communication между agents. SWIM protocol быстро распространяет информацию о node states через cluster. LAN gossip работает внутри одного datacenter. WAN gossip для communication между datacenters.

Команды мониторинга. `consul members` показывает все nodes в cluster. `consul operator raft list-peers` показывает какие servers участвуют в Raft consensus.

## Node в RabbitMQ

RabbitMQ node это один Erlang broker process. Cluster формируется из нескольких RabbitMQ instances:
```
Cluster
  ├─ rabbit@node1
  ├─ rabbit@node2
  └─ rabbit@node3
```

Data distribution зависит от типа queue. Classic queues живут на одной node, могут mirror-иться на другие (mirroring deprecated). Quorum queues реплицируются на 3 nodes через Raft consensus — automatic failover. Metadata (exchanges, bindings, users) синхронизируется на всех nodes.

Split-brain scenario критически важен для RabbitMQ. Если сеть разделила nodes (network partition) — каждая часть может думать «я главная» и продолжать работать independently.

Политики handling network partitions. ignore — каждая часть работает independently, приводит к data conflicts после reconciliation. pause_minority — часть без quorum majority блокируется, только majority работает. autoheal — при восстановлении сети выбирается «главная» часть, minority side перезапускается теряя changes.

Правильная стратегия — pause_minority для избежания split-brain data loss. Majority продолжает работать, minority останавливается до восстановления сети.

Мониторинг через rabbitmqctl. `rabbitmqctl cluster_status` показывает состояние cluster. `rabbitmqctl list_queues name node` показывает какая queue на каком node.

## Node в PostgreSQL

PostgreSQL использует primary-replica architecture. Primary (иногда называется master) принимает все writes. Один primary в cluster. Replicas (иногда называется standby) только для reads — они следят за WAL primary и применяют same changes к своим data files.

```
Application → write → [Primary]
                 read → [Replica 1]
                       [Replica 2]
```

Streaming replication это основной механизм. Primary отправляет WAL stream на replicas. Replicas применяют changes сохраняя consistent copy данных.

Sync vs async replication. Sync replication — commit возвращается только когда replica подтвердила receipt WAL. Медленнее (extra round-trip) но zero data loss при primary failure. Async replication — commit возвращается сразу без ожидания. Быстрее но может потерять несколько recent transactions при primary crash.

Failover это process когда primary упал и replica promoted до primary. Manual failover — administrator вручную выполняет promotion. Automatic failover через tools вроде Patroni (использует etcd или Consul для coordination), repmgr, pg_auto_failover.

Read replicas в приложении настраиваются через отдельные datasources:
```yaml
spring:
  datasource:
    write:
      url: jdbc:postgresql://pg-primary:5432/knp
    read:
      url: jdbc:postgresql://pg-replica:5432/knp
```

Caveat replication lag. Replica может отставать на 100 миллисекунд до 1 секунды. Read сразу после write через replica может показать old data. Не подходит для read-after-write critical сценариев. Для критических читаемых сразу после write — читать с primary или использовать sync replication.

## Node в Kafka

Kafka node называется broker. Один broker это один Kafka process. Cluster обычно из 3-9 brokers для HA plus scaling.

Каждый broker выполняет несколько функций. Хранит parts of topics (partitions). Обслуживает producers и consumers для своих partitions. Реплицирует данные с и на другие brokers для fault tolerance.

Роли внутри cluster. Controller это один broker выбранный через Raft для coordination — управляет partition leader election, обрабатывает administrative операции (создание topic, реассignment partitions). Leader — для каждой partition один broker leader принимающий writes. Follower — replicas следящие за leader и синхронизирующие данные.

ISR (In-Sync Replicas) это replicas которые догоняют leader (лаг меньше threshold). При падении leader — новый leader выбирается только из ISR. Это гарантирует что новый leader имеет самые recent committed данные.

## Node в Elasticsearch

Elasticsearch поддерживает несколько ролей per node. Один physical node может выполнять несколько ролей одновременно.

Master node управляет cluster — создание индексов, распределение shards между data nodes, tracking membership. Data node хранит данные и обслуживает queries — search и indexing operations. Coordinating node принимает client requests и координирует их execution across cluster без хранения данных. Ingest node выполняет pre-processing документов перед их indexing.

В small clusters один node может выполнять все роли. В larger clusters роли разделяются для performance и reliability. Например 3 dedicated master-eligible nodes plus 5 data nodes plus 2 coordinating nodes.

## Node в Cassandra и Redis Cluster

Cassandra это peer-to-peer architecture. Все nodes равноправны — нет «главного» master. Данные распределяются через consistent hashing (token ring). Каждая node ответственна за определённый range hash values. Replication factor определяет сколько replicas каждого данного (обычно 3).

Redis Cluster использует 16384 hash slots распределённые между master nodes. Каждый master имеет несколько replicas для HA. При failure master один из replicas promoted до master.

Общее для обоих — нет single point of coordination. Sharding через hash даёт horizontal scalability. Client library знает cluster topology и routes requests к нужным nodes.

## Node в Zookeeper

Zookeeper ensemble это 3 или 5 nodes работающих вместе через Zab protocol (похожий на Raft). Ensemble предоставляет distributed coordination services для других systems.

Использование. Kafka classic mode использовал Zookeeper для metadata и coordination. Hadoop использует для coordination. HBase использует. Многие distributed systems полагаются на Zookeeper для reliable coordination.

Kafka 3.x переходит на KRaft (Kafka's own Raft implementation). Убирает dependency на Zookeeper. В 4.x Zookeeper support полностью удаляется. Тренд — избавление от external coordination systems в пользу internal Raft.

## Node в Neo4j graph database

Здесь термин node имеет другое значение — вершина графа не «сервер». Не путать с cluster nodes.

```
(a:Person)-[:KNOWS]->(b:Person)
   node A            node B
```

Confusingly два разных концепта используют одно слово. При обсуждении Neo4j важно уточнять — data node в graph modeling или cluster node в distributed setup.

## Общие принципы работы с nodes

Failure detection это фундаментальная задача. Как система понимает что node упал?

Heartbeat это периодические keep-alive messages от каждой node. Другие nodes ожидают heartbeat в определённом interval. Пропуск N heartbeats подряд — node считается dead.

Gossip protocol используется в некоторых systems (Consul, Cassandra). Nodes «сплетничают» друг с другом обмениваясь информацией о состоянии соседей. Информация быстро распространяется через cluster.

Active health checks это explicit проверки через probes — HTTP endpoint, TCP connect, custom protocol. Более definitive чем heartbeat но требует больше resources.

Threshold detection обычно требует несколько consecutive failures до объявления node dead. Предотвращает flapping из-за transient issues (temporary network delays).

Consensus algorithms решают проблему как множество nodes договариваются о common state. Раньше обсуждались briefly, теперь подробнее.

Raft более простой и популярный algorithm. Одна node становится leader через election, остальные — followers. Все writes идут через leader. Leader реплицирует изменения на followers. При падении leader — followers инициируют новую election для выбора нового leader. Используется в etcd, Consul, Kafka KRaft, quorum queues в RabbitMQ, Nomad, TiKV.

Paxos это оригинальный consensus algorithm, сложнее Raft. Использовался в Zookeeper (Zab это вариация Paxos). Много academic contributions но practical implementations сложные.

Quorum это минимальное большинство nodes для принятия решения. 3 nodes требуют quorum 2 — можно потерять 1 node. 5 nodes требуют quorum 3 — можно потерять 2 nodes.

Почему нечётное число nodes. 4 nodes требуют quorum 3 — при потере 2 nodes невозможна работа. То же fault tolerance как 3 nodes (потеря 1) но дороже. Всегда предпочтительнее нечётное число.

Split-brain это критическая проблема где сеть разделяет cluster на несколько частей и каждая думает что «главная». Классический пример — 5 nodes разделяются 3 plus 2 из-за network partition. Часть с 3 имеет quorum и продолжает работать. Часть с 2 нет quorum — должна остановиться иначе data conflict при восстановлении.

Правильно построенные системы через Raft не дают split-brain благодаря quorum requirement. Плохо построенные (некоторые старые MySQL replication setups без proper coordination) split-brain приводит к data loss или corruption.

Cascading failure это когда падение одного node создаёт chain reaction. Одна node упала — nagruzka перераспределилась на других — они не справились — тоже упали. Может привести к полному outage cluster.

Защита от cascading failures. Circuit breaker fast fails при обнаружении проблем downstream. Rate limiting ограничивает traffic to prevent overload. Backpressure сигнализирует upstream что downstream перегружен. Auto-scaling добавляет capacity при росте load. Bulkheads изолируют resource pools для разных dependencies.

## Reference таблица типичных counts

Практические типичные конфигурации:

| System | Typical count | Notes |
|---|---|---|
| K8s control plane | 3 или 5 | HA plus Raft (etcd) |
| K8s worker | 3-1000+ | сколько нужно для workload |
| Consul server | 3 или 5 | Raft consensus |
| Consul agent | 1 на каждый node | DaemonSet в K8s |
| RabbitMQ cluster | 3 | quorum queues требуют 3 |
| Kafka broker | 3-9 | replication factor 3 |
| PostgreSQL | 1 primary + 1-N replicas | streaming replication |
| Elasticsearch | 3-N | 3 master-eligible минимум |
| Zookeeper ensemble | 3 или 5 | Zab consensus |
| Redis Cluster | 3 master + 3 replicas минимум | |

Общий pattern — 3 nodes минимум для HA plus consensus based systems. 5 nodes для higher fault tolerance. Больше при need для scaling или performance requirements.

## Как КНП использует nodes

Практика из КНП environment plus memory кейсов.

K8s nodes — несколько VM (172.158.0.12 extprod/intprod host, другие). Разные namespaces (knp, fno, tax-report) размещаются на общем worker pool. Control plane обычно 3 masters для HA.

Consul servers — 3 обычно для HA plus Raft consensus. Consul agents на каждом K8s worker node как DaemonSet — locally accessible service registry для applications.

PostgreSQL обычно primary plus replica configuration. Через PgBouncer как connection pooler. db-knp это PgBouncer в pod-сети — accessible только внутри cluster.

RabbitMQ cluster для messaging. Quorum queues для важных данных обеспечивающие HA.

Hazelcast distributed cache используется. Memory кейс taxrep21-hazelcast-kesh66-unreachable упоминает что .66:5702 unreachable означает один member cluster недоступен — необходимо monitoring для detection.

Elasticsearch ELK stack для logs. Memory кейс knp-prod-historical-logs-elk описывает что kubectl logs показывает только current, исторические доступны только через Elasticsearch queries.

Kubelet и kube-proxy на каждом worker node — стандартная Kubernetes infrastructure.

## Итоги

Node это универсальный термин в distributed systems означающий один экземпляр — VM, container, process. Разные technologies имеют свои concepts nodes с специфическими характеристиками.

Kubernetes nodes — control-plane plus worker. Компоненты kubelet, kube-proxy, container runtime на каждом. Node lifecycle через Ready, NotReady, SchedulingDisabled. Taints и tolerations для placement control.

Consul nodes — servers (Raft consensus) plus agents (на каждой application node). Gossip protocol для communication.

RabbitMQ nodes работают в cluster. Quorum queues для HA. pause_minority policy для split-brain protection.

PostgreSQL primary plus replicas. Streaming replication sync или async. Read replicas для scaling reads с caveat replication lag.

Kafka brokers cluster. Controller для coordination. Leader/follower per partition. ISR для reliable failover.

Elasticsearch multiple roles per node. Master, data, coordinating, ingest.

Cassandra peer-to-peer. Redis Cluster hash slots. Zookeeper ensemble. Neo4j graph node (не путать).

Общие принципы. Failure detection через heartbeat, gossip, active health checks. Raft consensus most popular. Quorum обеспечивает split-brain protection. Нечётное число nodes оптимально. Cascading failure защищается через circuit breakers, rate limiting, bulkheads.

Дальше — logging как критическая observability составляющая. SLF4J, Logback, ELK stack и best practices.
