# 32. Транзакции: ACID, isolation levels, propagation

Всё что нужно знать про транзакции до того, как разбирать `@Transactional`.

---

## 1. Что такое транзакция

**Транзакция** — логическая единица работы с БД. Набор операций, которые должны выполниться **как одна**: либо все, либо ни одна.

Классический пример — банковский перевод:
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;   -- списали у Alice
UPDATE accounts SET balance = balance + 100 WHERE id = 2;   -- зачислили Bob
COMMIT;
```

Если между двумя UPDATE упадёт БД — деньги «потерялись». Транзакция гарантирует: либо оба UPDATE применены (COMMIT), либо ни один (ROLLBACK).

---

## 2. ACID

Четыре свойства правильной транзакции.

### 2.1 Atomicity (атомарность)

«Всё или ничего». Транзакция либо вся зафиксирована, либо вся откачена.

Реализация: **WAL** (Write-Ahead Log). Прежде чем изменение попадёт в таблицу — пишется в лог. При crash в середине tx — undo из лога.

### 2.2 Consistency (согласованность)

БД до транзакции валидна → БД после транзакции валидна. Все constraints (FK, UNIQUE, CHECK) соблюдены.

**Не** гарантирует бизнес-логическую консистентность (это уже приложение).

### 2.3 Isolation (изоляция)

Параллельные транзакции не мешают друг другу. Строгая изоляция — как будто транзакции выполнены последовательно.

**Уровни изоляции** — компромисс между строгостью и performance (см. §4).

### 2.4 Durability (долговечность)

После COMMIT изменения гарантированно на диске, переживут падение.

Реализация: **fsync** WAL. Commit возвращает OK только когда `fsync()` завершился.

---

## 3. Проблемы конкурентного доступа

Прежде чем говорить про уровни — что вообще может пойти не так.

### 3.1 Dirty read

T1 изменил строку, но не закоммитил.
T2 читает — видит незакоммиченное изменение.
T1 откатывает.
T2 «поверила» в то, чего не было.

```
T1: UPDATE accounts SET balance = 200 WHERE id = 1;
T2: SELECT balance FROM accounts WHERE id = 1;   -- видит 200
T1: ROLLBACK;                                     -- balance был 100
```

### 3.2 Non-repeatable read

T1 читает строку.
T2 обновляет и коммитит.
T1 читает ту же строку — другое значение.

```
T1: SELECT balance FROM accounts WHERE id = 1;   -- 100
T2: UPDATE accounts SET balance = 200 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;   -- 200 (изменилось в той же tx!)
```

### 3.3 Phantom read

T1 читает набор строк по условию.
T2 вставляет новую строку, попадающую под условие.
T1 повторяет — появилась «новая» строка.

```
T1: SELECT COUNT(*) FROM accounts WHERE balance > 100;   -- 5
T2: INSERT INTO accounts VALUES (10, 200); COMMIT;
T1: SELECT COUNT(*) FROM accounts WHERE balance > 100;   -- 6
```

Отличие от non-repeatable — тут не изменение, а появление/исчезновение строк по условию.

### 3.4 Lost update

Две транзакции читают, обе меняют, обе коммитят. Первое изменение потеряно.

```
T1: SELECT balance FROM accounts WHERE id = 1;   -- 100
T2: SELECT balance FROM accounts WHERE id = 1;   -- 100
T1: UPDATE ... SET balance = 100 + 50; COMMIT;   -- 150
T2: UPDATE ... SET balance = 100 - 20; COMMIT;   -- 80  ← +50 потеряно!
```

Решение — optimistic (`@Version`) или pessimistic (`SELECT FOR UPDATE`) locking.

### 3.5 Write skew

Специфичная проблема serializable. Две транзакции читают один и тот же набор, каждая обновляет свою часть, вместе нарушают инвариант.

Классический пример — дежурство врачей: правило «минимум один врач на дежурстве». T1 снимает врача A (видит что B на дежурстве). T2 снимает врача B (видит что A на дежурстве). Обе коммитятся — 0 врачей.

---

## 4. Уровни изоляции (SQL стандарт)

Компромисс между строгостью и производительностью.

| Уровень | Dirty read | Non-repeatable | Phantom | Lost update |
|---|---|---|---|---|
| **READ UNCOMMITTED** | ✅ | ✅ | ✅ | ✅ |
| **READ COMMITTED** | ❌ | ✅ | ✅ | ✅ |
| **REPEATABLE READ** | ❌ | ❌ | ✅ (стандарт)<br>❌ (PG!) | ❌ (PG) |
| **SERIALIZABLE** | ❌ | ❌ | ❌ | ❌ |

✅ = проблема возможна. ❌ = не возможна.

### 4.1 READ UNCOMMITTED

Видит всё, включая незакоммиченное. Максимально быстро, максимально небезопасно.

**PostgreSQL не поддерживает** — молча повышает до READ COMMITTED.

Использовать только для аналитики где неточность допустима.

### 4.2 READ COMMITTED (default для PG)

Видит только закоммиченное. Каждый SELECT видит свежий snapshot.

Non-repeatable и phantom всё ещё возможны (два SELECT в одной tx могут увидеть разные данные).

**По умолчанию в PostgreSQL** — самый практичный компромисс.

### 4.3 REPEATABLE READ

Snapshot фиксируется на первом SELECT. Внутри tx все повторные чтения одинаковы.

Стандартный SQL позволяет phantom read. **PostgreSQL реализует более строго** — фактически даёт snapshot isolation, phantom тоже недоступны.

Используется когда нужны консистентные множественные чтения в одной tx (например, отчёты).

### 4.4 SERIALIZABLE

Полная изоляция. Как будто транзакции выполнены последовательно.

**Cost**: PG использует SSI (Serializable Snapshot Isolation) — при обнаружении конфликта одна из tx получает `40001 could not serialize access` → приложение должно retry.

Медленнее (много retry на нагрузке), но самая правильная семантика.

Использовать когда критичен write skew.

### 4.5 Что выбрать

- **READ COMMITTED** — default, устраивает 95% случаев.
- **REPEATABLE READ** — консистентные множественные чтения (отчёты, batch).
- **SERIALIZABLE** — только когда логика чувствительна к write skew И готов handle retry.

В ИСНА (memory `knp-e2e-runner-hikari-isolation-poisoning`) — стандарт **READ COMMITTED** явно, иначе `isolation=-1` (opt-in) может отравить пул PgBouncer.

---

## 5. Как реализуются уровни

### 5.1 Lock-based (MySQL InnoDB)

Явные блокировки строк / диапазонов. SELECT ставит shared lock, UPDATE — exclusive. Плохо: contention.

### 5.2 MVCC (PostgreSQL, Oracle)

Многоверсионный. Читатели видят свою версию, не блокируют писателей. Каждый tuple имеет xmin/xmax (см. `28-postgresql-internals.md`).

Snapshot берётся:
- В READ COMMITTED — на каждый statement.
- В REPEATABLE READ / SERIALIZABLE — один на всю tx.

---

## 6. Propagation — как транзакции комбинируются

Что происходит когда транзакционный метод A вызывает транзакционный метод B?

Ответ зависит от **propagation** метода B (значение по умолчанию — REQUIRED).

### 6.1 REQUIRED (default)

Если есть внешняя транзакция — участвовать. Нет — создать новую.

```
A (@Transactional)
  ├─ B (@Transactional REQUIRED)     ← участвует в A's tx
  ...
