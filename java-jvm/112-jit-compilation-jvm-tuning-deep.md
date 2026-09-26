# 112. JIT compilation и JVM tuning: HotSpot изнутри

## Зачем это знать

Java-приложение стартует медленно, «прогревается» под нагрузкой, потом работает быстро. Все это знают. Мало кто понимает **почему** — что реально происходит в моменты «прогрева», почему первые тысячи запросов медленнее, почему иногда после долгой работы производительность внезапно падает («deoptimization storm»), почему разные микробенчмарки дают разные результаты в зависимости от того как их писать.

Ответ — в JIT. Just-In-Time compiler в HotSpot — сложная многоуровневая система, которая наблюдает за исполнением bytecode и превращает горячие методы в оптимизированный native код. Работает автоматически, но параметры влияют. Знать как — разница между «странно медленно, увеличим heap» и «код cache переполнен, надо `-XX:ReservedCodeCacheSize=512M`». Между «загадочная деоптимизация» и «понял, у меня три реализации interface, inline cache стал megamorphic, метод перестал inlined».

JVM tuning — не только GC (тот в 111). Есть ещё code cache, thread stacks, metaspace, direct memory, JIT compilation threads, ergonomics vs manual sizing, container awareness. Каждая область — свои параметры, свои trade-off'ы. Правило то же что для GC: **не тюньте наугад**. Измерять, менять один параметр, снова измерять. Дефолты Java 21 в контейнерах хорошие, вмешательство нужно только когда доказана проблема.

Разберём: базу — почему interpreter + JIT, а не «просто компилятор». HotSpot two compilers (C1 «client», C2 «server») и tiered compilation с пятью tiers. Triggers для компиляции — invocation counter, back-edge counter, method size limits. Method inlining как самая важная оптимизация и что её ломает. Escape analysis + scalar replacement — почему `new` может не аллоцировать. Deoptimization — когда C2 «сдаётся» и возвращает управление interpreter'у. Inline caches — monomorphic / bimorphic / megamorphic и как они деградируют. Code cache — где хранится native код и что если переполнится. GraalVM как альтернативный C2. AOT compilation (Native Image). Практическая диагностика: `-XX:+PrintCompilation`, LogCompilation + JITWatch, JFR events. JVM tuning в целом — heap sizing формулы, thread stacks, metaspace, direct memory, container awareness через cgroups (Java 11+). Практические рецепты per workload. Диагностика проблем JIT: compilation queue full, code cache full, deopt storms.

## Interpreter и JIT: почему две фазы

Классический компилируемый язык (C, Rust) компилирует код заранее в native. Классический интерпретируемый (Python старый, Ruby MRI) исполняет исходник строчка за строчкой. Java выбрала **гибридный подход** — bytecode + runtime компиляция.

Почему не сразу компилировать всё в native? Три причины:

1. **Startup time**. Компиляция дорогая. Если компилировать всё сразу, приложение стартует медленно. Interpreter умеет запустить bytecode немедленно.
2. **Profile-guided optimization**. Компилируя после наблюдения за реальным исполнением, можно оптимизировать под реальный workload: какие ветки if чаще true, какие типы реально приходят в polymorphic call sites, какие циклы горячие. Ahead-of-time компилятор такое не знает.
3. **Space efficiency**. Native код в разы больше bytecode. Если компилировать всё — code cache огромный. Компилируем только hot methods — остальное в interpreter.

Java runtime решение: интерпретировать всё сначала (быстрый старт), собирать профиль (какие методы горячие, какие типы приходят), компилировать hot методы в native (быстрое исполнение), при необходимости деоптимизировать обратно (если предположения оптимизатора нарушились).

## HotSpot: C1 и C2

HotSpot содержит **два JIT-компилятора**:

**C1 (client compiler)** — быстрый, простые оптимизации. Компилирует за миллисекунды. Полученный код в 2-5 раз быстрее interpreter'а, но не оптимальный. Изначально задумывался для клиентских (short-lived, desktop) приложений где важен быстрый старт.

**C2 (server compiler)** — медленный, агрессивные оптимизации. Компиляция занимает сотни миллисекунд-секунды. Полученный код в 10-100 раз быстрее interpreter'а, приближается к оптимизированному C++. Задумывался для серверных приложений.

До Java 8 можно было выбрать только один: `-client` или `-server`. С Java 8 tiered compilation по умолчанию — используются **оба**.

## Tiered compilation: 5 tiers

Метод в HotSpot проходит через **5 уровней**:

- **Tier 0** — interpreter. Первый запуск, собирается базовый профиль.
- **Tier 1** — C1 без профилирования. Максимально быстро, без счётчиков. Используется если C2 недоступен.
- **Tier 2** — C1 с частичным профилированием. Собирает базовые счётчики (invocation, back-edge).
- **Tier 3** — C1 с полным профилированием. Плюс type profile для virtual calls, счётчики branch (какая ветка если чаще).
- **Tier 4** — C2 (или GraalVM). Использует профиль из tier 3 для агрессивных оптимизаций.

**Обычный путь метода**: 0 → 3 → 4. Interpreter, потом C1 с профилированием, потом C2.

**Быстрые (не hot) методы**: могут остаться на tier 0 или уйти в tier 2 (без профилирования, C2 всё равно не понадобится).

**Compilation queue переполнен**: метод может уйти сразу в tier 1 (C1 без профилирования), потом позже в tier 4.

Опция для просмотра: `-XX:+PrintCompilation`:

```
    123    5       3       java.lang.String::indexOf (70 bytes)
    145    6       4       java.util.HashMap::get (23 bytes)
    167    7       3   %   com.example.Foo::process (150 bytes)
```

Формат: `[время в мс] [compile ID] [tier] [флаги] [метод (размер)]`. `%` означает on-stack replacement (OSR, компиляция цикла прямо во время его исполнения).

## Triggers для компиляции

Метод компилируется когда становится «горячим». Определяется через **счётчики**:

**Invocation counter** — сколько раз метод вызван. Инкрементируется при каждом входе.

**Back-edge counter** — сколько раз выполнена итерация цикла внутри метода. Инкрементируется на каждой back edge (переход назад в bytecode).

При достижении thresholds метод отправляется в compilation queue соответствующего tier.

Дефолтные thresholds (tiered):
- Tier 3 (C1 with profiling): invocation ≥ 200 or back-edges ≥ 4000.
- Tier 4 (C2): invocation ≥ 5000 or back-edges ≥ 15000.

Опция: `-XX:CompileThreshold=<N>` (по умолчанию 10000 для non-tiered, для tiered есть отдельные `-XX:TierNInvocationThreshold`).

Если метод содержит долгий цикл — **on-stack replacement** (OSR). Компилируется прямо во время исполнения, JVM переключает управление на native код на середине цикла (перенося local variables). Иначе один вызов метода со миллионом итераций никогда не был бы скомпилирован (invocation counter = 1).

**Method size limits**. Слишком большие методы не компилируются или компилируются с ограничениями:
- `-XX:MaxInlineSize=35` (bytes) — метод для inlining должен быть меньше.
- `-XX:FreqInlineSize=325` — для «горячих» методов лимит выше.
- `-XX:HugeMethodLimit=8000` — методы больше не компилируются C2.

Из этого правило: **держи методы маленькими**. 8000-байтовый метод (мегабайты сгенерированного кода typically в byte code) — не будет оптимизирован. Разбить на несколько.

## Method inlining: самая важная оптимизация

**Inlining** — подстановка тела вызываемого метода в место вызова. `a.foo()` заменяется на реальный код `foo()`, вставленный inline.

Почему это критично:
- **Устраняет overhead вызова** (сохранение frame, jump, восстановление) — сотни наносекунд экономятся.
- **Открывает возможности для дальнейших оптимизаций**. После inline JIT видит контекст, может делать constant folding, dead code elimination, escape analysis на inlined коде.

Пример. Оригинальный код:

```java
int sum = 0;
for (int i = 0; i < 1000; i++) {
    sum += compute(i);
}

int compute(int x) {
    return x * 2 + 1;
}
```

После inlining:

```java
int sum = 0;
for (int i = 0; i < 1000; i++) {
    sum += i * 2 + 1;
}
```

Дальше — loop unrolling, vectorization (SIMD инструкции), устранение мёртвого кода. Итог: цикл превращается в несколько CPU-инструкций.

**Что мешает inlining**:
- **Размер метода** > `-XX:MaxInlineSize` (35 bytes для не-hot методов).
- **Виртуальные вызовы с megamorphic inline cache** — если тип не известен, inline невозможен (не знаем какое тело подставить).
- **native методы** — inline невозможен (нет bytecode).
- **synchronized блоки** — могут блокировать inline (зависит от контекста).

Опция для просмотра решений inlining: `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining`:

```
@ 5   com.example.Foo::compute (12 bytes)   inline (hot)
@ 5   com.example.Bar::foo (200 bytes)     too big
@ 5   com.example.Baz::run (50 bytes)      not inlineable (megamorphic call)
```

