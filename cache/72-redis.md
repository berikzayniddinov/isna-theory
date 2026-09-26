# 72. Redis

In-memory key-value store. Стандарт де-факто для кэша.

---

## 1. Что такое Redis

**Redis** (**RE**mote **DI**ctionary **S**erver) — in-memory KV store + структуры данных.

- Написан на **C** (быстрый, малое потребление).
- **Single-threaded** для команд (простота, нет locks).
- **In-memory** — всё в RAM, persistence опционально.
- **Datatype-aware** — не только байты, а structures (list, set, hash).
- **Sub-millisecond latency** типично.
- Open source (BSD → SSPL с 2024).

Создан Salvatore Sanfilippo (antirez) в 2009.

---

## 2. Основные варианты использования

- **Cache** — самый частый.
- **Session store** — HTTP-сессии для web-app.
- **Rate limiting** — счётчики.
- **Distributed lock** — Redlock.
- **Pub/Sub** — простой messaging.
- **Streams** — event log (аналог Kafka).
- **Leaderboards** — sorted sets.
- **Counters, real-time analytics**.
- **Geospatial** — near-me queries.

---

## 3. Datatypes

Redis — не просто `key → string`. Богатый набор структур.

### 3.1 String

Простейший тип — байтовая строка (до 512 MB).

```
SET user:1 "Berik"
GET user:1                    → "Berik"

SET counter 0
INCR counter                  → 1
INCR counter                  → 2
DECR counter                  → 1

APPEND user:1 " Chotam"       → "Berik Chotam"
STRLEN user:1                 → 12

SETEX cache:key 60 "value"    → set + TTL 60 sec
```

Использование: cache, counters, TTL objects.

### 3.2 List

Двусторонняя очередь (linked list или ziplist).

```
LPUSH tasks "task1"           → left push
RPUSH tasks "task2"           → right push
LRANGE tasks 0 -1             → ["task1", "task2"]

LPOP tasks                    → "task1"
RPOP tasks                    → "task2"

BLPOP tasks 5                 → blocking pop (5 sec timeout)
```

Использование: очереди, timelines, recent history.

### 3.3 Hash

Map field → value.

```
HSET user:1 name "Berik" age 30 city "Almaty"
HGET user:1 name              → "Berik"
HGETALL user:1                → {name: "Berik", age: "30", city: "Almaty"}
HINCRBY user:1 age 1          → 31
```

Использование: **объекты** — компактнее чем много отдельных String keys.

### 3.4 Set

Неупорядоченное множество уникальных элементов.

```
SADD tags "java" "spring" "hibernate"
SMEMBERS tags                 → {"java", "spring", "hibernate"}
SISMEMBER tags "java"         → 1

SINTER tags1 tags2            → intersection
SUNION tags1 tags2            → union
SDIFF tags1 tags2             → difference
```

Использование: unique visitors, tags, permissions.

### 3.5 Sorted Set (ZSet)

Set + каждому элементу — score. Отсортировано по score.

```
ZADD leaderboard 1000 "alice" 2000 "bob" 500 "berik"
ZRANGE leaderboard 0 -1 WITHSCORES
    → berik 500, alice 1000, bob 2000

ZRANGEBYSCORE leaderboard 500 1500
ZINCRBY leaderboard 100 "berik"
```

Использование: leaderboards, priority queues, time-series.

### 3.6 Bitmap

Битовое представление.

```
SETBIT online:2026-09-07 42 1     → user 42 online
GETBIT online:2026-09-07 42       → 1
BITCOUNT online:2026-09-07        → сколько online
BITOP AND result day1 day2         → intersection
```

Использование: presence, feature flags per user, bloom filters (grubi).

### 3.7 HyperLogLog

Approximate unique count. **12 KB** для миллионов уникальных.

```
PFADD unique-visitors "user1" "user2" "user3"
PFCOUNT unique-visitors           → ~3
PFMERGE total site1 site2
```

Не 100% точно (~0.81% error), но memory-эффективно.

### 3.8 Geospatial

