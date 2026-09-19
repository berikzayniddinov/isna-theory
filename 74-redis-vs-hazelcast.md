# 74. Redis vs Hazelcast — разница

Развёрнутое сравнение. Когда что выбрать.

---

## 1. Быстрый обзор

**Redis** — external process, любой язык клиент, богатые datatypes.

**Hazelcast** — Java-native, может быть embedded, distributed collections в стиле Java.

Оба — in-memory storage / cache. Разные подходы.

---

## 2. Таблица сравнения

| | Redis | Hazelcast |
|---|---|---|
| Язык реализации | C | Java |
| Основная модель | Client-server | Peer-to-peer |
| Embedded | Нет | Да (внутри JVM) |
| Threading | Single-threaded (per shard) | Multi-threaded |
| Datatypes | 9 built-in (string, list, hash, set, zset, stream, ...) | Distributed Java collections (Map, Set, Queue, List, MultiMap) |
| Persistence | RDB + AOF (мощно) | Volatile default; Hot Restart (Enterprise), MapStore |
| Consistency | Eventually consistent | Strong (CP subsystem) + Eventual (default) |
| Sharding | Redis Cluster (16384 slots) | Automatic (271 partitions) |
| Replication | Async master-replica | Sync/async backup partitions |
| HA | Sentinel (failover master) | Automatic (backup partitions) |
| Auto-discovery | Manual или Sentinel | Multicast, TCP, K8s, Consul, AWS |
| Pub/Sub | Yes | Yes (ITopic) |
| Streams | Yes (Redis Streams) | Reliable Topic + Jet streaming |
| Transactions | MULTI/EXEC + WATCH | Distributed tx (2PC) |
| Distributed lock | Redlock (сложный) | ILock, FencedLock (проще для Java) |
| Latency | Sub-millisecond (network) | Microseconds (embedded), sub-ms (client) |
| Throughput | 100k+ ops/sec per instance | 100k+ ops/sec per node |
| Memory usage | Малое | Больше (JVM overhead) |
| Ecosystem | Огромный (все языки) | Java-centric |
| Community | Гораздо больше | Средне |
| Cloud managed | AWS ElastiCache, Redis Cloud, Azure Cache | AWS, Azure, GCP, Hazelcast Cloud |
| License | BSD → SSPL (2024), Valkey fork | Apache 2.0 (Enterprise отдельно) |

---

## 3. Архитектура

### 3.1 Redis

**External processes**. Приложение — client.

```
┌─── App 1 ──┐  ┌─── App 2 ──┐  ┌─── App 3 ──┐
│   Client   │  │   Client   │  │   Client   │
└──────┬─────┘  └──────┬─────┘  └──────┬─────┘
       └───────────────┼───────────────┘
                       │
                       ▼
                 ┌─────────┐
                 │  Redis  │  (Cluster: N shards)
                 │         │
                 └─────────┘
```

Простая модель. Redis — отдельный сервер.

### 3.2 Hazelcast (embedded)

Приложения **сами являются** node кластера.

```
┌─── App 1 (HZ node) ──┐  ┌─── App 2 (HZ node) ──┐  ┌─── App 3 (HZ node) ──┐
│   IMap data          │◄►│   IMap data          │◄►│   IMap data          │
│   partitions 0-90    │  │   partitions 91-180  │  │   partitions 181-270 │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘
```

Данные **в JVM** приложений. Нет network hop для local partition.

### 3.3 Hazelcast (client-server)

Отдельный кластер:
```
┌─── App 1 ──┐  ┌─── App 2 ──┐
│   Client   │  │   Client   │
└──────┬─────┘  └──────┬─────┘
       └───────┼───────┘
               │
     ┌─────────┼─────────┐
     ▼         ▼         ▼
┌───HZ 1──┐ ┌HZ 2─┐ ┌HZ 3─┐
└─────────┘ └─────┘ └─────┘
```

Похоже на Redis.

---

## 4. Datatypes

### 4.1 Redis — богатые structures

- **String** — с increment, TTL.
- **List** — LPUSH/RPOP.
- **Hash** — map field→value.
- **Set** — union/intersect/diff.
- **Sorted set** — leaderboards.
- **Bitmap, HyperLogLog, Geo, Streams**.

Каждая структура — специфичные команды.

### 4.2 Hazelcast — Java collections

- **IMap** — ConcurrentMap.
- **IList** — List.
- **ISet** — Set.
- **IQueue** — BlockingQueue.
- **MultiMap** — Map<K, Collection<V>>.
- **ReplicatedMap** — replicated (не partitioned).
- **ITopic** — pub/sub.

Прозрачно как Java-объекты (`.put()`, `.get()`).

**Плюс Hazelcast**: Java-программисту привычно.
**Плюс Redis**: специализированные commands (sorted set для leaderboard проще).

---

## 5. Performance

### 5.1 Latency