```

Один commit, один rollback. Внутренний rollback → внешняя тоже упадёт (mark as rollback-only).

### 6.2 REQUIRES_NEW

**Всегда новая tx**. Внешняя приостанавливается (suspended) на время выполнения.

```
A (@Transactional)                    tx1: BEGIN
  ├─ B (@Transactional REQUIRES_NEW)  tx1: SUSPEND
      ...                             tx2: BEGIN
                                      tx2: COMMIT
                                      tx1: RESUME
  ...                                 tx1: COMMIT
```

Использование: независимый audit-лог (упало приложение → аудит-запись остаётся).

**Кавет**: требует два connection в пуле одновременно! Может привести к deadlock пула.

### 6.3 NESTED

Savepoint внутри внешней транзакции. Rollback внутренней возвращает к savepoint, внешняя продолжается.

```
A (@Transactional)              tx: BEGIN
  ├─ B (@Transactional NESTED)  tx: SAVEPOINT sp1
      throw                      tx: ROLLBACK TO sp1
  continue                       tx: (working)
                                 tx: COMMIT
```

Требует поддержки savepoint (PG — да).

Использование: попытка операции с возможным откатом без потери всей tx.

### 6.4 MANDATORY

Требует внешнюю транзакцию. Нет → `IllegalTransactionStateException`.

Использование: метод «строго часть чьей-то tx».

### 6.5 SUPPORTS

Если есть — участвует; нет — работает без tx.

Использование: read-методы, которые могут вызываться и с tx, и без.

### 6.6 NOT_SUPPORTED

Если есть — suspend; работает без tx.

Использование: долгие операции которые не должны блокировать tx (audit, аналитика).

### 6.7 NEVER

Если есть — исключение. Работает только без tx.

Использование: специфичные джобы, не должны быть в tx.

### 6.8 Таблица

| Propagation | Внутренняя tx | Внешняя tx |
|---|---|---|
| REQUIRED | участвует | — |
| REQUIRES_NEW | новая | suspend |
| NESTED | savepoint | — |
| MANDATORY | участвует | ошибка если нет |
| SUPPORTS | участвует | работает без |
| NOT_SUPPORTED | работает без | suspend |
| NEVER | работает без | ошибка если есть |

---

## 7. XA / distributed transactions (2PC)

Что делать когда две БД или БД + брокер должны фиксироваться атомарно?

### 7.1 Two-Phase Commit (2PC)

Координатор + участники.

Фаза 1 — **prepare**:
- Координатор → всем: «готовы?».
- Каждый участник записывает изменения в prepared state (не коммитит).
- Отвечает «yes / no».

Фаза 2 — **commit / abort**:
- Если все «yes» → координатор → всем: «commit».
- Если хоть один «no» → всем «abort».

### 7.2 XA (X/Open XA)

Стандарт 2PC. **XA-resource** — БД / MQ / etc, поддерживающий XA-протокол.

Java: **JTA (Java Transaction API)** — интерфейс для управления XA. `UserTransaction`, `TransactionManager`.

Реализации в JEE-сервере: Atomikos, Bitronix, Narayana. В Spring Boot — вручную настраивать.

### 7.3 Почему не используют

- **Медленно** — 2 round-trip.
- **Блокирующе** — если координатор упал между фазами, ресурсы залипли в prepared.
- **Сложно** — требует правильной настройки, восстановления после падений.
- **Не масштабируется** — плохо в микросервисах.

### 7.4 Альтернатива — Saga

Compensation-based. Каждая tx локальная (в одной БД), при ошибке — вызываются compensating actions.

Пример order flow:
```
1. CreateOrder (order-service, local tx)
2. ReserveInventory (inventory-service, local tx)
3. ChargePayment (payment-service, local tx)

