# 83. Threads в Java глубоко: Platform vs Virtual — от OS до Loom

## Зачем это знать

Concurrency в Java — тема, где легко работать успешно, не понимая происходящего. `ExecutorService`, `CompletableFuture`, `@Async`, `parallelStream()` — всё работает, threads создаются, задачи параллелятся. Однако именно понимание внутренностей — что такое OS thread, как JVM его использует, чем именно virtual thread отличается от platform — определяет, будет ли ваш сервис справляться с нагрузкой в 10000 concurrent запросов или задыхаться уже на 500.

Есть три причины, зачем это знать глубоко. Первая — производительность. Если вы работаете с thread pool'ами вслепую, вы либо создаёте слишком мало threads (низкий throughput), либо слишком много (OOM, cache thrashing). Понимая, что OS thread — это ~1 MB стек плюс kernel task_struct, и что context switch между ними — это микросекунды, вы правильно оцените размеры пулов и разницу между CPU-bound и I/O-bound задачами.

Вторая — переход на virtual threads. Java 21 добавил Project Loom. Многие команды сейчас мигрируют, и делают это ошибочно. Меняют `newFixedThreadPool` на `newVirtualThreadPerTaskExecutor` и удивляются, что производительность не выросла — потому что не удалили `synchronized` блоки с блокирующим I/O внутри, которые пинят carrier threads. Или не поняли, что connection pool из 20 connections остался bottleneck, независимо от того, сколько virtual threads наверху. Понимая Loom изнутри — continuation, mount/unmount, carrier pool — вы избежите этих граблей.

Третья — memory model. `synchronized`, `volatile`, `AtomicInteger`, `ConcurrentHashMap` — все они опираются на Java Memory Model. Без понимания happens-before вы будете писать код, который «работает у меня локально», но фейлится под нагрузкой в проде. Классический double-checked locking, race conditions в singleton'ах, невидимые изменения между потоками — всё это из-за незнания JMM.

Мы разберём четыре уровня: OS thread (что делает kernel), Java platform thread (обёртка над OS thread), Java virtual thread (continuation on carrier), и process (контейнер для threads). Дальше — thread pools (ThreadPoolExecutor, ForkJoinPool, work-stealing), synchronization primitives (synchronized, volatile, ReentrantLock, Semaphore, atomic classes, concurrent collections), Java Memory Model с happens-before. Отдельно — глубокое погружение в virtual threads: continuation, stack chunks, mount/unmount, pinning, что происходит при blocking I/O внутри. Детальное сравнение platform vs virtual, где что использовать, типичные ошибки миграции. Real-world примеры (fan-out через StructuredTaskScope, Kafka batch processing). Debug и observability: jstack, JFR, async-profiler, метрики.

## Четыре уровня потоков

Слово «поток» перегружено. Разделяй чётко.

Первый уровень — **Process**. Это единица изоляции в OS. Свой virtual address space, свой набор file descriptors, свой user/group ID, свой working directory. Java-приложение = один процесс. Из одного бинаря `java` можно запустить сотни независимых процессов, каждый со своей памятью и состоянием.

Второй уровень — **OS thread (kernel thread)**. Единица планирования внутри процесса. В Linux — `task_struct` в ядре. Kernel scheduler переключает threads между CPU. Все threads одного процесса шарят memory (heap, globals, file descriptors), но каждый имеет свой stack, свой register set, свой program counter.

Третий уровень — **Java platform thread** (`java.lang.Thread`). Java-обёртка над OS thread. **1 Java Thread = 1 OS thread = 1 kernel task_struct** (1:1 mapping). Каждый `new Thread(...).start()` через JNI создаёт новый OS thread. Именно эта модель существовала в JVM с самого начала — до Java 21.

Четвёртый уровень — **Java virtual thread** (Java 21+, Project Loom). Совсем другая штука. Виртуальный поток — легковесная Java-обёртка над **continuation** (snapshot стека), которая **не привязана 1:1** к OS thread. JVM использует пул **carrier threads** (обычно `ForkJoinPool` с `availableProcessors()` потоков). Один carrier может обслуживать миллионы virtual threads — mount VT, выполнять его код до блокирующей операции, unmount, mount другой VT, и так далее. Это M:N mapping: миллион VT на десятки carriers.

Ключевое: **до Java 21 в JVM было только одно понимание thread'а — platform thread**. Virtual threads добавили новый слой поверх, но platform thread никуда не делся. Он остаётся carrier, на котором крутятся virtual threads. И для CPU-bound задач, для legacy кода с `synchronized`, для GC/JIT/scheduler threads — platform thread всё ещё используется.

## Что такое поток на уровне OS

Процесс vs поток на уровне ядра. **Процесс** — единица изоляции. **Thread** — единица планирования внутри процесса.

Все threads процесса шарят memory. Один поток может увидеть изменения, сделанные другим потоком, без каких-либо явных механизмов передачи данных (это же одна память). Но именно из-за этого нужны synchronization primitives — иначе race conditions.

