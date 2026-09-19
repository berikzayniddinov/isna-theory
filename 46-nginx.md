# 46. nginx: event-driven HTTP-сервер, reverse proxy, load balancer

## Зачем понимать nginx глубже базового конфига

Разработчик который впервые встречает nginx обычно видит его как «black box что-то делает с HTTP». Копирует конфиг из туториала, менеджит через `nginx -s reload`, надеется что работает. И часто действительно работает — nginx крайне robust продукт с разумными defaults. Но когда система hits scale — 10000 concurrent connections, latency-sensitive traffic, SSL termination overhead, cache invalidation — понимание внутренностей становится критичным.

Разница между инженером «использующим nginx» и «понимающим nginx» очевидна в production incidents. Первый видит «502 Bad Gateway», перезапускает nginx, надеется. Второй знает что 502 значит что nginx получил invalid response от backend — usually connection closed unexpectedly, upstream не отвечает в proxy_read_timeout, protocol mismatch — и проверяет конкретные вещи. Знает что worker_processes auto создаёт worker per CPU core plus каждый worker — single-threaded event loop через epoll обслуживающий тысячи concurrent connections. Знает разницу между master и worker процессами и как reload работает через forking new workers без drop traffic. Знает что access.log в JSON format сильно упрощает ELK parsing по сравнению с default combined format.

В этом файле разберём nginx глубоко. Fundamental architectural choice — event-driven vs process-per-request и почему это критично для performance. Модель master-worker процессов. Configuration structure с контекстами. Reverse proxy mechanics — что реально происходит при proxy_pass с headers, timeouts, keepalive. Load balancing algorithms и sticky session mechanics. SSL/TLS termination с session caching, HTTP/2. GZIP compression trade-offs. Proxy cache для reducing backend load. Rate limiting механика через leaky bucket algorithm. Location matching priority — часто источник bugs. WebSocket proxying. Logs plus troubleshooting. nginx-ingress в Kubernetes как automation layer.

## Event-driven architecture: почему это работает

nginx originally designed by Игорь Сысоев в 2004 году как ответ на C10K problem — как обслуживать 10000 concurrent connections на одном сервере. Apache HTTPD в то время использовал process/thread-per-connection model — каждый connection занимал OS process или thread с ~1 MB stack overhead. 10000 connections требовали 10 GB памяти только на stacks плюс context switch overhead убивал throughput.

nginx выбрал fundamentally different approach — event-driven single-threaded workers. Каждый worker — one thread обрабатывающий thousands connections concurrently через async I/O primitives (epoll on Linux, kqueue on BSD, IOCP on Windows).

Как это работает mechanistically. Worker calls epoll_wait — kernel returns list of file descriptors ready для I/O (readable data available, writable без blocking). Worker iterates через ready FDs — reads request, writes response, всё без blocking. Never sleeps на individual connection — always doing work либо waiting on epoll для ready events.

Contrast с Apache. Apache thread accepts connection, reads request (blocking), writes response (blocking). While thread waits на I/O — thread idle consuming stack memory но not doing work. 10000 concurrent connections require 10000 threads mostly idle.

nginx worker обслуживает 10000+ connections на one thread без per-connection memory overhead. Only work-in-progress connections consume meaningful resources. Idle connections just entries в epoll watch list.

Trade-offs. nginx code must be strictly non-blocking. Any blocking call (disk I/O, DNS lookup, backend call) blocks entire worker — thousands of connections stalled. Requires careful design и specific system calls (aio, sendfile) для avoiding blocking.

Apache easier to program — traditional blocking code. But scale issues fundamental.

## Master-worker модель процессов

Реальная nginx deployment includes multiple processes coordinated together:
```
┌─── master process (root) ──────────────┐
│  Читает конфиг                          │
│  Управляет workers                      │
│  Логирует                               │
└────────┬────────────────────────────────┘
         │  fork
    ┌────┼─────┬─────┬─────┐
    ▼    ▼     ▼     ▼     ▼
┌────────┐ ┌────────┐  ...
│worker 1│ │worker 2│  (обычно = число CPU)
│(non-   │ │(non-   │
│block   │ │block   │
│ I/O)   │ │ I/O)   │
└────────┘ └────────┘
```

Master process runs as root usually — needs privileges для binding privileged ports (80, 443). Reads configuration files. Manages workers — starts, stops, monitors, restarts if crashed. Handles signals для reload и graceful shutdown. Rarely does actual HTTP work.

