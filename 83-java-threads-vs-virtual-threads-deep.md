# 83. Threads в Java глубоко: Platform vs Virtual — от OS до Loom

Файл про **потоки на всех уровнях**: от OS thread через JVM platform thread до virtual thread (Project Loom). Что реально происходит в ядре, как JVM их использует, чем отличаются, где ловушки.

Связано с: `19-java-21-virtual-threads.md` (базовый обзор VT — тут углубляем), `43-java-basics-primitives-memory.md` (JVM памяти).

---

## 0. Ментальная модель — 4 уровня

Слово "поток" перегружено. Разделяй:

```
┌──────────────────────────────────────────────┐
│ Level 4: Virtual Thread (JVM-managed)        │  Java 21+, миллионы, ~5KB stack на heap
│         ↕ mount/unmount                      │
├──────────────────────────────────────────────┤
│ Level 3: Platform Thread (java.lang.Thread)  │  Java-обёртка над OS thread, 1:1 mapping
│         ↕ thin wrapper                       │
├──────────────────────────────────────────────┤
│ Level 2: OS Thread (kernel thread)           │  Linux task_struct, ~1MB stack, kernel-scheduled
│         ↕ syscall clone()                    │
├──────────────────────────────────────────────┤
│ Level 1: Process                             │  PID, VM space, FDs, containers для threads
└──────────────────────────────────────────────┘
```

**Ключевое**: до Java 21 в JVM было ТОЛЬКО одно понимание thread'а — platform thread, обёртка вокруг OS thread. **1 Java thread = 1 OS thread** (1:1 model). Virtual threads добавляют слой **M:N mapping** — миллионы Java virtual threads на десятки OS-потоков (carrier'ов).

---

## 1. Что такое поток на уровне OS

### 1.1 Процесс vs поток

**Процесс** — единица изоляции OS:
- Свой virtual address space (memory).
- Свой набор file descriptors.
- Свой user/group ID.
- Свой working directory.

**Thread** — единица планирования внутри процесса:
- Все threads процесса **шарят memory** (heap, globals).
- Каждый thread имеет свой **stack**, свой **register set**, свой **program counter**.
- Кернал планирует threads (не процессы напрямую).

В Linux разделение между thread и process — **условное**: и то и другое = `task_struct` в ядре. Просто threads создаются с флагом `CLONE_VM` — шарят address space. Процессы — без флага.

### 1.2 clone() syscall

Всё создание threads/processes в Linux — через один syscall `clone()`:
```c
long clone(unsigned long flags, void *child_stack, ...);
```

Flags определяют что шарить:
- `CLONE_VM` — memory (обязательно для thread).
- `CLONE_FS` — filesystem info.
- `CLONE_FILES` — open file descriptors.
- `CLONE_SIGHAND` — signal handlers.
- `CLONE_THREAD` — та же thread group (одинаковый PID для `getpid()`, разный TID).

`pthread_create` в glibc — обёртка над `clone` с типовым набором флагов для POSIX thread.

`fork()` — тот же `clone()` без CLONE_* флагов — полное copy-on-write разделение.

### 1.3 Стек OS thread

Каждый OS thread имеет свой **stack** — область памяти для local variables, return addresses, function frames.

Размер стека в Linux по умолчанию — **8 MB** (ulimit -s). Но не резервируется физически сразу: virtual mapping + on-demand page allocation. Реально используется столько, сколько глубина рекурсии + local vars.

**Guard page** — специальная защитная страница внизу стека. Если thread рекурсирует так глубоко что попадает в guard page → SIGSEGV → JVM конвертирует в `StackOverflowError`.

JVM переопределяет стек OS thread'а через флаг `-Xss` (по умолчанию **1 MB на 64-bit**):
```
-Xss512k    # 512 KB на thread
-Xss2m      # 2 MB на thread
```

Меньше `-Xss` = больше threads помещается в память. Больше = меньше `StackOverflowError` при глубокой рекурсии.

### 1.4 Context switch

Kernel переключается между threads (несколько раз в мс). Что происходит:

1. **Save current state**: сохранить registers, PC, стек pointer в TCB (Thread Control Block).
2. **Update scheduler bookkeeping**: statistics, priority queues.
3. **Restore new thread state**: загрузить registers из TCB нового.
4. **TLB flush** (если разные address spaces): expensive для процессов, для threads одного процесса — не нужно.
5. **Cache invalidation**: L1/L2 cache холодный, warm-up после switch.

Стоимость: **~1-10 microseconds** на воспроизводимом hardware. Плюс cache warm-up — реально ~50-100 microseconds для полного восстановления производительности.

**Voluntary switch** (thread сам блокируется на I/O) — часто.
**Involuntary switch** (time slice expired, preempted) — тоже часто.

### 1.5 Scheduling

Linux использует **CFS (Completely Fair Scheduler)** — каждому thread'у дают время proportionally to priority (nice value).

Real-time scheduling (`SCHED_FIFO`, `SCHED_RR`) — для критичных задач, вне обычной handedness.

**Priority в Java** (`Thread.setPriority(1-10)`) — маппится на nice value в Linux, но эффект **минимальный** (Linux не даёт Java менять priority ниже обычного nice range без CAP_SYS_NICE).

