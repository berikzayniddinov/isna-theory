# 16. Java 11 → 21: язык и синтаксис

Что появилось в языке между Java 11 (LTS) и Java 21 (LTS). Основа для миграционных вопросов на собесе.

---

## 1. Хронология LTS-релизов

- **Java 8** (2014) — lambda, Stream, Optional, java.time.
- **Java 11** (2018) — `var` для локальных переменных, HttpClient, single-file source.
- **Java 17** (2021) — records, sealed classes, pattern matching for instanceof, text blocks.
- **Java 21** (2023) — virtual threads, pattern matching for switch, record patterns, sequenced collections.

Между Java 11 и Java 21 — три больших LTS-скачка. Разберём главные новшества.

---

## 2. `var` — вывод типа локальных переменных (Java 10, доступен в 11)

```java
// до
Map<String, List<Fno>> fnosByStatus = new HashMap<>();
List<String> names = getNames();

// с var
var fnosByStatus = new HashMap<String, List<Fno>>();
var names = getNames();
```

Ограничения:
- Только для **локальных** переменных (не поля, не параметры).
- Нужен инициализатор (`var x;` — ошибка).
- Не для lambda-параметров (без типа) — они и так без типа.
- Не для `null` (без типа неоднозначно).

**Правило**: используй когда тип очевиден из RHS (`var list = new ArrayList<String>()`). Не злоупотребляй — читаемость страдает если тип неясен.

---

## 3. Text blocks (Java 15+)

Многострочные строки без escape.

```java
// до
String json = "{\n" +
              "  \"name\": \"Fno\",\n" +
              "  \"reg\": \"12345\"\n" +
              "}";

// с text blocks
String json = """
              {
                "name": "Fno",
                "reg": "12345"
              }
              """;
```

- Тройные кавычки.
- Отступ — по минимальному отступу непустых строк (не считая закрывающих `"""`).
- Escape работают, но реже нужны.
- `\` в конце строки — «продолжение без \n».

Удобно для SQL, HTML, JSON.

---

## 4. Records (Java 16+)

**Immutable data carrier** — класс только для хранения данных.

```java
// до — 40 строк boilerplate
public class FnoDto {
    private final String regNum;
    private final FnoStatus status;
    private final LocalDateTime createdAt;

    public FnoDto(String regNum, FnoStatus status, LocalDateTime createdAt) {
        this.regNum = regNum;
        this.status = status;
        this.createdAt = createdAt;
    }

    public String regNum() { return regNum; }
    public FnoStatus status() { return status; }
    public LocalDateTime createdAt() { return createdAt; }

    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
}

// после — 1 строка
public record FnoDto(String regNum, FnoStatus status, LocalDateTime createdAt) {}
```

Что даётся автоматически:
- Финальные поля.
- Конструктор `FnoDto(String, FnoStatus, LocalDateTime)`.
- Accessor'ы `regNum()`, `status()`, `createdAt()` (без `get`!).
- `equals`, `hashCode`, `toString` по всем полям.

### 4.1 Custom конструктор

```java
public record FnoDto(String regNum, FnoStatus status) {
    // compact constructor — валидация
    public FnoDto {
        if (regNum == null || regNum.isBlank()) {
            throw new IllegalArgumentException("regNum required");
        }
    }
}
```

### 4.2 Static factory

```java
public record FnoDto(String regNum, FnoStatus status) {
    public static FnoDto of(String regNum) {
        return new FnoDto(regNum, FnoStatus.NEW);
    }
}
```

### 4.3 Ограничения

- Не extends (наследование).
- Все поля final.
- Не удобны для сущностей (JPA обычно требует no-arg ctor + мутабельность).

**Использование**: DTO, value objects, tuples, ключи Map.

---

## 5. Sealed classes (Java 17+)

Ограничивают кто может наследоваться.

```java
public sealed interface FnoEvent
    permits FnoSubmitted, FnoRejected, FnoApproved {
}

