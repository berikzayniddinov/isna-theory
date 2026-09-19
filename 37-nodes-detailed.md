# 37. Nodes: универсальный термин в distributed systems

## Зачем понимать разные типы nodes

Слово «node» одно из самых overloaded терминов в мире распределённых систем. В Kubernetes это виртуальная машина где размещаются pods. В Consul это server participating в Raft consensus или client agent на application node. В PostgreSQL primary или replica database instance. В Kafka это broker. В Elasticsearch может играть роль master, data, coordinating или ingest. В Neo4j это вообще vertex графа не сервер вовсе. Один термин — множество значений.

Разработчик который путает эти concepts часто попадает в неприятные ситуации. Consul node down не означает Kubernetes node down. Kafka broker failure имеет specific semantics отличные от PostgreSQL replica failure. Split-brain в Elasticsearch отличается от split-brain в RabbitMQ по detection и recovery mechanisms. Правильная реакция на инцидент требует правильного understanding какой именно node type failed.

Разница между инженером «знающим про nodes» и «понимающим nodes» проявляется когда нужно спроектировать resilient system или troubleshoot production incident. Первый знает что «нужны 3 nodes для quorum». Второй знает что 3 nodes дают fault tolerance 1 (потеря одного oк), а 5 nodes — 2 (можно потерять 2). Знает что этот compute нечетное число оптимально потому что 4 nodes требуют quorum 3 — тот же fault tolerance как 3 nodes но дороже. Знает что Raft based systems (etcd, Consul, quorum queues) automatically prevent split-brain через quorum requirements а некоторые старые systems (MySQL master-master replication без proper coordination) split-brain приводит к data loss.

В этом файле разберём node concept как universal abstraction plus specific implementations в различных technologies used in КНП infrastructure. Общие distributed systems concepts — failure detection, consensus algorithms (Raft plus Paxos briefly), quorum, split-brain protection, cascading failure. Kubernetes nodes deep — control plane vs worker, конкретные компоненты, node lifecycle plus taints и tolerations. Consul nodes — server plus agent architecture, gossip protocol basics. RabbitMQ nodes и network partition handling. PostgreSQL primary-replica с streaming replication mechanics. Kafka brokers и controller. Elasticsearch node roles. Cassandra plus Redis Cluster peer-to-peer. Zookeeper ensemble. Practical patterns node counts.

## Общее определение и key concepts

Node это универсальный термин в distributed systems означающий один экземпляр в системе. Физическая машина. Виртуальная машина. Docker контейнер. Kubernetes Pod. Процесс на сервере. Конкретное воплощение зависит от technology и абстракции.

Идея одна — система состоит из нескольких node которые общаются между собой и работают как единое целое. Каждый node имеет свои ресурсы (CPU, memory, storage), может независимо принимать запросы и выполнять работу. Отказ одного node системе желательно переживать через redundancy и failover.

Ключевые свойства distributed systems которые определяют почему nodes существуют и как они взаимодействуют.

Failure tolerance. Один node может упасть — hardware failure, network issue, software crash, deployment. Система должна продолжать работать. Достигается через redundancy — если один node fails, другие продолжают service. Level tolerance измеряется в количестве simultaneously tolerable failures.

Consensus. Множество nodes должны договариваться о общем состоянии — какая configuration active, кто сейчас leader, какой order writes был applied. Consensus algorithms решают эту problem — Raft, Paxos, Zab. Каждый имеет свои properties и trade-offs.

Split-brain. Сеть может разделить nodes на несколько частей — network partition. Каждая часть может думать что «главная». Без proper handling это приводит к data conflicts когда partitions merge. Правильно спроектированные systems предотвращают через quorum-based decisions — часть без majority не может принимать critical decisions.

Quorum. Минимальное большинство nodes для принятия решения. Ключевая concept для preventing split-brain. Обычно N/2 + 1 где N total nodes. 3 nodes → quorum 2. 5 nodes → quorum 3. Всегда нечетное число оптимально — 4 nodes требуют quorum 3, тот же fault tolerance что 3 nodes но дороже.

Cascading failure. Отказ одного node может создать chain reaction. Один падает → нагрузка перераспределилась на других → они перегружены → падают → всё больше падает. Защита через circuit breakers (fast fail предотвращает waste ресурсов на doomed calls), rate limiting (throttle traffic to sustainable levels), backpressure (signal upstream to slow down), auto-scaling (add capacity под нагрузкой), bulkheads (isolate resource pools).

