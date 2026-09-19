# 50. Distributed data + Saga pattern

Как обеспечить консистентность когда транзакция затрагивает несколько сервисов.

---

## 1. Проблема distributed transactions

Классический пример: **order flow** в e-commerce.

Заказ:
1. **Order-service** — создать заказ (INSERT в Orders БД).
2. **Inventory-service** — зарезервировать товары (UPDATE Inventory БД).
3. **Payment-service** — списать деньги (INSERT в Payments БД + external gateway).
4. **Shipping-service** — создать shipment (INSERT в Shipping БД).

Если между шагами что-то упало → **inconsistent state**:
- Заказ создан, но не оплачен.
- Деньги списаны, но товар не зарезервирован.
- Товар зарезервирован, но заказ отменён.

**В монолите** — одна DB-транзакция решает всё. **В микросервисах** — не работает (разные БД, разные процессы, разные сервисы).

---

## 2. Почему XA / 2PC не решение

**2PC (Two-Phase Commit)** — классический distributed transaction:
- **Prepare**: координатор → всем «готовы?».
- **Commit** или **Abort**.

Разбирали в файле `32-transactions-acid-isolation-propagation.md`.

Почему **не работает в микросервисах**:
- **Blocking** — все ресурсы залочены на время всех phases.
- **Coordinator single point of failure** — если координатор упал между prepare и commit — ресурсы залипли.
- **Не масштабируется** — координатор становится bottleneck.
- **Не все системы поддерживают** — Kafka, MongoDB, REST-APIs XA не умеют.
- **Медленно** — 2 round-trip.
- **Availability trade-off** — при network partition блокирует всё.

CAP теорема: **не бывает одновременно Consistency + Availability + Partition tolerance**. XA жертвует Availability.

**Правильно для микросервисов**: **eventual consistency** через **Saga** + **Outbox** + идемпотентность.

---

## 3. Что такое Saga

**Saga** — последовательность **локальных транзакций**. Каждая — в одном сервисе + одной БД.

При ошибке — **compensating transactions** откатывают предыдущие.

```
Happy path:
  Create Order → Reserve Inventory → Charge Payment → Create Shipping

Failure на Charge Payment:
  Create Order → Reserve Inventory → Charge Payment (FAIL)
                                       ↓
                                     Release Inventory ← Cancel Order
```

Ключевое: **нет rollback в БД-смысле** — есть **compensating actions** (обратные операции).

Consistency **eventual** — временно система в inconsistent state, но приходит в consistent.

---

## 4. Два вида Saga

### 4.1 Choreography (хореография)

Каждый сервис слушает events + публикует свои. **Нет центрального оркестратора**.

```
[Order-svc]  ──OrderCreated──►  [broker]
                                    │
                                    ▼
                              [Inventory-svc]
                                    │
                                    │  Reserve success:
                                    ▼
                              InventoryReserved ──►  [broker]
                                                        │
                                                        ▼
                                                    [Payment-svc]
                                                        │
                                                        │  Charge success:
                                                        ▼
                                                    PaymentCompleted ──► [broker]
                                                                              │
                                                                              ▼
                                                                        [Shipping-svc]
                                                                              │
                                                                              ▼
                                                                     ShipmentCreated
```

При ошибке — **compensation events**:
```
[Payment-svc] PaymentFailed ──► [broker]
                                    │
                                    ▼
                              [Inventory-svc]  → ReleaseInventory + InventoryReleased
                                                              │
                                                              ▼
                                                       [Order-svc]  → OrderCancelled
```

### 4.1.1 Плюсы

- **Decentralized** — нет единой точки отказа.
- Каждый сервис знает только «что произошло у меня» + «на что реагирую».
- Естественный event-driven.
- Простые сервисы.

### 4.1.2 Минусы

- **Сложно понять весь flow** — логика размазана по всем сервисам.
- **Cyclic dependencies** легко создать.
- Отладка кошмар.
- Тесно связано с broker'ом (Rabbit/Kafka).
- Изменение flow → правки во всех сервисах.

