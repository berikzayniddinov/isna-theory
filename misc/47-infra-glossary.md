# 47. Инфраструктурный глоссарий: сервер, VM, pod, CPU, buffer, connection

## Зачем нужен reference на базовые термины

Разработчик регулярно encounters инфраструктурные concepts не связанные напрямую с business logic его приложения. Термины letme «под капотом» K8s, network layer, OS mechanisms. Understanding этих terms critical для reading system architecture documents, discussing deployment issues с ops teams, troubleshooting production incidents. Часто разница между «работает» и «работает reliably» лежит в этих fundamentals.

Разница между разработчиком «знающим базовое» и «понимающим infrastructure» проявляется в incident scenarios. Первый видит «container OOMKilled», перезапускает pod, надеется на fix. Второй знает что OOMKilled means container exceeded memory limit — kernel killed process due to cgroup memory constraint. Investigation goes к memory usage patterns, heap sizing, potential leaks. Знает что context switch между threads costs microseconds — high thread count creates significant overhead beyond individual thread costs. Знает difference между file descriptor limits ulimit -n и per-process limits соединяющих множество networking issues.

В этом файле разберём critical infrastructure concepts глубоко enough для reasoning про system behavior. Сервер как general concept и its manifestations (physical, VM, container). Virtual Machine mechanics плюс discrimination от Java VM (JVM). Container mechanics через Linux namespaces plus cgroups. Kubernetes pod как composite unit deployment. CPU architecture — cores, threads, cache hierarchy, context switching, cgroup CPU limits в containers. RAM management. Buffers на многих levels — TCP, disk, application, log. Network connections plus TCP handshake, connection pools, file descriptors, HTTP keepalive. Process vs thread differences. Sockets plus TCP/UDP. Kernel vs user space plus system calls overhead.

## Сервер: общая концепция

Server это компьютер обслуживающий запросы других компьютеров. Simple definition охватывающая множество manifestations.

Physical server — реальная железка в data center. Rack-mounted unit с CPUs, RAM, storage, networking. Provides raw computational resource. Historically dominant deployment model. Requires physical management — power, cooling, network cabling.

Virtual server (VM) — программная эмуляция physical computer. Multiple VMs share physical hardware через hypervisor. Each VM appears как independent computer к software running внутри. Isolation between VMs.

Container — процесс с изоляцией через OS mechanisms. Shares kernel with host. Lightweight compared to VMs. Fast startup, low overhead.

Классы серверов по функции. Web server (nginx, Apache) обслуживает HTTP requests. Application server (Tomcat, JBoss) hosts application code. Database server (PostgreSQL, MySQL) persistent data storage. Message broker (RabbitMQ, Kafka) inter-service communication. Cache server (Redis, Memcached) fast access to cached data. File server storage plus retrieval. DNS server name resolution. Mail server email handling.

В микросервисной архитектуре — каждый микросервис обычно = один «server» role. Instances того же service могут быть multiple для scaling.

## Virtual Machine mechanics

VM это программная эмуляция физического компьютера. Внутри VM — своя ОС (guest OS) which считает что работает на реальном железе. Reality — hypervisor virtualizes все hardware access.

Hypervisor это software layer управляющий VMs. VMware ESXi, KVM (Kernel Virtual Machine, встроенный Linux), Hyper-V (Microsoft), Xen. Provides каждой VM.

Virtual CPU mapped к physical cores. Multiple VMs могут share cores через time-slicing — hypervisor scheduler decides какая VM runs on which core when. Modern CPUs имеют hardware virtualization support (Intel VT-x, AMD-V) reducing overhead.

Virtual RAM — chunk physical memory allocated к VM. Guest OS думает что имеет full RAM. Modern hypervisors support memory ballooning — reclaim unused memory from VMs, share pages между VMs (deduplication).

Virtual disk — обычно file или block device на host storage. Appears как physical disk к guest. Can be sparse (allocate space as used) или fully allocated.

Virtual network interface — connects VM к virtual network. Multiple VMs могут share physical network через bridged или NAT configurations.

Two hypervisor types. Type 1 (bare-metal) — hypervisor runs напрямую на hardware. VMware ESXi, Xen, KVM. Better performance, standard для production. Type 2 (hosted) — hypervisor runs как application inside host OS. VirtualBox, VMware Workstation. Convenient для development, overhead higher.

