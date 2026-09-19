# 23. RabbitMQ: паттерны и troubleshooting в проде

Кластеризация, паттерны использования, мониторинг, частые проблемы.

---

## 1. Паттерны использования

### 1.1 Work queue

**Одна очередь, много consumer'ов**. Классика.

```
publisher → work.queue → [consumer1, consumer2, consumer3, ...]
```

Broker раскидывает по round-robin (с учётом prefetch и ack).

Использование: распределённая обработка задач. Например, распечатать 1000 отчётов — 10 consumer'ов быстро закроют.

### 1.2 Pub-Sub (fanout)

**Один exchange fanout → много очередей → много подписчиков**.

```
publisher → fanout.exchange → [queue1, queue2, queue3]
                                  ↓        ↓        ↓
                              consumer1 consumer2 consumer3
```

Использование: broadcast событий, инвалидация кэша.

### 1.3 Topic routing

**Fanout по паттерну**.

```
publisher → topic.exchange
                │
   ├─ "orders.kz.new"  → kz-orders queue
   ├─ "orders.uz.new"  → uz-orders queue
   └─ "orders.*.new"   → all-orders queue
```

Использование: routing по бизнес-контексту.

### 1.4 RPC (request/reply)

Синхронное общение через Rabbit. Producer публикует запрос с `reply-to` = его temporary queue, ждёт ответ.

```
client → request.queue → server
   ↑                        │
   └── reply.queue (temp) ──┘
```

**Плохо** для микросервисов — сложнее HTTP. Использовать редко.

### 1.5 Priority queue

Очередь с приоритетами:
```
x-max-priority: 10
```

Producer ставит priority (0-10). Broker отдаёт с высшим priority первым.

Использование: срочные сообщения перегоняют обычные.

### 1.6 Delayed / scheduled messages

Плагин `rabbitmq-delayed-message-exchange`:
```java
props.setHeader("x-delay", 60000);   // отправить через 60 сек
```

Или через TTL + DLX «дедаловский» трюк:
1. Отправить в holding-queue с TTL=60000.
2. TTL истёк → сообщение → DLX → целевая queue.

Использование: retry backoff, «напомнить через час».

### 1.7 Saga / choreography

Каждый сервис реагирует на события, публикует свои. Нет единого дирижёра.

```
OrderCreated → InventoryService → InventoryReserved →
    PaymentService → PaymentReceived → OrderConfirmed
```

Если что-то падает — компенсирующие события (`InventoryReleased`, `PaymentRefunded`).

Сложно моделировать и отлаживать, но масштабируется.

---

## 2. Кластеризация

### 2.1 Classic mirroring (deprecated)

Классический кластер: очереди зеркалируются между узлами. Master + slaves. При падении master'а — один из slaves становится новым master'ом.

Проблемы:
- Split-brain при сетевом разделении.
- Медленно (репликация синхронная).
- Deprecated в новых версиях RabbitMQ.

### 2.2 Quorum queues

**Современный подход** (RabbitMQ 3.8+). Основан на **Raft** — тот же алгоритм что у etcd, Consul.

- Обычно 3 или 5 replicas.
- Кворум записывает и подтверждает.
- Automatic failover.
- Нет split-brain.

```java
QueueBuilder.durable("my.q")
    .quorum()
    .build();
```

**Правило**: для новых очередей — quorum. Для важных данных — обязательно.

### 2.3 Federation и Shovel

Для multi-datacenter или asymmetric топологии:
- **Federation** — «мосты» между кластерами для обмена сообщениями.
- **Shovel** — перекачка из очереди A на кластере 1 в очередь B на кластере 2.

Используется для DR (disaster recovery), геораспределённых систем.

---

## 3. Мониторинг

### 3.1 Что мониторить

- **Queue depth** (`messages_ready`) — сколько ждёт обработки. Рост = отставание consumer'а.
- **Consumer count** — сколько consumer'ов на очереди. 0 → alarm.
- **Publish rate / Consume rate** — балансировка.
- **Redeliver rate** — сколько повторных доставок. Рост → проблемы обработки.
- **Connection count, Channel count**.
- **Memory usage broker'а** — приближается к `vm_memory_high_watermark`.
- **Disk free** — memory alarm.
- **Network partitions** — split-brain кластера.