В Linux разделение между thread и process условное. И то, и другое — `task_struct` в ядре. Threads создаются через syscall `clone()` с флагом `CLONE_VM` — шарят address space. Процессы (через `fork()`) — без `CLONE_VM`, каждый со своей памятью через copy-on-write.

`clone()` — универсальный syscall в Linux, лежащий в основе всего создания процессов и потоков:

```c
long clone(unsigned long flags, void *child_stack, ...);
```

Флаги определяют, что шарить:

- `CLONE_VM` — memory (обязательно для thread).
- `CLONE_FS` — filesystem info (working directory, umask).
- `CLONE_FILES` — open file descriptors.
- `CLONE_SIGHAND` — signal handlers.
- `CLONE_THREAD` — та же thread group (одинаковый PID для `getpid()`, разный TID).

`pthread_create` в glibc — обёртка над `clone` с типовым набором флагов для POSIX thread. `fork()` — обёртка над `clone` без CLONE_* флагов — полное разделение через copy-on-write.

Каждый OS thread имеет свой **stack**. Область памяти для local variables, return addresses, function frames. Размер стека в Linux по умолчанию — **8 MB** (проверить `ulimit -s`). Но физически не резервируется сразу: virtual mapping + on-demand page allocation. Реально используется столько, сколько глубина рекурсии + local vars.

**Guard page** — специальная защитная страница снизу стека. Если thread рекурсирует так глубоко, что попадает в guard page — SIGSEGV, kernel убивает процесс (или JVM ловит и конвертирует в `StackOverflowError`).

JVM переопределяет размер стека через флаг `-Xss` (по умолчанию **1 MB на 64-bit**). Меньше `-Xss` = больше threads помещается в память. Больше = меньше `StackOverflowError` при глубокой рекурсии.

**Context switch** — kernel переключается между threads (несколько раз в мс). Что происходит:

1. **Save current state** — сохранить registers, PC, стек pointer в TCB (Thread Control Block).
2. **Update scheduler bookkeeping** — statistics, priority queues.
3. **Restore new thread state** — загрузить registers из TCB нового.
4. **TLB flush** (если разные address spaces) — expensive для процессов, для threads одного процесса не нужно.
5. **Cache invalidation** — L1/L2 cache холодный после switch, warm-up требуется.

Стоимость context switch: **~1-10 микросекунд** на воспроизводимом hardware. Плюс cache warm-up — реально ~50-100 микросекунд для полного восстановления производительности.

Voluntary switch (thread сам блокируется на I/O) — часто. Involuntary switch (time slice expired, preempted) — тоже часто.

**Scheduling** в Linux — Completely Fair Scheduler (CFS). Каждому thread'у дают время proportionally to priority (nice value). Real-time scheduling (`SCHED_FIFO`, `SCHED_RR`) — для критичных задач, вне обычной handedness. `Thread.setPriority(1-10)` в Java маппится на nice value, но эффект минимальный (Linux не даёт Java менять priority ниже обычного nice range без CAP_SYS_NICE).

Максимум threads в системе:

- `/proc/sys/kernel/threads-max` — глобальный лимит (~4 миллиона обычно).
- `/proc/sys/vm/max_map_count` — memory mappings (каждый стек = mapping).
- `ulimit -u` — процессов/threads per user (~1000-30000 default).
- `RLIMIT_STACK × N` — общая память под все стеки.

Практически: **10000-50000 platform threads в JVM** — уже проблема. RAM забита стеками (5000 × 1MB = 5GB только под стеки, не считая heap).

## Java Platform Thread — что это

`java.lang.Thread` — Java-класс, представляющий один OS thread. Каждый `new Thread(...)` при `start()` вызывает JNI-метод:

```java
public synchronized void start() {
    // ... state checks
    start0();  // native method
}
```

`start0()` — native через JNI. Вызывает `pthread_create` в JVM (`os_linux.cpp` в HotSpot). **1 Java Thread ↔ 1 OS thread (pthread) ↔ 1 kernel task_struct**. Один-к-одному.

Thread states в Java:

- **NEW** — Thread создан, но `start()` не вызван.
- **RUNNABLE** — может исполняться (running или ready to run — Java не различает).
- **BLOCKED** — ждёт входа в `synchronized` (monitor lock).
- **WAITING** — Object.wait(), LockSupport.park(), Thread.join() без timeout.
- **TIMED_WAITING** — то же с timeout. Thread.sleep(), wait(ms).
- **TERMINATED** — run() завершилась.

Смотреть: `Thread.getState()` из Java кода или `jstack <pid>` из shell.

Java thread stack — обычный OS thread stack. Размер `-Xss`. Хранит:

- **Frame'ы** — по одному на каждый вызов метода.
- В frame: local variables (primitive + reference), operand stack, return PC.
- Reference'ы указывают в heap — там реальные объекты.

Frame layout:

```
+------------------+
| Return PC        |
| Frame Pointer    |
| Local Variables  |  (this, args, локальные)
| Operand Stack    |  (JVM stack machine, для арифметики)
+------------------+
```

Каждый Java-метод компилируется в bytecode, работающий с operand stack этого frame'а. `iadd`, `iload`, `istore` — операции на operand stack.

Стоимость создания platform thread на типичном Linux 64-bit:

- **Time**: ~50-500 микросекунд (syscall + инициализация).
- **Memory**: `-Xss` (default 1MB) — зарезервировано в virtual space. Физически — по мере использования.
- **File descriptors**: 2-3 на thread (eventfd, mutexes).

Не большая цена per thread, но накапливается. Тысячи threads = гигабайты virtual space + файловые дескрипторы.

**Daemon threads**. `thread.setDaemon(true)` — daemon threads не мешают JVM завершиться. Когда все не-daemon threads закончились, JVM exit'ит, даже если daemons ещё работают.

Использование: background tasks (GC threads, JIT compiler threads, scheduled executors), которые не должны продлевать жизнь приложения. Правило: пул executor'а обычно должен быть daemon — иначе приложение зависает при shutdown, ожидая worker'ов.

## Thread pools — как правильно управлять platform threads

Создание thread на каждую задачу дорого:

- ~50-500 микросекунд создания.
- Ресурсоёмко (стек, FDs, kernel bookkeeping).
- Burst создания тысяч threads → OOM или system freeze.

Thread pool — переиспользование threads: task в очередь → free worker берёт task → выполняет → возвращается в pool.

**ThreadPoolExecutor** — базовая реализация:

```java
new ThreadPoolExecutor(
    corePoolSize,           // сколько всегда живых worker'ов
    maximumPoolSize,        // абсолютный максимум
    keepAliveTime, TimeUnit.SECONDS,   // idle time до убийства выше core
    workQueue,              // BlockingQueue<Runnable>
    threadFactory,
    rejectionHandler
)
```

Логика submit:

1. Если running workers < `corePoolSize` → создать новый worker, дать task.
2. Иначе — попытаться положить в queue.
3. Если queue **полна** и workers < `maximumPoolSize` → создать worker.
4. Если workers == `maximumPoolSize` → rejection handler.

Queue types:

- **SynchronousQueue** — нет buffer'а, каждый offer ждёт poll. Идея: «либо worker сразу берёт, либо создаём новый». Использует `Executors.newCachedThreadPool()`. Даёт unlimited threads (опасно).
- **LinkedBlockingQueue** (unbounded) — очередь без лимита. Workers остаются на `corePoolSize`, всё копится в queue. Использует `Executors.newFixedThreadPool()`. Опасность: очередь может расти до OOM.
- **ArrayBlockingQueue** (bounded) — фиксированный размер. Разумный выбор для prod.

Rejection policies (что делать, когда queue полна и workers на maximum):

- **AbortPolicy** (default) — `RejectedExecutionException`.
- **CallerRunsPolicy** — task выполняется в потоке, который submit'ил (естественный backpressure).
- **DiscardPolicy** — тихо игнорировать.
- **DiscardOldestPolicy** — выбросить самый старый task из queue, положить новый.

Для prod часто **CallerRunsPolicy** — если очередь переполнена, вызывающий поток сам выполняет задачу, тормозя приём новых запросов. Естественный backpressure.

Анти-паттерны Executors factory methods:

- `Executors.newFixedThreadPool(N)` — использует `LinkedBlockingQueue` без лимита. Если producer'ы быстрее consumer'ов, очередь растёт до OOM. Не использовать в prod.
- `Executors.newCachedThreadPool()` — `SynchronousQueue` + unlimited threads. Burst создаст тысячи threads → OOM или system freeze.

Правило: используй `new ThreadPoolExecutor(...)` напрямую с bounded queue и явной rejection policy.

**ForkJoinPool — work-stealing** — специализированный пул для divide-and-conquer. Каждый worker имеет свою deque (double-ended queue). Task разбивается на subtask'и через `fork()`.

Work stealing: если worker закончил свои tasks, «крадёт» из хвоста deque другого worker'а. Балансирует нагрузку без centralized queue → лучше scaling.

Использования:

- `parallelStream()` — использует `ForkJoinPool.commonPool()`.
- `CompletableFuture.async()` без executor'а — commonPool.
- **Carrier threads для virtual threads** (default).

Default parallelism = `Runtime.getRuntime().availableProcessors()`. Опасность: если task в parallelStream блокируется, commonPool забит, вся система тормозит. Не мешай CPU-bound и I/O-bound в одном commonPool.

## Synchronization primitives — как threads координируются

Стандартные механизмы, критичные для понимания где virtual threads поведут себя по-другому.

**`synchronized`** — ключевое слово Java. Каждый объект имеет monitor (mutex). `synchronized(obj)` захватывает monitor, другие threads блокируются на входе.

```java
private final Object lock = new Object();

public void method() {
    synchronized (lock) {
        // critical section
    }
}
```

