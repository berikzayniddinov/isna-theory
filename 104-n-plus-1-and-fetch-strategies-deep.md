# 104. N+1 проблема и стратегии fetch в Hibernate — deep-dive

## Зачем это знать

N+1 — самая распространённая и самая коварная проблема производительности в JPA/Hibernate приложениях. Она встречается везде, часто скрыта в на первый взгляд невинных строках кода, и остаётся невидимой на dev-стенде с 10 записями в таблице. А потом в проде на 10000 пользователях приложение начинает выдавать 30-секундные timeouts на страницах, которые в UI выглядят простыми. И причина одна: где-то в коде `user.getOrders().forEach(...)` превращается в 10001 SQL-запрос вместо одного.

Есть три причины, зачем копать это глубоко. Первая — 80% медленных endpoints в enterprise Spring Boot приложениях страдают от N+1. Не от плохих SQL-запросов, не от отсутствия индексов, не от медленной БД. От того, что Hibernate за одну «бизнес-операцию» шлёт сотни или тысячи маленьких запросов, каждый из которых сам по себе быстрый (1-5 мс), но суммарно даёт секунды latency.

Вторая — стратегии решения N+1 не эквивалентны. `JOIN FETCH`, `@EntityGraph`, `@BatchSize`, `FetchMode.SUBSELECT`, DTO projections — все они борются с N+1, но по-разному, с разными trade-offs, и с разными подводными камнями. `JOIN FETCH` может создать Cartesian product и duplicate rows. `@BatchSize` работает только для последующих lazy fetches. `FetchMode.SUBSELECT` может выполнить subquery, который сам себя убьёт. Без понимания различий вы будете применять неправильные фиксы к неправильным проблемам.

Третья — многие «фиксы N+1» на самом деле ломают код или создают новые проблемы. Наивное `fetch = FetchType.EAGER` на `@OneToMany` — путь к слезам: Hibernate начнёт загружать все orders при каждом чтении user, независимо от того, нужны они или нет. `JOIN FETCH` на нескольких коллекциях сразу — исключение `MultipleBagFetchException` или Cartesian product с миллионом дубликатов. `@BatchSize(size = 1000)` — может привести к OOM на entity с большим составом.

Мы разберём, что такое N+1 физически — какие запросы генерирует Hibernate и почему. Как lazy loading работает, что такое `PersistentBag`, как triggers инициализация. Все стратегии решения: `JOIN FETCH` (с ловушкой Cartesian product), `@EntityGraph` (JPA стандарт, более гибкий, чем JOIN FETCH), `@BatchSize` (batch loading для последующих lazy), `FetchMode.SUBSELECT` (одним subquery для всех), DTO projections (обход entity graph целиком). Отдельно — почему `@OneToMany` лучше держать lazy, а не переключать на eager. Разберём multiple collections problem, unique vs bag, `LinkedHashSet` vs `List`. Пройдёмся по практическим паттернам в Spring Data Repository: `@Query` с fetch, `@EntityGraph` на repository методах, dynamic entity graphs. Диагностика: включение SQL логов, `hibernate.generate_statistics`, p6spy, Datasource Proxy. И под конец — практическое руководство: как найти N+1 в существующем приложении и как правильно его исправить.

## Что такое N+1 — физически

Начнём с примера. Классическая ситуация из проекта КНП: у налогоплательщика (`Taxpayer`) есть много налоговых деклараций (`Declaration`).

```java
@Entity
public class Taxpayer {
    @Id private Long id;
    private String name;
    
    @OneToMany(mappedBy = "taxpayer")   // default: LAZY
    private List<Declaration> declarations;
}

@Entity
public class Declaration {
    @Id private Long id;
    private String type;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "taxpayer_id")
    private Taxpayer taxpayer;
}
```

Код в сервисе:

```java
@Transactional(readOnly = true)
public List<TaxpayerDTO> getAllTaxpayersWithDeclarations() {
    List<Taxpayer> taxpayers = taxpayerRepository.findAll();
    
    return taxpayers.stream()
        .map(t -> new TaxpayerDTO(t.getName(), t.getDeclarations().size()))
        .collect(toList());
}
```

Что происходит в SQL логах:

```sql
-- Первый запрос: получить всех taxpayers
SELECT id, name FROM taxpayers;
-- Вернулось 1000 rows

-- N последующих запросов при обращении к getDeclarations() каждого:
SELECT id, type, taxpayer_id FROM declarations WHERE taxpayer_id = 1;
SELECT id, type, taxpayer_id FROM declarations WHERE taxpayer_id = 2;
SELECT id, type, taxpayer_id FROM declarations WHERE taxpayer_id = 3;
...
SELECT id, type, taxpayer_id FROM declarations WHERE taxpayer_id = 1000;
```

**1 + N = 1 + 1000 = 1001 запрос**. Отсюда название «N+1 problem».

Если каждый запрос занимает 2 мс (round-trip к БД + parse + execute), суммарно **2 секунды**. При 100 таких страниц одновременно — БД захлёбывается от количества connections, HikariCP pool exhausted, HTTP 500.

Тот же результат можно было получить одним запросом:

```sql
SELECT t.id, t.name, count(d.id) AS decl_count
FROM taxpayers t
LEFT JOIN declarations d ON d.taxpayer_id = t.id
GROUP BY t.id, t.name;
```

Один запрос, 5 мс, 400x быстрее. Тот же результат.

Ключевой момент: N+1 не в самом Hibernate. Он в **комбинации lazy loading + iteration по коллекции**. Hibernate загружает `Taxpayer` без деклараций (lazy). Когда ты обращаешься к `taxpayer.getDeclarations()`, Hibernate триггерит запрос только для этого одного taxpayer. Ты в цикле по 1000 taxpayer'ам — 1000 запросов.

## Как lazy loading работает изнутри

Понять механику lazy loading — половина понимания N+1.

Когда Hibernate загружает `Taxpayer`, он смотрит на `@OneToMany`:

- Если `fetch = FetchType.EAGER` — сразу загружает коллекцию.
- Если `fetch = FetchType.LAZY` (default для `@OneToMany`) — вместо реального `List<Declaration>` подставляет **прокси**.

Что такое прокси. Hibernate использует библиотеку bytecode enhancement (ByteBuddy или CGLib) или свои специализированные типы коллекций:

- Для `List` → `PersistentBag` (implements `List`, но без гарантии unique).
- Для `Set` → `PersistentSet`.
- Для `Map` → `PersistentMap`.

Внутри прокси хранит:

- Reference на `Session` (Hibernate persistence context).
- Отметку «загружено ли значение».
- Owner entity (какой Taxpayer владеет).

Первое обращение к любому методу прокси, требующему данных (`size()`, `iterator()`, `contains()`, `get(i)`), триггерит **initialization**:

```java
// Внутри PersistentBag.initialize()
if (!initialized) {
    Session session = getSession();
    if (session == null) {
        throw new LazyInitializationException(
            "could not initialize proxy - no Session");
    }
    // Отправляет SELECT в БД
    List<Declaration> actual = session.createQuery(
        "SELECT d FROM Declaration d WHERE d.taxpayer.id = :id")
        .setParameter("id", ownerId)
        .list();
    this.list = actual;
    initialized = true;
}
```

Именно здесь возникает `LazyInitializationException`: если ты обратился к lazy коллекции **вне транзакции** (Session закрыта), инициализация невозможна.

**Ключевая деталь**: даже такой невинный код триггерит запрос:

```java
if (taxpayer.getDeclarations().isEmpty()) { ... }
```

`isEmpty()` в `PersistentBag` вызывает initialization. Есть оптимизация: `PersistentCollection.wasInitialized()` можно проверить без загрузки. Но 99% кода этого не делает.

## Полная классификация N+1

N+1 возникает не только в `@OneToMany`. Разберём все паттерны.

**`@OneToMany` (коллекция)** — обращение к коллекции lazy-родителя. Пример выше.

**`@ManyToOne` (одиночная ассоциация)** — обращение к lazy-предку из child.

```java
List<Declaration> declarations = declarationRepository.findAll();
for (Declaration d : declarations) {
    d.getTaxpayer().getName();  // N+1: каждый доступ = SELECT taxpayer
}
```

Даже если `@ManyToOne` default fetch — EAGER (в JPA), Hibernate использует специальные оптимизации для eager loads через join. Но если явно указан LAZY (что рекомендуется) — точно N+1.