### 1.6 Максимум threads в системе

Linux имеет несколько лимитов:
- `/proc/sys/kernel/threads-max` — глобальный лимит (обычно ~4 миллиона).
- `/proc/sys/vm/max_map_count` — memory mappings (каждый стек = mapping).
- `ulimit -u` — процессов/threads per user (~1000-30000).
- `RLIMIT_STACK` × N — общая память под все стеки.

Практически: **10000-50000 platform threads в JVM** — уже проблема. RAM забита стеками (5000 × 1MB = 5GB только под стеки).

---

## 2. Java Platform Thread — что это

### 2.1 java.lang.Thread

Java-класс представляет один OS thread. Каждый `new Thread(...)` при `start()` вызывает JNI:

```java
public synchronized void start() {
    // ... state checks
    start0();  // native method
}
```

`start0()` — JNI, вызывает `pthread_create` в JVM (`os_linux.cpp` в HotSpot).

**1 Java Thread ↔ 1 OS thread (pthread) ↔ 1 kernel task_struct**. Один-к-одному.

### 2.2 Thread states

```
NEW  →  RUNNABLE  ⇄  BLOCKED  (waiting for monitor)
                ⇄  WAITING  (Object.wait, park)
                ⇄  TIMED_WAITING  (sleep, timed wait)
                →  TERMINATED
```

- **NEW** — Thread создан, но `start()` не вызван.
- **RUNNABLE** — может исполняться (running или ready to run — Java не различает).
- **BLOCKED** — ждёт входа в `synchronized` (monitor lock).
- **WAITING** — Object.wait(), LockSupport.park(), Thread.join() без timeout'а.
- **TIMED_WAITING** — то же с timeout'ом. Thread.sleep(), wait(ms).
- **TERMINATED** — run() завершилась.

Смотреть: `Thread.getState()` или `jstack <pid>`.

### 2.3 Stack

Java thread stack = обычный OS thread stack. Размер — `-Xss`.

Хранит:
- **Frame'ы** — по одному на каждый вызов метода.
- В frame: local variables (primitive + reference), operand stack, return PC.
- Reference'ы указывают в **heap** — вот там реальные объекты.

Frame layout:
```
+------------------+
| Return PC        |
| Frame Pointer    |
| Local Variables  |  (в том числе `this`, args, локальные)
| Operand Stack    |  (JVM stack machine, для арифметики)
+------------------+
```

Каждый Java-метод компилируется в bytecode, который работает с **operand stack** этого frame'а. `iadd`, `iload`, `istore` — операции на operand stack.

### 2.4 Стоимость создания platform thread

Замеры для типичного Linux 64-bit:
- **Time**: ~50-500 μs (syscall + инициализация).
- **Memory**: `-Xss` (default 1MB) = зарезервировано в virtual space. Физически — по мере использования.
- **File descriptors**: 2-3 на thread (eventfd, mutexes).

Не большая цена per thread, но **накапливается**. Тысячи threads = гигабайты virtual space + файловые дескрипторы.

### 2.5 Демо-потоки (daemon)

`thread.setDaemon(true)` — daemon threads не мешают JVM завершиться. Когда все не-daemon threads закончились, JVM exit'ит, даже если daemons ещё работают.

Использование: background tasks (GC, JIT compiler, scheduled executors), которые не должны продлевать жизнь приложения.

Правило: пул executor'а обычно **daemon** — иначе приложение зависает при shutdown.

---

## 3. Thread pools — как правильно управлять platform threads

### 3.1 Зачем нужны пулы

Создание thread на каждый task:
- Дорого (50-500 μs).
- Ресурсоёмко (стек, FDs).
- Может привести к OOM при burst.

Пул — переиспользование threads: task в очередь → free worker берёт task → выполняет → возвращается в pool.

### 3.2 ThreadPoolExecutor — базовая реализация

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

**Логика submit**:
1. Если running workers < `corePoolSize` → создать новый worker, дать task.
2. Иначе — попытаться положить в queue.
3. Если queue **полна** и workers < `maximumPoolSize` → создать worker.
4. Если workers == `maximumPoolSize` → rejection handler.

### 3.3 Queue types

- **SynchronousQueue** — нет buffer'а, каждый offer ждёт poll. Идея: «либо worker сразу берёт, либо создаём новый».
  - Использует `Executors.newCachedThreadPool()`. Даёт unlimited threads (опасно).
  
- **LinkedBlockingQueue** (unbounded) — очередь без лимита. Workers остаются на `corePoolSize`, всё копится в queue.
  - Использует `Executors.newFixedThreadPool()`. Опасность: очередь может расти до OOM.
  
- **ArrayBlockingQueue** (bounded) — фиксированный размер. Разумный выбор для prod.

### 3.4 Rejection policies

Что делать когда queue полна и workers на maximum:
- **AbortPolicy** (default) — `RejectedExecutionException`.
- **CallerRunsPolicy** — task выполняется в потоке, который submit'ил (backpressure).
- **DiscardPolicy** — тихо игнорировать.
- **DiscardOldestPolicy** — выбросить самый старый task из queue, положить новый.

