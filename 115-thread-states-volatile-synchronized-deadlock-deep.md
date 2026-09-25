# 115. Потоки: состояния, volatile, synchronized, deadlock

## Зачем это знать

Многопоточный Java-код обманчиво прост. Написал `new Thread(() -> ...)`, запустил, «работает». В однопоточных тестах работает, на локальной машине работает, на CI работает. В прод под нагрузкой — раз в тысячу запросов происходят странности: счётчик показывает 999 вместо 1000, кэш возвращает stale-данные через 30 минут после обновления, приложение зависает и не отвечает пока не перезапустишь, thread dump показывает 200 потоков в состоянии BLOCKED на одном мониторе. Каждый из этих симптомов — прямое следствие непонимания как реально работает многопоточность в Java.

Разница между «пишу многопоточный код» и «понимаю concurrency» — это способность отвечать: почему один поток может **никогда** не увидеть изменения сделанные другим (даже спустя часы). Что реально делает `volatile` и почему `count++` не становится атомарным даже если `count` объявлен `volatile`. В чём разница между `synchronized` методом и `synchronized(this)` блоком. Почему deadlock — не «плохой код», а вполне предсказуемая ситуация возникающая при определённом порядке захвата ресурсов. Как читать thread dump и по стеку понимать что за проблема.

Всё сводится к трём фундаментальным вещам, которые надо понять глубоко: **thread states** (что реально делает поток в каждом состоянии), **memory model** (visibility и happens-before), **synchronization primitives** (`volatile`, `synchronized`, `ReentrantLock`, atomics — что каждый гарантирует и что не гарантирует). Без этой базы любой прод-инцидент с многопоточностью превращается в гадание.

Разберём: жизненный цикл потока и все 6 состояний (NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED) — что реально означает каждое, когда происходят переходы, как это выглядит на уровне OS. Race conditions как фундаментальная проблема. Java Memory Model — почему visibility между потоками не гарантирована по умолчанию, что такое happens-before. `volatile` детально — memory barriers, что даёт и что не даёт, типовые применения. `synchronized` детально — intrinsic lock (monitor), reentrant свойство, memory effects, разница между `synchronized` методом и блоком. Полное сравнение `volatile` vs `synchronized` — когда что использовать. Deadlock — механика, dining philosophers, обнаружение в проде через thread dump, стратегии обхода (lock ordering, timeout, tryLock). Livelock и starvation. Альтернативы — `ReentrantLock`, `Atomic*` классы, concurrent collections. Диагностика в проде через `jstack` и thread dump analysis.

Threads overview — файл 83. Virtual threads специфика — 110. Connection pools + I/O wait — 107. Здесь фокус конкретно на состояниях, synchronization primitives и типовых проблемах concurrency.

## Жизненный цикл потока и 6 состояний

Java Thread имеет 6 возможных состояний, определённых enum `Thread.State`. Каждое состояние точно означает что делает (или не делает) поток.

**Диаграмма переходов**:

```
                     ┌──────┐
                     │  NEW  │  ← Thread создан, но не запущен
                     └──┬────┘
                        │ start()
                        ▼
                  ┌──────────┐
                  │ RUNNABLE │  ← выполняется на CPU или готов выполняться
                  └────┬─────┘
                       │
        ┌──────────────┼──────────────┬──────────────┐
        │              │              │              │
        ▼              ▼              ▼              ▼
   ┌─────────┐   ┌─────────┐    ┌──────────┐   ┌────────────┐
   │ BLOCKED │   │ WAITING │    │  TIMED_  │   │ TERMINATED │
   │         │   │         │    │ WAITING  │   │            │
   └────┬────┘   └────┬────┘    └────┬─────┘   └────────────┘
        │             │              │
        │ lock got    │ notified     │ timeout / notified
        │             │              │
        └─────────────┴──────────────┘
                      ▼
                  RUNNABLE
```

### NEW: Thread создан, но не запущен

```java
Thread t = new Thread(() -> doWork());
System.out.println(t.getState());   // NEW
```

Объект Thread существует в куче JVM. Внутренние структуры OS для этого потока **ещё не созданы**. Никаких pthread, никакого стека, никакой памяти для регистров. Только Java-объект.

Практически не встречается в прод-диагностике — окно между `new Thread(...)` и `.start()` обычно микросекунды. Если увидел NEW в thread dump — кто-то создал thread и «забыл» запустить (обычно баг).

### RUNNABLE: выполняется или готов

Самое частое состояние. Означает одно из двух:

1. **Реально выполняется** на CPU прямо сейчас.
2. **Готов выполняться**, ждёт своей очереди в OS scheduler'е.

JVM не различает эти подсостояния — с её точки зрения тред «может исполнять bytecode». Реальное различие только у OS (running vs ready).