Координаты + радиус queries.

```
GEOADD places 76.9126 43.2565 "Almaty"
GEODIST places "Almaty" "Astana" km
GEORADIUS places 76.9 43.2 100 km
```

Использование: nearby search.

### 3.9 Streams

Append-only log (аналог Kafka).

```
XADD orders * customer c1 amount 100
XREAD COUNT 10 STREAMS orders 0
XGROUP CREATE orders processor $
XREADGROUP GROUP processor consumer1 COUNT 10 STREAMS orders >
```

Consumer groups как в Kafka. Persistence, replay.

Используется как lightweight Kafka.

---

## 4. TTL и expiration

Ключи могут иметь expiration:
```
SET session:abc "user1" EX 3600     → истечёт через 1 час
EXPIRE key 60                        → установить TTL
TTL key                              → сколько осталось
PERSIST key                          → убрать TTL
```

Redis удаляет expired keys:
- **Lazy** — при обращении.
- **Active** — фоновый sweep каждые 100 мс.

---

## 5. Eviction policies

Когда memory заполнена — что делать?

`maxmemory` — лимит.
`maxmemory-policy`:
- **noeviction** — reject writes.
- **allkeys-lru** — evict наименее используемое (LRU).
- **allkeys-lfu** — least frequently used.
- **volatile-lru** — только keys с TTL.
- **volatile-lfu**.
- **volatile-ttl** — с ближайшим TTL первыми.
- **allkeys-random**.
- **volatile-random**.

Для cache — **allkeys-lru** обычно.

---

## 6. Persistence

Redis **в памяти**, но может сохранять на диск.

### 6.1 RDB (Snapshot)

Периодический snapshot всей БД:
```
save 900 1        # каждые 15 мин если хоть 1 изменение
save 300 10       # каждые 5 мин если 10 изменений
save 60 10000     # каждую минуту если 10k изменений
```

Плюсы: **компактный** файл, быстрый restart.
Минусы: **потери** между snapshots (до 15 мин).

### 6.2 AOF (Append-Only File)

Каждая команда записывается в лог:
```
appendonly yes
appendfsync everysec        # fsync раз в секунду (compromise)
appendfsync always          # каждую команду (медленно, надёжно)
appendfsync no              # OS решает (небезопасно)
```

Плюсы: минимум потерь (1 сек).
Минусы: **большие файлы**, медленнее restart.

**Compaction (BGREWRITEAOF)** — переписать AOF, убрав избыточное.

### 6.3 Hybrid

Оба одновременно. RDB для быстрого restart + AOF для durability между snapshots.

### 6.4 Или ничего

Для чистого кэша persistence не нужен (потеря = переспросить у источника).

---

## 7. Replication

Master → replicas:
```
slaveof master-host 6379    # на replica
```

- **Async** replication (default) — быстро, но potential loss.
- Read scale (клиенты могут читать с replicas).
- **`WAIT` command** — sync-like (ждать N replicas).

### 7.1 Redis Sentinel

**Sentinel** — HA solution.

3+ Sentinel процесса следят за master. При падении master — автоматический failover:
1. Sentinel обнаруживают down.
2. Голосование за нового master.
3. Клиенты обновляются (Sentinel направляет).

Для HA но **не для scale** (данные всё ещё на одном master).

### 7.2 Redis Cluster

Sharding + replication.

- **16384 hash slots** распределены между master nodes.
- Каждый key → hash(key) % 16384 → slot → master.
- Каждый master имеет 1+ replicas.
- Автоматический failover.

Минимум: 3 master + 3 replicas.

**Hash tag** — форсировать одинаковый slot:
```
{user1}.orders     # hash по "user1"
{user1}.profile    # тоже "user1" → same slot
```

Для multi-key ops (transactions, pipelines, keys должны быть на одном node).

---

## 8. Transactions

**MULTI / EXEC**:
```
MULTI
SET user:1 "Berik"
INCR counter
EXPIRE user:1 3600
EXEC
```

