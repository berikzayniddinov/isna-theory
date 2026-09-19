# 47. Инфра-глоссарий: сервер, VM, pod, CPU, buffer, connection

Базовые термины инфраструктуры одной страницей. Ссылки на файлы с деталями.

---

## 1. Сервер (server)

**Server** — компьютер, который **обслуживает запросы** других компьютеров.

Может быть:
- **Физический** — реальная железка в дата-центре.
- **Виртуальный** (VM) — программа-эмулятор.
- **Контейнер** — процесс с изоляцией.

Класс сервера:
- Web server (nginx, Apache).
- Application server (Tomcat, JBoss).
- Database server (PostgreSQL, MySQL).
- Message broker (RabbitMQ, Kafka).
- Cache server (Redis, Memcached).
- File server.
- DNS, mail, ...

В микросервисной архитектуре: **каждый микросервис = один сервер** (обычно).

---

## 2. Виртуальная машина (VM)

**Virtual Machine** — программная эмуляция физического компьютера.

### 2.1 Как работает

**Hypervisor** (VMware ESXi, KVM, Hyper-V) — управляет VM. Даёт каждой VM:
- Свой virtual CPU (маппится на реальные ядра).
- Свою RAM (кусок физической).
- Свой disk (файл / block device).
- Свой network interface.

Внутри VM — **своя ОС** (Windows / Linux / etc), которая думает что бежит на железе.

### 2.2 Типы

- **Type 1 (bare-metal)** — hypervisor напрямую на железе (VMware ESXi, Xen, KVM).
- **Type 2 (hosted)** — hypervisor как программа в ОС (VirtualBox, VMware Workstation).

### 2.3 vs Container

| | VM | Container |
|---|---|---|
| ОС | Своя | Общая с хостом |
| Размер | GB | MB |
| Старт | Минуты | Секунды |
| Изоляция | Сильная | Слабее (общий kernel) |
| Overhead | Много | Мало |

Details — файл `09-docker-detailed.md`.

### 2.4 Ещё одно значение — Java VM

**JVM** — не «виртуальная машина как VMware», а **байткод-интерпретатор**. Другой уровень абстракции.

Не путать.

---

## 3. Контейнер (container)

**Container** = изолированный процесс + всё его окружение (JDK, библиотеки, конфиг).

Реализация — Linux namespaces + cgroups. Не своя ОС, kernel общий с хостом.

Инструменты: **Docker**, **containerd**, **Podman**, **CRI-O**.

Детально — файл `09-docker-detailed.md`.

---

## 4. Pod (в Kubernetes)

**Pod** — минимальная единица деплоя в K8s. Обычно 1 pod = 1 контейнер.

Может быть sidecar-паттерн: pod с 2-3 контейнерами делящими:
- Network (один IP).
- IPC.
- Volumes.

При **не** делящими:
- Файловую систему.
- Процессы.

**Pod эфемерен**: умер → появился новый **с другим IP**. Никогда не обращайся к pod по IP напрямую — только через Service.

Детально — файл `10-kubernetes-detailed.md`.

---

## 5. CPU

### 5.1 Устройство

**CPU (Central Processing Unit)** — процессор.

- **Ядро (core)** — физическая единица выполнения.
- **Поток (thread)** — логическая (через **Hyper-Threading** одно ядро выполняет 2 потока).
- Пример: Intel Xeon 8 cores × 2 HT = 16 logical CPUs.

### 5.2 Иерархия памяти CPU

```
Регистр       ~1 ns    (int, обрабатываемый сейчас)
L1 cache      ~1 ns    (32-64 KB)
L2 cache      ~4 ns    (256 KB - 1 MB)
L3 cache      ~15 ns   (несколько MB, общий на socket)
RAM           ~100 ns  (GB, «медленная» относительно cache)
Disk (SSD)    ~100 μs  (в 1000× медленнее RAM)
```

Разница между регистром и RAM — **100×**. Между RAM и SSD — **1000×**. Отсюда важность cache-friendly кода.

### 5.3 CPU в контейнерах