- **Redis**: sub-millisecond per operation (network round-trip).
- **Hazelcast embedded**: microseconds (local partition, in JVM).
- **Hazelcast client-server**: sub-millisecond.

**Hazelcast embedded быстрее** для local reads (без network).
**Redis выигрывает** на shared cluster сценариях.

### 5.2 Throughput

Оба могут делать 100k+ ops/sec per node.

Redis single-threaded — high per-core efficiency.
Hazelcast multi-threaded — использует все cores.

### 5.3 Memory

- **Redis**: C, компактно.
- **Hazelcast**: Java, много overhead (object headers, references).

Одинаковый dataset — Redis требует **меньше памяти** в 2-3×.

---

## 6. Consistency

### 6.1 Redis

- **Eventually consistent** (master → replica async).
- **Strong** можно через `WAIT` (sync-like).
- **Redis Cluster** — не поддерживает transactions across shards.

### 6.2 Hazelcast

- **Eventually consistent** default (backup async).
- **Strong** можно через `sync-backups`.
- **CP subsystem** — Raft-based для strong consistency (FencedLock, IAtomicLong).

Hazelcast имеет **и strong, и eventual** — выбор per-usecase.

---

## 7. Distributed lock

### 7.1 Redis

**Redlock algorithm** — сложный, требует несколько Redis nodes:
- Acquire на N/2+1 nodes → lock held.
- TTL для auto-release.

**Redisson** в Java упрощает:
```java
RLock lock = redisson.getLock("myLock");
lock.lock();
try { ... } finally { lock.unlock(); }
```

### 7.2 Hazelcast

Простой Java API:
```java
FencedLock lock = hz.getCPSubsystem().getLock("myLock");
lock.lock();
try { ... } finally { lock.unlock(); }
```

`FencedLock` через CP subsystem (Raft) — сильные гарантии.

**Hazelcast чище для Java**. Redis + Redisson тоже работает, но больше layers.

---

## 8. Persistence

### 8.1 Redis

Мощная:
- **RDB** — snapshot файл.
- **AOF** — append-only log.
- **Hybrid** — оба.

Для durable data.

### 8.2 Hazelcast

Слабее:
- **Volatile** default.
- **MapStore** — write-through в БД.
- **Hot Restart** (Enterprise) — snapshot.
- **Persistence** (5.0+) — WAL для CP.

Обычно Hazelcast используется как **чистый cache** без persistence.

**Redis лучше** для durable KV.

---

## 9. Ecosystem

### 9.1 Redis

