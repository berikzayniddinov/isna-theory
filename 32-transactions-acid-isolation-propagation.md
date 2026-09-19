# 32. Транзакции: ACID, isolation levels, propagation

## Что такое транзакция

Транзакция это логическая единица работы с базой данных. Набор операций которые должны выполниться как одно целое — либо все успешно, либо ни одна не применяется. Фундаментальная абстракция обеспечивающая consistency данных при concurrent modifications.

Классический пример банковского перевода иллюстрирует необходимость транзакций. Списываем 100 у Алисы, зачисляем 100 Бобу — это две операции которые должны happen атомарно. Если между ними падает БД, деньги «потеряются» — списаны у одного, не зачислены другому. Транзакция гарантирует что либо обе UPDATE применились (COMMIT) либо ни одна не применилась (ROLLBACK).

Классический SQL для перевода:
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

BEGIN открывает транзакцию. Между BEGIN и COMMIT/ROLLBACK находится transaction scope. COMMIT фиксирует все изменения атомарно. ROLLBACK откатывает — как будто ничего не было.

## ACID свойства

Правильная транзакция обладает четырьмя свойствами известными как ACID. Каждое имеет специфический смысл и специфический механизм реализации.

Atomicity (атомарность) — «всё или ничего». Транзакция либо вся зафиксирована либо вся откачена, никаких partial states. Реализация в PostgreSQL через Write-Ahead Log (WAL). Прежде чем изменение попадёт в actual data files оно пишется в WAL. При crash в середине транзакции — recovery может undo изменения используя WAL. Гарантия что незакоммиченные изменения не остаются в БД.

Consistency (согласованность) означает что БД до транзакции была валидна и БД после транзакции остаётся валидной. Все constraints (FOREIGN KEY, UNIQUE, CHECK, NOT NULL) соблюдены. Важное уточнение — consistency в ACID это database constraints consistency, не бизнес-логическая консистентность. Приложение должно обеспечивать бизнес-инварианты самостоятельно. БД проверяет только structural constraints.

Isolation (изоляция) — параллельные транзакции не мешают друг другу. Строгая изоляция как будто транзакции выполнены последовательно даже если фактически они выполняются concurrently. Уровни изоляции представляют собой компромисс между строгостью гарантий и performance. Разбор различных уровней ниже.

Durability (долговечность) — после COMMIT изменения гарантированно на диске и переживут падение сервера. Реализация через fsync WAL — commit возвращает OK клиенту только после успешного flush WAL записи на диск. Настройка fsync=on обязательна для реальной durability. fsync=off быстрее но может потерять последние committed transactions при crash.

## Проблемы конкурентного доступа

Прежде чем обсуждать уровни изоляции полезно понять что может пойти не так при concurrent transactions.

Dirty read происходит когда одна транзакция читает изменения другой ещё не committed транзакции. T1 обновляет строку, T2 читает — видит незакоммиченное значение, T1 делает ROLLBACK. T2 «поверила» в данные которых на самом деле не было. Может привести к incorrect business decisions на основе несуществующих данных.

Non-repeatable read это когда одна транзакция читает одну и ту же строку дважды и получает разные значения. T1 читает строку — видит balance=100. T2 обновляет и коммитит balance=200. T1 читает опять — видит balance=200. Внутри одной транзакции данные «изменились» что нарушает consistency перспективы транзакции.

Phantom read похожа на non-repeatable но касается набора строк по условию а не конкретной строки. T1 выполняет SELECT COUNT(*) WHERE balance > 100 — получает 5. T2 вставляет новую строку с balance=200 и коммитит. T1 повторяет запрос — теперь 6 строк. Новая «phantom» строка появилась в результате того же query. Отличие от non-repeatable — не изменение существующей строки, а появление или исчезновение строк по условию.

Lost update происходит когда две транзакции читают, обе изменяют, обе коммитят — одно изменение потеряно. T1 читает balance=100. T2 читает balance=100 (в другой сессии). T1 обновляет balance=150 и коммитит. T2 обновляет balance=80 (на основе своего чтения 100 минус 20) и коммитит. Итог 80 вместо ожидаемого 130. Изменение T1 полностью «затерто». Решается через optimistic locking (@Version) или pessimistic (SELECT FOR UPDATE).

