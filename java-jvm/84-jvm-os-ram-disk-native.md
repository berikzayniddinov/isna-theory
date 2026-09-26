# 84. JVM ↔ OS: RAM, диск, процессы, threads — глубокая теория

## Зачем это знать

Java-приложение живёт не в вакууме. Оно — процесс в OS, использующий ресурсы через syscalls. Каждый вызов `System.out.println`, каждое чтение файла, каждое сетевое соединение — это работа ядра. Каждый гигабайт heap — это virtual memory pages, которые kernel маппит в физическую RAM по мере обращения. Каждый OOMKilled в Pod'е — это cgroup memory limit, который вы превысили, и Linux OOM killer убил ваш процесс.

Большинство Java-разработчиков видит только верхушку этого айсберга. Они знают, что есть heap и что можно настроить его через `-Xmx`. Но не знают, что RSS их приложения в контейнере складывается из heap + metaspace + code cache + thread stacks + direct memory + native buffers, и что `-Xmx4g` в контейнере с limit 4Gi — гарантированный OOMKilled. Не знают, что `Files.write` кладёт данные не на диск, а в page cache, и что если сервер упадёт до fsync — данные потеряны. Не знают, что `-Xmx` не резервирует RAM физически, что overcommit разрешает virtual allocation больше, чем есть RAM, и что реальная аллокация происходит при first write через page fault.

Всё это критично, когда что-то ломается в проде. OOMKilled — почему, если я выставил `-Xmx3g` в container с limit 4Gi? Медленное чтение из БД — это page cache холодный или сеть? Приложение висит 10 секунд после старта — это slow JVM startup или ждём БД? High CPU — это GC, JIT, или наш код? Каждый из этих симптомов требует понимания слоёв: JVM, kernel, OS, hardware.

Мы разберём JVM как OS-процесс: fork/exec, PID, signals, shutdown hooks. Виртуальную память: VSS vs RSS, page tables, overcommit, OOM killer. JVM memory layout полностью: heap (young/old, все GC), metaspace, code cache, thread stacks, direct memory, native memory через NMT. Container awareness: cgroups v2, `-XX:+UseContainerSupport`, `MaxRAMPercentage`. Disk I/O через все слои: streams (java.io), NIO channels, memory-mapped files, page cache, fsync, direct I/O, io_uring, file descriptors. Network I/O: blocking sockets, NIO Selectors, epoll, Netty, VT + network. Syscalls стоящие за обычными Java-операциями через strace. Наблюдаемость через OS tools (ps, top, free, iostat) и JVM tools (jcmd, jstack, jmap, JFR). Практические сценарии диагностики: OOMKilled, high CPU, high disk/network I/O, slow startup.

## JVM как OS-процесс

С точки зрения ядра Java-приложение — обычный процесс, представленный `task_struct` в Linux (или аналогичной структурой в других OS). Ключевые атрибуты:

- **PID** — уникальный идентификатор.
- **Virtual address space** — 48-bit на x86_64 (256 TB теоретически, практически меньше из-за kernel space разделения).
- **File descriptor table** — открытые файлы, sockets, pipes.
- **Signal handlers** — как реагировать на SIGTERM, SIGKILL, SIGSEGV.
- **Credentials** — UID/GID (кто запустил).
- **Working directory**.
- **Environment variables**.
- **Process group / session** — для job control.

Java-приложение = процесс, запущенный `java -jar app.jar`. Один экземпляр JVM = один процесс. Один процесс = один heap, одна set of threads, один set of file descriptors.

**Что происходит при `java -jar app.jar`**:

1. **Shell fork()** — создаётся child process (клон текущего shell).
2. **`execve("/usr/bin/java", ["java", "-jar", "app.jar"], envp)`** — child заменяет свой memory на бинарник Java.
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

Всё это — 100-500 мс для стандартной JVM. AOT-компилированные native images (GraalVM) — 20-50 мс.

PID, PPID, PGID, SID — иерархия процессов:

- **PID** — процесс.
- **PPID** — parent process (кто fork'нул). Если parent умер, PPID становится 1 (init/systemd).
- **PGID** — process group. `Ctrl+C` посылает SIGINT всей группе.
- **SID** — session. Detached processes имеют свою SID (nohup).

Внутри Docker контейнера ты видишь PID 1 (это твой процесс). На хосте у него другой PID (namespace isolation).

**fork() и Java** — очень плохая идея. `fork()` дублирует весь процесс (copy-on-write memory pages). В Java это очень дорого:

- JVM = ~100 MB - 2 GB heap + другие области.
- Fork копирует все page tables — медленно.
- Child получает копию только calling thread (POSIX behavior). Другие JVM threads в child висят в half-state → deadlock likely.

Итог: **никогда не форкай Java-процесс**. `Runtime.exec()` использует `posix_spawn` (или `vfork+execve` на старых системах) — безопасно.

**Сигналы**. OS может послать процессу signal — асинхронное уведомление. JVM ловит и обрабатывает:

- **SIGTERM (15)** — «пожалуйста, завершись». JVM запускает shutdown hooks, exits gracefully. **Это то, что kubelet шлёт перед убийством Pod'а.**
- **SIGKILL (9)** — «умри немедленно». Ядро убивает процесс, JVM не может ловить. Shutdown hooks не запускаются.
- **SIGINT (2)** — Ctrl+C, обычно = SIGTERM behavior.
- **SIGHUP (1)** — terminal disconnect, для daemons часто reload config.
- **SIGSEGV (11)** — segfault, доступ к невалидной памяти. JVM крашится с `hs_err_pid*.log`.
- **SIGABRT (6)** — assertion failure, native crash.

Register shutdown hook:

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // cleanup on SIGTERM
    connectionPool.close();
    log.info("Graceful shutdown");
}));
```

Хуки не гарантированны при SIGKILL / OOM / power loss.

## Виртуальная память процесса

Одна из самых важных вещей понимать — разницу между **виртуальной** и **физической** памятью.

**Физическая память** — реальные ячейки DDR RAM на плате. Скажем, 32 GB на сервере. Это единственная реальная RAM, все процессы делят её.

**Виртуальная память** — каждый процесс имеет **свой** 48-bit address space (256 TB теоретически). Ядро через MMU (Memory Management Unit) маппит virtual pages → physical page frames. Каждый процесс думает, что имеет всю память в своём распоряжении, но реально физическая аллокация делается по требованию.

**Page** — единица mapping'а, обычно **4 KB** (huge pages — 2 MB или 1 GB). Все memory operations работают через pages.

**Page table** — структура в ядре, хранит mapping virtual page → physical page frame. Per process (плюс kernel page tables).

**TLB (Translation Lookaside Buffer)** — cache недавних translations в CPU. Full flush TLB — expensive (context switch между процессами). Между threads одного процесса не флашится, потому что address space один.

Ключевые метрики memory usage:

- **VSS (Virtual Set Size)** — сколько virtual памяти зарезервировано. Может быть намного больше RAM.
- **RSS (Resident Set Size)** — сколько физической памяти реально используется. То, что реально в RAM.
- **PSS (Proportional Set Size)** — RSS с учётом shared памяти (делится между процессами).
- **USS (Unique Set Size)** — только приватная память процесса.

Пример: JVM с `-Xmx4g` — VSS может быть 6-8 GB (heap + metaspace + code cache + threads + native libraries reserved). RSS — только то, что реально написано, обычно 500 MB - 2 GB для стартующего Spring Boot приложения:

```bash
ps -o pid,vsz,rss,cmd -p <pid>
#   PID    VSZ    RSS CMD
# 12345 8123456 987654 java -jar app.jar
# VSZ = 8 GB virtual, RSS = 987 MB physical
```

Важно понимать: VSS ≠ RSS. `-Xmx4g` **не** значит, что JVM сразу занимает 4 GB физической RAM. Это max limit heap'а, virtual reservation. Реально used physical — по мере обращения к страницам.

**Memory mapping regions** процесса в Linux — смотреть `/proc/<pid>/maps`:

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

`[anon]` = anonymous mapping (не привязан к файлу), обычно через `mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)`. JVM heap — большой anonymous mapping.

**Overcommit** — Linux по умолчанию разрешает `mmap` больше памяти, чем есть RAM+swap. Реальное выделение — при **first write** (write triggers page fault → kernel allocates physical page).

Плюс: приложения могут резервировать много (`-Xmx8g` не значит 8 GB физически сразу). Минус: если все процессы вдруг захотят реально использовать резерв — **OOM killer** просыпается, убивает процесс с наибольшим `oom_score`.

Настройки:

```
/proc/sys/vm/overcommit_memory
  0: heuristic (default)
  1: always overcommit
  2: never overcommit (strict accounting)
