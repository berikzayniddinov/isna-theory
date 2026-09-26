# 50. Saga pattern: distributed transactions в microservices

## Зачем нужен новый подход к транзакциям

Разработчик привыкший к монолиту понимает transactions как ACID guarantee. BEGIN, work, COMMIT — либо всё либо ничего. Database enforces atomicity. Simple mental model работающая десятилетиями для single-database applications.

В микросервисах эта модель ломается. Order creation touching Order service, Inventory service, Payment service, Shipping service — every service свой database. No single transaction can encompass writes к всем. Failures между steps leave system в inconsistent state. Business needs — atomic ordering process — clashes с technical reality distributed data.

Разница между разработчиком «знающим distributed transactions» и «понимающим Saga» очевидна в architectural decisions. Первый reaches для XA / 2PC — «distributed ACID»! Второй знает что XA fundamentally не работает для микросервисов — blocking, coordinator SPOF, doesn't scale, not all systems support XA (Kafka, MongoDB, REST APIs). Знает что CAP theorem означает fundamental trade-off — cannot have Consistency plus Availability plus Partition tolerance simultaneously. Знает что microservices choose Availability plus Partition tolerance — accepting eventual consistency через Saga plus Outbox plus idempotency.

В этом файле разберём Saga pattern глубоко. Проблема distributed transactions в microservices. Почему XA / 2PC не решение (technical reasons detailed). Что такое Saga fundamentally. Два вида — Choreography vs Orchestration (deep comparison). Compensating actions plus их limitations. State machines saga. Идемпотентность обязательное свойство. Consistency levels achievable. Практическая реализация в Spring/Java (choreography plus orchestration examples). Saga vs Event Sourcing взаимоотношение. Реальные грабли — расшатанные компенсации, race conditions, long-running sagas, идемпотентность.

## Проблема distributed transactions

Классический пример order flow в e-commerce.

Заказ requires steps.
1. Order-service — создать заказ (INSERT в Orders БД).
2. Inventory-service — зарезервировать товары (UPDATE Inventory БД).
3. Payment-service — списать деньги (INSERT в Payments БД плюс external gateway call).
4. Shipping-service — создать shipment (INSERT в Shipping БД).

Если между шагами что-то упало — inconsistent state.
- Заказ создан, но не оплачен — customer видит order, но money не списаны.
- Деньги списаны, но товар не зарезервирован — customer paid, no goods to ship.
- Товар зарезервирован, но заказ отменён — inventory locked, order gone.

В монолите — одна DB-транзакция решает всё. BEGIN, do all updates, COMMIT. Atomicity guaranteed. Failure автоматически rollback.

В микросервисах — не работает. Different databases. Different processes. Different services. Different network partitions. Cannot enclose everything in one transaction.

Business requirement непреклонен — customer expects atomic outcome (order created plus payment charged plus goods shipping) или nothing. Technical reality не поддерживает classical ACID atomicity. Bridge через different approach — Saga pattern.

## Почему XA / 2PC не решение

Two-Phase Commit (2PC) — классический distributed transaction protocol.

Phase 1 (prepare). Координатор → всем участникам «готовы commit?». Каждый participant готовит write но не commits. Locks acquired. Responds «yes» или «no».

Phase 2 (commit или abort). Если все «yes» → координатор → всем «commit». Все finalize. Если хоть один «no» → координатор → всем «abort». Все rollback.

Промежуточные проблемы. Blocking — все ресурсы залочены на время всех phases. Concurrent transactions wait. Throughput drops. Coordinator single point of failure — если координатор упал между prepare и commit — участники stuck в prepared state, не могут ни commit ни abort. Recovery complex.

Не масштабируется. Координатор становится bottleneck. Latency accumulates. Every transaction requires 4+ network round-trips.

Не все системы поддерживают. Kafka, MongoDB, REST APIs не implement XA protocol. Только specific RDBMSs plus specific JMS providers.

Медленно. 2 round-trip минимум для 2PC. Latency multiplied. Not acceptable для user-facing operations.

Availability trade-off. При network partition — блокирует всё. If любой participant unreachable — transaction stuck. Cannot make progress.

CAP теорема (Eric Brewer) — не бывает одновременно Consistency + Availability + Partition tolerance в distributed system. Network partitions inevitable в distributed systems. Must choose — sacrifice Consistency (accept eventual consistency) или Availability (block during partitions). XA жертвует Availability.

Правильно для микросервисов — eventual consistency через Saga plus Outbox plus идемпотентность. Sacrifice immediate consistency ради Availability plus Partition tolerance.

## Что такое Saga

Saga — последовательность локальных транзакций. Каждая — в одном сервисе плюс одной БД. Local transaction ACID within its scope.

При ошибке — compensating transactions откатывают предыдущие. Не rollback в classical смысле — new transactions cancelling effects of previous.

Happy path:
```
Create Order → Reserve Inventory → Charge Payment → Create Shipping
```

Каждый step succeeds. Business process complete.

Failure сценарий:
```
Create Order → Reserve Inventory → Charge Payment (FAIL)
                                     ↓
                                   Release Inventory ← Cancel Order
```

