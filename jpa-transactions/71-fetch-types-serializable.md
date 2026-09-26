# 71. FetchType EAGER / LAZY + Serializable

FetchType в JPA/Hibernate детально. Разница Hibernate 5 (Java 11) vs Hibernate 6 (Java 21). Serializable подробно.

---

## 1. FetchType — что это

**FetchType** — когда Hibernate загружает **связанные** данные при работе с сущностью.

Два варианта:
- **`FetchType.EAGER`** — загрузить **сразу** вместе с родительской сущностью.
- **`FetchType.LAZY`** — загрузить **при первом обращении** (лениво).

Определяется на **ассоциации** (`@ManyToOne`, `@OneToMany`, `@OneToOne`, `@ManyToMany`).

---

## 2. Defaults

**Ключевое для собеса**:

| Annotation | Default fetch |
|---|---|
| `@ManyToOne` | **EAGER** ⚠️ |
| `@OneToOne` | **EAGER** ⚠️ |
| `@OneToMany` | **LAZY** ✅ |
| `@ManyToMany` | **LAZY** ✅ |

**Правило запомнить**: **`ToOne` — EAGER, `ToMany` — LAZY**.

Логика:
- `ToOne` — одна сущность, "маленький" JOIN → грузить дёшево.
- `ToMany` — коллекции, может быть много → грузить только когда нужно.

**Практика**: defaults опасны. `@ManyToOne EAGER` — часто причина N+1 и лишних SQL. **Всегда указывай `LAZY` явно**.

---

## 3. EAGER — примеры

### 3.1 Код

```java
@Entity
class Fno {
    @Id Long id;
    String regNum;

    @ManyToOne(fetch = FetchType.EAGER)   // явно EAGER
    @JoinColumn(name = "taxpayer_id")
    Taxpayer taxpayer;
}
```

### 3.2 Что происходит при `em.find(Fno.class, 1L)`

Hibernate генерирует **JOIN**:
```sql
SELECT f.id, f.reg_num, f.taxpayer_id,
       t.id, t.name, t.rnn, ...
FROM fno f
LEFT JOIN taxpayer t ON f.taxpayer_id = t.id
WHERE f.id = 1
```

**Один SQL** — обе сущности загружены.

### 3.3 Плюсы

- Нет LazyInit.
- Всё под рукой сразу.
- Один SQL.

### 3.4 Минусы

- **Всегда** тянет связанное — даже если не нужно.
- **N+1** легко получить (см. §4).
- **Тяжёлый JOIN** для больших таблиц.
- **Полное дерево** может быть огромным (если EAGER рекурсивно).

### 3.5 Кавет — N+1 с EAGER

```java
@Entity
class Fno {
    @ManyToOne(fetch = EAGER)
    Taxpayer taxpayer;   // всегда EAGER
}

// запрос через JPQL:
List<Fno> list = em.createQuery("SELECT f FROM Fno f WHERE f.status = :s", Fno.class)
    .setParameter("s", NEW).getResultList();
```

**Что происходит**:
```sql
SELECT * FROM fno WHERE status = 'NEW';       -- 1

-- для каждого fno — SEPARATE query для taxpayer:
SELECT * FROM taxpayer WHERE id = 1;           -- +1
SELECT * FROM taxpayer WHERE id = 2;           -- +1
...
SELECT * FROM taxpayer WHERE id = 100;         -- +1
```

**101 SQL query!** Это классический **N+1**.

Даже с EAGER — Hibernate решает делать separate query, не JOIN (потому что через JPQL, не через `em.find()`).

Fix:
```java
// JOIN FETCH явно
@Query("SELECT f FROM Fno f JOIN FETCH f.taxpayer WHERE f.status = :s")
List<Fno> findByStatus(@Param("s") FnoStatus s);
```

**Мораль**: EAGER не гарантирует один SQL. LAZY + JOIN FETCH предсказуемее.

---

## 4. LAZY — примеры

### 4.1 Код

```java
@Entity
class Fno {
    @Id Long id;
    String regNum;

    @ManyToOne(fetch = FetchType.LAZY)   // рекомендую всегда явно
    @JoinColumn(name = "taxpayer_id")
    Taxpayer taxpayer;

    @OneToMany(mappedBy = "fno", fetch = FetchType.LAZY)
    List<FnoLine> lines = new ArrayList<>();
}
```