**`@OneToOne`** — аналогично `@ManyToOne` в терминах N+1.

**Nested associations** — цепочки. `Taxpayer → Declaration → TaxItem` — три уровня lazy. Итерация по taxpayer'ам с обращением к items — N+1 на уровне 2 умножается на N+1 на уровне 3.

**Проекции и вычисленные поля**. Если у entity есть `@Formula` или `@ColumnTransformer` с subquery — N+1 может возникнуть даже без explicit ассоциации.

**Каскад в двух направлениях**. `Taxpayer.declarations` и `Declaration.taxpayer` — bi-directional. Если оба lazy, легко случайно триггернуть N+1 в неожиданном месте (например, `equals()` метод, использующий `taxpayer`).

## Как это выглядит в дев vs проде

На dev-стенде с 10 taxpayer'ами и 5 declarations каждый:

```
1 + 10 = 11 запросов
Каждый ~1 мс на localhost
Общее время: ~15 мс
```

Незаметно. Разработчик не видит проблему.

В проде с 100000 taxpayer'ов:

```
1 + 100000 = 100001 запросов
Каждый ~2 мс к БД в K8s (network latency + query time)
Общее время: 200 секунд
Плюс HikariCP pool exhausted → connections queue → timeout
```

Классическая история: «работало на локале, в проде тормозит».

Правильный подход — включать SQL логирование даже на dev и следить за количеством запросов.

## Стратегия 1: JOIN FETCH

Первый и самый прямой способ решить N+1. Hibernate специфический синтаксис JPQL.

```java
public interface TaxpayerRepository extends JpaRepository<Taxpayer, Long> {
    
    @Query("SELECT t FROM Taxpayer t JOIN FETCH t.declarations")
    List<Taxpayer> findAllWithDeclarations();
}
```

Что происходит в SQL:

```sql
SELECT t.id, t.name, 
       d.id AS d_id, d.type AS d_type, d.taxpayer_id AS d_txid
FROM taxpayers t
INNER JOIN declarations d ON d.taxpayer_id = t.id;
```

Один SQL. Hibernate парсит результат: для каждого row извлекает Taxpayer + Declaration, собирает graph, устанавливает `taxpayer.declarations` list с материализованными entities.

Плюсы:

- Один запрос.
- Полностью инициализированный граф — можно обращаться к `taxpayer.getDeclarations()` без ленивой инициализации.
- Работает вне транзакции (данные уже в памяти).

Минусы:

- **Cartesian product problem**. Если у taxpayer 100 declarations, и вы делаете `JOIN FETCH` на две коллекции — `declarations` и `taxItems` — получите 100 × 500 = 50000 rows на одного taxpayer.

```java
// Плохо! Cartesian product
@Query("SELECT t FROM Taxpayer t 
        JOIN FETCH t.declarations 
        JOIN FETCH t.taxItems")
```

Hibernate детектит это и бросает `MultipleBagFetchException`. Причина: `List` (=`Bag`) не имеет уникальности, невозможно правильно deduplicate. Обходится через `Set` (но тогда порядок теряется) или через `LinkedHashSet` (порядок сохранён, deduplication работает).

- **Duplicate rows**. Один taxpayer с 5 declarations в SQL results — 5 rows с полями taxpayer'а. Hibernate обычно deduplicate'ит через primary key. Но если делать `taxpayerRepo.findAllWithDeclarations()` без `DISTINCT`, count может быть неверный.

```java
@Query("SELECT DISTINCT t FROM Taxpayer t JOIN FETCH t.declarations")
```

`DISTINCT` в Hibernate до 6.0 применялся дважды — на SQL уровне (медленно) и на entity уровне (дедупликация). С Hibernate 6+ есть `passDistinctThrough=false` hint, чтобы не передавать DISTINCT в SQL.

- **Pagination ломается**. `Pageable` + `JOIN FETCH` — Hibernate не может корректно paginate. Логирует warning `HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!` — грузит **всё**, пагинирует в памяти. На больших данных = OOM.

- **Не подходит для optional relationships**. `INNER JOIN` fetch отфильтровывает taxpayer'ов без declarations. Нужен `LEFT JOIN FETCH`.

**Правильная форма для одной коллекции**:

```java
@Query("SELECT DISTINCT t FROM Taxpayer t LEFT JOIN FETCH t.declarations")
List<Taxpayer> findAllWithDeclarations();
```