VMs vs containers key differences:

| | VM | Container |
|---|---|---|
| ОС | Своя (guest OS) | Общая с host |
| Размер | GB (full OS included) | MB (application plus libraries only) |
| Старт | Минуты (boot OS) | Секунды (start process) |
| Изоляция | Сильная (hardware-level virt) | Слабее (namespace-based) |
| Overhead | Много (OS overhead) | Мало (native processes) |

Different use cases. VMs для strong isolation, running different OSes, legacy applications requiring specific environments. Containers для application deployment где host OS acceptable, faster iteration, denser packing на hardware.

Java VM (JVM) — completely different concept. Not VM в смысле VMware. Bytecode interpreter — runs Java bytecode compiled from source. Provides platform independence — same bytecode runs everywhere JVM exists. Not virtualizing hardware — abstracting language runtime.

## Container mechanics

Container это isolated процесс plus его окружение (JDK, libraries, config). Implementation через Linux namespaces plus cgroups. Not separate OS — shares kernel with host.

Linux namespaces provide isolation dimensions.

PID namespace — process IDs. Container's processes see себя как PID 1 (init) plus own PID space. Host sees actual PIDs (different values).

Network namespace — networking. Own network interfaces, routing tables, iptables rules. Independent от host network.

Mount namespace — filesystem view. Own filesystem tree. Can bind-mount specific host paths для sharing.

UTS namespace — hostname, domain name. Container's hostname independent от host.

IPC namespace — inter-process communication (semaphores, shared memory). Isolated per container.

User namespace — user IDs. UID 0 (root) inside container может быть unprivileged user на host.

cgroups control resource usage. Memory cgroup limits max RAM. CPU cgroup limits CPU shares or hard limits. Block IO cgroup limits disk bandwidth. Network cgroup limits bandwidth or shapes traffic.

Together — namespaces isolate view, cgroups limit resources. Container это process с restricted view plus resource limits.

Runtimes for containers. Docker популярный original tool. containerd more focused modern runtime. Podman daemonless alternative к Docker. CRI-O Kubernetes-focused runtime. All implement OCI (Open Container Initiative) specifications.

## Kubernetes pod

Pod — минимальная единица деплоя в K8s. Обычно 1 pod = 1 container. Может быть multi-container (sidecar pattern) sharing.

Containers в одном pod share several resources. Network namespace — same IP address, same port space (containers can не listen on same port). IPC namespace — inter-process communication accessible between containers. Volumes — shared storage between containers.

Что НЕ shared. Filesystems — each container has separate filesystem. Processes — each container's process visible only within its container.

Multi-container patterns. Sidecar — helper container for main app (logging agent, service mesh proxy, metrics exporter). Ambassador — proxy для outbound connections (encapsulating service discovery). Adapter — normalizing output for standard monitoring tools.

Pod эфемерен — умер и появился новый с другим IP address. Каждый restart pods имеет different IP. Application code should NEVER hard-code pod IPs. Communication должно go через Kubernetes Services which provide stable endpoints поверх changing pod IPs.

## CPU: cores, threads, cache hierarchy

CPU (Central Processing Unit) — процессор. Modern CPUs имеют multiple cores plus multiple threads per core.

Core — физическая единица выполнения. Actual processing hardware. Independent instruction execution.

Thread — логическая единица. Через Hyper-Threading (Intel) или SMT (AMD) один core выполняет 2 threads simultaneously. Sharing core resources plus efficiencies.

Example. Intel Xeon 8 cores × 2 HT threads per core = 16 logical CPUs. Operating system sees 16 CPUs. Programs могут schedule 16 threads simultaneously executing. But actual parallelism limited к 8 (physical cores).

CPU cache hierarchy critical для performance:
```
Регистр       ~1 ns    (int, обрабатываемый сейчас)
L1 cache      ~1 ns    (32-64 KB)
L2 cache      ~4 ns    (256 KB - 1 MB)
L3 cache      ~15 ns   (несколько MB, общий на socket)
RAM           ~100 ns  (GB, «медленная» относительно cache)
Disk (SSD)    ~100 μs  (в 1000× медленнее RAM)
```