### 4.2 Что происходит при `em.find(Fno.class, 1L)`

```sql
SELECT * FROM fno WHERE id = 1;
```

**Один SQL** — только Fno. `taxpayer` и `lines` **не загружены**.

### 4.3 Прокси для LAZY entity (@ManyToOne LAZY)

`fno.getTaxpayer()` возвращает **прокси** — CGLib subclass Taxpayer:

```java
public class Taxpayer$$HibernateProxy$$xyz extends Taxpayer {
    private LazyInitializer li;

    public String getName() {
        return li.getImplementation().getName();   // ← если ещё не загрузил → SQL
    }
}
```

При первом обращении к любому геттеру (кроме `getId()`) → **SQL**:
```sql
SELECT * FROM taxpayer WHERE id = ?
```

### 4.4 Прокси для LAZY коллекций (@OneToMany LAZY)

`fno.getLines()` возвращает **PersistentBag** / **PersistentList** — Hibernate-specific.

Не загружено пока не вызвал:
- `.size()`.
- `.iterator()` / `.forEach()`.
- `.get(0)`.
- `.contains(...)`.

При первом обращении → SQL:
```sql
SELECT * FROM fno_line WHERE fno_id = ?
```

### 4.5 Плюсы LAZY

- **Не грузит лишнее**.
- Быстрые селекты родителя.
- Гибкость.

### 4.6 Минусы LAZY

- **LazyInitializationException** если сессия закрыта.
- **N+1** если не осторожно.
- Прокси может сбивать `.equals()`, `.getClass()`.

---

## 5. LazyInitializationException

### 5.1 Когда

```java
@Service
class FnoService {
    @Transactional(readOnly = true)
    public Fno load(Long id) {
        return repo.findById(id).orElseThrow();
    }
    // session закрылась после return
}

// в контроллере:
Fno f = svc.load(1L);
f.getTaxpayer().getName();   // ← BOOM!
```

Ошибка: `LazyInitializationException: could not initialize proxy - no Session`.

### 5.2 Почему

Прокси нужен Session (открытая tx) для SELECT. После return из `@Transactional` метода session закрывается. Обращение к прокси не может резолвить.

### 5.3 Решения

**A) DTO в сервисе** (best):
```java
@Transactional(readOnly = true)
public FnoDto load(Long id) {
    Fno f = repo.findById(id).orElseThrow();
    return new FnoDto(f.getRegNum(), f.getTaxpayer().getName());   // загружаем ВНУТРИ tx
}
```

**B) JOIN FETCH**:
```java
@Query("SELECT f FROM Fno f JOIN FETCH f.taxpayer WHERE f.id = :id")
Optional<Fno> findByIdWithTaxpayer(@Param("id") Long id);
```

**C) @EntityGraph**:
```java
@EntityGraph(attributePaths = {"taxpayer", "lines"})
Optional<Fno> findById(Long id);
```

**D) FetchType.EAGER** (ленивое):
Убивает производительность, ломает всё под нагрузкой. **Не делай**.

**E) OSIV** (плохо):
`spring.jpa.open-in-view: true` (default) — session до конца HTTP-запроса. Скрывает N+1, всех глушит. Отключай.

---

## 6. Управление fetch на runtime

### 6.1 JOIN FETCH

```java
@Query("SELECT f FROM Fno f JOIN FETCH f.taxpayer WHERE f.status = :s")
List<Fno> findByStatusWithTaxpayer(@Param("s") FnoStatus s);
```

Один SQL с JOIN.

**Кавет с Pageable + JOIN FETCH коллекций**:
```
HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!
```

Hibernate загружает всё, режет в памяти. Не масштабируется.

### 6.2 @EntityGraph

Декларативно:
```java
@EntityGraph(attributePaths = {"taxpayer", "lines", "lines.product"})
@Query("SELECT f FROM Fno f WHERE f.status = :s")
List<Fno> findByStatus(@Param("s") FnoStatus s);
```

Hibernate строит нужные JOIN'ы.

### 6.3 @BatchSize

Не JOIN, а «пачки лениво»:
```java
@Entity
@BatchSize(size = 20)
class Taxpayer { ... }
```

Или на связи:
```java
@OneToMany(mappedBy = "fno")
@BatchSize(size = 50)
List<FnoLine> lines;
```