В bytecode: `monitorenter` и `monitorexit`. В JVM — оптимизации:

- **Biased locking** — если один поток захватывает lock часто (устарело в JDK 15+, удалено в 18).
- **Thin lock** — CAS на mark word объекта (без OS mutex).
- **Fat lock** (inflated) — реальный OS mutex, если contention высокий.

Проблемы `synchronized`:

- Не interruptible: `thread.interrupt()` не выведет из `synchronized`.
- Не fair: любой ждущий может получить lock (голодание возможно).
- Нет timeout.
- **Pinning virtual threads** — критичная проблема Loom.

**`volatile`** — модификатор поля. Гарантирует visibility между потоками:

- Запись в volatile → memory barrier (store-release).
- Чтение volatile → memory barrier (load-acquire).
- Формирует happens-before отношение.

```java
private volatile boolean running = true;

// Thread 1:
running = false;

// Thread 2:
while (running) {
    // reads latest value
}
```

Без volatile — Thread 2 может **никогда** не увидеть изменение (JIT inline'ит переменную в регистр).

`volatile` **не даёт атомарности** для `count++` (это read-modify-write). Для этого — `AtomicInteger` или `synchronized`.

**`java.util.concurrent.locks`** — более гибкие альтернативы `synchronized`.

**ReentrantLock**:

```java
Lock lock = new ReentrantLock();
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}
```

Плюсы над `synchronized`:

- `tryLock(timeout)` — с таймаутом.
- `lockInterruptibly()` — interruptible.
- `newCondition()` — множественные условия ожидания (вместо одного monitor).
- Fair mode (`new ReentrantLock(true)`) — гарантирует FIFO порядок.
- **Не пинит virtual threads** — ключевое для Loom.

**ReadWriteLock** — читателей много (параллельно), писатель один (эксклюзивно):

```java
ReadWriteLock rwl = new ReentrantReadWriteLock();
rwl.readLock().lock();   // многие могут держать
rwl.writeLock().lock();  // только один
```

Для read-heavy workload — большой выигрыш.

**StampedLock** (Java 8+) — три режима: read, write, optimistic read. Быстрее ReadWriteLock, но сложнее использовать (optimistic read требует manual validation).

**Semaphore** — ограничивает concurrent access. Внутри — счётчик permits:

```java
Semaphore sem = new Semaphore(10);   // максимум 10 concurrent

sem.acquire();
try {
    // work — не более 10 threads одновременно
} finally {
    sem.release();
}
```

Использования:

- Ограничить concurrency (пул connections вручную).
- Rate limiting.
- Управление ресурсами.
- **Backpressure для virtual threads** — важно, когда VT много, но downstream ограничен.

**CountDownLatch** — «жди пока N threads не сделают countDown». Одноразовый.

```java
CountDownLatch latch = new CountDownLatch(3);

// Workers:
executor.submit(() -> { doWork(); latch.countDown(); });

// Main:
latch.await();  // ждёт пока 3 воркера не вызовут countDown
```

**CyclicBarrier** — «все N threads встретятся здесь». Многоразовый.

```java
CyclicBarrier barrier = new CyclicBarrier(4);
barrier.await();  // ждёт пока 4 threads не подтянутся
```

**Phaser** — как CyclicBarrier, но с динамическим количеством threads. Сложнее, реже используется.

**Atomic classes (CAS-based)** — lock-free через **Compare-And-Swap**.

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();  // атомарно
```

Внутри — retry loop:

```java
public int incrementAndGet() {
    int current;
    do {
        current = get();
    } while (!compareAndSet(current, current + 1));
    return current + 1;
}
```

Быстрее `synchronized` при низком contention. При высоком — cache-line bouncing (много CAS retry).

**LongAdder** — «distributed» AtomicLong для high-contention случаев. Массив ячеек, каждый thread пишет в свою (по hash TID). `sum()` — сумма всех. Быстрее AtomicLong при contention, чуть медленнее при чтении.

**Concurrent Collections**:

- **ConcurrentHashMap** — thread-safe HashMap. С Java 8 внутри: bin по индексу — обычный HashMap bucket. Empty bin — CAS для insert. Non-empty — synchronized блок на первом node bin'а. Read — lock-free (через volatile happens-before).
- **CopyOnWriteArrayList** — при write создаёт новую копию массива. Читатели видят старую. Дорого для write, идеально для read-heavy + rare write (списки listeners, config).
- **BlockingQueue implementations**: `ArrayBlockingQueue` (fixed size, one lock), `LinkedBlockingQueue` (optionally bounded, two locks), `SynchronousQueue` (0 capacity, handoff), `PriorityBlockingQueue` (heap-based), `DelayQueue` (delayed consumption).

## Java Memory Model — что нужно знать, чтобы использовать threads

Без специальных гарантий compiler или CPU может **переупорядочить** операции:

```java
// Thread 1
data = 42;
ready = true;

// Thread 2
if (ready) {
    System.out.println(data);  // может быть 0!
}
```

Причины reorder:

- Compiler optimizations (JIT).
- CPU out-of-order execution.
- Store buffers (write не сразу в memory).
- Cache coherence delays.

Тонкость: без явных barriers Thread 2 может увидеть `ready == true` до того, как `data` стало 42. Это не bug JIT — это фундаментальное свойство современных CPU и optimizations. Java Memory Model определяет, при каких условиях reorder не допустим.

**Happens-before** — центральное правило JMM. Partial order операций: если A happens-before B, то эффекты A видны в B (memory operations правильно упорядочены).

Правила:

- **Program order** — операции внутри одного thread'а в порядке (написанном).
- **Monitor lock** — unlock happens-before последующего lock того же monitor'а.
- **Volatile** — write happens-before последующего read того же volatile.
- **Thread start** — `start()` happens-before actions в новом thread'е.
- **Thread join** — actions в thread happens-before `join()` return.
- **Final fields** — init в конструкторе happens-before любого read final field'а (если объект не escape'ится из конструктора).

Volatile практически:

```java
private volatile int state;

// Thread 1
data = compute();     // (1)
state = READY;        // (2) volatile write

// Thread 2
if (state == READY) { // (3) volatile read
    use(data);        // (4) — гарантированно видит result of (1)
}
```

Volatile write в (2) happens-before volatile read в (3) → всё, что было до (2), видимо в (4).

**Double-checked locking singleton** — классическая ошибка:

```java
class Lazy {
    private static Lazy INSTANCE;
    public static Lazy get() {
        if (INSTANCE == null) {                // (1)
            synchronized (Lazy.class) {
                if (INSTANCE == null) {         // (2)
                    INSTANCE = new Lazy();      // (3)
                }
            }
        }
        return INSTANCE;
    }
}
```

Проблема: (3) — это несколько операций (allocate → constructor → assign). Другой поток может увидеть частично сконструированный объект в (1). Fix: `private static volatile Lazy INSTANCE`. Volatile гарантирует visibility и предотвращает reorder.

Ещё проще — через class holder:

```java
class Lazy {
    private static class Holder {
        static final Lazy INSTANCE = new Lazy();
    }
    public static Lazy get() {
        return Holder.INSTANCE;
    }
}
```

Class loading гарантирует happens-before, thread-safe без volatile.

## Virtual Threads — глубокие internals

Теперь к Loom. Что такое virtual thread изнутри, как работает mount/unmount, где pinning, что происходит при I/O.

**Continuation** — фундамент Loom. Snapshot текущего состояния выполнения (stack + PC + registers), который можно приостановить и возобновить.

JVM API (internal): `jdk.internal.vm.Continuation`. Публичный API — через virtual threads.

Псевдокод:

```java
Continuation c = new Continuation(scope, () -> {
    print("A");
    Continuation.yield(scope);  // suspend
    print("B");
});

c.run();   // prints "A", suspends
c.run();   // prints "B", finished
```

Virtual thread — это Continuation, mounted на carrier thread. При block'е на I/O — yield → carrier свободен для другого VT.

**Stack chunks** — как хранится стек VT. Обычный OS thread стек = contiguous memory allocation, ~1 MB. Слишком дорого для миллионов threads.

VT стек хранится в **heap** как **stack chunk** — маленькие blocks (~256 B - 32 KB, дырявый allocation). При mount:

1. JVM копирует stack chunks из heap на реальный OS stack carrier'а.
2. VT исполняется, стек растёт на real stack carrier'а.
3. При unmount: копируется обратно в heap (только использованная часть).

Итог: VT потребляет только фактически используемый стек в heap. Тысячи мелких VT ~5-20 KB каждый. Миллион VT — 5-20 GB (сильно зависит от глубины стека).

**Mount / Unmount lifecycle**. Mount: VT прикрепляется к carrier thread. Carrier runs VT code as if it were own. Unmount: VT останавливается, carrier свободен для другого VT.

Unmount происходит при:

- **I/O blocking calls**: `Socket.read`, `Files.read`, JDBC.
- `LockSupport.park` (базовый механизм — используется в ReentrantLock, Semaphore, etc.).
- `Thread.sleep`.
- `Thread.yield`.

Ключевое: **JVM переписывает JDK blocking API**, чтобы они yield'или continuation вместо блокировки OS thread'а.

Пример: `Socket.getInputStream().read()` в Java 21 — internally проверяет `is VT?` → если да → `LockSupport.park` + async I/O через epoll/kqueue + registration в selector.

**Carrier thread pool**. Default: `ForkJoinPool` с parallelism = `availableProcessors()`.

Configuration:

```
-Djdk.virtualThreadScheduler.parallelism=16
-Djdk.virtualThreadScheduler.maxPoolSize=256
-Djdk.virtualThreadScheduler.minRunnable=1
```

Carriers — daemon platform threads с именем `ForkJoinPool-1-worker-N`. Смотреть carriers из VT:

```java
Thread.currentThread();
// "VirtualThread[#21]/runnable@ForkJoinPool-1-worker-3"
```

**Pinning** — VT не может unmount'иться, carrier заблокирован.

Причины pinning:

- **`synchronized` block с blocking call внутри** — JVM не yield'ит стек синхронизированной области.
- **Native method** (JNI) — JVM теряет контроль.
- **File I/O до Java 21** (fixed в 21 через io_uring, но не все pathways).

Как проверить:

```
-Djdk.tracePinnedThreads=full     # каждое pinning event с полным stack trace
-Djdk.tracePinnedThreads=short    # только summary
```

Fix pinning — заменить `synchronized` на `ReentrantLock`:

```java
// Плохо
synchronized (lock) {
    Thread.sleep(100);   // pins carrier!
}

// Хорошо
lock.lock();
try {
    Thread.sleep(100);   // fine, VT unmounts
} finally {
    lock.unlock();
}
```

**Что происходит при I/O** внутри VT:

```
// Virtual Thread runs:
InputStream in = socket.getInputStream();
int b = in.read();

// Что происходит:
// 1. JDK Socket implementation проверяет: is VT?
// 2. Если да:
//    a. socket.setNonBlocking()
//    b. register FD in NIO poller (epoll on Linux)
//    c. LockSupport.park() — yield continuation
//    d. carrier свободен
// 3. NIO poller (single dedicated thread) sees FD readable
// 4. unpark(virtualThread) — VT ставится в scheduler queue
// 5. Free carrier picks up, resumes VT
// 6. VT reads bytes (non-blocking now returns immediately)
```

Ключевая идея: **blocking API на surface, non-blocking underneath**. Программист пишет привычный императивный код, JVM превращает в async.

**ThreadLocal и VT — проблема**. ThreadLocal хранит одно значение на thread. С миллионом VT — миллион копий:

```java
ThreadLocal<byte[]> BUFFER = ThreadLocal.withInitial(() -> new byte[8192]);
// 1M VT × 8KB = 8GB памяти
```

Решения:

1. **ScopedValue** (Java 21 preview, стабилизируется):

```java
static final ScopedValue<String> USER = ScopedValue.newInstance();

ScopedValue.where(USER, "berik").run(() -> {
    // USER.get() == "berik"
    nestedCall();
});
```

Immutable, ограничена scope. Не наследуется автоматически child threads.

2. **Пул объектов** вместо ThreadLocal кэша.
3. **Просто аллокация** — small buffer'ы через TLAB (Thread-Local Allocation Buffer) фактически бесплатны для GC.

## Platform vs Virtual — детальное сравнение

| Аспект | Platform Thread | Virtual Thread |
|--------|-----------------|----------------|
| **Underlying** | 1:1 к OS thread | Continuation on carrier |
| **Creation cost** | 50-500 μs, syscall | ~1 μs, heap allocation |
| **Memory** | `-Xss` (default 1 MB) reserved | ~5-20 KB used in heap |
| **Maximum count** | ~5-10K на серверe (RAM limit) | Миллионы |
| **Scheduling** | OS kernel (CFS) | JVM scheduler (ForkJoinPool) |
| **Context switch** | ~1-10 μs (kernel) | ~200-500 ns (JVM) |
| **Priority** | OS priority (nice) | Игнорируется |
| **ThreadLocal cost** | Дешёво (тысячи копий) | Может быть проблемой (миллионы) |
| **Debugging** | jstack, jconsole | Могут быть huge dumps |
| **When to use** | CPU-bound, small pool | I/O-bound, many concurrent |
| **Pinning issues** | N/A | synchronized + I/O, native code |
| **Blocking cost** | Freezes OS thread | Unmounts, carrier free |
| **Thread pool** | Обязателен для нормальной работы | Не нужен (per-task VT дёшев) |

Правильный use case для каждого:

**Platform thread**:

- CPU-bound background computation (JIT, GC — как раз они).
- Небольшое количество долгоживущих worker'ов (Netty event loop, Kafka consumer).
- Legacy code с heavy `synchronized`.
- Interactive user thread (Swing/AWT — они на своих platform threads).

**Virtual thread**:

- HTTP request handling (Tomcat с VT).
- Fan-out к нескольким external services.
- Concurrent processing большого списка (parallel iteration с I/O per item).
- WebSocket / SSE / long polling.
- Batch import из БД (thousands of concurrent DB queries — но помни о connection pool).

Ошибки миграции на VT:

**Использовать VT для CPU-bound**. Не быстрее, часто медленнее (overhead scheduler'а).

**`Executors.newFixedThreadPool(N)` заменить на `newVirtualThreadPerTaskExecutor()` без думалок**. Если у тебя всего 10 tasks, старый пул может быть лучше (нет overhead создания VT).

**Не удалить `synchronized` блоки** — если внутри I/O, carriers пинятся, «virtual threads не помогают».

**Забыть про backpressure** — VT дешёвые, но downstream (БД, external API) не масштабируется вместе. Semaphore для rate limiting.

**ThreadLocal caches** — 1M копий по 10KB = 10GB. Rethink.

## Real-world примеры

**Fan-out через StructuredTaskScope**:

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user = scope.fork(() -> userClient.get(userId));
    var orders = scope.fork(() -> orderClient.getAll(userId));
    var address = scope.fork(() -> addressClient.get(userId));

    scope.join().throwIfFailed();

    return new UserFullInfo(user.get(), orders.get(), address.get());
}
```

3 HTTP-запроса параллельно, каждый на своём VT. Latency = max(3), не sum.

Без VT — либо `CompletableFuture.thenCombine(...)` (усложняет код), либо блокирующие вызовы sequentially (медленно).

StructuredTaskScope гарантирует:

- Если один task упадёт с exception → остальные automatically cancelled.
- Scope не exits пока все subtasks закончились (`join()`).
- Читабельный код в стиле try-with-resources.

**Обработка Kafka batch**:

```java
List<Record> records = consumer.poll(...);   // 500 records

try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    List<Subtask<Result>> results = records.stream()
        .map(r -> scope.fork(() -> processRecord(r)))
        .toList();
    
    scope.join().throwIfFailed();
    
    List<Result> ok = results.stream().map(Subtask::get).toList();
    consumer.commitSync();
}
```

500 VT параллельно обрабатывают records. Если каждый делает 3 DB call'а — тысячи concurrent connections **необходимо ограничить** semaphore'ом или через DB pool size.

**Ограничение concurrency через Semaphore**:

```java
Semaphore dbSemaphore = new Semaphore(100);  // максимум 100 concurrent DB call

try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    for (int i = 0; i < 10_000; i++) {
        final int id = i;
        scope.fork(() -> {
            dbSemaphore.acquire();
            try {
                return db.query(id);
            } finally {
                dbSemaphore.release();
            }
        });
    }
    scope.join().throwIfFailed();
}
```

10000 VT созданы, но только 100 concurrent реально в БД. Остальные ждут в `semaphore.acquire()` (VT unmount'ит carrier при wait).

## VT и JDBC — bottleneck остаётся

Классическая ловушка. Приложение мигрирует на virtual threads, ожидая, что тысячи concurrent запросов к БД будут работать. На практике — HikariCP connection pool из 20 соединений (default) становится bottleneck. 1000 VT ждут 20 connections — 980 в очереди в `pool.getConnection()`.

VT не создают магическую scalability. Они убирают cost на само создание threads и на context switching, но не решают проблему ограниченных downstream ресурсов.

Что делать:

- **Увеличить pool size** — но БД имеет свой лимит (Postgres `max_connections` обычно 100-200).
- **Batch queries** — вместо 1000 запросов один с `IN (...)`.
- **Semaphore для контроля concurrency** — явно ограничить N concurrent DB calls перед acquire connection.
- **R2DBC** — reactive JDBC, но тогда обычно не нужен VT (уже async).

Плюс к этому — JDBC driver'ы должны быть VT-aware. С Java 21 стандартные драйверы (PostgreSQL JDBC 42.5+, MySQL 8.1+) обновлены на использование NIO / non-blocking sockets. При I/O wait VT правильно unmount'ится. Но старые версии драйверов используют blocking sockets — VT пинится.

## Spring Boot и VT

Spring Boot 3.2+ поддерживает VT из коробки:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Автоматически:

- Tomcat request handler → virtual threads (каждый запрос — свой VT).
- `@Async` → VT.
- `@Scheduled` → VT.
- WebFlux — уже non-blocking, VT не применяется.

Ручная настройка Tomcat handler:

```java
@Bean
TomcatProtocolHandlerCustomizer<?> customizer() {
    return handler -> handler.setExecutor(
        Executors.newVirtualThreadPerTaskExecutor());
}
```

Практическая выгода. Раньше Tomcat имел thread pool 200 threads. При I/O bound endpoints (HTTP → external API → response) 200 concurrent запросов — потолок. С VT — 10000+ concurrent, без изменения кода.

Но: backpressure на downstream. Если ваш сервис делает вызов к внешнему API, и API держит 200 connections limit — теперь 10000 VT ждут этих 200 connections. Downstream сервис страдает больше.

Правило: VT — не автомат. Требует мышления о backpressure на всей цепи.

## Debug и observability

**jstack** — классический tool. Дампит все threads в текст:

```bash
jstack <pid>
```

Для VT — с Java 21 shows all VT (может быть миллионы), но обычно `jstack` показывает только carriers + few live VT. Для полного dump'а VT — новый tool.

**jcmd Thread.dump_to_file** — Java 21+ scalable dump в JSON:

```bash
jcmd <pid> Thread.dump_to_file -format=json /tmp/threaddump.json
```

Формат хорошо парсится, включает full stacktraces всех VT, показывает mount status. Работает даже для миллиона VT.

**JFR (Java Flight Recorder)** — continuous profiling с низким overhead. Events для thread:

- `jdk.ThreadStart`, `jdk.ThreadEnd`.
- `jdk.ThreadSleep`, `jdk.ThreadPark`.
- `jdk.VirtualThreadStart`, `jdk.VirtualThreadEnd`.
- `jdk.VirtualThreadPinned` — pinning events.
- `jdk.VirtualThreadSubmitFailed`.

Запись:

```bash
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr MyApp
```

Анализ — JMC (Java Mission Control) или jfr CLI.

**Async Profiler** — third-party, но de-facto standard для Java performance:

```bash
./profiler.sh -e cpu -d 30 -f flamegraph.html <pid>
```

Работает с VT корректно. Показывает CPU hotspots.

**Prometheus metrics** через Micrometer:

```
jvm.threads.states{state="runnable"}
jvm.threads.pinned.count           # VT-specific
jvm.threads.virtual.parked.count   # VT-specific
```

Grafana dashboard — visualizing thread pool utilization, pinning events, VT count over time.

**Отладка pinning** через `-Djdk.tracePinnedThreads=full`. При каждом pinning event выводится stack trace. Ищи `<== monitors:1` в trace — точка, где мы в `synchronized` block'е с blocking.

Более structured — через JFR: `jdk.VirtualThreadPinned` events. Показывает call site и duration. Fix — заменить `synchronized` на `ReentrantLock`. Или убрать I/O из critical section.

## Правило миграции на VT — практическая последовательность

Если хотите мигрировать существующее Java-приложение на virtual threads:

1. **Java 21+** и **Spring Boot 3.2+** — обязательное условие.
2. **Замените `Executors.newFixedThreadPool` на `newVirtualThreadPerTaskExecutor`** для I/O-bound работы. CPU-bound задачи оставьте на platform threads.
3. **Найдите `synchronized` блоки с blocking I/O внутри** — grep по коду, `-Djdk.tracePinnedThreads=short` в load tests. Замените на `ReentrantLock`.
4. **Проверьте ThreadLocal usage** — если есть большие caches per-thread, migrate to ScopedValue или пул объектов.
5. **Добавьте backpressure** (Semaphore) для downstream ресурсов — БД, external APIs.
6. **Load tests с k6/JMeter** — замеряйте throughput и latency до и после под нагрузкой. Reality check.
7. **Мониторинг** — JFR + Prometheus VT metrics в production. Ищите pinning events.

## Заключение

Threads — тема, где легко работать без понимания и внезапно упереться в потолок. Мы разобрали четыре уровня: process (изоляция), OS thread (kernel task_struct, `clone()`, ~1 MB stack, kernel scheduling), Java platform thread (1:1 к OS thread, `-Xss` default 1 MB, 5-10K практический max), Java virtual thread (continuation on carrier, ~10 KB heap, миллионы possible, JVM scheduling).

**Synchronization**:

- `synchronized` — простой, но пинит VT при I/O.
- `ReentrantLock` — гибче + VT-safe.
- `volatile` — visibility + ordering, не atomicity.
- Atomic classes — CAS-based lock-free.
- Concurrent collections — thread-safe with fine-grained locking.

**JMM** — happens-before определяет visibility между threads. Volatile/monitor/final/thread lifecycle establish happens-before.

**Virtual threads внутренности**:

- Continuation = snapshot стека.
- Stack chunks хранятся в heap, копируются на carrier при mount.
- Blocking I/O в JDK переписан на LockSupport.park() → unmount → epoll.
- Carrier — platform thread из ForkJoinPool.

**Ловушки VT**:

- Pinning в `synchronized` + I/O — использовать `ReentrantLock`.
- JDBC connection pool — bottleneck, semaphore для rate limit.
- ThreadLocal — memory issue при миллионах VT, использовать ScopedValue.
- Backpressure не появляется автоматически — semaphore/rate limit для downstream.
- CPU-bound — VT не быстрее platform, только overhead.

**Debugging**: `jstack`, `jcmd Thread.dump_to_file -format=json`, JFR events, async-profiler.

Практический совет для становления. Возьмите один Spring Boot микросервис КНП. Замерьте текущий throughput под нагрузкой k6/JMeter (например, 200 concurrent HTTP request'ов). Включите `spring.threads.virtual.enabled=true`. Запустите тот же load test. Смотрите на metrics: если throughput не вырос, значит либо ваш bottleneck — CPU или downstream, либо есть pinning (`-Djdk.tracePinnedThreads=full` покажет). Разберитесь с pinning, добавьте backpressure на downstream, повторите замер. Реальное улучшение приходит только после этой практики — VT не автомат.

Дальше — читайте JEP 444 (Virtual Threads spec), JEP 428 (Structured Concurrency), JEP 429 (Scoped Values). Портируйте маленький сервис на VT в staging. Load test. Смотрите JFR events. И только потом трогайте прод.
