# 73. Hazelcast

Java-native distributed in-memory data grid. В ИСНА активно используется.

---

## 1. Что такое Hazelcast

**Hazelcast** — **distributed in-memory data grid (IMDG)** для Java.

- Написан на **Java**.
- **Distributed** — данные разделены между несколькими node.
- **In-memory** — всё в RAM (persistence опциональна).
- **Peer-to-peer** — все ноды равноправны (нет master).
- **Java-native** — прямые Java-объекты, `Map`/`Set`/`Queue` интерфейсы.
- **Embedded** режим — можно поднять внутри Spring Boot приложения.

Компания Hazelcast Inc. Open source (Apache 2.0) + Enterprise version.

---

## 2. Что даёт

### 2.1 Distributed data structures

`IMap<K, V>` — распределённая Map (partitioned).
`IList`, `ISet`, `IQueue`, `MultiMap`, `ReplicatedMap`, `ITopic`.

Java-код:
```java
HazelcastInstance hz = Hazelcast.newHazelcastInstance();
IMap<String, User> users = hz.getMap("users");
users.put("berik", new User("Berik"));
User u = users.get("berik");   // может быть на другой ноде — прозрачно
```

Выглядит как обычная Map, но данные распределены.

### 2.2 Distributed computing

- **IExecutorService** — Runnable/Callable выполнить на кластере.
- **EntryProcessor** — атомарно обработать элемент IMap на его ноде.

### 2.3 Distributed synchronization

- **ILock** — distributed lock.
- **IAtomicLong**, **IAtomicReference** — атомарные значения.
- **ICountDownLatch**, **ISemaphore**.

### 2.4 Distributed messaging

- **ITopic** — pub/sub.
- **IQueue** — distributed queue.

### 2.5 Near cache

Local cache на клиенте с TTL. Ускоряет reads.

---

## 3. Архитектура

### 3.1 Cluster

Node discoveries друг друга и образуют **cluster**:
- Discovery через **multicast** (dev) или **TCP-IP** список (prod) или **Kubernetes**, **AWS**, **Consul**.
- Все ноды равноправны.

```
┌─── Node 1 ───┐   ┌─── Node 2 ───┐   ┌─── Node 3 ───┐
│  partition:  │   │  partition:  │   │  partition:  │
│   0-90       │◄─►│   91-180     │◄─►│  181-270     │
│              │   │              │   │              │
│ + backups    │   │ + backups    │   │ + backups    │
└──────────────┘   └──────────────┘   └──────────────┘
```

### 3.2 Partitions

По default — **271 partitions**. Каждый key → hash → partition → owner node.

Данные распределены **равномерно** между node.

### 3.3 Backups

Каждая partition имеет **backup** на другой node.

`backup-count=1` (default) — 1 backup.
`sync-backups=1` — sync replication.

При падении node — backup становится primary.

### 3.4 Rebalance

Node join / leave → rebalance partitions (redistribution).

Может быть медленный при большом дата set.

---

## 4. Embedded vs Client-Server

### 4.1 Embedded

Hazelcast **внутри** Spring Boot приложения:
```java
@Bean
HazelcastInstance hazelcast() {
    Config config = new Config();
    return Hazelcast.newHazelcastInstance(config);
}
```

Приложение = node в кластере.

Плюсы:
- Нет network hop.
- Простой setup.
- Быстро (в JVM).

Минусы:
- Restart pod = уход node из cluster (rebalance).
- Разделение resources с приложением.

Хорошо для: cache в микросервисах.

### 4.2 Client-Server

Hazelcast — **отдельный кластер**, приложения — clients.

```java
ClientConfig config = new ClientConfig();
config.getNetworkConfig().addAddress("hazelcast:5701");
HazelcastInstance client = HazelcastClient.newHazelcastClient(config);
```

Плюсы:
- Independent scaling.
- Данные переживают restart приложений.
- Много клиентов → один кластер.

Минусы:
- Extra network hop.
- Ещё один компонент управлять.

---

## 5. IMap — основной use case

Распределённая **ConcurrentMap**:

```java
IMap<String, User> users = hz.getMap("users");

users.put("berik", new User("Berik"));
User u = users.get("berik");

// Atomic
users.putIfAbsent("berik", new User("Berik"));
users.replace("berik", oldValue, newValue);

// EntryProcessor — атомарно на owner node
users.executeOnKey("berik", entry -> {
    User u = entry.getValue();
    u.setLastLogin(now());
    entry.setValue(u);
    return null;
});
```

### 5.1 Настройки IMap

```java
MapConfig mapConfig = new MapConfig("users")
    .setBackupCount(1)
    .setTimeToLiveSeconds(3600)       // TTL
    .setMaxIdleSeconds(600)             // idle expiration
    .setEvictionConfig(new EvictionConfig()
        .setEvictionPolicy(LRU)
        .setSize(10000)
        .setMaxSizePolicy(PER_NODE));
```

- **Backup count** — сколько replicas.
- **TTL / max idle** — expiration.
- **Eviction policy** — LRU / LFU / RANDOM.
- **Max size** — лимит.

### 5.2 Near cache

Local cache на клиенте:
```java
mapConfig.setNearCacheConfig(new NearCacheConfig()
    .setTimeToLiveSeconds(60)
    .setMaxSizePolicy(...)
    .setInvalidateOnChange(true));
```

Первый `get` — network hop к owner; кэшируется локально; последующие — из RAM клиента.

Ускоряет reads в разы. Но: stale data если invalidation lag.

---

## 6. Persistence

По default — **volatile**. Restart cluster = потеря данных.

Опции:
- **MapStore** — write-through / write-behind в БД.
- **Hot Restart Persistence** (Enterprise) — snapshot на диск.
- **Hazelcast Persistence** (Platform 5.0+) — CP subsystem + WAL.

Для чистого cache persistence не нужна (переспросить у источника).

---

## 7. Distributed lock

```java
IMap<String, ?> map = hz.getMap("myMap");
Lock lock = map.getLock("myLock");

lock.lock();
try {
    // критическая секция
} finally {
    lock.unlock();
}
```

Работает через кластер — только один node в моменте держит.

С 4.0 — **FencedLock** через CP subsystem (Raft-based, сильнее гарантии).

---

## 8. Pub/Sub через ITopic

```java
ITopic<String> topic = hz.getTopic("orders");

topic.addMessageListener(msg -> {
    System.out.println("Got: " + msg.getMessageObject());
});

topic.publish("new order 42");
```

Не persistent — offline subscribers пропускают.

---

## 9. Distributed computing

### 9.1 IExecutorService

```java
IExecutorService exec = hz.getExecutorService("myExec");

Future<String> result = exec.submit(new MyCallable());

// на конкретной ноде
exec.submitToMember(callable, member);

// на всех
Map<Member, Future<String>> results = exec.submitToAllMembers(callable);
```

Compute на кластере параллельно.

### 9.2 EntryProcessor

Атомарно на owner node ключа:
```java
users.executeOnKey("berik", entry -> {
    User u = entry.getValue();
    u.incrementLoginCount();
    entry.setValue(u);
    return u.getLoginCount();
});
```

Без network round-trip для read-modify-write.

---

## 10. Configuration

### 10.1 XML

```xml
<hazelcast>
    <cluster-name>my-cluster</cluster-name>

    <network>
        <port>5701</port>
        <join>
            <tcp-ip enabled="true">
                <member>node1:5701</member>
                <member>node2:5701</member>
                <member>node3:5701</member>
            </tcp-ip>
        </join>
    </network>

    <map name="users">
        <backup-count>1</backup-count>
        <time-to-live-seconds>3600</time-to-live-seconds>
        <eviction eviction-policy="LRU" max-size-policy="PER_NODE" size="10000"/>
    </map>
</hazelcast>
```

### 10.2 YAML

```yaml
hazelcast:
  cluster-name: my-cluster
  network:
    port: 5701
    join:
      tcp-ip:
        enabled: true
        member-list:
          - node1:5701
          - node2:5701
  map:
    users:
      backup-count: 1
      time-to-live-seconds: 3600
```