Write skew это специфичная проблема serializable уровня. Две транзакции читают один и тот же набор данных, каждая изменяет свою часть, вместе нарушают invariant. Классический пример — правило «минимум один врач на дежурстве». T1 читает список on-call врачей видя A и B on-call, снимает A потому что B on-call. T2 параллельно делает то же самое — видит A и B on-call, снимает B потому что A on-call. Обе коммитятся — 0 on-call врачей, invariant нарушен.

## Уровни изоляции SQL стандарт

SQL стандарт определяет четыре уровня изоляции представляющие trade-offs между строгостью и performance.

| Уровень | Dirty read | Non-repeatable | Phantom | Lost update |
|---|---|---|---|---|
| READ UNCOMMITTED | возможен | возможен | возможен | возможен |
| READ COMMITTED | нет | возможен | возможен | возможен |
| REPEATABLE READ | нет | нет | возможен (стандарт) / нет (PG) | нет (PG) |
| SERIALIZABLE | нет | нет | нет | нет |

READ UNCOMMITTED разрешает всё включая dirty reads. Максимально быстро потому что не требует consistency guarantees. Максимально небезопасно. PostgreSQL не поддерживает этот уровень — при попытке установить его молча повышает до READ COMMITTED. Использование ограничено analytical queries где неточность допустима.

READ COMMITTED это default в PostgreSQL. Видит только committed данные. Каждый SELECT видит свежий snapshot committed данных на момент SELECT. Non-repeatable и phantom reads всё ещё возможны потому что несколько SELECT в одной transaction могут видеть разные данные если другие transactions committed между ними. Самый практичный компромисс для большинства сценариев.

REPEATABLE READ фиксирует snapshot на первом SELECT в транзакции. Все повторные чтения внутри транзакции возвращают согласованные данные. SQL стандарт разрешает phantom reads в этом уровне. PostgreSQL реализует более строго — фактически даёт snapshot isolation где phantom reads тоже недоступны. Полезен для reports и batch операций требующих consistent multiple reads.

SERIALIZABLE даёт полную изоляцию — как будто transactions выполнены последовательно. PostgreSQL использует SSI (Serializable Snapshot Isolation) — при обнаружении conflict одна из transactions получает ошибку 40001 could not serialize access. Приложение должно retry операцию. Медленнее (много retries под нагрузкой) но самая правильная семантика для сценариев чувствительных к write skew.

Практический выбор. READ COMMITTED устраивает 95 процентов случаев. REPEATABLE READ для консистентных multiple чтений — отчёты, batch. SERIALIZABLE только когда логика чувствительна к write skew и приложение готово handle retry.

В КНП стандарт READ COMMITTED явно установленный. Memory кейс knp-e2e-runner-hikari-isolation-poisoning показал что opt-in isolation равное -1 может отравить pool PgBouncer subsequent transactions в pool получают wrong isolation. Явное указание уровня обязательно.

## Реализация уровней изоляции

Существует два основных подхода к реализации isolation levels.

Lock-based подход используемый в MySQL InnoDB классически. Explicit locks на строки или ranges. SELECT ставит shared lock, UPDATE берёт exclusive. Concurrent readers ok но writer блокируется если читатели держат shared lock. Приводит к contention — много waits и потенциально deadlocks.

MVCC подход используемый в PostgreSQL и Oracle. Многоверсионный concurrency control. Читатели видят свою snapshot version данных, не блокируют писателей. Каждый tuple имеет xmin/xmax системные columns определяющие видимость для конкретных transactions.

Snapshot берётся в разные моменты в зависимости от уровня. В READ COMMITTED snapshot обновляется на каждый statement — каждый SELECT видит свежие committed данные. В REPEATABLE READ и SERIALIZABLE один snapshot на всю transaction — все statements видят consistent data.

MVCC даёт лучшую concurrency чем lock-based. Читатели никогда не блокируют писателей. Писатели блокируют других писателей той же row. Читатели не блокируют друг друга. Trade-off — накопление dead tuples требующее periodic VACUUM.

## Propagation комбинирование транзакций

Что происходит когда транзакционный метод A вызывает транзакционный метод B? Ответ зависит от propagation атрибута B. По умолчанию REQUIRED но существует семь опций для разных сценариев.

REQUIRED это default. Если есть внешняя транзакция — метод участвует в ней. Нет внешней — создаётся новая. Один commit на всю цепочку, один rollback. Внутренний rollback метки транзакцию как rollback-only — при попытке commit внешней получается UnexpectedRollbackException.
```
A (@Transactional)
  ├─ B (@Transactional REQUIRED)     ← участвует в A's tx
  ├─ C (@Transactional REQUIRED)     ← участвует в A's tx
```