public record FnoSubmitted(Long id, LocalDateTime at) implements FnoEvent {}
public record FnoRejected(Long id, String reason) implements FnoEvent {}
public record FnoApproved(Long id, String approver) implements FnoEvent {}
```

Гарантия: **только эти три класса** могут реализовывать `FnoEvent`. Компилятор знает — можно писать **exhaustive switch** без default (см. §7).

Подтипы должны быть:
- `final` (нельзя дальше наследовать),
- `sealed` (продолжают ограничение),
- `non-sealed` (открывают наследование).

Применение: моделирование алгебраических типов данных (ADT) как в Kotlin/Scala.

---

## 6. Pattern matching for `instanceof` (Java 16+)

```java
// до
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// после
if (obj instanceof String s) {
    System.out.println(s.length());
}
```

Переменная `s` доступна в теле if-а. Не нужна ручная кастовка.

Работает в условиях:
```java
if (obj instanceof String s && !s.isEmpty()) {
    System.out.println(s);
}
```

---

## 7. Pattern matching for `switch` (Java 21)

Комбо records + sealed + switch = мощная штука.

```java
// как раньше
String describe(Object obj) {
    if (obj instanceof Integer i) return "int " + i;
    else if (obj instanceof String s) return "string " + s;
    else if (obj == null) return "null";
    else return "other";
}

// с pattern matching
String describe(Object obj) {
    return switch (obj) {
        case Integer i -> "int " + i;
        case String s -> "string " + s;
        case null -> "null";
        default -> "other";
    };
}
```

С sealed — exhaustive без default:
```java
String handle(FnoEvent e) {
    return switch (e) {                              // компилятор знает все подтипы
        case FnoSubmitted s -> "submitted " + s.id();
        case FnoRejected r -> "rejected " + r.reason();
        case FnoApproved a -> "approved by " + a.approver();
        // default не нужен!
    };
}
```

### 7.1 Record patterns (Java 21)

Деструктуризация:
```java
switch (event) {
    case FnoSubmitted(Long id, var at) -> log("submitted " + id + " at " + at);
    case FnoRejected(Long id, String reason) -> log("rejected " + id + ": " + reason);
    default -> {}
}
```

### 7.2 Guards (when)

```java
switch (fno) {
    case Fno f when f.status() == NEW -> processNew(f);
    case Fno f when f.status() == REJECTED -> reprocess(f);
    default -> {}
}
```

---

## 8. Switch expressions (Java 14+)

Не только statement, но выражение:

```java
// до
String label;
switch (status) {
    case NEW: label = "новый"; break;
    case REJECTED: label = "отклонён"; break;
    default: label = "-";
}

// после
String label = switch (status) {
    case NEW -> "новый";
    case REJECTED -> "отклонён";
    default -> "-";
};
```

- Стрелка `->` вместо `:` — нет fallthrough.
- Множественные значения: `case NEW, DRAFT -> "черновик"`.
- Блок: `case X -> { ... yield "value"; }`.

---

## 9. Enhanced Enum (не новое, но часто в паре)

`enum` уже был мощным, но теперь в комбо со switch expressions выглядит красиво:

```java
enum FnoStatus {
    NEW {
        @Override public boolean isTerminal() { return false; }
    },
    SUBMITTED {
        @Override public boolean isTerminal() { return false; }
    },
    APPROVED {
        @Override public boolean isTerminal() { return true; }
    };

    public abstract boolean isTerminal();
}
```

Или через switch expression:
```java
enum FnoStatus {
    NEW, SUBMITTED, APPROVED, REJECTED;

    public boolean isTerminal() {
        return switch (this) {
            case APPROVED, REJECTED -> true;
            case NEW, SUBMITTED -> false;
        };
    }
}
```

---

## 10. `_` — unnamed variable (Java 21 preview, финал в 22)

```java
try {
    // ...
} catch (Exception _) {                       // не важно имя
    log("error");
}

for (var _ : list) {                          // просто итерация без переменной
    count++;
}

record Point(int x, int y) {}
if (obj instanceof Point(int x, _)) {         // деструктуризация с игнором
    System.out.println(x);
}
```

Только в preview на 21. В 22+ — обычная фича.

---

## 11. Другие мелкие изменения

### 11.1 `String::indent`, `String::stripIndent` (Java 12+)
```java
"    hello".indent(-2);          // "  hello\n"
```

### 11.2 `String::formatted`
```java
"Hello, %s! Age: %d".formatted("Berik", 32);
```

### 11.3 `List.of`, `Map.of` (Java 9+, но не всегда помнят)
```java
List<String> list = List.of("a", "b", "c");  // immutable
Map<String, Integer> map = Map.of("a", 1, "b", 2);
```

### 11.4 Stream.toList() (Java 16+)
```java
// раньше
list.stream().filter(...).collect(Collectors.toList());

