# 117. OAuth 2.0 и OpenID Connect: протокол глубоко

## Зачем это знать

Ты пишешь backend. Пользователь заходит в приложение — надо понять кто он и что ему можно. Раньше было просто: логин + пароль в форму, session cookie, всё в одном сервере. Сейчас всё сложнее. Пользователь входит через Google, Facebook, корпоративный SSO. Frontend отдельно от backend'а, mobile app отдельно. Микросервисы должны понимать что за пользователь пришёл. Third-party приложение хочет доступ к твоему API от имени пользователя. Всё это — не про пароли, а про **делегирование доступа**.

**OAuth 2.0** — это протокол делегирования доступа (не аутентификации!). **OpenID Connect (OIDC)** — надстройка над OAuth 2.0 добавляющая аутентификацию. Разница фундаментальна и её путают чаще всего.

Разница между «использую Spring Security с Keycloak» и «понимаю OAuth/OIDC» — способность за минуту ответить: зачем существуют разные flows и когда какой применять. Что такое `code`, `token`, `id_token` — три разных ответа сервера. Почему Implicit flow deprecated и что вместо. Что реально делает PKCE и зачем. Как выглядит правильный refresh token flow. Почему нельзя использовать `access_token` для идентификации пользователя. Разница между `scope` и `role`. Что такое `state` параметр и почему без него — CSRF-уязвимость.

Разберём: зачем появился OAuth (проблема которую решает) — не про replace пароля, а про делегирование. Четыре роли протокола (Resource Owner, Client, Authorization Server, Resource Server) — кто что делает. Основные grant types (flows): Authorization Code (главный), Client Credentials (M2M), Refresh Token, Device Code, deprecated (Implicit, Password Grant). Authorization Code + PKCE — современный стандарт для SPA/mobile. Access token, refresh token, id token — три разных, для чего каждый. OIDC как надстройка — что добавляет к OAuth. Discovery endpoint, JWKS, introspection. Scopes vs claims vs roles. Common pitfalls — CSRF без state, hardcoded secrets в клиенте, refresh token без rotation, revocation проблема. Реальные сценарии в enterprise.

JWT детально — в 118. Keycloak как реализация — в 119. Spring Security integration — в 120. Здесь фокус на самом протоколе OAuth/OIDC.

## Зачем появился OAuth: реальная проблема

До OAuth типовой сценарий выглядел так. Твоё приложение хочет прочитать email пользователя из его Gmail. Как получить доступ? Единственный вариант — попросить у пользователя его пароль от Google. Пользователь вводит в твоё приложение свои Google credentials, приложение логинится под ним и читает email.

Проблемы этого подхода:

- **Приложение видит пароль пользователя**. Даже если ты честный — password compromise на твоей стороне = compromise Google-аккаунта.
- **Полный доступ**. Приложение хочет читать email, но получает **полный** доступ к аккаунту — может удалить письма, поменять пароль, отписаться от services. Нет granularity.
- **Нет отзыва**. Пользователь захотел «отобрать» доступ у приложения — единственный способ поменять Google-пароль (что сломает все другие сервисы).
- **Не audit-friendly**. Google не видит что действие сделало третье приложение — видит только «пользователь X сделал action Y».

**OAuth решает эту проблему**. Идея: пользователь **не даёт свой пароль** третьему приложению. Вместо этого:

1. Приложение перенаправляет пользователя на **сервер Google** (не своё).
2. Пользователь логинится **напрямую в Google** (Google видит пароль, не приложение).
3. Google спрашивает «Приложение X хочет доступ к вашей почте (только чтение). Разрешить?»
4. Пользователь соглашается → Google выдаёт приложению **токен доступа** (не пароль!).
5. Приложение использует токен для API-запросов к Google от имени пользователя.

Что получается:
- **Приложение не видит пароля** — Google видит.
- **Доступ ограничен**: только чтение почты, ничего больше.
- **Отзыв возможен**: пользователь заходит в Google settings → «отобрать доступ у X» → токен становится невалидным.
- **Audit**: Google логирует «действие сделано через приложение X».

Это фундаментальная идея OAuth — **делегирование ограниченного доступа третьему приложению без передачи credentials**.

