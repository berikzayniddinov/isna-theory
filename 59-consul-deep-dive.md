# 59. Consul углубленно

Углубление файла `11-consul-detailed.md`: Raft внутри, gossip, ACL, multi-DC.

---

## 1. Быстрый recap

Consul = service registry + health + KV + DNS + multi-DC + service mesh (Connect).

Открытый вопрос: **как это работает под капотом**.

---

## 2. Архитектура — что живёт где

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

Два уровня:
- **Consensus (Raft)** — servers only. Strong consistency. Хранение состояния.
- **Gossip (Serf)** — все agents. Быстрое распространение состояния.

---

## 3. Raft — consensus protocol

**Raft** — алгоритм консенсуса. Проще Paxos, широко используется (etcd, Consul, TiKV, MongoDB WiredTiger).

### 3.1 Идея

Есть **лог операций** (append-only). Все ноды должны согласиться на **порядок** операций.

Гарантия: если операция в log'е → она **одинаковая на всех нодах**.

### 3.2 Роли

- **Leader** — один. Принимает writes, реплицирует followers.
- **Follower** — пассивные, следуют leader.
- **Candidate** — временно, во время выборов.

### 3.3 Elections

1. Все ноды стартуют как **Follower**.
2. Follower имеет **election timeout** (150-300 мс random).
3. Если timeout истёк и нет heartbeat от leader → становится **Candidate**.
4. Candidate голосует за себя + просит голоса других (`RequestVote` RPC).
5. Если получил большинство (quorum) → становится **Leader**.
6. Leader шлёт периодические heartbeats (`AppendEntries` без entries) — Followers не таймаутятся.

Random timeout — избежание split votes.

### 3.4 Log replication

Client пишет в leader:
1. Leader appendит entry в свой log.
2. Leader шлёт `AppendEntries` followers.
3. Followers appendят + возвращают ack.
4. Когда leader получил ack от **большинства (quorum)** — entry **committed**.
5. Leader применяет entry в state machine.
6. Leader возвращает success клиенту.
7. Followers применяют entry на следующем heartbeat.

### 3.5 Quorum

- **3 servers** → quorum 2 (может потерять 1).
- **5 servers** → quorum 3 (может потерять 2).
- **7 servers** → quorum 4 (может потерять 3).

**Правило**: **нечётное число** (3 или 5 для production).

**Почему нечётное**:
- 4 servers → quorum 3. Потеря 2 → нет quorum → не работает. То же что 3 servers, но дороже + больше failure surface.

### 3.6 Split-brain

Сеть разделила: 3 servers на две части (2+1).
- Часть с 2 — нет quorum → **не может выбрать нового leader** → блокируется.
- Часть с 1 — тем более.

Возможен только один leader в один момент времени → нет split-brain.

Когда сеть восстанавливается — минорная часть fetch'ит недостающие entries от лидера.

### 3.7 Персистентность

Raft log записывается на диск (fsync) до ack. Гарантия durability.

При рестарте node — восстанавливает log из диска.

---

## 4. Gossip — SWIM protocol

Между всеми agents (clients + servers) — **gossip**.

### 4.1 Что распространяется

- Membership (кто в кластере, кто ушёл).
- Failure detection (кто мёртв).
- User events.
- Metadata.

Быстро (сек), но не strong consistency.

### 4.2 Как работает (SWIM)

Каждый agent периодически:
1. Выбирает **случайного** peer.
2. Отправляет ping.
3. Если нет ответа → выбирает **k случайных** peers → просит их проверить.
4. Если и они не могут → peer помечается suspect.
5. Через timeout suspect → dead.

Информация распространяется gossip-style: peer A знает A→B, шлёт C, C шлёт D, ... экспоненциально быстро.

### 4.3 LAN vs WAN

- **LAN gossip** — внутри DC, часто (сотни мс).
- **WAN gossip** — между DC, реже (секунды).

WAN — только между **servers** разных DC.

### 4.4 Serf

Consul использует **Serf** — HashiCorp'овская реализация SWIM.

---

## 5. Consul Server vs Agent

### 5.1 Server

- 3 или 5 в DC (Raft quorum).
- Хранит state (services, health, KV).
- Участвует в consensus.
- Обрабатывает queries.

Настройка:
```
consul agent -server -bootstrap-expect=3 -data-dir=/opt/consul
```

### 5.2 Agent (Client)

- На каждой node (или как sidecar в pod).
- Легковесный.
- Форвардит запросы серверам.
- Локально выполняет health checks.
- Кэширует данные.

Настройка:
```
consul agent -data-dir=/opt/consul -retry-join=server1
```

### 5.3 Приложения общаются с local agent

Приложение → `localhost:8500` → agent → server.

Плюсы:
- Локальная latency.
- Agent кэширует.
- Не нужно знать server addresses.

---

## 6. Service registration

### 6.1 Через HTTP API

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

### 6.2 Через config файл

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

### 6.3 Через Spring Cloud

Автоматически (см. `11-consul-detailed.md`).

