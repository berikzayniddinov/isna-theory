# 31. Load Balancer: виды, как работает, за что отвечает

## Что такое Load Balancer

Load Balancer (LB) это компонент распределяющий входящий трафик между несколькими серверами или инстансами приложения. Ключевая инфраструктурная составляющая любой highload системы обеспечивающая scalability и high availability.

Задачи которые решает Load Balancer охватывают несколько связанных проблем. Распределение нагрузки — предотвращение перегрузки одного сервера когда доступны другие. High availability — если один сервер упал трафик автоматически идёт на живые. Масштабирование — можно добавить или удалить инстансы приложения без прерывания сервиса. Health checking — LB следит какие серверы отвечают правильно. SSL termination — расшифровка HTTPS один раз на LB, дальше внутри кластера plain HTTP что упрощает и ускоряет обработку. Rate limiting и security функции вроде WAF (Web Application Firewall) и DDoS защиты часто интегрированы в LB.

Хорошая аналогия — банк с несколькими кассирами и один администратор направляющий клиентов к свободному кассиру. Администратор не выполняет банковские операции сам, но эффективно распределяет клиентов и знает состояние каждого кассира.

## Основные концепты

Терминология LB важна для понимания документации и обсуждений архитектуры. Frontend это то что видит клиент — VIP (Virtual IP) плюс порт на которых LB принимает соединения. Backend или upstream — реальные серверы за LB обрабатывающие запросы. Pool или target group — набор backend серверов образующих одну группу. Health check — периодическая проверка живости backend через TCP или HTTP probe. Algorithm — правило распределения запросов между backends. Session affinity или sticky session — привязка клиента к конкретному backend для stateful сценариев.

Общая схема работы:
```
                Client
                  │
                  ▼
            ┌───────────┐
            │Load       │  ← Frontend: 200.10.20.30:443 (VIP)
            │Balancer   │
            └─────┬─────┘
                  │  select backend by algorithm
     ┌────────────┼────────────┐
     ▼            ▼            ▼
  [App 1]      [App 2]      [App 3]  ← Backend pool
   10.0.1.5    10.0.1.6    10.0.1.7
```

## Разница L4 и L7

Классификация load balancers по уровню OSI где они работают.

L4 (Transport Layer) работает с TCP и UDP пакетами. Ничего не знает про HTTP или HTTPS содержимое. Смотрит только на IP адреса и порты source/destination. Пересылает пакеты не изменяя. Быстрый потому что нет парсинга application data. Может балансировать любой TCP протокол — PostgreSQL, Redis, gRPC, WebSocket.

Примеры L4 балансировщиков. AWS NLB (Network Load Balancer) для managed решения в AWS. HAProxy в TCP mode когда используется как L4 proxy. LVS (Linux Virtual Server) — kernel-level Linux решение. F5 BIG-IP LTM в L4 конфигурации для enterprise. Kubernetes Service (ClusterIP) — тоже L4 балансировка работающая через iptables или IPVS правила.

Плюсы L4. Extremely fast — миллионы пакетов в секунду достижимы. Простой internally. Работает для любого TCP-based протокола, не привязан к HTTP. Минусы. Не может делать routing по URL или headers — не понимает HTTP. SSL termination невозможен — не разбирает TLS. Не понимает retry или smart routing на уровне application logic.

L7 (Application Layer) работает с HTTP запросами. Понимает URL, headers, cookies, method. Парсит HTTP запрос полностью. Может routing по Host header (различать knp.kgd.gov.kz vs arm.kgd.gov.kz на одном IP), по URL path (/api/knp vs /api/fno), по method, headers, cookies. SSL termination поддерживается — LB имеет certificate и обрабатывает TLS. Может modify headers, rewrite URLs. Compression, caching, application-level features.

Примеры L7 балансировщиков. nginx как наиболее популярный open source. Envoy modern proxy основа service mesh (Istio, Linkerd). Traefik с хорошей K8s интеграцией. HAProxy в HTTP mode. AWS ALB (Application Load Balancer) для managed L7. Kubernetes Ingress Controller (nginx-ingress, traefik-based) для declarative K8s HTTP routing. Spring Cloud Gateway как L7 внутри Java приложения.