Для prod часто — **CallerRunsPolicy** (естественный backpressure). Или custom с логированием.

### 3.5 Anti-patterns Executors factory

`Executors.newFixedThreadPool(N)` — использует `LinkedBlockingQueue` без лимита. Если producer'ы быстрее consumer'ов — очередь растёт до OOM.

`Executors.newCachedThreadPool()` — `SynchronousQueue` + unlimited threads. Burst создаст тысячи threads → OOM или system freeze.

**Правило**: используй `new ThreadPoolExecutor(...)` напрямую с bounded queue.

### 3.6 ForkJoinPool — work-stealing

Специализированный пул для divide-and-conquer. Каждый worker имеет свою **deque** (double-ended queue). Task разбивается на subtask'и через `fork()`.

**Work stealing**: если worker закончил свои tasks, «крадёт» из хвоста deque другого worker'а. Балансирует нагрузку без centralized queue.

Использования:
- `parallelStream()` — использует `ForkJoinPool.commonPool()`.
- `CompletableFuture.async()` без executor'а — commonPool.
- **Carrier threads для virtual threads** (default).

Default parallelism = `Runtime.getRuntime().availableProcessors()`.

**Осторожно**: если task в parallelStream блокируется — commonPool забит, вся система тормозит. Не мешай CPU-bound и I/O-bound в одном commonPool.

---

## 4. Synchronization primitives — как threads координируются

Стандартные механизмы, критичны для понимания где virtual threads поведут себя по-другому.

### 4.1 synchronized

Ключевое слово Java. Каждый объект имеет **monitor** (mutex). `synchronized(obj)` захватывает monitor. Другие threads блокируются на входе.

```java
private final Object lock = new Object();

public void method() {
    synchronized (lock) {
        // critical section
    }
}
```

**Внутренности**:
- В bytecode: `monitorenter` и `monitorexit`.
- В JVM — оптимизации:
  - **Biased locking** — если один поток захватывает lock часто (устарело в JDK 15+, удалено в 18).
  - **Thin lock** — CAS на mark word объекта (без OS mutex).
  - **Fat lock** (inflated) — реальный OS mutex, если contention высокий.

Проблемы:
- **Не interruptible**: `thread.interrupt()` не выведет из `synchronized`.
- **Не fair**: любой ждущий может получить lock (голодание).
- **Нет timeout**.
- **Pinning virtual threads** (см. §6).

### 4.2 volatile

Модификатор поля. Гарантирует **visibility** между потоками:
- Запись в volatile → **memory barrier** (store-release).
- Чтение volatile → **memory barrier** (load-acquire).
- Формирует **happens-before** отношение.

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

**Не даёт атомарности** для `count++` (это read-modify-write). Для этого — `AtomicInteger`.

### 4.3 java.util.concurrent.locks

**ReentrantLock** — как synchronized, но лучше:
```java
Lock lock = new ReentrantLock();
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}
```

Плюсы:
- `tryLock(timeout)` — с таймаутом.
- `lockInterruptibly()` — interruptible.
- `newCondition()` — множественные условия ожидания (вместо одного monitor).
- Fair mode (`new ReentrantLock(true)`).
- **Не пинит virtual threads!** — ключевое для Loom.

**ReadWriteLock** — читателей много (параллельно), писатель один (эксклюзивно).
```java
ReadWriteLock rwl = new ReentrantReadWriteLock();
rwl.readLock().lock();   // многие могут держать
rwl.writeLock().lock();  // только один
```

Для read-heavy workload — большой выигрыш.

**StampedLock** (Java 8+) — три режима: read, write, optimistic read. Быстрее ReadWriteLock, но сложнее использовать (optimistic read требует manual validation).

### 4.4 Semaphore

Ограничивает concurrent access. Внутри — счётчик permits.
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
- **Backpressure для virtual threads** — важно.

### 4.5 CountDownLatch, CyclicBarrier, Phaser

**CountDownLatch** — «жди пока N threads не сделают countDown». Одноразово.
```java
CountDownLatch latch = new CountDownLatch(3);

// Workers:
worker.run(() -> { doWork(); latch.countDown(); });

// Main:
latch.await();  // ждёт пока 3 воркера не вызовут countDown
```

**CyclicBarrier** — «все N threads встретятся здесь». Многоразовый.
```java
CyclicBarrier barrier = new CyclicBarrier(4);
barrier.await();  // ждёт пока 4 threads не подтянутся
```

**Phaser** — как CyclicBarrier, но динамическое количество threads. Сложнее, реже используется.

### 4.6 Atomic classes (CAS-based)

`AtomicInteger`, `AtomicLong`, `AtomicReference` — lock-free через **CAS (Compare-And-Swap)**.

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

Быстрее synchronized при низком contention. При высоком — cache-line bouncing (много CAS retry).

**LongAdder** — «distributed» AtomicLong для high-contention случаев. Массив ячеек, каждый thread пишет в свою. `sum()` — сумма всех. Быстрее AtomicLong при contention, чуть медленнее при чтении.

### 4.7 Concurrent Collections

