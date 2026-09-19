# 18. Java 11 → 21: API и миграция

Что добавилось в стандартной библиотеке, что удалено, какие грабли при миграции.

---

## 1. Новые API

### 1.1 `HttpClient` (Java 11, но улучшения)

С Java 11 в JDK есть **встроенный HTTP-клиент**. Не нужен Apache HttpClient / OkHttp для простых задач.

```java
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .connectTimeout(Duration.ofSeconds(5))
    .build();

HttpRequest req = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users/1"))
    .header("Accept", "application/json")
    .GET()
    .build();

// синхронно
HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());

// асинхронно
CompletableFuture<HttpResponse<String>> future =
    client.sendAsync(req, HttpResponse.BodyHandlers.ofString());
```

Плюсы: HTTP/2 out of the box, reactive-friendly, без внешних зависимостей.

### 1.2 String методы (Java 11-15)

```java
"  hello  ".strip();               // "hello" (Unicode-aware, лучше trim())
"".isBlank();                       // true (пустые/whitespace)
"a\nb\nc".lines()                   // Stream<String>: "a", "b", "c"
    .forEach(System.out::println);
"ab".repeat(3);                     // "ababab"

// text block indent
"line1\nline2".indent(4);           // с 4 пробелами перед каждой строкой

// String::formatted (Java 15)
"Hello, %s!".formatted("World");    // альтернатива String.format
```

### 1.3 Optional (Java 9-11)

```java
opt.isEmpty();                      // Java 11
opt.orElseThrow();                  // без аргументов = NoSuchElementException
opt.ifPresentOrElse(v -> ..., () -> ...);
opt.or(() -> Optional.of("default"));
opt.stream();                       // Stream<T> из Optional (0 или 1 элемент)
```

### 1.4 Collectors.toList() → Stream.toList() (Java 16)

```java
// раньше
list.stream().filter(...).collect(Collectors.toList());

// теперь
list.stream().filter(...).toList();
```

Разница:
- `toList()` даёт **immutable** список.
- `Collectors.toList()` даёт `ArrayList` (mutable).

**Кавет**: если раньше писал `Collectors.toList()` и потом `.add()` — при миграции упадёт `UnsupportedOperationException`.

### 1.5 Stream.mapMulti (Java 16)

Более эффективная альтернатива `flatMap` для случаев когда каждый элемент даёт 0-1 результат.

```java
Stream.of(1, 2, 3, 4, 5)
    .<Integer>mapMulti((e, consumer) -> {
        if (e % 2 == 0) consumer.accept(e * 10);
    })
    .toList();   // [20, 40]
```

### 1.6 Files (Java 11-12)

```java
String content = Files.readString(Path.of("file.txt"));
Files.writeString(Path.of("out.txt"), "content");
```

### 1.7 var в lambda-параметрах (Java 11)

```java
list.stream()
    .map((var e) -> e.toUpperCase())     // теперь можно
    .toList();
```

Смысл: аннотации на параметрах:
```java
list.stream()
    .filter((@NonNull var e) -> !e.isEmpty())
    .toList();
```

### 1.8 `Predicate.not` (Java 11)

```java
list.stream()
    .filter(Predicate.not(String::isBlank))
    .toList();
```

Читабельнее чем `s -> !s.isBlank()`.

### 1.9 Instant / LocalDateTime мелочи

```java
LocalDate.now().datesUntil(LocalDate.now().plusDays(7))  // Java 9
    .toList();                                            // Stream<LocalDate>

Duration.ofDays(1).toMillisPart();                        // Java 9
```

### 1.10 Random (Java 17)

Иерархия улучшена: `RandomGenerator` interface, `RandomGeneratorFactory`.
```java
RandomGenerator rng = RandomGenerator.of("L64X128MixRandom");
rng.nextInt(100);
```

### 1.11 SequencedCollection (Java 21)

Новый интерфейс — «коллекция с известным порядком». `LinkedHashMap`, `LinkedHashSet`, `List` теперь имеют:
```java
list.getFirst(); list.getLast();
list.addFirst(x); list.addLast(x);
list.reversed();                    // reversed view

map.firstEntry(); map.lastEntry();
```

---

## 2. Java 9+ модули (Jigsaw)

Java 9 ввела **модульную систему**. Ключевые понятия:

### 2.1 Модуль

`module-info.java`:
```java
module kz.gov.kgd.isna.knp {
    requires spring.core;
    requires spring.boot;
    exports kz.gov.kgd.isna.knp.api;
    // не exports kz.gov.kgd.isna.knp.internal — приватно
}
```