**Важная ловушка**: RUNNABLE **включает** blocking system calls. Поток который делает `socket.read()` и ждёт данных от сети — JVM показывает RUNNABLE. Потому что с точки зрения JVM тред «выполняется в native коде», хотя реально он в kernel wait queue сокета не потребляет CPU (см. файл 107).

Чтобы понять что тред RUNNABLE реально делает — смотреть stacktrace:

```
"http-nio-8080-exec-15" #123 daemon prio=5 os_prio=0 tid=0x... nid=0x...
   java.lang.Thread.State: RUNNABLE
        at sun.nio.ch.SocketDispatcher.read0(Native Method)   ← ждёт сеть!
        at sun.nio.ch.SocketDispatcher.read(SocketDispatcher.java:47)
        ...
        at org.postgresql.core.PGStream.receiveChar(...)
        ...
        at ru.example.Repository.findById(Repository.java:42)
```

RUNNABLE + `SocketDispatcher.read0` — ждёт сеть, не CPU. Пример из файла 78 — важно понимать что RUNNABLE не всегда «работает».

### BLOCKED: ждёт monitor lock

Поток пытается войти в `synchronized` блок или метод, но monitor уже удерживается другим потоком. OS ставит его в специальную очередь ожидания monitor'а.

```java
class Cache {
    public synchronized Object get(String key) {   // monitor lock на this
        // ...
    }
}

Cache cache = new Cache();

// Thread 1
cache.get("A");   // получил monitor, выполняется

// Thread 2 (пока Thread 1 в get)
cache.get("B");   // BLOCKED, ждёт освобождения monitor
```

Thread 2 в BLOCKED — не потребляет CPU, kernel держит его в wait queue связанной с этим monitor. Как только Thread 1 выйдет из `synchronized` метода — kernel будит одного из ждущих (fairness zависит от JVM/OS).

В thread dump:

```
"exec-42" BLOCKED (on object monitor)
        at ru.example.Cache.get(Cache.java:15)
        - waiting to lock <0x00000007f8a0cd28> (a ru.example.Cache)
        at ...
```

`waiting to lock <0x...>` — идентификатор monitor'а. По нему можно найти кто **держит** этот monitor (тоже в dump'е `- locked <0x00000007f8a0cd28>`).

Много BLOCKED на одном monitor'е — контеншен, узкое место. Разбор — файл 78.

### WAITING: ждёт условие (без timeout)

Поток вызвал метод, который переводит его в ожидание без определённого timeout. Классические причины:

- `Object.wait()` — ждёт `notify()` от другого потока на том же мониторе.
- `Thread.join()` — ждёт завершения другого треда.
- `LockSupport.park()` — низкоуровневая парковка (используется в concurrent коллекциях, ReentrantLock, etc).

```java
private final Object lock = new Object();

// Thread 1
synchronized (lock) {
    lock.wait();   // WAITING
}

// Thread 2
synchronized (lock) {
    lock.notify();   // Thread 1 переходит в BLOCKED (пытается получить lock обратно), потом RUNNABLE
}
```

В thread dump:

```
"consumer-1" WAITING (parking)
        at jdk.internal.misc.Unsafe.park(Native Method)
        at java.util.concurrent.locks.LockSupport.park(...)
        at ...
        at java.util.concurrent.LinkedBlockingQueue.take(...)
        at ...
```

`WAITING (parking)` с `LockSupport.park` — тред припаркован через futex (см. файл 107). Не потребляет CPU. Разбудить может только явный `unpark(thread)`.

### TIMED_WAITING: ждёт с таймаутом

То же что WAITING, но с определённым таймаутом. Причины:

- `Thread.sleep(long)` — ждёт время.
- `Object.wait(long)` — ждёт notify или timeout.
- `Thread.join(long)` — ждёт завершения другого треда или timeout.
- `LockSupport.parkNanos(long)`, `parkUntil(long)`.
- `BlockingQueue.poll(timeout)`.

При достижении timeout — поток проснётся автоматически. При notify/unpark раньше — проснётся раньше.

В thread dump `TIMED_WAITING (sleeping)` для sleep, `TIMED_WAITING (parking)` для park с timeout, `TIMED_WAITING (on object monitor)` для `wait(long)`.

### TERMINATED: поток завершился

`run()` метод вернул управление (нормально или через exception). Поток больше не выполнится, но объект Thread ещё живёт в куче (пока GC не соберёт).

```java
Thread t = new Thread(() -> System.out.println("hello"));
t.start();
t.join();
System.out.println(t.getState());   // TERMINATED
```

Практически: TERMINATED в thread dump встречается редко — обычно завершённые треды удаляются из active list. Если видишь много TERMINATED — что-то не так с thread lifecycle management.

### Практическое понимание состояний

Для диагностики важно уметь классифицировать состояния:

- **RUNNABLE (реально работает)** — CPU-bound работа. Проверить: если код CPU-bound — нормально; если тред должен ждать что-то — что-то пошло не так (busy loop, spin).
- **RUNNABLE в native (SocketDispatcher.read0 и т.п.)** — ждёт I/O (сеть, диск). Не потребляет CPU. Нормально для waiting'а на внешние ресурсы.
- **BLOCKED** — ждёт monitor lock. Много BLOCKED на одном lock'е — узкое место в `synchronized`.
- **WAITING/TIMED_WAITING (parking)** — часто нормально: consumer'ы ждут сообщений, thread pool workers ждут задач.
- **WAITING/TIMED_WAITING на конкретных методах** (Object.wait, join) — coordination между тредами, обычно by design.

## Race condition: фундаментальная проблема

Прежде чем разбирать synchronization primitives — понять что от них защищаемся.

**Race condition** — ситуация когда результат работы программы зависит от порядка выполнения потоков. Порядок недетерминирован (решает scheduler), результат — тоже.

Классический пример: инкремент счётчика.

```java
class Counter {
    private int count = 0;
    
    public void increment() {
        count++;
    }
    
    public int get() {
        return count;
    }
}
```

Кажется что `count++` — одна операция. На самом деле три:

1. **Read**: прочитать текущее значение count из памяти в регистр CPU.
2. **Modify**: увеличить регистр на 1.
3. **Write**: записать новое значение обратно в память.

Один тред делает эти три операции подряд, никаких проблем. Два треда параллельно — сценарий проблемы:

```
Time    Thread A                    Thread B                    count в памяти
────────────────────────────────────────────────────────────────────────────
T=0     read count (0)                                          0
T=1                                 read count (0)              0
T=2     modify: 0 + 1 = 1                                       0
T=3                                 modify: 0 + 1 = 1           0
T=4     write count (1)                                         1
T=5                                 write count (1)             1
```

Оба треда прочитали 0, оба увеличили до 1, оба записали 1. Итог — 1 вместо ожидаемых 2. Один инкремент потерян.

Если 1000 тредов делают по 1000 инкрементов — ожидаем 1 000 000, реально получаем 800-950 тысяч. Каждый забег даёт разное число.

Это race condition. Решается **атомарностью** — операция должна быть неделимой, или другой тред не может влезть посередине.

**Второй пример race condition — visibility**:

```java
class Runner {
    private boolean running = true;
    
    public void run() {
        while (running) {   // читает running постоянно
            doWork();
        }
    }
    
    public void stop() {
        running = false;
    }
}

// Тред 1 в другом потоке:
runner.run();

// Тред 2:
Thread.sleep(1000);
runner.stop();   // ожидаем что тред 1 остановится
```

На большинстве JVM тред 1 **никогда** не остановится. Почему? JIT-компилятор увидел что `running` не меняется внутри цикла, «оптимизировал» — загрузил значение в регистр CPU один раз и никогда не перечитывает из памяти. Изменение `running = false` в тред 2 записало в память, но тред 1 читает регистр, не память.

Это тоже race condition — visibility между тредами не гарантирована по умолчанию. Решается через `volatile` (см. ниже).

## Java Memory Model: happens-before

Java Memory Model (JMM) — набор правил определяющих когда изменения одного потока становятся видимы другому. Основа — отношение **happens-before**.

Если действие A **happens-before** действие B, то:
1. Все изменения памяти сделанные A видны после B.
2. Порядок A и B гарантирован (A выполнится до B).

Без happens-before компилятор и CPU могут переупорядочить операции (reordering) для оптимизации. Один поток этого не заметит (обеспечивается as-if-serial semantic). Между потоками — заметит, потому что видимость и порядок теряются.

**Основные правила happens-before**:

- **Program order**: в одном треде операция A перед B в исходном коде → A happens-before B.
- **Monitor lock**: unlock happens-before последующего lock того же monitor'а.
- **Volatile**: write volatile happens-before последующего read того же volatile.
- **Thread start**: `Thread.start()` happens-before любое действие в запущенном треде.
- **Thread termination**: любое действие в треде happens-before возврат `join()` для этого треда.
- **Interruption**: `Thread.interrupt()` happens-before обнаружение прерывания в целевом треде.
- **Constructor**: завершение конструктора happens-before любое действие с объектом (после публикации).
- **Transitivity**: если A happens-before B, и B happens-before C, то A happens-before C.

Практически: чтобы одно изменение в тред 1 было видно тред 2, между ними должно быть **какое-то** happens-before отношение. Иначе JVM не обязана обеспечивать visibility.

## volatile: visibility, но не atomicity

`volatile` — модификатор поля. Гарантирует **visibility** между тредами:

- Запись в volatile → memory barrier (store-release). Значение сразу видно всем последующим чтениям в других тредах.
- Чтение volatile → memory barrier (load-acquire). Всегда читается из памяти, не из регистра/кэша CPU.

Формально: **write в volatile happens-before последующего read того же volatile**.