K8s измеряет CPU в **millicores** (m):
- 1000m = 1 полное ядро.
- 500m = 0.5 ядра (throttling).
- 100m = 0.1 ядра.

Настройка pod:
```yaml
resources:
  requests: { cpu: 200m }
  limits:   { cpu: "2" }     # 2 ядра max
```

**Cgroups CPU** — если pod превышает limit, kernel throttles (не kill).

### 5.4 Context switch

Переключение между процессами/потоками. Не бесплатно — ~1-10 μs.

Много потоков → много context switches → CPU занят переключением, не полезной работой.

Отсюда — **virtual threads** (см. `19-java-21-virtual-threads.md`) без context switch overhead.

### 5.5 Как понять CPU-bound

- `top` показывает `%CPU` ~100%.
- В приложении — thread dump показывает потоки в RUNNABLE постоянно.

Fix: оптимизация алгоритма, параллелизация, кэширование результата.

---

## 6. RAM (память)

Основная оперативная память.

### 6.1 Единицы

1 KB = 1024 bytes.
1 MB = 1024 KB.
1 GB = 1024 MB.

Типичный сервер: 8-256 GB RAM.

### 6.2 В контейнерах

```yaml
resources:
  requests: { memory: 512Mi }
  limits:   { memory: 1Gi }
```

Превысил limit → **OOMKilled** (kernel убивает процесс).

### 6.3 Отличия от кэша CPU

RAM — DRAM, дешевая, много. Cache CPU — SRAM, дорогая, мало.

Приложение работает **в RAM**. Cache CPU — прозрачно, ускоряет доступ к «горячим» данным RAM.

---

## 7. Buffer (буфер)

**Buffer** — временное хранилище данных между **источником** и **потребителем**.

Идея: **сгладить разницу в скоростях** или упростить batching.

### 7.1 TCP buffer

Kernel держит буферы приёма/отправки на каждом TCP-соединении.

Настройка Linux:
```
net.core.rmem_max = 16777216       # 16 MB max receive
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
```

Если приложение не читает быстро → буфер заполнен → sender замедляется (backpressure).

### 7.2 Disk buffer

Kernel кэширует чтение/запись диска (**page cache**).

Кэш чтения — не читать с диска повторно.
Кэш записи — накопить и flush пачками (`fsync` для гарантии).

Отсюда — `free -h` показывает `used + free + buff/cache`. `buff/cache` может освободиться при необходимости.

### 7.3 Java ByteBuffer

```java
ByteBuffer buf = ByteBuffer.allocate(1024);    // heap
ByteBuffer direct = ByteBuffer.allocateDirect(1024);   // off-heap
```

Используется для I/O (NIO, файлы, сокеты). Direct buffer — не под GC, DMA-friendly.

### 7.4 Application buffer

Producer буферизует сообщения перед отправкой (Kafka `linger.ms + batch.size`).
Consumer буферизует принятые до обработки.
JDBC batch — накапливаем INSERT.

### 7.5 Log buffer

Async log appender — очередь в памяти, background thread пишет на диск.

Приложение не блокируется на I/O. Кавет: при crash — необлитая очередь теряется.

---

## 8. Connection (соединение)

### 8.1 TCP connection

Полнодуплексный канал между двумя IP:port pair'ами.

**TCP handshake** (3-way):
```
Client → SYN → Server
Client ← SYN-ACK ← Server
Client → ACK → Server
Connected!
```

~1 RTT (round-trip time) = ~1 ms local, ~100 ms cross-region.

**TLS handshake** — ещё 1-2 RTT сверху для криптографии.

### 8.2 Connection pool

Переиспользование открытых connections. Экономия handshake.

- **JDBC pool** (HikariCP) — БД. См. `29-postgresql-spring-hikaricp.md`.
- **HTTP client pool** (Apache HttpClient, OkHttp) — HTTP downstream.
- **Rabbit / Kafka** — internal pooling.

### 8.3 Connection lifecycle

1. **Establish** — TCP + TLS handshake.
2. **Idle** — открыт, не используется. Держится некоторое время.
3. **In-use** — приложение работает с ним.
4. **Close** — TCP 4-way handshake close.