/proc/sys/vm/overcommit_ratio  # для mode 2
```

В контейнерах — cgroup memory limit важнее, чем OS-level overcommit. Cgroup ограничивает **своей** OOM machinery, независимо от host.

## JVM memory layout — все области

JVM использует **несколько отдельных memory regions**, не только «heap». Понимать все — необходимо для troubleshooting OOM.

**Java Heap** — основное место для объектов, созданных `new`. Управляется GC.

Разделён на:

- **Young generation** — новые объекты, быстрый GC (minor GC).
  - **Eden** — куда попадают новые объекты.
  - **Survivor S0, S1** — после первого GC (объекты, пережившие minor GC).
- **Old generation (tenured)** — долгоживущие объекты после нескольких GC.
- **Humongous** (G1 specifically) — большие объекты > half of region size.

Настройка:

```
-Xms2g             # начальный размер heap
-Xmx4g             # максимальный размер
-XX:NewRatio=2     # young:old ratio
-XX:SurvivorRatio=8  # eden:survivor ratio
```

Различные GC внутри HotSpot:

- **G1 GC** (default с JDK 9) — regions по 1-32 MB. Хорошо балансирует latency и throughput. Для heap > 4 GB.
- **ZGC** — colored pointers, sub-ms pauses. Для very-low-latency heap > 8 GB.
- **Shenandoah** — concurrent compaction.
- **Parallel GC** — throughput-oriented, для batch processing.

**Metaspace** (было PermGen до Java 8) — хранит **class metadata**: класс structure, method info, constant pool, bytecode.

Не в heap! Хранится в **native memory** (mmap-based).

Настройка:

```
-XX:MetaspaceSize=128m           # initial
-XX:MaxMetaspaceSize=512m        # limit (default unlimited!)
```

Grows automatically до `MaxMetaspaceSize` или до OOM, если не задан.

**OutOfMemoryError: Metaspace** — обычно классы утекают: dynamic classloading без освобождения (Groovy scripting, JSP recompilation, старый Tomcat undeployment). Проверить:

```bash
jcmd <pid> VM.native_memory summary  # если включена NMT
jmap -clstats <pid>                  # class loaders + count
```

**Code Cache** — JIT-скомпилированный native code. Bytecode → x86 machine code через C1/C2 compilers.

```
-XX:InitialCodeCacheSize=64m
-XX:ReservedCodeCacheSize=240m    # default для JDK 8+
```

Если заполнен → JIT stops compiling → интерпретация → **производительность падает**. JVM warning в логах:

```
CodeCache is full. Compiler has been disabled.
```

Fix: увеличить `ReservedCodeCacheSize` или включить flushing (`-XX:+UseCodeCacheFlushing`).

С JDK 9+ разделён на 3 сегмента (non-nmethod, profiled, non-profiled).

**Thread stacks** — каждый platform thread имеет свой stack, обычно **1 MB** (`-Xss1m`).

Native memory, вне heap. Не управляется GC.

Виртуальные threads — stack chunks в heap.

**Direct memory (off-heap)** — `ByteBuffer.allocateDirect(size)` выделяет память **вне Java heap**. Используется NIO, Netty, PostgreSQL JDBC driver (bulk operations), Kafka clients.

Не управляется GC напрямую! Освобождается, когда `DirectByteBuffer` объект собран (через `Cleaner`) — может быть с задержкой.

Настройка:

```
-XX:MaxDirectMemorySize=1g       # default = Xmx
```

Tricky: если `MaxDirectMemorySize` не задан явно, default = `Runtime.maxMemory()` = Xmx. При активном NIO — может удвоить memory usage.

OOM: `java.lang.OutOfMemoryError: Direct buffer memory`.

**Native memory** — что использует:

- JVM internal (GC bookkeeping, JIT compiler working sets).
- Native libraries via JNI (например Jansi для colored output, netty-tcnative для SSL, JNA).
- `MappedByteBuffer` (mmap файлы).
- ZIP/JAR read (кэши в native).

Не покрывается `-Xmx`. Смотреть через **NMT (Native Memory Tracking)**:

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
```