Worker processes run as unprivileged user (nginx или www-data). Each worker — single thread с event loop. Multiple workers utilize multiple CPU cores — one worker doesn't scale beyond one core because single-threaded.

worker_processes auto (default recommended) creates one worker per CPU core. On 8-core machine — 8 workers всего. Load balancing between workers happens at OS level — kernel distributes new connections between listening workers через socket sharding или similar mechanisms.

worker_connections controls max concurrent connections per worker. Default 1024 — тоо low для serious production. Realistic values 10240 или больше. worker_processes × worker_connections = theoretical maximum concurrent connections.

Reload без downtime — важная feature для zero-downtime deployments. `nginx -s reload` sends signal к master. Master perезагружает config. Master forkает new workers с new config. Sends signal к old workers чтобы прекратили accepting new connections. Old workers finish in-flight requests. Old workers exit gracefully. Смена configuration без drop existing connections.

Проверка config перед reload через `nginx -t`. Validates syntax plus semantics without applying. Should always run before reload — избежать deploying broken config.

## Configuration structure

Configuration organized в hierarchical contexts:
```nginx
# Main context (process level)
worker_processes auto;
user nginx;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    # HTTP-level settings
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    sendfile on;
    keepalive_timeout 65;

    upstream backend {
        # ...
    }

    server {
        # virtual host
        listen 80;
        server_name example.com;

        location / {
            # for specific path
        }
    }
}

stream {
    # TCP/UDP proxy
}
```

Contexts nested. Main context — global settings applying весь process. events context — worker settings (connections per worker, use of specific event mechanisms). http context — все HTTP configuration. server contexts — virtual hosts (different domains handled на same nginx instance). location contexts — path-specific configuration. stream context для TCP/UDP proxying (non-HTTP protocols).

Directives inherit from outer contexts unless overridden в inner. keepalive_timeout defined в http applied к всем server blocks unless server overrides.

sendfile on enables zero-copy file serving — kernel sends file bytes directly к socket без copying через user space. Massive performance boost для static content serving.

## Reverse proxy mechanics

Base setup:
```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
    }
}
```

Что происходит при request. nginx receives HTTP request. Matches location block based on URL. proxy_pass directive triggers proxying — nginx opens connection к backend (или reuses keep-alive connection). Sends modified request. Receives response. Sends к client.

Headers manipulation critical. Backend without proper headers loses context.

Host header preservation. proxy_set_header Host $host preserves original Host from client. Without этого — backend receives «localhost» или whatever proxy_pass targets. Applications с virtual hosts break.

X-Real-IP transmits actual client IP. Backend sees nginx IP как $remote_addr — original client IP lost. X-Real-IP header sets к $remote_addr в nginx (real client from nginx perspective).

X-Forwarded-For chains через multiple proxies. If request already has X-Forwarded-For, nginx appends its client IP. Traces full path через proxy chain. $proxy_add_x_forwarded_for automatically appends.

X-Forwarded-Proto tells backend original scheme. Client uses HTTPS to nginx, nginx uses HTTP к backend. Application logic (например redirect URLs) requires knowing original scheme.

Без этих headers backend loses critical context — real client IP, original hostname, original protocol. Bugs manifest в logging (wrong IP), security decisions (incorrect authorization based on IP), URL generation (wrong scheme).

Timeouts control patience. proxy_connect_timeout controls TCP connection establishment. If backend not responding к connection attempt within timeout — connection fails. Default 60 seconds usually too long — 10 seconds better для fast fail.

proxy_send_timeout controls sending request body. Rarely limiting factor unless large uploads.

proxy_read_timeout controls waiting для response from backend. Most important timeout usually. Default 60 seconds — sometimes too short для legitimate long operations. Configure based on actual workload.

proxy_http_version 1.1 required для keepalive upstream connections. Default 1.0 creates new connection per request к backend — massive overhead. 1.1 permits keep-alive reuse.

## Upstream и load balancing