Разница между register и RAM — 100×. Между RAM и SSD — 1000×. Отсюда critical importance cache-friendly кода — data локально accessible через caches dramatically faster than RAM access.

Practical implications. Sequential data access (arrays) predictable к CPU prefetcher — brings data к cache before needed. Random access (linked lists, hash lookups) unpredictable — cache misses frequent — slow. Modern algorithm design prefers cache-friendly data structures.

CPU в containers. K8s измеряет CPU в millicores (m):
- 1000m = 1 полное ядро.
- 500m = 0.5 ядра (throttling).
- 100m = 0.1 ядра.

Pod resource specification:
```yaml
resources:
  requests: { cpu: 200m }
  limits:   { cpu: "2" }
```

Requests это гарантированный minimum — scheduler places pod на node with sufficient available. Limits максимальный upper bound — cgroups CPU throttling когда pod exceeds. Throttled tasks slowed down not killed.

Context switch — переключение между процессами/потоками. Not free — approximately 1-10 μs overhead per switch. Много potоков → много context switches → CPU занят переключением not useful work.

Отсюда — virtual threads в Java 21 (файл 19). Instead of OS threads (heavy context switches), virtual threads managed by JVM (lightweight scheduling). Allows millions of virtual threads without context switch storm.

Diagnosing CPU-bound scenarios. `top` показывает %CPU близко к 100% — CPU-bound. В application — thread dump показывает потоки в RUNNABLE state consistently — computation not I/O bound. Fixes — optimize algorithm complexity, parallelize hot paths, cache results.

## RAM management

Основная оперативная память. Units — 1 KB = 1024 bytes. 1 MB = 1024 KB. 1 GB = 1024 MB. Typical enterprise server имеет 8-256 GB RAM.

RAM в containers через K8s specifications:
```yaml
resources:
  requests: { memory: 512Mi }
  limits:   { memory: 1Gi }
```

Requests guaranteed minimum. Limits hard cap. При exceeding limit — OOMKilled. Kernel убивает процесс через OOM killer choosing worst offender. Not throttling like CPU — hard termination.

Отличия от CPU cache. RAM это DRAM technology — cheaper, denser, но slower than CPU caches. CPU caches SRAM — faster, more expensive, less dense. RAM measured в GB. Cache в MB (L3) или KB (L1/L2).

Приложение работает directly с RAM. CPU cache — прозрачное для программиста acceleration frequently accessed data. JIT compilers плюс OS help optimize placement.

## Buffer: temporary storage patterns

Buffer это временное хранилище данных между источником и потребителем. Идея — сгладить разницу в скоростях или упростить batching operations.

TCP buffer — kernel maintains receive и send buffers per TCP connection. Application writes к socket — kernel buffers данные для sending. Application reads — kernel provides из receive buffer already filled by network.

Configuration в Linux:
```
net.core.rmem_max = 16777216       # 16 MB max receive
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
```

If application doesn't read fast enough — buffer fills — TCP advertises reduced window — sender slows down (TCP flow control). Natural backpressure mechanism.

Disk buffer (page cache) — kernel caches file reads и writes. Read cache prevents re-reading same file. Write cache batches multiple writes into fewer disk operations. Explicit fsync forces flush для durability guarantees.

Отсюда `free -h` shows `used + free + buff/cache`. buff/cache appears used но can be freed by kernel when needed для actual application memory. Not «lost» memory.

Java ByteBuffer два types. `ByteBuffer.allocate(1024)` allocates в JVM heap. Standard managed memory. `ByteBuffer.allocateDirect(1024)` allocates outside heap. DMA-friendly для NIO operations. Not managed by regular GC — cleaner references handle disposal.

Application buffers common в various libraries. Kafka producer buffers messages до `linger.ms` или `batch.size`. Consumer buffers polled records между process calls. JDBC batching accumulates INSERTs.

Log buffer — async log appender queues events. Background thread writes к disk. Application не blocks на log I/O. Caveat — при JVM crash не yet-written events lost.

## TCP connection lifecycle

TCP connection — полнодуплексный канал между two IP:port pair'ами. Established через 3-way handshake:
```
Client → SYN → Server
Client ← SYN-ACK ← Server
Client → ACK → Server
Connected!
```

Approximately 1 RTT (round-trip time). Local network 1 ms. Cross-region 100 ms или more.