## Inline caches: monomorphic / bimorphic / megamorphic

Виртуальный вызов (`interface method`, `abstract method`) в Java — головная боль для JIT: неизвестно какая реализация будет вызвана. HotSpot использует **inline caches** для оптимизации.

**Monomorphic** — если во всех наблюдаемых вызовах приходил один тип. JIT кэширует: «если тип X, вызываем X.method() (inline можно)». Быстро, `getClass() == X` проверка + inline тело.

**Bimorphic** — два разных типа. JIT кэширует оба: «если X — так, если Y — эдак, иначе fallback». Всё ещё быстро.

**Megamorphic** — три и больше типов. JIT сдаётся, генерирует обычный vtable lookup. Медленно, inline невозможен.

Практический эффект: **держи interface implementations редкими на hot paths**. Если у тебя `List<Handler>` и в hot loop `handler.handle()` вызывается 10 разных реализаций — call site megamorphic, никакого inline, медленно. Если бы был один — 100x быстрее (после inlining).

Обход в дизайне: sealed interfaces, generic вместо interface, разделение hot paths по типам. Не всегда возможно, но осознавать стоит.

## Escape analysis и scalar replacement

JIT-оптимизация, которая может **не аллоцировать** объекты которые синтаксически создаются через `new`.

**Escape analysis** — анализ «уходит ли объект за границы метода». Если объект создан, используется только внутри метода и никуда не «убегает» (не сохраняется в поле, не возвращается, не передаётся другому потоку) — можно оптимизировать.

**Scalar replacement** — объект заменяется на его отдельные поля (scalars), которые кладутся в CPU регистры или на stack. Никакой heap allocation, никакой GC pressure.

Пример:

```java
public int compute(int x, int y) {
    Point p = new Point(x, y);
    return p.x + p.y;
}
```

Escape analysis: `p` не уходит из метода → scalar replacement. После оптимизации `Point` не создаётся вообще, только два int в регистрах.

**Что ломает escape analysis**:
- `synchronized` на объекте (нужен real objects для monitor).
- Передача объекта в неinlined метод (JIT не знает что тот с ним сделает).
- Присвоение в static или instance поле.
- Использование в `Thread.start()` или подобных escape-путях.

Из этого — **не бойтесь маленьких short-lived объектов в hot path**. HotSpot их часто оптимизирует до нуля аллокаций. `new Point(x, y).distance(other)` может быть буквально бесплатно.

Опция для просмотра: `-XX:+UnlockDiagnosticVMOptions -XX:+PrintEscapeAnalysis`.

## Deoptimization: когда JIT «сдаётся»

C2 делает **спекулятивные оптимизации** на основе профиля. «В tier 3 все 10000 вызовов приходил только один тип → компилирую с inline того типа». Что если после компиляции приходит другой тип?

**Deoptimization**. JIT встраивает **guards** — проверки предположений. При нарушении предположения (тип другой, ветка не та, класс загрузился с новым методом) — управление передаётся обратно interpreter'у, native код помечается как invalid.

Причины deoptimization:
- **Class loading** — новый класс подгружен, mono-morphic call site стал bimorphic. Скомпилированный код с inline устарел.
- **Uncommon trap** — ветка кода которая на этапе профилирования не выполнялась, вдруг выполняется. Native код не содержит оптимизации для неё.
- **Type mismatch** — inline cache guard провалился.
- **Class hierarchy change** — новый subclass переопределил метод.

Deoptimization дорогая: переход interpreter → C1 → снова C2 занимает миллисекунды. Если случается регулярно (**deopt storm**) — производительность деградирует.

Опция для наблюдения: `-XX:+PrintCompilation` показывает deoptimizations как строку типа `made not entrant` или `made zombie`.

Ловля deopt storm через JFR event `jdk.Deoptimization`. Периодические deopts на одном месте — сигнал что-то не так с профилем (workload изменился) или полиморфизм.

## Code cache: где живёт скомпилированный код

Native код, произведённый JIT, хранится в **code cache** — специальном участке памяти вне heap. Размер по умолчанию — 240 MB (`-XX:ReservedCodeCacheSize=240m`).

Code cache разделён на три сегмента (Java 9+, `-XX:+SegmentedCodeCache`):
- **Non-nmethods** — VM internal code (interpreter, stubs).
- **Profiled nmethods** — C1 tier 3 (короткоживущие, будут replaced на C2).
- **Non-profiled nmethods** — C2 tier 4 (долгоживущие).