Понимание этих concepts позволяет reasoning про distributed systems в general terms независимо от specific technology. Ниже разбор specific implementations.

## Kubernetes nodes

Kubernetes node это физическая или виртуальная машина где бегут pods. Kubernetes cluster состоит из nodes играющих одну из двух ролей — control plane или worker.

Control plane nodes управляют cluster. Обычно 3 для HA — quorum 2 обеспечивает tolerance loss одного. Каждый control plane node запускает несколько components. kube-apiserver — REST API cluster плюс central coordination point. etcd — persistent KV storage через Raft consensus, hosts cluster state. kube-scheduler — assigns new pods к available worker nodes. kube-controller-manager — runs controllers implementing desired state (Deployment, ReplicaSet, StatefulSet controllers).

Worker nodes host actual user workloads — pods running application containers. На каждом worker node running components. kubelet — agent responsible for launching containers через container runtime, monitoring their health, reporting status к control plane. kube-proxy — implements Kubernetes Service abstraction через iptables или IPVS rules. Container runtime — actually runs containers, обычно containerd или CRI-O (Docker as runtime deprecated).

Node lifecycle managed через specific states. Ready — kubelet responsive, node healthy, pods могут scheduled. NotReady — kubelet не отвечает, pods after grace period будут evicted (rescheduled на других nodes). SchedulingDisabled — cordoned state during maintenance, existing pods run but new pods не scheduled.

Taints and tolerations управляют pod placement. Taint on node repels pods without matching toleration. `kubectl taint node worker-1 gpu=true:NoSchedule` — только pods с toleration «gpu=true» могут be scheduled here. Use cases — dedicated nodes для GPU workloads, IO-intensive applications, isolated tenants requiring strict separation.

Node affinity как soft-plus preferences для placement. nodeSelector это simple label match. nodeAffinity supports required (must match) plus preferred (softer preferences with weights). Позволяет expressing complex placement rules.

В КНП K8s infrastructure состоит из multiple VMs. Из memory examples — 172.158.0.12 как extprod/intprod host, 172.19.21.66 как etprod-kesh. Different namespaces (knp, fno, tax-report) все размещаются на shared worker pool. Control plane обычно 3 masters для HA.

## Consul nodes: server plus agent architecture

Consul node это один Consul process instance. Consul cluster имеет two node types с очень разными roles.

Server nodes участвуют в Raft consensus, хранят все data (services registry, KV store, ACLs). Обычно 3 или 5 servers в cluster для HA plus consensus quorum. Server failure когда quorum сохраняется — cluster продолжает функционировать. Failure below quorum threshold — cluster becomes unavailable для writes. Critical infrastructure — их падение breaks service discovery для всех services.

Client agents это lightweight processes на каждой node где running applications. Не хранят data — только forward requests к servers. Deployed как DaemonSet в Kubernetes — по одному на каждый worker node. Applications обращаются к local agent через localhost который forwards to servers. Extra hop но обеспечивает locality (агент рядом с application) plus DoS protection (server contact goes через client agents not directly from every application instance).

Важно понимать distinction. Consul node не то же что Kubernetes node. Consul is its own abstraction with own concept of nodes. Example real architecture — K8s cluster с 10 worker nodes. На каждом бежит Consul client agent как DaemonSet — 10 client agents. Plus 3 separate Consul server (могут быть в K8s или на separate VMs) — 3 servers. Total 13 Consul nodes, 10 K8s nodes.

Gossip protocol используется для communication between agents. SWIM protocol быстро распространяет информацию о node states через cluster. LAN gossip работает inside single datacenter. WAN gossip для cross-datacenter communication. Message-efficient — не broadcast, а randomized peer-to-peer exchanges.

Monitoring commands. `consul members` shows все nodes в cluster с их health status. `consul operator raft list-peers` shows какие server nodes участвуют в Raft consensus. Regular monitoring этих commands crucial для operational awareness.

## RabbitMQ nodes

RabbitMQ node это один Erlang broker process. Cluster формируется из нескольких RabbitMQ instances joined together:
```
Cluster
  ├─ rabbit@node1
  ├─ rabbit@node2
  └─ rabbit@node3
```