**ConcurrentHashMap** — thread-safe HashMap. С Java 8 внутри:
- Bin по индексу — обычный HashMap bucket.
- Empty bin — CAS для insert.
- Non-empty — synchronized блок на первом node bin'а.
- Read — lock-free (через volatile happens-before).

**CopyOnWriteArrayList** — при write создаёт **новую копию массива**. Читатели видят старую. Дорого для write, идеально для read-heavy + rare write (списки listeners, config).

**BlockingQueue implementations**:
- `ArrayBlockingQueue` — fixed size, one lock (fairness option).
- `LinkedBlockingQueue` — optionally bounded, two locks (head + tail).
- `SynchronousQueue` — 0 capacity, handoff.
- `PriorityBlockingQueue` — heap-based priority.
- `DelayQueue` — задержанное потребление.

---

## 5. Java Memory Model (JMM) — необходимо понять чтобы использовать threads

### 5.1 Проблема

Без специальных гарантий compiler/CPU может **переупорядочить** операции:
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
- **Compiler optimizations** (JIT).
- **CPU out-of-order execution**.
- **Store buffers** (write не сразу в memory).
- **Cache coherence delays**.

### 5.2 Happens-before

JMM определяет **partial order**: если A happens-before B, то эффекты A видны в B.

Rules:
- **Program order** — операции внутри одного thread'а в порядке.
- **Monitor lock** — unlock happens-before последующего lock того же monitor'а.
- **Volatile** — write happens-before последующего read того же volatile.
- **Thread start** — start() happens-before actions в новом thread'е.
- **Thread join** — actions в thread happens-before join() returns.
- **Final fields** — init в конструкторе happens-before любого read final field'а (если объект не escape'ится из конструктора).

### 5.3 Volatile практически

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

Volatile write в (2) happens-before volatile read в (3) → всё что было до (2) видимо в (4).

### 5.4 Двойная проверка singleton (double-checked locking)

Классическая ошибка:
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

Проблема: (3) — это несколько операций (allocate → constructor → assign). Другой поток может увидеть **частично сконструированный** объект в (1).

Fix: **`private static volatile Lazy INSTANCE;`**. Volatile гарантирует visibility и предотвращает reorder.

Или ещё проще:
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

---

## 6. Virtual Threads — глубокие internals

Уже базово покрыто в `19-java-21-virtual-threads.md`. Здесь — внутренности.

### 6.1 Continuation — фундамент Loom

**Continuation** — snapshot текущего состояния выполнения (stack + PC + registers), который можно **приостановить** и **возобновить**.

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

Virtual thread — это Continuation, mounted на carrier thread. При block'е на I/O — **yield** → carrier свободен для другого VT.

### 6.2 Stack chunks — как хранится стек VT

Обычный OS thread стек = contiguous memory allocation, ~1MB. Слишком дорого для миллионов threads.

VT стек хранится в **heap** как **stack chunk** — маленькие blocks (~256B - 32KB, дырявый allocation). При mount:
1. JVM копирует stack chunks из heap на реальный OS stack carrier'а.
2. VT исполняется, стек растёт на real stack.
3. При unmount: копируется обратно в heap (только использованная часть).

Итог: VT потребляет только **фактически используемый стек** в heap. Тысячи мелких VT ~5-20KB каждый. Миллион VT — 5-20 GB (сильно зависит от глубины стека).

### 6.3 Mount / Unmount lifecycle

**Mount**: VT прикрепляется к carrier thread. Carrier runs VT code as if it were own.
**Unmount**: VT останавливается, carrier свободен для other VT.

Unmount происходит:
- **I/O blocking calls**: `Socket.read`, `Files.read`, JDBC.
- `LockSupport.park` (базовый механизм — используется в ReentrantLock, Semaphore, etc.).
- `Thread.sleep`.
- `Thread.yield`.

Ключевое: **JVM переписывает JDK blocking API** чтобы они yield'или continuation вместо блокировки OS thread'а.

Пример: `Socket.getInputStream().read()` в Java 21 — internally проверяет is VT? → если да → `LockSupport.park` + async I/O через epoll/kqueue → registration в selector.

### 6.4 Carrier thread pool

Default: `ForkJoinPool` с parallelism = `availableProcessors()`.

Configuration:
```
-Djdk.virtualThreadScheduler.parallelism=16     # число carrier'ов
-Djdk.virtualThreadScheduler.maxPoolSize=256   # верхний лимит
-Djdk.virtualThreadScheduler.minRunnable=1     # минимум активных
```

Carriers — daemon platform threads с именем `ForkJoinPool-1-worker-N`.

**Смотреть carriers**:
```java
Thread.currentThread();  // если VT — carrier доступен через .toString()
// "VirtualThread[#21]/runnable@ForkJoinPool-1-worker-3"
```

### 6.5 Pinning — детали

**Pinning** = VT НЕ может unmount'иться, carrier заблокирован.

Причины:
1. **`synchronized` block** с blocking call внутри.
2. **Native method** (JNI) — JVM не контролирует.
3. **File I/O** до Java 21 (fixed в 21 через io_uring, но не все pathways).

**Как проверить**:
```
-Djdk.tracePinnedThreads=full     # каждое pinning event с полным stack trace
-Djdk.tracePinnedThreads=short    # только summary
```

