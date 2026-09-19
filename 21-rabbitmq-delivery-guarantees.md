# 21. RabbitMQ: гарантии доставки

Ack modes, publisher confirms, dead-letter, retry, идемпотентность.

---

## 1. Три модели доставки

- **At-most-once** — «максимум один раз». Сообщение может потеряться, но не задвоится. Без ack.
- **At-least-once** — «минимум один раз». Не потеряется, может задвоиться. С ack + возможным re-delivery.
- **Exactly-once** — «ровно один раз». Идеал, но дорог. Достигается через at-least-once + идемпотентность consumer'а.

**Правило**: RabbitMQ даёт at-least-once. Идемпотентность на твоей стороне.

---

## 2. Consumer acknowledgments — глубже

### 2.1 Ack modes

- **auto (autoAck=true)** — сообщение считается доставленным как только отправлено consumer'у.
  - Плюсы: быстрее.
  - Минусы: если consumer упал во время обработки — сообщение потеряно.
- **manual (autoAck=false)** — consumer явно подтверждает `basicAck` после обработки.
  - Единственный надёжный способ.

Всегда manual.

### 2.2 Ack, Nack, Reject

- **`basicAck(deliveryTag, multiple)`** — «обработал успешно».
  - `multiple=true` — акнулись все до этого тэга (bulk).
- **`basicNack(deliveryTag, multiple, requeue)`** — «не смог».
  - `requeue=true` — вернуть в очередь (будет доставлено снова).
  - `requeue=false` — выбросить (или в DLX, см. §5).
- **`basicReject(deliveryTag, requeue)`** — то же что nack на одно сообщение.

### 2.3 Что если не ack и не nack?

Сообщение «в работе» (unacknowledged) пока consumer держит канал. Если connection закрыт (crash, disconnect) — broker автоматически re-queue это сообщение → другой consumer получит.

Отсюда следствие: **crash consumer'а = сообщение не теряется**, при условии manual ack и durable queue.

### 2.4 Redelivered flag

Когда сообщение re-delivered (после nack requeue / crash) — у него `redelivered=true`. Consumer может использовать для «поймать вторую попытку».

---

## 3. Publisher confirms

### 3.1 Проблема

`basicPublish` возвращает управление сразу — TCP-write и forget. Не знаешь дошло ли до brokerа. Broker упал в момент send — сообщение потеряно.

### 3.2 Confirms

`ch.confirmSelect()` — включает режим подтверждений. Дальше можно:

**Синхронно** (медленно):
```java
ch.basicPublish(...);
if (ch.waitForConfirms(5000)) {
    // подтверждено
} else {
    // timeout / nack
}
```

**Асинхронно** (быстро):
```java
ch.addConfirmListener(
    (deliveryTag, multiple) -> log.debug("acked {}", deliveryTag),
    (deliveryTag, multiple) -> log.error("nacked {}", deliveryTag)
);
ch.basicPublish(...);
// продолжаем
```

Broker подтверждает после того как сообщение записано в durable queue (или в memory для non-durable).

### 3.3 Publisher retries

Если publish failed (nack или timeout) — retry:
```java
int attempts = 0;
while (attempts < 3) {
    try {
        ch.basicPublish(...);
        if (ch.waitForConfirms(5000)) break;
    } catch (Exception e) { }
    attempts++;
}
```

Осторожно с дублями. Идемпотентность consumer'а обязательна.

---

## 4. Mandatory + returns

Что если publish в exchange, у которого нет bindings?

**По умолчанию** — сообщение silently выкидывается. **Producer не знает.**

**С `mandatory=true`** — broker вернёт сообщение через `ReturnListener`:
```java
ch.addReturnListener(returned -> {
    log.warn("Unroutable: {}, exchange={}, routingKey={}",
        returned.getReplyText(), returned.getExchange(), returned.getRoutingKey());
    // сохранить/повторить/alert
});
ch.basicPublish(exchange, routingKey, true /* mandatory */, false, props, body);
```

Использовать всегда — иначе тихие потери когда кто-то удалил binding.

---

## 5. Dead Letter Exchange (DLX)

Что делать с сообщениями которые consumer не может обработать?

### 5.1 Схема

Настраиваешь очередь с DLX:
```
Queue: my-queue
  arguments:
    x-dead-letter-exchange: my.dlx
    x-dead-letter-routing-key: my-queue.failed
```

Сообщение попадает в DLX если:
1. **nack/reject с `requeue=false`** — consumer явно выбросил.
2. **TTL истёк** — сообщение слишком долго в очереди.
3. **max-length превышен** — очередь переполнена, старые вытесняются.