```nginx
upstream backend {
    server 10.0.1.5:8080 weight=3;
    server 10.0.1.6:8080 weight=1;
    server 10.0.1.7:8080 backup;
    server 10.0.1.8:8080 max_fails=3 fail_timeout=30s;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

Upstream block defines pool backends. server directives list individual endpoints с parameters.

weight allocates traffic proportionally. weight=3 versus weight=1 means server gets 3× traffic. Useful для heterogeneous hardware или canary deployments (5% weight на new version).

backup marker excludes server from normal rotation. Only receives traffic when all primary servers unavailable. Fallback capacity.

max_fails и fail_timeout implement passive health checks. Server marked unhealthy after max_fails failures within fail_timeout window. Excluded from rotation until fail_timeout elapses. Automatic recovery attempts.

Algorithms для balancing. round_robin default — simple rotation. least_conn routes к server with fewest active connections — better distribution for varying request costs. ip_hash routes based on client IP hash — sticky sessions без cookies. hash <key> permits custom hash key. random — random selection. least_time (Plus only commercial feature) — response time based routing.

Active health checks (commercial nginx Plus) — proactive probing servers. Open-source alternative через community module nginx_upstream_check_module. Passive default sufficient для many scenarios но active more responsive to failures.

Sticky sessions через ip_hash простой mechanism без state. Same client always к same backend based on IP hash. Downside — IP hash uneven если significant traffic from single IP (corporate NAT, mobile carrier). Better cookie-based sticky sessions available в Plus.

## SSL/TLS termination

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    add_header Strict-Transport-Security "max-age=31536000" always;

    location / {
        proxy_pass http://backend;
    }
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

SSL termination — nginx decrypts HTTPS traffic, forwards plain HTTP к backend. Backend receives standard HTTP requests без SSL overhead.

listen 443 ssl http2 enables both HTTPS listening и HTTP/2 protocol. HTTP/2 offers multiplexing (multiple requests through one connection), header compression, server push. Requires TLS 1.2+.

ssl_certificate и ssl_certificate_key point к certificate plus private key files. Certificate authorities issue certificates — Let's Encrypt для free automated renewal, commercial CAs для enterprise. Certificate chain sometimes needed — combine cert plus intermediate CA cert в one file.

ssl_protocols TLSv1.2 TLSv1.3 restricts to modern protocols. Older TLS 1.0 и 1.1 deprecated из-за security vulnerabilities. SSL 2 и 3 completely broken — never enable.

ssl_ciphers restrict allowed cipher suites. HIGH:!aNULL:!MD5 excludes weak ciphers. Modern recommended ciphersuites regularly updated — check current best practices через Mozilla SSL Configuration Generator.

ssl_session_cache shared:SSL:10m enables session resumption. Full SSL handshake expensive — session cache lets clients reuse existing session через session ID. Speeds up repeated connections.

Strict-Transport-Security (HSTS) header instructs browsers к always use HTTPS. Prevents downgrade attacks. max-age=31536000 (one year) told browsers to require HTTPS для one year. always modifier ensures header present даже на error responses.

HTTP-to-HTTPS redirect через separate server block listening на 80. return 301 (permanent redirect) sends browsers к HTTPS version. $host preserves domain, $request_uri preserves path plus query.

Let's Encrypt через certbot automatic renewal. Certificates valid 90 days — automatic renewal cron script keeps them fresh. Enterprise deployments может использовать longer-lived commercial certs или ACME protocol с internal CA.

## GZIP compression

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1000;
gzip_comp_level 6;
gzip_vary on;
```

Compression reduces bandwidth significantly для text content — 5-10× reduction typical для HTML, CSS, JS, JSON. Binary content (images, videos) already compressed — не benefits from gzip.

gzip_types lists MIME types к compress. Default only text/html. Explicit list includes common web content types.

gzip_min_length prevents compression small responses где overhead exceeds savings. 1000 bytes threshold reasonable — smaller responses transmit faster uncompressed.

gzip_comp_level controls compression trade-off. 1 fastest, minimal compression. 9 slowest, maximum compression. 6 balanced default.

gzip_vary adds Vary: Accept-Encoding header. Signals downstream caches что response depends on Accept-Encoding request header. Prevents caching gzipped response for client that not supports gzip.

Brotli newer alternative offering 10-15% better compression than gzip. Requires ngx_brotli module (not в base nginx). Preferred для modern deployments где module available.

## Proxy cache

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m
                 max_size=1g inactive=60m use_temp_path=off;