Все команды выполняются atomically (single thread). Но:
- **Нет rollback** при ошибке одной команды.
- **Optimistic** через `WATCH`:

```
WATCH counter
val = GET counter
MULTI
SET counter (val + 1)
EXEC          # если counter изменился с момента WATCH → EXEC вернёт nil
```

Использование: distributed counters, реализация distributed lock.

---

## 9. Pipelining

Отправить batch команд без waiting:
```
Client → cmd1 → cmd2 → cmd3 → ... (не ждёт ответов)
Server ← reply1 ← reply2 ← reply3
```

Уменьшает round-trips. 1000 команд по одной = 1000 × RTT; pipelined = 1-2 RTT.

Аналог batching в JDBC (см. `69-tcp-batching-flushing.md`).

---

## 10. Lua scripts

Выполнить сложную логику атомарно на сервере:
```lua
-- rate-limit.lua
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
end
return current
```

```
EVAL script 1 rate:user:42 60
```

Плюсы:
- Atomicity (одним script).
- Меньше round-trips.
- Custom логика.

Использование: rate limit, distributed lock, batch operations.

---

## 11. Pub/Sub

Простой messaging:
```
Publisher: PUBLISH channel:orders "new order 42"

Subscriber: SUBSCRIBE channel:orders
    → получит сообщение
```

Нет persistence — если subscriber offline, сообщение потерялось.

Для durable messaging — Redis Streams или Kafka.

---

## 12. Distributed lock (Redlock)

```lua
-- try acquire
SET lock:resource1 "unique-token" NX EX 30

-- released
if get == token then delete
```

**Redlock algorithm** — распределённый lock через несколько Redis nodes для reliability.

Использование в Java: **Redisson** (см. §16).

---

## 13. Spring Data Redis

### 13.1 Зависимости

```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

По default — **Lettuce** client (netty, async). Альтернатива — Jedis.

### 13.2 Конфигурация

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD}
      timeout: 2s
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0
```

### 13.3 RedisTemplate

```java
@Autowired RedisTemplate<String, Object> redisTemplate;

// String
redisTemplate.opsForValue().set("key", "value");
redisTemplate.opsForValue().set("key", "value", Duration.ofSeconds(60));
Object v = redisTemplate.opsForValue().get("key");

// Hash
redisTemplate.opsForHash().put("user:1", "name", "Berik");
Map<Object,Object> all = redisTemplate.opsForHash().entries("user:1");

// List
redisTemplate.opsForList().leftPush("queue", "task");
Object t = redisTemplate.opsForList().rightPop("queue");

// Set / ZSet — аналогично
```

### 13.4 @Cacheable

```gradle
implementation 'org.springframework.boot:spring-boot-starter-cache'
```

```java
@EnableCaching
class Config {
    @Bean
    CacheManager cacheManager(RedisConnectionFactory cf) {
        return RedisCacheManager.builder(cf)
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10)))
            .build();
    }
}

@Service
class UserService {
    @Cacheable("users")
    public User getById(Long id) {
        return userRepo.findById(id).orElseThrow();
    }

    @CacheEvict(value = "users", key = "#user.id")
    public void update(User user) { ... }
}
```

Прозрачно кэширует результаты в Redis.

### 13.5 Redis как Session store

```gradle
implementation 'org.springframework.session:spring-session-data-redis'
```

```java
@EnableRedisHttpSession
class Config { }
```

HTTP-сессии в Redis → sticky не нужен, cluster-friendly.

---

## 14. Redisson

Высокоуровневый Java client для Redis.

```gradle
implementation 'org.redisson:redisson-spring-boot-starter:3.24.0'
```

Даёт distributed Java-объекты:
```java
@Autowired RedissonClient redisson;

// Distributed lock
RLock lock = redisson.getLock("my-lock");
lock.lock();
try { ... } finally { lock.unlock(); }

// Distributed collections
RMap<String, User> map = redisson.getMap("users");
RQueue<String> queue = redisson.getQueue("tasks");
RScheduledExecutorService executor = redisson.getExecutorService("exec");

// Rate limiter
RRateLimiter limiter = redisson.getRateLimiter("api");
limiter.trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.SECONDS);
```