Пример решения visibility проблемы:

```java
class Runner {
    private volatile boolean running = true;
    
    public void run() {
        while (running) {   // каждая итерация — свежее чтение
            doWork();
        }
    }
    
    public void stop() {
        running = false;   // барьер, изменение видно всем
    }
}
```

Теперь `runner.stop()` из другого треда — гарантированно увидит тред 1 при следующей проверке цикла.

**Что volatile НЕ даёт — atomicity для составных операций**:

```java
private volatile int count = 0;

public void increment() {
    count++;   // всё ещё race condition!
}
```

`count++` это read + modify + write. Каждая из трёх операций видима другим тредам, но между ними могут вклиниться операции других тредов. Даже с volatile увидим потерю инкрементов.

Для атомарного инкремента нужен `AtomicInteger`:

```java
private final AtomicInteger count = new AtomicInteger(0);

public void increment() {
    count.incrementAndGet();   // атомарно, через CAS (Compare-And-Swap)
}
```

Или `synchronized`:

```java
private int count = 0;

public synchronized void increment() {
    count++;
}
```

**Классические use cases для volatile**:

**1. Flag pattern**. Одиночный boolean или reference для сигнализации между тредами:

```java
private volatile boolean shutdown = false;

// Producer thread:
while (!shutdown) {
    processNextItem();
}

// Main thread:
shutdown = true;
```

**2. Double-Checked Locking (DCL) с volatile**. Классический singleton паттерн:

```java
private static volatile Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {              // (1) быстрая проверка без lock
        synchronized (Singleton.class) {
            if (instance == null) {      // (2) повторная проверка под lock
                instance = new Singleton();   // (3) создание
            }
        }
    }
    return instance;
}
```

**Volatile критически важен для DCL**. Без него `new Singleton()` — не атомарная операция (allocate → constructor → assign), другой тред может увидеть частично сконструированный объект в (1). Volatile гарантирует visibility + предотвращает reorder внутри `new`.

**3. Publication of immutable data**. Готовый snapshot данных публикуется через volatile ссылку:

```java
private volatile Config config;

// Reader threads:
Config c = config;   // видят последнее значение
if (c.getFeatureEnabled()) { ... }

// Writer thread:
Config newConfig = loadConfig();
config = newConfig;   // атомарная публикация
```

Все reader'ы после publication увидят новую конфигурацию. Config сам immutable (final поля) — safe для concurrent read.

**Volatile для примитивов**. `volatile int` работает для чтения/записи всего int (это одна операция на большинстве архитектур). Но `volatile long` и `volatile double` — Java не гарантирует атомарность 64-битных операций на 32-битных архитектурах без volatile. С volatile — гарантирует.

**Volatile не для составных операций**. Всё что не read или write целиком (составные проверки, инкремент, list operations) — volatile не поможет. Нужен synchronized, ReentrantLock, atomic классы или concurrent collections.

## synchronized: intrinsic lock и mutual exclusion

`synchronized` — ключевое слово Java для взаимного исключения (mutual exclusion). Гарантирует что только один поток одновременно исполняет synchronized блок для конкретного объекта.

**Механика — intrinsic lock (monitor)**. У каждого Java-объекта есть встроенный monitor (в JVM реализован через ObjectMonitor). `synchronized` захватывает этот monitor при входе и освобождает при выходе. Другие треды пытающиеся захватить тот же monitor — в состоянии BLOCKED.

Формы синтаксиса:

**Метод synchronized**:

```java
public synchronized void method() {
    // monitor lock на this (для instance method)
}

public static synchronized void staticMethod() {
    // monitor lock на Class object
}
```

**Блок synchronized**:

```java
public void method() {
    synchronized (someObject) {
        // lock на someObject
    }
}

public void otherMethod() {
    synchronized (this) {   // эквивалентно `public synchronized void`
        // lock на this
    }
}
```

Разница блока и метода: блок позволяет выбирать **объект lock'а** явно и **уменьшить критическую секцию** до нужного минимума. Метод захватывает monitor на всё время выполнения.

**Reentrant**. Если тред уже держит monitor и снова входит в synchronized блок на том же мониторе — не блокируется (иначе был бы deadlock самого с собой). Считается «глубина» захвата, при выходе счётчик уменьшается, при 0 — реально освобождается.

```java
public synchronized void a() {
    b();   // не deadlock — тот же thread, тот же monitor
}

public synchronized void b() {
    // работает
}
```

**Memory effects**. `synchronized` даёт не только mutual exclusion, но и visibility:
- Захват monitor'а = memory barrier чтения. Все изменения видимые предыдущему держателю становятся видимы этому треду.
- Освобождение monitor'а = memory barrier записи. Все изменения этого треда становятся видимы последующим захватчикам.