Итог для N Fno:
```sql
SELECT * FROM fno;                                   -- 1
SELECT * FROM taxpayer WHERE id IN (1,2,3,...,20);   -- +1 на 20
SELECT * FROM taxpayer WHERE id IN (21,...,40);       -- +1
```

Вместо `1 + N` → `1 + N/batch_size`.

Глобально:
```yaml
spring.jpa.properties.hibernate.default_batch_fetch_size: 20
```

---

## 7. Hibernate 5 (Java 11) vs Hibernate 6 (Java 21) — разница

В ИСНА master = Java 11 + Hibernate 5. master-21 = Java 21 + Hibernate 6 (пришёл с Spring Boot 3).

### 7.1 Hibernate 6 стал строже к LazyInit

Hib5 иногда **прощал** и молча делал fallback запрос. Hib6 — **бросает exception**.

Классика: Bean method вне tx, но с активным EntityManager из-за OSIV.

Hib5: работает, Hib6: LazyInit.

### 7.2 getById deprecated → getReferenceById

**Hibernate 5**:
```java
Fno f = repo.getById(1L);   // proxy, без SELECT
```

**Hibernate 6**:
```java
Fno f = repo.getReferenceById(1L);   // то же
```

`getById` теперь = `findById` + `.orElseThrow()` — то есть **SELECT сразу**.

Кавет: если код полагался что `getById` даёт proxy без SELECT (для FK-связи) → в Hib6 всё изменилось.

Реальный ИСНА-кейс `taxreport21-java21-runtime-regressions`:
- Старый код: `Fno f = repo.getById(...); toDto(f);`
- В Hib5 (master) — работало (proxy → SELECT только при обращении).
- В Hib6 (master-21) — было `getById` = proxy → `toDto` **вне tx** → LazyInit.
- Fix: `findById` (SELECT сразу).

### 7.3 Изменения в dialects

- `PostgreSQLDialect` в Hib6 сам выбирает версию (не надо задавать `PostgreSQL95Dialect`).
- Deprecated warnings при использовании старых имён.

### 7.4 Изменения в generated SQL

Hib6 иногда генерирует другой SQL:
- Больше bind parameters.
- Разные aliases.
- Optimized JOIN order.

Обычно совместимо, но edge cases бывают.

### 7.5 Deprecated / removed APIs

- `Session.load()` → `getReference()`.
- `Criteria API` (legacy) → `JPA CriteriaBuilder`.
- Multi-tenancy изменилось.

### 7.6 Из memory ИСНА

- `taxreport21-java21-runtime-regressions` — LazyInit проявился где Hib5 молчал.
- `knp-fo-sync-notification-bugs` — 6 багов включая Hib6 LazyInit + self-invocation `@Transactional`.
- `taxrep-master21-yml-reconcile-gaps` — миграция yml на Java 21, некоторые Hib-настройки не проброшены.

**Правило миграции**:
1. Отключить OSIV.
2. Заменить `getById` → `findById` (или explicit `getReferenceById`).
3. Проверить все LazyInit места.
4. DTO из сервиса, не Entity.
5. Тесты покрывают fetching flows.

---

## 8. Serializable

### 8.1 Что такое

**`java.io.Serializable`** — **marker interface** (пустой, без методов).

```java
public interface Serializable {
    // пустой
}
```

Реализация:
```java
public class Order implements Serializable {
    private static final long serialVersionUID = 1L;

    private Long id;
    private String customerId;
    private transient String cachedSummary;   // не сериализуется

    // ...
}
```

Object этого класса можно **сериализовать** — превратить в поток байт.

### 8.2 Зачем

- **Сохранить состояние** в файл.
- **Передать по сети** (RMI, distributed cache Hazelcast).
- **Сохранить в БД** как BLOB.
- **Клонировать** через serialize+deserialize (deep copy).
- **HTTP Session** в кластере (Tomcat replication).

### 8.3 Использование

**Serialize**:
```java
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("order.ser"))) {
    oos.writeObject(order);
}
```

**Deserialize**:
```java
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("order.ser"))) {
    Order o = (Order) ois.readObject();
}
```

### 8.4 serialVersionUID

**Version ID** для проверки совместимости.

```java
private static final long serialVersionUID = 1L;
```