**Реальный memory footprint** — ключевая формула:

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
- 200 threads × 1 MB = 200 MB stacks.
- Direct memory ~100 MB (Netty).
- Native ~150 MB.
- **Total RSS ~1.3 GB**.

Container с memory limit 1 GB → **OOMKilled** после старта. Правило JVM в контейнере: `Xmx = ~70-75% container limit`.

**`-XX:MaxRAMPercentage`** — более гибкое чем `-Xmx`:

```
-XX:MaxRAMPercentage=75.0
-XX:InitialRAMPercentage=50.0
-XX:MinRAMPercentage=50.0
```

JVM читает cgroup limit и вычисляет heap как %. Полезно в контейнерах — не надо hardcode'ить Xmx.

## Container awareness (cgroups)

Linux **cgroups v2** ограничивает ресурсы процесса. Для контейнеров Kubernetes использует cgroups через kubelet + containerd.

Файлы в `/sys/fs/cgroup/`:

- `memory.max` — hard limit. При превышении → **OOM killer** внутри cgroup.
- `memory.high` — soft limit. Throttling + reclaim, но не убийство.
- `memory.current` — сколько используется сейчас.
- `memory.events` — count of oom, high events.

**OOMKilled в K8s** — cgroup OOM убил процесс. Exit code 137 (128 + 9 SIGKILL). Причина в контейнере — `memory.max` exceeded.

**JVM container awareness** — JDK 10+ (и backport в 8u191) автоматически детектирует cgroup limits:

- `Runtime.getRuntime().availableProcessors()` — cgroup CPU quota, не всё ядро хоста.
- `Runtime.getRuntime().maxMemory()` — cgroup memory limit, не хостовая RAM.

Настройки (обычно включены by default):

```
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=75.0
```

Без `UseContainerSupport` (старый JDK 8):

- JVM видит всю хостовую память (например 128 GB) вместо cgroup 1 GB.
- Ставит heap ~30% = 30 GB.
- **Мгновенный OOMKilled**.

Fix: обновить JDK или явно `-Xmx800m`.

**CPU limits и JVM**. `limits.cpu: 2` в K8s → cgroup CFS quota: за каждые 100 мс позволено 200 мс CPU time (2 core equivalent).

**Throttling** — если превысил, приложение замедляется (не убивают). Для JVM это болезненно:

- GC threads throttled → длинные GC pauses.
- JIT compilation stalls.
- Обычный код тормозит.

Метрика в cadvisor/Prometheus: `container_cpu_cfs_throttled_periods_total`.

**Recommendation**: не ставь CPU limit для Java-приложений вообще. Только `requests` (гарантия). Пусть JVM использует свободные CPU когда есть.

Проверка cgroup изнутри контейнера:

```bash
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

## Disk I/O в Java — все слои

Начнём с общей картины. Каждое обращение к диску проходит через несколько слоёв:

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

Каждый слой добавляет latency, но все важны для performance.

**Streams (java.io)** — классический blocking API:

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
- `.read()` — syscall `read(fd, buf, 1)`. Медленно, если делать по 1 байту.
- `BufferedInputStream` — читает чанком (default 8 KB) в internal buffer, потом отдаёт побайтно. Один syscall на 8K bytes.

Правило: всегда оборачивай `FileInputStream` в `BufferedInputStream`. 100× быстрее.

**NIO Channels (java.nio)** — более современный API, работает через `ByteBuffer`:

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

**Memory-mapped files (mmap)**:

```java
try (FileChannel channel = FileChannel.open(Paths.get("big.dat"), READ)) {
    MappedByteBuffer buf = channel.map(READ_ONLY, 0, channel.size());
    // buf теперь как обычный byte array, но backed by disk
    byte b = buf.get(1_000_000);   // не syscall, page fault → kernel загружает страницу
}
```

Что происходит:

- Kernel создаёт mapping virtual page → file offset.
- Первое обращение к странице → **page fault** → kernel читает 4 KB из файла → возвращает.
- Последующие обращения — прямое чтение из RAM (страница в page cache).

Плюсы: нет копирования kernel → user buffer. Random access через pointer arithmetic. OS сам управляет caching (page cache).

Минусы: address space limited (2 GB per mapping на 32-bit). Изменения persisted через `msync()` или на close. Не подходит для sequential streaming (просто оверхед).

Использование: базы данных (Kafka Log Segments используют mmap для fast reads), большие индексы.

**Page cache** — ключевая оптимизация Linux. Все чтения/записи файлов проходят через page cache в RAM.

- Первое чтение файла → syscall → диск → страница в page cache → отдаётся приложению.
- Следующее чтение того же файла → syscall → **cache hit** → не идёт до диска → возвращает.

Смотреть:

```bash
free -h
#              total     used   free   shared  buff/cache  available
# Mem:          32Gi     8Gi   4Gi    100Mi   20Gi        23Gi
```

`buff/cache` — page cache + buffer cache. Ядро автоматически освобождает при memory pressure.

Кажется, что памяти мало (`free` 4 GB), но `available` учитывает page cache — реально доступно 23 GB.

Влияние на Java. Читаешь тот же файл дважды → второе чтение быстрое (в RAM). Kafka reads — очень fast, если данные в page cache (обычно недавно записанные). **JVM restart не теряет page cache** — новый процесс тут же читает быстро (важно для K8s pod restart).

**Write и fsync**. `fileChannel.write(buf)` — кладёт данные в **page cache**, не на диск!

Kernel периодически (обычно 30 сек, `/proc/sys/vm/dirty_expire_centisecs`) сбрасывает dirty pages на диск.

Если сервер упал между write и flush → **данные потеряны**.

`fsync` — гарантирует запись:

```java
fileChannel.force(true);   // fsync — данные + metadata на диске
fileChannel.force(false);  // fdatasync — только данные (без metadata)
```

fsync медленный (~1-10 мс на HDD, ~50-500 μs на NVMe SSD). Нужен для:

- Databases (WAL).
- Message brokers (durable messages).
- Не нужен для temp/cache файлов.

**Direct I/O (bypass page cache)** — `O_DIRECT` flag обходит page cache, читает/пишет напрямую с диска.

Полезно для databases со своим buffer manager (PostgreSQL, Oracle). Java до JDK 10 не поддерживал! JDK 10+ — `ExtendedOpenOption.DIRECT`. Обычно **не нужно** для application code — page cache ускоряет всё.

**io_uring** — новая async I/O infrastructure Linux (kernel 5.1+). Заменяет старый POSIX AIO.

Ring buffers для submission и completion, share'ятся между user space и kernel. **Zero syscall overhead** для batch I/O.

Java пока не имеет прямой поддержки, но Netty 4.1.x+ имеет `IOUringEventLoopGroup`. Loom (Java 21) внутри может использовать io_uring для virtual thread I/O. Обычно application-level не трогает.

**File descriptors — лимиты**. Каждый открытый файл/socket/pipe — file descriptor (integer). У процесса есть лимит:

```bash
ulimit -n            # soft limit
ulimit -Hn           # hard limit
cat /proc/<pid>/limits | grep "open files"
```

Default обычно 1024 (пугающе низкий для сервера). Обычно повышают до 65536+.

```bash
ls /proc/<pid>/fd/ | wc -l   # сколько FDs используется
```

**Too many open files** exception — забыл закрыть stream / socket / connection в pool. Try-with-resources решает проблему.

В K8s задаётся через securityContext или через init container:

```yaml
securityContext:
  sysctls:
  - name: fs.file-max
    value: "1000000"