Проблемы:
- **Stale connection** — сеть/firewall убил, но пул не знает. Fix: `keepalive` + `test-on-borrow`.
- **Leak** — приложение взяло, не отдало. Fix: `try-with-resources`, `leak-detection-threshold`.
- **Exhausted pool** — все в use, новые ждут. Fix: увеличить pool или починить утечку.

### 8.4 HTTP keep-alive

По умолчанию HTTP/1.1 — connection **не закрывается** после ответа (keep-alive). Следующий запрос переиспользует TCP.

HTTP/2 — multiplexing (много запросов через одно соединение параллельно).

WebSocket — переиспользование TCP как full-duplex stream.

### 8.5 File descriptors

В Linux каждое TCP-соединение = **file descriptor**. Лимит через `ulimit -n` (default 1024, prod 65535+).

Утечка FD → «Too many open files». Мониторить `lsof -p <pid> | wc -l`.

---

## 9. Процесс vs Поток

### 9.1 Процесс

**Process** — изолированный экземпляр программы.

Каждый процесс имеет:
- Свой PID.
- Свою virtual memory (isolated).
- Свои открытые файлы, sockets.
- Свои threads (минимум 1 — main).

Создание процесса — **дорого** (fork в Linux).

### 9.2 Поток (thread)

**Thread** — единица исполнения **внутри процесса**.

Потоки одного процесса делят:
- Virtual memory (heap).
- Открытые файлы, sockets.

Свои:
- Stack.
- Registers.
- PC.

Создание — быстрее чем процесс. Общение через shared memory (легче / опаснее).

### 9.3 В Java

Раньше: 1 Java Thread = 1 OS thread. Дорого.

С Java 21: **Virtual Threads** — миллионы дешёвых потоков, JVM их шедулит на пул OS-потоков.

Детально — `19-java-21-virtual-threads.md`.

---

## 10. Socket

**Socket** = endpoint для сети. Пара (IP, port).

### 10.1 Types

- **Stream** (TCP) — надёжный, ordered.
- **Datagram** (UDP) — быстрый, без гарантий.
- **Unix domain socket** — IPC на одном хосте (быстрее TCP).

### 10.2 Server + Client

**Server socket**: `bind(port) + listen()` → ждёт connections.
При `accept()` → создаётся **новый socket** для этой connection.

**Client socket**: `connect(server_ip, port)` → устанавливает.

### 10.3 File descriptor

Socket = file descriptor в Linux. `read()/write()` как с файлом.

---

## 11. Kernel vs User space

### 11.1 Kernel space

Ядро Linux. Управляет:
- CPU scheduling.
- Memory.
- I/O (диск, сеть).
- Файловая система.
- Процессы.

Работает в **privileged mode**.

### 11.2 User space

Обычные процессы (nginx, Java, ваше приложение).

Работают в **user mode**. Не могут напрямую в железо / память других процессов.

### 11.3 System call

Чтобы приложение сделало I/O — нужно **system call**: `read`, `write`, `send`, `recv`, `open`, `close`.

При этом:
1. Переключение user → kernel mode (**context switch, дорого**).
2. Kernel выполняет операцию.
3. Возврат в user.

Дорогие: `~100 ns - несколько μs`. Много sys calls → CPU-bound.

Отсюда — **batch I/O** (одним syscall много данных), **async I/O** (epoll, io_uring).

---

## 12. Terminology cheat sheet

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

---

## 13. Куда копать за деталями

- **VM / контейнер / pod** → `09-docker-detailed.md`, `10-kubernetes-detailed.md`.
- **CPU / память Java** → `43-java-basics-primitives-memory.md`, `02-jvm-jdk-jre-bytecode.md`.
- **Connection pool / HikariCP** → `29-postgresql-spring-hikaricp.md`.
- **TCP / L4 / L7** → `31-load-balancer.md`.
- **Virtual threads** → `19-java-21-virtual-threads.md`.
- **Metrics для мониторинга** → `44-prometheus-grafana-metrics.md`.
- **Nodes в разных системах** → `37-nodes-detailed.md`.

---

## 14. Собесные вопросы

