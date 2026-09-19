# 17. Java 11 → 21: JVM, GC, производительность

Что изменилось «под капотом»: GC, JIT, старт, память.

---

## 1. GC — сборщики мусора

Между 11 и 21 GC-подсистема сильно улучшилась.

### 1.1 Что было в Java 11

По умолчанию — **G1** (с Java 9).
Другие:
- **Parallel GC** — старый, для batch-задач.
- **Serial GC** — для маленьких приложений.
- **CMS** (Concurrent Mark-Sweep) — устаревал.
- **ZGC** и **Shenandoah** — экспериментальные (preview).

### 1.2 Что стало в Java 21

По умолчанию — всё ещё G1, но:
- **CMS удалён** (Java 14).
- **ZGC production-ready** (Java 15).
- **ZGC generational** (Java 21) — большой апгрейд.
- **Shenandoah production-ready** (Java 15).

### 1.3 G1 (Garbage First)

Region-based: heap делится на регионы (~1-32 MB). Каждый регион — Eden / Survivor / Old.

- Concurrent phases + короткие STW.
- Целевая пауза: `-XX:MaxGCPauseMillis=200` (default).
- Хорошо для heap 4-64 GB.

Улучшения между 11 и 21:
- Меньше памяти на metadata.
- Быстрее старт.
- Умнее реагирует на аллокационные всплески.
- Concurrent full GC (не всегда STW).

### 1.4 ZGC (Z Garbage Collector)

С Java 15 — production. С Java 21 — **generational** (было single-gen).

Ключевое:
- Паузы **<1 мс** даже на heap в терабайты.
- Concurrent compaction (без STW).
- Использует colored pointers (тэги в указателях).
- Хорошо для low-latency сервисов (trading, real-time).

Включение:
```
-XX:+UseZGC                    # Java 15+
-XX:+UseZGC -XX:+ZGenerational # Java 21+
```

Минусы:
- Больше памяти (up to 2-3x heap overhead).
- Не для маленьких heap (<8 GB — G1 лучше).

### 1.5 Shenandoah

От RedHat. Аналог ZGC — concurrent compaction. Плюс: без colored pointers, работает на x86 и ARM. Не по умолчанию в OpenJDK, но есть.

### 1.6 Выбор для микросервисов

Для типичного Spring Boot микросервиса (heap 512 MB - 2 GB): **G1**. Хватает.

Для больших сервисов и требований low-latency: **ZGC**.

В ИСНА — обычно G1 (heap 512 MB - 2 GB типовой).

---

## 2. Container awareness

### 2.1 Проблема Java 8

JVM смотрела на память **хоста**, а не контейнера. Ставишь `-Xmx1g` в контейнере с 512 MB → OOMKilled.

Или без явного `-Xmx` — JVM брала 25% от хостовой RAM (могло быть 64 GB) → сразу OOMKilled.

### 2.2 Что стало

С **Java 10+** JVM видит cgroup-лимиты. С **Java 15+** — cgroup v2.

Автоматический выбор heap:
```
-XX:MaxRAMPercentage=75.0     # 75% от cgroup memory limit
-XX:InitialRAMPercentage=50.0
-XX:MinRAMPercentage=50.0
```

Правило: явно указывай в Dockerfile:
```
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0"
```

Или классически `-Xms512m -Xmx1024m`.

### 2.3 CPU awareness

JVM видит cgroup CPU quota. Отсюда решается количество:
- GC threads.
- ForkJoinPool.commonPool() размер.
- ParallelStream.

Флаг `-XX:ActiveProcessorCount=N` — переопределить.

---

## 3. CDS и AppCDS — быстрый старт

**CDS (Class Data Sharing)** — механизм для ускорения загрузки классов.

### 3.1 Идея

Стандартные JDK-классы (java.lang.*, java.util.*) не меняются. Можно один раз распарсить их метаданные, сохранить в файл, и при следующих запусках JVM загружать быстрее.

С Java 12+ — **default CDS archive** уже поставляется с JDK.

### 3.2 AppCDS — для твоих классов

Ещё лучше: подготовь архив, включающий все классы твоего Spring Boot приложения.