Формально: unlock happens-before последующего lock того же monitor'а. Отсюда следует: изменения сделанные в одной `synchronized` секции видны в другой `synchronized` секции на том же объекте.

**Полный пример защиты составной операции**:

```java
class BankAccount {
    private double balance = 0;
    
    public synchronized void deposit(double amount) {
        balance += amount;   // race-free
    }
    
    public synchronized void withdraw(double amount) {
        if (balance >= amount) {   // read+modify+write, вся секция atomic
            balance -= amount;
        }
    }
    
    public synchronized double getBalance() {
        return balance;
    }
}
```

Все методы synchronized на `this`. В любой момент только один тред может выполнять любой из них. `withdraw` — check-then-act атомарно, никаких race conditions.

**Проблемы `synchronized`**:

- **Не interruptible**. Тред в BLOCKED не реагирует на `interrupt()`. Ждёт monitor'а бесконечно.
- **Нет timeout**. Нельзя сказать «жди monitor 5 секунд и сдавайся».
- **Только один condition**. `wait/notify` работают только с одним «условием ожидания».
- **Реалиpation внутри synchronized блокирует другие треды**. Все read'ы и write'ы через один monitor — потолок параллелизма.

Для случаев где эти ограничения важны — `java.util.concurrent.locks.ReentrantLock` (см. ниже).

## volatile vs synchronized: детальное сравнение

Часто путают. Ключевые различия:

| | volatile | synchronized |
|-|----------|--------------|
| **Что даёт** | Только visibility между тредами | Visibility + mutual exclusion |
| **Что защищает** | Одиночную read/write операцию | Составные операции (весь блок) |
| **Blocking** | Никогда не блокирует (только memory barrier) | Блокирует если lock held другим тредом |
| **Атомарность `count++`** | Нет | Да |
| **Overhead** | Очень маленький (barrier + read from memory) | Больше (acquire monitor, возможный context switch) |
| **Область применения** | Одно поле, простой flag, immutable reference | Критическая секция с несколькими операциями |
| **Reordering защита** | Да, вокруг volatile access | Да, вокруг synchronized блока |
| **Deadlock возможен** | Нет | Да |

**Когда volatile достаточно**:

- Одиночный flag: `private volatile boolean shutdown`.
- Publication immutable объекта: `private volatile Config config`.
- Single writer, multiple readers: только один тред пишет, много читают.
- DCL для singleton (с обязательным synchronized внутри).

**Когда нужен synchronized**:

- Составные операции: check-then-act, read-modify-write.
- Атомарность нескольких изменений: перевод денег между счетами.
- Несколько связанных полей должны быть согласованы: обновление сразу нескольких.
- Coordination через wait/notify.

**Пример когда volatile недостаточно**:

```java
private volatile int count = 0;

public boolean incrementIfLessThan(int max) {
    if (count < max) {   // (1) read
        count++;          // (2) read-modify-write
        return true;
    }
    return false;
}
```

Даже с volatile — race condition. Между (1) и (2) другой тред может увеличить count. Второй тред увидит count < max, но при (2) count уже увеличился до max. Итог — count > max, инвариант нарушен.

Fix через synchronized:

```java
private int count = 0;

public synchronized boolean incrementIfLessThan(int max) {
    if (count < max) {
        count++;
        return true;
    }
    return false;
}
```

Вся проверка + инкремент атомарны.

Или через atomic classes:

```java
private final AtomicInteger count = new AtomicInteger(0);

public boolean incrementIfLessThan(int max) {
    while (true) {
        int current = count.get();
        if (current >= max) return false;
        if (count.compareAndSet(current, current + 1)) {
            return true;
        }
        // CAS failed — retry
    }
}
```

CAS-based, lock-free, часто быстрее synchronized при высоком contention.

## Deadlock: механика и обнаружение

Deadlock — ситуация когда два (или больше) тредов **ждут друг друга по кругу**. Никто не может продолжить.

**Классический пример**:

```java
class Resource {
    public synchronized void useResourceA(Resource b) {
        // Использует this
        b.useResourceB();   // пытается захватить b
    }
    
    public synchronized void useResourceB() {
        // работает
    }
}

Resource r1 = new Resource();
Resource r2 = new Resource();

// Thread 1
r1.useResourceA(r2);   // захватил r1, пытается захватить r2

// Thread 2 (в тот же момент)
r2.useResourceA(r1);   // захватил r2, пытается захватить r1
```

Timeline:

```
T=0    Thread 1: захватил monitor r1
       Thread 2: захватил monitor r2
       
T=1    Thread 1: пытается захватить monitor r2 → BLOCKED (Thread 2 держит)
       Thread 2: пытается захватить monitor r1 → BLOCKED (Thread 1 держит)
       
T=∞    Оба треда BLOCKED навсегда. Deadlock.
```