Data distribution depends на queue type. Classic queues hosted on one node, могут mirror to other nodes for HA (mirroring deprecated in newer versions). Quorum queues replicate на 3 nodes через Raft consensus — automatic failover. Metadata (exchanges, bindings, users) synchronized на всех nodes через own coordination mechanism.

Split-brain scenario критически important для RabbitMQ. Если network partition разделяет nodes — каждая часть может думать «я главная» и продолжать accept messages independently. When network восстанавливается — conflicts как reconcile.

Три политики handling network partitions. ignore — каждая partition работает independently. Приводит к data conflicts при reconciliation. Not recommended. pause_minority — partition без majority (не quorum) pauses operations. Только majority side accepts writes. При recovery minority side resumes syncing from majority. autoheal — при network recovery выбирается «главная» partition (usually с most clients), minority side restarts losing changes.

Правильная стратегия — pause_minority. Обеспечивает split-brain protection. Trade-off — minority side unavailable during partition (users на that side могут be affected). Но избегает data conflicts гораздо важнее.

Monitoring через rabbitmqctl commands. `rabbitmqctl cluster_status` shows cluster state включая network partitions if any. `rabbitmqctl list_queues name node` shows какая queue на каком node.

## PostgreSQL primary-replica architecture

PostgreSQL uses primary-replica architecture (sometimes called master-slave но modern terminology avoids these terms). Primary accepts все writes. Один primary в cluster. Replicas или standby servers accept только reads — они track primary's WAL и apply same changes к own data files.

```
Application → write → [Primary]
                 read → [Replica 1]
                       [Replica 2]
```

Streaming replication это основной mechanism. Primary sends WAL stream to replicas real-time. Replicas apply changes maintaining consistent copy. WAL это Write-Ahead Log — sequential record всех database changes.

Sync vs async replication trade-offs. Sync replication — commit возвращается только когда replica подтвердила receipt WAL. Slower (extra round-trip network) но zero data loss при primary failure — все committed changes точно available на replica. Async replication — commit возвращается сразу без ожидания replica. Faster но может потерять recent transactions при primary crash between commit и replica sync.

Failover это process когда primary упал и replica promoted до primary. Manual failover — administrator manually executes promotion command. Automatic failover через tools like Patroni (uses etcd or Consul for coordination), repmgr, pg_auto_failover. Automatic recommended для production — human response too slow для minute-level uptime SLAs.

Read replicas в приложении настраиваются через separate DataSources:
```yaml
spring:
  datasource:
    write:
      url: jdbc:postgresql://pg-primary:5432/knp
    read:
      url: jdbc:postgresql://pg-replica:5432/knp
```

Routing логика в приложении directs writes к primary DataSource, reads к replica. Может быть implicit через @Transactional(readOnly = true) hints но requires custom AbstractRoutingDataSource implementation.

Caveat replication lag. Replica может отставать от primary — от microseconds до seconds depending on load. Read сразу после write через replica может return old data. Не подходит для read-after-write critical scenarios. Для critical читаемых immediately после write — читать с primary или use sync replication.

## Kafka nodes

Kafka node называется broker. Один broker это один Kafka process. Cluster обычно из 3-9 brokers для HA plus scaling.

Каждый broker выполняет несколько functions. Хранит parts of topics (partitions). Обслуживает producers и consumers для своих partitions. Реплицирует данные с и на другие brokers for fault tolerance.

Роли inside cluster. Controller — один broker выбранный через Raft (в KRaft mode) для coordination. Manages partition leader election, обрабатывает administrative операции (create/delete topics, reassignment partitions при failures). Failure controller triggers election of new controller. Leader — для каждой partition один broker leader принимающий writes и обслуживающий reads. Follower — replicas following leader, syncing data.

ISR (In-Sync Replicas) это replicas догоняющие leader (лаг меньше threshold configured through replica.lag.time.max.ms). При падении leader — новый leader выбирается только из ISR. Guarantees что new leader имеет самые recent committed данные. Если ISR empty (all replicas lag too much) — configurable behavior. unclean.leader.election.allowed=false (recommended) — better unavailability than data loss. true — worst case data loss preferred over unavailability.

Detailed Kafka discussion в следующих файлах (39, 40).

## Elasticsearch node roles

