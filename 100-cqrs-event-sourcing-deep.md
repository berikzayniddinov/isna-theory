# 100. CQRS и Event Sourcing: глубокая реализация

## Ещё раз про мотивацию

В файле 97 мы кратко обсудили CQRS и Event Sourcing как паттерны моделирования данных. Здесь погрузимся глубже: как эти паттерны работают на уровне реализации, какие типичные ошибки возникают, как применить их к реальному PostgreSQL + Java стеку. Это одна из самых сложных тем в архитектуре данных, и часто она применяется неправильно — либо преждевременно (превращая простой CRUD в сложную event-driven систему), либо неполностью (получая недостатки без преимуществ).

Обе техники — не просто паттерны, а фундаментальные сдвиги в мышлении. CQRS требует принять что read и write модели могут расходиться. Event Sourcing требует принять что текущее состояние — это результат последовательности событий, а не первичная сущность. Оба несовместимы с наивным «одна таблица = одна модель», но зато открывают возможности, которые в классическом подходе недостижимы.

В этом финальном файле разберём реализацию обоих паттернов подробно. CQRS с примерами разделения моделей: как выглядит write side, как read side, как синхронизация. Event Sourcing как архитектура: event store, aggregate reconstruction, snapshots, projections. Реализация на PostgreSQL. Комбинирование с Kafka. Где эти паттерны действительно нужны, а где создают проблемы без пользы.

## CQRS: базовая идея ещё раз

Command Query Responsibility Segregation. Идея простая: разделить операции на две категории. **Commands** — изменяют state, ничего не возвращают (кроме ok/error). **Queries** — читают state, ничего не меняют. И для каждой категории — своя модель, оптимизированная под её задачу.

В обычной архитектуре одна модель обслуживает и то, и другое. Domain object User имеет поля, методы для изменения (setEmail), возвращается при чтении. Repository сохраняет и извлекает. Одна таблица users в БД.

В CQRS write side и read side разделены. Write side — те же command handlers, изменяющие state, обычно через normalized модель. Read side — query handlers, читающие из **отдельной** структуры, оптимизированной под конкретные view (denormalized, indexed под конкретные фильтры). Между ними — синхронизация: команды публикуют события, read side обновляется по событиям.

Классический пример. Заказы. Write side — orders + order_items + customers + products (normalized, 4 таблицы). Query — API для мобильного приложения хочет вернуть list заказов пользователя со всеми деталями (item names, customer info, total). В обычной модели — 4-way JOIN, при 100M заказов — тормозит.

CQRS решение. Read side — таблица orders_view (denormalized): order_id, user_id, user_name, items (JSONB), total, status, created_at. Один SELECT — мгновенно. Обновляется по событиям OrderCreated / OrderUpdated / CustomerNameChanged / ItemAdded — соответствующая orders_view запись пересчитывается.

## Реализация CQRS: write side

Write side обычно ближе к traditional архитектуре. Command handlers принимают команды, валидируют, обновляют state, publish события.

Example on Java:

```java
public class CreateOrderCommand {
    private UUID orderId;
    private UUID userId;
    private List<OrderItem> items;
}

public class OrderCommandHandler {
    private final OrderRepository orderRepo;
    private final EventPublisher eventPublisher;
    
    @Transactional
    public void handle(CreateOrderCommand cmd) {
        // Валидация
        validateItems(cmd.getItems());
        
        // Создание domain object
        Order order = Order.create(cmd.getOrderId(), cmd.getUserId(), cmd.getItems());
        
        // Сохранение в write model
        orderRepo.save(order);
        
        // Публикация события (в той же транзакции через outbox)
        eventPublisher.publish(new OrderCreated(
            order.getId(),
            order.getUserId(),
            order.getItems(),
            order.getTotalAmount(),
            order.getCreatedAt()
        ));
    }
}
```

Ключевое: команда и публикация событий — в одной транзакции. Через Outbox pattern (файл 93). Гарантирует, что либо и order сохранён, и событие опубликовано, либо ни того ни другого.

Write model — normalized structure для integrity. Foreign keys, constraints, всё как обычно в реляционной модели.

## Read side

Read side — это отдельные модели (**projections**), одна на каждый query pattern. Обновляются подписчиками на события.

Пример проекции для мобильного приложения:

```sql
CREATE TABLE order_summaries (
    order_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    user_name TEXT NOT NULL,  -- денормализовано
    items JSONB NOT NULL,       -- денормализовано  
    total NUMERIC NOT NULL,     -- предвычислено
    status TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
CREATE INDEX ON order_summaries(user_id, created_at DESC);
```

Обновление проекции:

```java
public class OrderSummaryProjection {
    private final JdbcTemplate jdbc;
    
    @EventHandler
    public void on(OrderCreated event) {
        String userName = getUserName(event.getUserId());
        
        jdbc.update("""
            INSERT INTO order_summaries 
            (order_id, user_id, user_name, items, total, status, created_at, updated_at)
            VALUES (?, ?, ?, ?::jsonb, ?, 'PENDING', ?, ?)
            """, 
            event.getOrderId(), event.getUserId(), userName,
            toJson(event.getItems()), event.getTotalAmount(),
            event.getCreatedAt(), Instant.now());
    }
    
    @EventHandler
    public void on(OrderStatusChanged event) {
        jdbc.update("UPDATE order_summaries SET status = ?, updated_at = ? WHERE order_id = ?",
            event.getNewStatus(), Instant.now(), event.getOrderId());
    }
    
    @EventHandler
    public void on(CustomerNameChanged event) {
        jdbc.update("UPDATE order_summaries SET user_name = ? WHERE user_id = ?",
            event.getNewName(), event.getUserId());
    }
}
```

Заметьте: при CustomerNameChanged обновляются все order_summaries этого customer'а. Это OK для read model — там имя денормализовано, если поменялось у customer, надо обновить в orders. В write model оно хранится один раз в таблице users.

Query API:

```java
public class OrderQueryHandler {
    public List<OrderSummary> getUserOrders(UUID userId, int page, int size) {
        return jdbc.query("""
            SELECT * FROM order_summaries 
            WHERE user_id = ? 
            ORDER BY created_at DESC 
            LIMIT ? OFFSET ?
            """, ..., userId, size, page * size);
    }
}
```

Простой SELECT, никаких JOIN'ов, мгновенный.

## Разные типы read models

Одна из мощных сторон CQRS — можешь создавать любое количество read models под разные use cases.

**Order Summary View** — для клиентского мобильного приложения (мы делали выше).

**Admin Search View** — для оператора call-center, ищет заказ по номеру телефона клиента, по email, по product name.

```sql
CREATE TABLE order_search_index (
    order_id UUID PRIMARY KEY,
    search_text tsvector,  -- полнотекстовый search
    customer_phone TEXT,
    customer_email TEXT,
    product_names TEXT[],
    total NUMERIC,
    created_at TIMESTAMP
);
CREATE INDEX ON order_search_index USING gin(search_text);
CREATE INDEX ON order_search_index(customer_phone);
CREATE INDEX ON order_search_index(customer_email);
```

**Analytics View** — агрегаты для dashboard.

```sql
CREATE TABLE daily_sales_summary (
    day DATE,
    country VARCHAR(2),
    product_category TEXT,
    total_amount NUMERIC,
    order_count INT,
    PRIMARY KEY (day, country, product_category)
);
```

Каждая проекция подписана на нужные события, обновляется независимо. Одно событие OrderCreated обновляет всех: summary, search index, analytics.

Read models могут быть в разных технологиях. OrderSummary — PostgreSQL. Search — Elasticsearch. Analytics — ClickHouse. Каждая оптимизирована под свою задачу.

## Синхронизация: eventual consistency

Read side обновляется асинхронно. Между write и updated read view есть задержка — миллисекунды до секунд в зависимости от реализации.

Это создаёт проблемы user experience. Классика: пользователь создал заказ, редиректится на страницу списка заказов. Если read view ещё не обновилась — заказа нет. Пользователь думает, что операция не удалась.

Решения.

**Read your writes** — читать write model для только что изменённых сущностей. После creation'а конкретного заказа, читаем этот заказ из write model напрямую. Другие пользователи читают из read model.

**Wait for consistency** — после write ждём до N ms или до подтверждения обновления read view. Наивно, но работает для простых случаев.

**Optimistic UI** — front-end показывает изменение сразу локально, не дожидаясь read view. Если операция упадёт — показать ошибку, откатить.

**Return updated data in write response** — write API возвращает не только "OK", но и обновлённую сущность. Клиент не нужно даже read'ать.