Что делает:
- Сериализация записывает `serialVersionUID` в поток.
- Десериализация проверяет — если не совпадает → `InvalidClassException`.

Обеспечивает: старая сериализованная форма не десериализуется в **изменённый** класс.

**Правило**:
- Явно указывать (иначе JVM сам вычислит от структуры — при любом изменении класса поломает существующие serialized объекты).
- Менять когда структура несовместима.

Kavет: без `serialVersionUID` — любое изменение (добавили поле!) ломает существующие сериализованные объекты.

### 8.5 transient

Поле **не сериализуется**:
```java
private transient String cachedData;
```

Для:
- Кэшированных значений.
- Ссылок на runtime ресурсы.
- Sensitive данных (passwords).
- Cyclic references через lazy load.

При deserialize — значение = **default** (null, 0, false).

### 8.6 Правила Serializable

- **Все поля** должны быть Serializable (или `transient` / `static`).
- **Super class** может не быть Serializable — тогда его поля не сериализуются (default значения при deserialize).
- Static поля **не сериализуются** (принадлежат классу, не instance).
- `transient` поля — не сериализуются.

### 8.7 Кавет с не-Serializable полями

```java
class Order implements Serializable {
    private Customer customer;   // Customer НЕ Serializable
}
```

При serialize → **NotSerializableException**. Все поля должны быть Serializable.

### 8.8 Custom serialization

`writeObject` / `readObject` для контроля:
```java
private void writeObject(ObjectOutputStream out) throws IOException {
    out.defaultWriteObject();
    // custom
}

private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
    in.defaultReadObject();
    // reconstruct transient fields
    this.cache = new HashMap<>();
}
```

Полезно для reconstruct transient state.

### 8.9 Externalizable

Полный контроль (не use default):
```java
public class Order implements Externalizable {
    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeLong(id);
        out.writeUTF(customerId);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException {
        id = in.readLong();
        customerId = in.readUTF();
    }
}
```

Быстрее (нет reflection). Redko используется.

---

## 9. Deserialization security (CVE!)

### 9.1 Проблема

Deserialize — это создание **произвольного объекта** по данным потока. Если поток от untrusted источника → **RCE (Remote Code Execution)**.

Атакующий отправляет специально сформированный serialized объект → при `readObject()` вызываются setter'ы / constructors классов **из classpath**. Через gadget chains — RCE.

Классика:
- **CVE-2015-4852 (Apache Commons Collections)** — RCE через WebLogic/JBoss/Jenkins.
- **Log4Shell**-style vulnerabilities.

### 9.2 Never deserialize untrusted data

Правило: **никогда** deserialize данные от:
- HTTP body.
- URL params.
- Cookies.
- Внешнего API.

Если очень надо — **strict class filtering**:
```java
ObjectInputStream ois = new ObjectInputStream(in) {
    @Override
    protected Class<?> resolveClass(ObjectStreamClass osc) {
        String name = osc.getName();
        if (!allowedClasses.contains(name)) {
            throw new InvalidClassException("Not allowed: " + name);
        }
        return super.resolveClass(osc);
    }
};
```

Java 9+: `ObjectInputFilter` для standard filtering.

### 9.3 Альтернативы

**JSON (Jackson)**:
- Не выполняет constructors произвольных классов.
- Требует явный type (не volatile).
- Безопаснее.

**Protobuf**:
- Schema-based.
- Binary, компактный.
- Безопасный.

**MessagePack**, **Avro** — аналоги.

**Правило современности**: **не использовать Java serialization** для внешнего обмена. Только для internal кэшей / cluster session.

---

## 10. Serializable в JPA / Spring

### 10.1 Entity implements Serializable

JPA-спека **рекомендует** Entity implements Serializable:
```java
@Entity
class Fno implements Serializable {
    private static final long serialVersionUID = 1L;
    // ...
}
```

Зачем:
- **Detached instances** передаются между слоями.
- **HTTP Session replication** в кластере.
- **Second-level cache** (Hazelcast) требует Serializable.
- **JMS / RMI** messages.

Обязательно для **@Id** (composite ключей) — `equals`/`hashCode` работают правильно.

### 10.2 Session cluster (Tomcat)

Sticky sessions или replication → Serializable объекты в session:
```java
request.getSession().setAttribute("user", user);   // user должен быть Serializable
```