### 3.2 Management UI

`http://rabbit:15672/`. Показывает всё в реальном времени.

### 3.3 Prometheus

Плагин `rabbitmq_prometheus`:
```
http://rabbit:15672/metrics
```

Grafana dashboard с queue depth, rates, resource usage.

### 3.4 Alerting

- Queue depth > N (например 10000) → alert.
- No consumers → alert.
- Memory alarm active → critical.
- DLQ growing → team review.

---

## 4. Частые проблемы

### 4.1 «Сообщения теряются»

Проверка:
1. Queue **durable=true**?
2. Message **persistent** (`deliveryMode=2`)?
3. Consumer **manual ack** и ack вызывается **после** обработки?
4. **Publisher confirms** включены?
5. **Mandatory + returns** для отлова unroutable?

Обычно проблема — одно из этого пункта не выполнено.

### 4.2 «Consumer стоит, ничего не берёт»

- Consumer подписан? Смотри `basic.consume-ok`.
- Prefetch не 0 (случайно поставили 0 = broker не шлёт).
- Все сообщения in-flight (unacked) — увеличь prefetch или разберись почему не ack'аются.
- Connection упал — `spring-rabbit` авто-переподключит, но иногда молча.
- `noAck` = auto с ошибкой = сообщения теряются молча.

### 4.3 «Broker медленный»

- Memory alarm — очереди в RAM переполнили. Пересмотри lazy queues, увеличь `vm_memory_high_watermark`.
- Disk alarm — журнал WAL растёт.
- Много каналов на connection — `channel_max` лимит.
- Много connections — `connection_max` лимит.
- Flow control — producer'ы замедлены.

### 4.4 Redelivery loop

Сообщение обрабатывается → падает → requeue → снова падает → ...

Причины:
- Retry без max-attempts.
- `default-requeue-rejected: true`.
- Deserialization error (не сможешь распарсить никогда).

Лечение:
- `max-attempts` + `RejectAndDontRequeueRecoverer` → в DLX.
- `default-requeue-rejected: false`.

### 4.5 Producer «отправил», но не пришло

- Publisher confirms НЕ включены → ты не знаешь дошло ли.
- Exchange не существует / другое имя.
- Routing key не сматчился → сообщение выброшено (mandatory спасает).
- Consumer вообще не подписан (можно проверить в UI).

### 4.6 Дубли сообщений

- Consumer не ack'нул → broker re-delivered.
- Publisher retry без idempotency ключа.

Всегда идемпотентный consumer.

### 4.7 Порядок нарушен

- Concurrent consumers → нет гарантии порядка.
- Nack requeue возвращает в head.
- Sharding нужен по бизнес-ключу.

Если порядок важен → один consumer, `prefetch=1`, никаких requeue.

---

## 5. Backpressure

Что делать когда producer быстрее consumer'а?

### 5.1 Queue растёт

Варианты:
1. **Больше consumer'ов** — scale.
2. **Batch** — обрабатывать пачками.
3. **Оптимизация consumer'а** — профайлинг.
4. **Дропать старое** — TTL, max-length.
5. **Замедлить producer** — publisher rate limit.

### 5.2 max-length

```
x-max-length: 100000
x-overflow: drop-head    # или reject-publish (не принимать новые)
```

При переполнении — старое дропается (или в DLX).

### 5.3 max-length-bytes

Ограничение по размеру:
```
x-max-length-bytes: 1000000000   # 1 GB
```

---

## 6. Безопасность

- **Users + passwords** — не guest/guest в проде.
- **Permissions per vhost** — user только к нужному vhost.
- **TLS** — encrypted connections (порт 5671).
- **Sensitive data** — не в message body или зашифрованы (при недоверенном сегменте).

---

## 7. HA-хитрости

### 7.1 Idempotent consumer

Обязательно:
```java
if (processedRepo.existsByMessageId(msgId)) return;
process(...);
processedRepo.save(msgId);
```

Таблица processed_messages с индексом на message_id + TTL cleanup.