---

## 7. Health checks — глубже

### 7.1 Типы

**HTTP check**:
```json
{
  "HTTP": "http://localhost:8080/health",
  "Interval": "10s",
  "Timeout": "3s",
  "TLSSkipVerify": false
}
```

Consul дёргает URL, 200-399 = passing.

**TCP check**:
```json
{
  "TCP": "localhost:5432",
  "Interval": "30s"
}
```

Только handshake TCP.

**Script check**:
```json
{
  "Args": ["/usr/local/bin/check_something.sh"],
  "Interval": "10s"
}
```

Exit 0 = passing, 1 = warning, 2 = critical.

Требует `enable_script_checks: true` в config (security).

**Docker check**:
```json
{
  "DockerContainerID": "abc123",
  "Shell": "/bin/sh",
  "Args": ["/health.sh"]
}
```

Выполняет команду в контейнере.

**gRPC check**:
```json
{
  "GRPC": "localhost:9000",
  "Interval": "10s"
}
```

Standard gRPC health protocol.

**TTL check** — service сам должен уведомлять:
```json
{
  "TTL": "30s"
}
```

Service шлёт `PUT /v1/agent/check/pass/<check-id>` периодически. Если не шлёт за TTL → critical.

Для legacy систем которые не могут HTTP endpoint.

### 7.2 Статусы

- **passing** — healthy.
- **warning** — предупреждение (custom значение).
- **critical** — unhealthy.
- **unknown**.

### 7.3 DeregisterCriticalServiceAfter

Если service critical дольше N времени — Consul автоматически deregister.

Полезно для «мёртвых» инстансов после crash без graceful shutdown.

---

## 8. KV store

Простое ключ-значение хранилище.

### 8.1 API

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

### 8.2 CAS (Compare-And-Swap)

Для optimistic concurrency:
```bash
# Get с index
curl http://localhost:8500/v1/kv/counter
# Response: {"ModifyIndex": 5, "Value": "..."}

# Update if index матчит
curl -X PUT http://localhost:8500/v1/kv/counter?cas=5 -d 'new_value'
```

Если ModifyIndex не 5 (кто-то изменил) — операция не пройдёт.

### 8.3 Transactions

Атомарные операции над multiple keys:
```bash
curl -X PUT http://localhost:8500/v1/txn -d '
[
  { "KV": { "Verb": "set", "Key": "a", "Value": "..." } },
  { "KV": { "Verb": "cas", "Key": "b", "Index": 10, "Value": "..." } }
]'
```

Всё или ничего.

### 8.4 Watches

Подписаться на изменения:
```bash
curl "http://localhost:8500/v1/kv/foo?index=5&wait=30s"
```

Long-polling: возвращает если key изменился или через 30s.

Через **`consul watch`** — запускать команду при изменении.

### 8.5 Spring Cloud Consul Config

Конфиг из Consul KV:
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

При старте читает `config/application/data`, `config/isna-knp/data`. Работает через `bootstrap.yml` / `spring.config.import`.

---

## 9. Multi-DC federation

Consul может связать несколько DC:

```
┌── DC1 (Moscow) ──┐    ┌── DC2 (Almaty) ──┐
│  3 servers       │    │  3 servers       │
│  ...             │    │  ...             │
└──────────────────┘    └──────────────────┘
        │                       │
        └───── WAN gossip ──────┘
```

Каждый DC — независимый Raft. Между ними — WAN gossip для discovery.

### 9.1 Cross-DC queries

```bash
# Discovery в другом DC
curl http://localhost:8500/v1/catalog/service/foo?dc=dc2
```

DNS:
```
foo.service.dc2.consul
```

### 9.2 Prepared queries

Для failover:
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

Если orders в dc1 нет — Consul пробует dc2, потом dc3.

Для disaster recovery.

---

## 10. ACL system

Consul поддерживает ACL для authentication + authorization.

### 10.1 Policies

Rules на permissions:
```hcl
service "isnaKnpUser" {
    policy = "write"
}
key_prefix "config/isna-knp/" {
    policy = "read"
}
```

### 10.2 Tokens

```bash
consul acl token create -policy-name my-policy
```

Возвращает **secret token**. Клиент шлёт в header `X-Consul-Token`.

### 10.3 Bootstrap

Первый token (bootstrap):
```bash
consul acl bootstrap
```

Root token — управляет всем.

### 10.4 В prod обязательно

Без ACL — любой может register / modify services. Security hole.

---

## 11. Consul Connect (service mesh)

Consul умеет быть service mesh: mTLS между сервисами, intentions (кто может кому).

Реализуется через sidecar proxies (Envoy).

Приложение → localhost:proxy → mTLS → remote proxy → remote app.

Плюсы:
- Encryption in-transit.
- Central identity/authorization.

Минусы:
- Сложнее сетевая схема.
- Latency.

В ИСНА не используется (обычные HTTP + Consul discovery).

---

## 12. Consul в K8s

Два подхода:

### 12.1 Consul вне K8s