```

## Network I/O

Аналогично disk I/O — разные API с разными характеристиками.

**Blocking sockets (java.net)** — классический API:

```java
try (Socket socket = new Socket("host", 80);
     OutputStream out = socket.getOutputStream();
     InputStream in = socket.getInputStream()) {
    out.write(request);
    int b = in.read();  // блокирует thread пока данные не придут
}
```

Один thread на соединение. При 10000 concurrent — 10000 threads. Не масштабируется.

**NIO Selectors (java.nio)** — non-blocking:

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

**epoll — Linux механика**. Три syscalls:

- `epoll_create1()` — создать epoll instance (FD).
- `epoll_ctl(epfd, ADD/MOD/DEL, fd, event)` — управлять watched FDs.
- `epoll_wait(epfd, events, maxevents, timeout)` — блокировать пока не появится событие.

Эффективно даже для 100k sockets (O(1) на event vs O(n) в старом `select()`).

Edge-triggered vs Level-triggered:

- **Level-triggered** (default) — событие пока условие есть (много wake-up'ов).
- **Edge-triggered** — только на переход состояния (один wake-up per event, эффективнее но сложнее).

**Netty** — framework для NIO. Каждый `EventLoopGroup` имеет несколько threads, каждый thread — свой Selector с thousands of channels. Producer сервисы в КНП (`isnaknpgateway`) — часто на Netty.

**Virtual threads + network** — Java 21. Network I/O в JDK автоматически yield'ит virtual thread при block. Внутри — NIO Selector shared for all VT.

Результат: пишешь blocking код (`socket.read()`), JVM использует epoll underneath. Императивный API, реактивная производительность.

## System calls за обычными Java-операциями

Каждая Java-операция превращается в syscalls. Знать что именно — полезно для performance analysis.

**strace на Java**:

```bash
strace -f -p <pid>            # trace всех syscalls
strace -f -e trace=network java MyApp     # только network
strace -c -p <pid>            # count summary
```

`System.out.println("hello")`:

```
write(1, "hello\n", 6) = 6
```

Один syscall на печать. Просто.

`Files.write(path, bytes)`:

```
openat(AT_FDCWD, "/tmp/f", O_WRONLY|O_CREAT|O_TRUNC, 0644) = 5
write(5, "hello", 5) = 5
close(5) = 0
```

Три syscall'а. Каждый ~1-10 μs.

**Network syscall trace** — `new URL("https://api.com").openStream()`:

```
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 6
connect(6, {sin_family=AF_INET, sin_port=htons(443), sin_addr=inet_addr("...")}, 16) = 0
(TLS handshake — многочисленные read/write)
write(6, "GET / HTTP/1.1\r\n...", 76) = 76
read(6, "HTTP/1.1 200 OK\r\n...", 8192) = 1024
close(6) = 0
```

Каждый HTTP request — минимум 4-5 syscalls (connect, write, N reads, close).

**GC syscalls** — G1 GC при collection:

- `mmap()` — если нужно расширить heap.
- `madvise(MADV_DONTNEED)` — вернуть страницы ядру (после uncommit).
- `mprotect()` — защита memory regions.

Обычно скрыто от программиста.

**Thread create** — `Thread.start()`:

```
clone(child_stack=..., flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|...) = 12345
```

Один syscall создаёт OS thread. Дальше mutex init, TLS init.

## Наблюдаемость: как всё это мониторить

Разбор инструментов по слоям.

**OS-level tools**:

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

**JVM tools**:

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

**Prometheus metrics** через Micrometer в Spring Boot:

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

Grafana dashboard «JVM (Micrometer)» — standard visualization.

**Node exporter (для K8s)** — DaemonSet на каждой ноде экспортирует OS metrics: CPU, memory, disk, network per host. Container-level (через cAdvisor встроен в kubelet):

- `container_memory_working_set_bytes`
- `container_cpu_cfs_throttled_periods_total`
- `container_fs_reads_bytes_total`
- `container_network_receive_bytes_total`

## Практические сценарии диагностики

**Pod OOMKilled**. Симптомы: `kubectl describe pod` → `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`.

Диагностика:

1. **Проверить memory limit vs Xmx**. Если `limits.memory: 1Gi`, а `-Xmx1g` — плохо. Native memory + threads + metaspace добавят 200-500 MB → RSS 1.2-1.5 GB → OOM.
2. **Fix**: `-Xmx700m` (или `-XX:MaxRAMPercentage=70`) при `limits.memory: 1Gi`.
3. **Или**: увеличить memory limit.

Более глубоко:

- **Real memory leak?** — если Xmx правильный, но RSS растёт со временем.
  - Heap dump: `jcmd <pid> GC.heap_dump`. Открыть в Eclipse MAT. Найти dominant tree.
  - Native leak: `-XX:NativeMemoryTracking=detail` + `jcmd VM.native_memory baseline/diff`.
- **Direct memory leak?** — `jcmd VM.native_memory` в разделе "Other" растёт. Часто Netty без `.release()`.
- **Class loader leak?** — Metaspace growing. `jmap -clstats` покажет.

**High CPU**:

```bash
top -H -p <pid>                    # thread'ы по CPU
# найти nid (native id) hot thread'а
printf '%x\n' <nid>                # в hex
jstack <pid> | grep 0x<hex_nid>    # найти в stack trace
```

Показывает what code съедает CPU. Часто:

- Infinite loop.
- GC hyperactive → увеличить heap или tune.
- JIT compilation (первые минуты после старта).
- Regex catastrophic backtracking.

Async Profiler для flame graph:

```bash
./profiler.sh -e cpu -d 30 -f flame.html <pid>
```

**High disk I/O**:

```bash
iotop -p <pid>            # что процесс делает с диском
pidstat -d 1 -p <pid>     # read/write bytes per second
```

Причины:

- **Logging** — избыточно, синхронно. Async appender с buffer.
- **GC** — если swap (обычно нет swap в K8s).
- **Application** — bulk imports, cache warmup.
- **Page cache miss** — файлы не в RAM.

**High network**:

```bash
ss -tnp | grep <pid>                # open sockets
iftop -f "host <pod-ip>"            # bandwidth
tcpdump -i eth0 -w capture.pcap port 8080
```

Причины:

- Chatty microservice (много мелких calls вместо batch).
- Connection pool exhaustion → создание новых.
- gRPC/HTTP keep-alive off.

**Slow startup**. Spring Boot стартует 30 сек — можно ускорить:

- **Lazy initialization**: `spring.main.lazy-initialization: true`.
- **Class Data Sharing (CDS)**: JDK 13+ auto, ускоряет class loading.
- **AOT / GraalVM Native Image**: старт 20-50 мс.
- **Spring AOT** (3.0+): ahead-of-time processing beans.

## Production-настройки JVM в K8s Pod'е

Собираем всё вместе:

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
    -XX:NativeMemoryTracking=summary
```