1. **Разница физ. сервера, VM, контейнера?** — VM: полная ОС + hypervisor; контейнер: процесс + изоляция, общий kernel; физ.: реальное железо.
2. **Что такое pod в K8s?** — Мин. единица деплоя; 1+ контейнер с общим IP/volumes.
3. **CPU core vs thread?** — Core — физическая единица; thread — логическая (через HT 1 core = 2 threads).
4. **Что такое context switch?** — Переключение CPU между потоками; ~1-10 μs.
5. **Что такое buffer?** — Временное хранилище между источником и потребителем.
6. **Что такое TCP handshake?** — 3-way (SYN, SYN-ACK, ACK) для установления TCP.
7. **Что такое connection pool?** — Переиспользование открытых соединений (JDBC, HTTP).
8. **Что такое file descriptor?** — Handle на open file/socket в Linux.
9. **Разница process и thread?** — Process: свой memory, изоляция; thread: делит memory с другими threads в процессе.
10. **Kernel space vs user space?** — Kernel: privileged (управляет железом); user: обычные процессы.
11. **Что такое system call?** — Запрос от user к kernel (read/write/open); context switch overhead.
12. **Что такое namespace в Linux?** — Изоляция «видимости»: PID, network, mount, IPC.
13. **cgroup?** — Ограничение ресурсов (CPU, memory) для группы процессов.
14. **RTT?** — Round-Trip Time; local ~1 ms, cross-region ~100 ms.
15. **HTTP keep-alive?** — TCP не закрывается после HTTP-ответа; переиспользуется для следующих запросов.

---

## Итог

- **Сервер** = физ / VM / контейнер / pod, обслуживает запросы.
- **VM** = своя ОС на hypervisor.
- **Контейнер** = процесс + namespaces + cgroups, общий kernel.
- **Pod** = мин. единица K8s.
- **CPU** = core × HT threads; иерархия cache → RAM → disk (100× × 1000×).
- **Buffer** = временное хранилище (TCP, disk, application).
- **Connection** = TCP endpoint pair; keep-alive + pool.
- **File descriptor** = handle на socket/file.
- **Process** = изоляция; **thread** = разделяет memory; **virtual thread** = JVM-managed.
- **System call** = user → kernel (context switch dorogо).

---

## Финальный итог блока

Файлы 32-47 добавлены. Всего в `isna-theory\` теперь **47 файлов** (~30-50 страниц каждый).

Полный список последних блоков:

**@Transactional** (32-35):
- 32 — ACID, isolation, propagation, XA, Saga
- 33 — @Transactional изнутри (прокси, self-invocation, PlatformTransactionManager)
- 34 — продвинутое (rollback, listeners, savepoints, TransactionTemplate)
- 35 — @Transactional + JPA (PersistenceContext, flush, LazyInit, OSIV)

**Spring Cloud** (36):
- 36 — Config, Consul, LoadBalancer, Feign, Circuit Breaker, Gateway, Sleuth/Tracing

**Nodes** (37):
- 37 — K8s / Consul / RabbitMQ / PG / Kafka / Elastic / consensus / quorum / split-brain

**Логирование** (38):
- 38 — SLF4J / Logback / MDC / structured JSON / ELK / правила

**Kafka** (39-42):
- 39 — основы (broker, topic, partition, offset, retention)
- 40 — producer / consumer / offsets / groups / rebalancing / delivery semantics
- 41 — Spring Kafka (@KafkaListener, error handler, DLT)
- 42 — прод (transactions, EOS, monitoring, common issues)

**Java основы + мониторинг + инфра** (43-47):
- 43 — примитивы / объекты / память (heap, non-heap, stack, direct)
- 44 — Prometheus + Grafana + Micrometer (метрики RED/USE)
- 45 — Elasticsearch (indexes, shards, replicas, roles, ELK)
- 46 — nginx (event-driven, reverse proxy, LB, SSL, cache, ingress)
- 47 — инфра-глоссарий (сервер / VM / pod / CPU / buffer / connection)

Готово. Скажи что дальше: Redis / Kubernetes продвинутое / distributed systems (CAP, eventual consistency) / testing / observability целиком / что-то другое.