Servers на отдельных VM, K8s pods имеют agent как sidecar / DaemonSet.

Плюсы: Consul отдельный, не зависит от K8s.

Минусы: две системы discovery (Consul + K8s Service).

### 12.2 Consul в K8s

**Helm chart** от HashiCorp. Servers + agents как StatefulSet + DaemonSet.

Плюсы: unified deploy.

### 12.3 В ИСНА

Consul отдельно (не в K8s). Приложения в K8s pods имеют доступ к Consul через сервис.

---

## 13. Мониторинг Consul

### 13.1 Metrics

Consul экспортирует metrics в:
- **Statsd**.
- **DogStatsD**.
- **Prometheus** (`/v1/agent/metrics?format=prometheus`).

Ключевые:
- `consul.raft.leader.dispatch_log` — latency writes.
- `consul.raft.state.leader` — 1 если этот server leader.
- `consul.serf.member.left` — уходящие members.
- `consul.rpc.query` — rate queries.
- `consul.catalog.service` — services count.

### 13.2 UI

`http://consul:8500/ui/`.

Показывает:
- Services + инстансы + статусы.
- Nodes.
- KV.
- Intentions.

### 13.3 Логи

```
INFO agent: Synced service: service=isnaKnpUser
WARN agent: Check "http-check" is now critical
```

---

## 14. Типовые проблемы

### 14.1 Leader instability

Частые re-elections. Причины:
- Сеть (heartbeats таймаутят).
- Server перегружен.

Fix: диагностика network + resources.

### 14.2 Slow writes

Все writes через leader. Может стать bottleneck.

Fix:
- **Prepared queries** для read.
- **Cache** результатов на клиенте.
- **Больше serversов** не помогает write (только 1 leader).

### 14.3 «Service показывает passing, но приложение сломано»

Проверь тип check:
- `tcpSocket` может врать (порт открыт, app мёртв).
- Используй HTTP на `/actuator/health` который знает реальное состояние.

### 14.4 Deregistration issues

Из memory ИСНА `knp-fs-consul-deregister-after-db-flap`: под живой, но Consul dereg'нул после флапа БД (pool=1 + TCP-only readiness). Fix — `rollout restart` пода.

### 14.5 Split brain при network partition

Раз есть quorum-based Raft — split brain невозможен. Одна сторона (без quorum) блокируется, другая продолжает.

**Кавет**: если DC разделён 3+3 — обе части имеют 3 (нет большинства ни у одной) → **обе не работают**. Отсюда 5-server кластер безопаснее.

---

## 15. Best practices

1. **3 или 5 servers** на DC.
2. **Servers separate** от worker nodes.
3. **HTTP health checks** на `/actuator/health`.
4. **query-passing: true** обязательно.
5. **DeregisterCriticalServiceAfter** для очистки dead инстансов.
6. **ACL включён** в prod.
7. **TLS** между agents и servers.
8. **Backup KV** (snapshot).
9. **Multi-DC** для DR.
10. **Мониторинг**: Prometheus + Grafana.

---

## 16. Собесные вопросы

1. **Как работает Consul под капотом?** — Raft (consensus) + gossip (SWIM) слои.
2. **Что делает Raft?** — Consensus algorithm: leader election + log replication + quorum.
3. **Почему 3 или 5 servers?** — Quorum (majority) требует нечётное; безопасно потерять (N-1)/2.
4. **Split brain в Consul?** — Невозможен благодаря quorum; часть без quorum блокируется.
5. **Что такое gossip / SWIM?** — Peer-to-peer протокол для быстрого распространения membership + failure detection.
6. **Разница Server и Agent?** — Server: Raft, хранит state. Agent: локальный на каждой ноде, форвардит + health.
7. **Health check types?** — HTTP, TCP, Script, Docker, gRPC, TTL.
8. **TTL check — когда?** — Legacy систем, которые не могут HTTP endpoint; app сам «пингует».
9. **Что делает DeregisterCriticalServiceAfter?** — Автоматически удаляет сервис из registry после N времени critical.
10. **Consul KV — для чего?** — Distributed KV store; конфиги, feature flags, leader election.
11. **CAS (Compare-And-Swap) в KV?** — Оптимистичная блокировка через ModifyIndex.
12. **Multi-DC — как?** — Каждый DC свой Raft; WAN gossip для discovery; failover через prepared queries.
13. **Что такое Consul Connect?** — Service mesh; mTLS через Envoy sidecar.
14. **ACL system — зачем?** — Authentication + authorization; policies + tokens.
15. **Как избежать split votes при election?** — Random election timeout (150-300 мс).

---

## Итог

- **Raft** для strong consistency (state).
- **SWIM/Gossip** для fast membership.
- **3 или 5 servers** на DC.
- **Quorum** предотвращает split-brain.
- **KV** с CAS для config + coordination.
- **Multi-DC + Prepared Queries** для DR.
- **ACL + TLS** в prod.

Следующий — `60-spring-annotations-detailed.md`.