**Что если code cache переполнен**:

```
CodeCache is full. Compiler has been disabled.
```

JIT перестаёт компилировать новые методы. Все hot методы, которые ещё не скомпилированы, остаются в interpreter. Производительность резко падает.

Симптомы в проде: приложение работало хорошо, через несколько часов начало тормозить, throughput упал в разы. `jstat -codecache` показывает use близко к max.

Обход: увеличить `-XX:ReservedCodeCacheSize=512m` (для больших приложений часто нужно). Мониторить через JMX bean `java.lang:type=MemoryPool,name=CodeHeap 'profiled nmethods'` etc.

## GraalVM: альтернативный C2

**GraalVM** — Java-написанный JIT-компилятор, разработанный Oracle Labs как замена C2. Использует те же интерфейсы HotSpot, но реализует более агрессивные оптимизации особенно для функционального стиля Java (stream API, lambdas).

Опция: `-XX:+UnlockExperimentalVMOptions -XX:+UseJVMCICompiler`. С Java 21 GraalVM Community Edition часть JDK distribution от Oracle.

Реальный эффект: для stream-heavy кода 5-30% быстрее C2. Для обычного OOP — сопоставимо или чуть медленнее. Компиляция несколько дольше (GraalVM написан на Java, сам компилируется JIT'ом бутстрап проблема).

Для большинства enterprise workloads разница минимальна. Если приложение stream-heavy или functional-style — попробовать.

## AOT compilation: GraalVM Native Image

**Native Image** — полная ahead-of-time компиляция в native executable. Не bytecode + JVM, а готовый бинарник для целевой платформы.

Плюсы:
- Мгновенный старт (миллисекунды vs секунды у JVM).
- Маленький footprint памяти (нет JIT, code cache, metaspace).
- Идеально для serverless / Lambda (cold start).
- Меньше surface для CVE.

Минусы:
- Долгая компиляция (минуты-десятки минут).
- Ограничения (reflection, dynamic class loading требуют hints через config files).
- Peak performance хуже чем JIT'ованный код (JIT в runtime знает больше о профиле).
- Не все библиотеки поддерживают (проверяют через `native-image --tracing-agent` при обычном run для сбора reflection hints).

Actual use case: CLI tools, serverless functions, edge computing. Для long-running enterprise servers обычная JVM с JIT лучше.

## Диагностика JIT в проде

**`-XX:+PrintCompilation`** — базовый инструмент. Пишет в stdout каждую компиляцию. Для production обычно слишком многословно (тысячи строк). Для локального анализа — незаменимо.

**`-XX:+LogCompilation -XX:+UnlockDiagnosticVMOptions`** — детальный XML лог всех решений JIT (какой метод, почему компилируется, что inlined, что нет). Огромный файл. Открывать через **JITWatch** — visual tool для анализа. Показывает graph компиляций, ассемблер (если включить `-XX:+PrintAssembly` с hsdis), inlining decisions.

**JFR events для JIT**:
- `jdk.Compilation` — каждая компиляция.
- `jdk.CompilerFailure` — неудачная компиляция.
- `jdk.Deoptimization` — деоптимизации.
- `jdk.CodeCacheFull` — переполнение кэша.

Собирается через:
```bash
jcmd <pid> JFR.start duration=60s filename=jit.jfr settings=profile
```

Открывать в JMC → JIT Compiler section.

**jstat для быстрого взгляда**:
```bash
jstat -compiler <pid>
# Compiled  Failed  Invalid   Time   FailedType FailedMethod
jstat -printcompilation <pid> 1000
# Каждую секунду список активных compilations
```

## JVM tuning в целом: не только GC

**Heap sizing** — обсуждено в 111. Правило: `-Xms == -Xmx` во избежание resize. Для контейнеров 70-80% от cgroup memory.limit.

**Thread stack** — `-Xss<size>` (default 1 MB на Linux 64-bit). Каждый platform thread резервирует эту память под stack. 1000 threads = 1 GB виртуальной памяти. Обычно уменьшают до 256-512 KB (`-Xss256k`) для приложений с большим числом тредов. Для VT (файл 110) не применяется — VT имеют heap-based stack.

**Metaspace** — где хранятся class metadata. По умолчанию — без лимита (растёт с загрузкой классов). Обычно ставят `-XX:MaxMetaspaceSize=256m` (или больше для приложений с много классами — Spring apps 512-1024 MB). Без лимита возможен неконтролируемый рост при classloader leaks (hot deploy без правильного unload).

**Direct memory** — off-heap memory для NIO buffers (Netty, gRPC, JDBC drivers, protobuf). По умолчанию — `-Xmx` (сколько heap). Часто нужно ограничить: `-XX:MaxDirectMemorySize=512m`. Иначе может съесть всю доступную память контейнера мимо heap → OOMKilled.

**Container awareness (Java 11+)** — JVM в контейнере автоматически:
- Читает `/sys/fs/cgroup/memory/memory.limit_in_bytes` для расчёта available memory.
- Читает `/sys/fs/cgroup/cpu/cpu.shares` для расчёта available CPUs (влияет на GC threads, ForkJoinPool, JIT threads).

Обычно ergonomics подбирает нормально. Явно можно указать `-XX:MaxRAMPercentage=75.0` (75% от cgroup memory на heap).

**Compilation threads** — `-XX:CICompilerCount=N`. Обычно autoselect (2 для C1, N для C2 в зависимости от ncores). Для heavy startup workloads можно увеличить (`-XX:CICompilerCount=6`) — быстрее прогрев.

## Практические рецепты

**Обычный web-сервис (heap 4-8 GB, container с 4 CPU)**:

```
-Xms8g -Xmx8g
-XX:+UseG1GC
-XX:MaxMetaspaceSize=512m
-XX:MaxDirectMemorySize=1g
-XX:ReservedCodeCacheSize=256m
-Xss512k
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heap.hprof
-Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=10,filesize=100M
```

**Latency-critical service (heap 32 GB, ZGC)**:

```
-Xms32g -Xmx32g
-XX:+UseZGC
-XX:+ZGenerational
-XX:MaxMetaspaceSize=1g
-XX:MaxDirectMemorySize=2g
-XX:ReservedCodeCacheSize=512m
-XX:CICompilerCount=6
-Xlog:gc*:file=/var/log/gc.log
```

**Serverless / short-lived (Lambda)**:

Либо GraalVM Native Image (лучший выбор для sub-second cold start), либо:

```
-Xms256m -Xmx256m
-XX:+UseSerialGC
-XX:TieredStopAtLevel=1  # только C1, быстрый прогрев
-Xshare:auto  # использовать shared class data
```

Отключение C2 (`TieredStopAtLevel=1`) — жертвуем peak performance ради startup time.

**Batch job (throughput критичен)**:

```
-Xms16g -Xmx16g
-XX:+UseParallelGC
-XX:ParallelGCThreads=16
-XX:MaxDirectMemorySize=2g
-XX:ReservedCodeCacheSize=256m
```

## Диагностика проблем JIT

**Симптом: приложение работало ok, через N часов начало тормозить**.

Проверить code cache: `jstat -codecache <pid>`. Если использование близко к 100% max — переполнение. Увеличить `-XX:ReservedCodeCacheSize`.

Симптом logging: `CodeCache is full. Compiler has been disabled` в stderr.

**Симптом: деоптимизации в JFR**.

`jdk.Deoptimization` events много и на одном call site. Обычно причины: (1) новый класс подгружен превратил monomorphic в bimorphic/megamorphic; (2) uncommon trap — ветка кода которая раньше не выполнялась вдруг стала выполняться (например, workload изменился).

Обход: полиморфизм упростить, редкие ветки переписать чтобы не выглядели редкими (или, наоборот, вынести в отдельные методы).

**Симптом: приложение медленное первые минуты**.

Прогрев JIT. Нормально. Обход:
- **Warmup phase** перед приёмом трафика — прогнать типовые запросы искусственно (readiness probe с realistic workload).
- **`-XX:CompileThreshold=100`** — компилировать раньше (пожертвовать точностью профиля ради быстрого прогрева).
- **AppCDS (Application Class Data Sharing)** — предварительно построенный class data archive ускоряет class loading при старте.
- **GraalVM Native Image** — для случаев где cold start критичен.

**Симптом: `Method too big` в PrintCompilation**.

Метод > `HugeMethodLimit` (8000 bytes). C2 не компилирует. Рефакторить на меньшие методы.

**Симптом: high CPU в JIT thread ('C2 CompilerThread')**.

Много компиляций одновременно (обычно при старте). Норма — со временем стабилизируется. Если постоянно — deopt storm (см. выше) или dynamic class generation (генерируются новые классы через ByteBuddy/CGLIB — JIT их пытается скомпилировать).

## Warmup стратегии

Для сервисов чувствительных к latency сразу после старта:

**1. Load test перед приёмом трафика**. Отдельный warmup phase — приложение стартует, healthcheck возвращает NOT_READY, load generator шлёт synthetic requests, после N tысяч запросов healthcheck → READY, начинает получать трафик. Даёт time для JIT прогреться.

**2. AppCDS (Application Class Data Sharing)**. `-XX:ArchiveClassesAtExit=app.jsa` при первом запуске создаёт archive загруженных классов. Дальше `-XX:SharedArchiveFile=app.jsa` — при старте классы загружаются из archive (быстрее), memory-mapped (shared между JVM instances на той же машине). Ускорение startup 30-50%.

**3. GraalVM Native Image** — если приемлемы ограничения.

**4. CRaC (Coordinated Restore at Checkpoint)** — Java 21+, экспериментально. Приложение прогревается, делаем checkpoint (snapshot состояния процесса), при рестарте — restore из checkpoint. Прогрев уже был, JIT-скомпилированный код готов. Cold start в миллисекундах. Проблема — не все библиотеки поддерживают (нужно правильно handle file descriptors, connections).

## Заключение

JIT в HotSpot — сложная система: interpreter собирает профиль, C1 быстро компилирует горячие методы с профилированием, C2 (или GraalVM) агрессивно оптимизирует по собранному профилю. Tiered compilation (5 tiers, обычно путь 0 → 3 → 4) даёт баланс startup vs peak performance.

**Method inlining** — самая важная оптимизация. Устраняет overhead вызова и открывает дальнейшие оптимизации (constant folding, dead code elimination, escape analysis). Ограничивается размером метода (`MaxInlineSize=35`, `FreqInlineSize=325`) и megamorphic call sites.

**Escape analysis + scalar replacement** — объекты не аллоцируются на heap если не «убегают» из метода. Синтаксический `new` может не вызвать GC pressure.

**Inline caches** — monomorphic (быстро), bimorphic (нормально), megamorphic (медленно, inline невозможен). Полиморфизм на hot paths стоит дорого.

**Deoptimization** — C2 сдаётся когда предположения нарушены (новый тип, class loading, uncommon trap). Возврат в interpreter, потом снова компиляция. Deopt storm — постоянные деоптимизации, деградация производительности.

**Code cache** — 240 MB по умолчанию. Переполнение (`CodeCache is full`) отключает JIT. Симптом «работал ok, начал тормозить через часы». Увеличить `-XX:ReservedCodeCacheSize=512m`.

**GraalVM** — альтернативный C2. 5-30% быстрее для stream/functional Java, comparable для OOP. **Native Image** — полная AOT компиляция. Мгновенный старт, малый footprint, ограничения (reflection нужны config), хуже peak performance чем JIT. Для serverless/Lambda.

**JVM tuning в целом**: `-Xms == -Xmx` (избежать resize); `-Xss256-512k` для приложений с много тредами (не для VT); `-XX:MaxMetaspaceSize` обязательно (иначе может расти неконтролируемо при classloader leaks); `-XX:MaxDirectMemorySize` для off-heap (Netty, JDBC); container awareness Java 11+ auto через cgroups (обычно достаточно).

**Диагностика**: `-XX:+PrintCompilation` для локального анализа; `-XX:+LogCompilation` + JITWatch для глубокого; JFR events (`jdk.Compilation`, `jdk.Deoptimization`, `jdk.CodeCacheFull`) для production; `jstat -codecache`, `jstat -compiler` для quick check.

**Практические рецепты**: web-сервис — G1 + разумные лимиты metaspace/direct/codecache; latency-critical — ZGC + больше compilation threads; serverless — Serial GC + `TieredStopAtLevel=1` (только C1) или GraalVM Native; batch — Parallel GC.

**Проблемы**: code cache переполнен (увеличить); deopt storm (упростить полиморфизм); долгий warmup (AppCDS, artificial warmup, CRaC); huge method (рефакторить).

Правило JVM tuning: не крутить наугад. Дефолты Java 21 в контейнере обычно хорошие. Тюнить только когда доказана проблема через метрики или JFR. Один параметр за раз, замерять эффект, документировать в codebase зачем именно этот параметр стоит.

GC deep — файл 111. VT — 110. JVM/OS/RAM/native — 84. Threads — 83. Здесь была глубина по JIT (два компилятора, tiered, inlining, escape analysis, deopt, code cache) и общий JVM tuning (все области кроме GC — metaspace, direct memory, thread stacks, container awareness), с практическими рецептами per workload и диагностикой в проде.