```bash
# 1. Прогнать приложение с trainingом
java -XX:ArchiveClassesAtExit=app.jsa -jar app.jar

# 2. Запускать с архивом
java -XX:SharedArchiveFile=app.jsa -jar app.jar
```

Спад cold-start на 10-30%. Для Spring Boot Layered — можно вынести в отдельный слой.

### 3.3 CRaC (Coordinated Restore at Checkpoint) — Java 21 preview

Ещё дальше: **сохранить всё состояние JVM в файл** после warmup. Восстановить за миллисекунды.

Пока preview, в проде редко.

---

## 4. Native compilation (GraalVM)

Не Java 21, но популярная альтернатива.

### 4.1 Что это

**GraalVM Native Image** — компилятор AOT (ahead-of-time). Собирает Java в **нативный executable** без JVM.

Плюсы:
- Cold-start **10-100 мс** (vs 5-15 сек JVM).
- Меньше памяти (~50-100 MB vs 400+ MB).

Минусы:
- Долгий build (минуты).
- Нет JIT — оптимизации только AOT.
- Reflection/dynamic classloading требуют явной конфигурации.
- Некоторые Spring/Hibernate фичи не работают из коробки.

### 4.2 Spring Boot AOT

С Boot 3.0+ есть `spring-boot-maven-plugin process-aot`. Готовит проект под GraalVM.

В ИСНА не используется массово — legacy код + Hibernate тонкости слишком много.

---

## 5. JIT улучшения

### 5.1 C2 tiered compilation

Осталась двух-уровневая:
- **C1** — быстрая компиляция, слабая оптимизация.
- **C2** — долгая, сильная.

С 21 — C2 стала эффективнее (patterns, inlining, escape analysis).

### 5.2 Loom pinning (важно для virtual threads)

Virtual threads (см. `19-java-21-virtual-threads.md`) плохо работают внутри `synchronized` и native. JVM их «pins» — прикрепляет к carrier thread → теряется преимущество.

### 5.3 GraalVM as JIT

Можно использовать GraalVM compiler вместо HotSpot JIT (в OpenJDK Graal — с 11):
```
-XX:+UnlockExperimentalVMOptions -XX:+EnableJVMCI -XX:+UseJVMCICompiler
```

Иногда быстрее C2, иногда медленнее — надо мерить.

---

## 6. JFR (Java Flight Recorder)

Профилировщик, встроенный в JVM. Раньше был коммерческий Oracle JDK — с Java 11 в OpenJDK, бесплатный.

### 6.1 Записать

```bash
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar

# или в runtime
jcmd <pid> JFR.start duration=60s filename=recording.jfr
```

### 6.2 Проанализировать

**JDK Mission Control (JMC)** — GUI-инструмент от Oracle. Показывает:
- CPU hotspots.
- GC pauses.
- Memory allocations.
- Threads, locks.
- I/O.

Использую для диагностики «почему приложение тормозит» — гораздо лучше heap dump.

---

## 7. Metaspace

Java 8+ метаданные классов лежат в **Metaspace** (не в heap, отдельная native область).

По умолчанию: не ограничен (`-XX:MaxMetaspaceSize=∞`).

Растёт при:
- Загрузке новых классов.
- Динамической генерации (Hibernate proxy, CGLib).
- Классов долгоживущих ClassLoader'ов.

**Утечка**: если создаёшь ClassLoader и не выгружаешь → его классы копятся в Metaspace → OOM `Metaspace`.

Фиксируй в Docker:
```
-XX:MaxMetaspaceSize=256m
```

С Java 16+ появилась Elastic Metaspace — уменьшает overhead / fragmentation.

---

## 8. String deduplication

G1 может дедуплицировать `String` (одинаковые char[] заменяет ссылками):
```
-XX:+UseStringDeduplication
```

Помогает в приложениях с большим количеством одинаковых строк (JSON парсинг). Может сэкономить 5-15% heap.

---

## 9. Startup time — сравнение

Типичный Spring Boot микросервис:

| | Cold start | Memory на старте |
|---|---|---|
| Java 8 + Boot 1.x | 6-10 сек | 300-400 MB |
| Java 11 + Boot 2.4 | 4-7 сек | 250-350 MB |
| Java 17 + Boot 3.0 | 3-5 сек | 200-300 MB |
| Java 21 + Boot 3.2 + CDS | 2-4 сек | 200-280 MB |
| Java 21 GraalVM native | 100-300 мс | 50-100 MB |

