# 31. Load Balancer: виды, как работает, за что отвечает

Что такое LB, зачем, разница L4/L7, server-side vs client-side, алгоритмы.

---

## 1. Что такое Load Balancer

**Load Balancer (LB)** — компонент, который распределяет входящий трафик между несколькими серверами (инстансами приложения).

Задачи:
1. **Распределение нагрузки** — чтобы один сервер не перегружался.
2. **High availability** — если один сервер упал, трафик идёт на живые.
3. **Масштабирование** — можно добавить/удалить инстансы без остановки сервиса.
4. **Health checking** — LB знает какие серверы живы.
5. **SSL termination** — расшифровать HTTPS один раз на LB, дальше идти plain HTTP внутри кластера.
6. **Rate limiting**, **security** — WAF, DDoS-защита.

Аналогия: в банке есть N кассиров и один администратор, который направляет клиентов к свободному кассиру.

---

## 2. Основные концепты

- **Frontend** — то, что видит клиент (VIP + порт).
- **Backend / upstream** — реальные серверы за LB.
- **Pool / target group** — набор backend'ов.
- **Health check** — периодическая проверка живости backend'а.
- **Algorithm** — правило распределения.
- **Session affinity / sticky session** — привязка клиента к одному backend.

Схема:
```
                Client
                  │
                  ▼
            ┌───────────┐
            │Load       │  ← Frontend: 200.10.20.30:443
            │Balancer   │
            └─────┬─────┘
                  │
     ┌────────────┼────────────┐
     ▼            ▼            ▼
  [App 1]      [App 2]      [App 3]  ← Backend pool
   10.0.1.5    10.0.1.6    10.0.1.7
```

---

## 3. L4 vs L7

Классификация по уровню OSI.

### 3.1 L4 (Transport Layer)

Работает с **TCP/UDP пакетами**. Ничего не знает про HTTP/HTTPS содержимое.

Что делает:
- Смотрит только IP + port (source/destination).
- Пересылает пакеты не изменяя.
- Быстрый (нет парсинга).
- Может балансировать любой TCP-протокол (не только HTTP).

Примеры:
- **AWS NLB (Network Load Balancer)**.
- **HAProxy в TCP mode**.
- **LVS (Linux Virtual Server)**.
- **F5 BIG-IP LTM в L4 mode**.
- **K8s Service (ClusterIP)** — тоже L4, работает через iptables.

Плюсы:
- Быстро (миллионы пакетов в секунду).
- Простой.
- Работает для любого TCP (Postgres, Redis, gRPC, WebSocket).

Минусы:
- Не может routing по URL / header.
- SSL термирировать не может (нужен L7).
- Не понимает retry / smart-routing.

### 3.2 L7 (Application Layer)

Работает с **HTTP-запросами**. Понимает URL, headers, cookies.

Что делает:
- Парсит HTTP-запрос.
- Может routить по:
  - Host header (`knp.kgd.gov.kz` vs `arm.kgd.gov.kz`).
  - URL path (`/api/knp` vs `/api/fno`).
  - Method, headers, cookies.
- SSL termination.
- Modify headers, rewrite URLs.
- Compression, caching.

Примеры:
- **nginx**.
- **Envoy**.
- **Traefik**.
- **HAProxy в HTTP mode**.
- **AWS ALB (Application Load Balancer)**.
- **K8s Ingress Controller** (nginx-ingress, traefik).
- **Spring Cloud Gateway** (в приложении).

Плюсы:
- Богатая маршрутизация.
- SSL termination.
- Rate limiting, WAF.
- HTTP-специфичные фичи (compression, cache).

Минусы:
- Медленнее L4 (парсинг).
- Только HTTP/HTTPS обычно.
- Больше memory footprint.

### 3.3 Практика

Обычно **L4 перед L7**:
```
Internet → NLB (L4) → ALB / nginx (L7) → App
```

L4 распределяет по нескольким L7 инстансам для их HA.

---

## 4. Server-side vs client-side LB

### 4.1 Server-side LB

Классический подход. **LB отдельная сущность**, клиент шлёт ему.

```
Client → LB → [server1, server2, server3]
```

Плюсы:
- Клиент простой, не знает про топологию.
- Централизованное управление.
- Легко подключать не-Java клиентов.

Минусы:
- LB — точка отказа (нужен свой HA).
- Лишний hop (Client → LB → Server = 2 сетевых прыжка).
- Меньше tunability на клиенте.

