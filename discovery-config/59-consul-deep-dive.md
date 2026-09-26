# 59. Consul deep dive: Raft, Gossip, KV, ACL, Multi-DC

## Зачем углубляться в Consul

Файл 11 покрыл Consul как service registry со стандартной high-level perspective. Разработчик может configure application для Consul discovery, register services, use health checks. Работает большинство ежедневных tasks. Реальные production incidents требуют deeper understanding. Consul cluster split-brained — почему? Как выбирает нового leader? Consul writes медленные — что bottleneck? Multi-DC federation — как работает failover?

Разница между разработчиком «использующим Consul» и «понимающим Consul internals» проявляется в incident response и capacity planning. Первый сталкивается с Consul issue — открывает docs, tries solutions randomly. Второй знает что Consul basis — Raft для consensus (state), Gossip для membership. Знает что Raft requires quorum majority — 3 servers tolerate 1 failure, 5 servers tolerate 2. Знает разницу server и agent — servers участвуют в consensus и хранят state, agents на каждой node forward к servers. Знает что все writes идут через leader — write throughput bottleneck. Знает что health checks через различные mechanisms (HTTP, TCP, TTL) с разными trade-offs.

В этом файле разберём Consul detail. Быстрый recap. Архитектура — что живёт где. Raft protocol в деталях — elections, log replication, quorum. Gossip protocol (SWIM) — membership и failure detection. Consul Server vs Agent — role separation. Service registration mechanisms. Health checks types глубоко. KV store с CAS и transactions. Multi-DC federation. ACL system. Consul Connect (service mesh). Deployment в K8s. Мониторинг. Common issues. Best practices.

## Recap: что такое Consul

Consul = service registry plus health plus KV plus DNS plus multi-DC plus service mesh (Connect). Multi-functional platform HashiCorp.

Открытый вопрос — как это работает под капотом.

## Архитектура

Two-layer architecture:
```
┌────── DATACENTER (dc1) ──────────────────────────────┐
│                                                       │
│  ┌───── CONSENSUS LAYER (Raft) ────────────────┐    │
│  │                                              │    │
│  │  Server 1 (LEADER)  ◄─Raft log replic─►  Server 2 (Follower)
│  │                                          │    │
│  │           ▲                              │    │
│  │           │                              │    │
│  │        Raft log  ◄─────────────►  Server 3 (Follower)
│  │                                              │    │
│  └──────────────────────────────────────────────┘    │
│              ▲                                        │
│              │                                        │
│  ┌───── GOSSIP LAYER (Serf/SWIM) ──────────────┐    │
│  │                                              │    │
│  │  Agent 1 (client)  ◄─gossip─►  Agent 2      │    │
│  │       │                              │        │    │
│  │       ▼                              ▼        │    │
│  │  App 1                          App 2        │    │
│  │                                              │    │
│  │  Agent N (client)                            │    │
│  └──────────────────────────────────────────────┘    │
│                                                       │
└───────────────────────────────────────────────────────┘
        │  (WAN gossip)                       │
        └────────────────────────────────────►│
                                            other DC
```

Two levels different guarantees.

Consensus (Raft) — servers only. Strong consistency. Хранение state. Persistent decisions.

Gossip (Serf) — все agents. Быстрое распространение состояния. Membership информация. Failure detection.

Two-layer approach — Raft для reliable state management, Gossip для scalable membership tracking. Different requirements different solutions.

## Raft: consensus protocol

Raft — алгоритм консенсуса. Проще Paxos, широко используется (etcd, Consul, TiKV, MongoDB WiredTiger).

Идея. Есть лог операций (append-only). Все ноды должны согласиться на порядок операций. Гарантия — если операция в log, она одинаковая на всех нодах.

Roles. Leader — один. Принимает writes, реплицирует followers. Follower — пассивные, следуют leader. Candidate — временно, во время выборов.

Elections mechanism. Все ноды стартуют как Follower. Follower имеет election timeout (150-300 мс random). Если timeout истёк и нет heartbeat от leader — становится Candidate. Candidate голосует за себя plus просит голоса других через RequestVote RPC. Если получил большинство (quorum) — становится Leader. Leader шлёт периодические heartbeats (AppendEntries без entries) — Followers не таймаутятся.