Из DLX сообщение маршрутизируется в **dead-letter queue** — обычно для ручного разбора.

```
                          ┌─── my-queue ───┐
producer ──► my.exchange ─►│                │──► consumer
                          │                │       │
                          │                │       │ nack requeue=false
                          │                │       ▼
                          │  x-dead-letter │──► my.dlx ──► my.dlq (dead-letter queue)
                          └────────────────┘             │
                                                         ▼
                                                    manual review / alert
```

### 5.2 Зачем

- Не терять «плохие» сообщения (нельзя обработать, но нельзя и потерять — надо разобраться).
- Изолировать: плохое сообщение не блокирует остальные.
- Alerting: DLQ растёт → шлём алерт SRE.

### 5.3 Пример из ИСНА

Приёмка ФНО из очереди `knp.approvals` в tax-rep. Если ФНО битое (парсинг упал) — в `knp.approvals.dlq`. Команда разбирается.

---

## 6. Retry с backoff

Проблема: consumer упал → сообщение re-queued → сразу опять пришло → опять упал. **Infinite loop.**

Решения.

### 6.1 Retry topic pattern

Основная очередь → retry.5s → retry.30s → retry.5m → DLQ.

Каждая retry-очередь с TTL. Сообщение попадает в retry, ждёт TTL, потом в основную (или следующий retry). Header `x-retry-count` считает попытки.

### 6.2 Spring Retry

Через Spring AMQP (см. следующий файл) `RetryInterceptor`:
```java
@Bean
RetryOperationsInterceptor retryInterceptor() {
    return RetryInterceptorBuilder.stateless()
        .maxAttempts(3)
        .backOffOptions(1000, 2.0, 10000)   // start=1s, multiplier=2, max=10s
        .recoverer(new RejectAndDontRequeueRecoverer())   // после N попыток → DLX
        .build();
}
```

### 6.3 В consumer'е самостоятельно

```java
int retries = getRetries(delivery);
if (retries > 3) {
    ch.basicNack(tag, false, false);   // → DLX
    return;
}
try {
    process(delivery);
    ch.basicAck(tag, false);
} catch (RetryableException e) {
    Thread.sleep(backoff(retries));
    ch.basicNack(tag, false, true);    // requeue
}
```

Плохо блокировать поток `Thread.sleep`. Лучше через retry-очереди.

---

## 7. TTL

Время жизни сообщения / очереди.

### 7.1 Per-message TTL

```java
props.setExpiration("60000");   // 60 сек
ch.basicPublish(exchange, key, props, body);
```

Через 60 сек — DLX (если настроен) или выбрасывается.

### 7.2 Per-queue TTL

```
x-message-ttl: 60000
```

Все сообщения в очереди с TTL. Полезно для «уведомлений которые устаревают».

### 7.3 Queue TTL

```
x-expires: 3600000
```

Очередь удаляется если нет consumer'ов и не используется 1 час.

---

## 8. Идемпотентность consumer'а

At-least-once → возможны дубли. Как избежать проблем?

### 8.1 Уникальный ключ операции

Каждое сообщение имеет `messageId` (или бизнес-ключ). Consumer проверяет — уже обработано?

```java
@Transactional
void receive(FnoSubmittedEvent event) {
    if (processedRepo.existsByMessageId(event.getMessageId())) {
        log.info("Skip duplicate {}", event.getMessageId());
        return;
    }
    processFno(event);
    processedRepo.save(new ProcessedMessage(event.getMessageId(), Instant.now()));
}
```

### 8.2 Идемпотентные операции

`UPDATE ... WHERE id=X SET status='PROCESSED' AND status='NEW'` — дубль не сработает второй раз (условие уже не выполнится).

### 8.3 UPSERT

`INSERT ... ON CONFLICT DO NOTHING/UPDATE` — вставка не задваивается.

Идемпотентность — **обязательное свойство** consumer'а в message-based системе.

---

## 9. Ordering

RabbitMQ гарантирует порядок **в одной очереди при одном consumer'е**.

Проблемы:
- Несколько consumer'ов на одной очереди → порядок не гарантируется.
- Rejects с requeue → сообщение возвращается в **head очереди** (нарушает порядок).
- Sharding: разные очереди — независимый порядок.

Если критичен порядок:
- Один consumer per queue (низкая пропускная способность).
- Sharding по бизнес-ключу (одна orderId всегда в одну queue → один consumer per queue → порядок внутри клиента).