Важно: **OAuth — не про аутентификацию пользователя в твоём приложении**. OAuth про доступ к чужому API от имени пользователя. Аутентификация — задача OpenID Connect (см. ниже).

## Четыре роли в OAuth

Протокол оперирует четырьмя ролями. Понимание кто что делает — половина понимания протокола.

**Resource Owner (RO)** — пользователь. Владелец данных, которые кто-то хочет получить. В нашем примере — владелец Gmail-ящика.

**Client** — приложение которое **хочет** доступ к данным пользователя. Твоё приложение хотящее читать email. Важно: Client — не user'ский клиент, а именно приложение-запросчик доступа.

**Resource Server (RS)** — сервер где лежат данные. Gmail API. Хранит защищённые ресурсы, проверяет токены доступа.

**Authorization Server (AS)** — сервер выдающий токены. Google's OAuth server (`accounts.google.com`). Аутентифицирует пользователя, спрашивает разрешение, выдаёт токены Client'у.

Часто **AS и RS — одна компания** (Google, Facebook, GitHub). Но технически это разные роли. В enterprise это тоже может быть: Keycloak — AS, ваши микросервисы — RS.

**Полный сценарий с четырьмя ролями**:

```
                Твоё приложение (Client)
                       │
                       │ 1. "Хочу доступ к почте пользователя Иван"
                       ▼
              Google OAuth (Authorization Server)
                       │
                       │ 2. Redirect Иван к нам залогиниться
                       ▼
                    Иван (Resource Owner)
                       │
                       │ 3. Вводит пароль в Google, соглашается дать доступ
                       ▼
              Google OAuth (Authorization Server)
                       │
                       │ 4. Redirect Ивана обратно к тебе с "code"
                       ▼
                Твоё приложение (Client)
                       │
                       │ 5. Отправляет code Google, обменивает на токен
                       ▼
              Google OAuth (Authorization Server)
                       │
                       │ 6. Возвращает access_token
                       ▼
                Твоё приложение (Client)
                       │
                       │ 7. Использует токен для запроса к API
                       ▼
              Gmail API (Resource Server)
                       │
                       │ 8. Проверяет токен, возвращает данные
                       ▼
                Твоё приложение (Client) — получает почту
```

Ключевое: Client никогда не видит пароль пользователя. Только токен. Токен ограничен scope'ом (что можно делать) и временем жизни.

## Authorization Code Flow: основной поток

**Authorization Code** — самый распространённый и правильный flow для web-приложений с backend'ом (server-side apps).

Разберём пошагово с реальными HTTP-запросами.