Для нескольких коллекций одним запросом — не сработает без явного использования `Set` или разбиения на несколько запросов.

## Стратегия 2: @EntityGraph

JPA стандартный механизм для указания, какие ассоциации подгружать. Более гибкий, чем JOIN FETCH.

```java
public interface TaxpayerRepository extends JpaRepository<Taxpayer, Long> {
    
    @EntityGraph(attributePaths = {"declarations"})
    List<Taxpayer> findAll();
    
    @EntityGraph(attributePaths = {"declarations", "declarations.taxItems"})
    Optional<Taxpayer> findWithFullGraphById(Long id);
}
```

Что происходит: Hibernate под капотом добавляет `LEFT JOIN FETCH` для указанных путей. Но с одним преимуществом — не нужно писать JPQL.

Также поддерживает **named entity graphs** через `@NamedEntityGraph` на entity:

```java
@Entity
@NamedEntityGraph(
    name = "Taxpayer.withDeclarationsAndItems",
    attributeNodes = {
        @NamedAttributeNode(value = "declarations", 
                          subgraph = "declarations-subgraph")
    },
    subgraphs = {
        @NamedSubgraph(name = "declarations-subgraph",
                     attributeNodes = @NamedAttributeNode("taxItems"))
    }
)
public class Taxpayer { ... }

// Repository:
@EntityGraph("Taxpayer.withDeclarationsAndItems")
List<Taxpayer> findAll();
```

Плюсы над `JOIN FETCH`:

- Declarative. Проще читать.
- Reusable — один named graph используется в разных методах.
- Работает с method naming conventions Spring Data (`findByName`, `findAllByStatus`) — не нужно писать `@Query`.
- Можно динамически строить graph:

```java
EntityGraph<Taxpayer> graph = em.createEntityGraph(Taxpayer.class);
graph.addAttributeNodes("declarations");
if (needTaxItems) {
    graph.addSubgraph("declarations").addAttributeNodes("taxItems");
}

TypedQuery<Taxpayer> query = em.createQuery("SELECT t FROM Taxpayer t", Taxpayer.class);
query.setHint("javax.persistence.loadgraph", graph);
```

Минусы:

- Те же проблемы, что и `JOIN FETCH`: Cartesian product на multiple collections, pagination.
- `MultipleBagFetchException` тоже возможен.

**Fetch graph vs Load graph**:

- `javax.persistence.fetchgraph` — только атрибуты, указанные в graph, будут eager, всё остальное — lazy.
- `javax.persistence.loadgraph` — атрибуты в graph eager, остальные fetch settings в аннотациях сохраняются.

В 90% случаев используется `loadgraph` (через `@EntityGraph` по умолчанию).

## Стратегия 3: @BatchSize

Совсем другой подход. Вместо одного большого запроса — **батчирование** последующих lazy fetches.

```java
@Entity
public class Taxpayer {
    @OneToMany(mappedBy = "taxpayer")
    @BatchSize(size = 50)
    private List<Declaration> declarations;
}
```

Что делает `@BatchSize(size = 50)`. При обращении к `taxpayer.getDeclarations()` первого taxpayer'а Hibernate триггерит SELECT. Но вместо запроса `WHERE taxpayer_id = 1` он берёт **50 taxpayer'ов из persistence context**, у которых declarations ещё не загружены, и делает:

```sql
SELECT d.* FROM declarations d 
WHERE d.taxpayer_id IN (1, 2, 3, ..., 50);
```

Один запрос — declarations для 50 taxpayer'ов сразу. При итерации по 1000 taxpayer'ов вместо 1000 запросов — 20 запросов. **50x speedup**.

Плюсы:

- Не создаёт Cartesian product — отдельные запросы для коллекций.
- Pagination работает (parent query нормально paginate'ится).
- Multiple collections работают — каждая batch'ится отдельно.
- Работает и на Session-level кэшах — Hibernate знает, какие taxpayer'ы уже в persistence context.

Минусы:

- Не solves проблему полностью — всё равно N/batch_size запросов вместо 1.
- Требует известности размера коллекции. `@BatchSize(size = 1000)` при 1000 entities — один запрос, но если entity редкий — теряется преимущество.
- Работает для lazy fetching, не для eager.

Настройка глобально через `hibernate.default_batch_fetch_size`:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 25
```

Применится ко всем коллекциям и `@ManyToOne` без явного `@BatchSize`.

**Практическое правило**: `@BatchSize` — хороший default для всех `@OneToMany` в проекте. `size` — 10-50 обычно оптимально. Больше — риск IN-clause performance issues (некоторые БД тормозят на IN с тысячами values).

## Стратегия 4: FetchMode.SUBSELECT

Ещё один Hibernate-specific подход:

```java
@Entity
public class Taxpayer {
    @OneToMany(mappedBy = "taxpayer")
    @Fetch(FetchMode.SUBSELECT)
    private List<Declaration> declarations;
}
```

Как работает. Когда любой из ранее загруженных `Taxpayer`'ов триггерит инициализацию `getDeclarations()`, Hibernate выполняет один subquery для **всех** taxpayer'ов, чей ID был в исходном запросе:

```sql
-- Первый запрос:
SELECT * FROM taxpayers WHERE status = 'ACTIVE';

-- При первом обращении к getDeclarations() — один subquery:
SELECT d.* FROM declarations d
WHERE d.taxpayer_id IN (
    SELECT t.id FROM taxpayers t WHERE t.status = 'ACTIVE'
);
```

Все declarations всех taxpayer'ов загружены одним запросом. Все subsequent `getDeclarations()` — no SQL, из persistence context.

Плюсы:

- Только 2 запроса всего (parent + child).
- Не создаёт Cartesian product.
- Работает для нескольких коллекций одновременно.

Минусы:

- Subquery может быть тяжёлый. Если parent query — сложный (много JOIN'ов), Hibernate повторяет его в subquery.
- Не подходит для entity, загружаемого по одному (`findById`) — там нет "batch" оригинального query.
- Может быть неоптимальный план на PostgreSQL — планировщик не всегда знает, что subquery маленький.
- Не работает с pagination (если parent paginate'ится, subquery видит только видимый page? нет, видит весь оригинальный query).

Используется реже, чем `@BatchSize`, но иногда — точное решение.

## Стратегия 5: DTO projections

Кардинально другой подход — не грузить entities вообще. Загружать сразу нужные данные в специальные DTO объекты.

```java
public class TaxpayerSummaryDTO {
    private String name;
    private long declarationCount;
    
    public TaxpayerSummaryDTO(String name, long declarationCount) {
        this.name = name;
        this.declarationCount = declarationCount;
    }
}

public interface TaxpayerRepository extends JpaRepository<Taxpayer, Long> {
    
    @Query("""
        SELECT new kz.isna.knp.dto.TaxpayerSummaryDTO(
            t.name, 
            COUNT(d.id)
        )
        FROM Taxpayer t
        LEFT JOIN t.declarations d
        GROUP BY t.id, t.name
        """)
    List<TaxpayerSummaryDTO> findAllSummaries();
}
```

Один запрос, конкретные поля. Никаких entities, никаких proxies, никаких lazy problems.

Или через Spring Data Projections (интерфейсные):

```java
public interface TaxpayerSummary {
    String getName();
    Long getDeclarationCount();
}

public interface TaxpayerRepository extends JpaRepository<Taxpayer, Long> {
    
    @Query("""
        SELECT t.name AS name, COUNT(d.id) AS declarationCount
        FROM Taxpayer t LEFT JOIN t.declarations d
        GROUP BY t.id, t.name
        """)
    List<TaxpayerSummary> findAllSummaries();
}
```

Spring Data создаст proxy-имплементацию интерфейса.

Плюсы:

- Один запрос, гарантированно.
- Загружаются только нужные колонки — экономия I/O.
- Никакой N+1 никогда.
- Работает с pagination нормально.
- Type-safe (в отличие от Tuple).

Минусы:

- Не entities — нельзя изменить и сохранить.
- Нужно писать DTO класс.
- Для сложных struct'ур со вложенностью — сложнее.

**Правило**: для **read-only** endpoints (списки, детали для отображения) — DTO projections. Для **изменяемых** entities (form editing, deletes) — entities с fetch strategies.

## Multiple collections — MultipleBagFetchException

Классическая ловушка. Смотрим:

```java
@Entity
public class Taxpayer {
    @OneToMany(mappedBy = "taxpayer")
    private List<Declaration> declarations;
    
    @OneToMany(mappedBy = "taxpayer")
    private List<Payment> payments;
}

// Repository:
@Query("SELECT DISTINCT t FROM Taxpayer t 
        LEFT JOIN FETCH t.declarations 
        LEFT JOIN FETCH t.payments")
List<Taxpayer> findAllFullyLoaded();
```

Запуск — исключение:

```
org.hibernate.loader.MultipleBagFetchException: 
cannot simultaneously fetch multiple bags: [taxpayer.declarations, taxpayer.payments]
```

Почему. В Hibernate `List` без `@OrderColumn` — это `Bag` (в терминах Hibernate). Bag — мультимножество без гарантии уникальности. Если вы делаете JOIN на две bags:

```
Taxpayer 1: 5 declarations × 10 payments = 50 rows на одного taxpayer в SQL
```

Hibernate не может корректно дедуплицировать — из 50 rows какие declarations относятся к каким payments? Ответ: **никак**, они независимы. Deduplication по primary key даёт Set-like поведение, но семантика `List` (с возможными duplicates) нарушается.

Решения:

**1. Заменить List на Set**:

```java
@OneToMany(mappedBy = "taxpayer")
private Set<Declaration> declarations = new LinkedHashSet<>();

@OneToMany(mappedBy = "taxpayer")
private Set<Payment> payments = new LinkedHashSet<>();
```

`Set` в Hibernate имеет уникальность, JOIN FETCH работает. `LinkedHashSet` сохраняет порядок.

**2. Разбить на два запроса**:

```java
// Первый запрос — taxpayer + declarations
List<Taxpayer> taxpayers = em.createQuery(
    "SELECT DISTINCT t FROM Taxpayer t LEFT JOIN FETCH t.declarations",
    Taxpayer.class).getResultList();

// Второй — те же taxpayer + payments
// Hibernate возьмёт taxpayer'ов из persistence context, добавит payments
em.createQuery(
    "SELECT DISTINCT t FROM Taxpayer t LEFT JOIN FETCH t.payments 
     WHERE t IN :taxpayers", Taxpayer.class)
    .setParameter("taxpayers", taxpayers)
    .getResultList();
```

Два запроса вместо одного — нет Cartesian product.

**3. Использовать @BatchSize или SUBSELECT для одной из коллекций**:

```java
@OneToMany(mappedBy = "taxpayer")
private List<Declaration> declarations;   // JOIN FETCH в query

@OneToMany(mappedBy = "taxpayer")
@BatchSize(size = 50)
private List<Payment> payments;           // lazy + batch
```

**Правило**: одну коллекцию — через JOIN FETCH. Остальные — через @BatchSize или SUBSELECT.

## Почему @OneToMany лучше держать LAZY

Соблазн: «сделаю fetch = EAGER, чтобы не думать о N+1». Так делать нельзя.

С `@OneToMany(fetch = FetchType.EAGER)`:

- Каждое чтение `Taxpayer` из БД грузит **все** его declarations. Даже если query называется `findByStatus("ACTIVE")` и вам не нужны declarations.
- Cartesian product включается по умолчанию — Hibernate добавляет JOIN.
- Списки страниц становятся тяжёлыми — на 1000 taxpayer'ов с 100 declarations каждый — 100000 rows в результате.
- Простые запросы становятся сложными:
  ```sql
  SELECT * FROM taxpayers WHERE id = 1;
  -- превращается в:
  SELECT t.*, d.* FROM taxpayers t 
  LEFT JOIN declarations d ON d.taxpayer_id = t.id
  WHERE t.id = 1;
  ```
- Проблема каскадирует — если `Declaration` имеет `EAGER` на `taxItems`, а `TaxItem` — на `payments`, каждый запрос тянет весь граф.

**Правило JPA best practice**: **все `@OneToMany`, `@ManyToMany`, `@OneToOne` — LAZY**. `@ManyToOne` — тоже стоит явно указывать LAZY, потому что default в JPA — EAGER, что тоже часто нежелательно:

```java
@ManyToOne(fetch = FetchType.LAZY)   // явно LAZY
@JoinColumn(name = "taxpayer_id")
private Taxpayer taxpayer;
```

Или глобально через bytecode enhancement — Hibernate начинает уважать LAZY даже без proxy generation.

Fetch стратегия должна быть решением **use case'а**, а не свойством entity. Разные endpoints нуждаются в разных данных. Указывайте через `JOIN FETCH` / `@EntityGraph` / DTO projections **на уровне запроса**.

## Практика в Spring Data Repository

Соберём всё вместе на реальном примере из КНП.

```java
@Entity
@Getter @Setter
public class Taxpayer {
    @Id private Long id;
    private String name;
    private String iin;
    
    @OneToMany(mappedBy = "taxpayer", fetch = FetchType.LAZY)
    @BatchSize(size = 30)   // default для всех обращений через lazy
    private Set<Declaration> declarations = new LinkedHashSet<>();
    
    @OneToMany(mappedBy = "taxpayer", fetch = FetchType.LAZY)
    @BatchSize(size = 30)
    private Set<Payment> payments = new LinkedHashSet<>();
}

@Entity
@Getter @Setter
public class Declaration {
    @Id private Long id;
    private String type;
    private LocalDate submittedAt;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "taxpayer_id")
    private Taxpayer taxpayer;
    
    @OneToMany(mappedBy = "declaration", fetch = FetchType.LAZY)
    @BatchSize(size = 30)
    private Set<TaxItem> items = new LinkedHashSet<>();
}
```

Repository:

```java
public interface TaxpayerRepository extends JpaRepository<Taxpayer, Long> {
    
    // 1. Простой findById — базовый lookup
    Optional<Taxpayer> findById(Long id);
    
    // 2. findById с полным графом — для detail страницы
    @EntityGraph(attributePaths = {"declarations", "declarations.items"})
    Optional<Taxpayer> findWithDeclarationsById(Long id);
    
    // 3. Список taxpayer'ов с количеством declarations — DTO projection
    @Query("""
        SELECT new kz.isna.knp.dto.TaxpayerListItemDTO(
            t.id, t.name, t.iin, COUNT(DISTINCT d.id)
        )
        FROM Taxpayer t
        LEFT JOIN t.declarations d
        WHERE t.status = 'ACTIVE'
        GROUP BY t.id, t.name, t.iin
        """)
    Page<TaxpayerListItemDTO> findActiveWithDeclarationCount(Pageable pageable);
    
    // 4. Полный поиск с одной коллекцией — JOIN FETCH
    @Query("""
        SELECT DISTINCT t FROM Taxpayer t
        LEFT JOIN FETCH t.declarations
        WHERE t.iin = :iin
        """)
    Optional<Taxpayer> findByIinWithDeclarations(@Param("iin") String iin);
    
    // 5. Список с несколькими коллекциями — Set + entity graph
    @EntityGraph(attributePaths = {"declarations", "payments"})
    List<Taxpayer> findAllByStatus(String status);
    
    // 6. Streaming для больших датасетов
    @QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "1000"))
    Stream<Taxpayer> streamAllByStatus(String status);
}
```

Разберём каждый метод.

**Метод 1** — базовый lookup, без deep fetch. Для простой валидации существования, без работы с ассоциациями.

**Метод 2** — детальная страница taxpayer'а с полным графом (declarations + items). `@EntityGraph` с nested path. Здесь ok использовать JOIN FETCH-like подход, потому что мы грузим один taxpayer (без Cartesian product exploding).

**Метод 3** — список для UI. **DTO projection** — самый эффективный вариант. Загружаем только name, iin, count — не entities. Работает с pagination. Никакого N+1.

**Метод 4** — специфичный use case (одна коллекция через JOIN FETCH). `DISTINCT` необходим, иначе Hibernate вернёт duplicates для taxpayer'ов с несколькими declarations.

**Метод 5** — если нужны entities (например, для business logic), несколько коллекций через `@EntityGraph` — работает, потому что коллекции — `Set`, не `List`.

**Метод 6** — streaming для очень больших данных. `HINT_FETCH_SIZE` — сколько rows JDBC driver подтягивает за раз. `Stream` позволяет обрабатывать без загрузки всей коллекции в память.

## Диагностика — как найти N+1

Первый шаг — увидеть проблему. Три подхода.

**1. Включить SQL логирование**.

`application.yml`:

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE   # видеть параметры
```