Ключевое:

1. **`Xmx ≈ 70% limits.memory`** — место для native memory.
2. **`requests == limits`** для memory → Guaranteed QoS, меньше вероятность eviction.
3. **Не ставь CPU limit** — GC threads не должны throttling'ать.
4. **`-XX:+HeapDumpOnOutOfMemoryError`** — автоматический heap dump при OOM.
5. **`-XX:+ExitOnOutOfMemoryError`** — не пытаться жить после OOM (kubelet рестартит Pod).
6. **NMT enabled** — для диагностики native memory leaks.

Как измерить реальное потребление памяти:

```bash
# 1. Container / cgroup level
kubectl top pod X -n ns             # memory usage cgroup

# 2. Process level
ps -o rss,vsz,cmd -p <pid>
cat /proc/<pid>/status | grep -E "VmRSS|VmSize"

# 3. Detailed JVM breakdown (нужен -XX:NativeMemoryTracking=detail)
jcmd <pid> VM.native_memory summary

# 4. Java heap detailed
jcmd <pid> GC.heap_info
jstat -gccapacity <pid>
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# 5. Direct memory
jcmd <pid> VM.native_memory detail | grep -A5 "malloc.*Other"
```

Правило: RSS ≈ Java Heap used + Metaspace + Code Cache + (Xss × threads) + Direct + JVM internal ≈ 1.3-1.8× heap used.