---

## 10. Prefetch — тонкая настройка

Prefetch = сколько сообщений broker может отдать consumer'у без ack.

- **prefetch=1** — по одному. Для медленных задач (10+ сек). Гарантия честного распределения между consumer'ами.
- **prefetch=10-50** — для типичной задачи (100 мс - 1 сек).
- **prefetch=100+** — только очень быстрые задачи (< 10 мс).

Слишком много → один consumer забирает всё, остальные простаивают. Слишком мало → низкая throughput.

---

## 11. Flow control

Broker может замедлить producer'а если очереди переполняются:
- **Memory alarm** — превышен `vm_memory_high_watermark`.
- **Disk alarm** — мало места.

Producer блокируется на `basicPublish` (или confirms nack). Consumer'ы работают дальше.

Мониторинг критичен — memory alarm надо ловить и разбираться.

---

## 12. Lazy queues

Обычные queues — сообщения в RAM + disk (если persistent). Lazy — только disk, минимум RAM.

Для больших очередей (миллионы сообщений):
```
x-queue-mode: lazy
```

Медленнее чтение, но не убивает broker памятью.

Quorum queues по умолчанию похожи на lazy.

---

## 13. Пример: полная надёжная схема

```
producer
  │  publisherConfirms=true, mandatory=true
  │  set messageId, timestamp
  ▼
exchange (durable=true)
  │
  ▼
queue (durable=true, x-dead-letter-exchange=my.dlx, x-message-ttl=3600000)
  │
  │  basicConsume manualAck, prefetch=10
  ▼
consumer
  │  дедуп по messageId
  │  process
  │  ↳ success → basicAck
  │  ↳ retryable → nack requeue=true (или через retry-топик)
  │  ↳ fatal → nack requeue=false → DLX → DLQ → alert
```

---

## 14. Реальный кейс из ИСНА

Memory `knp-fo-sync-notification-bugs`: NotificationSyncService — 6 багов, часть связана с очередями:
- `printStackTrace` вместо log.error → ELK не видит error → «тихие» потери.
- Тихий скип на ошибке (MAX offset) → потеря сообщений.
- @Transactional мёртв → грязные commits.

Правильный consumer:
1. Structured logging (не printStackTrace).
2. Явный ack/nack с валидными аргументами.
3. Метрики: rate ack/nack/redeliver.
4. Alert на DLQ.

---

## 15. Собесные вопросы

1. **Три модели доставки?** — at-most-once, at-least-once, exactly-once.
2. **Что даёт RabbitMQ и как достичь exactly-once?** — At-least-once; exactly-once = at-least-once + идемпотентность.
3. **Что такое publisher confirms?** — Broker подтверждает получение сообщения; sync или async.
4. **Что делает mandatory?** — Возвращает сообщение producer'у если не сматчилось ни на одну очередь.
5. **Что такое DLX?** — Exchange для «плохих» сообщений (nack, TTL истёк, max-length).
6. **Когда сообщение попадает в DLX?** — nack/reject с requeue=false, TTL expired, max-length exceeded.
7. **Как избежать redelivery loop?** — Retry с backoff, max-attempts, потом DLX.
8. **Что такое prefetch?** — Сколько сообщений может быть unacked у consumer'а; настройка concurrency.
9. **Почему нужна идемпотентность?** — At-least-once = возможны дубли; consumer должен быть идемпотентен.
10. **Как обеспечить идемпотентность?** — Уникальный messageId + processed_messages таблица, или идемпотентные UPDATE/UPSERT.
11. **Что такое TTL?** — Время жизни сообщения (per-message или per-queue).
12. **Что такое flow control?** — Broker замедляет producer'а при memory/disk alarm.
13. **Ordering в Rabbit — какие условия?** — Один consumer, одна queue, без requeue.
14. **Разница quorum и classic queues в failure?** — Quorum: replicated Raft, автоматический failover; classic: single-node или mirror-based (deprecated).

---

## Итог

- **At-least-once** default; идемпотентность обязательна.
- **Manual ack** — единственный надёжный способ.
- **Publisher confirms + mandatory** — гарантии на producer'е.
- **DLX + retry** — для «плохих» сообщений.
- **TTL** — устаревшие сообщения.
- **Prefetch** — контроль concurrency.
- **Идемпотентность** — через messageId / UPSERT / conditional UPDATE.
- **Ordering** — только при 1 consumer + 1 queue + no requeue.

Следующий — `22-spring-amqp.md`.