Для КНП правильный компромис: для критичных путей (проверка что заявка сохранилась) — read your writes через write model. Для listов и обычной navigation — read view с acceptable задержкой в 1-2 секунды.

## Event Sourcing

Event Sourcing идёт дальше CQRS. В обычной системе state хранится напрямую: user Bob имеет email a@b.com. При изменении — UPDATE users SET email = 'c@d.com'. Старое значение потеряно.

Event Sourcing переворачивает это. State не хранится напрямую. Хранится **последовательность событий**, приведших к текущему состоянию. UserCreated(bob, a@b.com), UserEmailChanged(bob, c@d.com), UserDeactivated(bob). Текущее состояние — результат последовательного применения всех событий (fold operation).

Event Store — таблица (или специальная БД), хранящая все события:

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    aggregate_type TEXT NOT NULL,
    event_type TEXT NOT NULL,
    event_data JSONB NOT NULL,
    version INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (aggregate_id, version)
);
CREATE INDEX ON events (aggregate_id, version);
```

Каждое изменение — INSERT новой строки. UPDATE не бывает. DELETE не бывает. Events immutable по определению.

Unique constraint на (aggregate_id, version) обеспечивает **optimistic concurrency**: если два процесса одновременно попытались добавить event для одного aggregate с одинаковой version, второй упадёт с constraint violation.

## Aggregate reconstruction

Как получить текущее состояние agregate из истории событий.

```java
public class UserAggregate {
    private UUID id;
    private String email;
    private boolean active;
    private int version;
    
    public static UserAggregate fromEvents(List<Event> events) {
        UserAggregate user = new UserAggregate();
        for (Event e : events) {
            user.apply(e);
        }
        return user;
    }
    
    private void apply(Event e) {
        switch (e.getType()) {
            case "UserCreated":
                UserCreated created = (UserCreated) e;
                this.id = created.getUserId();
                this.email = created.getEmail();
                this.active = true;
                break;
            case "UserEmailChanged":
                UserEmailChanged changed = (UserEmailChanged) e;
                this.email = changed.getNewEmail();
                break;
            case "UserDeactivated":
                this.active = false;
                break;
        }
        this.version = e.getVersion();
    }
}
```

При загрузке aggregate по ID:

```java
public UserAggregate loadUser(UUID userId) {
    List<Event> events = eventStore.getEventsForAggregate(userId);
    return UserAggregate.fromEvents(events);
}
```

Читаем все события этого aggregate из event store в порядке version. Строим текущее состояние.

## Snapshots

Проблема очевидна. Если у aggregate 10000 событий (например, активный user за годы), считывать и применять все каждый раз — медленно.

**Snapshots** — периодически сохранённое текущее состояние aggregate. При reconstruction — начинаем не с нуля, а с последнего snapshot'а, применяем только события после него.

```sql
CREATE TABLE snapshots (
    aggregate_id UUID PRIMARY KEY,
    aggregate_type TEXT NOT NULL,
    state JSONB NOT NULL,
    version INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

Загрузка с использованием snapshot:

```java
public UserAggregate loadUser(UUID userId) {
    // 1. Load latest snapshot (если есть)
    Optional<Snapshot> snapshot = snapshotStore.getLatest(userId);
    
    UserAggregate user;
    int fromVersion;
    if (snapshot.isPresent()) {
        user = deserializeUser(snapshot.get().getState());
        fromVersion = snapshot.get().getVersion();
    } else {
        user = new UserAggregate();
        fromVersion = 0;
    }
    
    // 2. Apply events after snapshot
    List<Event> events = eventStore.getEventsAfter(userId, fromVersion);
    for (Event e : events) {
        user.apply(e);
    }
    return user;
}
```

Snapshot policy — когда создавать snapshot. Обычно: каждые N событий, каждые M секунд/минут, или при значительных milestone событиях. Trade-off: чаще snapshots — меньше replay, больше storage.

## Event Sourcing + CQRS

Обычно эти паттерны идут вместе. Event Sourcing — write side (source of truth — events). CQRS — read side (projections built from events).

Архитектура:

```
Client -> Command Handler -> Event Store (writes events)
                                   |
                                   v
                                Kafka/EventBus  
                                   |
                    ---------------|------------
                    v              v          v
              Projection1    Projection2   ...
                 (SQL)       (Elastic)
```

Command идёт в handler, тот сохраняет event в event store, event публикуется в message broker, все projection handler'ы читают и обновляют свои модели.

Query идёт в conкретную projection, читает быстро из оптимизированной модели.

Плюсы этой архитектуры.

**Complete audit trail**. Все изменения записаны. Что было в системе в любой момент времени — можно восстановить. Для compliance систем это огромная ценность.

**Time travel**. Можно посмотреть состояние системы на любой момент прошлого. «Что было с этим заказом 15 января в 14:00?» — replay events до этой точки.

**New projections retroactively**. Через год бизнес требует новый вид отчёта. В обычной системе — только с этого момента, история потеряна. В Event Sourcing — replay всех событий с начала времён, build новую projection задним числом.

**Debugging**. Дебажить проблему — воспроизвести последовательность событий в тестовом окружении. Если что-то не так, точно видно что произошло и в каком порядке.

Минусы.

**Complexity**. Программисты думают в events, не в state. Требуется другой mindset.

**Storage**. Events накапливаются бесконечно. Terabytes для крупных систем через годы. Может потребоваться archiving старых событий.

**Rehydration**. Сложные aggregates с сотнями событий medленно загружаются даже с snapshots.

**Schema evolution**. Event schema тоже эволюционирует, но старые events нельзя изменить (они уже произошли). Нужен upcasting: переводить старые events в новый формат при чтении.

**Query in old data сложно**. Не можешь просто "SELECT WHERE user = X" — нужно replay всей истории.

## Реализация Event Store на PostgreSQL

Простая реализация:

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    aggregate_type TEXT NOT NULL,
    event_type TEXT NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB,
    version INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (aggregate_id, version)
);

