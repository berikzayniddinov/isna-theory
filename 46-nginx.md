# 46. nginx подробно

Что такое nginx, зачем, как конфигурировать.

---

## 1. Что такое nginx

**nginx** (произносится "engine-x") — HTTP-сервер + reverse proxy + load balancer.

Ключевые характеристики:
- **Event-driven** (async I/O) — тысячи соединений одним worker'ом.
- Написан на C — быстрый, малое потребление RAM.
- Open source + commercial (**nginx Plus**).
- Модульная архитектура.

Разработан в 2004 (Игорь Сысоев) как альтернатива Apache HTTP Server (C10K problem).

---

## 2. Типовые роли

### 2.1 Web server (static)

Отдавать static content (HTML, CSS, JS, images).

Быстрый: 10k+ req/sec на одном ядре.

### 2.2 Reverse proxy

Принимать HTTP → проксировать на backend (Java/Node/PHP приложение).

```
Client → nginx (443) → backend (8080)
```

Зачем:
- SSL termination.
- Кеширование.
- Compression.
- Rate limiting.
- Скрыть backend.

### 2.3 Load balancer (L7)

Распределять между несколькими backend'ами. См. файл `31-load-balancer.md`.

### 2.4 API gateway

Basic routing, auth, rate limit. Более продвинутое — Kong / Envoy / Spring Cloud Gateway.

### 2.5 SSL terminator

Расшифровать HTTPS, дальше HTTP внутри кластера.

### 2.6 Caching

Кеш ответов от backend, быстро отдавать повторно.

---

## 3. Архитектура (event-driven)

### 3.1 Модель процессов

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

### 3.2 Worker

Каждый worker — **single-thread**, но обрабатывает **тысячи соединений** через async I/O (epoll на Linux).

Модель Apache: **1 request = 1 thread**. Ограничение — тысячи одновременных.

Модель nginx: **1 worker = много requests**. Ограничение — десятки тысяч на worker.

### 3.3 Настройка

```nginx
worker_processes auto;              # = число CPU
worker_connections 10240;           # max connections на worker

events {
    use epoll;                       # Linux
    multi_accept on;                 # accept всех сразу
}
```

`worker_processes × worker_connections` = теоретический max одновременных соединений.

---

## 4. Конфигурация — структура

`/etc/nginx/nginx.conf` (или в контейнере — вложенное).

```nginx
# Main context (уровень процесса)
worker_processes auto;
user nginx;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    # HTTP-level настройки
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
            # для конкретного пути
        }
    }
}

stream {
    # TCP/UDP proxy
}
```

Уровни:
- **main** — глобальные.
- **events** — worker settings.
- **http** — HTTP config.
- **server** — virtual host.
- **location** — path-based.
- **stream** — TCP/UDP.

---

## 5. Reverse proxy

Базовый пример:
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

### 5.1 proxy_pass

`proxy_pass http://backend;` — куда проксировать.

### 5.2 Headers

- `Host` — оригинальный host (backend должен знать).
- `X-Real-IP` — реальный IP клиента (иначе backend видит IP nginx).
- `X-Forwarded-For` — цепочка proxies.
- `X-Forwarded-Proto` — http/https.

Без этих — backend теряет контекст.

### 5.3 Timeouts

- `proxy_connect_timeout` — установление соединения с backend.
- `proxy_send_timeout` — отправка запроса.
- `proxy_read_timeout` — ожидание ответа. Обычно самый важный (30-60 сек).

По умолчанию 60 сек — часто мало для долгих запросов.

---

## 6. Load balancing