server {
    location / {
        proxy_pass http://backend;
        proxy_cache my_cache;
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404 1m;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503;
        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

Cache reduces backend load. Repeated identical requests served from cache without hitting backend.

proxy_cache_path defines cache storage. Filesystem path. levels=1:2 creates two-level directory structure preventing too many files в one directory (filesystem inode issues). keys_zone shared memory zone для index. max_size caps total disk usage. inactive removes unused entries after specified time.

proxy_cache activates cache для specific location. Uses named zone defined ранее.

proxy_cache_valid maps HTTP status codes to cache duration. 200 responses cached 10 minutes. 404 cached only 1 minute — allow quick recovery when content added. Status codes without explicit valid не cached.

proxy_cache_use_stale returns stale cached content when backend fails. error, timeout, updating, http_500 etc. Provides graceful degradation — backend down but cached responses still served. Better UX than errors.

X-Cache-Status header exposes cache decisions. HIT — served from cache. MISS — cache empty, fetched from backend. BYPASS — cache skipped due to conditions. EXPIRED — cached entry expired, revalidated. UPDATING — stale entry served while background update happens. STALE — serving stale due to backend failure.

Cache invalidation notoriously difficult. Options — cache_purge module (commercial) permits explicit purge. Query parameter cache busting (?v=123) invalidates когда parameter changes. Short TTLs accept staleness for eventual freshness.

## Rate limiting

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://backend;
    }
}
```

Rate limiting protects backend from abuse plus DDoS mitigation. nginx implements leaky bucket algorithm.

limit_req_zone declares shared memory zone tracking rate per key. $binary_remote_addr uses binary IP representation (smaller memory footprint than string). rate=10r/s limits к 10 requests per second per IP.

limit_req applies limit к location. burst=20 permits temporary spike above rate — 20 requests accumulated allowed. nodelay serves burst requests immediately, subsequent requests within same second rate-limited.

Без nodelay burst requests delayed to maintain average rate. Similar overall throughput но different behavior — smoothing versus immediate rejection.

Response при exceeded limit — 503 by default. Customizable через limit_req_status.

Alternative keys для limiting. $server_name limits per server. $http_authorization limits per auth token. Custom variables permit application-specific policies.

## Location matching priority

Multiple locations matched в specific priority order:
```nginx
location = /exact { }             # exact match
location ^~ /prefix/ { }          # prefix, приоритет над regex
location ~ ^/regex$ { }           # regex, case-sensitive
location ~* \.jpg$ { }             # regex, case-insensitive
location / { }                     # fallback
```

Priority sequence. First — exact match `location = /path` — highest priority. Path exactly matches — no further evaluation. Second — prefix with `^~` modifier — takes priority over regex. If matches — skip regex evaluation. Third — regex matches (case-sensitive `~` or case-insensitive `~*`) evaluated в order defined. First matching regex used. Fourth — longest prefix match without `^~`.

Common bug — confusing priorities. Regex defined перед prefix but prefix `^~` takes priority. Order в config not evaluation order (except для regex among themselves).

Try_files directive для fallback logic:
```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

Common SPA pattern. Tries file matching URI. If not exists — tries directory. If neither — falls back к /index.html allowing SPA routing.

## WebSocket

```nginx
location /ws/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;
}
```

WebSocket starts as HTTP request with Upgrade header. Client requests protocol switch. Server accepts with 101 Switching Protocols. Connection становится bidirectional message stream.

nginx must pass through Upgrade и Connection headers correctly. Default proxy headers strip these — WebSocket handshake fails.

$http_upgrade variable — value of client's Upgrade header. Passed through. Connection: upgrade tells backend protocol switch requested.

proxy_read_timeout increased significantly. WebSocket connections live long — default 60 seconds закрывает idle connections. 1 hour (3600s) или больше for typical WebSocket usage.

## Logs

Access log format critical для debugging:
```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for" '
                'upstream=$upstream_addr rt=$request_time uct=$upstream_connect_time';

access_log /var/log/nginx/access.log main;
error_log /var/log/nginx/error.log warn;
```

Variables provide rich context. $remote_addr client IP. $request full request line (method plus URL plus version). $status HTTP response code. $request_time full request duration including nginx processing. $upstream_response_time time waiting for backend. $upstream_addr which backend served the request. $upstream_connect_time time establishing backend connection.

Timing metrics essential для troubleshooting. request_time высокий но upstream_response_time низкий — nginx overhead. Оба высокие — backend slow. Contradiction indicates network issue.