Payment failed. Reverse execution — Release Inventory (compensation for Reserve), Cancel Order (compensation for Create). Business rolled back semantically. System back к consistent state.

Ключевое отличие от classical rollback. Нет rollback в БД-смысле — есть compensating actions (обратные business operations). Order still exists в database — но status = CANCELLED. Inventory reservation removed. History preserved. Audit trail intact.

Consistency eventual — временно система в inconsistent state (between failure и compensation completion). Приходит в consistent state after all compensations executed.

## Choreography vs Orchestration

Два способа implement saga. Fundamentally different approaches к coordination.

Choreography — каждый сервис слушает events plus публикует свои. Нет центрального оркестратора. Decentralized coordination:
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
```

Каждый service. Reacts на incoming events. Executes local transaction. Publishes events indicating outcome. Другие services listen для events касающихся their concerns.

При ошибке — compensation events:
```
[Payment-svc] PaymentFailed ──► [broker]
                                    │
                                    ▼
                              [Inventory-svc]  → ReleaseInventory + InventoryReleased
                                                              │
                                                              ▼
                                                       [Order-svc]  → OrderCancelled
```

Плюсы choreography.

Decentralized — нет единой точки отказа. Каждый service autonomous.

Каждый сервис знает только «что произошло у меня» плюс «на что реагирую». Simple service logic.

Естественный event-driven — fits event-based architectures naturally.

Простые сервисы — no central coordinator to manage.

Минусы choreography.

Сложно понять весь flow — логика размазана по всем сервисам. Reading code single service не reveals full business process.

Cyclic dependencies легко создать. Service A publishes X, listens к Y. Service B publishes Y listens к Z. Complex dependency graphs.

Отладка кошмар. Trace requires collating events across services. Timeline reconstruction difficult.

Тесно связано с broker (Rabbit / Kafka). Vendor lock-in при sophisticated usage.

Изменение flow → правки во всех сервисах. New step requires modifying multiple services.

Когда использовать choreography. Простые sagas (2-3 шага). Естественная event-driven архитектура. Малое количество service-owners.

Orchestration — есть центральный оркестратор (state machine) руководит sagой.
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

Плюсы orchestration.

Централизованная логика saga — вся видна в одном месте. Reading orchestrator code reveals full process.

Легко отлаживать и изменить flow. State machine explicit. Trace shows saga progression.

State машина явна. Formal representation. Can be visualized (Camunda BPMN).

Можно visualize через diagramming tools. Business stakeholders can review flows.

Проще тестировать. Test orchestrator independently. Unit tests possible для saga logic.

Минусы orchestration.

Оркестратор — центральная сущность, но не SPOF если сам HA. Requires infrastructure for saga state.

Дополнительный сервис. One more thing to manage.

Больше infra (сохранение состояния saga). State machine persistence needed.

Реализации. Camunda / Zeebe — BPMN engine. Netflix Conductor. AWS Step Functions. Temporal / Cadence — workflow engines. Axon Framework — Java, sagas plus event sourcing. Custom — state machine plus БД для state.

Когда использовать orchestration. Complex sagas (5+ шагов, ветвление). Нужна visibility of state. Разные команды владеют сервисами. Большие enterprise systems.

## Compensating actions

Ключевая идея saga — обратные операции.

Не всегда возможны. Refund payment после charge — да, standard business operation. Un-send email — нет, email already delivered. Un-delete row — можно если soft delete implemented. Un-execute request — как правило нет.

Правило — осторожно с одноходовыми actions (email, physical shipment, external calls). Планируй «точки невозврата». Sometimes design saga avoiding these operations, sometimes accept as pointof-no-return.

Semantic vs technical compensation.

Technical — противоположная DB-операция. DELETE после INSERT. Simple но loses history — no trace что happened.

Semantic — новая операция отменяющая эффект. Refund vs delete Payment record. Audit trail preserved — both charge и refund visible historically. Business often requires this для compliance.

Часто semantic лучше. Preserves history critical для financial systems, regulatory reporting. Users benefit from seeing complete history даже when things get cancelled.

Backwards recovery vs forwards recovery.

Backwards — при ошибке rollback компенсациями. Undo previous work.

Forwards — retry шага до успеха. Например платёж временно недоступен — попробовать снова. Payment gateway transient failure — retry might succeed.

Часто combined. Retry N раз, потом compensate. Balance between opportunistic recovery и eventual cleanup.

## State машина saga

Saga = FSM (Finite State Machine). Formal representation states plus transitions.

Пример order saga:
```
States:
  STARTED → ORDER_CREATED → INVENTORY_RESERVED → PAYMENT_COMPLETED → SHIPPED → COMPLETED
                                                                          │
                                                                          └─→ FAILED (при ошибке)

Compensation states:
  PAYMENT_FAILED → INVENTORY_RELEASING → INVENTORY_RELEASED → ORDER_CANCELING → CANCELED