Примеры: nginx, HAProxy, AWS ALB, K8s Service.

### 4.2 Client-side LB

Клиент **сам** знает про все backend'ы, сам выбирает один.

```
Client с LB-logic
        │
        │  запрашивает список из registry (Consul, Eureka, K8s API)
        │
    [server1, server2, server3]
        │
        │  выбирает server2 по алгоритму
        ▼
    server2
```

Плюсы:
- Нет лишнего hop.
- Клиент видит топологию, может retry на другой инстанс.
- Custom алгоритмы (weighted, sticky).

Минусы:
- Логика LB в каждом клиенте.
- Библиотека нужна для каждого языка.
- Больше нагрузка на service registry.

Примеры:
- **Ribbon** (Spring Cloud Netflix, deprecated).
- **Spring Cloud LoadBalancer** — современная замена.
- **gRPC client-side LB**.
- **Feign** через SC LB.

**В ИСНА**: одновременно и **server-side** (nginx-ingress перед gateway), и **client-side** (Feign через Consul внутри).

---

## 5. Алгоритмы балансировки

### 5.1 Round Robin (RR)

Простейший: по очереди, по кругу.

```
request 1 → server A
request 2 → server B
request 3 → server C
request 4 → server A
request 5 → server B
...
```

Плюсы: справедливо, просто.
Минусы: не учитывает нагрузку разных серверов.

### 5.2 Weighted Round Robin

Каждому серверу вес.
```
A (weight=3), B (weight=1)
→ A A A B A A A B ...
```

Использование: разные мощности серверов; canary release (5% на новую версию).

### 5.3 Least Connections

Идёт на сервер с наименьшим количеством активных connections.

Плюсы: учитывает реальную занятость.
Минусы: LB должен считать connections (state).

### 5.4 Least Response Time

Идёт на сервер с наименьшей средней latency.

Учитывает latency + connections. Умный.

### 5.5 IP Hash

`hash(client_ip) % N` → тот же клиент → всегда на тот же сервер.

Плюсы: sticky session без cookies.
Минусы: неравномерно (один прокси = один hash = один сервер).

### 5.6 Consistent Hashing

`hash(request_key) → server`. При добавлении/удалении сервера — минимальное перераспределение.

Использование: кэш-серверы (Memcached, Redis Cluster).

### 5.7 Random

Просто случайный. Работает удивительно хорошо (Power of Two Choices).

### 5.8 Power of Two Choices

Выбираешь два случайных, идёшь на менее загруженный. Аппроксимация Least Connections без state.

---

## 6. Sticky sessions

**Sticky session (session affinity)** — тот же клиент → всегда тот же backend.

Зачем: если backend хранит session в памяти (не в Redis).

Реализация:
- **IP hash** — по IP клиента.
- **Cookie-based** — LB устанавливает cookie `LB_SERVER=A`, читает при следующих запросах.

**Правила**:
- **Stateless** приложения (JWT, session в Redis) — sticky **НЕ нужен**.
- Плохо для скэйла — нельзя equally распределить.
- Смерть sticky-серверa → все клиенты теряют session.

**Правильно**: делать stateless приложение. Sticky — костыль.

---

## 7. Health checks в LB

LB периодически проверяет каждый backend:

### 7.1 TCP check

Просто открыть TCP-соединение на порт. TCP handshake = OK.

Быстро, но грубо (порт открыт, а приложение сломано).

### 7.2 HTTP check

`GET /health` → 200-399 = OK.

Обычно на `/actuator/health/readiness` в Spring Boot. Подробно в файле `10-kubernetes-detailed.md`.

### 7.3 Custom check

Скрипт / кастомная логика.

### 7.4 Параметры

- **Interval** — как часто проверять (5-30 сек).
- **Timeout** — сколько ждать ответа.
- **Healthy threshold** — сколько успешных подряд чтобы считать здоровым.
- **Unhealthy threshold** — сколько неуспешных чтобы вывести.

При unhealthy → LB перестаёт слать трафик (не убивает backend, это уже дело K8s).

---

## 8. SSL termination

```
Client ──HTTPS──► LB ──HTTP──► Backend
```

LB расшифровывает HTTPS. Внутри кластера — plain HTTP (быстрее, легче отладка).

Настройка:
- LB имеет TLS-сертификат.
- Backend без SSL, слушает 8080.
- Пропускает через приватную сеть (не через интернет).

Если требуется end-to-end TLS (compliance) — **SSL passthrough** или **re-encryption**.

---

## 9. Как это в ИСНА

