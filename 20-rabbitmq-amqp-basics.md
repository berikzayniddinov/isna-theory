# 20. RabbitMQ: AMQP основы

RabbitMQ — message broker. Задача — понять модель AMQP: exchange, queue, binding, routing.

---

## 1. Зачем нужен message broker

### 1.1 Проблема синхронного общения

Сервис A вызывает B через HTTP синхронно. Проблемы:
- B недоступен → A падает.
- B медленный → A ждёт.
- Пиковая нагрузка на A → B перегружен.
- Тесная связь.

### 1.2 Что даёт broker

- **Асинхронность** — A положил сообщение, забыл, B заберёт когда сможет.
- **Буферизация** — пик нагрузки поглощается очередью.
- **Развязка** — A не знает про B (только про очередь).
- **Гарантии доставки** — сообщение не потеряется даже если B недоступен.
- **Fan-out** — одно событие → много подписчиков.

Аналогия: A — курьер, B — получатель. Раньше: курьер ждёт пока получатель откроет. Теперь: положил в почтовый ящик, ушёл, получатель заберёт когда сможет.

---

## 2. Что такое AMQP

**AMQP (Advanced Message Queuing Protocol)** — открытый стандарт для messaging (не только Rabbit; есть QPid, ActiveMQ Artemis).

RabbitMQ — самая популярная реализация. Написан на Erlang. Умеет и другие протоколы (MQTT, STOMP), но базовая модель — AMQP 0.9.1.

### 2.1 Ключевые концепты

- **Producer** — публикует сообщения.
- **Consumer** — читает сообщения.
- **Broker** — сам RabbitMQ.
- **Exchange** — «маршрутизатор». Принимает сообщения от producer, решает куда положить.
- **Queue** — очередь. Хранит сообщения до чтения consumer'ом.
- **Binding** — правило «этот exchange → эта queue по такому-то routing key».
- **Routing key** — метаданные сообщения, exchange использует для маршрутизации.
- **Virtual host (vhost)** — логическое разделение brokerа (аналог namespace).

---

## 3. Основная модель

```
                     ┌──────────────┐
Producer  ──publish──►│   Exchange   │
   (routing key)     └──────┬───────┘
                            │
                    ┌───────┴───────┐
                    │  binding by   │
                    │  routing key  │
                    │               │
                    ▼               ▼
                ┌───────┐      ┌───────┐
                │Queue 1│      │Queue 2│
                └───┬───┘      └───┬───┘
                    │              │
                    ▼              ▼
               Consumer 1    Consumer 2
```

Producer НЕ пишет в очередь напрямую. Пишет в exchange + routing key. Exchange по своим bindings решает в какие очереди положить.

---

## 4. Типы Exchange

### 4.1 Direct

Routing по точному совпадению routing key.

```
Exchange (type=direct)
    │
    ├── binding "orders.new"    → Queue A
    ├── binding "orders.paid"   → Queue B
    └── binding "orders.new"    → Queue C
```

Publish с routing="orders.new" → Queue A + Queue C.
Publish с routing="orders.paid" → Queue B.

Использование: точный routing, work queues.

### 4.2 Topic

Routing по pattern-matching с wildcards.

- `*` — одно слово.
- `#` — ноль или более слов.

```
Exchange (type=topic)
    │
    ├── binding "orders.*.new"      → Queue A
    ├── binding "orders.kz.#"       → Queue B
    └── binding "#.new"             → Queue C
```

Publish `orders.kz.new`:
- matches `orders.*.new` → A
- matches `orders.kz.#` → B
- matches `#.new` → C

Publish `orders.kz.paid`:
- matches `orders.kz.#` → B

Использование: pub-sub по темам.

### 4.3 Fanout

Игнорирует routing key. Все bindings получают всё.

```
Exchange (type=fanout)
    │
    ├── binding → Queue A
    ├── binding → Queue B
    └── binding → Queue C
```

Каждое сообщение → во все три очереди.

Использование: broadcast (уведомления, инвалидация кэша).

### 4.4 Headers

Routing по заголовкам сообщения (не по routing key). Редко используется.

### 4.5 Default exchange

У каждой очереди автоматически есть binding в **default exchange** (direct, `""`) с routing key = имя очереди. Позволяет писать «в очередь напрямую»:

```
publish(exchange="", routingKey="my-queue", body="...")
```

Кратко для простых случаев, но плохой стиль — теряется гибкость.

---

## 5. Queue — свойства

Основные атрибуты:

- **Durable** — при рестарте broker'а очередь остаётся.
- **Exclusive** — только текущее соединение, удаляется при disconnect.
- **Auto-delete** — удаляется когда последний consumer отписывается.
- **Arguments** — дополнительные (TTL, DLX, max-length, quorum).

### 5.1 Durable + persistent

Чтобы сообщения выживали рестарт broker'а:
1. Queue durable=true.
2. Сообщение publish с `deliveryMode=2` (persistent).

Без обоих — сообщения теряются.

### 5.2 Quorum queues vs Classic

- **Classic** — старые, single-node или mirror-based HA.
- **Quorum** — на Raft, репликация между узлами, сильнее гарантии, чуть медленнее.

Для новых проектов — quorum. Для legacy — classic.

---

## 6. Connection и Channel

- **Connection** — TCP-соединение с broker'ом. Тяжёлое, одно на приложение.
- **Channel** — легковесный «мультиплекс» внутри connection. Producer/consumer работает через channel.

```java
Connection conn = factory.newConnection();
Channel ch = conn.createChannel();
ch.basicPublish(exchange, routingKey, props, body);
```

Правило: один Connection на приложение, много Channel'ов (по одному на поток).

---

## 7. Publisher acknowledgments и mandatory

### 7.1 Publisher confirms

По умолчанию `basicPublish` — fire-and-forget. Не знаешь дошло ли до brokerа.

С confirms — broker подтверждает получение:
```java
ch.confirmSelect();
ch.basicPublish(...);
ch.waitForConfirms();     // блокирует пока не подтвердит
```

### 7.2 Mandatory + returns

`mandatory=true` — если сообщение не сматчилось ни на одну очередь → broker вернёт producer'у (иначе просто выбросится).

```java
ch.addReturnListener(rl -> log.warn("Unroutable: {}", rl.getReplyText()));
ch.basicPublish(exchange, routingKey, true, false, props, body);
//                                     mandatory
```

---

## 8. Consumer

### 8.1 Push vs Pull

**Push** (basicConsume) — broker сам шлёт сообщения когда есть, consumer callback вызывается.
**Pull** (basicGet) — consumer сам полит очередь. Плохо, медленно.

Всегда используй push.

### 8.2 Prefetch (QoS)

Сколько сообщений broker может отправить consumer'у без ack:
```java
ch.basicQos(10);   // не больше 10 в работе одновременно
```

Без prefetch broker отправит всё сразу → пусть consumer «залипнет» с 10000 in-flight. Правило: `prefetch=1` для медленных задач, `prefetch=50-100` для быстрых.

### 8.3 Ack modes

- **auto-ack** — сообщение считается доставленным сразу как отправлено consumer'у. Если consumer упадёт — сообщение потеряно.
- **manual ack** — consumer явно подтверждает.

```java
ch.basicConsume(queue, false /* autoAck=false */, (tag, delivery) -> {
    try {
        process(delivery.getBody());
        ch.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
    } catch (Exception e) {
        ch.basicNack(delivery.getEnvelope().getDeliveryTag(), false, true);
        // false = один тэг; true = requeue (вернуть в очередь)
    }
}, tag -> {});
```

Всегда используй manual ack — единственный способ надёжной доставки.

### 8.4 Reject / Nack

- `basicReject` — отвергнуть одно сообщение.
- `basicNack` — то же, но можно диапазон тэгов + requeue flag.
- `requeue=true` — вернуть в очередь (может лоопить бесконечно!).
- `requeue=false` — выбросить (или в DLX, см. следующий файл).

---

## 9. Virtual hosts

**Vhost** — логическая изоляция внутри brokerа. Разные vhosts — разные exchanges, queues, users, permissions.

Аналог: одна БД PostgreSQL vs много схем.

```
rabbitmq://user:pass@host:5672/knp          ← vhost "knp"
rabbitmq://user:pass@host:5672/fno          ← vhost "fno"
```

В ИСНА обычно один vhost `/` — множество очередей внутри. Иногда — отдельные для разных подсистем.

---

## 10. Пример: событийная модель ИСНА

Сценарий: пользователь подал ФНО в КНП. Надо отправить в АРМ (tax-rep) на приёмку.

### 10.1 Модель

```
knp-integration (producer)
       │
       │  publish (exchange="knp.events", routingKey="fno.submitted",
       │           body=JSON{fnoId=123, regNum=...})
       ▼
   ┌──────────────────┐
   │ Exchange         │
   │ knp.events       │
   │ (type=topic)     │
   └────┬──────────┬──┘
        │          │
   binding      binding
   "fno.*"      "audit.#"
        │          │
        ▼          ▼
   ┌──────────┐ ┌──────────┐
   │ fno.queue│ │audit.queue│
   └────┬─────┘ └────┬──────┘
        │            │
        ▼            ▼
    tax-rep      audit-svc
   consumer     consumer
```