**Fix synchronized pinning** — заменить на `ReentrantLock`:
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

### 6.6 Что происходит при I/O

```java
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

Ключевая идея: **blocking API на surface, non-blocking underneath**. Программист пишет привычный код, JVM превращает в async.

### 6.7 ThreadLocal и VT — проблема

ThreadLocal хранит одно значение на thread. С миллионом VT — миллион копий:
```java
ThreadLocal<byte[]> BUFFER = ThreadLocal.withInitial(() -> new byte[8192]);
// 1M VT × 8KB = 8GB памяти
```

**Решения**:

1. **ScopedValue** (Java 21 preview, стабилизируется):
   ```java
   static final ScopedValue<String> USER = ScopedValue.newInstance();
   
   ScopedValue.where(USER, "berik").run(() -> {
       // USER.get() == "berik"
       nestedCall();
   });
   ```
   Immutable, ограничена scope. Не наследуется автоматически child threads.

2. **Пул объектов** вместо ThreadLocal кэша:
   ```java
   BUFFER_POOL.borrow(buffer -> {
       // use buffer
   });
   ```

3. **Просто аллокация**:
   ```java
   byte[] buffer = new byte[8192];   // GC разберётся
   ```
   Small buffer'ы — TLAB (Thread-Local Allocation Buffer), фактически бесплатно для GC.

---

## 7. Platform vs Virtual — детальное сравнение

| Аспект | Platform Thread | Virtual Thread |
|--------|-----------------|----------------|
| **Underlying** | 1:1 к OS thread | Continuation on carrier |
| **Creation cost** | 50-500 μs, syscall | ~1 μs, heap allocation |
| **Memory** | `-Xss` (default 1 MB) reserved | ~5-20 KB used in heap |
| **Maximum count** | ~5-10K на серверe (RAM limit) | Миллионы |
| **Scheduling** | OS kernel (CFS) | JVM scheduler (ForkJoinPool) |
| **Context switch** | ~1-10 μs (kernel) | ~200-500 ns (JVM) |
| **Priority** | OS priority (nice) | Игнорируется |
| **ThreadLocal cost** | Malo (тысячи копий) | Может быть проблемой (миллионы) |
| **Debugging** | jstack, jconsole | Могут быть huge dumps |
| **When to use** | CPU-bound, small pool | I/O-bound, many concurrent |
| **Pinning issues** | N/A | synchronized + I/O, native code |
| **Blocking cost** | Freezes OS thread | Unmounts, carrier free |
| **Thread pool** | Обязателен для нормальной работы | Не нужен (per-task VT дёшев) |

### 7.1 Правильный use case для каждого

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

### 7.2 Ошибки миграции на VT

**Использовать VT для CPU-bound** — не быстрее, часто медленнее (overhead scheduler'а).

**`Executors.newFixedThreadPool(N)` заменить на `newVirtualThreadPerTaskExecutor()` без думалок** — если у тебя всего 10 tasks, старый пул может быть лучше (нет overhead создания VT).

**Не удалить synchronized блоки** — если внутри I/O → carriers пинятся → «virtual threads не помогают».

**Забыть про backpressure** — VT дешёвые, но downstream (БД, external API) не масштабируется вместе. Semaphore для rate limiting.

**ThreadLocal caches** — 1M копий по 10KB = 10GB. Rethink.

---

## 8. Real-world примеры

### 8.1 Fan-out: собрать данные из 3 сервисов

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

### 8.2 Обработка Kafka batch

```java
// Прочитали 500 records из Kafka
List<Record> records = consumer.poll(...);

try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    List<Subtask<Result>> results = records.stream()
        .map(r -> scope.fork(() -> processRecord(r)))
        .toList();
    
    scope.join().throwIfFailed();
    
    List<Result> ok = results.stream().map(Subtask::get).toList();
    consumer.commitSync();
}
```

500 VT параллельно обрабатывают records. Если каждый делает 3 DB call'а — тысячи concurrent connections **необходимо ограничить semaphore'ом** или через DB pool size.

### 8.3 Ограничение concurrency через Semaphore

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

10000 VT созданы, но только 100 concurrent реально в БД. Остальные ждут в `semaphore.acquire()` (VT unmount'ит carrier).

---

## 9. Debug и observability

### 9.1 jstack

Классический tool. Дампит все threads в текст:
```bash
jstack <pid>
```

Для VT — с Java 21 shows all VT (может быть миллионы), но обычно `jstack` показывает только carriers + few live VT.

### 9.2 jcmd Thread.dump_to_file

Java 21+ — новый более scalable dump в JSON:
```bash
jcmd <pid> Thread.dump_to_file -format=json /tmp/threaddump.json
```

Формат хорошо парсится, включает full stacktraces всех VT, показывает mount status.

### 9.3 JFR (Java Flight Recorder)

Continuous profiling с низким overhead. Events для thread:
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

### 9.4 Async Profiler

Third-party, но de-facto standard для Java performance:
```bash
./profiler.sh -e cpu -d 30 -f flamegraph.html <pid>
```

Работает с VT корректно. Показывает CPU hotspots.

### 9.5 Prometheus metrics

Через Micrometer:
```java
Metrics.gauge("jvm.threads.states", 
    Tags.of("state", "runnable"), 
    Thread.getAllStackTraces(), 
    m -> countByState(m, State.RUNNABLE));