Random timeout critical. Обеспечивает избежание split votes. Если бы все Followers timeout'ились одновременно и все стали Candidates simultaneously — split votes, никто не получает majority, retry. Random spreads timeouts.

Log replication flow. Client пишет в leader. Leader appendит entry в свой log. Leader шлёт AppendEntries followers. Followers appendят plus возвращают ack. Когда leader получил ack от большинства (quorum) — entry committed. Leader применяет entry в state machine. Leader возвращает success клиенту. Followers применяют entry на следующем heartbeat.

Two-phase — first replicate to majority, then commit. Ensures no data loss даже если leader crashes перед full replication.

Quorum. 3 servers — quorum 2 (может потерять 1). 5 servers — quorum 3 (может потерять 2). 7 servers — quorum 4 (может потерять 3).

Правило — нечётное число (3 или 5 для production). Почему нечётное. 4 servers — quorum 3. Потеря 2 — нет quorum, не работает. То же что 3 servers, но дороже plus больше failure surface.

Split-brain protection. Сеть разделила — 3 servers на две части (2+1). Часть с 2 — нет quorum — не может выбрать нового leader — блокируется. Часть с 1 — тем более. Возможен только один leader в один момент времени — нет split-brain.

Когда сеть восстанавливается — минорная часть fetch'ит недостающие entries от лидера. Automatic recovery.

Персистентность. Raft log записывается на диск (fsync) до ack. Гарантия durability. Даже если node crashes, log preserved. При рестарте node восстанавливает log из диска.

## Gossip: SWIM protocol

Между всеми agents (clients + servers) — gossip. Fast, eventually consistent membership plus failure detection.

Что распространяется. Membership (кто в кластере, кто ушёл). Failure detection (кто мёртв). User events. Metadata.

Быстро (сек), но не strong consistency. Different guarantees чем Raft.

Как работает SWIM. Каждый agent периодически. Выбирает случайного peer. Отправляет ping. Если нет ответа — выбирает k случайных peers, просит их проверить. Если и они не могут — peer помечается suspect. Через timeout suspect — dead.

Информация распространяется gossip-style. Peer A знает A→B, шлёт C, C шлёт D, экспоненциально быстро. Randomization prevents hotspots.

Elegant design. Scalable — каждый agent talks к constant number peers regardless total size. Reliable — indirect probes catch cases где direct communication fails but node alive.

LAN vs WAN. LAN gossip — внутри DC, часто (сотни мс). WAN gossip — между DC, реже (секунды).

WAN — только между servers разных DC. Cross-DC coordination different frequency plus mechanism.

Serf. Consul использует Serf — HashiCorp's реализация SWIM. Same team, tight integration.

## Consul Server vs Agent

Server. 3 или 5 в DC (Raft quorum). Хранит state (services, health, KV). Участвует в consensus. Обрабатывает queries.

Настройка:
```
consul agent -server -bootstrap-expect=3 -data-dir=/opt/consul
```

Agent (Client). На каждой node (или как sidecar в pod). Легковесный. Форвардит запросы серверам. Локально выполняет health checks. Кэширует данные.

Настройка:
```
consul agent -data-dir=/opt/consul -retry-join=server1
```

Приложения общаются с local agent через localhost:8500 → agent → server.

Плюсы этой топологии. Локальная latency — agent right on same node. Agent кэширует — reduces server load. Не нужно знать server addresses — agent handles routing.

Server placement. Dedicated VMs typically. Не совмещать с worker workloads — servers should be reliably available.

## Service registration

Multiple mechanisms.

Через HTTP API:
```
PUT http://localhost:8500/v1/agent/service/register
{
  "Name": "isnaKnpUser",
  "ID": "isnaKnpUser-uuid",
  "Address": "10.0.1.5",
  "Port": 8080,
  "Tags": ["v1.0.42", "prod"],
  "Meta": { "version": "1.0.42" },
  "Check": {
    "HTTP": "http://10.0.1.5:8080/actuator/health",
    "Interval": "15s",
    "Timeout": "3s",
    "DeregisterCriticalServiceAfter": "1m"
  }
}
```