- Client libraries **на всех языках** (Python, JS, Go, Ruby, PHP, C#, Java, Rust, ...).
- Documentation, tutorials, community огромны.
- Cloud managed: AWS ElastiCache, Redis Cloud, Azure Cache, Google Memorystore.
- Redis Modules (RediSearch, RedisJSON, RedisGraph, RedisTimeSeries).

### 9.2 Hazelcast

- **Java-центричный**. Есть C++, C#, Python clients, но менее развитые.
- Documentation хорошая, но меньше community.
- Cloud: Hazelcast Cloud, AWS/Azure/GCP marketplace.
- Hazelcast Jet для streaming.

**Redis имеет неоспоримое преимущество в ecosystem**.

---

## 10. Управление / operations

### 10.1 Redis

- **Redis Sentinel** — сложный setup для HA.
- **Redis Cluster** — сложный setup + client-side sharding awareness.
- **Rebalance manual** после resize.
- **Monitoring**: INFO, MONITOR, redis_exporter.

### 10.2 Hazelcast

- **Auto-discovery** — легче setup.
- **Auto-rebalance** on join/leave.
- **Automatic backup**.
- **Management Center** — UI.

**Hazelcast проще operationally** — meets less DevOps overhead.

---

## 11. Когда что выбрать

### 11.1 Redis выигрывает

- **Polyglot** (не только Java).
- **Богатые structures нужны** (leaderboards, streams).
- **Persistence важна** (durable KV, session store).
- **Огромный community / ecosystem** нужен.
- **Cloud managed** (ElastiCache).
- **Rate limiting** (INCR + EXPIRE идеально).
- **Pub/Sub** простой.
- **Small memory footprint**.

### 11.2 Hazelcast выигрывает

- **Java-centric** приложение.
- **Embedded** cache (нет сети).
- **Distributed computing** (Executor, EntryProcessor).
- **Strong consistency** нужна (CP subsystem).
- **Hibernate 2LC** — стандарт де-факто.
- **Auto-discovery** приоритетна.
- **Простая operational модель**.

### 11.3 Не имеет значения

- Простой KV cache — оба работают одинаково.

---

## 12. Пример: cache в Spring Boot

### 12.1 Redis

```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
implementation 'org.springframework.boot:spring-boot-starter-cache'
```

```java
@EnableCaching
class Config {
    @Bean
    CacheManager cacheManager(RedisConnectionFactory cf) {
        return RedisCacheManager.builder(cf).build();
    }
}
```

### 12.2 Hazelcast

```gradle
implementation 'com.hazelcast:hazelcast-spring'
implementation 'org.springframework.boot:spring-boot-starter-cache'
```

```java
@EnableCaching
class Config {
    @Bean
    HazelcastInstance hz() { return Hazelcast.newHazelcastInstance(); }

    @Bean
    CacheManager cacheManager(HazelcastInstance hz) {
        return new HazelcastCacheManager(hz);
    }
}
```

Оба работают через **`@Cacheable`** аннотации — прозрачно для приложения.

---

## 13. В ИСНА

### 13.1 Используется Hazelcast

Из memory:
- **`knp-form-hz5-actuator-cache-nosuchmethod`** — миграция HZ 3.12 → 5.3.8, каветы actuator.
- **`taxrep21-hazelcast-kesh66-unreachable`** — Hazelcast client в tax-report21.
- **`knp-form-arm-hazelcast3-not-migrated-flap`** — arm форма на HZ3.

Активно как distributed cache между JVM'ами.

### 13.2 Redis

Не в основных memory-заметках ИСНА. Возможно используется точечно, но не как central cache.

**Причина выбора Hazelcast** в ИСНА:
- Java-стек (Spring Boot микросервисы).
- Hibernate 2LC.
- Embedded вариант.
- Legacy изначально.

Миграция на Redis — было бы огромное усилие, и не даёт value.

---

## 14. Как принять решение

Спрашиваешь:

1. **Мы Java-only?** → Hazelcast подходит.
2. **Мы polyglot?** → Redis.
3. **Нужна persistence KV?** → Redis.
4. **Нужны rich structures (leaderboards, streams)?** → Redis.
5. **Нужен embedded cache?** → Hazelcast.
6. **Нужна strong consistency + distributed lock?** → Hazelcast (проще) или Redis+Redisson.
7. **Hibernate 2LC?** → Hazelcast или Ehcache/Infinispan (Redis не поддерживается natively).
8. **Cloud managed приоритет?** → Redis (ElastiCache dominant).
9. **Team знает Redis?** → Redis.
10. **Team знает Hazelcast?** → Hazelcast.

---

## 15. Alternatives (кратко)

Другие in-memory / caching solutions:

- **Memcached** — только KV strings, простой, старый. Redis почти всегда лучше.
- **Ehcache** — Java library, standalone in-JVM cache. Простой.
- **Infinispan** — RedHat's answer to Hazelcast. Java-native.
- **Apache Ignite** — computing + storage grid.
- **Aerospike** — flash-optimized KV store.
- **KeyDB** — Redis fork, multi-threaded.
- **Valkey** — Redis fork after license change (Linux Foundation).
- **DragonflyDB** — modern Redis-compatible, C++.

---

## 16. Собесные вопросы

1. **Redis vs Hazelcast — главное отличие?** — Redis C external process, любой язык; Hazelcast Java-native, может embedded.
2. **Кто быстрее?** — Depends: HZ embedded для local ops быстрее; Redis выигрывает при shared cluster.
3. **Persistence?** — Redis мощный (RDB+AOF); HZ слабее (MapStore, Enterprise).
4. **Богатые structures?** — Redis больше (sorted sets, geo, streams); HZ — Java collections.
5. **Distributed lock?** — Redis: Redlock (Redisson library); HZ: FencedLock через CP subsystem.
6. **Hibernate 2LC?** — Hazelcast стандартный выбор; Redis не поддерживается natively.
7. **Embedded?** — Только Hazelcast.
8. **Ecosystem?** — Redis намного больше, все языки.
9. **Cloud managed?** — Redis dominant (ElastiCache и т.д.).
10. **Consistency?** — Оба eventual default; HZ имеет CP subsystem для strong.
11. **HA?** — Redis: Sentinel/Cluster; HZ: automatic backups.
12. **Memory efficiency?** — Redis экономичнее (C vs JVM overhead).
13. **Когда Redis?** — Polyglot, persistence, rich structures, cloud managed.
14. **Когда Hazelcast?** — Java-only, embedded, Hibernate 2LC, strong consistency.
15. **В ИСНА что?** — Hazelcast (Java-стек, Hibernate 2LC).

---

## Итог

- **Redis** — C, external, polyglot, богатые structures, persistence.
- **Hazelcast** — Java, embedded/client-server, distributed collections, Hibernate 2LC.
- **Choose Redis**: polyglot, cache with persistence, cloud managed, rich structures.
- **Choose Hazelcast**: Java-only, embedded, Hibernate cache, strong consistency нужна.
- **В ИСНА**: Hazelcast активно (кэши, Hibernate 2LC, миграция HZ3→HZ5 болезненная).

Следующий — `75-kalkan-ecp.md`.