**Setup**: Твоё приложение зарегистрировано в Google, получен `client_id` (публичный) и `client_secret` (приватный, хранится только на backend'е).

### Шаг 1. Client → Authorization Server: redirect пользователя

Пользователь на твоём сайте кликает «Login with Google». Твой backend делает redirect (HTTP 302) на:

```
https://accounts.google.com/o/oauth2/v2/auth
    ?response_type=code
    &client_id=YOUR_CLIENT_ID
    &redirect_uri=https://yourapp.com/callback
    &scope=openid email profile
    &state=RANDOM_STRING_ABC123
```

Параметры:
- `response_type=code` — говорим что хотим получить `authorization code` (не токен сразу).
- `client_id` — твой идентификатор.
- `redirect_uri` — куда Google вернёт пользователя после логина. Должен совпадать с зарегистрированным в Google.
- `scope` — какие разрешения запрашиваем. `openid email profile` = OIDC login + email + basic profile.
- `state` — случайная строка, защита от CSRF. Backend её генерирует, сохраняет в сессии, потом сверяет при возврате.

### Шаг 2. Пользователь на Google

Браузер пользователя открыт на `accounts.google.com`. Пользователь:
1. Логинится в Google (если ещё не залогинен) — вводит пароль **Google видит, ты нет**.
2. Google показывает consent screen: «Приложение X просит доступ к: email, profile. Разрешить?».
3. Пользователь нажимает «Allow».

### Шаг 3. Authorization Server → Client: redirect с code

Google делает redirect пользователя обратно на `redirect_uri` с параметрами:

```
https://yourapp.com/callback
    ?code=SPLX10z3z8gCU9Fc8h3fbNc98Kf7ldF
    &state=RANDOM_STRING_ABC123
```

Твой backend получает `code` (authorization code) и `state`.

**Проверить state**! Backend сравнивает пришедший `state` с сохранённым в сессии. Не совпадает → CSRF-атака или ошибка → отвергнуть.

`code` — короткоживущий (обычно 60 секунд), одноразовый. Сам по себе бесполезен для API — надо обменять.

### Шаг 4. Client → Authorization Server: обмен code на токены

Твой backend делает POST на **token endpoint** Google (напрямую, не через браузер):

```http
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=SPLX10z3z8gCU9Fc8h3fbNc98Kf7ldF
&redirect_uri=https://yourapp.com/callback
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
```

Ключевое: **`client_secret` идёт только с backend'а**, не через браузер. Именно поэтому secret'у можно доверять.

### Шаг 5. Authorization Server → Client: токены

Google отвечает JSON'ом:

```json
{
  "access_token": "ya29.a0AfH6SMBx1234567890abcdef...",
  "expires_in": 3600,
  "refresh_token": "1//0gRlwCXtRQ8_LCgYIARAAGBASNwF...",
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjRjMTFm...",
  "scope": "openid email profile",
  "token_type": "Bearer"
}
```

Что получил Client:
- **access_token** — для запросов к API. Обычно живёт 1 час.
- **refresh_token** — для получения новых access_token'ов когда старый истёк. Долгоживущий (может месяцы).
- **id_token** — JWT с информацией о пользователе (только при OIDC `scope=openid`).
- **expires_in** — сколько секунд живёт access_token.
- **token_type** — обычно "Bearer" (значит используется как `Authorization: Bearer <token>`).

### Шаг 6. Client → Resource Server: запрос к API

Твой backend делает запрос к Gmail API:

```http
GET https://gmail.googleapis.com/gmail/v1/users/me/messages
Authorization: Bearer ya29.a0AfH6SMBx1234567890abcdef...
```

Resource Server проверяет token (валидность, expiration, scope) и возвращает данные.

**Всё**. Пользователь залогинен в твоём приложении (через id_token), твой backend имеет access_token для запросов к Gmail от его имени, refresh_token на случай истечения.

## Authorization Code + PKCE: для SPA и mobile

Проблема Authorization Code — требует `client_secret`. Backend может хранить secret. **SPA (single-page app) и mobile — не могут** (код на клиенте, декомпилируем/DevTools/etc.).

Раньше для SPA использовали **Implicit flow** (получить token напрямую в URL fragment, минуя code). Проблемы:
- Token в URL — попадает в browser history, логи прокси.
- Нет refresh token (по спецификации).
- Уязвимо к token substitution attacks.

Сейчас **deprecated**. Правильный подход — **Authorization Code с PKCE**.

**PKCE** (Proof Key for Code Exchange, произносится «пикси») — расширение Authorization Code для клиентов не имеющих secret. Идея:

1. Клиент генерирует случайную строку **code_verifier** (43-128 символов).
2. Считает **code_challenge = BASE64URL(SHA256(code_verifier))**.
3. Отправляет code_challenge при request'е code (шаг 1).
4. При обмене code на token отправляет **code_verifier** (шаг 4).
5. Authorization Server проверяет `SHA256(code_verifier) == code_challenge`.

Даже если злоумышленник перехватил `code` (через redirect URL) — не может обменять на token без code_verifier. code_verifier никогда не передаётся до шага 4, живёт только в памяти клиента.

**Пример с PKCE**:

Шаг 1 (redirect на Google):
```
https://accounts.google.com/o/oauth2/v2/auth
    ?response_type=code
    &client_id=YOUR_CLIENT_ID
    &redirect_uri=https://yourapp.com/callback
    &scope=openid email
    &state=RANDOM
    &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
    &code_challenge_method=S256
```

Шаг 4 (обмен code на token) — без secret, но с verifier:
```http
POST /token
grant_type=authorization_code
&code=SPLX10z3z8gCU9Fc8h3fbNc98Kf7ldF
&redirect_uri=https://yourapp.com/callback
&client_id=YOUR_CLIENT_ID
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

**Правило современности**: PKCE — обязателен для SPA и mobile. **Также рекомендуется для confidential clients** (backend с secret) — не помешает, добавляет defense-in-depth.

## Client Credentials Flow: M2M авторизация

Всё что было выше — **user-facing** flows. Пользователь участвует, соглашается.

Но что если один сервис (микросервис A) хочет получить доступ к другому (микросервис B)? Пользователя нет — только два сервера. **Client Credentials flow** — для этого.

Client получает токен от Authorization Server напрямую, без user interaction:

```http
POST /token
grant_type=client_credentials
&client_id=SERVICE_A_ID
&client_secret=SERVICE_A_SECRET
&scope=api:read api:write
```

Ответ:
```json
{
  "access_token": "eyJ...",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

Никакого refresh_token (клиент может получить новый когда истечёт). Никакого id_token (нет пользователя). Просто access_token представляющий **сам сервис**.

Внутри токена (если JWT) — subject будет client_id, не user_id.

**Use case**: микросервисы между собой. Batch jobs. Backend-to-backend API calls. Инструменты (Postman для API, CI/CD публикующий метрики).

## Refresh Token Flow

Access token живёт короткое время (обычно 1 час). Это ограничивает impact компрометации — украли токен, максимум час у злоумышленника. После — токен просроченный.

Но заставлять пользователя логиниться каждый час — плохой UX. **Refresh token** решает это.

При Authorization Code flow вместе с access_token выдаётся refresh_token. Долгоживущий (месяцы). Используется чтобы получить новый access_token без user interaction:

```http
POST /token
grant_type=refresh_token
&refresh_token=1//0gRlwCXtRQ8_LCgYIARAAGBAS...
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
```

Ответ — новый access_token (и часто новый refresh_token, см. rotation ниже):
```json
{
  "access_token": "ya29.NEW_ONE...",
  "expires_in": 3600,
  "refresh_token": "1//0NEW_REFRESH...",   // rotated!
  "token_type": "Bearer"
}
```

Client заменяет старый access_token на новый, продолжает работать. Пользователь ничего не заметил.

**Refresh Token Rotation** — best practice. При каждом использовании refresh_token — Authorization Server выдаёт **новый** refresh_token, старый инвалидирует. Защита: если refresh_token украли, атакующий может использовать его один раз, но следующий refresh legit-клиента вернёт «invalid refresh token» — сигнал что что-то не так, session terminated.

**Хранение refresh_token**:
- **Backend**: в БД, ассоциирован с user session. Нормально.
- **SPA**: httpOnly cookie (не доступен JavaScript). Не в localStorage (XSS-уязвим).
- **Mobile**: secure storage (Keychain iOS, Keystore Android).

## Device Code Flow: для устройств без браузера

Smart TV, консоли, CLI-tools — нет удобного способа ввести пароль. **Device Code flow**:

1. Устройство запрашивает `device_code` и `user_code` у AS:
   ```
   POST /device/code
   client_id=DEVICE_APP_ID
   scope=openid email
   ```
   Ответ:
   ```json
   {
     "device_code": "GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS",
     "user_code": "WDJB-MJHT",
     "verification_uri": "https://example.com/device",
     "expires_in": 900,
     "interval": 5
   }
   ```
2. Устройство показывает пользователю: «Зайди на example.com/device на телефоне, введи код WDJB-MJHT».
3. Пользователь идёт с телефона, вводит код, логинится, соглашается.
4. Устройство **polls** AS каждые `interval` секунд:
   ```
   POST /token
   grant_type=urn:ietf:params:oauth:grant-type:device_code
   device_code=GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS
   client_id=DEVICE_APP_ID
   ```
5. Сначала получает `authorization_pending`. Когда пользователь согласился — получает access_token.

Использование: `gcloud auth login` в CLI, Netflix на TV, `kubectl` OIDC login.

## Deprecated flows: Implicit и Password Grant

Раньше были стандартными, сейчас **не использовать**:

**Implicit Flow** (`response_type=token`). Access token возвращался прямо в URL fragment после логина, без обмена code. Проблемы: token в browser history, нет refresh token, уязвимости. Заменён на Authorization Code + PKCE.

**Password Grant** (`grant_type=password`). Client отправляет username/password пользователя напрямую AS. Полностью противоречит философии OAuth (Client видит пароль!). Был только для migration из legacy систем. **Deprecated в OAuth 2.1**.

Если видишь эти flows в новых системах — красный флаг. Использовать только для legacy интеграций.

## Три токена: access, refresh, id

Разные токены — разные назначения. Путать = уязвимость.

**Access Token** — «пропуск» для API. Отправляется в каждом запросе к Resource Server. Короткоживущий (15 мин - 1 час). Может быть opaque (случайная строка, RS должен проверять через introspection) или JWT (RS проверяет подпись локально). Содержит информацию о пользователе, scope, expiration.

Использование:
```http
GET /api/orders
Authorization: Bearer <access_token>
```

**Refresh Token** — только для получения новых access_token'ов. **Никогда** не отправляется к Resource Server. Всегда идёт только к Authorization Server через token endpoint. Долгоживущий (дни-месяцы). Обычно opaque.

**НЕ использовать refresh_token как proof of identity!** Его назначение — только refresh access_token'а.

**ID Token** — **только OIDC**. JWT с информацией о пользователе. Кто он, когда логинился, какие claims. Используется **клиентом** для аутентификации пользователя. **НЕ используется для API запросов**.

Разница access_token vs id_token — критична:
- **access_token** = «имею доступ к API» → отправляется на API.
- **id_token** = «я такой-то пользователь» → используется клиентом для user info.

Использовать id_token для API запросов — anti-pattern. API должен проверять access_token, не id_token.

## OpenID Connect: аутентификация поверх OAuth

**Проблема**: OAuth — про **делегирование доступа**, не про **аутентификацию**. OAuth говорит «у клиента есть access_token с scope X», не говорит «пользователь Y залогинился».

Многие приложения хотели именно аутентификацию — «login with Google», «login with Facebook». Пытались использовать OAuth: получил access_token, попросил `/userinfo` от Google, получил email → «пользователь залогинен». Работало, но с уязвимостями (token substitution — можно вставить чужой access_token, никак не проверить что он выдан правильному пользователю в правильный момент).

**OIDC** решает через **id_token** — специальный JWT с чётко определёнными claims об аутентификации:

```json
{
  "iss": "https://accounts.google.com",   // кто выдал
  "sub": "10769150350006150715113",        // subject = user ID
  "aud": "YOUR_CLIENT_ID",                  // для кого предназначен
  "exp": 1712345678,                        // когда истечёт
  "iat": 1712342078,                        // когда выдан
  "auth_time": 1712342000,                  // когда пользователь залогинился
  "nonce": "n-0S6_WzA2Mj",                  // от replay attacks
  "email": "user@example.com",
  "email_verified": true,
  "name": "Иван Иванов",
  "picture": "https://..."
}
```

Клиент проверяет:
- **Подпись** (JWT signed AS'ом).
- **`iss`** — правильный AS.
- **`aud`** — этот id_token для меня (мой client_id).
- **`exp`** — не истёк.
- **`nonce`** — совпадает с тем что я отправлял (защита от replay).

Если всё ок — пользователь аутентифицирован. Claims (email, name) — надёжные, подписаны AS.

**Активация OIDC** — просто `scope=openid` в OAuth request. Всё остальное — тот же OAuth Authorization Code flow, только вместе с access_token возвращается id_token.

**Разница OAuth vs OIDC простыми словами**:
- **OAuth** = «этому клиенту разрешён такой-то доступ к API».
- **OIDC** = «этот пользователь такой-то, вот доказательство».

## Discovery endpoint: как найти endpoints

OAuth не стандартизирует URL endpoints. Google — свои, Auth0 — свои, Keycloak — свои. Как клиенту узнать где что?

**OIDC Discovery** — стандартный endpoint `/.well-known/openid-configuration` на каждом OIDC provider. Возвращает JSON с URL всех endpoints:

```
GET https://accounts.google.com/.well-known/openid-configuration
```

```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "revocation_endpoint": "https://oauth2.googleapis.com/revoke",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "response_types_supported": ["code", "token", "id_token", ...],
  "scopes_supported": ["openid", "email", "profile"],
  ...
}
```

Клиент читает этот endpoint при старте, кэширует конфигурацию, использует правильные URL. Spring Security при `issuer-uri` в конфиге автоматически делает discovery.

## JWKS: как проверить подпись JWT

`jwks_uri` из discovery ведёт на **JSON Web Key Set** — публичные ключи AS. JWT подписан приватным ключом AS, проверяется публичным. Клиент/RS получает JWKS, использует для проверки.

```
GET https://www.googleapis.com/oauth2/v3/certs
```

```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "abc123",         // key ID — какой ключ подписал
      "use": "sig",
      "alg": "RS256",
      "n": "0vx7agoebGcQSuuP...",   // модуль
      "e": "AQAB"                     // экспонента
    },
    { "kty": "RSA", "kid": "def456", ... }
  ]
}
```

JWT header содержит `kid` — идентификатор ключа. Клиент находит нужный ключ в JWKS по `kid`, проверяет подпись.

**Key rotation**. AS периодически меняет ключи (безопасность). Клиент должен обновлять JWKS кэш (обычно раз в час). Если получил JWT с неизвестным `kid` — refresh JWKS, попробовать снова. Если всё равно unknown — token подделка.

Spring Security кэширует JWKS автоматически, при неизвестном kid обновляет.

## Scopes vs claims vs roles

Часто путают. Разные вещи для разных целей.

**Scopes** — что **клиент запрашивает** у AS. `openid email profile calendar.read`. AS показывает пользователю: «Client X просит доступ к: email, profile, calendar (read only)». Пользователь соглашается — access_token получает эти scopes.

Scope определяет **разрешения токена**. RS проверяет: у токена есть нужный scope для этой операции?

```java
@GetMapping("/api/calendar/events")
@PreAuthorize("hasAuthority('SCOPE_calendar.read')")
public List<Event> getEvents() { ... }
```

**Claims** — атрибуты пользователя в токене. `email`, `name`, `given_name`, `family_name`, `preferred_username`, custom claims (`department`, `tenant_id`, `permissions`). Приходят в id_token и/или через userinfo endpoint.

**Roles** — часто custom claim. Список ролей пользователя: `["ADMIN", "USER"]`. Не стандарт OAuth, но de-facto в enterprise. Keycloak имеет встроенный concept ролей, кладёт в токен.

**Разница scope vs role**:
- **Scope** — про **клиента**: что можно делать через API от лица пользователя.
- **Role** — про **пользователя**: кто он в системе.

Пример: пользователь имеет role ADMIN. Client (mobile app) запросил scope `profile email`. Access_token в mobile app содержит: user's roles=[ADMIN] И scopes=[profile, email]. Client через это app **не может** делать admin операции (нет соответствующего scope), даже если user — admin.

## Token Introspection: проверка opaque токенов

JWT токены проверяются локально (подпись). Но некоторые AS используют **opaque токены** (случайные строки без payload). Как RS проверить такой токен?

**Introspection endpoint** (RFC 7662):

```
POST /oauth2/introspect
token=ya29.a0AfH6SMBx...
```

Ответ:
```json
{
  "active": true,
  "scope": "read write",
  "client_id": "app123",
  "username": "user@example.com",
  "exp": 1712345678,
  "sub": "user_id_12345"
}
```

RS делает introspection call к AS для каждого incoming запроса. Плюсы: real-time revocation (если AS отозвал токен, introspection вернёт `active: false`). Минусы: extra network call на каждый API запрос → медленно.

Решения: короткие caches (10 сек), или использовать JWT для performance (проверка локальная, но revocation сложнее).

## Common pitfalls: реальные ошибки

**Пропущенный `state` параметр** — CSRF-уязвимость. Атакующий начинает OAuth flow со своим аккаунтом, вставляет callback URL с своим code в honest пользователя, тот "логинится" в аккаунте атакующего. `state` фиксирует: код возвращённый Google должен соответствовать сессии инициировавшего flow клиента.

**`client_secret` в frontend коде** — критическая ошибка. Всё что в JS/mobile — расшифровывается атакующим. Secret скомпрометирован → атакующий получает токены от имени клиента. Fix: PKCE вместо secret для SPA/mobile.

**Access token в URL** — попадает в browser history, referer headers, server logs. Использовать `Authorization: Bearer` header, не query параметр `?access_token=...`.

**Refresh token без rotation** — украли refresh_token, атакующий может пользоваться месяцами. Rotation каждое использование + revocation при подозрении.

**Использование id_token для API** — id_token не для API, для клиента. API проверяет access_token. Смешение = уязвимость.

**Не проверять `aud`** в id_token. Клиент получил id_token, но не проверил что он выдан **ему**. Атакующий может подсунуть id_token выданный другому клиенту.

**Не проверять `exp`** — использование истёкшего токена. Всегда проверять!

**Хранение refresh_token в localStorage** (SPA). XSS-уязвимость → атакующий читает localStorage → украл token. Использовать httpOnly cookie.

## Реальные сценарии

**Web app + backend**. Пользователь заходит на `myapp.com`. Backend делает Authorization Code flow с Keycloak. Получает id_token (проверяет — пользователь такой-то), сохраняет access_token в session на backend'е. Каждый запрос от пользователя использует cookie сессии на backend'е, backend внутренне использует access_token для API вызовов к другим микросервисам. **Стандарт для web apps**.

**SPA + backend**. Пользователь заходит на `myspa.com`. SPA делает Authorization Code + PKCE. Получает access_token, хранит в памяти (не localStorage!) или в httpOnly cookie через backend. Использует access_token для запросов к API напрямую. Refresh_token — в httpOnly cookie.

**Mobile app + backend**. Как SPA + PKCE, но токены хранятся в secure storage OS (Keychain/Keystore). Могут быть долгоживущие (недели), в отличие от web.

**Microservices**. Frontend получает access_token от Keycloak. Отправляет в API Gateway с каждым запросом. Gateway проверяет токен (JWT signature), пропускает дальше. Внутренние микросервисы либо доверяют Gateway (Zero Trust: но проверяют сами), либо тоже проверяют. Между микросервисами — Client Credentials flow для service-to-service вызовов не связанных с пользователем.

## Заключение

**OAuth 2.0** — протокол делегирования доступа. НЕ про пароль пользователя в третьем приложении, а про **токен** ограниченного доступа. Позволяет приложениям работать с чужими API от имени пользователя без передачи credentials.

**OpenID Connect** — надстройка над OAuth добавляющая **аутентификацию**. Разница: OAuth = «доступ к API»; OIDC = «кто пользователь». Активация через `scope=openid`, получается **id_token**.

**Четыре роли**: Resource Owner (пользователь), Client (приложение), Authorization Server (выдаёт токены), Resource Server (проверяет и отдаёт данные). AS + RS часто одна компания.

**Основные flows**:
- **Authorization Code** — главный для web с backend. Через code exchange с secret.
- **Authorization Code + PKCE** — для SPA/mobile без secret. code_verifier + code_challenge.
- **Client Credentials** — M2M, service-to-service. Без пользователя.
- **Refresh Token** — обновление access_token без user interaction.
- **Device Code** — для устройств без браузера (TV, CLI).
- **Deprecated**: Implicit (заменён Auth Code + PKCE), Password Grant (не использовать).

**Три токена**:
- **Access Token** — пропуск для API. Короткоживущий. Может быть JWT или opaque.
- **Refresh Token** — только для получения новых access_token'ов. Никогда не отправлять к RS.
- **ID Token** — только OIDC. JWT для аутентификации пользователя. Клиент проверяет, НЕ для API.

**Discovery endpoint** `/.well-known/openid-configuration` — стандарт для нахождения endpoints AS. JWKS для проверки JWT подписи (публичные ключи AS).

**Scopes vs Claims vs Roles**: scope = что клиент делает через API; claims = атрибуты пользователя в токене; roles = de-facto custom claim о ролях пользователя.

**Token Introspection** для opaque токенов — RS спрашивает AS «этот токен ещё активен?». Real-time revocation. Медленнее чем JWT local validation.

**Common pitfalls**: пропущенный state (CSRF), client_secret в frontend, access_token в URL, refresh token без rotation, id_token для API запросов, не проверенный aud/exp, localStorage для refresh token в SPA.

**Реальные сценарии**: web app + backend (Authorization Code standard), SPA (Code + PKCE), mobile (Code + PKCE + secure storage), microservices (Gateway checks JWT, service-to-service через Client Credentials).

JWT детально (структура, подпись, security) — файл 118. Keycloak как реализация — 119. Spring Security integration — 120. Здесь была теория самого протокола OAuth 2.0 и OIDC.