Elasticsearch supports несколько node roles configured per instance. Один physical node может выполнять несколько roles simultaneously или dedicated к specific role.

Master node manages cluster — создание/deletion indices, распределение shards между data nodes, tracking node membership. Master election через Zen Discovery (classical) или newer voting configuration. Minimum 3 master-eligible nodes рекомендовано для preventing split-brain.

Data node хранит indexed данные и выполняет queries. Actual work of searching, indexing, aggregating happens here. Scalable horizontally — more data nodes for more capacity. Sharding across data nodes distributes load.

Coordinating node accepts client requests и координирует execution across cluster. Client-facing node without data storage. Splits requests to relevant shards, aggregates results, returns to client. Can be dedicated coordinating nodes (recommended для large clusters) или combined с other roles.

Ingest node performs pre-processing documents before indexing. Ingest pipelines transformation data через chain of processors — parse, enrich, drop fields, extract values. Optional feature.

В small clusters один node может выполнять все roles combined. В larger clusters roles separated для performance и reliability. Например 3 dedicated master-eligible nodes plus 5 data nodes plus 2 coordinating nodes.

## Cassandra и Redis Cluster peer-to-peer

Cassandra это peer-to-peer architecture. Все nodes равноправны — нет «главного» master node. Данные распределяются через consistent hashing (token ring). Каждая node ответственна за определённый range hash values. Replication factor определяет сколько replicas каждого данного (обычно 3 for durability).

Read/write consistency через tunable consistency levels. Client specifies quorum requirements per operation. ONE — respond after single replica confirms. QUORUM — majority replicas. ALL — все replicas. Trade-off между performance и consistency guarantees per operation.

Redis Cluster uses 16384 hash slots distributed between master nodes. Каждый master имеет несколько replicas для HA. При failure master один of the replicas promoted до master automatically. Redis Cluster requires minimum 3 masters + 3 replicas for meaningful HA.

Sharding через hash gives horizontal scalability. Client library знает cluster topology и routes requests к соответствующим nodes. Rebalancing possible через movement slots between nodes.

## Zookeeper ensemble

Zookeeper ensemble это 3 или 5 nodes работающих вместе через Zab protocol (similar to Raft). Ensemble предоставляет distributed coordination services для других systems.

Usage historical. Kafka classical mode использовал Zookeeper для metadata и coordination. Hadoop и HBase используют. Многие distributed systems полагались на Zookeeper для reliable coordination.

