# 25. OAuth2 / OIDC теория

Что такое OAuth2, зачем OIDC, роли, grant types, tokens, PKCE.

---

## 1. Что такое OAuth2

**OAuth2** — открытый стандарт для **делегированной авторизации**. Позволяет одному приложению получить доступ к ресурсам пользователя на другом сервисе без пароля.

Классический пример: «Войти через Google», «Разрешить приложению X читать ваши файлы Google Drive».

**OAuth2 ≠ Authentication.** OAuth2 сам по себе — не про «кто ты», а про «что тебе можно». Отсюда **OIDC** (см. §7) — расширение для authentication.

---

## 2. Четыре роли

**Resource Owner** — пользователь, владелец данных.

**Client** — приложение, которое хочет доступ к данным. Может быть:
- Web-app (backend).
- SPA (frontend в браузере).
- Native (mobile, desktop).
- Machine-to-machine.

**Authorization Server** — выдаёт токены. Проверяет пользователя. Keycloak, Okta, Auth0, Google.

**Resource Server** — API с защищёнными данными. Проверяет токены при запросах.

Пример:
- **Resource Owner** — Берик.
- **Client** — сторонний фитнес-трекер.
- **Authorization Server** — Google OAuth2.
- **Resource Server** — Google Fit API.

---

## 3. Основная схема (authorization code)

```
Пользователь         Client             Auth Server          Resource Server
    │                  │                      │                     │
    │  клик "Login"    │                      │                     │
    ├─────────────────►│                      │                     │
    │                  │  Redirect to AS      │                     │
    │                  │  (authorization_code)│                     │
    │◄─────────────────┤                      │                     │
    │                                         │                     │
    │  Логин + consent (пароль на AS)         │                     │
    ├────────────────────────────────────────►│                     │
    │                                         │                     │
    │  Redirect обратно на Client с code      │                     │
    │◄────────────────────────────────────────┤                     │
    │                  │                      │                     │
    ├─────────────────►│  code                │                     │
    │                  │                      │                     │
    │                  │  Обмен code + secret на token             │
    │                  ├─────────────────────►│                     │
    │                  │                      │                     │
    │                  │  access_token,       │                     │
    │                  │  refresh_token       │                     │
    │                  │◄─────────────────────┤                     │
    │                  │                      │                     │
    │                  │  API request с access_token                │
    │                  ├─────────────────────────────────────────►│
    │                  │                      │                     │
    │                  │  Проверка токена     │                     │
    │                  │◄─────────────────────────────────────────┤
    │                  │                                              │
    │                  │  Data                                        │
    │                  │◄─────────────────────────────────────────┤
```

Ключевые моменты:
- Client никогда не видит пароль пользователя.
- Пользователь логинится на AS (доверенный сайт).
- AS выдаёт короткоживущий access_token + долгоживущий refresh_token.
- Client несёт токен в API.

---

## 4. Grant types (типы получения токена)

### 4.1 Authorization Code (main для UI)

Классика для web-приложений с backend. Описано выше (§3).

Плюсы: пароль пользователя не проходит через client. Refresh_token остаётся на сервере.

### 4.2 Authorization Code + PKCE

Расширение для SPA/mobile (нет secure способа хранить client_secret).

**PKCE (Proof Key for Code Exchange)** — client генерирует случайную `code_verifier`, шлёт хеш `code_challenge = SHA256(code_verifier)` вместе с authorize-запросом. При обмене code на token — прикладывает оригинальный `code_verifier`. AS проверяет.

Гарантирует что перехватить authorization_code недостаточно — нужен ещё code_verifier, которого нет.

**С 2024** PKCE рекомендуется даже для web-app с secret.

### 4.3 Client Credentials (machine-to-machine)

Один сервис вызывает другой. Нет пользователя.

```
POST /token
grant_type=client_credentials
client_id=my-service
client_secret=xxx
```