```

VT metrics — новые:
```
jvm.threads.pinned.count
jvm.threads.virtual.parked.count
```

Grafana dashboard — visualizing thread pool utilization, pinning events, VT count over time.

---

## 10. Собесные вопросы с ответами

### Q1: Что такое thread на уровне OS?

Kernel thread — единица планирования внутри процесса. В Linux — `task_struct` в ядре. Threads одного процесса шарят memory, file descriptors, PID. У каждого — свой stack (обычно 1-8 MB), свой register set, PC.

Создаётся через syscall `clone()` с флагами (`CLONE_VM`, `CLONE_FILES`, `CLONE_THREAD`). `pthread_create` в glibc — обёртка над clone.

Kernel scheduler (CFS в Linux) переключает threads между CPU. Context switch — сохранение/восстановление register state, ~1-10 μs.

### Q2: Как Java Thread связан с OS thread?

**1:1 mapping** для platform threads. `new Thread().start()` → JNI → `pthread_create` → OS thread. Один Java Thread object = один OS thread = один kernel task_struct.

Для virtual threads — **M:N**: миллионы VT на десятки carrier'ов (platform threads в ForkJoinPool). JVM sам управляет scheduling'ом VT между carriers.

### Q3: Сколько platform threads можно создать в JVM?

Ограничено памятью и OS лимитами.

- Stack size `-Xss` (default 1MB) × N threads = virtual memory reserved.
- OS `ulimit -u` — максимум threads per user.
- Linux `/proc/sys/vm/max_map_count` — thread'ы создают mappings.

Практически: **5-10 тысяч** на приличном сервере — уже проблема. Стеки жрут память, много threads → много context switches → cache misses.

### Q4: Как работают Virtual threads?

VT — легковесная Java-обёртка вокруг **continuation** (snapshot стека). Не 1:1 к OS thread'у.

**Mount**: JVM выбирает carrier (platform thread из ForkJoinPool), копирует stack chunks из heap на carrier stack, начинает выполнение.

**Unmount**: При blocking I/O или `park()` — JVM копирует стек обратно в heap, освобождает carrier для другого VT.

**Ключевое**: JVM переписывает стандартные JDK blocking API (`Socket.read`, JDBC, `Thread.sleep`) чтобы они yield'или continuation. Программист пишет обычный императивный код, JVM превращает в async underneath.

### Q5: В чём разница platform и virtual threads?

| | Platform | Virtual |
|-|----------|---------|
| Underlying | OS thread | Continuation on carrier |
| Cost | 1 MB stack, μs to create | ~10 KB heap, ns to create |
| Max count | Тысячи | Миллионы |
| Scheduling | OS kernel | JVM ForkJoinPool |
| Blocking | Freezes OS thread | Unmounts, carrier free |
| Best for | CPU-bound | I/O-bound |

### Q6: Когда virtual threads не помогают?

1. **CPU-bound** — виртуальность не ускоряет compute. VT не быстрее platform для чистого CPU.
2. **Много `synchronized` с блокирующим I/O внутри** — pinning убивает преимущество, carriers заняты.
3. **JNI-heavy** — native code pins carrier до возврата.
4. **Downstream не масштабируется** — миллион VT ждут DB connection из pool в 20 → 999980 в очереди, узкое место осталось.
5. **Small number of tasks** — overhead scheduler'а перевесит benefit.

### Q7: Что такое pinning в VT?

Pinning — ситуация когда VT **не может unmount'иться**, carrier остаётся заблокированным на I/O.

Причины:
- Blocking внутри `synchronized` block — JVM не yield'ит стек синхронизированной области.
- Native method call (JNI) — JVM теряет контроль.

Fix:
- Заменить `synchronized` на `ReentrantLock` — pinning-safe.
- Диагностика: `-Djdk.tracePinnedThreads=full`.

### Q8: Что такое carrier thread?

Platform thread из специального пула (ForkJoinPool по умолчанию), на котором JVM выполняет virtual threads. Количество — по умолчанию `availableProcessors()`.

Один carrier может обслужить много VT последовательно: mount → execute до block → unmount → mount другой VT.

Настройка:
```
-Djdk.virtualThreadScheduler.parallelism=16
```

### Q9: Разница ReentrantLock и synchronized?

Функционально похожи — оба взаимоисключающая критическая секция.

**ReentrantLock преимущества**:
- `tryLock(timeout)` — с таймаутом.
- `lockInterruptibly()` — interruptible.
- Fair mode (`new ReentrantLock(true)`).
- Multiple Condition per lock (через `newCondition()`).
- **Не пинит virtual threads** — критично для Loom.
- Явный API, легче ошибиться.

**synchronized преимущества**:
- Компактный синтаксис.
- Не забыть `unlock()` (RAII через блок).
- JVM оптимизации (biased lock — удалено, thin/fat).

Для нового кода на Java 21+ — **ReentrantLock по умолчанию**.

### Q10: Что такое happens-before?

Правило JMM: если A happens-before B, эффекты A видны в B (memory operations правильно упорядочены).

Rules:
- Program order — операции в одном thread'е.
- Monitor: unlock happens-before subsequent lock.
- Volatile: write happens-before subsequent read.
- Thread start: start() happens-before actions в новом thread'е.
- Thread join: actions in thread happens-before join() return.
- Final fields: constructor init happens-before any read.

Без happens-before — компилятор/CPU могут reorder операций, thread может увидеть stale data.

### Q11: Зачем нужен volatile?

**Visibility**: гарантирует что запись из одного thread'а видима другому thread'у (иначе JIT может inline'ить значение в register).

**Ordering**: memory barriers на write (store) и read (load) — предотвращают reorder.

**Не даёт атомарности** для read-modify-write операций (`counter++`). Для этого — `AtomicInteger` или `synchronized`.

Пример: `volatile boolean running = true` — стандарт для stop flag'а между threads.

### Q12: Почему `Executors.newFixedThreadPool(N)` опасен?

Использует `LinkedBlockingQueue` **без лимита**. Если producer'ы быстрее consumer'ов — очередь растёт неограниченно → OOM.

Правильнее — `new ThreadPoolExecutor(...)` напрямую с **bounded queue** (например `ArrayBlockingQueue(1000)`) + rejection policy (например `CallerRunsPolicy` для backpressure).

### Q13: Что такое ForkJoinPool и work-stealing?

Специализированный пул для divide-and-conquer задач. Каждый worker имеет свою **deque** (double-ended queue). Task разбивается на subtask'и через `fork()`.

**Work stealing**: worker закончивший свои tasks крадёт задачи из хвоста deque другого worker'а. Балансирует load без centralized queue → лучше scaling.

Используется:
- `parallelStream()`.
- `CompletableFuture.async()` без явного executor'а.
- **Carrier pool для virtual threads** (default).

### Q14: Как ThreadLocal ведёт себя с virtual threads?

ThreadLocal хранит **одно значение на thread**. С миллионом VT — миллион копий → память как проблема.

Решения:
1. **ScopedValue** (Java 21+) — immutable, scoped, лучше для VT.
2. Пул объектов вместо кэша.
3. Обычная аллокация (small objects через TLAB — почти бесплатно).

Не убирай ThreadLocal слепо — для carriers их немного, ThreadLocal остаётся ok. Только для **VT-per-request** паттерна нужно думать.

### Q15: Что такое CAS (Compare-And-Swap)?

Атомарная операция: сравнить текущее значение с expected и, если совпадает, заменить на new. В hardware — обычно `LOCK CMPXCHG` на x86.

Основа lock-free алгоритмов и `AtomicInteger`:
```java
public int incrementAndGet() {
    int current;
    do {
        current = get();
    } while (!compareAndSet(current, current + 1));  // CAS
    return current + 1;
}
```

Быстрее synchronized при низком contention (нет OS lock). При высоком — retry overhead (cache line bouncing между CPU).

### Q16: Зачем структурированная concurrency?

**StructuredTaskScope** (Java 21 preview → 23 stable) — управление жизненным циклом группы связанных задач.

Проблема без неё:
```java
Future<X> f1 = executor.submit(...);
Future<Y> f2 = executor.submit(...);
// Если f1 упадёт — f2 всё равно выполняется, ресурсы утекают
```

Решение:
```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<X> s1 = scope.fork(() -> task1());
    Subtask<Y> s2 = scope.fork(() -> task2());
    scope.join().throwIfFailed();
    return combine(s1.get(), s2.get());
}
```

- При ошибке одной — остальные отменяются.
- Scope guarantees что все закончились до выхода из try-with-resources.
- Читабельный код, единый parent-child scope.

### Q17: Почему `AtomicLong` может быть медленнее `LongAdder`?

`AtomicLong` — одна ячейка. При high contention (много threads пишут одновременно) — cache line bouncing: каждый CPU периодически invalidate'ит cache других.

`LongAdder` — распределённое хранение по массиву cells. Каждый thread пишет в свою cell (по hash TID). При read — сумма всех cells.

Trade-off: write быстрее, read медленнее. Для counter'ов (write-heavy, редко читаем) — LongAdder win.

### Q18: Как VT работает с JDBC?

**Проблема**: JDBC blocking по природе. Если VT делает `preparedStatement.executeQuery()` — блокирует carrier до response БД.

**С Java 21**: JDBC driver'ы обновляются на использование NIO / non-blocking sockets внутри → при I/O wait VT правильно unmount'ится.

**Но**: JDBC connection pool (HikariCP default 10-20) остаётся bottleneck'ом. 1000 VT ждут 20 connections → 980 в очереди в `pool.getConnection()`.

**Решение**:
- Увеличить pool size (но БД имеет свой лимит).
- Batch queries.
- Semaphore для контроля concurrency перед acquire connection.
- R2DBC — reactive JDBC, но тогда обычно не нужен VT (уже async).

### Q19: Как включить VT в Spring Boot?

Spring Boot 3.2+:
```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Автоматически:
- Tomcat request handler → virtual threads.
- @Async → VT.
- @Scheduled → VT.
- WebFlux — уже non-blocking, VT не применяется.