REQUIRES_NEW всегда создаёт новую транзакцию. Внешняя приостанавливается (suspended) на время выполнения внутренней. Полезно для independent audit logs — даже если main transaction откатывается, audit entry остаётся:
```
A (@Transactional)                    tx1: BEGIN
  ├─ B (@Transactional REQUIRES_NEW)  tx1: SUSPEND
      ...                             tx2: BEGIN
                                      tx2: COMMIT
                                      tx1: RESUME
  ...                                 tx1: COMMIT
```

Важный caveat — требует два connection в pool одновременно (suspended tx1 держит свой connection, new tx2 берёт другой). Может привести к pool exhaustion при массовом использовании.

NESTED использует savepoint внутри внешней транзакции. Rollback внутренней возвращает к savepoint, внешняя продолжается. Реализуется через SAVEPOINT SQL команду:
```
A (@Transactional)              tx: BEGIN
  ├─ B (@Transactional NESTED)  tx: SAVEPOINT sp1
      throw                      tx: ROLLBACK TO sp1
  continue                       tx: (продолжаем)
                                 tx: COMMIT
```

Требует поддержку savepoints БД (PostgreSQL поддерживает). Отличие от REQUIRES_NEW — работает в одной transaction (один connection), при rollback внешней nested тоже откатывается. Полезен для «попытки с возможностью отката» без потери всей transaction.

MANDATORY требует существующей внешней транзакции. Если нет — бросает IllegalTransactionStateException. Использование когда метод «строго часть чьей-то транзакции» — программное указание что метод не должен вызываться самостоятельно.

SUPPORTS работает как в транзакции так и без. Если есть — участвует. Нет — работает без транзакции. Использование для read методов которые могут вызываться и в transactional и в non-transactional контекстах.

NOT_SUPPORTED всегда работает без транзакции. Если есть внешняя — suspends её на время. Использование для долгих операций которые не должны быть в transaction — audit logging, analytics, external calls.

NEVER работает только без транзакции. Если есть — бросает exception. Использование для специфических jobs которые не должны выполняться в transaction context.

Сводная таблица behavior:

| Propagation | Внутренняя tx | Внешняя tx |
|---|---|---|
| REQUIRED | участвует | использует существующую |
| REQUIRES_NEW | новая | suspend |
| NESTED | savepoint | использует существующую |
| MANDATORY | участвует | error если нет |
| SUPPORTS | участвует или без | ok либо без |
| NOT_SUPPORTED | работает без | suspend |
| NEVER | работает без | error если есть |

## Distributed transactions и 2PC

Что делать когда две БД или БД плюс message broker должны фиксироваться атомарно? Классическое решение — Two-Phase Commit (2PC).

2PC работает через координатор и участников. Фаза prepare — координатор запрашивает всех «готовы commit?». Каждый участник записывает изменения в prepared state (persist but not committed), отвечает yes или no. Фаза commit — если все ответили yes координатор говорит всем commit, если хоть один no — говорит всем abort.

XA (X/Open XA) это стандарт для 2PC. XA-resource это database, message queue или другой resource поддерживающий XA protocol. Java Transaction API (JTA) это interface для управления XA transactions. UserTransaction, TransactionManager для application, XAResource для resource providers.

Реализации XA в JEE серверах — Atomikos, Bitronix, Narayana. В Spring Boot требует manual configuration. Не common practice.

Почему 2PC редко используют в микросервисах. Медленно — два round-trip минимум для commit. Блокирующий — если координатор упал между фазами, ресурсы застряли в prepared state до восстановления координатора. Сложно — требует правильной конфигурации всех участников, recovery процедур. Плохо масштабируется — координатор bottleneck.

Saga pattern как альтернатива для microservices. Compensation-based подход. Каждый шаг это local transaction в одном сервисе. При failure на любом шаге — вызываются compensating actions отменяющие предыдущие steps.

Пример order flow. CreateOrder в order-service local tx. ReserveInventory в inventory-service local tx. ChargePayment в payment-service local tx. При failure на шаге 3 — RefundPayment (compensation), ReleaseInventory (compensation), CancelOrder (compensation).

Реализации Saga. Orchestration — центральный оркестратор направляет steps. Choreography — сервисы координируются через events без central controller.

