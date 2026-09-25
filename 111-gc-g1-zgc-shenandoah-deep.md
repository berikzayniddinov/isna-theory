# 111. Garbage Collectors: G1, ZGC, Shenandoah глубоко

## Зачем это знать

GC — самая непонятная часть JVM для большинства инженеров. Работает «сам по себе», пока не работает. Симптомы прод-инцидентов: длинные stop-the-world паузы (сервис не отвечает 5 секунд), OutOfMemoryError несмотря на «есть свободная память», странное поведение под нагрузкой (throughput проседает, latency растёт периодами). Без модели того как GC устроен внутри — эти инциденты диагностируются гаданием.

Разница между «знаю что есть GC» и «понимаю GC» — это способность прочитать GC-log и увидеть за строками работу конкретных алгоритмов. Понять почему у G1 «пилообразный» heap, почему ZGC даёт паузы в миллисекунды даже на 100GB heap, почему Shenandoah существует параллельно с ZGC, что такое humongous allocation и как её ловить. Знать какой GC подобрать под workload: throughput-heavy batch → Parallel GC; low-latency HTTP-сервис → ZGC или Shenandoah; общий баланс на heap 4-32 GB → G1 (default с Java 9).

Одна из главных ошибок — тюнить GC параметры («попробуем поставить `-XX:MaxGCPauseMillis=50`») не понимая что за ними стоит. GC — не magic knob, а конкретный алгоритм с trade-offs. Уменьшил цель по паузе → GC делает больше маленьких коллекций → throughput падает. Увеличил heap → маленькие коллекции реже, но большие дольше. Всё связано, каждый параметр — компромисс.