Плюсы L7. Богатая маршрутизация — множество опций для routing decisions. SSL termination упрощает backend. Rate limiting, WAF, application-specific features. HTTP-специфичные оптимизации — compression, caching, keepalive management. Минусы. Медленнее L4 из-за парсинга. Обычно только HTTP/HTTPS. Больше memory footprint per connection.

Практика типично комбинирует L4 перед L7:
```
Internet → NLB (L4) → ALB / nginx (L7) → App
```

L4 распределяет по нескольким L7 инстансам обеспечивая их собственную HA. L7 handles application routing. Каждый слой оптимизирован для своей роли.

## Server-side против client-side балансировки

Server-side LB это классический подход. LB отдельная сущность через которую идёт клиент:
```
Client → LB → [server1, server2, server3]
```

Плюсы включают простоту клиента — не знает про топологию, только адрес LB. Централизованное управление — конфигурация в одном месте. Легко подключать не-Java клиенты потому что нет library dependency. Минусы. LB сам становится точкой отказа — требует HA конфигурации. Extra hop — Client to LB to Server это два сетевых прыжка вместо одного. Меньше возможностей для tuning на клиенте.

Примеры включают nginx, HAProxy, AWS ALB, Kubernetes Service — все server-side LB.

Client-side LB имеет логику балансировки в самом клиенте:
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

Клиент сам знает про все backend инстансы через service registry (Consul, Eureka). Сам выбирает один по алгоритму. Соединяется напрямую с выбранным.

Плюсы. Нет extra hop — connection напрямую к server. Клиент видит топологию — может делать intelligent retry на другой instance при failure. Custom алгоритмы легко реализуются — weighted, sticky by business key. Минусы. Логика LB встроена в каждый клиент — дублирование. Library needed per язык. Больше нагрузка на service registry.

Примеры client-side LB. Ribbon от Netflix (deprecated в пользу Spring Cloud LoadBalancer). Spring Cloud LoadBalancer как современная замена Ribbon. gRPC имеет встроенный client-side LB. Feign через Spring Cloud LoadBalancer.

В КНП используются одновременно оба подхода. Server-side через nginx-ingress перед gateway для внешнего трафика. Client-side через Feign плюс Consul для internal service-to-service communication.

## Алгоритмы балансировки

Round Robin (RR) самый простой алгоритм — распределение по очереди циклически:
```
request 1 → server A
request 2 → server B
request 3 → server C
request 4 → server A
...
```

Плюсы простоты и справедливости. Минусы в неучёте реальной нагрузки — если один сервер медленнее, он получает равное количество запросов и деградирует.

Weighted Round Robin даёт каждому серверу вес отражающий его capacity:
```
A (weight=3), B (weight=1) → A A A B A A A B ...
```

Использование для серверов разной мощности, canary release где 5 процентов трафика идёт на новую версию.

Least Connections направляет на сервер с наименьшим количеством активных connections. Плюсы в учёте реальной занятости. Минусы — LB должен считать connections требуя state.

Least Response Time идёт на сервер с наименьшей средней latency. Учитывает и loading и производительность backend. Умный но требует continuous measurement.

IP Hash использует hash(client_ip) modulo N чтобы направить того же клиента всегда на тот же сервер. Плюсы — sticky session без cookies, реализуется на уровне L4. Минусы — неравномерное распределение когда один прокси создаёт много connections с одного IP.

Consistent Hashing применяет hash(request_key) mapping на server. Ключевое свойство — при добавлении или удалении сервера минимальное перераспределение (только 1/N ключей перемещаются). Использование для cache серверов Memcached, Redis Cluster где перераспределение ведёт к cache miss.

Random просто случайный выбор. Работает удивительно хорошо в практике из-за принципа Power of Two Choices — простое выбирание случайного backend даёт приемлемое распределение.

Power of Two Choices как продвинутый алгоритм — выбираются два случайных backends, из них выбирается менее загруженный. Аппроксимация Least Connections без хранения state в LB. Хороший баланс simplicity и performance.

