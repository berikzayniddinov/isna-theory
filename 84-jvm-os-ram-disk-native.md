# 84. JVM ↔ OS: RAM, диск, процессы, threads — глубокая теория

Файл про то как Java-приложение реально живёт в операционной системе: **что такое process** с точки зрения ядра, **как JVM использует RAM** (virtual vs physical, все области памяти), **как читает/пишет диск** (streams / NIO / mmap / page cache), **file descriptors**, **container awareness** (cgroups), **системные вызовы** за обычными Java-операциями.

Связано с: `83-java-threads-vs-virtual-threads-deep.md` (threads), `17-java-11-to-21-jvm.md` (JVM/GC), `43-java-basics-primitives-memory.md` (примитивы), `28-postgresql-internals.md` (тоже про I/O), `09-docker-detailed.md` (контейнерная изоляция).

---

## 0. Ментальная модель

```
    ┌──────────────────────────────────────────────────────────┐
    │  Java Application                                        │
    │  ┌────────────────────────────────────────────────────┐  │
    │  │  JVM (HotSpot)                                     │  │
    │  │  ┌───────────┬─────────────┬──────────┬─────────┐  │  │
    │  │  │   Heap    │  Metaspace  │Code Cache│ Threads │  │  │
    │  │  │  (young/  │  (classes)  │  (JIT)   │ (stacks)│  │  │
    │  │  │   old)    │             │          │         │  │  │
    │  │  └───────────┴─────────────┴──────────┴─────────┘  │  │
    │  │  ┌──────────────────────────────────────────────┐  │  │
    │  │  │ Direct memory (NIO ByteBuffer, off-heap)     │  │  │
    │  │  └──────────────────────────────────────────────┘  │  │
    │  └────────────────────────────────────────────────────┘  │
    │                            ↕ syscalls                    │
    └──────────────────────────────────────────────────────────┘
                                 ↕
    ┌──────────────────────────────────────────────────────────┐
    │  Linux Kernel                                            │
    │  ┌────────────┬───────────┬────────────┬─────────────┐   │
    │  │ Scheduler  │  Memory   │  VFS +     │  Network    │   │
    │  │ (CFS)      │ Manager   │  Page      │  stack      │   │
    │  │            │ (mmap,    │  Cache     │  (sockets,  │   │
    │  │            │  MMU)     │            │  epoll)     │   │
    │  └────────────┴───────────┴────────────┴─────────────┘   │
    └──────────────────────────────────────────────────────────┘
                                 ↕
    ┌──────────────────────────────────────────────────────────┐
    │  Hardware:  CPU cores  |  RAM (DIMM)  |  SSD/HDD  |  NIC │
    └──────────────────────────────────────────────────────────┘
```

Java-программист чаще всего видит только верхнюю коробочку. Senior понимает всю картину — потому что 90% production issues (OOM, GC pauses, disk I/O latency, ImagePullBackOff, читай — «работает медленно») диагностируются на границах между слоями.

---

## 1. JVM как OS-процесс

### 1.1 Что такое процесс

С точки зрения ядра — `task_struct` (в Linux). Ключевые атрибуты:
- **PID** — уникальный идентификатор.
- **Virtual address space** — 48-bit (на x86_64), теоретически 256 TB, практически меньше из-за kernel space разделения.
- **File descriptor table** — открытые файлы, sockets, pipes.
- **Signal handlers** — как реагировать на SIGTERM, SIGKILL, SIGSEGV.
- **Credentials** — UID/GID (kто запустил).
- **Working directory**.
- **Environment variables**.
- **Process group / session** — для job control.

Java-приложение = процесс запущенный `java -jar app.jar`. Один экземпляр JVM = один процесс.

### 1.2 Что происходит при `java -jar app.jar`

1. **shell fork()** — создаётся child process (клон текущего shell).
2. **execve("/usr/bin/java", ["java", "-jar", "app.jar"], envp)** — child заменяет свой memory на бинарник Java.
3. **Kernel loader** читает ELF-заголовок `java` binary, mapping'ит секции в virtual memory.
4. **glibc initialization** — стандартная библиотека C.
5. **JVM main()** — создаёт JVM instance:
   - `os::init()` — детект OS, CPU count, page size.
   - Allocate heap через `mmap()` (обычно `MAP_PRIVATE | MAP_ANONYMOUS`).
   - Init GC.
   - Init thread system (main thread = calling thread).
   - Load system classes (`java.lang.Object`, `java.lang.Class`, ...).
   - Init `jvm.dll` / `libjvm.so`.
6. **Load main class** из `-jar` / classpath.
7. **Call `main(String[])`**.

Всё это — 100-500ms для стандартной JVM. AOT-компилированные native images (GraalVM) — 20-50ms.

### 1.3 PID, PPID, PGID, SID