Полная картина запроса `https://knp.kgd.gov.kz/api/fno/submit`:

```
Пользователь
    │  HTTPS 443
    ▼
[External LB / firewall]           ← L4 / L7
    │
    ▼
[nginx-ingress]                     ← L7, terminates SSL, routing по host
    │  routes: knp.kgd.gov.kz → svc/isnaknpgateway
    ▼
[K8s Service: isnaknpgateway]      ← L4 (iptables), ClusterIP
    │  round-robin по подам gateway
    ▼
[isna-knp-gateway pod]              ← Java 11, Zuul
    │  routes: /services/X → Consul lookup + client-side LB
    │  (или через K8s Service внутри)
    ▼
[Consul discovery]                  ← client-side LB (Ribbon / SC LoadBalancer)
    │  дал inst: 10.0.1.5:8080
    ▼
[isna-knp-integration pod]
    │
    ├─ через Feign → Consul lookup → isnaknpuser pod
    ├─ через Feign → Consul lookup → isnaknpfno pod
    └─ БД / RabbitMQ
```

Реальный ИСНА-кейс `knp-fo-consul-lb-mr1223-latent-mine`: при миграции Ribbon → Spring Cloud LoadBalancer сменилась case-sensitivity — `isnaKnpUser` не резолвился → 500.

---

## 10. K8s Service как LB

K8s Service = **L4 LB** для подов внутри кластера.

Механика:
1. Endpoints controller следит за подами с матчащим label.
2. Kube-proxy на каждой ноде создаёт iptables/IPVS правила: «трафик на 10.96.0.5:8080 → раскинуть по IPs подов».
3. Балансировка round-robin по iptables.

Типы (см. `10-kubernetes-detailed.md`):
- **ClusterIP** — внутри кластера.
- **NodePort** — порт на всех нодах.
- **LoadBalancer** — просит облачный LB.

---

## 11. K8s Ingress как L7 LB

**Ingress** = declarative HTTP-роутинг. Реализуется **Ingress Controller**-ом (обычно nginx).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: knp-ingress
spec:
  tls:
    - hosts: [knp.kgd.gov.kz]
      secretName: knp-tls
  rules:
    - host: knp.kgd.gov.kz
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: isnaknpgateway
                port: { number: 8080 }
```

Ingress Controller (nginx-ingress) читает Ingress-ресурсы → генерирует nginx.conf → reload nginx.

---

## 12. Spring Cloud LoadBalancer

Client-side LB **внутри приложения**. Замена Ribbon.

Работает автоматически с Feign / RestTemplate / WebClient если есть `spring-cloud-starter-loadbalancer`.

```java
@Autowired RestTemplate rest;