## Sticky sessions

Session affinity или sticky session привязывает клиента к одному backend. Тот же клиент всегда попадает на тот же server.

Зачем нужны. Если backend хранит session в local memory (не в Redis или shared storage) — sessions работают только на одном instance. Пользователь после login должен продолжать попадать на тот instance где сессия создана.

Реализации. IP hash — по IP адресу клиента, самый простой но неравномерный. Cookie-based — LB устанавливает cookie например LB_SERVER=A, читает при next requests для routing. Более гибкий, работает через proxy.

Правила использования. Stateless приложения (JWT-based auth, session в Redis) — sticky sessions НЕ нужны. Sticky плохо для масштабирования — нельзя equally распределить. Смерть sticky server — все связанные клиенты теряют session. Правильный approach — сделать приложение stateless. Sticky sessions это костыль скрывающий architectural проблему.

## Health checks

LB периодически проверяет каждый backend несколькими способами.

TCP check это простейший — просто открытие TCP соединения на порт. Успешный handshake равно OK. Быстро но грубо — порт может быть открыт даже если приложение сломано.

HTTP check это стандартный для web приложений. GET /health возвращает 200-399 равно OK. Обычно проверяется /actuator/health/readiness в Spring Boot приложениях. Более точный чем TCP check потому что проверяет что application layer отвечает.

Custom check через script или custom logic для specific требований. Может проверять database connectivity, downstream services, любые critical dependencies.

Параметры health check. Interval — как часто проверять, обычно 5-30 секунд. Timeout — сколько ждать ответа. Healthy threshold — сколько успешных подряд check чтобы считать backend здоровым (обычно 2-3). Unhealthy threshold — сколько неудачных чтобы вывести из pool (обычно 2-3). Пороги предотвращают flapping — быструю смену status на transient failures.

При unhealthy backend LB перестаёт направлять на него трафик. Не убивает backend процесс — это ответственность оркестратора (Kubernetes через liveness probe). LB просто skip failing backend.

## SSL termination

Классическая архитектура:
```
Client ──HTTPS──► LB ──HTTP──► Backend
```

LB расшифровывает HTTPS одним обработчиком. Внутри кластера трафик plain HTTP что быстрее и легче для debugging и tracing. Настройка требует TLS certificate на LB, backend без SSL слушает 8080 или другой HTTP port, коммуникация LB-backend через приватную сеть (не через public internet).

Плюсы простота management сертификатов — только на LB, не на каждом backend. Better performance backends потому что нет SSL overhead. Легче observability и debugging plain HTTP.

Если требуется end-to-end TLS для compliance или security requirements — используются SSL passthrough (LB не расшифровывает, просто forwards TLS) или re-encryption (LB расшифровывает и заново шифрует к backend).

## Как это в КНП

Полная картина запроса https://knp.kgd.gov.kz/api/fno/submit проходит через несколько уровней:

```
Пользователь
    │  HTTPS 443
    ▼
[External LB / firewall]           ← L4 / L7
    │  DDoS protection, WAF
    ▼
[nginx-ingress]                     ← L7, terminates SSL, routing по host
    │  routes: knp.kgd.gov.kz → svc/isnaknpgateway
    ▼
[K8s Service: isnaknpgateway]      ← L4 (iptables/IPVS), ClusterIP
    │  round-robin по подам gateway
    ▼
[isna-knp-gateway pod]              ← Java 11, Zuul
    │  routes: /services/X → Consul lookup + client-side LB
    ▼
[Consul discovery]                  ← client-side LB (Spring Cloud LoadBalancer)
    │  дал instance: 10.0.1.5:8080
    ▼
[isna-knp-integration pod]
    │
    ├─ через Feign → Consul → isnaknpuser pod
    ├─ через Feign → Consul → isnaknpfno pod
    └─ БД / RabbitMQ
```

Каждый уровень имеет свою роль. External LB защищает от external threats и балансирует global traffic. nginx-ingress terminates SSL и routing по host header. K8s Service простой L4 balancer к подам gateway. Gateway маршрутизирует по service names. Client-side LB через Consul distribution между конкретными подами.