Возвращается только access_token (без refresh_token, без user info).

Использование: backend-to-backend, background jobs, cron.

### 4.4 Refresh Token

Когда access_token истёк — обменять refresh_token на новый.

```
POST /token
grant_type=refresh_token
refresh_token=xxx
client_id=my-app
```

Возвращается новый access_token + иногда rotated refresh_token.

### 4.5 Password Grant (deprecated!)

Client сам собирает пароль и шлёт AS:
```
grant_type=password
username=berik
password=xxx
```

**НЕ ИСПОЛЬЗОВАТЬ** — противоречит философии OAuth2 (client видит пароль). Оставлено только для legacy.

### 4.6 Implicit Grant (deprecated!)

Для SPA: AS возвращает token прямо в URL fragment. Небезопасно. Заменён на Authorization Code + PKCE.

### 4.7 Device Code

Для TV, IoT где нет клавиатуры. Устройство показывает код, пользователь идёт на браузер, вводит.

---

## 5. Токены

### 5.1 Access Token

- Короткоживущий (обычно 5-60 мин).
- Прикладывается к каждому запросу к Resource Server: `Authorization: Bearer <token>`.
- Формат обычно **JWT** (см. §6), но может быть opaque (случайная строка).

### 5.2 Refresh Token

- Долгоживущий (дни, недели).
- Никогда не отправляется в Resource Server.
- Только на AS для получения нового access_token.
- Обычно хранится в secure storage (HttpOnly cookie / secure keychain).

### 5.3 ID Token (OIDC)

Информация о пользователе. Всегда JWT. Только для аутентификации.

---

## 6. JWT (JSON Web Token)

Формат самодостаточных токенов. Три части через точки:

```
<header>.<payload>.<signature>

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJiZXJpayIsImV4cCI6MTcwMDAwMDAwMH0.aBcDeF...
```

### 6.1 Header

Base64URL JSON:
```json
{
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key-id-1"
}
```

- `alg` — алгоритм подписи (RS256, ES256, HS256).
- `kid` — ID ключа (для нахождения в JWKS).

### 6.2 Payload (claims)

Base64URL JSON:
```json
{
    "sub": "berik",           // subject (user id)
    "iss": "https://keycloak.isna/realms/knp",  // issuer
    "aud": "isna-knp-integration",  // audience
    "exp": 1700000000,        // expiration (unix timestamp)
    "iat": 1699999000,        // issued at
    "roles": ["ADMIN", "USER"],
    "email": "berik@example.com"
}
```

Стандартные claims:
- `iss` (issuer) — кто выдал.
- `sub` (subject) — user id.
- `aud` (audience) — для кого.
- `exp` (expiration) — до когда действителен.
- `iat` (issued at) — когда выдан.
- `nbf` (not before) — с какого момента действителен.
- `jti` (JWT ID) — уникальный id для revoke.

Custom claims — что угодно (roles, permissions, tenant_id).

### 6.3 Signature

`HMACSHA256(base64url(header) + "." + base64url(payload), secret)`.

Или с RSA: private key подписывает, public key проверяет.

**Важно**: payload читаем всеми (просто base64), но подпись — гарантия что не подделан.

### 6.4 JWS vs JWE

- **JWS (JSON Web Signature)** — подпись без шифрования. Payload видно.
- **JWE (JSON Web Encryption)** — шифрованный payload. Никто кроме получателя не прочитает.

99% случаев — JWS.

### 6.5 Проверка JWT в Resource Server

1. Разбить на `header.payload.signature`.
2. Проверить `alg` (не `none` — атака!).
3. Найти public key по `kid` (обычно из JWKS-endpoint).
4. Проверить signature.
5. Проверить `exp`, `nbf`.
6. Проверить `iss`, `aud`.
7. Извлечь claims → GrantedAuthority.

Spring Security делает это через `JwtDecoder` + `NimbusJwtDecoder`.