Разберём: базу — что такое garbage, reachability analysis, GC roots. Generational hypothesis и почему young/old. Основные фазы (mark/sweep/compact) и общие свойства (STW vs concurrent, tracing, generational). Каждый GC отдельно: Serial (исторический, embed'ы и мелкие CLI), Parallel (throughput-oriented, batch), G1 (default, регионы, mixed GC, remembered sets, SATB), ZGC (colored pointers, load barriers, sub-ms паузы), Shenandoah (Brooks forwarding, concurrent compaction, ARM-friendly). Сравнение с числами. Как выбирать под workload. Тюнинг: какие параметры реально влияют и как. Диагностика в проде: чтение GC-log, jstat, JFR, где искать причину плохих пауз. Типовые проблемы: humongous allocations, promotion failures, allocation stalls, string dedup. Реальные prod-рецепты.

## База: что такое garbage и reachability

Объект в куче становится «мусором» (garbage) когда до него **больше не дойти** ни из одного живого корня программы. Определение через reachability — способ формально решить «нужен ли этот объект» без ручного отсчёта reference count.

**GC roots** — стартовые точки reachability-анализа:

- **Локальные переменные и параметры** всех активных методов (frames на stack всех тредов).
- **Static поля** всех загруженных классов.
- **JNI references** — объекты удерживаемые native кодом через `NewGlobalRef`.
- **Внутренние JVM объекты** — Thread objects, ClassLoader'ы, некоторые внутренние структуры.
- **Synchronized monitors** — объекты, на которых кто-то держит monitor lock.
- **Finalizer queue** — объекты ждущие finalize().

GC делает **transitive closure**: от roots обходит все ссылки, помечает достижимые. Всё что не помечено — garbage.

Отсюда — фундаментальный алгоритм **Mark & Sweep**:

1. **Mark**: обход графа объектов от roots, пометка достижимых.
2. **Sweep**: обход всей heap, освобождение непомеченных.

Простой mark-sweep оставляет **фрагментацию** — свободная память разбита на дыры разного размера между живыми объектами. Аллокация большого объекта может провалиться даже при формально свободных гигабайтах.

Решение — **Compact**: после sweep живые объекты сдвигаются в начало heap, свободная память становится единым непрерывным блоком. Дорогая операция (перемещение объектов, обновление всех ссылок на них), но необходимая для долгоживущего процесса.

Есть альтернативный подход — **Copying GC**: heap делится на две половины, аллокация идёт в одну, при GC живые объекты копируются в другую (компактно), первая полностью освобождается. Просто и быстро, но эффективно только для young generation (мало живых объектов).

## Generational hypothesis: почему young/old

Эмпирическое наблюдение из десятилетий работы GC: **большинство объектов умирают молодыми**. Web-запрос создаёт временные DTO, парсеры создают промежуточные структуры, StringBuilder'ы, iterator'ы — всё это живёт миллисекунды и умирает. Только небольшая часть объектов (кэши, пулы соединений, singleton bean'ы) живёт долго.

Из этого — **generational hypothesis**: если разделить объекты на young (недавно созданные) и old (пережившие несколько GC), можно оптимизировать. Young — часто, быстро (copying GC, много мусора). Old — редко (компактно, большинство живо).

Классическая архитектура (Serial, Parallel, до-G1 CMS):

```
        Young Generation                    Old Generation
    ┌────────┬───────┬───────┐         ┌────────────────────┐
    │  Eden  │  S0   │  S1   │         │      Tenured       │
    └────────┴───────┴───────┘         └────────────────────┘
    ▲                                     ▲
    new                                   promote после N GC
```

**Eden** — куда идут все новые объекты. **Survivor** (S0/S1) — куда копируются пережившие minor GC (обычно 2 space'а для copying). **Tenured (Old)** — куда попадают объекты пережившие несколько minor GC (обычно 15, `-XX:MaxTenuringThreshold`).

**Minor GC** (young collection) — быстрый, только young. Live objects из Eden копируются в one Survivor. Live objects из другого Survivor копируются в тот же (или в Old если tenuring). Eden обнуляется. Обычно паузы в миллисекунды, потому что young маленький и живых объектов мало.

**Major GC / Full GC** — обход всей heap включая Old. Дороже, паузы сотни миллисекунд-секунды. Модерн GC (G1, ZGC, Shenandoah) стараются избежать Full GC, но он остаётся как fallback.

## Классификация GC

Свойства, по которым GC различаются:

**Stop-the-world (STW) vs concurrent**. STW — приложение полностью останавливается на время GC (все application threads paused). Concurrent — GC работает параллельно с приложением, паузы только в конкретных фазах (обычно короткие для начала/конца mark). Concurrent сложнее (нужны write barriers, synchronization с mutator threads), но даёт короткие паузы.

**Tracing vs reference counting**. Tracing — mark & sweep от roots (все JVM GC). Reference counting — каждый объект хранит счётчик ссылок (Python, Swift). Reference counting не работает для циклов, требует дополнительной коллекции для них.

**Generational vs monolithic**. Generational — Young/Old split (Serial, Parallel, G1). Monolithic — вся heap как один пул (ZGC, Shenandoah изначально; в новых версиях оба получили generational вариант).

**Region-based vs contiguous**. Region-based — heap разбит на много мелких регионов (1-32 MB для G1, 2^n MB для ZGC), GC работает с регионами независимо. Позволяет частичный collection. Contiguous — Young/Old как непрерывные блоки (старые GC).

**Compacting vs non-compacting**. Компактит после sweep (все современные) или нет (CMS не компактил старое поколение — источник его фрагментации и депрекации).

## Serial GC — история и когда до сих пор используется

Простейший, single-threaded, полностью STW. Аллокация в Eden, copying для young, mark-sweep-compact для old. Одним тредом, все application threads паузятся.

Опция включения: `-XX:+UseSerialGC`.

Плюсы: минимум overhead, наименьший heap footprint (нет метаданных для parallelism), простой. Идеально для embed-систем, CLI-инструментов, малых контейнеров с 100-200 MB heap.

Минусы: single-threaded → на многоядерных системах не использует ресурсы. Паузы масштабируются с размером heap. Для heap > 1 GB паузы становятся неприемлемыми.

Where used today: JVM в маленьких Docker container'ах (< 500 MB heap), Cloud Functions/Lambda cold start (быстрый init), embedded systems. Java 21 автоматически выбирает Serial для контейнеров с < 2 GB memory и 1 CPU.

## Parallel GC — throughput champion

Тот же алгоритм что Serial (copying young, mark-sweep-compact old), но параллельный: несколько GC threads (по умолчанию = ncores) одновременно обрабатывают collection. Всё ещё STW — application threads стоят, но GC threads работают параллельно.

Опция: `-XX:+UseParallelGC`. Число GC threads: `-XX:ParallelGCThreads=N`.

Плюсы: максимальный throughput. Приложение делает больше полезной работы за единицу времени, чем с любым concurrent GC (concurrent тратит cycles на coordination). Отлично для batch processing, где latency отдельных операций не важна, важна общая пропускная способность.

Минусы: паузы растут с heap. На 8 GB heap Full GC пауза может быть 2-5 секунд. Для user-facing систем — неприемлемо.

Where used today: batch jobs (ETL, ML training, data processing), где throughput критичен. До Java 9 был дефолтом.

## G1 GC — дефолт с Java 9

**G1** (Garbage-First) — компромисс между throughput и latency. Дефолтный сборщик Java 9+ для heap ≥ 2 GB.

Ключевая идея — **region-based heap**. Heap разбит на равные регионы (по 1, 2, 4, 8, 16 или 32 MB, автоматически или через `-XX:G1HeapRegionSize`). Каждый регион в моменте — либо Eden, либо Survivor, либо Old, либо Humongous, либо Free. Роли регионов **меняются** — регион был Eden, после young GC стал Survivor, потом Free, потом опять Eden.

```
[E][E][E][S][O][O][H][H][F][E][O][O][F][F][S][O]...
 E=Eden, S=Survivor, O=Old, H=Humongous, F=Free
```

**Humongous** — регион для объектов размером > 50% размера региона. Аллоцируются напрямую в Old (не проходят через young). Занимают целые регионы, если объект больше — цепочку смежных. Источник проблем (см. ниже).

**Young collection** в G1 — STW, копирует живые объекты из Eden в Survivor или Old. Параллельно, короткие паузы (10-100 ms обычно).

**Mixed collection** — G1 в одном collection cycle обрабатывает young регионы **и часть old** регионов (выбирает те где больше всего garbage — отсюда «Garbage-First»). Позволяет постепенно освобождать Old без длинного Full GC.

**Concurrent Mark cycle**:

1. **Initial Mark** — STW, короткая пауза, помечает roots.
2. **Concurrent Marking** — параллельно с приложением, обходит граф от roots.
3. **Remark** — STW, финализирует mark (обрабатывает изменения сделанные mutator'ами через **SATB — Snapshot At The Beginning**).
4. **Cleanup** — считает garbage per region, готовит список для mixed collections.

**SATB** (Snapshot At The Beginning) — механизм для concurrent mark. При начале mark JVM «делает снимок»: все объекты живые на момент старта считаются живыми до конца mark, даже если mutator их «удалил» (обнулил ссылку). Write barrier записывает старое значение поля перед перезаписью в SATB queue → mark их обработает. Это чуть менее точно (может остаться floating garbage до следующего цикла), но проще для реализации.

**Remembered Sets (RSets)** — G1 хранит для каждого региона информацию о ссылках **из других регионов на этот**. При young collection не надо сканировать весь Old — достаточно RSet young регионов. Ускоряет minor collections кардинально. Стоимость — write barrier обновляет RSets при cross-region ссылках, плюс место в heap на RSets (обычно 5-20% overhead).

Опция: `-XX:+UseG1GC` (дефолт с Java 9). Цель по паузе: `-XX:MaxGCPauseMillis=200` (дефолт 200 ms — soft target, не жёсткая гарантия).

**Реальные характеристики G1 в prod (heap 4-16 GB)**: paused 50-200 ms обычно, throughput ~95% (5% overhead на GC), footprint overhead ~5-10% (RSets). Full GC редко (только при исчерпании), длится секунды.

Проблемы G1:

- **Humongous allocations**. Объект > 50% размера региона занимает целый регион (или несколько смежных). Массивы > 8 MB при region_size = 16 MB — humongous. Аллоцируются в Old напрямую, могут вызвать преждевременный Full GC.
- **RSets overhead**. Много cross-region references → большие RSets → память + время mark. Особенно плохо при большом heap (100+ GB).
- **Latency spike** при mixed collections — иногда паузы вылетают за MaxGCPauseMillis.

## ZGC — sub-millisecond pauses

**ZGC** (Z Garbage Collector) — LOW-latency GC от Oracle, production-ready с Java 15. Целевая пауза **< 10 ms** (реально часто < 1 ms) независимо от размера heap. Работает на heap от 8 MB до 16 TB.

Ключевая инновация — **colored pointers**. В 64-битном указателе на объект часть битов используется для GC-метаданных (mark bits, remap bits, forwarding bits). При каждой загрузке ссылки (`load barrier`) JVM проверяет эти биты и делает нужное действие (переместить объект, обновить ссылку, отметить как живой).

Схема (упрощённо для x86 64-bit):

```
64-bit pointer:
[unused 16 bits][marked0][marked1][remapped][finalizable][... 42 bits address ...]
                          ↑ colored bits — meta для GC
```

**Load barrier** — код, вставляемый компилятором перед каждой загрузкой object reference. Проверяет colored bits, если объект должен быть перемещён — перемещает и обновляет ссылку. Всё это concurrent с приложением, паузы минимальны.

**Все фазы concurrent** кроме двух коротких STW:

- **Pause Mark Start** — короткий (< 1 ms), помечает roots.
- **Concurrent Mark** — параллельно с приложением, обходит граф.
- **Pause Mark End** — короткий, финализирует.
- **Concurrent Prepare for Relocation** — считает какие регионы освобождать.
- **Pause Relocate Start** — короткий, помечает начало relocation.
- **Concurrent Relocation** — перемещает живые объекты. Load barrier обновляет ссылки лениво (когда приложение обращается) — pointer updated on read.

ZGC работает с **регионами** (называются **pages** — small 2 MB, medium 32 MB, large N × 2 MB). Похоже на G1, но регионы могут содержать объекты разных generation (в базовом non-generational ZGC).

**Generational ZGC** (JEP 439, Java 21) — добавил young/old split, что даёт лучший throughput на большинстве workloads (было throughput penalty ~10-15% у non-generational).

Опция: `-XX:+UseZGC` (single-generation до Java 20). `-XX:+UseZGC -XX:+ZGenerational` (generational Java 21). С Java 23 generational — default для ZGC, non-generational deprecated.

**Реальные характеристики (heap 32-500 GB)**: паузы 10 μs - 2 ms (микросекунды!), throughput ~85-95% (было хуже до generational), footprint overhead **20-30%** (colored pointers + heap для relocation reserves).

Плюсы: паузы независят от heap size. Идеально для больших heap (100 GB+) и latency-critical (HFT, real-time analytics, low-latency APIs). Concurrent compaction — нет фрагментации.

Минусы: overhead на throughput vs Parallel GC (5-15% меньше polezной работы). Больший footprint. Требует 64-bit JVM (colored pointers). До Java 21 не был generational — терял throughput.

## Shenandoah — параллельная альтернатива

**Shenandoah** от Red Hat — второй низколатентный GC. Появился в OpenJDK 12 как experimental, production-ready в Java 15. Аналогичные цели что ZGC: sub-millisecond пауз на большом heap.

Ключевая идея — **Brooks forwarding pointers**. Каждый объект имеет дополнительное поле (forwarding pointer), которое указывает на «актуальную копию» объекта (сам на себя обычно, на новое место после перемещения). Load barrier читает forwarding pointer перед доступом к объекту.

Отличие от ZGC: не использует биты в указателе, работает на любой архитектуре (включая ARM, где ZGC до Java 15 не работал). Compaction concurrent, как у ZGC.

Фазы (упрощённо):

1. **Init Mark** — STW, короткая.
2. **Concurrent Marking** — параллельно.
3. **Final Mark** — STW.
4. **Concurrent Cleanup** — освобождает регионы без живых объектов сразу.
5. **Concurrent Evacuation** — перемещает живые объекты в новые регионы.
6. **Init Update Refs** — STW.
7. **Concurrent Update References** — обновляет ссылки на перемещённые объекты.
8. **Final Update Refs** — STW.

Больше STW пауз чем у ZGC (но каждая короткая), evacuation отделён от update refs (у ZGC они «размазаны» по load barriers).

Опция: `-XX:+UseShenandoahGC`. По умолчанию не включён.

**Реальные характеристики**: паузы 1-10 ms (чуть хуже ZGC на средней heap, сопоставимо на большой), throughput ~90-95%, footprint overhead ~10-20% (меньше чем ZGC).

Когда выбирать Shenandoah vs ZGC:

- **ARM64 архитектура** (например Graviton в AWS) — Shenandoah работал раньше, ZGC получил поддержку позже. Оба сейчас работают, разница минимальна.
- **Heap 4-32 GB** — Shenandoah часто чуть быстрее по throughput.
- **Heap 100+ GB, критично низкая latency** — ZGC.
- **RedHat/OpenJDK экосистема** — Shenandoah поставляется, поддерживается.

## Сравнение с числами

Приблизительные характеристики (реальные зависят от workload):

| GC | Пауза (heap 8GB) | Пауза (heap 100GB) | Throughput | Footprint overhead | Use case |
|----|------------------|---------------------|------------|---------------------|----------|
| Serial | 200-500 ms | N/A (не подходит) | 100% | ~2% | embed, CLI, < 1 GB |
| Parallel | 500 ms - 2 s | 5-20 s | 100% baseline | ~3% | batch, throughput-focused |
| G1 | 50-200 ms | 200-500 ms | ~95% | ~10% | default, 4-32 GB, balanced |
| ZGC | < 2 ms | < 5 ms | ~90% (generational) | ~25% | latency-critical, large heap |
| Shenandoah | 1-10 ms | 5-20 ms | ~92% | ~15% | latency-critical, ARM, mid-large |

Throughput = процент от Parallel GC (baseline). Footprint = дополнительная память сверх heap на GC bookkeeping.

## Как выбирать GC

Алгоритм принятия решения:

**Heap < 1 GB, single CPU (контейнеры, mini services)** → **Serial** (Java 21 auto-selects). Overhead минимальный.

**Batch processing (ETL, ML training), throughput критичнее latency** → **Parallel** GC. Максимум полезной работы за единицу времени.

**Обычный enterprise (web-сервисы, API, микросервисы), heap 4-32 GB** → **G1** (дефолт). Хороший баланс, паузы приемлемые для user-facing.

**Latency-critical (HFT, real-time systems, gaming, streaming), паузы > 50 ms недопустимы** → **ZGC** (generational, Java 21+) или **Shenandoah**.

**Very large heap (100 GB+)** → **ZGC**. Паузы не зависят от heap size.

**ARM64 (AWS Graviton, Apple Silicon)** — оба (ZGC и Shenandoah) сейчас работают, выбор по latency vs throughput.

## Тюнинг: какие параметры реально влияют

Тюнинг GC — искусство измерения, не гадания. Правило: **менять один параметр, измерять, повторять**. Не менять по 5 параметров сразу.

**Общие параметры (для всех GC)**:

- `-Xms<size>` — начальный размер heap. Ставить = `-Xmx` для избежания resize при старте (uneven latency).
- `-Xmx<size>` — максимальный. Для контейнеров с cgroup limit — 70-80% от memory.limit.
- `-XX:MaxMetaspaceSize=<size>` — предел Metaspace. Без ограничения может расти неконтролируемо при dynamic classloading.

**Для G1**:

- `-XX:MaxGCPauseMillis=<ms>` (default 200) — soft target для паузы. G1 подстраивает размеры young и mixed cycles. Слишком маленькое (< 50) → частые маленькие collections, throughput падает.
- `-XX:G1HeapRegionSize=<size>` — размер региона. Auto: heap/2048, округлённое до степени 2. Явное указание нужно если много humongous allocations (увеличить чтобы объекты стали non-humongous).
- `-XX:G1NewSizePercent`, `-XX:G1MaxNewSizePercent` — процент heap на young generation. Auto-tuned, обычно не трогать.
- `-XX:ConcGCThreads=<N>` — треды для concurrent mark. Дефолт = ncores/4. Больше = быстрее mark, меньше CPU для приложения.

**Для ZGC / Shenandoah**:

- `-XX:MaxGCPauseMillis` — учитывается как hint.
- `-XX:ConcGCThreads` — треды для concurrent фаз. Для ZGC critical — влияет на скорость relocation.
- `-XX:SoftMaxHeapSize=<size>` (ZGC) — soft target для использования heap. Позволяет иметь большой max, но обычно жить в меньшем.

**Общая рекомендация**: не тюньте GC пока не измерили что он реально проблема. `-Xms == -Xmx`, дефолтный GC (G1), нормальный `MaxMetaspaceSize` — покрывает 90% случаев. Дальнейший тюнинг — по метрикам, не по вере.

## Диагностика в проде: GC-логи

GC-логи — единственный надёжный источник понимания что GC реально делает.

**Включение GC logging** (Java 9+, unified logging):

```
-Xlog:gc*:file=/var/log/gc.log:time,uptime,level,tags:filecount=10,filesize=100M
```

Логирует всё про GC в /var/log/gc.log, ротация по 100 MB × 10 файлов.

Пример строки G1 young collection:

```
[2026-09-25T10:15:23.456+0500][123.456s][info][gc,start] GC(42) Pause Young (Normal) (G1 Evacuation Pause)
[2026-09-25T10:15:23.478+0500][123.478s][info][gc] GC(42) Pause Young (Normal) (G1 Evacuation Pause) 4096M->2048M(8192M) 22.345ms
```

Читаем: GC #42, тип young evacuation, heap до `4096M`, после `2048M`, всего `8192M`, пауза `22.3 ms`. Освобождено 2 GB — нормальная эффективная young collection.

Плохой пример (Full GC):

```
[info][gc] GC(85) Pause Full (G1 Compaction Pause) 7800M->5200M(8192M) 4200ms
```

Full GC на 4.2 секунды. Освободил только 2.6 GB из 7.8 — heap близко к исчерпанию. Красный флаг.

Что искать в GC-логах:

- **Частые Full GC** — приложение упирается в heap, промоушен в Old не успевает освобождаться. Увеличить heap или искать memory leak.
- **Растущий Old после каждого GC** — memory leak или недостаточный размер Young (объекты промоутятся преждевременно).
- **Паузы вылетают за target** — GC не успевает, увеличить heap или переключить GC.
- **`to-space exhausted`** — G1 не хватило места для evacuation. Часто предвестник Full GC.
- **`Humongous allocation`** — большие объекты (arrays, buffers) идут в Old напрямую.

**JFR (Java Flight Recorder)** — production-friendly замена GC-логам с гораздо большим объёмом информации:

```
jcmd <pid> JFR.start duration=60s filename=recording.jfr settings=profile
```

События: `jdk.GarbageCollection`, `jdk.G1GarbageCollection`, `jdk.YoungGarbageCollection`, `jdk.OldGarbageCollection`, `jdk.GCPhasePause`, `jdk.PromotionFailed`, `jdk.EvacuationFailed`. Открыть в JMC — визуализация GC pauses over time, allocation rate, promotion rate.

**jstat** — быстрый snapshot в CLI:

```bash
jstat -gc <pid> 1000 10
# Каждую секунду 10 раз: S0C S1C S0U S1U EC EU OC OU MC MU YGC YGCT FGC FGCT
```

`S0U/S1U` — Survivor used. `EU` — Eden used. `OU` — Old used. `MU` — Metaspace used. `YGC` / `FGC` — счётчики young / full collections. `YGCT` / `FGCT` — время в них.

Растущий `OU` при стабильной нагрузке — memory leak. Растущий `FGC` — приложение упирается.

## Типовые проблемы

**Humongous allocations**. `jdk.ObjectAllocationOutsideTLAB` event в JFR или в GC-логе `G1: Humongous allocation`. Обычно — большие массивы (byte[], char[]) из парсеров (XML, JSON), buffers для сетевых операций.

Обход: увеличить размер региона G1 (`-XX:G1HeapRegionSize=32M` вместо auto 8-16M — тогда объекты до 16 MB становятся non-humongous). Или переписать код — стримить парсер вместо загрузки целиком в память.

**Promotion failures / evacuation failures**. G1 не хватило Old регионов для промоушен. `to-space exhausted` в логе. Обычно предвестник Full GC.

Обход: увеличить heap. Уменьшить `-XX:G1NewSizePercent` (меньше young → меньше промоушенов). Проверить есть ли реально утечка.

**Allocation stalls (ZGC)**. Приложение аллоцирует быстрее чем ZGC успевает освобождать → app threads блокируются на аллокации. Метрика JFR: `jdk.ZAllocationStall`.

Обход: увеличить heap. Увеличить `ConcGCThreads`. Если постоянно — GC не подходит под workload (может быть G1 лучше).

**OOM: Java heap space**. Heap исчерпан. Не потому что «мало памяти» — потому что живые объекты занимают всё. Ищи leak через heap dump (см. 78).

**OOM: Metaspace**. Слишком много загруженных классов. Обычно — hot deploy in-place без правильного unload'а (Tomcat redeployment leaks), либо динамическая генерация классов (ByteBuddy, CGLIB на каждый вызов вместо кэша).

Обход: `-XX:MaxMetaspaceSize`, найти утечку classloader'ов через heap dump с `jhat`/`Eclipse MAT`.

**String deduplication**. При много дубликатных строк (например, десериализация одинаковых JSON) — включить `-XX:+UseStringDeduplication` (для G1 и ZGC). GC находит одинаковые String'и в Old, заменяет их char[] на общий. Экономия десятки процентов heap при определённых workloads.

## Практические рецепты

**Web-сервис 4-8 GB heap, обычный enterprise API**:

```
-Xms8g -Xmx8g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100
-XX:+UseStringDeduplication
-Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=10,filesize=100M
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heap-dump.hprof
```

**Latency-critical service, 32 GB heap, целевая пауза < 5 ms**:

```
-Xms32g -Xmx32g
-XX:+UseZGC
-XX:+ZGenerational
-XX:ConcGCThreads=4
-Xlog:gc*:file=/var/log/gc.log
```

**Batch job, throughput критичен**:

```
-Xms16g -Xmx16g
-XX:+UseParallelGC
-XX:ParallelGCThreads=8
```

## Быстрая диагностика при инциденте «сервис тормозит, паузы»

Первое что делать:

**1. Проверить GC-log** — есть ли Full GC. Если да и частые — увеличить heap или искать leak.

**2. `jstat -gc <pid> 1000 10`** — увидеть текущее состояние. Растёт ли Old? Метаspace?

**3. Взять heap dump**: `jcmd <pid> GC.heap_dump /tmp/dump.hprof`. Открыть в Eclipse MAT — Dominator Tree покажет что удерживает память.

**4. Собрать JFR**: `jcmd <pid> JFR.start duration=60s filename=diag.jfr settings=profile`. Открыть в JMC — Garbage Collection tab покажет паузы, allocation rate, promotion rate.

**5. Проверить cgroup memory**: `cat /sys/fs/cgroup/memory/memory.stat | grep -E "^(cache|rss|swap)"`. Иногда «GC не работает» — это на самом деле cgroup killer убил процесс за превышение памяти (`dmesg` покажет OOM killer).

## Заключение

GC — не магия, а конкретные алгоритмы с чётко определёнными trade-offs. Reachability analysis от GC roots определяет живые объекты. Generational hypothesis (большинство объектов умирают молодыми) даёт young/old split для оптимизации. Основные фазы — mark / sweep / compact — реализованы по-разному в разных GC.

**Serial** — single-threaded, для < 1 GB heap, embed/CLI. Java 21 auto-select для маленьких контейнеров.

**Parallel** — multi-threaded STW, максимальный throughput, для batch. Был default до Java 9.

**G1** — default с Java 9. Region-based, generational, mixed collections, SATB для concurrent mark, RSets для cross-region references. Паузы 50-200 ms, throughput ~95%, footprint ~10%. Балансированный выбор для 4-32 GB.

**ZGC** — sub-millisecond pauses через colored pointers и load barriers. Всё concurrent кроме коротких STW. Generational с Java 21 (default с Java 23). Для latency-critical и очень больших heap (100+ GB). Overhead throughput 5-15%, footprint 20-30%.

**Shenandoah** — альтернатива ZGC от Red Hat через Brooks forwarding pointers. Работает без специальных архитектурных требований. Похожие характеристики что ZGC, чуть меньше footprint, чуть больше пауз.

**Выбор GC**: heap < 1 GB → Serial. Batch throughput → Parallel. Общий enterprise 4-32 GB → G1 (default). Latency-critical или heap > 100 GB → ZGC. ARM или RedHat экосистема → Shenandoah равно ZGC.

**Тюнинг**: `-Xms == -Xmx` для избежания resize. `-XX:MaxGCPauseMillis` — soft target для G1/ZGC. `-XX:MaxMetaspaceSize` для защиты от неограниченного роста classloaders. Не менять по 5 параметров сразу — измерять эффект каждого изменения.

**Диагностика в проде**: GC-log (`-Xlog:gc*`) — единственный надёжный источник. JFR через `jcmd JFR.start` для systematic анализа. `jstat -gc` для быстрого snapshot. Full GC частые = leak или недостаточный heap. Растущий Old = leak. Паузы за target = GC не справляется, переключить.

**Типовые проблемы**: humongous allocations (большие массивы, обход через `G1HeapRegionSize`); promotion failures (heap мал); allocation stalls (ZGC не успевает); OOM heap (leak или недостаточный heap); OOM metaspace (classloader leak при hot deploy).

**Практический рецепт для web-сервиса**: `-Xms=-Xmx=8g -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:+UseStringDeduplication -Xlog:gc*:file=... -XX:+HeapDumpOnOutOfMemoryError`. Работает 90% случаев enterprise API.

JVM/OS/RAM/native — в 84. Threads и VT — в 83/110. Prod-диагностика heap — в 78 (jcmd, MAT). Здесь была глубина по GC: алгоритмы каждого сборщика, реальные характеристики, выбор под workload, тюнинг с пониманием, диагностика в проде.