Реальный кейс knp-fo-consul-lb-mr1223-latent-mine из КНП memory. При миграции с Ribbon на Spring Cloud LoadBalancer изменилась case-sensitivity resolving service names. isnaKnpUser не резолвился как раньше — Ribbon был case-insensitive, Spring Cloud LoadBalancer строгий. Результат 500 ошибки. Урок — миграция даже совместимых по API компонентов может иметь semantic differences.

## Kubernetes Service как LB

K8s Service это L4 LB для подов внутри cluster. Абстракция над dynamic set подов позволяющая stable network endpoint.

Механика работы. Endpoints controller следит за подами имеющими matching labels. При добавлении/удалении пода — endpoint list обновляется. Kube-proxy на каждой ноде читает endpoint updates и создаёт iptables или IPVS правила. Правила говорят «трафик на 10.96.0.5:8080 (Service ClusterIP) — раскинуть round-robin между IP адресами подов». Балансировка происходит на kernel level без user-space proxy overhead.

Типы K8s Services. ClusterIP это internal-only Service доступный только внутри cluster через ClusterIP address. Стандартный тип для service-to-service. NodePort exposes Service на port каждой ноде — external доступ через любой node IP. LoadBalancer просит cloud provider настроить external LB (например AWS ELB) указывающий на нужные ноды. ExternalName это alias через DNS CNAME для external services.

## Kubernetes Ingress как L7 LB

Ingress это declarative HTTP routing в K8s. Ресурс определяющий rules маршрутизации, реализуется Ingress Controller (обычно nginx-ingress).

Пример Ingress ресурса:
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

Ingress Controller (например nginx-ingress) читает Ingress ресурсы через K8s API. Динамически генерирует nginx configuration reflecting rules. Reloads nginx при changes. Обеспечивает declarative HTTP routing без manual nginx configuration management.

Преимущества. Kubernetes-native — управление через k8s API и manifests. Declarative — желаемое состояние описывается, controller обеспечивает. Multiple hosts и paths на одном external IP. TLS termination централизованно. WAF, rate limiting через annotations или CRDs.

## Spring Cloud LoadBalancer

Client-side LB библиотека внутри Java приложения. Замена deprecated Ribbon.

Работает автоматически с Feign, RestTemplate, WebClient при наличии зависимости spring-cloud-starter-loadbalancer. Пример использования через RestTemplate:
```java
@Autowired RestTemplate rest;

// Используется service name а не URL
UserDto u = rest.getForObject("http://isnaknpuser/api/users/1", UserDto.class);
```

Под капотом LoadBalancerInterceptor перехватывает запрос. Резолвит isnaknpuser через ServiceInstanceListSupplier получающий список из Consul, Eureka, или K8s. Выбирает конкретный instance по алгоритму. Заменяет service name на реальный IP:port. Fixing URL и продолжает execution.

Кастомный алгоритм регистрируется как bean:
```java
@Bean
ReactorLoadBalancer<ServiceInstance> customLB(
        Environment env,
        LoadBalancerClientFactory factory) {
    String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
    return new RandomLoadBalancer(
        factory.getLazyProvider(name, ServiceInstanceListSupplier.class), 
        name);
}
```

Доступные встроенные алгоритмы — RoundRobinLoadBalancer (default) и RandomLoadBalancer. Custom реализации через ReactorLoadBalancer interface.

## Типовые продукты Load Balancer

Обзор основных tools используемых в индустрии.

nginx это L7 (HTTP) в основном, может L4 через stream module. Open source версия достаточна для большинства use cases, есть commercial nginx Plus с extra features. Легковесный, быстрый. Хороший для static content serving плюс reverse proxy. Самый популярный в industry.

HAProxy это L4 плюс L7 балансировщик специализирующийся именно на балансировке (в отличие от nginx который также web server). Богатые метрики через statistics interface. ACLs (Access Control Lists) для sophisticated routing. Используется в местах требующих serious load balancing capability.