- **`requires`** — от чего зависит.
- **`exports`** — какие пакеты доступны наружу.
- **`opens`** — доступ для reflection.
- **`provides ... with ...`** — service loader.

### 2.2 Практика

**В прод-микросервисах модули обычно НЕ используются**. Слишком сложно, ломает много библиотек, работать с classpath проще.

В ИСНА — модулей нет; classpath-based.

### 2.3 Классические грабли

**Illegal reflective access** — Java 9+ по умолчанию запрещает reflection в JDK internal классы. Многие библиотеки (Jackson, Lombok, Hibernate) обходят.

Флаги для их работы (обычно в JVM args):
```
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/java.util=ALL-UNNAMED
```

С Java 17+ — эти опции обязательны, если библиотека их требует. С Java 21 — некоторые старые библиотеки без `--add-opens` не работают.

---

## 3. Что удалено / deprecated

### 3.1 javax → jakarta (Java EE → Jakarta EE)

**САМАЯ БОЛЬШАЯ ГОЛОВНАЯ БОЛЬ** миграции.

Java EE переехала под Eclipse Foundation → пакет `javax.*` переименован в `jakarta.*`.

Примеры:
- `javax.persistence.Entity` → `jakarta.persistence.Entity`
- `javax.servlet.http.HttpServletRequest` → `jakarta.servlet.http.HttpServletRequest`
- `javax.validation.constraints.NotNull` → `jakarta.validation.constraints.NotNull`
- `javax.annotation.PostConstruct` → `jakarta.annotation.PostConstruct`
- `javax.transaction.Transactional` → `jakarta.transaction.Transactional`

**Spring Boot 3.0+ полностью на jakarta**. Boot 2.x — javax.

При миграции: нужно везде правки импортов + возможно обновление сторонних зависимостей.

### 3.2 SecurityManager

Deprecated в Java 17, готовится к удалению. Для микросервисов практически не используется — просто помнить.

### 3.3 Applets, Web Start

RIP полностью.

### 3.4 Thread.stop/suspend/resume

Deprecated for removal в Java 21.

### 3.5 Finalization

`Object.finalize()` deprecated. Использовать `try-with-resources` или `Cleaner`.

### 3.6 Nashorn JS

Удалён Java 15.

---

## 4. Реальные грабли миграции ИСНА (11 → 21)

Из memory и опыта.

### 4.1 Hibernate 6 стал строже

С JPA 3.0 (jakarta) + Hibernate 6 (Spring Boot 3):
- LazyInit ловится там где раньше молчал.
- `getById` deprecated → `getReferenceById`.
- Некоторые dialects deprecated.

Реальный кейс — memory `taxreport21-java21-runtime-regressions` (LazyInit через getById).

### 4.2 Spring Security jose/core skew

`OAuth NoSuchMethodError` — конфликт версий spring-security-jose vs core.

Memory `taxreport21-java21-runtime-regressions`. Решение: явно закрепить версии через BOM.

### 4.3 Hazelcast 3 → 5

Полная переработка API. Клиенты, embedded, конфиг.

Memory `knp-form-hz5-actuator-cache-nosuchmethod`: Boot 2.2 actuator звал `getNativeCache` на Hazelcast Cache — метод удалён в 5. Фикс — exclude auto-config.

### 4.4 JAXB (XML)

С Java 11 `javax.xml.bind.*` удалён из JDK. Нужно добавить как зависимость:
```gradle
implementation 'jakarta.xml.bind:jakarta.xml.bind-api'
runtimeOnly 'org.glassfish.jaxb:jaxb-runtime'
```

Плюс jakarta-переименование.

### 4.5 CORBA

Удалён Java 11.

### 4.6 Consul LB миграция

Memory `knp-fo-consul-lb-mr1223-latent-mine`: Ribbon → Spring Cloud LoadBalancer при миграции на Java 21 → case-sensitivity разная → `isnaKnpUser` не резолвится → 500.

### 4.7 Consul yml-reconcile

Memory `taxrep-master21-yml-reconcile-gaps`: при миграции недотянуты настройки Hib6 + `query-passing: true` в Consul → сервисы не находят passing-инстансы.

### 4.8 ShedLock

Memory `knp-fno21-shedlock-stale-image-dup-regnum`: перед мержем ShedLock в master-21 задеплоился старый образ Java 21 fno → джоба лупилась параллельно с Java 11 → дубли регномеров.