```

State machine хранится где-то. БД для orchestration-based sagas. Orchestrator state store.

При старте orchestrator читает state — продолжает с последнего успешного шага. Idempotent recovery — если orchestrator crashed после Inventory reserved, восстановление continues from Charge Payment step.

Explicit state model provides several benefits. Debugging — can see exactly where saga stuck. Monitoring — count sagas per state. Recovery — resume from crash preserved state. Testing — mock saga state, test transitions.

## Идемпотентность обязательна

При retry / re-delivery events consumer'ы могут получить сообщение несколько раз. Каждая операция saga должна быть идемпотентной.

ReserveInventory(orderId=42) — второй вызов не должен резервировать повторно. If already reserved for this order — return existing reservation, not create new one.

ChargePayment(orderId=42) — не списывать дважды. Duplicate charge protection essential для financial correctness.

Реализация.

INSERT ... ON CONFLICT DO NOTHING (UPSERT). Second attempt no-op silently.

Условный UPDATE. WHERE status='NEW' clause. Only affects rows still expecting operation.

Processed table. Уникальный key operation, проверять перед действием. Explicit deduplication table.

Обсуждали в файле 21 rabbitmq-delivery-guarantees детально.

## Consistency levels

Saga даёт eventual consistency. Between шагами saga система в промежуточном состоянии. Читатель может видеть «полу-завершённое».

Mitigation strategies.

Semantic locks. Помечать сущность как «в процессе»:
```sql
UPDATE orders SET status='PENDING_PAYMENT' WHERE id=42;
```

Другие процессы видят статус, не пытаются работать со «свежими». Application logic filters based on status.

Commutative operations. Если операции коммутативны — порядок не важен, не блокирует. Add credit +$100 plus Add credit +$50 — commutative, can happen in any order.

Pessimistic view. Клиент читает только «финальные» состояния. UI filter по status showing only COMPLETED orders. Intermediate states hidden.

Compensation transactions logged. Логируешь pending sagas — админ может ручную починку. Sagas stuck без automatic recovery flagged для human attention.

## Реализация в Spring/Java

Choreography — просто через events:
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

Order created triggers event. Inventory service reacts. Success or failure published. Order service listens к failures, initiates cancellation.

Orchestration — state machine:
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

Упрощённо. Реальный код сложнее (async, retry, timeouts, exact-once semantics).

## Saga vs Event Sourcing

Saga — координация distributed transaction. Cross-service business process management.

Event Sourcing — способ хранения (события как source of truth). Не текущее state stored, но история events that led к current state.

Часто вместе — saga генерирует commands plus events. Event sourcing хранит history событий каждого service.

Но независимы. Можно только saga (без ES). Можно только ES (без sagas). Оба вместе for complex event-driven systems.

## Что почитать

Chris Richardson — Microservices Patterns (одна из главных книг о микросервисах).

Sam Newman — Building Microservices (comprehensive intro).

microservices.io (Chris Richardson) — online catalog patterns.

Если серьёзно — начни с них. Systematic study preferred over piecemeal blog reading.

## Реальные грабли

Замотался в compensations. Компенсации могут падать — нужны compensations for compensations. Endless loop potential.

Fix. Логировать все шаги detailed. Manual intervention для failed sagas — human resolves after N retry attempts. Timeouts на compensations — bounded retry effort.

«Что если во время compensation другой message пришёл». Race conditions между forward flow и compensations. Order service compensating (cancelling order) при same time receives new event about same order. Ordering unclear.

Fix. Использовать saga instance ID плюс версионирование. Each operation carries saga instance ID. State machine ensures ordering. Version numbers prevent stale updates.

Long-running sagas. Некоторые sagas длятся дни (booking → travel → hotel check-in). Оркестратор должен переживать рестарты, retry периодически. Persist state carefully. Handle timeouts appropriately.

Идемпотентность. Обязательно. Иначе retry задваивает эффект. Every action must be safely repeatable.

Партии / concurrent sagas. Много одновременных sagas → contention на одних resources → deadlock potentially.

Semantic locks plus retry с backoff. If reservation contention detected, retry after delay. Progress eventually через exponential backoff.

## Итоги

Saga это distributed transaction через local tx plus compensations. Не classical ACID но eventual consistency practical для микросервисов.

XA/2PC не работает для микросервисов. Blocking, SPOF, doesn't scale, not universal. CAP theorem — trade-off. Microservices choose Availability plus Partition tolerance над Consistency.

Choreography — events, децентрализовано. Простые sagas, малое количество services.

Orchestration — central state machine. Complex sagas, visibility, easier debugging.

Compensating actions — обратные business operations. Semantic лучше technical usually. Не все operations reversible.

State machines explicit. Recovery from crashes через saved state. Monitoring visible per state.

Идемпотентность обязательна. Retry-safe operations. Duplicate detection through processed tables, upsert, conditional updates.

Consistency levels. Semantic locks через status flags. Pessimistic views filtering intermediate states. Logging failed sagas для manual intervention.

Реализация в Java. Choreography через event listeners plus @TransactionalEventListener. Orchestration через explicit state machine class.

Saga complements Event Sourcing but independent. Оба cover different concerns.

Grabli — endless compensation loops, race conditions, long-running sagas, idempotency issues, concurrent contention. Все solvable through discipline plus careful design.

Дальше — Outbox pattern для reliable event publishing linked с database transactions.