### 4.1.3 Когда использовать

- Простые sagas (2-3 шага).
- Естественная event-driven архитектура.
- Малое количество service-owners.

---

### 4.2 Orchestration (оркестрация)

Есть **центральный оркестратор** (state machine) — руководит sagой.

```
[Saga Orchestrator]
       │
       │  1. Command: CreateOrder
       ▼
   [Order-svc]  → OrderCreated event
       │
       ▼
[Saga Orchestrator] gets event
       │
       │  2. Command: ReserveInventory
       ▼
   [Inventory-svc]  → InventoryReserved event
       │
       ▼
[Saga Orchestrator]
       │
       │  3. Command: ChargePayment
       ▼
   [Payment-svc]  → PaymentCompleted event
       │
       ▼
[Saga Orchestrator]
       │
       │  4. Command: CreateShipping
       ▼
   [Shipping-svc]  → ShipmentCreated event
       │
       ▼
[Saga Orchestrator] → SagaCompleted
```

При ошибке — оркестратор шлёт compensating commands:
```
[Saga Orchestrator] → ReleaseInventoryCommand → [Inventory-svc]
                    → CancelOrderCommand      → [Order-svc]
```

### 4.2.1 Плюсы

- **Централизованная логика saga** — вся видна в одном месте.
- **Легко отлаживать / изменить flow**.
- **State машина явна**.
- Можно визуализировать (Camunda BPMN).
- Проще тестировать.

### 4.2.2 Минусы

- **Оркестратор** = центральная сущность, но не SPOF если сам HA.
- Дополнительный сервис.
- Больше infra (сохранение состояния saga).

### 4.2.3 Реализации

- **Camunda** / **Zeebe** — BPMN engine.
- **Netflix Conductor**.
- **AWS Step Functions**.
- **Temporal** / **Cadence** — workflow engines.
- **Axon Framework** — Java, sagas + event sourcing.
- **Custom** — state machine + БД для state.

### 4.2.4 Когда использовать

- Complex sagas (5+ шагов, ветвление).
- Нужна visibility of state.
- Разные команды владеют сервисами.
- Большие enterprise.

---

## 5. Compensating actions

Ключевая идея saga — **обратные операции**.

### 5.1 Не всегда возможны

- **Refund payment** после **charge** — да.
- **Un-send email** — нет.
- **Un-delete row** — можно, если soft delete.
- **Un-execute request** — как правило нет.

Правило: **осторожно с одноходовыми actions** (email, physical shipment, external calls). Планируй "точки невозврата".

### 5.2 Semantic vs technical compensation

- **Technical** — противоположная DB-операция (DELETE после INSERT).
- **Semantic** — новая операция, отменяющая эффект (Refund vs delete Payment record — audit trail важен).

Часто **semantic лучше** — сохраняешь историю «что произошло».

### 5.3 Backwards recovery vs forwards recovery

- **Backwards** — при ошибке rollback компенсациями.
- **Forwards** — retry шага до успеха (например, платёж временно недоступен — попробовать снова).

Часто **combined** — retry N раз, потом compensate.

---

## 6. State машина saga

Saga = FSM (Finite State Machine).

Пример order saga:
```
States:
  STARTED → ORDER_CREATED → INVENTORY_RESERVED → PAYMENT_COMPLETED → SHIPPED → COMPLETED
                                                                          │
                                                                          └─→ FAILED (при ошибке)

Compensation states:
  PAYMENT_FAILED → INVENTORY_RELEASING → INVENTORY_RELEASED → ORDER_CANCELING → CANCELED
```

Хранится где-то (БД, orchestrator state store).

При старте — orchestrator читает state → продолжает с последнего успешного шага.

---

## 7. Идемпотентность в saga

**Обязательно**.

При retry / re-delivery events consumer'ы могут получить сообщение несколько раз. Каждая операция saga должна быть идемпотентной:
- `ReserveInventory(orderId=42)` — второй вызов не должен резервировать повторно.
- `ChargePayment(orderId=42)` — не списывать дважды.