// теперь
list.stream().filter(...).toList();          // короче, immutable
```

### 11.5 `Files.readString`, `Files.writeString` (Java 11+)
```java
String content = Files.readString(Path.of("file.txt"));
Files.writeString(Path.of("out.txt"), content);
```

### 11.6 `Optional.orElseThrow()` без аргументов (Java 10+)
```java
Fno f = repo.findById(id).orElseThrow();     // NoSuchElementException
```

### 11.7 `Optional.isEmpty()` (Java 11+)
```java
if (opt.isEmpty()) { ... }                   // раньше !opt.isPresent()
```

### 11.8 Enhanced NullPointerException (Java 14+)
```java
// до: "NullPointerException at line 42"
// после: "Cannot invoke 'String.length()' because 'foo.bar' is null"
```

Включено по умолчанию с Java 15.

---

## 12. Что удалили

### 12.1 SecurityManager (deprecated в 17)

Старый механизм авторизации. Заменяется другим — но для микросервисов практически не используется.

### 12.2 CMS GC (removed в 14)

Concurrent Mark-Sweep удалён; используй G1 или ZGC.

### 12.3 Nashorn JS engine (removed в 15)

Скриптовый движок JavaScript в JVM.

### 12.4 Applet API (removed в 17)

RIP applets.

### 12.5 Некоторые методы `Thread` (deprecated for removal)

- `Thread.suspend()`, `Thread.resume()`, `Thread.stop()` — давно deprecated.
- В 21 — deprecated for removal.

---

## 13. Полный сравнительный пример

Одна и та же логика на Java 11 и Java 21.

**Java 11:**
```java
public class OrderProcessor {

    public String describeOrder(Object o) {
        if (o == null) {
            return "no order";
        }
        if (o instanceof Order) {
            Order order = (Order) o;
            if (order.getStatus() == Status.NEW) {
                return "new order " + order.getId();
            } else if (order.getStatus() == Status.PAID) {
                return "paid order " + order.getId();
            } else {
                return "other";
            }
        }
        return "not an order";
    }
}
```

**Java 21:**
```java
public sealed interface OrderResult permits NewOrder, PaidOrder, OtherOrder {}
public record NewOrder(Long id) implements OrderResult {}
public record PaidOrder(Long id, BigDecimal amount) implements OrderResult {}
public record OtherOrder(Long id) implements OrderResult {}

public class OrderProcessor {

    public String describeOrder(Object o) {
        return switch (o) {
            case null -> "no order";
            case NewOrder(var id) -> "new order " + id;
            case PaidOrder(var id, var amt) -> "paid order " + id + " for " + amt;
            case OrderResult result -> "other order";
            default -> "not an order";
        };
    }
}
```

Разница огромна — читаемость, безопасность типов, экспрессивность.

---

## 14. Собесные вопросы

1. **Что такое `var`? Где нельзя использовать?** — Вывод типа локальной переменной; не в полях/параметрах/без инициализатора.
2. **Что такое record?** — Immutable data carrier; авто-конструктор, accessors, equals/hashCode/toString.
3. **Разница record и обычного класса?** — record final, поля final, не наследуется от других классов.
4. **Что такое sealed class?** — Разрешает наследование только явно перечисленным классам.
5. **Что такое pattern matching в instanceof?** — Автоматически кастит переменную: `if (obj instanceof String s)`.
6. **Pattern matching в switch — что даёт?** — Тип + деструктуризация + guards; exhaustive для sealed.
7. **Разница switch statement и switch expression?** — Expression возвращает значение через `->` или `yield`.
8. **Text blocks — зачем?** — Многострочные строки без escape.
9. **Что такое record pattern?** — Деструктуризация record в switch/instanceof.
10. **Enhanced NullPointerException?** — Сообщение указывает конкретное поле в null-цепочке.
11. **Что такое unnamed variable `_`?** — Игнорирование переменной (preview в 21, финал в 22).
12. **Stream.toList() vs Collectors.toList()?** — Первый immutable, короче, из Java 16.

---

## Итог

Между Java 11 и 21 язык стал:
- **Меньше boilerplate** — record, var.
- **Более выразительный** — pattern matching, switch expressions, text blocks.
- **Более безопасный** — sealed + exhaustive switch, enhanced NPE, record patterns.
- **Ближе к функциональному** — деструктуризация, immutable-first.

Всё это доступно если ты на 21 — миграция как раз для этого.

Следующий — `17-java-11-to-21-jvm.md`.