**Классическая формулировка — Dining Philosophers**. Пять философов за круглым столом, между каждыми двумя — одна вилка. Чтобы есть, философ должен взять две вилки — слева и справа. Если все одновременно берут левую вилку — никто не может взять правую (её взял сосед). Все ждут вечно.

**Четыре необходимых условия deadlock** (Coffman conditions):

1. **Mutual exclusion**: ресурсы захватываются эксклюзивно. Один держатель за раз.
2. **Hold and wait**: тред держит ресурс, ждёт другой.
3. **No preemption**: ресурс нельзя отобрать, только владелец освобождает.
4. **Circular wait**: цикл ожидания среди тредов.

Если хоть одно условие нарушено — deadlock невозможен. Стратегии обхода строятся на нарушении одного из условий.

**Обнаружение deadlock в проде через thread dump**:

```bash
jstack <pid>
```

Или `jcmd <pid> Thread.print`. Java сама умеет обнаруживать deadlock и печатать:

```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f8b3c00e2f8 (object 0x00000007c0100d68, a Resource),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007f8b3c00e498 (object 0x00000007c0100d80, a Resource),
  which is held by "Thread-1"

Java stack information for the threads listed above:
===================================================
"Thread-1":
        at Resource.useResourceA(Resource.java:15)
        - waiting to lock <0x00000007c0100d80> (a Resource)
        - locked <0x00000007c0100d68> (a Resource)
        ...
"Thread-2":
        at Resource.useResourceA(Resource.java:15)
        - waiting to lock <0x00000007c0100d68> (a Resource)
        - locked <0x00000007c0100d80> (a Resource)
```

JVM сама указала: Thread-1 держит r1 и ждёт r2; Thread-2 держит r2 и ждёт r1. Кольцо, deadlock.

**Стратегии обхода deadlock**:

**1. Lock ordering (нарушить circular wait)**. Всегда захватывать ресурсы в одном и том же порядке.

```java
public void transfer(Account from, Account to, double amount) {
    // Всегда лочить в порядке возрастания id
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;
    
    synchronized (first) {
        synchronized (second) {
            from.withdraw(amount);
            to.deposit(amount);
        }
    }
}
```

Циклов быть не может — все треды лочат в одном порядке.

**2. tryLock с timeout (нарушить hold-and-wait)**. Использовать `ReentrantLock.tryLock(timeout)` вместо synchronized:

```java
ReentrantLock lockA = new ReentrantLock();
ReentrantLock lockB = new ReentrantLock();

public void transfer() {
    while (true) {
        if (lockA.tryLock(1, SECONDS)) {
            try {
                if (lockB.tryLock(1, SECONDS)) {
                    try {
                        // делаем работу
                        return;
                    } finally {
                        lockB.unlock();
                    }
                }
            } finally {
                lockA.unlock();
            }
        }
        // failed to acquire — retry с backoff
        Thread.sleep(random.nextInt(100));
    }
}
```

Если не получил lock за timeout — отпускаем всё и пробуем снова. Deadlock невозможен (никто не ждёт вечно).

**3. Single lock (нарушить mutual exclusion? нет — уменьшить гранулярность)**. Иногда проще использовать один глобальный lock вместо fine-grained locks per resource. Хуже throughput при contention, но deadlock невозможен.

**4. Lock-free структуры (нарушить mutual exclusion)**. `ConcurrentHashMap`, `AtomicInteger`, `LongAdder` — не используют locks, работают через CAS. Не могут deadlock, потому что нет locks для zwait.

## Livelock и Starvation: родственники deadlock

**Livelock** — треды активно работают, но не прогрессируют. Классика: два человека встретились в узком коридоре, оба шагают в одну сторону чтобы пропустить друг друга. Затем оба шагают в другую сторону. И так до бесконечности.

В коде — обычно из-за неудачного retry:

```java
public void transfer(Account from, Account to) {
    while (true) {
        if (from.tryLock() && to.tryLock()) {
            // do work
            return;
        }
        // failed — release, retry immediately
        from.unlock();
        to.unlock();
    }
}
```

Если два треда одновременно захотели transfer в противоположных направлениях — оба захватят один lock, не получат второй, отпустят, попробуют снова, снова conflict. CPU занят, работа не делается.

Fix — **random backoff**:

```java
if (from.tryLock()) {
    if (to.tryLock()) { /* work */ return; }
    from.unlock();
}
Thread.sleep(random.nextInt(100));   // случайная пауза
```

Каждый тред ждёт случайное время → следующая попытка не столкнётся с той же ситуацией.

**Starvation** — тред не может получить нужный ресурс потому что другие постоянно перехватывают. Классика: high-priority треды монополизируют CPU, low-priority никогда не выполняются. Или fairness проблема — `synchronized` не гарантирует FIFO порядок ожидающих, «невезучий» тред может ждать очень долго.

Fix — использовать fair locks:

```java
ReentrantLock lock = new ReentrantLock(true);   // fair mode
```