При ошибке на шаге 3:
- RefundPayment (compensation)
- ReleaseInventory (compensation)
- CancelOrder (compensation)
```

Реализации: Orchestration (центральный оркестратор) vs Choreography (события).

### 7.5 Outbox pattern

Атомарный commit БД + публикация в брокер без XA:

1. В одной tx: сохранить бизнес-данные + запись в `outbox` таблицу.
2. Отдельный job читает `outbox` → публикует в Rabbit/Kafka → удаляет запись.

Гарантия: если commit прошёл — outbox запись есть → рано или поздно опубликуется.

**Наиболее практичный подход** для микросервисов вместо XA.

---

## 8. Optimistic vs Pessimistic locking

### 8.1 Pessimistic (`SELECT FOR UPDATE`)

Явно захватить блокировку строки. Другие ждут.

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;   -- lock row
-- ... работа ...
UPDATE accounts SET balance = ... WHERE id = 1;
COMMIT;   -- lock released
```

Плюсы:
- Гарантия — никто не изменит.
- Простая логика.

Минусы:
- Блокирует других.
- Deadlock возможен.
- Плохо масштабируется.

### 8.2 Optimistic (`@Version`)

Проверка версии при UPDATE.

```sql
UPDATE accounts SET balance = ..., version = version + 1
    WHERE id = 1 AND version = 5;
-- если 0 rows updated → кто-то опередил → retry
```