### 10.2 Producer

```java
@Autowired RabbitTemplate rabbit;

void publishFnoSubmitted(Long id) {
    rabbit.convertAndSend(
        "knp.events",              // exchange
        "fno.submitted",            // routing key
        new FnoSubmittedEvent(id));
}
```

### 10.3 Consumer

```java
@RabbitListener(queues = "fno.queue")
void receive(FnoSubmittedEvent event) {
    processFno(event.getFnoId());
    // Spring auto-ack при success; на exception — nack (requeue или DLX)
}
```

### 10.4 Что даёт

- KNP не знает про tax-rep напрямую (только про exchange).
- tax-rep упал → сообщения копятся, при рестарте — обработает.
- Добавили audit-svc — просто новая queue + binding, KNP не меняем.

---

## 11. Топология в ИСНА

В ИСНА RabbitMQ — центральная шина.

Основные очереди (примерно):
- `knp.approvals` — из KNP в tax-rep для приёмки ФНО/ФО.
- `notifications` — уведомления.
- `charges` — разноска (пример memory `knp-raznoska-charge-posting-diag`).
- `eaes` — реестры ЕАЭС (memory `knp-eaes-*`).
- `sync.*` — межсистемная синхронизация (memory `knp-fno-outer-sync-esb-dead-route`, `knp-fo-sync-notification-bugs`).

Обычно с DLX (dead-letter exchange) для проблемных сообщений — см. следующий файл.

---

## 12. Management UI и CLI

### 12.1 Management UI

Web-морда на порту 15672.
- Overview, connections, channels.
- Exchanges, queues.
- Publish/consume вручную для дебага.
- Просмотр сообщений в очереди.

Реальный пример: смотреть где застряли approvals — количество в очереди, скорость обработки.

### 12.2 CLI

```bash
rabbitmqctl list_queues name messages consumers
rabbitmqctl list_exchanges
rabbitmqctl list_bindings
rabbitmqctl list_connections
rabbitmqctl purge_queue my.queue
rabbitmqctl add_vhost knp
rabbitmqctl set_permissions -p knp user ".*" ".*" ".*"
```

### 12.3 HTTP API

Аналог CLI через REST:
```bash
curl -u guest:guest http://localhost:15672/api/queues
curl -u guest:guest http://localhost:15672/api/exchanges
```

---

## 13. Собесные вопросы

1. **Зачем нужен message broker?** — Асинхронность, буферизация, развязка, гарантии.
2. **Что такое AMQP?** — Стандарт messaging; Rabbit — реализация.
3. **Основные концепты AMQP?** — Producer/Consumer/Broker/Exchange/Queue/Binding/RoutingKey.
4. **4 типа exchange?** — Direct (по точному key), Topic (wildcards), Fanout (все), Headers (по заголовкам).
5. **Разница direct и topic?** — Direct = точное совпадение; topic = pattern с * и #.
6. **Что такое default exchange?** — Direct exchange "" с автоматическим binding на каждую очередь по имени.
7. **Что делает mandatory флаг?** — Если сообщение никуда не сматчилось — вернуть producer'у.
8. **Что такое publisher confirms?** — Broker подтверждает получение сообщения.
9. **Разница auto-ack и manual ack?** — Auto — сразу; manual — consumer явно подтверждает после обработки. Без manual — потеря при падении consumer'а.
10. **Что такое prefetch (QoS)?** — Сколько сообщений broker может отправить consumer'у без ack.
11. **Разница quorum и classic queues?** — Quorum на Raft, replicated, надёжнее; classic — legacy.
12. **Что такое vhost?** — Логическая изоляция (аналог namespace).
13. **Разница connection и channel?** — Connection = TCP; channel = мультиплекс внутри connection, дешёвый.

---

## Итог

- **AMQP** = producer → exchange → (binding) → queue → consumer.
- **Exchange types**: direct, topic, fanout, headers.
- **Queue** durable + message persistent = переживают рестарт.
- **Manual ack** + **prefetch** — обязательно для надёжности.
- **Publisher confirms** + **mandatory** — гарантия producer'а.
- **Vhost** — изоляция.
- **Connection + Channel** — один TCP, много каналов.
- **Management UI** — must для дебага.

Следующий — `21-rabbitmq-delivery-guarantees.md`.