Плюсы: Java-friendly abstractions, distributed structures.

---

## 15. Monitoring

- **`INFO`** — статистика.
- **`MONITOR`** — все команды real-time.
- **`SLOWLOG`** — медленные команды.
- **RedisInsight** — GUI.
- **redis-cli --stat** — live stats.
- **redis_exporter** для Prometheus.

Метрики:
- `used_memory` — RAM.
- `total_commands_processed`.
- `instantaneous_ops_per_sec`.
- `hit_rate` (для cache).
- `evicted_keys`.
- `expired_keys`.
- `connected_clients`.

---

## 16. Best practices

1. **Явно maxmemory + eviction policy** для кэша.
2. **Правильные datatypes** — Hash вместо кучи String'ов.
3. **TTL для всего** в кэше.
4. **Keys naming** convention: `user:1:profile`.
5. **Pipelining** для батчей.
6. **Redis Cluster** для scale + Sentinel для HA (или Cluster делает оба).
7. **Persistence по необходимости** — часто не нужна для чистого кэша.
8. **Не хранить огромные values** (>1 MB) — блокирует single thread.
9. **Не используй `KEYS *`** в проде (О(N)); используй `SCAN`.
10. **Мониторинг** hit rate + memory.

---

## 17. Собесные вопросы

1. **Что такое Redis?** — In-memory KV store с богатыми datatypes.
2. **Redis single-threaded — как быстрый?** — Нет context switch, нет locks, in-memory operations очень быстрые.
3. **Datatypes Redis?** — String, List, Hash, Set, ZSet, Bitmap, HyperLogLog, Geospatial, Stream.
4. **TTL — что делает?** — Automatic expiration; lazy + active sweep.
5. **Eviction policies?** — noeviction, allkeys-lru, allkeys-lfu, volatile-lru, volatile-ttl, random.
6. **Persistence в Redis?** — RDB (snapshot) + AOF (append-only log); hybrid.
7. **Разница Redis Sentinel и Cluster?** — Sentinel: HA (failover master); Cluster: HA + sharding.
8. **Что такое hash slots?** — 16384 slots в Cluster; hash(key) % 16384 → slot → master.
9. **Что такое hash tag?** — `{tag}.key` форсирует одинаковый slot (для multi-key ops).
10. **Transactions в Redis?** — MULTI/EXEC atomically, но no rollback; WATCH для optimistic.
11. **Что такое pipelining?** — Batch команд без waiting per-command reply.
12. **Что такое Redis Streams?** — Append-only log с consumer groups; lightweight Kafka alternative.
13. **Pub/Sub — durable?** — Нет; если subscriber offline — потеря; для durable — Streams.
14. **Distributed lock в Redis?** — SET NX EX + unique token; Redlock для nodes; Redisson для Java.
15. **Что такое Redisson?** — Java-native client с distributed objects (locks, maps, queues).
16. **Как использовать в Spring?** — Spring Data Redis (RedisTemplate, @Cacheable), Spring Session.
17. **KEYS * — почему опасно?** — O(N) блокирует single thread; используй SCAN.
18. **Redis vs Memcached?** — Redis богатые datatypes + persistence; Memcached только simple KV.
19. **Как мониторить?** — INFO, SLOWLOG, redis_exporter → Prometheus, RedisInsight GUI.
20. **Lua scripts — зачем?** — Atomicity + fewer round-trips; custom логика на сервере.

---

## Итог

- **Redis** = in-memory KV + богатые structures.
- **Sub-millisecond latency** типично.
- **9 datatypes** — не только key-value.
- **Persistence** RDB/AOF (опционально).
- **Sentinel / Cluster** для HA / scale.
- **TTL + eviction policy** обязательно для кэша.
- **Spring Data Redis** + **Redisson** для Java.
- Use cases: **cache, session, rate limit, distributed lock, leaderboard**.

Следующий — `73-hazelcast.md`.