Урок: миграция Java = не только `sourceCompatibility=21`, но синхронизация всех сопутствующих версий, ретестирование scheduled-джоб.

### 4.9 gateway на Java 11

Memory `knp-gateway-no-java21`: `isna-knp-gateway` остался на Java 11 (Zuul не поддерживает Java 21 нормально). Не включать в миграцию.

### 4.10 Sync-сервисы

Memory `knp-fo-sync-notification-bugs`: 6 багов в NotificationSyncService, часть — Hibernate 6 стал строже к LazyInit.

---

## 5. Стратегия миграции

Из ИСНА-опыта:

1. **Master-21 линия параллельно с master**. Не merge, а cherry-pick или ре-разработка (memory `taxrep-master21-not-behind-master-content`).
2. **Постепенно, не всё сразу**. Один модуль за другим.
3. **Смок-тесты после каждого сервиса** (Java 21 контур `knp21`, `fo21`, `fno21`, `tax-report21`).
4. **Проверить сопутствующие: Hibernate 6, Boot 3, Hazelcast 5**.
5. **Внимание к транзитивным зависимостям**: `./gradlew dependencies`.
6. **Внимание к автогенерации кода**: Lombok, MapStruct должны быть совместимы (обновить процессоры).
7. **JAXB, JAX-WS** — не забыть добавить как зависимости.
8. **javax → jakarta** — глобальный поиск-замена + review.
9. **Тестировать прод-нагрузку** локально — не только unit.

---

## 6. Инструменты миграции

### 6.1 jdeps

Проверяет какие JDK API используются, помогает найти:
- Removed API.
- Internal API.

```
jdeps --jdk-internals app.jar
```

### 6.2 OpenRewrite

Автоматическая рефакторинг-система. Рецепты для javax→jakarta, Boot 2→3, Java 11→17→21.

```gradle
plugins {
    id 'org.openrewrite.rewrite' version '6.6.0'
}
rewrite {
    activeRecipe('org.openrewrite.java.migrate.UpgradeToJava21')
}
```

### 6.3 Error Prone

Компилятор-плагин от Google, находит проблемные паттерны на compile-time.

---

## 7. Собесные вопросы

1. **Что нового в стандартной библиотеке между 11 и 21?** — HttpClient, `String.strip/isBlank/lines/repeat`, `Optional.isEmpty/orElseThrow`, `Files.readString`, `Stream.toList`, `Stream.mapMulti`, `Predicate.not`, SequencedCollection.
2. **Разница `String::strip` и `String::trim`?** — strip Unicode-aware (правильно работает с не-ASCII whitespace).
3. **Разница `Stream.toList()` и `Collectors.toList()`?** — toList immutable, короче; Collectors.toList mutable ArrayList.
4. **Что такое модули Java 9?** — Модульная система (`module-info.java`), явные exports/requires; в микросервисах редко используется.
5. **Что такое `--add-opens`?** — Разрешить reflection в internal-пакеты JDK.
6. **javax → jakarta — что это?** — Java EE переехала в Eclipse, пакеты `javax.*` переименованы в `jakarta.*`. Boot 3.0+ на jakarta.
7. **Что удалено между 11 и 21?** — CMS GC, Nashorn, Applets, javax.xml.bind (JAXB), CORBA, SecurityManager (deprecated).
8. **Как мигрировать проект с Boot 2 (javax) на Boot 3 (jakarta)?** — OpenRewrite / глобальная замена импортов + review + обновление зависимостей.
9. **Что такое SequencedCollection?** — Интерфейс Java 21, коллекции с известным порядком (getFirst, getLast, reversed).
10. **HttpClient JDK vs Apache HttpClient?** — JDK для простых, HTTP/2, reactive; Apache для сложных сценариев (connection pool, retry, cookie management).

---

## Итог

- **Новое API**: HttpClient, String improvements, `toList()`, `mapMulti`, SequencedCollection.
- **Модули** есть с 9, но не используются массово.
- **javax → jakarta** — главная головная боль Boot 3 миграции.
- **Реальные грабли ИСНА**: Hibernate 6 строже, Spring Security skew, Hazelcast 5 API, Ribbon→SC LB.
- **Стратегия**: параллельная линия master-21, постепенно, много смок-тестов.
- **Инструменты**: jdeps, OpenRewrite.

Следующий — `19-java-21-virtual-threads.md`.