Outbox pattern это practical подход для atomic commit БД plus message broker без XA. В одной transaction — сохранить бизнес-данные plus запись в outbox таблицу. Отдельный job читает outbox — публикует в Rabbit/Kafka — удаляет запись. Гарантия — если commit прошёл то outbox запись есть, публикация состоится eventually. Наиболее практичный подход для микросервисов.

## Optimistic против Pessimistic locking

Два подхода к handling concurrent updates.

Pessimistic locking через SELECT FOR UPDATE явно захватывает lock на строку. Другие транзакции ждут пока lock не освободится. Простая семантика — гарантия что никто не изменит между read и write. Deadlocks возможны при complex lock acquisition patterns. Плохо масштабируется потому что concurrent writes serialized.

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- работа с данными
UPDATE accounts SET balance = ... WHERE id = 1;
COMMIT;
```

Optimistic locking через @Version добавляет version column к entity. При UPDATE проверяется version — если не совпадает значит кто-то опередил, retry требуется. В JPA автоматически через @Version аннотацию.

```sql
UPDATE accounts SET balance = ..., version = version + 1
    WHERE id = 1 AND version = 5;
-- если 0 rows updated — конфликт, retry
```

Плюсы optimistic. Не блокирует читателей — concurrent reads без issues. Хорошо масштабируется — только conflicting writes требуют retry. Быстро когда конфликты редки.

Минусы. Приложение должно обрабатывать retry — сложность в коде. Работает только для сценариев с редкими конфликтами — если конфликты частые overhead retry превышает savings.

Практический выбор. Optimistic для обычных CRUD с редкими конфликтами. Pessimistic для критичных операций вроде counters, deposits, financial transactions где ordering критичен и корректность важнее throughput.

## Транзакции в микросервисах

Локальные транзакции внутри одного сервиса и одной БД — обычные @Transactional, ничего сложного. Всё описанное выше применимо напрямую.

Между сервисами XA практически не используется. Saga или Outbox pattern стандартный подход. Local transactions in each service плюс coordination через events.

Между БД и message broker (Kafka или Rabbit) обычно Outbox. Kafka имеет свои transactions но координация с БД всё равно через outbox для reliability.

Идемпотентность как обязательное свойство. Retry делает атомарность условной — операция может выполниться несколько раз. Consumer должен быть идемпотентен через unique keys или conditional updates. Без идемпотентности любая retry ситуация приводит к duplication.

## Реальные примеры

Правильный банк-перевод в одной БД:
```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- проверка баланса на достаточность
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

Pessimistic lock через FOR UPDATE предотвращает concurrent modifications. READ COMMITTED достаточен потому что операции короткие и на конкретных строках.

Правильный банк-перевод через events для микросервисов. POST /transfer вызывает account-service который делает local tx списание с account 1 плюс INSERT в outbox запись TransferInitiated. Job публикует TransferInitiated в Kafka. account-service (или другой) consumer в local tx делает зачисление на account 2 плюс INSERT TransferCompleted. Job публикует TransferCompleted.

При неудаче зачисления — publish TransferFailed event. account-service обрабатывает event через compensation — возвращает деньги на account 1.

## Итоги

ACID это четыре гарантии transactions — Atomicity через WAL, Consistency через constraints, Isolation через locks или MVCC, Durability через fsync.

Четыре уровня изоляции представляют trade-offs. READ UNCOMMITTED быстрый но небезопасный. READ COMMITTED default практичный. REPEATABLE READ для consistent multiple reads. SERIALIZABLE для write skew scenarios с retry logic.

PostgreSQL MVCC даёт читателям snapshot без блокировок писателей. Реализация через xmin/xmax системные columns и snapshot management per transaction или per statement.

Семь propagation типов покрывают различные сценарии combining transactions. REQUIRED default. REQUIRES_NEW для independent operations. NESTED для savepoint-based error recovery. Остальные для специфических cases.

Distributed transactions через XA и 2PC избегать в микросервисах. Saga plus Outbox pattern предпочтительнее — local transactions plus event-driven coordination.

Optimistic locking через @Version стандарт для UPDATE в concurrent scenarios с редкими конфликтами. Pessimistic locking через SELECT FOR UPDATE для критичных операций с частыми conflicts.

Идемпотентность обязательное свойство в message-based системах. Retry возможен всегда, идемпотентность предотвращает duplication effects.

Дальше — @Transactional изнутри, как Spring реально implements через AOP proxies и PlatformTransactionManager.