---

## 7. OIDC (OpenID Connect)

OAuth2 говорит про авторизацию (доступ к ресурсам). Не даёт стандартного способа узнать «кто пользователь».

**OIDC** — расширение OAuth2 для **аутентификации**. Добавляет:
- **ID Token** — JWT с claims о пользователе.
- **Userinfo endpoint** — REST для запроса user info.
- **Стандартные claims** — sub, name, email, picture.
- **Discovery document** — `/.well-known/openid-configuration` с описанием AS.

Scope `openid` — обязателен для получения ID token.

Пример запроса:
```
authorize?client_id=xxx&scope=openid profile email&response_type=code&...
```

Ответ:
```json
{
    "access_token": "...",
    "refresh_token": "...",
    "id_token": "eyJ...",   // ← вот это OIDC
    "expires_in": 3600
}
```

Keycloak, Okta, Google — все OIDC-provider'ы.

---

## 8. Scopes

Ограничение прав токена:
```
scope=read:orders write:orders
```

Client запрашивает конкретные scope, пользователь одобряет. Токен содержит только выданные.

Client Credentials обычно с scope `service`.

---

## 9. Discovery endpoint

Стандартный URL:
```
https://keycloak.isna/realms/knp/.well-known/openid-configuration
```

JSON со всеми URL'ами:
```json
{
    "issuer": "https://keycloak.isna/realms/knp",
    "authorization_endpoint": "https://keycloak.isna/realms/knp/protocol/openid-connect/auth",
    "token_endpoint": "https://keycloak.isna/realms/knp/protocol/openid-connect/token",
    "userinfo_endpoint": "https://keycloak.isna/realms/knp/protocol/openid-connect/userinfo",
    "jwks_uri": "https://keycloak.isna/realms/knp/protocol/openid-connect/certs",
    "grant_types_supported": ["authorization_code", "refresh_token", "client_credentials"],
    "scopes_supported": ["openid", "profile", "email"]
}
```

Spring Security читает это автоматически:
```yaml
spring.security.oauth2.resourceserver.jwt.issuer-uri: https://keycloak.isna/realms/knp
```

---

## 10. JWKS (JSON Web Key Set)

Публичные ключи AS для проверки JWT подписей.

```
GET https://keycloak.isna/realms/knp/protocol/openid-connect/certs

{
    "keys": [
        {
            "kid": "abc123",
            "kty": "RSA",
            "alg": "RS256",
            "n": "...",
            "e": "AQAB"
        }
    ]
}
```

Spring кэширует JWKS. При появлении JWT с новым `kid` — обновляет кэш.

**Rotation**: AS периодически меняет ключи. Новые токены с новым `kid`, старые проверяются старым ключом (пока не истекут).

---

## 11. Token introspection (для opaque tokens)

Если access_token не JWT, а opaque строка — Resource Server спрашивает AS:
```
POST /introspect
token=xxx
client_id=my-service
client_secret=xxx
```

Ответ:
```json
{
    "active": true,
    "sub": "berik",
    "exp": 1700000000,
    "scope": "read:orders"
}
```

Плюсы: можно revoke мгновенно.
Минусы: сетевой round-trip на каждый запрос → медленно.

**JWT** = проверка локально, но revoke сложнее.

---

## 12. Refresh token flow

```
Client                                 Auth Server
  │                                          │
  │  access_token истёк (500 / 401)          │
  │                                          │
  │  POST /token                             │
  │  grant_type=refresh_token                │
  │  refresh_token=xxx                       │
  ├─────────────────────────────────────────►│
  │                                          │
  │  new access_token + (rotated) refresh    │
  │◄─────────────────────────────────────────┤
  │                                          │
  │  повторить оригинальный запрос          │
```

Обычно client имеет interceptor:
- 401 от Resource Server → попробовать refresh → повторить запрос.
- Если refresh упал → пользователь должен снова залогиниться.