Цифры примерные, зависят от количества auto-config и JPA.

---

## 10. Реальные ИСНА-кейсы Java 21 миграции

Из memory:

- **`knp-fo-prod-java21-migration-clean-baseline`**: прод-ФО мигрирован java11→21 (Amazon Corretto 21.0.11); все 20 подов Running, 0 рестартов, ноль Java21-регрессий.
- **`taxrep21-hazelcast-kesh66-unreachable`**: Java 21, все 53 пода Running, GC/JIT-регрессий нет; единственное — Hazelcast client 5.6 не может достучаться до недоступного .66 (это НЕ Java 21 регрессия, это инфра).
- **`taxreport21-java21-runtime-regressions`**: два вида регрессий появились:
  1. **LazyInit** (Hibernate 6 стал строже) — не про Java 21, но про новую версию Hibernate которая пришла вместе.
  2. **OAuth NoSuchMethodError** (spring-security jose/core skew в NZ+320) — конфликт версий транзитивных зависимостей.
- **`knp-e2e-prod-java21-smoke-ops`**: как гонять прод-смок против knp21 контура (Java 21) локально.

Вывод: сам Java 21 в проде **стабилен**. Проблемы — от параллельно обновляемых Hibernate 6, Spring Boot 3, Hazelcast 5.

---

## 11. Диагностика в проде

### 11.1 GC log

```
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=10M
```

Ищи:
- Долгие паузы (`Pause Full` > 500 мс = плохо).
- Частые Young GC (аллокационное давление).
- Тренд Old size вверх без освобождения (утечка).

### 11.2 Native Memory Tracking

```
-XX:NativeMemoryTracking=summary
jcmd <pid> VM.native_memory summary
```

Показывает где куда память ушла: heap, metaspace, code cache, threads, direct memory.

### 11.3 jcmd

Универсальный:
```bash
jcmd <pid> VM.version           # версия JVM
jcmd <pid> Thread.print         # thread dump
jcmd <pid> GC.heap_info         # состояние heap
jcmd <pid> GC.run               # forced GC
jcmd <pid> JFR.start duration=60s filename=r.jfr
```

---

## 12. Собесные вопросы

1. **Какой GC по умолчанию в Java 21?** — G1.
2. **ZGC — когда использовать?** — Low-latency приложения, большой heap, паузы <10 мс.
3. **Разница G1 и ZGC?** — G1 region-based, короткие STW; ZGC concurrent compaction, паузы <1 мс, но больше overhead.
4. **Что удалено между 11 и 21?** — CMS GC, Nashorn, Applets, SecurityManager (deprecated).
5. **Что такое container awareness?** — JVM видит cgroup-лимиты (memory, CPU), не хостовые.
6. **`-XX:MaxRAMPercentage`?** — Процент от cgroup limit для heap.
7. **Что такое CDS/AppCDS?** — Class Data Sharing, ускоряет старт за счёт предкомпилированных метаданных.
8. **GraalVM Native Image — за и против?** — За: 100 мс старт, малая память. Против: долгий build, нет JIT, тонкая настройка reflection.
9. **Что такое JFR?** — Java Flight Recorder, встроенный профилировщик, бесплатный с Java 11.
10. **Metaspace — что и когда OOM?** — Область метаданных классов; OOM при утечке ClassLoader'ов.
11. **Что нового в Java 21 в GC?** — Generational ZGC.
12. **Почему Java 21 быстрее стартует чем 11?** — CDS default, оптимизации класслоадера, лучший JIT.

---

## Итог

- **G1** — default, работает везде.
- **ZGC** — для low-latency и big heap.
- **Container awareness** с Java 10+; фиксируй `MaxRAMPercentage`.
- **CDS/AppCDS** — ускорение старта.
- **GraalVM Native** — крайний случай для быстрого старта.
- **JFR** — правильный инструмент для профилирования в проде.
- **Metaspace** — не забывать лимит + следить за утечками ClassLoader.

Следующий — `18-java-11-to-21-api-migration.md`.