JSON format для ELK ingestion:
```nginx
log_format json '{"ts":"$time_iso8601","addr":"$remote_addr","status":$status,'
                '"path":"$request_uri","rt":$request_time}';
```

Machine-parseable directly в structured search. Better для centralized logging.

## nginx-ingress в Kubernetes

nginx-ingress один of most popular Ingress Controllers. Reads Ingress resources — generates nginx.conf — reloads:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: knp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
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
                name: isna-knp-gateway
                port: { number: 8080 }
```

Controller runs как pod в K8s cluster. Watches Ingress resources через K8s API. When Ingress changes — generates new nginx.conf reflecting current state. Reloads nginx для applying config.

Annotations provide nginx-specific configuration. Hundreds available — rate limiting, custom timeouts, authentication, request rewriting, WAF rules. Extension mechanism для surfacing nginx features через K8s API.

TLS certificates stored в Kubernetes Secrets. Referenced from Ingress. Controller loads certs into nginx configuration automatically. Renewal через external tools (cert-manager) integrated seamlessly.

В КНП — nginx-ingress перед isna-knp-gateway. Standard architecture для external traffic ingress. Kubernetes-native way managing HTTP routing.

## Диагностика

nginx -t validates config без applying. Always run перед reload deploying to production.

tail -f /var/log/nginx/access.log показывает real-time traffic. Analyzing patterns — which paths popular, error rates, response times.

Error log с level warn или above shows problematic issues. debug level generates massive output — only enable during troubleshooting.

stub_status module для basic metrics:
```nginx
location /stub_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
}
```

Returns simple text metrics — active connections, requests handled, reading/writing/waiting counts. Basic health indicator.

Prometheus exporter — nginx-prometheus-exporter — scrapes stub_status output plus Plus module metrics (if commercial). Standard monitoring integration.

Access log analysis via basic Unix tools:
```bash
awk '{print $9}' access.log | sort | uniq -c | sort -rn
# 12345 200
#   234 404
#    45 500
```

Top HTTP status codes distribution. Elevated 5xx indicates backend problems. Elevated 4xx often client issues.

Slow requests identification:
```bash
awk '$NF > 1 {print}' access.log
```

Requests taking more than 1 second. Investigation candidates.

## Best practices

worker_processes auto matches CPU count. Rarely need to override.

worker_connections 10240+. Default 1024 too restrictive для serious production.

Timeouts explicit for all proxy operations. Never rely на defaults для business-critical paths.

Log format JSON для ELK integration. Structured logs дramatically easier to analyze.

GZIP enabled для text content. Bandwidth savings substantial.

HTTPS enforced с HSTS. TLS 1.2+ only. Strong ciphers only.

Reload через `nginx -s reload` not restart. Zero downtime deployments.

Health checks configured for upstream — max_fails, fail_timeout. Automatic failure detection.

Rate limiting on public endpoints. DDoS mitigation.

Proxy headers set correctly — Host, X-Real-IP, X-Forwarded-For, X-Forwarded-Proto. Backend context preserved.

## Итоги

nginx event-driven architecture обслуживает thousands concurrent connections на one thread per worker. Master-worker модель.

Configuration hierarchical — main, events, http, server, location contexts. Directives inherit unless overridden.

Reverse proxy mechanics require correct headers для preserving client context. Timeouts explicit. keepalive backend connections через HTTP/1.1.

Upstream block defines backend pool. Multiple algorithms — round_robin, least_conn, ip_hash. Health checks passive (max_fails) or active (Plus only).

SSL/TLS termination с HTTP/2 support. Session caching для performance. HSTS для security. Let's Encrypt для free certs.

GZIP compression 5-10× reduction для text content. Threshold plus level tuning.

Proxy cache reduces backend load. proxy_cache_use_stale provides graceful degradation. Cache invalidation notoriously difficult.

Rate limiting через leaky bucket algorithm. Per-IP или custom keys.

Location matching complex priority — exact, prefix с ^~, regex, longest prefix. Common source of bugs.

WebSocket proxying requires Upgrade/Connection headers passthrough plus long timeouts.

Logs configurable через log_format. Timing variables plus JSON format для ELK.

nginx-ingress в Kubernetes automates nginx config generation from Ingress resources. Annotations expose nginx features. Standard external traffic entry point.

Reload zero-downtime через master forking new workers. nginx -t validation обязательно.

Дальше — glossary инфраструктурных терминов для reference across различных technologies.