Реализация:
- **`INSERT ... ON CONFLICT DO NOTHING`** (UPSERT).
- **Условный UPDATE** (`WHERE status='NEW'`).
- **Processed table** (уникальный key operation, проверять перед действием).

Обсуждали в `21-rabbitmq-delivery-guarantees.md` детально.

---

## 8. Consistency levels

Saga даёт **eventual consistency**:
- Между шагами saga система в промежуточном состоянии.
- Читатель может видеть «полу-завершённое».

Как этого избежать / mitigate:

### 8.1 Semantic locks

Помечать сущность как «в процессе»:
```sql
UPDATE orders SET status='PENDING_PAYMENT' WHERE id=42;
```

Другие процессы видят статус, не пытаются работать со «свежими».

### 8.2 Commutative operations

Если операции коммутативны — порядок не важен, не блокирует.

Пример: `Add credit +$100` и `Add credit +$50` — можно в любом порядке.

### 8.3 Pessimistic view

Клиент читает только «финальные» состояния — фильтр по status.

### 8.4 Compensation transactions

Логируешь `pending` sagas → админ может ручную починку.

---

## 9. Реализация в Spring / Java

### 9.1 Choreography — просто через events

```java
@Service
class OrderService {
    @Autowired KafkaTemplate template;

    @Transactional
    public Order createOrder(OrderRequest req) {
        Order o = new Order(req, Status.PENDING);
        orderRepo.save(o);
        return o;
    }
}

@Component
class OrderEventPublisher {
    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void publish(OrderCreated event) {
        template.send("orders", event);
    }
}

@Component
class InventoryListener {
    @KafkaListener(topics = "orders")
    @Transactional
    public void handle(OrderCreated event) {
        if (processedRepo.exists(event.getOrderId())) return;
        try {
            reserve(event.getItems());
            processedRepo.save(...);
            events.publish(new InventoryReserved(...));
        } catch (OutOfStock e) {
            events.publish(new InventoryFailed(event.getOrderId(), e.getMessage()));
        }
    }
}

@Component
class InventoryFailedListener {
    @KafkaListener(topics = "inventory-failed")
    public void handle(InventoryFailed event) {
        orderService.cancelOrder(event.getOrderId(), event.getReason());
    }
}
```

### 9.2 Orchestration — state machine

```java
@Component
class OrderSaga {

    @Autowired OrderRepository orderRepo;
    @Autowired InventoryClient inventoryClient;
    @Autowired PaymentClient paymentClient;
    @Autowired ShippingClient shippingClient;
    @Autowired SagaStateRepository sagaRepo;

    public void start(OrderRequest req) {
        SagaState state = new SagaState(req);
        state.setStep(Step.CREATE_ORDER);
        sagaRepo.save(state);
        proceed(state);
    }

    void proceed(SagaState state) {
        try {
            switch (state.getStep()) {
                case CREATE_ORDER:
                    orderRepo.save(new Order(state.getRequest()));
                    state.setStep(Step.RESERVE_INVENTORY);
                    break;
                case RESERVE_INVENTORY:
                    inventoryClient.reserve(state.getItems());
                    state.setStep(Step.CHARGE_PAYMENT);
                    break;
                case CHARGE_PAYMENT:
                    paymentClient.charge(state.getAmount());
                    state.setStep(Step.CREATE_SHIPPING);
                    break;
                case CREATE_SHIPPING:
                    shippingClient.create(state.getShippingInfo());
                    state.setStep(Step.COMPLETED);
                    break;
                case COMPLETED:
                    return;
            }
            sagaRepo.save(state);
            proceed(state);
        } catch (Exception e) {
            state.setStep(Step.COMPENSATING);
            state.setError(e.getMessage());
            sagaRepo.save(state);
            compensate(state);
        }
    }

    void compensate(SagaState state) {
        // reverse-order compensations
        if (state.wasStep(Step.CHARGE_PAYMENT)) {
            paymentClient.refund(state.getPaymentId());
        }
        if (state.wasStep(Step.RESERVE_INVENTORY)) {
            inventoryClient.release(state.getReservationId());
        }
        if (state.wasStep(Step.CREATE_ORDER)) {
            orderRepo.cancelOrder(state.getOrderId());
        }
        state.setStep(Step.COMPENSATED);
        sagaRepo.save(state);
    }
}
```