Envoy это modern L7 proxy являющийся основой service mesh платформ Istio и Linkerd. Мощная динамическая конфигурация через xDS API — configuration updates без restart. Observability first — extensive metrics и tracing из коробки. Полезен для больших microservices deployments.

Traefik это L7 балансировщик с хорошей интеграцией с K8s и Docker. Автоматически discovers backends через labels в container orchestrators. Проще в конфигурации чем nginx для K8s environments.

AWS ALB, NLB, GLB это managed решения от AWS. ALB (Application Load Balancer) L7. NLB (Network Load Balancer) L4 с extreme performance. GLB (Global Load Balancer) globally distributed anycast для world-wide applications. Не надо обслуживать infrastructure — cloud provider handles.

F5 BIG-IP это enterprise решение hardware plus software. Дорогое но feature-rich — WAF, DDoS protection, SSL offload с hardware acceleration. Используется в enterprise environments где стоимость оправдана требованиями compliance или performance.

## Graceful drain

При удалении backend (deploy new version, scale down) важно не потерять in-flight requests.

Плохой подход. Kill process instantly через SIGKILL. Все in-flight requests теряются с 500 errors. Пользователи получают ошибки. Retry на клиенте может помочь но лучше избежать проблемы вовсе.

Хороший подход через graceful drain. Убрать backend из LB pool — LB перестаёт направлять new requests. Дать time для дообработки in-flight requests (30-60 секунд обычно достаточно). Только после этого убить процесс.

Kubernetes делает это автоматически при удалении пода. Pod переходит в Terminating state. Endpoints controller убирает pod IP из Service endpoints — LB перестаёт направлять трафик. SIGTERM отправляется процессу — приложение должно gracefully shutdown. terminationGracePeriodSeconds (default 30) определяет сколько ждать до SIGKILL. Если приложение завершилось раньше — pod удаляется сразу.

Spring Boot имеет built-in graceful shutdown:
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

При shutdown Spring Boot прекращает принимать новые HTTP requests, дообрабатывает in-flight, потом завершает процесс. Синхронизировано с K8s termination sequence — timeout-per-shutdown-phase должен быть меньше terminationGracePeriodSeconds для правильного порядка.

## Итоги

Load Balancer распределяет трафик обеспечивая high availability и scaling. Health checks определяют работающие backends. SSL termination упрощает backend infrastructure.

L4 против L7 разные уровни. L4 быстрый и универсальный для любых TCP protocols. L7 понимает HTTP давая rich routing capabilities но медленнее. Обычно комбинируются — L4 перед L7 для их HA.

Server-side против client-side LB. Server-side (nginx, HAProxy) традиционный подход. Client-side (Spring Cloud LoadBalancer, gRPC) убирает extra hop и даёт больше гибкости. В enterprise часто оба используются одновременно.

Алгоритмы. Round Robin default простой. Least Connections умнее но требует state. IP Hash для sticky без cookies. Consistent Hashing для cache scenarios. Power of Two Choices как efficient approximation.

Sticky sessions необходимы для stateful backends но лучше делать stateless приложения. Sticky это костыль скрывающий architectural проблему.

Health checks обязательны. HTTP лучше TCP потому что проверяет application layer. Пороги предотвращают flapping.

SSL termination на LB — стандарт для simplicity. E2E TLS для compliance requirements через passthrough или re-encryption.

В КНП цепочка nginx-ingress → K8s Service → gateway → Consul (client-side LB) → микросервисы. Каждый слой имеет свою role.

K8s Service это L4 через iptables/IPVS. Ingress это declarative L7 через Ingress Controller (nginx-ingress наиболее популярный).

Spring Cloud LoadBalancer заменил Ribbon. Автоматически интегрируется с RestTemplate, WebClient, Feign. Custom алгоритмы возможны.

Типовые products. nginx самый популярный. HAProxy специализированный. Envoy modern для service mesh. Traefik K8s-friendly. Cloud managed (AWS ALB/NLB) для cloud environments. F5 BIG-IP enterprise.

Graceful drain обязателен для zero-downtime deployments. K8s + Spring Boot shutdown работают together.

Дальше — фундаментальная тема transactions ACID isolation propagation как основа @Transactional понимания.