// Используя service name (не URL!)
UserDto u = rest.getForObject("http://isnaknpuser/api/users/1", UserDto.class);
```

Под капотом:
1. `LoadBalancerInterceptor` перехватывает.
2. Резолвит `isnaknpuser` через ServiceInstanceListSupplier (Consul, Eureka, K8s).
3. Выбирает инстанс по алгоритму.
4. Заменяет URL на реальный IP.

Кастомный алгоритм:
```java
@Bean
ReactorLoadBalancer<ServiceInstance> customLB(Environment env,
        LoadBalancerClientFactory factory) {
    String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
    return new RandomLoadBalancer(
        factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
}
```

Доступные алгоритмы: `RoundRobinLoadBalancer` (default), `RandomLoadBalancer`.

---

## 13. Типовые продукты LB

### 13.1 nginx

- L7 (HTTP), может L4 (stream module).
- Open source + commercial (nginx Plus).
- Легковесный, быстрый.
- Хороший для static + reverse proxy.

### 13.2 HAProxy

- L4 + L7.
- Специализация — балансировка.
- Богатые метрики, ACLs.

### 13.3 Envoy

- L7 modern proxy.
- Основа service mesh (Istio, Linkerd).
- Мощная динамическая конфигурация через xDS API.
- Observability first (метрики + tracing).

### 13.4 Traefik

- L7.
- Хорошая интеграция с K8s / Docker.
- Автоматически discovers backends.

### 13.5 AWS ALB / NLB / GLB

- Managed (не надо обслуживать).
- ALB = L7, NLB = L4, GLB = глобальный anycast.

### 13.6 F5 BIG-IP

- Enterprise, дорогой.
- Hardware + software.
- WAF, DDoS protection, SSL offload.

---

## 14. Graceful drain

Когда убираешь backend (deploy, scale down):

Плохо:
1. `kill -9 pod` → все in-flight запросы теряются.

Хорошо:
1. Убрать backend из pool (LB перестаёт слать новых).
2. Дать time для дообработки in-flight (30-60 сек).
3. Убить pod.

K8s делает автоматически:
- Preлib pod → status=Terminating.
- Endpoints controller убирает pod из Service (LB перестаёт слать).
- SIGTERM → приложение graceful shutdown.
- `terminationGracePeriodSeconds` (default 30) — сколько ждать.
- SIGKILL если не завершилось.

Spring Boot graceful shutdown:
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

---

## 15. Собесные вопросы

1. **Что такое Load Balancer, зачем?** — Распределение трафика между инстансами, HA, скэйлинг.
2. **Разница L4 и L7 LB?** — L4 = TCP/UDP (быстро, для любого протокола); L7 = HTTP (routing по URL/header, SSL).
3. **Server-side vs client-side LB?** — SSLB отдельная сущность; CSLB логика в клиенте (Feign+Consul).
4. **Алгоритмы балансировки?** — Round-robin, weighted RR, least connections, IP hash, consistent hash, random, power of two choices.
5. **Что такое sticky session?** — Клиент → всегда тот же backend; нужен для stateful; лучше избежать (сделать stateless).
6. **Как LB узнаёт что backend жив?** — Health check (TCP / HTTP / custom) с interval + threshold.
7. **Что такое SSL termination?** — LB расшифровывает HTTPS, дальше внутри кластера HTTP.
8. **Что такое K8s Service, тип LB?** — L4 LB на iptables/IPVS через kube-proxy.
9. **Что такое Ingress?** — Declarative HTTP-роутер в K8s, реализуется Ingress Controller.
10. **Spring Cloud LoadBalancer — что это?** — Client-side LB в приложении (замена Ribbon), интегрируется с Feign/RestTemplate.
11. **Что такое graceful drain?** — Убрать backend из пула, подождать in-flight, потом убить.
12. **Consistent hashing — зачем?** — Минимизировать перераспределение при добавлении/удалении сервера; для кэшей.
13. **Разница nginx и HAProxy?** — nginx больше про reverse proxy + static, HAProxy специализация на балансировке.
14. **Что такое Envoy?** — L7 proxy, основа service mesh, dynamic конфиг через xDS.
15. **Как выбрать алгоритм?** — Round-robin если equal servers; least-connections если разные времена обработки; IP hash для sticky (без cookies).

---

## Итог

- **LB распределяет трафик**: HA + scaling + health checks.
- **L4** = TCP/UDP (K8s Service, NLB); **L7** = HTTP (nginx, Ingress, ALB).
- **Server-side** (nginx) + **client-side** (SC LoadBalancer / Feign) часто оба вместе.
- **Round Robin** — default; **Least Connections** — умнее; **IP Hash** — для sticky без cookies.
- **Health checks** — must; HTTP лучше TCP.
- **SSL termination** на LB — стандарт.
- **Graceful drain** через K8s + Spring Boot shutdown.
- В ИСНА цепочка: **nginx-ingress → K8s Service → gateway → Consul (client-side LB) → микросервисы**.

---

## Финальный итог всех блоков

Все файлы в `C:\Users\berik\Desktop\work projects\isna-theory\`:

**Foundation** (01-11):
- 01 обзорный
- 02-04 — Java базы (JVM, Gradle, JAR)
- 05-08 — Spring (IoC, Boot, servers, startup)
- 09-11 — инфра (Docker, K8s, Consul)

**JPA/Hibernate** (12-15):
- 12 JPA основы
- 13 Hibernate внутри
- 14 Spring Data JPA
- 15 Производительность

**Java 11→21** (16-19):
- 16 Синтаксис
- 17 JVM/GC
- 18 API/миграция
- 19 Virtual Threads

**RabbitMQ** (20-23):
- 20 AMQP основы
- 21 Гарантии доставки
- 22 Spring AMQP
- 23 Прод-паттерны

**Security** (24-27):
- 24 Spring Security
- 25 OAuth2/OIDC
- 26 Keycloak
- 27 SS + Keycloak

**PostgreSQL / Highload / LB** (28-31):
- 28 PG внутри
- 29 PG + HikariCP
- 30 Дорогостоящие операции
- 31 Load Balancer

Всего **31 файл теории**, около 300+ страниц. Для собеседования middle Java — покрывает весь стек ИСНА.