Ручная настройка:
```java
@Bean
TomcatProtocolHandlerCustomizer<?> customizer() {
    return handler -> handler.setExecutor(
        Executors.newVirtualThreadPerTaskExecutor());
}
```

### Q20: Как отладить pinning?

```
-Djdk.tracePinnedThreads=full
```

При каждом pinning event выводится stack trace. Ищи `<== monitors:1` в trace — точка где мы в `synchronized` block'е с blocking.

Более structured — через JFR:
```bash
java -XX:StartFlightRecording=duration=60s,filename=r.jfr,settings=profile MyApp
```

Открой в JMC, найди `jdk.VirtualThreadPinned` events. Показывает call site и duration.

Fix: заменить `synchronized` на `ReentrantLock`. Или убрать I/O из critical section.

---

## 11. Мини-чеклист «прочитал — знаю»

За 3-5 секунд:

- [ ] OS thread — kernel task_struct, clone() syscall.
- [ ] Java platform thread — 1:1 к OS thread, JNI pthread_create.
- [ ] Stack — `-Xss` default 1MB на platform, ~10KB heap на VT.
- [ ] Context switch cost: ~1-10 μs OS, ~200-500 ns JVM (VT).
- [ ] Максимум platform threads — 5-10K практически.
- [ ] Максимум virtual threads — миллионы.
- [ ] Thread states: NEW / RUNNABLE / BLOCKED / WAITING / TIMED_WAITING / TERMINATED.
- [ ] ThreadPoolExecutor: core / max / queue / rejection.
- [ ] `Executors.newFixedThreadPool` опасен (unbounded queue).
- [ ] ForkJoinPool — work-stealing, deque per worker.
- [ ] synchronized — monitor lock, thin/fat optimization.
- [ ] volatile — visibility + ordering, но не atomicity.
- [ ] ReentrantLock — tryLock, interruptible, VT-safe.
- [ ] Semaphore — permits, для rate limiting.
- [ ] Happens-before: monitor unlock, volatile write, thread start, thread join, final fields.
- [ ] AtomicInteger — CAS, retry loop.
- [ ] LongAdder — distributed для high contention.
- [ ] ConcurrentHashMap — CAS + synchronized per bin.
- [ ] CopyOnWriteArrayList — read-heavy.
- [ ] Virtual thread = continuation on carrier.
- [ ] Continuation — mount/unmount, stack chunks в heap.
- [ ] Carrier — ForkJoinPool, default = availableProcessors().
- [ ] Pinning — synchronized + I/O, JNI.
- [ ] Fix pinning — ReentrantLock.
- [ ] `-Djdk.tracePinnedThreads=full`.
- [ ] JDBC connection pool — bottleneck для VT.
- [ ] ThreadLocal — проблема для миллионов VT, используй ScopedValue.
- [ ] StructuredTaskScope — fan-out с proper lifecycle.
- [ ] Spring Boot 3.2+ — `spring.threads.virtual.enabled=true`.
- [ ] VT не помогает для CPU-bound.
- [ ] jstack / jcmd Thread.dump_to_file -format=json / JFR / async-profiler.