Не в проде — только для отладки. Огромный объём логов.

**2. Hibernate statistics**.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
```

После endpoint call можно посмотреть:

```java
SessionFactory sf = entityManagerFactory.unwrap(SessionFactory.class);
Statistics stats = sf.getStatistics();

System.out.println("Queries executed: " + stats.getQueryExecutionCount());
System.out.println("Entities loaded: " + stats.getEntityLoadCount());
System.out.println("Collections loaded: " + stats.getCollectionLoadCount());
```

Если запрос генерирует 500 queries — что-то не так. Норма для endpoint — 1-5 queries.

**3. p6spy или Datasource Proxy**.

Прокси JDBC-соединения. Логирует все запросы с параметрами и временем. Может создавать assertions в тестах: «этот метод не должен делать больше 3 SQL».

```yaml
# p6spy config
appender=com.p6spy.engine.spy.appender.Slf4JLogger
logMessageFormat=com.p6spy.engine.spy.appender.SingleLineFormat
```

Configuration DataSource:

```java
@Bean
DataSource dataSource(DataSource realDataSource) {
    return ProxyDataSourceBuilder
        .create(realDataSource)
        .name("DS-Proxy")
        .listener(new SLF4JQueryLoggingListener())
        .countQuery()
        .build();
}
```

**4. Test-level assertions**.

Библиотека `db-util` (github.com/vladmihalcea/db-util) позволяет assertion количества запросов:

```java
@Test
void findAllTaxpayers_shouldNotHaveNPlusOne() {
    LongAdder queryCount = SQLStatementCountValidator.reset();
    
    taxpayerService.getAllTaxpayersWithDeclarations();
    
    SQLStatementCountValidator.assertSelectCount(2);  // ожидаем 2 запроса, не 1001
}
```

Идеально для CI — прогонять integration тесты, ловить N+1 на этапе PR.

## Заключение

N+1 — не exotic bug, а системная проблема JPA/Hibernate приложений. Возникает из ленивой загрузки при итерации по коллекции. Простой пример: `taxpayers.forEach(t -> t.getDeclarations().size())` — 1001 SQL на 1000 taxpayer'ов вместо одного.

**Стратегии решения**:

- **JOIN FETCH** — один запрос, полностью инициализированный граф. Ловушки: Cartesian product на multiple collections, `MultipleBagFetchException`, `DISTINCT`-issue, pagination memory blow-up.

- **@EntityGraph** — декларативный аналог JOIN FETCH. JPA стандарт. Reusable через `@NamedEntityGraph`. Те же ловушки.

- **@BatchSize** — батчирование lazy fetches. Хороший default для всех `@OneToMany`. Один запрос IN () вместо N отдельных.

- **FetchMode.SUBSELECT** — один subquery для всех parent entities. Не работает для findById, subquery может быть тяжёлым.

- **DTO projections** — не entities, а конкретные поля. Идеально для read-only endpoints. Никакого N+1 никогда.

**Правило**: `@OneToMany` всегда LAZY. Fetch strategy — свойство use case'а, не entity. `@EntityGraph` для editable entities, DTO projections для readonly списков.

**Multiple collections** — MultipleBagFetchException при JOIN FETCH на нескольких. Решения: `Set<LinkedHashSet>` вместо List, разбиение на два запроса, mix JOIN FETCH + @BatchSize.

**Диагностика**: SQL логи в dev, Hibernate statistics, p6spy/DataSource Proxy для профилирования, `db-util` для test-level assertion количества запросов.

Практический совет: возьми endpoint своего сервиса КНП, включи SQL логи, вызови endpoint, посчитай количество SELECT'ов. Если больше 5 — вероятно N+1. Найди iteration по коллекции, добавь `@EntityGraph` или DTO projection, замерь снова. Через 10-20 таких оптимизаций научишься видеть N+1 на этапе code review без запуска. И самое главное — добавь в свой CI `db-util` assertions на количество запросов для критичных endpoints, чтобы N+1 не прокрался в прод через невнимательный PR.

Дальше — читай Vlad Mihalcea (high-performance-java-persistence.com), best resource по Hibernate performance. Портируй один существующий endpoint своего проекта, сравни SQL count до и после. Только через практику приходит настоящее понимание — какой fetch strategy выбирать в каком случае.