## Page fault и его влияние на Java

**Page fault** — процесс обращается к virtual page, которая не имеет physical mapping. Kernel handler:

1. Аллоцирует physical page.
2. Если backed by file (mmap) — читает из disk.
3. Обновляет page table.
4. Возвращает control процессу — тот повторяет доступ.

**Minor page fault** — страница уже в RAM, но не mapped у процесса (shared library, another process). Fast.

**Major page fault** — из disk. Slow (SSD ~100 μs, HDD ~10 мс).

Влияние на Java:

- Первое обращение к heap page — minor fault, аллоцируется.
- Swapping (редко в K8s) → major faults → **приложение замирает**.
- mmap файла — first read = major fault, потом cached.

Метрика: `perf stat -p <pid> -e page-faults` или `cat /proc/<pid>/status | grep min|maj_flt`.

## Оптимальные настройки heap для микросервиса

Общий подход:

1. **Start with `-XX:MaxRAMPercentage=70`** (70% container memory).
2. **`-Xms == -Xmx`** — no heap resize pauses. Или `MinRAMPercentage == MaxRAMPercentage`.
3. **G1GC** для heap > 4 GB, **Parallel GC** для < 2 GB (throughput lower latency).
4. **ZGC** для очень low latency (< 10 мс pause) heap > 8 GB.
5. Мониторинг:
   - `jvm.gc.pause` — GC pause histogram.
   - `jvm.memory.used{area="heap"}` — heap usage.
   - `container_memory_working_set_bytes` — real RSS.
6. **Adjust по метрикам**:
   - Long GC pauses (>1 s) → увеличить heap или сменить GC.
   - OOM после нескольких дней → memory leak, heap dump.
   - RSS растёт → NMT, native leak или Metaspace.

Cargo cult tunings типа `-XX:+UseCompressedOops -XX:+AggressiveOpts` — обычно **не нужны** в JDK 11+ (defaults sensible).

## Zero-copy и transferTo

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

## Заключение

Java-приложение — это **OS process**, использующий ресурсы через syscalls. Понимать все слои — обязательно для senior:

**Process уровень**: fork/exec, PID/PPID, signals (SIGTERM важен для graceful shutdown). Shutdown hooks работают до SIGKILL, но не после.

**Memory уровень**: Virtual (VSS) vs Physical (RSS) — always different. JVM memory ≠ heap: heap + metaspace + code cache + stacks + direct + native. `-Xmx` ≈ 70% container limit (место для native). Container awareness (JDK 10+) читает cgroups автоматически. Overcommit + OOM killer в Linux, cgroup memory limit в контейнерах → exit 137.

**Disk I/O**: Streams (java.io) — blocking, buffered. NIO Channels — direct buffer, ByteBuffer. mmap для random access. `transferTo` для zero-copy. Page cache ускоряет всё (не путать с memory pressure). `fsync` медленный, но нужен для durability.

**Network I/O**: Blocking sockets — 1 thread per connection. NIO Selectors через epoll — 1 thread per thousands of connections. Virtual threads — blocking API + non-blocking underneath.

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

Практический совет: возьмите любой прод Pod своего сервиса, разберите что реально в его RSS через NMT (`jcmd VM.native_memory summary`), сравните с `container_memory_working_set_bytes` в Grafana. Разница между heap used и RSS — это native, metaspace, threads, direct. Понимание этих цифр — база для правильной настройки resources в K8s.

Следующий шаг — разберитесь с одним конкретным инцидентом OOMKilled в вашей истории. Что было в heap dump? Какое было `-Xmx`? Какое `limits.memory`? Что показывало NMT? Пройдите цепочку рассуждений — и следующий OOMKilled решается за 10 минут, а не 3 часа.