---

## Итог

**Platform thread** = обёртка над OS thread (1:1 mapping). Дорого (~1MB стек), ограничено количеством (5-10K), OS scheduler.

**Virtual thread** = continuation on carrier thread (M:N mapping). Дёшево (~10KB heap), миллионы, JVM scheduler. Ключевое: **при I/O блокировке unmount'ится с carrier**, carrier свободен для другого VT. Программист пишет привычный blocking код, JVM превращает в async underneath.

**Синхронизация** — общая для обоих:
- `synchronized` — простой, но пинит VT при I/O.
- `ReentrantLock` — гибче + VT-safe.
- `volatile` — visibility + ordering.
- Atomic classes — CAS-based lock-free.
- Concurrent collections — thread-safe with fine-grained locking.

**JMM** — happens-before определяет visibility между threads. Volatile/monitor/final/thread lifecycle establish happens-before.

**Ловушки VT**:
- Pinning в synchronized + I/O — использовать ReentrantLock.
- JDBC connection pool — bottleneck, semaphore для rate limit.
- ThreadLocal — memory issue при миллионах VT, использовать ScopedValue.
- Backpressure не появляется автоматически — нужно semaphore/rate limit.
- CPU-bound — VT не быстрее platform, overhead.

**Debugging**: `jstack`, `jcmd Thread.dump_to_file -format=json`, JFR events, async-profiler.

**Правило миграции**:
1. Java 21+ и Spring Boot 3.2+.
2. Замени `Executors.newFixedThreadPool` на `newVirtualThreadPerTaskExecutor` для I/O-bound работы.
3. Замени `synchronized` (с I/O внутри) на `ReentrantLock`.
4. Trace pinning в тестах.
5. Проверь ThreadLocal — если много, migrate to ScopedValue.
6. Добавь backpressure (semaphore) для downstream ресурсов.

Дальше — читай JEP 444 (Virtual Threads spec), JEP 428 (Structured Concurrency), JEP 429 (Scoped Values). Практика — портируй маленький Spring Boot сервис на VT, замеряй throughput до/после под нагрузкой k6/JMeter.