CREATE INDEX ON events (aggregate_id, version);
CREATE INDEX ON events (aggregate_type, id);
CREATE INDEX ON events (created_at);
```

Append event:

```java
public void appendEvent(Event event) {
    jdbc.update("""
        INSERT INTO events 
        (aggregate_id, aggregate_type, event_type, event_data, version)
        VALUES (?, ?, ?, ?::jsonb, ?)
        """,
        event.getAggregateId(), event.getAggregateType(),
        event.getEventType(), toJson(event.getData()), event.getVersion()
    );
    // Constraint violation означает concurrent modification
    // — throw OptimisticLockException
}
```

Load events for aggregate:

```java
public List<Event> getEvents(UUID aggregateId) {
    return jdbc.query("""
        SELECT * FROM events 
        WHERE aggregate_id = ? 
        ORDER BY version
        """, ..., aggregateId);
}
```

Публикация events в Kafka — через Debezium (CDC) или polling publisher (файл 93).

Специализированные Event Store'ы: EventStoreDB (спроектирован специально), Axon Server, AWS EventBridge (managed). Для больших систем — оправдано. Для средних — PostgreSQL достаточно.

## Тonкости schema evolution

Events immutable. Но требования к схеме событий меняются со временем. Как быть?

**Upcasting**. При чтении старых events преобразуем их в актуальный формат.

```java
public Event upcast(Event oldEvent) {
    if (oldEvent.getType() == "UserCreatedV1") {
        // V1 не имел поля country, добавляем default
        Map<String, Object> data = oldEvent.getData();
        data.put("country", "UNKNOWN");
        return new Event("UserCreatedV2", data);
    }
    return oldEvent;
}
```

Все reads проходят через upcaster. Aggregate reconstruction всегда с актуальной schema, независимо от того когда events написаны.

**Event versioning**. Каждый event type имеет version. UserCreatedV1, UserCreatedV2, UserCreatedV3. Aggregate знает как обрабатывать все версии.

**Copy-transform**. Если нужны более серьёзные изменения — переписать events. Читаем все старые events, создаём новые с новой schema, записываем в новую таблицу или в тот же store как новые события. Старые оставляем для audit.

Правило: старайся не менять existing event types. Лучше добавь новый event с новым именем.

## Когда CQRS/ES действительно нужны

Оба паттерна — мощные, но complex. Применять их надо когда:

**CQRS хорош когда**:
- Reads и writes имеют существенно разные optimization requirements.
- Много разных query patterns на одни данные.
- Read scale существенно больше write scale (10:1+).
- Read models могут быть eventually consistent (задержки OK).

**CQRS плохо когда**:
- Обычный CRUD с одним view модели.
- Read требует strong consistency (нельзя терпеть задержку).
- Команда маленькая, complexity перевешивает пользу.

**Event Sourcing хорош когда**:
- Audit trail критически важен (финансы, healthcare, юридические системы).
- Бизнесу нужна возможность time travel — что было в системе на такой-то момент.
- Ретроактивные аналитические возможности важны.
- Domain хорошо ложится на event thinking (natural events).

**Event Sourcing плохо когда**:
- Обычная CRUD без audit requirements.
- Domain плохо ложится на events (что state, а не sequence of changes).
- Команда без опыта в event-driven паттернах.
- Performance sensitive queries по историческим данным.

Для КНП: Event Sourcing вероятно оправдан для обработки налоговых форм (полный audit trail подачи и корректировок формы — regulatory требование). CQRS — для отчётности (write model normalized, read model materialized aggregate). Для типичных CRUD-операций (справочные данные, конфиги) — обычная модель.

## Практические грабли

Финальный список ошибок, которых стоит избегать.

**Overuse of ES**. Event Sourcing для всего — dying by a thousand cuts. Complexity распределяется на всю систему, продуктивность падает. Применять только там где реально нужно.

**Too many event types**. Каждое поле в event vs новый event type — trade-off. Начинающие делают слишком fine-grained events (EmailChanged, PhoneChanged, AddressChanged...), потом их сотни. Better: UserProfileUpdated с payload.

**Event as method call**. Events должны быть про **что произошло** (past tense). "OrderShipped", "PaymentReceived". Не "ShipOrder" (что-то делать) — это command.

**Ignoring event ordering**. Events должны обрабатываться в правильном порядке. Kafka partitioning by aggregate_id гарантирует order per aggregate. Cross-aggregate order — обычно не важен, но если важен — сложно.

**Missing idempotency**. Consumer projection может получить event дважды (at-least-once delivery). Идемпотентно обрабатывать — inbox pattern.

**Big events**. Event payload огромный (весь aggregate). Медленно писать, медленно читать. Лучше — только изменения (delta), состояние reconstructуется из последовательности.

**No snapshotting strategy**. С миллионами events replay становится невыносимым. Snapshots обязательны для активных aggregates.

## Заключение

CQRS и Event Sourcing — мощные паттерны, меняющие сам подход к моделированию. CQRS разделяет read и write модели, позволяя каждой быть оптимальной под её задачу. Event Sourcing делает последовательность событий source of truth, открывая аудит, time travel, retroactive analytics.

CQRS реализуется как: write model normalized для integrity + write handlers, publish events. Read models denormalized для конкретных queries, updated через event handlers. Разные projections для разных query patterns, могут быть в разных технологиях. Синхронизация через events, eventual consistency между write и read.

Event Sourcing: события хранятся в event store, aggregate state — результат fold всех events. Snapshots для performance (не replay всё). Optimistic concurrency через unique constraint на (aggregate_id, version). Upcasting для schema evolution.

Комбинация ES + CQRS — стандартная. Event store как write model, projections как read models. Kafka как event bus между ними.

PostgreSQL — достаточная база для Event Store в большинстве случаев. Специализированные EventStoreDB для очень больших систем.

Не применяй эти паттерны везде. Обычный CRUD не требует их сложности. Applications — audit-critical (финансы, healthcare, regulatory), где ценность полного event log перевешивает сложность.

Для КНП правильный use case Event Sourcing — обработка налоговых форм (кто, когда, что подавал, все корректировки). Все действия становятся events, audit trail complete. CQRS для отчётности — read models pre-materialized под конкретные отчёты, обновляются через события.

Дальше — практика. Реализуй простую Event-Sourced систему (User Registration: UserRegistered, UserVerified, UserDeactivated). PostgreSQL как event store, простая projection в отдельной таблице. Проиграй все events, восстанови текущий state. Добавь snapshots. Потом добавь второй projection (например, search index). Каждый шаг открывает новые возможности и ограничения.

На этом заканчиваем большую серию файлов по PostgreSQL и базам данных вообще. От архитектуры DBMS через partitioning, replication, backup, security, monitoring до CQRS/ES — покрыты все основные темы, которые senior-инженер должен понимать для работы с реляционными базами в production. Практика — единственный способ действительно освоить эти концепции, и каждый описанный паттерн стоит попробовать самостоятельно на простом примере, прежде чем применять в production.