TLS handshake adds 1-2 RTT для encryption negotiation. Total connection establishment 30-100 ms с SSL включая all handshakes plus authentication.

Connection pool это переиспользование open connections. Amortizes handshake cost over many requests. Critical optimization.

JDBC pool (HikariCP) maintains persistent connections к DB. HTTP client pools (Apache HttpClient, OkHttp) reuse TCP connections для HTTP requests. Kafka producer/consumer maintain persistent connections к brokers.

Connection lifecycle states. Establish — TCP плюс TLS handshake. Idle — открыт, не используется, held for reuse. In-use — application actively using. Close — TCP 4-way close handshake gracefully terminates.

Common problems. Stale connection — network/firewall killed but pool doesn't know. Fix — keepalive probes plus test-on-borrow validation. Leak — application acquired but not released. Fix — try-with-resources plus leak-detection-threshold в HikariCP. Exhausted pool — все в use, new requests wait. Fix — increase pool или fix leak.

HTTP keep-alive default в HTTP/1.1. Connection не closes после request/response. Next request from same client reuses TCP connection. Amortizes handshake overhead.

HTTP/2 multiplexes multiple requests через one connection. Multiple concurrent requests interleaved. Better utilization single TCP connection.

WebSocket reuses TCP как full-duplex stream. Long-lived connection для bi-directional communication.

## File descriptors

В Linux каждое TCP connection = file descriptor. Also opened files, pipes, sockets — all file descriptors. Kernel resource per process.

Limit через ulimit -n. Default обычно 1024 — too low для serious production. Should be 65535 или higher.

Утечка FD leads к «Too many open files» errors. Monitor через `lsof -p <pid> | wc -l`. Investigate когда count keeps growing without corresponding workload growth.

## Process vs Thread

Process — изолированный экземпляр программы. Каждый имеет свой PID (process ID), свою virtual memory (isolated от других), свои open files и sockets, свои threads (минимум один — main thread).

Создание процесса дорого. fork() в Linux copies entire process state. Optimized через copy-on-write но still non-trivial cost.

Thread — единица исполнения внутри процесса. Threads одного процесса делят memory space (heap), open files, sockets. Свои — stack, registers, program counter.

Создание thread быстрее создания process. Communication между threads через shared memory (легче, но более dangerous — race conditions).

В Java — раньше 1 Java Thread = 1 OS thread. Heavy — expensive create, expensive context switch. С Java 21 — Virtual Threads — millions of lightweight threads. JVM schedules them onto pool OS threads. Efficient для I/O-bound workloads.

Detailed virtual threads в файле 19.

## Sockets

Socket = endpoint для сети. Pair (IP address, port).

Types. Stream (TCP) — reliable, ordered, connection-based. Datagram (UDP) — unreliable, unordered, faster, connectionless. Unix domain socket — IPC на one host через filesystem socket file, faster than TCP loopback.

Server sockets. bind(port) plus listen() — server prepares to accept connections. accept() — creates new socket для each incoming connection. Original listening socket continues accepting more connections.

Client sockets. connect(server_ip, port) — establishes connection к server. Client uses socket для sending/receiving data.

Socket = file descriptor в Linux. read()/write() work как with files. Uniform I/O API.

## Kernel vs User space

Kernel space — Linux kernel code. Manages CPU scheduling, memory allocation, I/O (disk, network), filesystem, process management. Works в privileged mode с full hardware access.

User space — обычные процессы. Applications, services, tools. Works в user mode с limited privileges. Cannot access hardware directly или memory других processes.

System calls — mechanism for user space processes к request kernel services. When application needs I/O — read, write, send, recv, open, close — invokes system call.

System call flow. Application makes function call (например read()). Library translates к syscall instruction. CPU switches user→kernel mode (context switch, expensive). Kernel executes actual operation. Returns к user mode. Application receives result.

Context switches cost 100 ns to microseconds. Many system calls per operation kills performance. Optimizations. Batch I/O — one syscall для multiple operations (writev, sendfile). Async I/O — epoll, io_uring — reduce syscall overhead through event notification instead of per-operation calls.

Java code goes через JVM runtime которое uses syscalls when needed. Direct sysscall costs mostly hidden but still relevant для performance-critical paths.