Fair lock гарантирует FIFO. Медленнее unfair (больше overhead), но никто не starves.

## Альтернативы: ReentrantLock, atomic, concurrent collections

`synchronized` — базовый примитив, но часто перегружен. Есть более гибкие альтернативы.

**ReentrantLock** — тот же mutual exclusion что synchronized, но с фичами:

```java
private final ReentrantLock lock = new ReentrantLock();

public void method() {
    lock.lock();
    try {
        // критическая секция
    } finally {
        lock.unlock();   // ОБЯЗАТЕЛЬНО в finally!
    }
}
```

Плюсы над synchronized:
- **tryLock()** с timeout — не ждать вечно.
- **lockInterruptibly()** — interruptible через `Thread.interrupt()`.
- **Fair mode** — FIFO порядок для избежания starvation.
- **Несколько Conditions** — можно ждать разные условия отдельно (не один общий `wait()`).

Минусы: verbose (нужен `try/finally`, легко забыть `unlock`), медленнее для simple cases (JVM оптимизирует `synchronized` через biased locking, thin locks — ReentrantLock всегда использует полный CAS).

Правило: `synchronized` для простых случаев, `ReentrantLock` когда нужны его фичи.

**ReadWriteLock** — оптимизация когда чтений намного больше записей:

```java
private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
private final Lock readLock = rwLock.readLock();
private final Lock writeLock = rwLock.writeLock();

public T get(K key) {
    readLock.lock();
    try {
        return map.get(key);
    } finally {
        readLock.unlock();
    }
}

public void put(K key, T value) {
    writeLock.lock();
    try {
        map.put(key, value);
    } finally {
        writeLock.unlock();
    }
}
```

Много readers могут читать параллельно (share readLock). Writer — эксклюзивно (write blocks readers). Полезно для read-heavy кэшей.