```nginx
upstream backend {
    server 10.0.1.5:8080 weight=3;
    server 10.0.1.6:8080 weight=1;
    server 10.0.1.7:8080 backup;         # только когда основные down
    server 10.0.1.8:8080 max_fails=3 fail_timeout=30s;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

### 6.1 Алгоритмы

- **`round_robin`** (default) — по очереди.
- **`least_conn`** — least connections.
- **`ip_hash`** — sticky по IP клиента.
- **`hash <key>`** — sticky по любому ключу.
- **`random`**.
- **`least_time`** (Plus only) — least response time.

### 6.2 Health checks

Passive:
- `max_fails` — max failures в `fail_timeout` окне → backend помечается unhealthy.
- Не отправлять на unhealthy пока `fail_timeout` не пройдёт.

Active (Plus only): периодические probes.

Community modules (nginx_upstream_check_module) — активные checks в OSS.

### 6.3 Sticky sessions

`ip_hash` — по IP клиента.
`sticky cookie` (Plus only) — по cookie.

---

## 7. Static content

```nginx
server {
    location / {
        root /var/www/html;
        index index.html;
    }

    location /assets/ {
        alias /var/www/assets/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

- **root** — корень + путь URL.
- **alias** — замена префикса пути.
- **expires** — Cache-Control header.

---

## 8. SSL/TLS

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
        proxy_pass http://backend;   # ← plain HTTP внутри
    }
}

# Redirect HTTP → HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

**SSL termination** = nginx расшифровывает, backend получает plain HTTP.

**HSTS** (Strict-Transport-Security) — браузер помнит что нужен HTTPS.

Для сертификатов — **Let's Encrypt** (free, auto-renewal через certbot).

---

## 9. GZIP / Brotli

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1000;
gzip_comp_level 6;
gzip_vary on;
```

Reduces bandwidth в 5-10× для текстовых типов.

Brotli — новее, лучше (10-15% меньше), нужен модуль `ngx_brotli`.

---

## 10. Cache

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

Кеширует ответы backend. `X-Cache-Status: HIT/MISS/BYPASS/EXPIRED`.

**`proxy_cache_use_stale`** — stale content если backend упал (graceful degradation).

---

## 11. Rate limiting

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://backend;
    }
}
```

- 10 req/sec per IP.
- Burst 20 — можно накопить 20 сверх lim, но queue.
- `nodelay` — не задерживать, сразу отказывать при превышении.

Защита от DDoS / abuse.

---

## 12. Locations

Множество путей:

```nginx
location = /exact { }             # exact match
location ^~ /prefix/ { }          # prefix, приоритет над regex
location ~ ^/regex$ { }           # regex, case-sensitive
location ~* \.jpg$ { }             # regex, case-insensitive
location / { }                     # fallback
```

Приоритет: exact > `^~` prefix > regex > обычный prefix.

Try_files:
```nginx
location / {
    try_files $uri $uri/ /index.html;   # SPA fallback
}
```

---

## 13. WebSocket

```nginx
location /ws/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;               # WebSocket живёт долго
}
```

`Upgrade` header — переключение HTTP → WebSocket.

---

## 14. Логи

Access log format:
```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for" '
                'upstream=$upstream_addr rt=$request_time uct=$upstream_connect_time';

access_log /var/log/nginx/access.log main;
error_log /var/log/nginx/error.log warn;
```

Переменные:
- `$remote_addr` — IP клиента.
- `$request` — метод + URL + версия.
- `$status` — HTTP code.
- `$request_time` — full request duration.
- `$upstream_response_time` — backend time.
- `$upstream_addr` — какой backend.

Формат — часто **JSON** для парсинга ELK:
```nginx
log_format json '{"ts":"$time_iso8601","addr":"$remote_addr","status":$status,'
                '"path":"$request_uri","rt":$request_time}';
```

---

## 15. Reload без downtime

```bash
nginx -s reload
```

Master перечитывает config, форкает новых workers с новым config, старые workers дообрабатывают текущие requests и умирают.

**Zero downtime** — normal way обновления config.

Проверка config перед reload:
```bash
nginx -t
```

---

## 16. nginx-ingress в Kubernetes

Один из самых популярных Ingress Controllers в K8s.