- **PID** — процесс.
- **PPID** — parent process (кто fork'нул). Если parent умер → PPID становится 1 (init/systemd).
- **PGID** — process group. `Ctrl+C` посылает SIGINT всей группе.
- **SID** — session. Detached processes имеют свою SID (nohup).

Внутри Docker контейнера ты видишь PID 1 (это твой процесс). На хосте у него другой PID (namespace isolation).

### 1.4 fork() и Java

**fork() дублирует весь процесс** — copy-on-write memory pages. В Java это очень дорого:
- JVM = ~100MB-2GB heap + другие области.
- Fork копирует все page tables → медленно.
- Child получает копию всех threads? Нет — только calling thread (POSIX behavior).
- **Другие threads в child висят в half-state** — deadlock likely.

Итог: **никогда не форкай Java-процесс**. `Runtime.exec()` использует `posix_spawn` (или `vfork+execve` на старых системах) — безопасно.

### 1.5 Сигналы

OS может послать процессу signal — асинхронное уведомление. JVM ловит и обрабатывает:

- **SIGTERM (15)** — «пожалуйста, завершись». JVM запускает shutdown hooks, exits gracefully. **Это то что kubelet шлёт перед убийством Pod'а.**
- **SIGKILL (9)** — «умри немедленно». Ядро убивает процесс, JVM не может ловить. Shutdown hooks не запускаются.
- **SIGINT (2)** — Ctrl+C, обычно = SIGTERM behavior.
- **SIGHUP (1)** — terminal disconnect, для daemons — часто reload config.
- **SIGSEGV (11)** — segfault, доступ к невалидной памяти. JVM крашится с hs_err_pid*.log.
- **SIGABRT (6)** — assertion failure, native crash.

Regisrer shutdown hook:
```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // cleanup on SIGTERM
    connectionPool.close();
    log.info("Graceful shutdown");
}));
```

Хуки не гарантированны при SIGKILL / OOM / power loss.

---

## 2. Виртуальная память процесса

### 2.1 Virtual vs Physical

**Физическая память** — реальные ячейки DDR RAM на плате. 32 GB на сервере.

**Виртуальная память** — каждый процесс имеет **свой** 48-bit address space (256 TB теоретически). Ядро через **MMU (Memory Management Unit)** маппирует virtual → physical. Каждый процесс думает что имеет всю память.

**Page** — единица mapping'а, обычно **4 KB** (huge pages — 2 MB / 1 GB).

**Page table** — структура в ядре, хранит mapping virtual page → physical page frame.

**TLB (Translation Lookaside Buffer)** — cache недавних translations в CPU. Full flush TLB — expensive (context switch между процессами).

### 2.2 VSS vs RSS

Ключевые метрики:

- **VSS (Virtual Set Size)** — сколько virtual памяти зарезервировано. Может быть намного больше RAM.
- **RSS (Resident Set Size)** — сколько физической памяти реально используется. То что реально в RAM.
- **PSS (Proportional Set Size)** — RSS с учётом shared памяти (делится между процессами).
- **USS (Unique Set Size)** — только приватная память процесса.

Пример: JVM с `-Xmx4g` — VSS может быть 6-8 GB (heap + metaspace + code cache + threads + native libraries). RSS — только то что **реально написано**, обычно 500MB-2GB для стартующего Boot приложения.

```bash
ps -o pid,vsz,rss,cmd -p <pid>
#   PID    VSZ    RSS CMD
# 12345 8123456 987654 java -jar app.jar
# VSZ = 8 GB virtual, RSS = 987 MB physical
```

### 2.3 Memory mapping regions

Процесс в Linux разделён на regions (смотреть `/proc/<pid>/maps`):

```
Address          Size    Perm  Backed by                Description
─────────────────────────────────────────────────────────────────
0x400000        16MB    r-xp   /usr/bin/java            Code (text)
0x1400000       2MB     rw-p   /usr/bin/java            Data/BSS
0x2000000       ...     rw-p   [heap]                   glibc heap (malloc)
0x7f0000000000  4GB     rw-p   [anon]                   JVM Java heap
0x7f0400000000  256MB   rw-p   [anon]                   Metaspace
0x7f0500000000  240MB   r-xp   /libjvm.so               JVM code
0x7f0600000000  1MB     rw-p   [anon]                   Thread stack 1
0x7f0600100000  1MB     rw-p   [anon]                   Thread stack 2
...
0x7fffffffe000  8KB     rw-p   [stack]                  Main thread stack
0xfffffffff600  4KB     r-xp   [vdso]                   Virtual DSO
```

`[anon]` = anonymous mapping (не привязан к файлу), обычно через `mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)`.

### 2.4 Overcommit

Linux по умолчанию — **overcommit memory**. Разрешает `mmap` больше памяти чем есть RAM+swap. Реальное выделение — при **first write** (write triggers page fault → kernel allocates physical page).

Плюс: приложения могут резервировать много (JVM `-Xmx8g` не значит 8 GB физически сразу).

Минус: если все процессы вдруг захотят реально использовать резерв — **OOM killer** просыпается, убивает процесс с наибольшим `oom_score`.

Настройки:
```
/proc/sys/vm/overcommit_memory
  0: heuristic (default)
  1: always overcommit
  2: never overcommit (strict accounting)
/proc/sys/vm/overcommit_ratio  # для mode 2
```

В контейнерах — cgroup memory limit важнее чем OS-level overcommit.

---

## 3. JVM memory layout — все области

JVM использует **несколько отдельных memory regions**, не только «heap». Понимать все — необходимо для troubleshooting OOM.

### 3.1 Java Heap

Основное место для объектов созданных `new`. Управляется GC.

Разделён на:
- **Young generation** — новые объекты, быстрый GC (minor GC).
  - **Eden** — куда попадают новые объекты.
  - **Survivor S0, S1** — после первого GC.
- **Old generation** (tenured) — долгоживущие объекты после нескольких GC.
- **Humongous** (G1) — большие объекты > half of region size.

Настройка:
```
-Xms2g       # начальный размер heap
-Xmx4g       # максимальный размер
-XX:NewRatio=2   # young:old ratio
-XX:SurvivorRatio=8  # eden:survivor ratio
```

**Внутри HotSpot реализация**:
- G1 GC (default с JDK 9) — regions по 1-32 MB.
- ZGC — colored pointers, sub-ms pauses.
- Shenandoah — concurrent compaction.
- Parallel GC — throughput-oriented.

### 3.2 Metaspace (было PermGen до Java 8)

Хранит **class metadata**: класс structure, method info, constant pool, bytecode.

Не в heap! Хранится в **native memory** (mmap-based).

Настройка:
```
-XX:MetaspaceSize=128m           # initial
-XX:MaxMetaspaceSize=512m        # limit (default unlimited!)
```

**Grows automatically** до `MaxMetaspaceSize` (или до OOM если не задан).

**OOM: Metaspace** — обычно классы утекают: dynamic classloading без освобождения (Groovy scripting, JSP recompilation, старый Tomcat undeployment).

Проверить:
```bash
jcmd <pid> VM.native_memory summary  # включена NMT
# ищи "Class - reserved / committed"
```

### 3.3 Code Cache

JIT-скомпилированный native code (byteocode → x86 machine code через C1/C2 compilers).

```
-XX:InitialCodeCacheSize=64m
-XX:ReservedCodeCacheSize=240m    # default для JDK 8+
```

Если заполнен — JIT stops compiling → интерпретация → **производительность падает**. JVM warning в логах:
```
CodeCache is full. Compiler has been disabled.
```

Fix: увеличить `ReservedCodeCacheSize` или включить flushing (`-XX:+UseCodeCacheFlushing`).

С JDK 9+ разделён на 3 сегмента (non-nmethod, profiled, non-profiled).

### 3.4 Thread stacks

Каждый platform thread — свой stack, обычно **1 MB** (`-Xss1m`).

Native memory, вне heap. Не управляется GC.

Виртуальные threads — stack chunks в heap (см. `83-java-threads-vs-virtual-threads-deep.md`).

### 3.5 Direct memory (off-heap)

`ByteBuffer.allocateDirect(size)` — выделяет память **вне Java heap**. Используется NIO, Netty, PostgreSQL JDBC driver (bulk operations), Kafka clients.

Не управляется GC напрямую! Освобождается когда `DirectByteBuffer` объект собран (через `Cleaner`) — может быть с задержкой.

Настройка:
```
-XX:MaxDirectMemorySize=1g       # default = Xmx
```

Тricky: если `MaxDirectMemorySize` не задан явно, default = **`Runtime.maxMemory()`** (то есть Xmx). При активном NIO — может удвоить memory usage.

OOM: `java.lang.OutOfMemoryError: Direct buffer memory`.

### 3.6 Native memory (JNI, mmap)

Что использует native memory:
- JVM internal (GC bookkeeping, JIT compiler working sets).
- Native libraries via JNI (например `Jansi` для colored output, `netty-tcnative` для SSL, JNA).
- `MappedByteBuffer` (mmap файлы).
- ZIP/JAR read (кэши в native).

Не покрывается `-Xmx`. Смотреть через **NMT (Native Memory Tracking)**.

Включить:
```
-XX:NativeMemoryTracking=detail
```

Смотреть:
```bash
jcmd <pid> VM.native_memory summary
# Total: reserved=6GB, committed=1.2GB
# - Java Heap (reserved=4GB, committed=800MB)
# - Class (reserved=1GB, committed=100MB)
# - Thread (reserved=128MB, committed=128MB)
# - Code (reserved=240MB, committed=45MB)
# - GC (reserved=200MB, committed=150MB)
# - Compiler (reserved=100MB, committed=20MB)
# - Internal (reserved=30MB, committed=30MB)
# - Symbol (reserved=20MB, committed=20MB)
# ...
```

### 3.7 Ключевое: реальный memory footprint

```
Total RSS = Heap (used) 
          + Metaspace 
          + Code Cache 
          + Thread stacks (Xss × threads)
          + Direct memory
          + Native (GC, JIT, libs, mmap)
          + Overhead (~5-10%)
```

Пример реального Spring Boot приложения:
- `-Xmx1g` heap → **used** ~600 MB.
- Metaspace ~200 MB.
- Code cache ~80 MB.
- 200 threads × 1MB = 200 MB stacks.
- Direct memory ~100 MB (Netty).
- Native ~150 MB.
- **Total RSS ~1.3 GB**.

Container с memory limit 1 GB → **OOMKilled** после старта. Правило JVM в контейнере: `Xmx = ~70-75% container limit`.

### 3.8 -XX:MaxRAMPercentage

Более гибкое чем `-Xmx`:
```
-XX:MaxRAMPercentage=75.0
-XX:InitialRAMPercentage=50.0
-XX:MinRAMPercentage=50.0
```

JVM читает cgroup limit и вычисляет heap как %. Полезно в контейнерах — не надо хардкодить Xmx.

---

## 4. Container awareness (cgroups)

### 4.1 Cgroups memory

Linux **cgroups v2** ограничивает ресурсы процесса. Для контейнеров Kubernetes использует cgroups через kubelet + containerd.

Файлы в `/sys/fs/cgroup/`:
- `memory.max` — hard limit. При превышении → **OOM killer** внутри cgroup.
- `memory.high` — soft limit. Throttling + reclaim, но не убийство.
- `memory.current` — сколько используется сейчас.
- `memory.events` — count of oom, high events.

**OOMKilled в K8s** — cgroup OOM убил процесс. Exit code 137 (128 + 9 SIGKILL). Причина в контейнере — memory.max exceeded.

### 4.2 JVM container awareness

JDK 10+ (и backport в 8u191) — **автоматически детектирует cgroup limits**:
- `Runtime.getRuntime().availableProcessors()` — cgroup CPU quota, не всё ядро хоста.
- `Runtime.getRuntime().maxMemory()` — cgroup memory limit, не хостовая RAM.

Настройки (обычно включены by default):
```
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=75.0
```

**Без UseContainerSupport (старый JDK 8)**:
- JVM видит всю хостовую память (например 128 GB) вместо cgroup 1 GB.
- Ставит heap ~30% = 30 GB.
- **Мгновенный OOMKilled**.

Fix: обновить JDK или явно `-Xmx800m`.

### 4.3 CPU limits и JVM

`limits.cpu: 2` в K8s → cgroup CFS quota: за каждые 100ms позволено 200ms CPU time (2 core equivalent).

**Throttling** — если превысил, приложение замедляется (не убивают). Для JVM это болезненно:
- GC threads throttled → длинные GC pauses.
- JIT compilation stalls.
- Обычный код тормозит.

Метрика в `cadvisor`/Prometheus: `container_cpu_cfs_throttled_periods_total`.

**Recommendation**: не ставь CPU limit для Java-приложений вообще. Только `requests` (гарантия). Пусть JVM использует свободные CPU когда есть.

### 4.4 Проверка cgroup изнутри контейнера

```bash
# В контейнере:
cat /sys/fs/cgroup/memory.max
# 1073741824 = 1 GB

cat /sys/fs/cgroup/cpu.max
# 200000 100000 = 2 cores

cat /sys/fs/cgroup/memory.current
# 543826432 = ~518 MB used
```

Java view:
```java
Runtime rt = Runtime.getRuntime();
System.out.println("max=" + rt.maxMemory());      // ~700 MB (75% of cgroup)
System.out.println("cpus=" + rt.availableProcessors()); // 2
```

---

## 5. Disk I/O в Java

### 5.1 Слои

```
Java code:  FileInputStream.read(buf)
      ↓
JDK:        читает через FileChannel / RandomAccessFile
      ↓
JVM:        syscall read(fd, buf, size)
      ↓
Kernel:     VFS layer (POSIX API)
      ↓
Filesystem: ext4 / xfs / btrfs — определяет layout на диске
      ↓
Block layer: планирование I/O, elevator schedulers
      ↓
Device driver: NVMe/SATA
      ↓
Hardware: SSD/HDD
```

Каждый слой добавляет latency, но все важны.

### 5.2 Streams (java.io)

Классический blocking API:
```java
try (FileInputStream fis = new FileInputStream("file.txt");
     BufferedInputStream bis = new BufferedInputStream(fis)) {
    int b;
    while ((b = bis.read()) != -1) {
        process(b);
    }
}
```

Что происходит:
- `FileInputStream` — открывает file descriptor через syscall `open()`.
- `.read()` — syscall `read(fd, buf, 1)`. Медленно если делать по 1 байту.
- `BufferedInputStream` — читает **чанком** (default 8 KB) в internal buffer, потом отдаёт побайтно. Один syscall на 8K bytes.

**Правило**: всегда оборачивай FileInputStream в BufferedInputStream. 100× быстрее.

### 5.3 NIO Channels (java.nio)

Более современный API, работает через ByteBuffer:
```java
try (FileChannel channel = FileChannel.open(Paths.get("file.txt"))) {
    ByteBuffer buf = ByteBuffer.allocate(8192);
    while (channel.read(buf) > 0) {
        buf.flip();
        process(buf);
        buf.clear();
    }
}
```

Плюсы над streams:
- Explicit buffer management.
- Direct buffers (off-heap) — избегают копирования JVM heap ↔ kernel buffer.
- Scatter/gather (одним syscall читаем в несколько buffer'ов).
- Non-blocking mode для sockets (не для файлов до io_uring).

### 5.4 Memory-mapped files (mmap)

```java
try (FileChannel channel = FileChannel.open(Paths.get("big.dat"), READ)) {
    MappedByteBuffer buf = channel.map(READ_ONLY, 0, channel.size());
    // buf теперь как обычный byte array, но backed by disk
    byte b = buf.get(1_000_000);   // не syscall, page fault → kernel загружает страницу
}
```

Что происходит:
- Kernel создаёт mapping virtual page → file offset.
- Первое обращение к странице → **page fault** → kernel читает 4KB из файла → возвращает.
- Последующие обращения — прямое чтение из RAM (страница в page cache).

**Плюсы**:
- Нет копирования kernel → user buffer.
- Random access через pointer arithmetic.
- OS сам управляет caching (page cache).

**Минусы**:
- Address space limited (2GB per mapping на 32-bit).
- Изменения persisted через `msync()` или на close.
- Не подходит для sequential streaming (просто оверхед).

**Использование**: базы данных (Kafka Log Segments используют mmap для fast reads), большие индексы.

### 5.5 Page cache

**Ключевая оптимизация Linux**. Все чтения/записи файлов проходят через **page cache** в RAM.

- Первое чтение файла → syscall → диск → страница в page cache → отдаётся приложению.
- Следующее чтение того же файла → syscall → **cache hit** → не идёт до диска → возвращает.

Смотреть:
```bash
free -h
#              total     used   free   shared  buff/cache  available
# Mem:          32Gi     8Gi   4Gi    100Mi   20Gi        23Gi
```

`buff/cache` — page cache + buffer cache. Ядро автоматически освобождает при memory pressure.

**Кажется что памяти мало** (`free` 4GB), но `available` учитывает page cache — реально доступно 23 GB.

**Влияние на Java**:
- Читаешь тот же файл дважды → второе чтение быстрое (в RAM).
- Kafka reads — очень fast если данные в page cache (обычно недавно записанные).
- **Restart JVM не теряет page cache** — новый процесс тут же читает быстро (важно для K8s pod restart).

### 5.6 Write и fsync

`fileChannel.write(buf)` — кладёт данные в **page cache**, не на диск!

Kernel периодически (обычно 30 сек, `/proc/sys/vm/dirty_expire_centisecs`) сбрасывает dirty pages на диск.

Если сервер упал между write и flush → **данные потеряны**.

**fsync** — гарантирует запись:
```java
fileChannel.force(true);   // fsync — данные + metadata на диске
fileChannel.force(false);  // fdatasync — только данные (без metadata)
```

fsync медленный (~1-10 ms на HDD, ~50-500 μs на NVMe SSD). Нужен для:
- Databases (WAL).
- Message brokers (durable messages).
- Не нужен для temp/cache файлов.

### 5.7 Direct I/O (bypass page cache)

`O_DIRECT` flag — обходит page cache, читает/пишет напрямую с диска.

- Полезно для databases со своим buffer manager (PostgreSQL, Oracle).
- Java до JDK 10 не поддерживал! JDK 10+ — `ExtendedOpenOption.DIRECT`.
- Обычно **не нужно** для application code — page cache ускоряет всё.

### 5.8 io_uring

Новая async I/O infrastructure Linux (kernel 5.1+). Заменяет старый POSIX AIO.

Ring buffers для submission и completion, share'ятся между user space и kernel. **Zero syscall overhead** для batch I/O.

- Java пока не имеет прямой поддержки, но Netty 4.1.x+ имеет `IOUringEventLoopGroup`.
- Loom (Java 21) внутри может использовать io_uring для virtual thread I/O.
- Обычно application-level не трогает.

### 5.9 File descriptors — лимиты

Каждый открытый файл/socket/pipe — file descriptor (integer). У процесса есть лимит:

```bash
ulimit -n            # soft limit
ulimit -Hn           # hard limit
cat /proc/<pid>/limits | grep "open files"
```

Default обычно 1024 (пугающе низкий для сервера). Обычно повышают до 65536+.

```bash
ls /proc/<pid>/fd/ | wc -l   # сколько FDs используется
```

**Too many open files** exception — забыл закрыть stream / socket / connection в pool. Try-with-resources spolves problem.

В K8s — задаётся через securityContext или через init container:
```yaml
securityContext:
  sysctls:
  - name: fs.file-max
    value: "1000000"
```

---

## 6. Network I/O

### 6.1 Blocking sockets (java.net)

Классический API:
```java
try (Socket socket = new Socket("host", 80);
     OutputStream out = socket.getOutputStream();
     InputStream in = socket.getInputStream()) {
    out.write(request);
    int b = in.read();  // блокирует thread пока данные не придут
}
```

Один thread на соединение. При 10000 concurrent — 10000 threads. Не масштабируется.

### 6.2 NIO Selectors (java.nio)

Non-blocking:
```java
Selector selector = Selector.open();
ServerSocketChannel server = ServerSocketChannel.open();
server.bind(new InetSocketAddress(8080));
server.configureBlocking(false);
server.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();   // блокирует пока хоть один канал не готов
    for (SelectionKey key : selector.selectedKeys()) {
        if (key.isAcceptable()) { /* new connection */ }
        if (key.isReadable()) { /* data available */ }
        if (key.isWritable()) { /* can write */ }
    }
}
```

Внутри — syscall `epoll_wait()` на Linux (или `kqueue` на macOS). Один thread обрабатывает тысячи sockets.

Основа для Netty, Undertow, Tomcat NIO connector, gRPC.

### 6.3 epoll — Linux механика

Три syscalls:
- `epoll_create1()` — создать epoll instance (FD).
- `epoll_ctl(epfd, ADD/MOD/DEL, fd, event)` — управлять watched FDs.
- `epoll_wait(epfd, events, maxevents, timeout)` — блокировать пока не появится событие.

Эффективно даже для 100k sockets (O(1) на event vs O(n) в старом `select()`).

**Edge-triggered vs Level-triggered**:
- Level-triggered (default) — событие пока условие есть (много wake-up'ов).
- Edge-triggered — только на переход состояния (один wake-up per event, эффективнее но сложнее).

### 6.4 Netty

Framework для NIO. Каждый **EventLoopGroup** имеет несколько threads, каждый thread — свой Selector с thousands of channels.

Producer сервисы в КНП (`isnaknpgateway`) — часто на Netty.

### 6.5 Virtual threads + network

Java 21 — network I/O в JDK **автоматически yield'ит virtual thread** при block. Внутри — NIO Selector shared for all VT.

Результат: пишешь blocking код (`socket.read()`), JVM использует epoll underneath. Императивный API, реактивная производительность.

---

## 7. System calls — что стоит за обычными Java-операциями

### 7.1 Strace на Java

```bash
strace -f -p <pid>            # trace всех syscalls
strace -f -e trace=network java MyApp     # только network
strace -c -p <pid>            # count summary
```

Пример: `System.out.println("hello")`:
```
write(1, "hello\n", 6) = 6
```

Один syscall на печать. Просто.

Пример: `Files.write(path, bytes)`:
```
openat(AT_FDCWD, "/tmp/f", O_WRONLY|O_CREAT|O_TRUNC, 0644) = 5
write(5, "hello", 5) = 5
close(5) = 0
```

Три syscall'а. Каждый ~1-10 μs.

### 7.2 Network syscall trace

`new URL("https://api.com").openStream()`:
```
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 6
connect(6, {sin_family=AF_INET, sin_port=htons(443), sin_addr=inet_addr("...")}, 16) = 0
(TLS handshake — многочисленные read/write)
write(6, "GET / HTTP/1.1\r\n...", 76) = 76
read(6, "HTTP/1.1 200 OK\r\n...", 8192) = 1024
close(6) = 0
```

Каждый HTTP request — минимум 4-5 syscalls (connect, write, N reads, close).

### 7.3 GC syscalls

G1 GC при collection:
- `mmap()` — если нужно расширить heap.
- `madvise(MADV_DONTNEED)` — вернуть страницы ядру (после uncommit).
- `mprotect()` — защита memory regions.

Обычно скрыто от программиста.

### 7.4 Thread create

`Thread.start()`:
```
clone(child_stack=..., flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|...) = 12345
```

Один syscall создаёт OS thread. Дальше mutex init, TLS init.

---

## 8. Наблюдаемость: как всё это мониторить

### 8.1 OS-level tools

```bash
# Процесс info
ps aux | grep java
top -H -p <pid>                    # threads внутри процесса

# Memory
free -h                            # общая
cat /proc/<pid>/status | grep Vm   # VmSize, VmRSS, VmPeak
cat /proc/<pid>/smaps              # detailed по регионам

# I/O
iotop                              # диск активность
iostat -x 1                        # утилизация SSD/HDD
pidstat -d 1 -p <pid>              # per-process I/O

# Network
ss -tnp | grep java                # sockets Java-процесса
iftop                              # bandwidth
tcpdump -i any port 8080           # low-level packets

# System calls
strace -f -p <pid> -e trace=file
strace -c -p <pid>                 # count по типу syscall

# CPU
mpstat -P ALL 1
perf top -p <pid>                  # CPU flame per function
```

### 8.2 JVM tools

```bash
# Base info
java -version
jps -lv                            # список JVM процессов
jinfo <pid>                        # arguments, sys props

# Memory
jstat -gc <pid> 1000               # GC stats каждую секунду
jstat -gccapacity <pid>            # region sizes
jcmd <pid> GC.heap_info            # summary

# Native memory (нужен -XX:NativeMemoryTracking=detail)
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.native_memory detail

# Heap dump
jcmd <pid> GC.heap_dump /tmp/heap.hprof
jmap -dump:live,format=b,file=/tmp/heap.hprof <pid>

# Thread dump
jcmd <pid> Thread.print
jstack <pid>
jcmd <pid> Thread.dump_to_file -format=json /tmp/threads.json  # Java 21+

# Классы
jcmd <pid> VM.class_hierarchy
jmap -clstats <pid>

# JIT
jcmd <pid> Compiler.codelist       # compiled methods

# Flight Recorder
jcmd <pid> JFR.start duration=60s filename=/tmp/rec.jfr
jcmd <pid> JFR.stop name=1

# Force GC
jcmd <pid> GC.run                  # System.gc()
```

### 8.3 Prometheus metrics

Через Micrometer в Spring Boot:
```
jvm.memory.used{area="heap",id="G1 Eden Space"}
jvm.memory.used{area="nonheap",id="Metaspace"}
jvm.gc.pause                       # GC pause time histogram
jvm.threads.live                   # live threads
jvm.threads.daemon
jvm.classes.loaded
jvm.buffer.memory.used{id="direct"}
process.cpu.usage                  # cgroup-aware
process.files.open
process.uptime
system.load.average.1m
```

Grafana dashboard "JVM (Micrometer)" — standard visualization.

### 8.4 Node exporter (для K8s)

`node-exporter` DaemonSet на каждой ноде экспортирует OS metrics:
- CPU, memory, disk, network per host.
- Container-level (через cAdvisor встроен в kubelet):
  - `container_memory_working_set_bytes`
  - `container_cpu_cfs_throttled_periods_total`
  - `container_fs_reads_bytes_total`
  - `container_network_receive_bytes_total`

---

## 9. Практические сценарии диагностики

### 9.1 Pod OOMKilled

Симптомы: `kubectl describe pod` → `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`.

Диагностика:
1. **Проверить memory limit vs Xmx**: если `limits.memory: 1Gi`, а `-Xmx1g` — плохо. Native memory + threads + metaspace добавят 200-500 MB → RSS 1.2-1.5 GB → OOM.
2. **Fix**: `-Xmx700m` (или `-XX:MaxRAMPercentage=70`) при `limits.memory: 1Gi`.
3. **Или**: увеличить memory limit.

Более глубоко:
- **Real memory leak?** — если Xmx правильный, но RSS растёт со временем.
  - Heap dump: `jcmd <pid> GC.heap_dump`. Открыть в Eclipse MAT. Найти doms tree.
  - Native leak: `-XX:NativeMemoryTracking=detail` + `jcmd VM.native_memory baseline/diff`.
- **Direct memory leak?** — `jcmd VM.native_memory` в разделе "Other" растёт. Часто Netty без `.release()`.
- **Class loader leak?** — Metaspace growing. `jmap -clstats` покажет.

### 9.2 High CPU

```bash
top -H -p <pid>                    # thread'ы по CPU
# найти nid (native id) hot thread'а
printf '%x\n' <nid>                # в hex
jstack <pid> | grep 0x<hex_nid>    # найти в stack trace
```

Показывает what code съедает CPU. Часто:
- Infinite loop.
- GC hyperactive → increase heap or tune.
- JIT compilation (первые минуты после старта).
- Regex catastrophic backtracking.

Async Profiler для flame graph:
```bash
./profiler.sh -e cpu -d 30 -f flame.html <pid>
```

### 9.3 High disk I/O

```bash
iotop -p <pid>            # что процесс делает с диском
pidstat -d 1 -p <pid>     # read/write bytes per second
```

Причины:
- **Logging** — избыточно, синхронно. Async appender с buffer.
- **GC** — если swap, dying (обычно нет swap в K8s).
- **Application** — bulk imports, cache warmup.
- **Page cache miss** — файлы не в RAM.

### 9.4 High network

```bash
ss -tnp | grep <pid>                # open sockets
iftop -f "host <pod-ip>"            # bandwidth
tcpdump -i eth0 -w capture.pcap port 8080
```

Причины:
- Chatty microservice (много мелких calls вместо batch).
- Connection pool exhaustion → создание новых.
- gRPC/HTTP keep-alive off.

### 9.5 Slow startup

Spring Boot стартует 30 сек — можно ускорить:
- **Lazy initialization**: `spring.main.lazy-initialization: true`.
- **Class Data Sharing (CDS)**: JDK 13+ auto, ускоряет class loading.
- **AOT / GraalVM Native Image**: старт 20-50 ms.
- **Spring AOT** (3.0+): ahead-of-time processing beans.

---

## 10. Собесные вопросы

### Q1: Разница VSS и RSS?

**VSS (Virtual Set Size)** — сколько virtual памяти зарезервировано процессом. Может быть значительно больше физической RAM (через mmap с overcommit).

**RSS (Resident Set Size)** — сколько **физической** памяти реально используется (страницы, которые есть в RAM).

Пример: JVM `-Xmx4g` может иметь VSS 6-8 GB (heap reserve + metaspace + code cache + native), но RSS всего 500 MB (реально used).

### Q2: Что означает `Exit Code 137`?

`137 = 128 + 9`. `128` — базовое смещение для signals. `9` — SIGKILL.

В K8s это обычно **OOMKilled** — cgroup memory limit exceeded, Linux OOM killer убил процесс. JVM не может обработать SIGKILL, shutdown hooks не запускаются.

Проверить: `kubectl describe pod` → `Last State: Terminated, Reason: OOMKilled`.

### Q3: Почему `-Xmx` не равен RSS?

RSS = Heap (used, не reserved!) + Metaspace + Code Cache + Thread stacks (Xss × N) + Direct memory + Native memory + overhead.

`-Xmx4g` — только heap **maximum**. Real memory usage больше на 30-70%.

Правило в контейнерах: `Xmx ≈ 70% container memory limit`, или `-XX:MaxRAMPercentage=70`.

### Q4: Что такое overcommit?

Linux позволяет `mmap` выделить больше virtual памяти, чем есть физической RAM + swap. Реальная аллокация — при **first write** (page fault).

Плюс: приложения могут резервировать много без immediate cost.
Минус: если все резервы вдруг понадобятся — **OOM killer** просыпается.

Настройка: `/proc/sys/vm/overcommit_memory` (0/1/2).

В K8s важнее cgroup memory limit — он enforced'ится независимо от OS overcommit.

### Q5: Что такое page cache и как влияет на Java?

**Page cache** — Linux keeps recently read/written file pages in RAM. Все file I/O проходят через него.

- Первое чтение файла → syscall → диск → page cache → returned.
- Следующее чтение того же файла → cache hit → быстро.

**Влияние на Java**:
- Kafka, PostgreSQL используют page cache для fast reads.
- `free -h` показывает `buff/cache` — это не «занятая» память, ядро freed её при memory pressure.
- **JVM restart не теряет page cache** — новый процесс сразу читает быстро.

### Q6: Что такое `fsync` и когда он нужен?

`write()` кладёт данные в page cache — не на диск! Ядро асинхронно сбрасывает грязные страницы (30 сек по умолчанию).

При crash в этот интервал — данные потеряны.

**fsync** — syscall который гарантирует запись на диск. `fileChannel.force(true)` в Java.

Медленный (1-10 ms HDD, 50-500 μs NVMe). Нужен для:
- Database WAL (write-ahead log).
- Kafka commit.
- Bank transactions.

Не нужен: temp files, cache files, logs (обычно).

### Q7: Как JVM детектирует container limits?

С JDK 10+ (backport 8u191) — flag `-XX:+UseContainerSupport` (default enabled).

JVM читает cgroup files:
- `/sys/fs/cgroup/memory.max` (cgroups v2) — memory limit.
- `/sys/fs/cgroup/cpu.max` — CPU quota.

Автоматически применяются к:
- `Runtime.availableProcessors()` — cgroup CPU, не хостовое ядро.
- `Runtime.maxMemory()` — если `-XX:MaxRAMPercentage`, вычисляется от cgroup.

Без UseContainerSupport (старый JDK 8) — JVM видит хост, ставит big heap → OOMKilled.

### Q8: Разница `-Xmx`, `-Xms` и `-XX:MaxRAMPercentage`?

- `-Xms` — начальный размер heap. JVM аллоцирует сразу.
- `-Xmx` — максимальный размер heap. JVM может расти до этого.
- `-XX:MaxRAMPercentage=75.0` — max heap как процент от **cgroup / hosted RAM**.

`-Xmx` — фиксированный. `MaxRAMPercentage` — гибкий, полезен в контейнерах, где size меняется.

Recommend: `-Xms == -Xmx` в prod (avoid heap resize pauses). Или использовать `MinRAMPercentage == MaxRAMPercentage`.

### Q9: Что такое direct memory и зачем оно?

`ByteBuffer.allocateDirect(size)` — память **вне Java heap**, allocated через `malloc` (или `mmap`).

Используется NIO, Netty, database drivers для bulk I/O.

**Плюс**: избегает копирования `heap ByteBuffer → kernel buffer` при I/O syscall.
**Минус**: не под управлением GC напрямую (освобождается через `Cleaner` с задержкой).

Настройка: `-XX:MaxDirectMemorySize=1g`. Default = `Runtime.maxMemory()` = Xmx (легко удвоить usage).

OOM: `java.lang.OutOfMemoryError: Direct buffer memory`.

### Q10: Что делает JVM при `Files.readAllBytes(path)`?

Псевдо-упрощённо:
1. Open syscall: `openat(AT_FDCWD, "path", O_RDONLY)` → returns fd.
2. Fstat syscall: `fstat(fd, &statbuf)` → получить size.
3. Allocate byte[size] в heap.
4. Read syscall: `read(fd, buffer, size)` → возвращает bytes.
5. Close syscall: `close(fd)`.

4 syscalls. Каждый ~1-10 μs. Плюс fsync НЕ вызывается (только чтение).

Для больших файлов — page cache задействован: если файл был читан недавно, все данные в RAM, read → memcpy.

### Q11: Что такое mmap и когда полезно?

`FileChannel.map(READ_ONLY, offset, size)` — маппит файл в virtual memory. Обращения к memory — kernel загружает страницы файла по требованию (page fault).

Плюсы:
- Random access через pointer arithmetic (не seek+read).
- Нет копирования kernel → user buffer.
- OS auto-manages caching (page cache).

Использования: Kafka log segments (fast read + writes), Lucene indexes, database mmap engines (SQLite optionally, MongoDB).

Не для streaming (sequential read просто overhead).

### Q12: Что происходит с thread stack памятью?

Каждый platform thread — свой stack size `-Xss` (default 1 MB).

**Virtual reservation**: 1 MB зарезервировано в virtual address space.
**Physical allocation**: страницы (обычно 4 KB) выделяются по мере глубины (on-demand через page fault).

Guard page снизу стека — SIGSEGV → JVM converts to `StackOverflowError`.

Virtual threads — стек не 1 MB fixed, а stack chunks в heap. Только used memory.

### Q13: Как понять сколько файлов открыто у Java-процесса?

```bash
ls /proc/<pid>/fd/ | wc -l          # total FDs
ls -l /proc/<pid>/fd/ | head        # что открыто (files, sockets, pipes)
lsof -p <pid>                       # detailed list
```

Проверить limit:
```bash
cat /proc/<pid>/limits | grep "open files"
```

`Too many open files` exception — либо повысить `ulimit -n`, либо (более важно) исправить утечку (закрывать streams / connections через try-with-resources).

### Q14: Что такое zero-copy и как использовать в Java?

**Zero-copy** — передача данных с disk → network без копирования в user space.

Обычный путь file → socket:
```
disk → kernel page cache → user buffer → kernel socket buffer → NIC
                       (copy)        (copy)
```

**Zero-copy**: kernel копирует напрямую из page cache → socket buffer (даже с `sendfile`).

В Java: `FileChannel.transferTo(target)` использует `sendfile()` syscall.
```java
fileChannel.transferTo(0, size, socketChannel);
```

Используется в Kafka producer/consumer, Netty static file serving, Tomcat sendfile.

Экономит CPU и memory bandwidth на high-throughput data transfer.

### Q15: Разница blocking vs non-blocking I/O?

**Blocking** (`Socket.read()`): thread ждёт пока данные придут. Пока ждёт — не может делать ничего другого. Простая модель, но требует thread per connection.

**Non-blocking** (NIO Selector): один thread обрабатывает много sockets через `epoll_wait()`. Возвращает immediately с "no data yet" или available events. Масштабирует до 100k+ connections.

Java 21 virtual threads — **blocking API снаружи, non-blocking внутри**. JDK автоматически yield'ит VT continuation при I/O block, использует epoll underneath. Комбинирует простоту blocking с масштабируемостью non-blocking.

### Q16: Что делать при OutOfMemoryError: Metaspace?

Metaspace хранит class metadata. Растёт при loading новых классов.

Причины:
- **Dynamic class generation** без освобождения: Groovy scripting, JSP recompilation.
- **Multiple classloaders** (Tomcat undeploy/redeploy leak).
- **Reflection-heavy libraries** (некоторые ORMs генерируют proxy classes).

Диагностика:
```bash
jmap -clstats <pid>                # class loaders + count
jcmd <pid> GC.class_stats          # class breakdown
jcmd <pid> VM.classloaders         # tree of classloaders
```

Fix:
- `-XX:MaxMetaspaceSize=512m` — жёстко ограничить (лучше OOM чем неопределённый рост).
- Найти классы которые не освобождаются (`jmap -clstats` → искать неожиданное количество).
- Обновить библиотеки (leak-fixes).

### Q17: Как правильно настроить JVM в K8s Pod'е?

```yaml
resources:
  requests: {memory: 2Gi, cpu: 500m}
  limits:   {memory: 2Gi, cpu: 2000m}   # или без CPU limit для Java
env:
- name: JAVA_TOOL_OPTIONS
  value: >-
    -XX:MaxRAMPercentage=70
    -XX:+UseG1GC
    -XX:+HeapDumpOnOutOfMemoryError
    -XX:HeapDumpPath=/tmp/heap.hprof
    -Djdk.tracePinnedThreads=short
    -XX:+ExitOnOutOfMemoryError
```

**Ключевое**:
1. `Xmx ≈ 70% limits.memory` (место для native memory).
2. `requests == limits` для memory — Guaranteed QoS, меньше вероятность eviction.
3. Не ставь CPU limit — GC threads не должны throttling'ать.
4. `-XX:+HeapDumpOnOutOfMemoryError` — автоматический heap dump при OOM.
5. `-XX:+ExitOnOutOfMemoryError` — не пытаться жить после OOM (kubelet рестартит Pod).

### Q18: Как измерить реальное потребление памяти JVM?

Слои:
```bash
# 1. Container / cgroup level
kubectl top pod X -n ns             # memory usage cgroup

# 2. Process level
ps -o rss,vsz,cmd -p <pid>
cat /proc/<pid>/status | grep -E "VmRSS|VmSize"

# 3. Detailed JVM breakdown (нужен -XX:NativeMemoryTracking=detail)
jcmd <pid> VM.native_memory summary
# показывает разбивку: Java heap, Metaspace, Code, Thread, GC, etc

# 4. Java heap detailed
jcmd <pid> GC.heap_info
jstat -gccapacity <pid>
jcmd <pid> GC.heap_dump /tmp/heap.hprof  # для offline analysis

# 5. Direct memory
jcmd <pid> VM.native_memory detail | grep -A5 "malloc.*Other"
```

Правило: RSS ≈ Java Heap used + Metaspace + Code Cache + (Xss × threads) + Direct + JVM internal ≈ 1.3-1.8× heap used.

### Q19: Что такое page fault и как влияет на Java?

**Page fault** — процесс обращается к virtual page, которая не имеет physical mapping. Kernel handler:
1. Аллоцирует physical page.
2. Если backed by file (mmap) — читает из disk.
3. Обновляет page table.
4. Возвращает control процессу — тот повторяет доступ.

**Minor page fault** — страница уже в RAM, но не mapped у процесса (shared library, another process). Fast.

**Major page fault** — из disk. Slow (SSD ~100 μs, HDD ~10 ms).

**Влияние на Java**:
- Первое обращение к heap page — minor fault, аллоцируется.
- Swapping (redko в K8s) → major faults → **приложение замирает**.
- mmap файла — first read = major fault, потом cached.

Метрика: `perf stat -p <pid> -e page-faults` или `cat /proc/<pid>/status | grep min|maj_flt`.

### Q20: Как оптимально настроить heap для микросервиса?

Общий подход:
1. **Start with `-XX:MaxRAMPercentage=70`** (70% of container memory).
2. **`-Xms == -Xmx`** — no heap resize pauses. Или `MinRAMPercentage == MaxRAMPercentage`.
3. **G1GC** для heap > 4GB, **Parallel GC** для < 2GB (throughput lower latency).
4. **ZGC** для очень low latency (< 10ms pause) heap > 8GB.
5. Мониторинг:
   - `jvm.gc.pause` — GC pause histogram.
   - `jvm.memory.used{area="heap"}` — heap usage.
   - `container_memory_working_set_bytes` — real RSS.
6. **Adjust по метрикам**:
   - Long GC pauses (>1s) → увеличить heap или сменить GC.
   - OOM после нескольких дней → memory leak, heap dump.
   - RSS растёт → NMT, native leak или Metaspace.

Cargo cult tunings типа `-XX:+UseCompressedOops -XX:+AggressiveOpts` — обычно **не нужны** в JDK 11+ (defaults sensible).

---

## 11. Мини-чеклист «прочитал — знаю»

За 3-5 секунд:

- [ ] Java = OS process (task_struct в Linux).
- [ ] Fork() дублирует всё через copy-on-write; для Java = deadlock (только calling thread выживает).
- [ ] Signals: SIGTERM (graceful), SIGKILL (unrecoverable, exit 137), SIGSEGV (native crash).
- [ ] Shutdown hooks обрабатывают SIGTERM, но не SIGKILL.
- [ ] Virtual memory: VSS (reserved), RSS (physical), PSS/USS.
- [ ] Overcommit — можно `mmap` больше RAM, OOM killer при реальной нехватке.
- [ ] Page = 4 KB (или huge pages 2MB/1GB).
- [ ] `/proc/<pid>/maps` — memory regions процесса.
- [ ] JVM memory: Heap + Metaspace + Code Cache + Thread stacks + Direct + Native.
- [ ] `-Xmx` — только heap max, RSS больше на 30-70%.
- [ ] `-XX:MaxRAMPercentage=70` в контейнерах.
- [ ] Container awareness: `-XX:+UseContainerSupport` (default in JDK 10+).
- [ ] Cgroups v2: `memory.max`, `cpu.max`, `memory.current`.
- [ ] Exit 137 = OOMKilled (SIGKILL от cgroup).
- [ ] Metaspace grows unbounded by default — set `-XX:MaxMetaspaceSize`.
- [ ] Code Cache full → JIT stops → performance drop.
- [ ] Direct memory (`ByteBuffer.allocateDirect`) — off-heap, `-XX:MaxDirectMemorySize`.
- [ ] Streams (java.io) — blocking, buffered в user space.
- [ ] NIO Channels — ByteBuffer, direct buffer support.
- [ ] `FileChannel.transferTo` — zero-copy sendfile.
- [ ] `FileChannel.map` — mmap, page fault on access.
- [ ] Page cache — все file I/O через RAM, `buff/cache` в `free`.
- [ ] fsync — гарантирует write на диск (медленно).
- [ ] File descriptors — sockets + files, limit `ulimit -n`.
- [ ] epoll (Linux) — NIO Selector внутри, O(1) events.
- [ ] Virtual threads I/O — auto-yield через epoll underneath.
- [ ] strace для syscall trace, perf для CPU flame.
- [ ] NMT (Native Memory Tracking) — `-XX:NativeMemoryTracking=detail` + `jcmd VM.native_memory`.
- [ ] `jcmd`, `jstack`, `jmap`, `jstat`, JFR — стандартный toolkit.
- [ ] Micrometer `jvm.memory.used`, `jvm.gc.pause`, `process.cpu.usage` — Prometheus.

---

## Итог

Java-приложение — это **OS process** который использует ресурсы через syscalls. Понимать все слои — обязательно для senior:

**Process уровень**:
- fork/exec, PID/PPID, signals (SIGTERM важен для graceful shutdown).
- Shutdown hooks работают до SIGKILL, но не после.

**Memory уровень**:
- Virtual (VSS) vs Physical (RSS) — always different.
- JVM memory ≠ heap: heap + metaspace + code cache + stacks + direct + native.
- `-Xmx` ≈ 70% container limit (место для native).
- Container awareness (JDK 10+) читает cgroups автоматически.
- Overcommit + OOM killer в Linux, cgroup memory limit в контейнерах → exit 137.

**Disk I/O**:
- Streams (java.io) — blocking, buffered.
- NIO Channels — direct buffer, ByteBuffer.
- mmap для random access.
- transferTo для zero-copy.
- Page cache ускоряет всё (не путать с memory pressure).
- fsync медленный, но нужен для durability.

**Network I/O**:
- Blocking sockets: 1 thread per connection.
- NIO Selectors через epoll: 1 thread per thousands of connections.
- Virtual threads: blocking API + non-blocking underneath.

**Наблюдаемость**:
- OS: `top`, `ps`, `free`, `iostat`, `ss`, `strace`, `perf`.
- JVM: `jcmd`, `jstack`, `jmap`, `jstat`, JFR, async-profiler.
- Metrics: Micrometer → Prometheus, cAdvisor per container.

**Production правила**:
- `-Xmx = 70% container memory`.
- Guaranteed QoS: `requests == limits`.
- Don't set CPU limit for Java.
- `-XX:+HeapDumpOnOutOfMemoryError`.
- Track: heap usage, GC pauses, direct memory, thread count, FD count, RSS.

Дальше — практика: возьми любой прод Pod своего сервиса, разбери что реально в его RSS через NMT, сравни с `container_memory_working_set_bytes` в Grafana. Всё встанет на место.