### 7.2 Transactional outbox

Проблема: как атомарно сохранить в БД + опубликовать в Rabbit? Если между ними падение → inconsistency.

**Outbox**:
1. В транзакции: сохранить бизнес-данные + запись в `outbox` таблицу.
2. Отдельный job читает outbox → публикует в Rabbit → удаляет из outbox.
3. Гарантия: если БД коммит прошёл — outbox запись есть → рано или поздно опубликуется.

### 7.3 Consumer graceful shutdown

При SIGTERM:
1. Прекратить принимать новые.
2. Дообрабатывать текущие.
3. Ack всё завершённое.
4. Close channel/connection.

Spring AMQP делает это через `SmartLifecycle`. Проверяй что shutdown-timeout достаточен.

---

## 8. Реальные кейсы из ИСНА

### 8.1 `knp-fo-sync-notification-bugs`

6 багов NotificationSyncService:
- processedDate-мкс молча теряется.
- `@Transactional` мёртв (self-invocation) → LazyInit.
- Тихий скип на ошибке (MAX offset).
- `printStackTrace` → ELK-слепая зона.
- Контекст-на-цикл (утечка).
- periodValue=0.

Фикс: MR !369 с явными транзакциями, structured logging, ретрай.

### 8.2 `knp-fno-outer-sync-esb-dead-route`

Sync-сервис для OUTER_FNO_INFO падал 1586×/13ч SOAP «маршрут не поднят». Не задача Rabbit, но урок: **retry на не-фиксимую проблему = шум и лишние потери**. Правильно — circuit breaker + флаг `@ConditionalOnProperty` для отключения.

### 8.3 Уроки

1. Все consumer'ы **идемпотентные** и **логирующие**.
2. `printStackTrace` = зло. Structured `log.error(msg, e)`.
3. Явные транзакции без self-invocation.
4. DLQ + alerting на его рост.
5. Метрики per очередь: rate ack/nack/redeliver.

---

## 9. Собесные вопросы

1. **Паттерны использования Rabbit?** — Work queue, pub-sub, topic routing, RPC, priority, delayed, saga.
2. **Разница quorum и classic queues?** — Quorum на Raft (Boot-in HA); classic — mirroring (deprecated).
3. **Что мониторить в проде?** — Queue depth, consumer count, publish/consume rate, redeliver rate, memory/disk broker'а.
4. **Что такое memory alarm?** — Broker останавливает producer'ов когда RAM > watermark.
5. **Как избежать redelivery loop?** — max-attempts + RejectAndDontRequeueRecoverer + defaultRequeueRejected=false.
6. **Как обеспечить порядок в Rabbit?** — Один consumer, prefetch=1, no requeue (или sharding по ключу).
7. **Transactional outbox — зачем?** — Атомарность БД-commit + publish; job читает outbox → публикует.
8. **Backpressure — что делать?** — Scale consumers / batch / TTL / max-length / rate limit producer.
9. **Federation vs Shovel?** — Federation — «мост» между кластерами; shovel — перекачка queue A → queue B.
10. **Как поступить с «плохим» сообщением?** — nack requeue=false → DLX → DLQ → alert → ручной разбор.

---

## Итог

- **Паттерны**: work queue, pub-sub, topic, RPC (редко), saga.
- **Quorum queues** для новых очередей.
- **Мониторинг**: queue depth, rates, DLQ growth, memory/disk.
- **Backpressure**: scale, batch, TTL, max-length.
- **Идемпотентность consumer'а** = обязательное свойство.
- **Transactional outbox** — атомарность БД + Rabbit.
- **Никогда** `printStackTrace`, `default-requeue-rejected: true`, `guest/guest`.

---

## Итог блока RabbitMQ

- 20 — AMQP основы (exchange, queue, binding, routing).
- 21 — гарантии доставки (ack, confirms, DLX, retry, идемпотентность).
- 22 — Spring AMQP (RabbitTemplate, @RabbitListener, prod-конфиг).
- 23 — паттерны и troubleshooting (кластеры, monitoring, кейсы).

Следующий блок — Spring Security + Keycloak (24-27). Начинаю с `24-spring-security-basics.md`.