Programmatic registration. Application at startup registers, at shutdown deregisters.

Через config файл:
```json
{
  "service": {
    "name": "isnaKnpUser",
    "port": 8080,
    "tags": ["v1.0.42"],
    "check": { ... }
  }
}
```

Static configuration. Loaded when agent starts. Reloaded on SIGHUP.

Через Spring Cloud автоматически. spring-cloud-consul-discovery starter. Register at startup, deregister at shutdown. Uses HTTP API under hood. See file 11 для details.

## Health checks глубже

Types. Different mechanisms для different scenarios.

HTTP check:
```json
{
  "HTTP": "http://localhost:8080/health",
  "Interval": "10s",
  "Timeout": "3s",
  "TLSSkipVerify": false
}
```

Consul дёргает URL, 200-399 = passing. Application level verification. Most common для web services.

TCP check:
```json
{
  "TCP": "localhost:5432",
  "Interval": "30s"
}
```

Только handshake TCP. Fast but shallow — port open doesn't guarantee application working properly.

Script check:
```json
{
  "Args": ["/usr/local/bin/check_something.sh"],
  "Interval": "10s"
}
```

Exit 0 = passing, 1 = warning, 2 = critical. Custom logic. Requires enable_script_checks: true в config (security).

Docker check:
```json
{
  "DockerContainerID": "abc123",
  "Shell": "/bin/sh",
  "Args": ["/health.sh"]
}
```

Выполняет команду в контейнере. Applications packaged в containers.

gRPC check:
```json
{
  "GRPC": "localhost:9000",
  "Interval": "10s"
}
```

Standard gRPC health protocol. Native support для gRPC services.

TTL check — service сам должен уведомлять:
```json
{
  "TTL": "30s"
}
```

Service шлёт PUT /v1/agent/check/pass/<check-id> периодически. Если не шлёт за TTL — critical.

Для legacy систем которые не могут HTTP endpoint. Push-based instead of pull-based.

Статусы. passing — healthy. warning — предупреждение (custom значение). critical — unhealthy. unknown.

DeregisterCriticalServiceAfter. Если service critical дольше N времени — Consul автоматически deregister. Полезно для «мёртвых» инстансов после crash без graceful shutdown.

Automatic cleanup prevents accumulating stale service registrations. Discovery queries return only actually alive services.

## KV store

Простое ключ-значение хранилище. Multiple use cases beyond just configuration.

API basic operations:
```bash
# Put
curl -X PUT http://localhost:8500/v1/kv/knp/config/feature-flag -d 'true'

# Get
curl http://localhost:8500/v1/kv/knp/config/feature-flag?raw

# List keys
curl http://localhost:8500/v1/kv/knp/?keys

# Recursive
curl http://localhost:8500/v1/kv/knp/?recurse

# Delete
curl -X DELETE http://localhost:8500/v1/kv/knp/config/feature-flag

# Delete recursive
curl -X DELETE http://localhost:8500/v1/kv/knp/?recurse
```

Hierarchical keys through / separator. Recurse for tree operations.

CAS (Compare-And-Swap) для optimistic concurrency:
```bash
# Get с index
curl http://localhost:8500/v1/kv/counter
# Response: {"ModifyIndex": 5, "Value": "..."}

# Update if index матчит
curl -X PUT http://localhost:8500/v1/kv/counter?cas=5 -d 'new_value'
```

Если ModifyIndex не 5 (кто-то изменил) — операция не пройдёт. Enables lock-free coordination protocols.

Transactions атомарные операции над multiple keys:
```bash
curl -X PUT http://localhost:8500/v1/txn -d '
[
  { "KV": { "Verb": "set", "Key": "a", "Value": "..." } },
  { "KV": { "Verb": "cas", "Key": "b", "Index": 10, "Value": "..." } }
]'
```