### 10.3 Programmatic

```java
Config config = new Config()
    .setClusterName("my-cluster")
    .setNetworkConfig(new NetworkConfig()
        .setPort(5701)
        .setJoin(new JoinConfig()
            .setTcpIpConfig(new TcpIpConfig()
                .setEnabled(true)
                .addMember("node1:5701"))));

HazelcastInstance hz = Hazelcast.newHazelcastInstance(config);
```

---

## 11. Discovery в K8s

```yaml
hazelcast:
  network:
    join:
      kubernetes:
        enabled: true
        namespace: default
        service-name: hazelcast-service
```

Ноды находят друг друга через K8s Service.

Или через **Consul** discovery (spring-cloud-hazelcast-consul-discovery).

---

## 12. Hibernate 2nd level cache

Классический use case — **кэш второго уровня Hibernate**.

```gradle
implementation 'org.hibernate:hibernate-jcache'
implementation 'com.hazelcast:hazelcast'
implementation 'com.hazelcast:hazelcast-hibernate53'
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region.factory_class: com.hazelcast.hibernate.HazelcastCacheRegionFactory
```

Все Entity с `@Cache` кэшируются в Hazelcast → доступно между JVM'ами.

В ИСНА так и используется.

---

## 13. Использование в ИСНА

Из memory:

### 13.1 knp-form-hz5-actuator-cache-nosuchmethod

**Ключевой кейс миграции HZ 3.12 → 5.3.8**:

- Spring Boot 2.2.4 actuator + Hazelcast 5 = 2 NoSuchMethodError:
  1. `getNativeCache` при СТАРТЕ (CacheMetricsAutoConfiguration).
  2. `getLocalEndpoint` при HEALTH (HazelcastHealthContributorAutoConfiguration).
- Симптом: pod старт → 0/1 при httpGet-readiness.
- **tcpSocket-деплойменты маскируют** (порт открыт до падения).
- **Fix**: `spring.autoconfigure.exclude` обе auto-configuration классы в 4 модулях.
- На проде обошли через env `SPRING_AUTOCONFIGURE_EXCLUDE` (не в git!).
- МР `!409` → master.

### 13.2 arm/INT форма ещё на HZ3, не на 5.6 → флап kesh → рестарты

Memory `knp-form-arm-hazelcast3-not-migrated-flap`:
- `prod-smoke-form-hazelcast` красный на INT.
- arm-isna форма до сих пор на **HZ-клиенте 3.12.5** и старый `kesh:5701`.
- kesh **флапает** → `HazelcastHealthIndicator` DOWN → k8s бьёт поды exit=143.
- EXT/knp зелёный (5.6, 172.19.1.4x), препрод на новом клиенте; cutover !400 доехал до EXT+ПП, не до arm.

### 13.3 taxrep21-hazelcast-kesh66-unreachable

- ns tax-report21 все 53 пода Running/0 рестартов.
- Единственный шум — Hazelcast client 5.6/порт5702 вечно ретраит недоступный member 172.19.21.66 (etprod-kesh).
- J11 прод чист (client 3.12/порт5701).
- Фикс = поднять .66:5702 или убрать из `HAZELCAST_CLIENT_NETWORK_CLUSTER_MEMBERS`.

**Урок**: миграция Hazelcast между major версиями = много каветов (API changes, health checks, actuator). В ИСНА разные модули на разных версиях.

---

## 14. Hazelcast 5 vs 3.x — что изменилось

Из memory ИСНА:
- **Package rename**: `com.hazelcast.core.*` → в 5.x реорганизовано.
- **`ICache`** deprecated → JCache API.
- **Cache Region Factory** — новые imports.
- **HealthContributor** API изменён (getLocalEndpoint).
- **Client protocol** — new binary protocol.
- **CP subsystem** для strong consistency (Raft).

Миграция требует:
- Обновить imports.
- Обновить конфиг (XML/YAML).
- Заново протестировать health checks.
- Exclude actuator auto-config если нужно.