---

## 13. Logout

- **RP-initiated logout** (OIDC): client шлёт пользователя на `end_session_endpoint`, AS завершает сессию.
- **Backchannel logout** — AS шлёт notification всем clients с уведомлением о logout (сложнее).
- Просто удалить токены на client — самое простое.

---

## 14. Common pitfalls

### 14.1 Токен в URL

```
GET /api?token=xxx
```

Токен попадает в логи (proxy, nginx), history браузера. Всегда `Authorization: Bearer`.

### 14.2 Долгий access_token

Access_token = 1 час минимум = 1 час нельзя revoke. Пример: увольнение сотрудника, компрометация. Держи короткими (5-15 мин), refresh часто.

### 14.3 Хранение токенов

- **Web-app** — HttpOnly cookie (не доступен для JS).
- **SPA** — sessionStorage / memory (уязвим к XSS!), но избегать localStorage.
- **Mobile** — secure keychain.

Никогда — в plain localStorage (XSS сразу украдёт).

### 14.4 Проверка `alg`

Атака: атакующий подписывает JWT `alg=none`. Некоторые библиотеки принимают. Всегда явно указывай допустимые `alg`.

### 14.5 Проверка `iss` и `aud`

Иначе токен от другого AS может пройти.

### 14.6 CSRF при OAuth2 flow

Параметр `state` в authorize-запросе — случайная строка. Проверяется на callback. Иначе — CSRF на OAuth flow (атакующий заставит браузер завершить OAuth от чужого имени).

PKCE тоже помогает.

---

## 15. Собесные вопросы

1. **OAuth2 vs OIDC?** — OAuth2 — авторизация (доступ к ресурсам); OIDC — надстройка для аутентификации (id_token).
2. **4 роли в OAuth2?** — Resource Owner, Client, Authorization Server, Resource Server.
3. **Grant types и когда какой?** — Authorization Code (UI), Client Credentials (M2M), Refresh Token, PKCE (SPA/mobile).
4. **Что такое PKCE?** — Расширение authorization code для клиентов без secret; code_verifier + code_challenge.
5. **Разница access_token, refresh_token, id_token?** — Access для API (короткий), refresh для обновления (долгий), id_token для инфо о пользователе (OIDC).
6. **Что такое JWT?** — Токен из header.payload.signature; payload читаем, подпись гарантирует подлинность.
7. **JWS vs JWE?** — JWS подпись без шифрования; JWE зашифрованный payload.
8. **Что такое JWKS?** — Endpoint с публичными ключами AS для проверки JWT.
9. **Что такое introspection?** — Спросить AS «валиден ли этот opaque token».
10. **Discovery endpoint?** — `/.well-known/openid-configuration` с URL'ами AS.
11. **Deprecated grant types?** — Password grant, Implicit — используй Authorization Code + PKCE.
12. **Как хранить токены на клиенте?** — Web = HttpOnly cookie; SPA = memory / sessionStorage; никогда localStorage без crypto.
13. **Что такое scope?** — Ограничение прав токена (`read:orders`).
14. **Что произойдёт если access_token истёк?** — 401 от Resource Server; client обменивает refresh на новый access.
15. **Как отзывать JWT?** — Сложно без introspection; короткий TTL + refresh + blacklist по `jti` или user_id.

---

## Итог

- **OAuth2** = делегированный доступ; **OIDC** = OAuth2 + аутентификация (id_token).
- **4 роли**: RO, Client, AS, RS.
- **Authorization Code + PKCE** = стандарт для UI.
- **Client Credentials** — machine-to-machine.
- **JWT** = header + payload + signature; base64, подпись RSA/HMAC.
- **JWKS** — публичные ключи AS.
- **Discovery** — auto-конфиг client'а.
- **Access short, refresh long**.
- Не бросай токены в URL, не храни в localStorage.

Следующий — `26-keycloak.md`.