Иначе — `NotSerializableException`.

### 10.3 Hazelcast distributed cache

Кэш второго уровня Hibernate через Hazelcast → все Entity Serializable.

В ИСНА Hazelcast используется активно (см. `knp-form-hz5-actuator-cache-nosuchmethod`).

### 10.4 RMI

Java RMI (remote method invocation) — тоже требует Serializable.

Практически не используется в микросервисах (REST + JSON вместо).

---

## 11. Best practices

### 11.1 FetchType

1. **Всегда явно `LAZY`** для `@ManyToOne`, `@OneToOne`.
2. **DTO в сервисе**, не Entity.
3. **JOIN FETCH / @EntityGraph** для нужных связей.
4. **@BatchSize** для fallback.
5. **OSIV off** (`spring.jpa.open-in-view: false`).
6. **Не полагайся на EAGER** — N+1 всё равно возможен.

### 11.2 Serializable

1. **Явный `serialVersionUID = 1L`**.
2. **`transient`** для не-сериализуемых полей.
3. **Никогда deserialize untrusted data**.
4. **JSON / Protobuf** для API.
5. **Serializable для Entity** — если Hazelcast / cluster.
6. **Обновлять `serialVersionUID`** при breaking changes.

---

## 12. Собесные вопросы

### FetchType

1. **Что такое FetchType?** — Когда Hibernate загружает связанные сущности: EAGER (сразу) или LAZY (лениво).
2. **Defaults?** — `@ManyToOne`/`@OneToOne` EAGER (опасно); `@OneToMany`/`@ManyToMany` LAZY.
3. **Правило?** — Всегда явно LAZY.
4. **Как работает LAZY?** — Прокси (CGLib subclass) для entity; PersistentBag/List для коллекций.
5. **LazyInitializationException — когда?** — Обращение к прокси вне активной Session.
6. **Как лечить LazyInit?** — DTO в сервисе, JOIN FETCH, @EntityGraph.
7. **Разница JOIN FETCH и @EntityGraph?** — Оба loading; EntityGraph декларативнее.
8. **@BatchSize — что даёт?** — Loading пачками (IN клауза), избегает N+1.
9. **Разница Hib5 (Java 11) и Hib6 (Java 21) в LazyInit?** — Hib6 строже, ловит там где Hib5 молчал.
10. **getById vs getReferenceById?** — Hib5: getById = proxy без SELECT; Hib6: getById deprecated, использовать getReferenceById.

### Serializable

11. **Что такое Serializable?** — Marker interface (`java.io.Serializable`); объект можно превратить в поток байт.
12. **Зачем serialVersionUID?** — Проверка совместимости версий класса при deserialize.
13. **transient — что делает?** — Поле не сериализуется, при deserialize = default.
14. **Deserialization security issue?** — Untrusted stream → RCE через gadget chains.
15. **Альтернативы Java serialization?** — JSON (Jackson), Protobuf, MessagePack, Avro.
16. **Зачем Entity implements Serializable?** — Detached instances, HTTP session, Hazelcast cluster.
17. **Externalizable vs Serializable?** — Externalizable — полный контроль (`writeExternal`/`readExternal`); быстрее.
18. **Static поля сериализуются?** — Нет, принадлежат классу.
19. **Что если поле не Serializable?** — NotSerializableException при serialize; либо transient, либо реализовать.
20. **Когда menять serialVersionUID?** — При breaking changes (удалил поле, изменил тип).

---

## Итог

### FetchType

- **EAGER default** для `@ManyToOne`/`@OneToOne` — опасно.
- **LAZY default** для коллекций — обычно OK.
- **Всегда явно LAZY** для ToOne.
- **LazyInit** — вечная проблема; лечится DTO.
- **Hib6 (Java 21) строже** к LazyInit — регрессии при миграции с Java 11.
- **getById в Hib6** = SELECT сразу; для proxy нужен `getReferenceById`.

### Serializable

- **Marker interface** для объектов, которые можно превратить в bytes.
- **`serialVersionUID`** обязателен.
- **`transient`** для не-serializable полей.
- **Никогда** не deserialize untrusted data (RCE risk).
- **JSON / Protobuf** — modern альтернативы.
- Для Entity + Hazelcast / session cluster — implements Serializable.

Итого **71 файл** в `isna-theory\`.