Всё или ничего. Multi-key atomicity через Raft consensus.

Watches подписаться на изменения:
```bash
curl "http://localhost:8500/v1/kv/foo?index=5&wait=30s"
```

Long-polling. Возвращает если key изменился или через 30s. Enables reactive configuration systems.

Через consul watch — запускать команду при изменении. Automation trigger.

Spring Cloud Consul Config. Конфиг из Consul KV:
```yaml
spring:
  cloud:
    consul:
      config:
        enabled: true
        format: yaml
        prefix: config
        default-context: application
```

При старте читает config/application/data, config/isna-knp/data. Работает через bootstrap.yml / spring.config.import.

Runtime configuration updates. Better чем redeploying для config changes.

## Multi-DC federation

Consul может связать несколько DC:
```
┌── DC1 (Moscow) ──┐    ┌── DC2 (Almaty) ──┐
│  3 servers       │    │  3 servers       │
│  ...             │    │  ...             │
└──────────────────┘    └──────────────────┘
        │                       │
        └───── WAN gossip ──────┘
```

Каждый DC — независимый Raft. Между ними — WAN gossip для discovery. DCs remain autonomous — network partition между DCs не breaks anything locally.

Cross-DC queries:
```bash
# Discovery в другом DC
curl http://localhost:8500/v1/catalog/service/foo?dc=dc2
```

DNS:
```
foo.service.dc2.consul
```

Explicit DC selection. Applications can query specific DC when needed.

Prepared queries для failover:
```json
{
  "Name": "orders",
  "Service": {
    "Service": "orders",
    "Failover": {
      "Datacenters": ["dc2", "dc3"]
    }
  }
}
```

Если orders в dc1 нет — Consul пробует dc2, потом dc3. Для disaster recovery. Application не needs to know DC topology.

## ACL system

Consul поддерживает ACL для authentication plus authorization.

Policies rules на permissions:
```hcl
service "isnaKnpUser" {
    policy = "write"
}
key_prefix "config/isna-knp/" {
    policy = "read"
}
```

Fine-grained access control. Service-level plus KV-prefix based.

Tokens:
```bash
consul acl token create -policy-name my-policy
```

Возвращает secret token. Клиент шлёт в header X-Consul-Token.

Bootstrap first token:
```bash
consul acl bootstrap
```

Root token — управляет всем. Setup once, used для creating other tokens plus policies.

В prod обязательно. Без ACL — любой может register / modify services. Security hole. Cluster access = full control.

## Consul Connect (service mesh)

Consul умеет быть service mesh — mTLS между сервисами, intentions (кто может кому).

Реализуется через sidecar proxies (Envoy).

Приложение → localhost:proxy → mTLS → remote proxy → remote app. Zero-trust security model.

Плюсы. Encryption in-transit. Central identity/authorization. Traffic management (retries, timeouts) на mesh level.

Минусы. Сложнее сетевая схема. Extra hops добавляют latency. Sidecar containers per pod resource overhead.

В КНП не используется. Обычные HTTP plus Consul discovery. Simpler operational model.

## Consul в K8s

Два подхода.

Consul вне K8s. Servers на отдельных VMs. K8s pods имеют agent как sidecar или DaemonSet. Плюсы — Consul отдельный, не зависит от K8s upgrades или issues. Минусы — две системы discovery (Consul plus K8s Service) coexisting.

Consul в K8s. Helm chart от HashiCorp. Servers как StatefulSet plus agents как DaemonSet. Плюсы — unified deploy, single infrastructure paradigm. Минусы — Consul availability tied к K8s cluster health.

В ИСНА Consul отдельно (не в K8s). Приложения в K8s pods имеют доступ к Consul через сервис. Isolation preferred для infrastructure critical to K8s workloads.

## Мониторинг Consul

Metrics. Consul экспортирует metrics в. Statsd. DogStatsD. Prometheus (/v1/agent/metrics?format=prometheus).

Ключевые metrics. consul.raft.leader.dispatch_log — latency writes. consul.raft.state.leader — 1 если этот server leader. consul.serf.member.left — уходящие members. consul.rpc.query — rate queries. consul.catalog.service — services count.