В JPA — автоматически через `@Version` поле.

Плюсы:
- Не блокирует читателей.
- Хорошо масштабируется.
- Быстро.

Минусы:
- Приложение должно handle retry.
- Работает только для «редко конфликтующих» сценариев.

### 8.3 Что выбрать

- **Optimistic** — обычные CRUD, редкие конфликты.
- **Pessimistic** — счётчики, deposits, финансовые операции.

---

## 9. Транзакции в микросервисах

### 9.1 Локальные

Внутри одного сервиса + одна БД — обычная tx. Ничего сложного.

### 9.2 Между сервисами — Saga / Outbox

XA практически не используется. Всё через events / compensations.

### 9.3 Между БД + брокер (Kafka/Rabbit)

**Outbox** — уже обсуждали.

Kafka имеет свои transactions (см. `42-kafka-prod`) — но с БД координацию всё равно через outbox.

### 9.4 Идемпотентность

Retry делает атомарность условной. Consumer должен быть идемпотентен (см. `21-rabbitmq-delivery-guarantees.md`).

---

## 10. Реальные примеры

### 10.1 Правильный банк-перевод (одна БД)

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- проверка баланса
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

Pessimistic lock. `READ COMMITTED` достаточно.

### 10.2 Правильный банк-перевод через events (микросервисы)

1. `POST /transfer` → account-service local tx: списание с account 1 + INSERT в outbox `TransferInitiated`.
2. Job публикует `TransferInitiated` в Kafka.
3. account-service (или другой) consumer → local tx: зачисление на account 2 + INSERT `TransferCompleted`.
4. Job публикует `TransferCompleted`.

Компенсация: при неудаче зачисления → publish `TransferFailed` → account-service возвращает деньги.

---

## 11. Собесные вопросы

1. **Что такое ACID?** — Atomicity/Consistency/Isolation/Durability.
2. **Уровни изоляции SQL?** — READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE.
3. **Что такое dirty read?** — Чтение незакоммиченных изменений.
4. **Non-repeatable vs phantom read?** — Non-repeatable — изменённая строка; phantom — появилась/пропала по условию.
5. **Default уровень PG?** — READ COMMITTED.
6. **Почему PG REPEATABLE READ строже стандарта?** — MVCC snapshot, phantom тоже недоступны.
7. **Что такое lost update, как избежать?** — Optimistic (@Version) или pessimistic (SELECT FOR UPDATE).
8. **Propagation типы?** — REQUIRED, REQUIRES_NEW, NESTED, MANDATORY, SUPPORTS, NOT_SUPPORTED, NEVER.
9. **REQUIRED vs REQUIRES_NEW?** — REQUIRED = участвует или создаёт; REQUIRES_NEW = всегда новая (suspend внешней).
10. **NESTED — как?** — Savepoint внутри внешней tx; rollback возвращает к savepoint.
11. **Что такое 2PC / XA?** — Distributed tx: prepare + commit; медленно, редко используется.
12. **Что такое Saga?** — Compensation-based distributed tx; локальные tx + compensating actions.
13. **Что такое outbox pattern?** — Атомарный commit БД + запись в outbox → job публикует.
14. **Optimistic vs pessimistic locking — когда что?** — Optimistic для редких конфликтов (CRUD); pessimistic для критичных (счётчики).
15. **Что такое MVCC?** — Многоверсионный concurrency control; читатели не блокируют писателей.

---

## Итог

- **ACID** — 4 гарантии транзакций.
- **4 уровня изоляции**, компромисс скорость/строгость. PG default = READ COMMITTED.
- **MVCC** в PG — читатели не блокируют писателей.
- **7 propagation** типов — REQUIRED default, REQUIRES_NEW / NESTED для особых случаев.
- **XA / 2PC** — избегать в микросервисах. Используй Saga + Outbox.
- **Optimistic locking** (`@Version`) — стандарт для UPDATE.
- **Идемпотентность** = обязательное свойство consumer'ов.

Следующий — `33-transactional-internals.md`.