## Terminology cheat sheet

Comprehensive quick reference:

| Термин | Одной строкой |
|---|---|
| Сервер | Компьютер обслуживающий запросы (физ / VM / контейнер) |
| VM | Программная эмуляция полного компьютера + своя ОС |
| Container | Процесс + изоляция (namespaces + cgroups), общий kernel |
| Pod | Мин. единица K8s: 1+ контейнер, общий IP + volumes |
| CPU / core / thread | Процессор / физ. ядро / логический поток (HT) |
| RAM | Оперативная память |
| Cache (CPU) | L1/L2/L3 — быстрая память между CPU и RAM |
| Buffer | Временное хранилище между источником и потребителем |
| Connection (TCP) | Канал между IP:port pair'ами (handshake, keep-alive) |
| Connection pool | Переиспользование открытых connections |
| File descriptor (FD) | Handle на open file/socket в Linux |
| Process | Изолированный экземпляр программы |
| Thread | Единица исполнения внутри процесса |
| Virtual thread | JVM-managed «зелёный» поток (Java 21+) |
| Socket | Endpoint для сети (IP, port) |
| Kernel space | Ядро Linux (privileged) |
| User space | Обычные процессы (non-privileged) |
| System call | Запрос от user к kernel (read, write, open) |
| Context switch | Переключение CPU между потоками/процессами |
| Namespace (Linux) | Изоляция «видимости» (PID, network, mount) |
| cgroup | Ограничение ресурсов (CPU, memory) |
| Hypervisor | Управляет VM (ESXi, KVM) |
| L4 / L7 | OSI: transport (TCP) / application (HTTP) |
| SSL / TLS | Криптографический протокол поверх TCP |
| DNS | Разрешение имён в IP |
| Firewall | Фильтр сетевого трафика |
| Load balancer | Распределение трафика между backends |
| Reverse proxy | Прокси перед backends |
| CDN | Distributed cache для static content |
| VPN | Encrypted tunnel через публичную сеть |
| MTU | Max Transmission Unit — размер пакета |
| RTT | Round-Trip Time — сеть latency |
| DDoS | Distributed Denial of Service — flood от многих |
| WAF | Web Application Firewall — L7 защита |

## Куда копать за деталями

Файл 09 docker-detailed — детали VM/container/pod.

Файл 10 kubernetes-detailed — K8s architecture.

Файл 43 java-basics-primitives-memory — CPU и память Java глубоко.

Файл 02 jvm-jdk-jre-bytecode — JVM internals.

Файл 29 postgresql-spring-hikaricp — connection pool detailed.

Файл 31 load-balancer — TCP / L4 / L7 concepts.

Файл 19 java-21-virtual-threads — virtual threads deeply.

Файл 44 prometheus-grafana-metrics — метрики observability.

Файл 37 nodes-detailed — nodes в разных distributed systems.

## Итоги

Сервер как general concept охватывает physical, VM, container, pod realizations. Каждый с trade-offs.

VM — full OS on hypervisor. Strong isolation, minute-scale startup, GB-scale sizes.

Container — process с namespaces plus cgroups. Shared kernel, seconds-scale startup, MB-scale sizes. Different tradeoff.

Pod в K8s — 1+ container sharing network, IPC, volumes. Ephemeral IPs — communicate through Services.

CPU cores plus threads (HT). Cache hierarchy critical (register → L1 → L2 → L3 → RAM 100× → disk 1000× slower). Context switch overhead ~1-10 μs. cgroup limits в containers.

RAM management с requests plus limits в containers. OOMKilled hard kill on exceed.

Buffer temporary storage smoothing rate differences. TCP kernel buffers, disk page cache, Java ByteBuffer heap vs direct, application-level batching.

TCP connection lifecycle с 3-way handshake ~1 RTT plus TLS 1-2 RTT. Connection pools amortize cost.

File descriptors как handles к files/sockets. ulimit constraint. Leaks manifested через «Too many open files».

Process vs thread. Threads share memory within process. Virtual threads (Java 21) million-scale через JVM scheduling.

Kernel space privileged, user space regular apps. System calls bridge, cost context switches. Batching plus async I/O reduce overhead.

Дальше — глубокое comparison монолит vs микросервисы, когда что выбирать, migration strategies.