---

## 15. Hazelcast Management Center

Web UI для мониторинга кластера:
- Nodes, health.
- Map stats (size, hit rate, evictions).
- Metrics (memory, CPU, ops/sec).
- Slow ops.

Отдельный процесс, подключается к кластеру.

---

## 16. Hazelcast Jet / Platform

**Hazelcast Jet** — streaming engine (аналог Kafka Streams / Flink).

С 5.0 объединён в **Hazelcast Platform** — IMDG + Jet + CP subsystem.

Для real-time аналитики над данными в IMap.

Редко используется в типовом кэше scenario.

---

## 17. Проблемы

### 17.1 Split brain

Network partition → две части кластера думают "я целая".

Merge policies:
- **PassThroughMergePolicy** — winner берёт.
- **PutIfAbsentMergePolicy**.
- **HigherHitsMergePolicy**.

Настраивается per Map.

### 17.2 Rebalance при deploy

Rolling restart pods → каждый leave/join = rebalance data. Медленно для больших data.

Fix: graceful shutdown, wait for rebalance.

### 17.3 Long GC / freeze

Один node зависший на GC → cluster считает мёртвым → removes → rebalance.

Настраивать GC / heartbeat timeouts.

### 17.4 Serialization

Все объекты в IMap Serializable (или use custom Serializer).

Изменение класса → не совместимо со старыми data → deserialize errors.

Fix: **Portable serialization** от Hazelcast (schema-based), Compact serialization (5.0+).

---

## 18. Best practices

1. **Явно `backup-count`** — не полагаться на defaults.
2. **Явно eviction policy + max size** для Maps.
3. **TTL** для кэшей.
4. **Serializable / Portable** для value objects.
5. **Not multicast** в prod — TCP-IP list или K8s discovery.
6. **Мониторинг** через Management Center.
7. **Graceful shutdown** для избегания rebalance storms.
8. **HZ версия совместимая** с Spring Boot (см. кейс ИСНА).
9. **NearCache** для read-heavy Maps.
10. **CP subsystem** для strong consistency (locks, atomics).

---

## 19. Собесные вопросы

1. **Что такое Hazelcast?** — Java-native distributed in-memory data grid.
2. **Embedded vs client-server?** — Embedded: HZ внутри app; client-server: отдельный кластер.
3. **Что такое IMap?** — Distributed ConcurrentMap; данные распределены по partitions.
4. **Partitions в HZ?** — 271 по default; hash(key) % partitions → node.
5. **Что такое backup-count?** — Сколько replicas каждой partition.
6. **Что такое NearCache?** — Local cache на клиенте; ускоряет reads.
7. **Discovery способы?** — Multicast (dev), TCP-IP list, K8s, AWS, Consul.
8. **CP subsystem — что даёт?** — Strong consistency через Raft (для locks, atomics).
9. **Serializable в HZ?** — Все value objects; или Portable/Compact serialization.
10. **Персистентность?** — Volatile default; MapStore, Hot Restart (Ent), Persistence (5+).
11. **Как использовать как Hibernate 2LC?** — `hibernate-hazelcast` module + factory class.
12. **Разница ILock и Java Lock?** — ILock distributed, работает через кластер.
13. **Rebalance — что?** — Redistribution partitions при join/leave; может быть медленно.
14. **Split brain?** — Network partition; merge policies разруливают при восстановлении.
15. **HZ vs Redis?** — Java-native + embedded (см. `74-redis-vs-hazelcast.md`).

---

## Итог

- **Hazelcast** = Java-native distributed IMDG.
- **Peer-to-peer** — все node равноправны.
- **271 partitions** + backups.
- **Embedded** внутри Spring Boot или **client-server**.
- **IMap** = distributed ConcurrentMap.
- **Hibernate 2LC** — классический use case.
- **CP subsystem** для strong consistency.
- В **ИСНА**: активно используется; каветы миграции HZ 3 → HZ 5 (memory про actuator conflict).

Следующий — `74-redis-vs-hazelcast.md`.