**Atomic classes** — lock-free через CAS (Compare-And-Swap):

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();      // атомарный ++
counter.compareAndSet(5, 10);   // если равно 5, установить 10 (atomic)
counter.updateAndGet(v -> v * 2);  // атомарное преобразование
```

Работают через CPU-инструкцию CAS (`LOCK CMPXCHG` на x86). Если между read и write никто не поменял значение — write проходит. Если поменял — retry.

Плюсы: lock-free (нет BLOCKED тредов), быстрее synchronized при низком contention. Минус: при высоком contention — cache-line bouncing (много CAS retry), может быть медленнее synchronized.

Классы: `AtomicInteger`, `AtomicLong`, `AtomicReference<T>`, `AtomicBoolean`. Плюс `LongAdder` / `DoubleAdder` — для случаев где нужно только инкрементировать (лучше scaling чем AtomicLong при высоком contention).

**Concurrent collections**:

- **ConcurrentHashMap** — thread-safe HashMap. Оптимизации внутри: bin-per-index, CAS для insert в empty bin, synchronized блок только на первом node bin'а (fine-grained). Read полностью lock-free (через volatile happens-before).
- **CopyOnWriteArrayList** — при каждой модификации копирует массив. Read totally free. Write очень дорогое. Для read-heavy случаев с редкими writes.
- **BlockingQueue** (`ArrayBlockingQueue`, `LinkedBlockingQueue`) — thread-safe queue с blocking `take()` и `put()`. Классика producer-consumer.
- **ConcurrentLinkedQueue** — lock-free queue. Быстрее blocking при contention, но без blocking `take()`.

Правило: если стандартная коллекция достаточно и concurrent нужен — использовать concurrent версию. Не оборачивать в `Collections.synchronizedMap(new HashMap())` (медленнее, огрубляет lock).

## Диагностика в проде

**Симптом «приложение тормозит»** — thread dump первое дело.

```bash
jstack <pid> > threads.txt
# или через jcmd (production-friendly)
jcmd <pid> Thread.print > threads.txt
```

Анализ:

**1. Считаем состояния**. Grep по `Thread.State` — сколько тредов в каждом состоянии. Много BLOCKED — есть contention на monitor. Много RUNNABLE — либо CPU-bound работа, либо busy waiting.

**2. Ищем deadlock**. Java сама пишет `Found one Java-level deadlock` если есть. Читаем — покажет кто кого держит.

**3. Ищем hot lock'и**. Много BLOCKED на одном мониторе (одинаковый `0x...` в `waiting to lock`) — этот monitor узкое место. Смотрим stacktrace держателя, оптимизируем.

**4. Ищем WAITING без явной причины**. Тред WAITING в `LockSupport.park` глубоко в коде — обычно нормально (thread pool workers, consumers). Но если тред явно должен работать (transaction handler) и WAITING — что-то не так.

**5. Long-running RUNNABLE в вычислениях**. Тред RUNNABLE в тривиальном коде — не должен занимать много времени. Если постоянно там — infinite loop или GC pressure.

**Инструменты автоматизации**:

- **fastthread.io** — загружаешь thread dump, получаешь визуализацию (группировка по состоянию, deadlock detection, топ blocked locks).
- **VisualVM** — offline анализ, поддержка sampling профилирования.
- **JFR (Java Flight Recorder)** для production: `jcmd <pid> JFR.start duration=60s filename=recording.jfr`. Событий `jdk.JavaMonitorEnter` показывают все contention events с длительностью.

**Профилактика deadlock'ов в коде**:

- Всегда захватывать locks в определённом порядке (по id, по имени класса — любой глобальный ordering).
- Не вызывать external code (callbacks, listeners) под lock — может привести к acquisition другого lock в неожиданном порядке.
- Использовать `tryLock` с timeout для критичных путей.
- Избегать nested locking когда возможно (одна операция = один lock).
- Юнит-тесты не ловят deadlock (обычно deterministic). Stress-testing с несколькими тредами более полезен.

## Заключение

**6 состояний потока** в Java: NEW (создан не запущен), RUNNABLE (выполняется или готов, включая blocking I/O), BLOCKED (ждёт synchronized monitor), WAITING (ждёт условие без timeout — Object.wait, join, LockSupport.park), TIMED_WAITING (то же с таймаутом), TERMINATED (завершился).

Ключевая ловушка: **RUNNABLE не значит «работает на CPU»**. Может быть в blocking syscall (SocketDispatcher.read0) — не потребляет CPU, ждёт I/O.

**Race condition** — результат зависит от порядка выполнения тредов. Классика: `count++` — три операции (read, modify, write), между которыми могут вклиниться другие треды.

**Java Memory Model** — правила когда изменения видимы между тредами. Основа — **happens-before**. Без happens-before между двумя операциями JVM не обязана обеспечивать visibility. Правила: program order, monitor lock (unlock happens-before lock), volatile (write happens-before read), thread start/join, transitivity.

**volatile** — visibility между тредами через memory barriers. Что даёт: одиночная read/write операция атомарна и visible. Что НЕ даёт: атомарность составных операций (`count++` всё ещё race condition). Use cases: flag pattern, DCL для singleton, publication immutable data.

**synchronized** — mutual exclusion через intrinsic lock (monitor). Reentrant (тред может входить в свой lock рекурсивно). Memory effects как у volatile (unlock happens-before lock). Формы: synchronized метод (lock на this / Class), synchronized блок (lock на любом объекте). Ограничения: не interruptible, нет timeout, один condition, блокирует другие треды.

**volatile vs synchronized**: volatile для одиночных read/write флагов / publications. synchronized для составных операций (check-then-act, read-modify-write). volatile быстрее (no blocking), synchronized мощнее (защищает всю критическую секцию).

**Deadlock** — треды ждут друг друга по кругу, никто не может продолжить. Четыре условия Coffman: mutual exclusion, hold-and-wait, no preemption, circular wait. Нарушить любое — deadlock невозможен. Обнаружение — thread dump с `Found one Java-level deadlock`. Стратегии: **lock ordering** (всегда в одном порядке — устранит circular wait), **tryLock с timeout** (устранит hold-and-wait), **single lock** (упрощение), **lock-free** (atomic, concurrent collections).

**Livelock** — треды активно работают, но не прогрессируют (взаимные retry). Fix — random backoff.

**Starvation** — тред не может получить lock из-за постоянного перехвата другими. Fix — fair locks (`new ReentrantLock(true)`).

**Альтернативы**: **ReentrantLock** (tryLock/timeout/interruptible/fair, verbose but гибче synchronized). **ReadWriteLock** для read-heavy (много readers параллельно, writer эксклюзивно). **Atomic classes** (lock-free через CAS, быстрее при низком contention). **Concurrent collections** (ConcurrentHashMap, BlockingQueue, ConcurrentLinkedQueue) — использовать вместо `synchronized wrapper`.

**Диагностика в проде** через thread dump (`jstack` / `jcmd Thread.print`): классифицировать состояния, искать deadlock, hot lock'и (много BLOCKED на одном monitor), long-running RUNNABLE (busy waiting/GC). Автоматизация — fastthread.io, VisualVM, JFR events.

**Профилактика deadlock**: consistent lock ordering, не вызывать external code под lock, tryLock с timeout для critical paths, избегать nested locking, stress-testing (юнит-тесты deadlock не ловят).

Threads overview — файл 83. Virtual threads специфика — 110. Connection pools + I/O wait — 107. Здесь была глубина по: 6 состояний потока с прод-примерами; race conditions и Java Memory Model; volatile (что даёт/не даёт); synchronized (monitor, reentrant, memory effects); детальное сравнение volatile vs synchronized; deadlock (Coffman conditions, обнаружение, обходы); livelock/starvation; альтернативы (ReentrantLock, atomic, concurrent); диагностика в проде.