Alerting on. Sustained non-leader state (all follower). Leader changes frequency (instability). Growing dispatch log latency (writes bottleneck). Member left events (nodes failing).

UI. http://consul:8500/ui/. Показывает services plus инстансы plus статусы. Nodes. KV. Intentions.

Logs:
```
INFO agent: Synced service: service=isnaKnpUser
WARN agent: Check "http-check" is now critical
```

Structured logging для operational awareness.

## Типовые проблемы

Leader instability. Частые re-elections. Причины. Сеть (heartbeats таймаутят). Server перегружен. Slow disk (Raft log fsync slow).

Fix. Диагностика network plus resources. Better disk (SSD required for Raft servers). Isolated network. Adequate CPU для Raft threads.

Slow writes. Все writes через leader. Может стать bottleneck при high load.

Fix. Prepared queries для read (cacheable, faster). Cache результатов на клиенте. Больше servers не помогает write — только 1 leader.

«Service показывает passing, но приложение сломано». Проверь тип check. TCP socket may lie (port open, app dead). Используй HTTP на /actuator/health который знает реальное состояние.

Deregistration issues. Из memory ИСНА knp-fs-consul-deregister-after-db-flap. Под живой, но Consul dereg'нул после флапа БД (pool=1 + TCP-only readiness). Fix — rollout restart пода plus improve readiness check.

Split brain при network partition. Раз есть quorum-based Raft — split brain невозможен. Одна сторона (без quorum) блокируется, другая продолжает.

Кавет. Если DC разделён 3+3 — обе части имеют 3 (нет большинства ни у одной) — обе не работают. Отсюда 5-server кластер безопаснее в этом сценарии.

## Best practices

3 или 5 servers на DC. Odd number для quorum.

Servers separate от worker nodes. Dedicated infrastructure. Predictable performance.

HTTP health checks на /actuator/health. Application-level verification. Reflects real state.

query-passing: true обязательно. Filter results to passing services only.

DeregisterCriticalServiceAfter. Automatic cleanup dead instances.

ACL включён в prod. Security requirement.

TLS между agents и servers. Encrypted communication.

Backup KV (snapshot). Disaster recovery.

Multi-DC для DR. Geographical distribution.

Мониторинг. Prometheus plus Grafana. Alerting on critical metrics.

## Итоги

Consul architecture — Raft (consensus, servers) plus Gossip (membership, all agents). Two-layer approach.

Raft для strong consistency (state). Leader election через random timeouts. Log replication через majority quorum. Split-brain prevention через quorum requirement.

3 или 5 servers для quorum. Нечётное число оптимально. Даже number — same fault tolerance как previous odd number, но дороже.

SWIM/Gossip для fast membership. Peer-to-peer, epidemic propagation. LAN и WAN levels.

Server vs Agent role separation. Servers held state via Raft. Agents forward к servers plus local health checks. Applications talk к local agent.

Service registration через HTTP API, config file, или Spring Cloud automatic.

Health checks types. HTTP, TCP, script, Docker, gRPC, TTL. Different mechanisms для different scenarios. HTTP на /actuator/health preferred для application-level verification.

KV store с CAS для optimistic concurrency. Transactions для multi-key atomicity. Watches для reactive systems. Spring Cloud Consul Config для runtime configuration.

Multi-DC federation через WAN gossip. Prepared queries для automatic failover between DCs.

ACL system для authentication и authorization. Policies plus tokens. Обязательно в prod.

Consul Connect — service mesh option через Envoy sidecars. mTLS, intentions. Не используется в КНП.

Consul в K8s — deploy separate или через Helm chart. КНП использует separate deployment.

Monitoring через Prometheus. Key metrics — leader stability, dispatch latency, member changes, service counts.

Common issues и fixes. Leader instability, slow writes, misleading health checks, deregistration flakiness.

Best practices. Odd server count. Separate servers. HTTP health checks. ACL. TLS. Backups. Monitoring.

Дальше — главные Spring аннотации детально с internals каждой.