Kafka 3.x переходит на KRaft (Kafka's own Raft implementation). Убирает dependency на Zookeeper. В Kafka 4.x Zookeeper support полностью удаляется. Тренд в industry — избавление от external coordination systems в пользу internal Raft consensus.

## Neo4j graph database — node ambiguity

Здесь термин node имеет другое значение — вершина графа не «сервер». Не путать с cluster nodes.

```
(a:Person)-[:KNOWS]->(b:Person)
   node A            node B
```

Two different concepts share one term. При обсуждении Neo4j important clarify — data node в graph modeling или cluster node в distributed setup.

## Общие принципы работы с nodes

Failure detection это фундаментальная задача. Как система понимает что node упал?

Heartbeat это периодические keep-alive messages от каждой node. Другие nodes ожидают heartbeat в определённом interval. Пропуск N heartbeats consecutively — node considered dead.

Gossip protocol используется в some systems (Consul, Cassandra). Nodes «сплетничают» друг с другом exchanging state information about neighbors. Information rapidly распространяется через cluster.

Active health checks это explicit проверки через probes — HTTP endpoint, TCP connect, custom protocol. Более definitive чем heartbeat но requires больше resources per check.

Threshold detection обычно requires несколько consecutive failures до declaring node dead. Prevents flapping из-за transient issues (temporary network delays, GC pauses).

Consensus algorithms решают problem как multiple nodes договариваются о common state.

Raft — более простой и популярный algorithm. Одна node становится leader через election, остальные — followers. Все writes идут через leader. Leader реплицирует изменения на followers. При падении leader — followers инициируют новую election. Used in etcd, Consul, Kafka KRaft, quorum queues в RabbitMQ, Nomad, TiKV.

Paxos — original consensus algorithm, сложнее Raft. Использовался в Zookeeper (Zab это вариация Paxos). Много academic contributions но practical implementations complex.

Quorum это минимальное majority nodes для decision. 3 nodes требуют quorum 2 — можно потерять 1 node. 5 nodes требуют quorum 3 — можно потерять 2 nodes.

Почему нечётное число nodes. 4 nodes требуют quorum 3 — при потере 2 nodes невозможна работа. Тот же fault tolerance как 3 nodes (потеря 1) но дороже. Всегда предпочтительнее нечётное число.

Split-brain это критическая проблема где сеть разделяет cluster на несколько частей и каждая думает что «главная». Classical пример — 5 nodes разделяются 3 plus 2 из-за network partition. Часть с 3 имеет quorum и продолжает работать. Часть с 2 нет quorum — должна остановиться иначе data conflict при recovery.

Правильно построенные системы через Raft не дают split-brain благодаря quorum requirement. Плохо построенные (некоторые старые MySQL replication setups без proper coordination) split-brain приводит к data loss или corruption.

Cascading failure это когда падение одного node создаёт chain reaction. Одна node упала — nagruzka перераспределилась на других — они не справились — тоже упали. Может привести к complete outage cluster.

Защита от cascading failures. Circuit breaker fast fails при обнаружении проблем downstream. Rate limiting ограничивает traffic to prevent overload. Backpressure сигнализирует upstream что downstream перегружен. Auto-scaling добавляет capacity при росте load. Bulkheads изолируют resource pools для разных dependencies.

## Reference таблица типичных counts

Practical типичные конфигурации:

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
| Redis Cluster | 3 master + 3 replicas | minimum for meaningful HA |

Общий pattern — 3 nodes минимум для HA plus consensus-based systems. 5 nodes для higher fault tolerance. Больше при need для scaling или performance requirements.

## Как КНП использует nodes

Practical usage из КНП environment plus memory кейсов.

K8s nodes — несколько VM (172.158.0.12 extprod/intprod host, другие). Разные namespaces (knp, fno, tax-report) размещаются на общем worker pool. Control plane обычно 3 masters для HA.

Consul servers — 3 обычно для HA plus Raft consensus. Consul agents на каждом K8s worker node как DaemonSet — locally accessible service registry для applications.

PostgreSQL обычно primary plus replica configuration. Через PgBouncer как connection pooler. db-knp это PgBouncer в pod-сети — accessible только внутри cluster.

RabbitMQ cluster для messaging. Quorum queues для важных данных обеспечивающие HA.

Hazelcast distributed cache used. Memory кейс taxrep21-hazelcast-kesh66-unreachable упоминает что .66:5702 unreachable означает один member cluster недоступен — необходимо monitoring для detection.

Elasticsearch ELK stack для logs. Memory кейс knp-prod-historical-logs-elk описывает что kubectl logs показывает только current, исторические доступны только через Elasticsearch queries.

Kubelet и kube-proxy на каждом worker node — стандартная Kubernetes infrastructure.

## Итоги

Node это универсальный термин в distributed systems означающий один экземпляр — VM, container, process. Разные technologies имеют свои concepts nodes с специфическими характеристиками. Не путать между technologies.

Kubernetes nodes — control-plane plus worker. Компоненты kubelet, kube-proxy, container runtime на каждом. Node lifecycle через Ready, NotReady, SchedulingDisabled. Taints и tolerations для placement control.

Consul nodes — servers (Raft consensus) plus agents (на каждой application node). Gossip protocol для communication.

RabbitMQ nodes работают в cluster. Quorum queues для HA. pause_minority policy для split-brain protection.

PostgreSQL primary plus replicas. Streaming replication sync или async. Read replicas для scaling reads с caveat replication lag.

Kafka brokers cluster. Controller для coordination. Leader/follower per partition. ISR для reliable failover.

Elasticsearch multiple roles per node. Master, data, coordinating, ingest.

Cassandra peer-to-peer. Redis Cluster hash slots. Zookeeper ensemble. Neo4j graph node (не путать).

Общие принципы. Failure detection через heartbeat, gossip, active health checks. Raft consensus most popular. Quorum обеспечивает split-brain protection. Нечётное число nodes оптимально. Cascading failure защищается через circuit breakers, rate limiting, bulkheads.

Дальше — logging как критическая observability составляющая. SLF4J, Logback, ELK stack и best practices.