Читает **Ingress resources** → генерирует nginx.conf → reload.

Пример:
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

Аннотации nginx-ingress — сотни настроек (rate limit, auth, timeouts).

**В ИСНА**: nginx-ingress перед `isna-knp-gateway`.

---

## 17. nginx vs Apache

| | nginx | Apache HTTPD |
|---|---|---|
| Модель | Event-driven | Process/thread per request |
| Concurrency | 10k+ | тысячи |
| Memory | Малая | Больше |
| Static files | Отлично | Хорошо |
| Config | Простая | Больше .htaccess-магии |
| Модули | Compile-time | Dynamic |
| Reverse proxy | Стандарт | Тоже, но nginx популярнее |

Для новых проектов — nginx.

---

## 18. Диагностика

### 18.1 Проверка config

```bash
nginx -t
```

### 18.2 Логи

```bash
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

### 18.3 Метрики

Stub_status module:
```nginx
location /stub_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
}
```

Возвращает:
```
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
Reading: 6 Writing: 179 Waiting: 106
```

Prometheus exporter: **nginx-prometheus-exporter**.

### 18.4 Топ ошибок

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -rn
# 12345 200
#   234 404
#    45 500
```

### 18.5 Медленные запросы

```bash
awk '$NF > 1 {print}' access.log | head    # запросы > 1 sec
```

---

## 19. Best practices

1. **worker_processes auto** (= CPU).
2. **worker_connections** побольше (10k+).
3. **Timeouts** явные (не default).
4. **Log format** — JSON для ELK.
5. **GZIP** для текстовых.
6. **HTTPS + HSTS + TLS 1.2+**.
7. **Reload** через `-s reload`, не kill.
8. **Health checks** на upstream (`max_fails`).
9. **Rate limit** для публичных endpoints.
10. **Proxy headers** (Host, X-Real-IP, X-Forwarded-*).

---

## 20. Собесные вопросы

1. **Что такое nginx?** — Event-driven HTTP-сервер + reverse proxy + LB, написан на C.
2. **Модель nginx vs Apache?** — nginx event-driven (async, много connections в worker); Apache process/thread per request.
3. **Что такое upstream?** — Группа backend'ов для балансировки.
4. **Алгоритмы LB в nginx?** — round_robin (default), least_conn, ip_hash, hash, random.
5. **Что делает proxy_pass?** — Проксирует запрос на указанный upstream/URL.
6. **Зачем X-Real-IP, X-Forwarded-For?** — Backend получает реальный IP клиента (иначе IP nginx).
7. **Что такое SSL termination?** — nginx расшифровывает HTTPS, backend получает plain HTTP.
8. **Как настроить sticky session?** — ip_hash или sticky cookie (Plus).
9. **Что такое proxy_read_timeout?** — Максимум ожидания ответа от backend.
10. **Как reload config без downtime?** — `nginx -s reload`; master forkает новых workers с новым config.
11. **Что такое nginx-ingress?** — K8s Ingress Controller на базе nginx; читает Ingress resources → генерирует nginx.conf.
12. **Как ограничить rate?** — `limit_req_zone` + `limit_req`.
13. **Как кэшировать?** — `proxy_cache_path` + `proxy_cache`.
14. **Что даёт `keepalive`?** — Переиспользование TCP-соединений между запросами.
15. **Как посмотреть кто самый частый клиент?** — `awk '{print $1}' access.log | sort | uniq -c | sort -rn | head`.

---

## Итог

- **nginx** = event-driven, много connections на worker.
- **Reverse proxy + LB + SSL + cache** — типовые роли.
- **worker_processes auto + worker_connections 10k+** = базовая настройка.
- **upstream** для балансировки.
- **Proxy headers** обязательно.
- **Reload без downtime** = стандарт.
- **nginx-ingress** = ingress controller в K8s.
- Access.log в **JSON** для ELK.

Следующий (последний) — `47-infra-glossary.md`.