Упрощённо — реальный код сложнее (async, retry, timeouts).

---

## 10. Saga vs Event Sourcing

- **Saga** — координация distributed transaction.
- **Event Sourcing** — способ хранения (события как source of truth).

**Часто вместе**: saga генерирует commands + events; event sourcing хранит historia.

Но независимы — можно только saga, только ES, оба.

---

## 11. Что почитать / литература

- **Chris Richardson — "Microservices Patterns"** (одна из главных книг).
- **Sam Newman — "Building Microservices"**.
- **microservices.io** (Chris Richardson).

Если серьёзно — начни с них.

---

## 12. Реальные грабли

### 12.1 Замотался в compensations

Компенсации могут падать → нужны compensations for compensations. Endless loop.

Fix:
- Логировать все шаги.
- **Manual intervention** для failed sagas.
- **Timeouts** на compensations.

### 12.2 «Что если во время compensation другой message пришёл»

Race conditions между forward flow и compensations. Использовать **saga instance ID** + версионирование.

### 12.3 Long-running sagas

Некоторые sagas длятся дни (booking → travel → hotel check-in). Оркестратор должен переживать рестарты, retry периодически.

### 12.4 Идемпотентность

Обязательно. Иначе retry → задваивает эффект.

### 12.5 Партии / concurrent sagas

Много одновременных sagas → contention на одних resources → deadlock.

Semantic locks + retry с backoff.

---

## 13. Собесные вопросы

1. **Что такое Saga?** — Последовательность локальных tx + compensating actions для distributed data consistency.
2. **Почему не 2PC/XA в микросервисах?** — Blocking, coordinator SPOF, не масштабируется, не все системы поддерживают.
3. **Choreography vs Orchestration?** — Choreography: события, децентрализовано; Orchestration: центральный оркестратор, state machine.
4. **Плюсы choreography?** — Decentralized, простые сервисы, естественный event-driven.
5. **Минусы choreography?** — Сложно понять весь flow, cyclic dependencies, отладка.
6. **Плюсы orchestration?** — Централизованная логика, visibility, легче отлаживать.
7. **Что такое compensating action?** — Обратная операция, откатывающая эффект (Refund после Charge).
8. **Backwards vs forwards recovery?** — Backwards: компенсации; forwards: retry до успеха.
9. **Что такое eventual consistency?** — Между шагами система в промежуточном state; в итоге приходит в consistent.
10. **Как обеспечить идемпотентность в saga?** — Уникальный ID + processed table / UPSERT / conditional UPDATE.
11. **Что такое semantic locks в saga?** — Помечать сущность как «в процессе» чтобы другие видели.
12. **Известные orchestrators?** — Camunda, Zeebe, Netflix Conductor, Temporal, AWS Step Functions, Axon.
13. **Saga vs 2PC — когда что?** — 2PC для monolith с XA-resources; Saga для microservices (eventual consistency).
14. **CAP теорема — как связано с sagas?** — Saga жертвует Consistency (immediate) в пользу Availability + Partition tolerance.

---

## Итог

- **Saga** = distributed transaction через local tx + compensations.
- **Choreography** (events) vs **Orchestration** (state machine).
- **Eventual consistency**, не ACID.
- **Compensating actions** — обратные операции.
- **Идемпотентность обязательна**.
- **Semantic locks** для скрытия промежуточных состояний.
- **Orchestrators**: Camunda / Zeebe / Temporal / Axon.

Следующий — `51-outbox-inbox-pattern.md`.
