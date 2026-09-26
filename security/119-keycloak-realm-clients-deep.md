# 119. Keycloak глубоко: realm, clients, users, roles, federation, admin

## Зачем это знать

Ты открываешь Keycloak admin console. Видишь: Realm dropdown в углу, Users, Groups, Roles, Clients в левой панели, вкладки Sessions, Events, Realm Settings. Куча настроек в каждом клиенте — Access Type, Standard Flow Enabled, Valid Redirect URIs, Mappers, Roles, Client Scopes. Проще всего было в туториале — кликнул несколько кнопок, работает. В реальности возникают вопросы: почему нельзя всё в master realm положить (разработчики так делают в dev). Как правильно моделировать роли — realm-level или per-client. Что реально даёт «Confidential» vs «Public» access type кроме секрета. Что такое «bearer-only» и зачем нужен если это не полноценный клиент. Почему при `implicit` flow нельзя получить refresh token. Что такое user federation и когда лучше LDAP-sync вместо копирования users в Keycloak. Как правильно сделать HA setup — можно ли просто два инстанса.

Разница между «настроил Keycloak в docker-compose» и «понимаю Keycloak» — способность за минуту ответить на конкретные архитектурные вопросы. Понимать в какой realm класть какое приложение. Правильно моделировать иерархию ролей с composite. Знать какие mappers существуют и когда какой применять. Уметь настроить federation с корпоративным LDAP так чтобы users автоматически появлялись при первом login. Понимать что такое `First Broker Login` flow при social login. Знать как масштабировать Keycloak в prod (не «один docker container»).

Разберём глубоко: что такое Keycloak архитектурно (три функции в одном — IdP, AS, User Management). Realm как центральная концепция изоляции — когда создавать разные, master vs application realms, что нельзя между realms. Clients — все типы (Public, Confidential, Bearer-only) с реальными сценариями когда какой. Все settings клиента детально с примерами (Redirect URIs — почему строго проверяются, Web Origins для CORS, Access Type impact на flows). Users — атрибуты, credentials, federated identities, sessions. Roles — realm vs client, composite roles с примером иерархии. Groups и связь с ролями. Mappers детально — все типы, что кладут в JWT, реальный use case для каждого. Endpoints Keycloak (стандартные OIDC + Admin REST API). Authentication flows и как их кастомизировать (2FA, terms, custom SPI). User Federation с LDAP/AD — что реально происходит при sync. Identity Brokering — SSO через Google/Facebook. Themes — кастомизация UI. Admin API для автоматизации. HA setup в prod — cluster, БД, cache, load balancer. Мониторинг и troubleshooting. Best practices и типовые ошибки.

OAuth/OIDC протокол — файл 117. JWT детально — 118. Spring Security + Keycloak integration — 120. Здесь фокус на самом Keycloak.

## Что такое Keycloak: три функции в одном

Проще всего понять Keycloak через то, **что он делает** для приложения.

Раньше (без Keycloak) типичное приложение имело:
1. **Таблицу users в БД** — id, username, email, password_hash.
2. **Login endpoint** — принимает username/password, проверяет hash, создаёт session.
3. **Session store** — cookies или Redis, чтобы user не логинился каждый запрос.
4. **Middleware проверки прав** — «этот endpoint только для admin».
5. **UI**: страница login, forgot password, change password, my profile.
6. **Регистрация** — новый user создаётся сам.
7. **Password reset** — flow с email, token, новая password page.
8. **Email verification** — подтвердить email при регистрации.
9. **Session management** — logout, «выйти со всех устройств».
10. **Аудит** — кто когда login, failed attempts, admin actions.

Каждое приложение имеет **всю эту машинерию** свою. Дублирование. Когда компания имеет 20 приложений — 20 таблиц users, 20 login endpoints, 20 разных UI логина. Пользователь имеет 20 разных аккаунтов, помнит 20 паролей.

**Keycloak решает — централизует всё это**. Одна установка Keycloak = один источник identity для всей компании. 20 приложений доверяют Keycloak.

**Три функции Keycloak в одном**:

**1. Identity Provider (IdP)** — где живут пользователи. Хранит users, hashes паролей, атрибуты (email, name), credentials (password, OTP). Или федерирует с внешним источником (LDAP, AD, Google). Аутентифицирует user'а — проверяет credentials, решает «да, это Иван».

**2. Authorization Server (OAuth 2.0 AS)** — выдаёт токены. Приложения-клиенты регистрируются в Keycloak (client_id, client_secret, redirect URIs). После успешной аутентификации user'а Keycloak выдаёт клиенту access_token, refresh_token, id_token. Все OAuth/OIDC endpoints (`/auth`, `/token`, `/userinfo`, `/jwks`, `/logout`) — Keycloak.

**3. Access Management** — роли, группы, permissions. Определяет что кому можно. Роли в токене — приложение видит и авторизует. Fine-grained authorization через Keycloak Authorization Services — специальный feature для сложных сценариев (Attribute-Based Access Control).

Все три функции — в одном сервере. Одна БД, одна admin console, один процесс.

**Плюсы централизации identity в Keycloak**:

- **SSO** (Single Sign-On) — залогинился в одно приложение, автоматически залогинен во всех. Работает через shared session Keycloak.
- **Один set of credentials** для user'а — один пароль на все приложения.
- **Управление users в одном месте** — HR создал user в Keycloak, доступен во всех приложениях сразу.
- **Стандартные протоколы** — приложения общаются через OIDC/SAML, независят от Keycloak специфически. Можно поменять Keycloak на Auth0 без изменения приложений.
- **Готовые UI** — login page, forgot password, account management, email templates. Не пишешь сам.
- **Enterprise features** — 2FA, brute force detection, session management, audit log, LDAP federation, social login. Всё готовое, кастомизируется.

**Стек Keycloak**:

- Написан на **Java**. До версии 17 работал на WildFly (Java EE), с версии 17+ мигрировал на **Quarkus** (modern, native compilation option). Quarkus сделал startup в 10x быстрее — теперь ~10 секунд вместо минуты.
- Хранит данные в **БД**: PostgreSQL, MySQL, MariaDB, MSSQL, Oracle. Для dev — embedded H2 (не для prod!). В prod — обязательно внешняя БД.
- **Cache через Infinispan** — distributed cache между нодами в HA setup. Ускоряет проверку sessions, tokens, user data.
- **Frontend**: Freemarker templates для server-rendered UI (login page, account console) + JavaScript admin console (v2 на React). Можно кастомизировать через themes.
- **Admin REST API** — всё что делается через admin console доступно через HTTP API. Полная автоматизация setup.

## Realm: центральная концепция

**Realm** — независимое «пространство» в Keycloak. Свои users, свои clients, свои roles, свои settings. Realms полностью изолированы — user из realm A **не может login в realm B**, даже с тем же именем и паролем. Это разные users в разных пространствах.

**Аналогия**. Представь Keycloak как большое здание. Каждый realm — отдельный «офис» в здании. Свои сотрудники (users), свои поставщики (clients), свои правила (settings). Сотрудник одного офиса не имеет ключа от другого — они физически изолированы.

Другая аналогия — **tenant в multi-tenant SaaS**. Realm = один клиент SaaS. У него свои users, свой конфиг, свои приложения.

### Master realm — специальный

**Master realm** создаётся автоматически при первом запуске Keycloak. Содержит **admin пользователей самого Keycloak** — тех кто управляет Keycloak (создаёт другие realms, настраивает cluster, обновляет version).

**Правило**: **никогда не использовать master для приложений**. Только для админов Keycloak. Приложения — в собственных realms.

Почему? Master realm имеет особые привилегии — его admin user может управлять всеми realms. Если положить обычных users в master и один из них станет compromised — можно скомпрометировать весь Keycloak.

### Приложенческие realms

Создаются админом Keycloak под конкретные нужды. Примеры naming:

- `myapp` — единственное приложение.
- `production`, `staging`, `dev` — по environments (если один Keycloak на все окружения — редко, обычно каждое своё).
- `acme-corp`, `beta-inc`, `startup-xyz` — по клиентам SaaS.
- `internal`, `external` — разделение внутренних сотрудников и внешних клиентов.

### Когда создавать разные realms

**Полная изоляция users**. Например SaaS: каждый клиент = свой realm. Пользователи acme-corp не знают что существует beta-inc. Разные login pages, разные URLs (`keycloak/realms/acme-corp/...`).

**Разные политики authentication**. Realm A требует MFA для всех, realm B — только для admin. Разные session timeouts (realm A — 30 мин idle, realm B — 8 часов). Разные password policies.

**Разные окружения**. dev / staging / prod — обычно отдельные Keycloak instances (не realms). Но иногда — один Keycloak, разные realms.

**Разные проекты внутри компании**. Разработка проекта A на своём realm, разработка B на своём. Их разработчики не пересекаются, users отдельные.

### Когда НЕ создавать realm на каждое приложение

Если несколько приложений одной команды используют **общих users** и должны иметь SSO между собой — **один realm с несколькими clients**. Это стандартный подход для микросервисной архитектуры одного продукта.

Пример: продукт CRM состоит из web-app, mobile-app, admin-panel, batch-jobs. Все пользуются одними и теми же users. Один realm `crm`, четыре clients (`crm-web`, `crm-mobile`, `crm-admin`, `crm-batch`). SSO между всеми, users один раз для всех.

Плохой пример: положить каждое из этих 4 приложений в свой realm. Users нужно дублировать в 4-х realms. SSO не работает. Изменение user (email) — надо в 4-х местах.

### Realm Settings — что настраивается

Открываешь realm → Settings. Много вкладок:

**Login** tab — как выглядит login flow:
- **Registration allowed** — могут ли users сами регистрироваться. Если да — «Sign up» появляется на login page.
- **Forgot password** — доступна ли восстановление пароля.
- **Remember me** — checkbox «запомнить меня» на login (продлевает session).
- **Email as username** — использовать email как username при логине.
- **Login with email** — можно логиниться и через username, и через email.
- **Duplicate emails** — можно ли иметь двух users с одним email (обычно off).
- **Verify email** — требовать подтверждения email после регистрации.
- **Edit username** — может ли user изменить свой username.

**Tokens** tab — timeouts:
- **Access Token Lifespan** — по умолчанию 5 мин. Часто увеличивают до 15-30 мин.
- **SSO Session Idle** — сколько может быть неактивна session (по умолчанию 30 мин).
- **SSO Session Max** — абсолютный максимум session (по умолчанию 10 часов).
- **Client Session Idle/Max** — для offline sessions.
- **Refresh Token Max Reuse** — можно ли использовать один refresh несколько раз (по умолчанию 0 = rotation).

**Security Defenses**:
- **Brute Force Detection** — после N failed attempts блок пользователя.
- **X-Frame-Options** — защита от clickjacking.
- **Content-Security-Policy** — какие ресурсы можно загружать.
- **HSTS** — Strict-Transport-Security header.

**Password Policy** — правила паролей:
- Minimum length (12+ рекомендуется).
- Digits, special chars, uppercase, lowercase required.
- Not username / email.
- Password history (не повторять последние N).
- Password expiration (сколько дней).
- Hashing algorithm (PBKDF2 default).

**OTP Policy** — если используется 2FA:
- TOTP (time-based, Google Authenticator) vs HOTP (counter-based).
- Digits (6 или 8).
- Period (30 сек стандарт).
- Algorithm (SHA1 default, SHA256/SHA512).

**Themes** — какие темы использовать:
- Login theme — страница логина.
- Account theme — user self-service.
- Admin theme — admin console.
- Email theme — email templates.

### URLs realm'а

Каждый realm имеет свои endpoints:

```
https://keycloak.example.com/realms/myapp/                                        — realm root
https://keycloak.example.com/realms/myapp/.well-known/openid-configuration        — OIDC discovery
https://keycloak.example.com/realms/myapp/protocol/openid-connect/auth            — authorization endpoint (для redirect)
https://keycloak.example.com/realms/myapp/protocol/openid-connect/token           — token endpoint (обмен code, refresh)
https://keycloak.example.com/realms/myapp/protocol/openid-connect/userinfo        — user info
https://keycloak.example.com/realms/myapp/protocol/openid-connect/certs           — JWKS (публичные ключи)
https://keycloak.example.com/realms/myapp/protocol/openid-connect/logout          — logout endpoint
https://keycloak.example.com/realms/myapp/protocol/openid-connect/token/introspect — token introspection
https://keycloak.example.com/realms/myapp/protocol/openid-connect/revoke          — revocation
https://keycloak.example.com/realms/myapp/account/                                — user self-service UI
https://keycloak.example.com/realms/myapp/protocol/saml                           — SAML endpoint (если используется)
```

Discovery endpoint (`/.well-known/openid-configuration`) — самый важный. Клиенты (Spring Security) читают его при старте, узнают все URLs автоматически. Не хардкодить URLs в конфиге приложений — использовать discovery.

## Clients: приложения работающие с Keycloak

**Client** в Keycloak = **приложение** которое хочет использовать Keycloak для аутентификации/авторизации. Каждое приложение регистрируется как отдельный client. Получает `client_id` (публичный идентификатор), опционально `client_secret` (приватный).

**Важно понимать**: client это **приложение**, не user. Не «клиент компании» а «клиентское приложение» в терминах OAuth.

Примеры clients:
- `myapp-web` — web-приложение с backend'ом.
- `myapp-spa` — single-page app (React/Angular).
- `myapp-mobile` — мобильное приложение.
- `myapp-api` — REST API за SPA.
- `myapp-batch` — batch job который делает M2M вызовы.
- `myapp-admin` — admin panel.

### Три типа clients (Access Type)

Определяют как client аутентифицируется и какие flows может использовать.

**Confidential** — с secret. Приложение может **безопасно хранить credentials**. Backend server, batch jobs, microservices — код на сервере, доступ к secret'у ограничен операторами.

- Имеет `client_secret` который используется при обмене code на token и при Client Credentials flow.
- Может использовать **все flows** включая Client Credentials.
- **Standard Flow** (Authorization Code) — да.
- **Direct Access Grants** — да (но обычно off, Password Grant deprecated).
- **Service Accounts** — да (для Client Credentials).

**Public** — без secret. Приложение **не может безопасно хранить credentials**. SPA (React), mobile apps — код на клиенте, любой secret скомпрометирован.

- Нет `client_secret`.
- Использует **Authorization Code + PKCE** flow (PKCE заменяет secret доказательством владения code_verifier).
- Standard Flow — да.
- **Service Accounts — нет** (нужен secret).

**Bearer-only** — специальный тип для API/микросервисов которые **только принимают JWT и проверяют**, не делают login flow.

- Нет login endpoints — user не может через этот client залогиниться.
- Нет secret (не нужен — не общается с AS в login flows).
- Только валидирует tokens.
- Пример: internal API за API Gateway. Gateway делает OAuth flow, API просто проверяет JWT.

**Deprecated в новых версиях Keycloak** — bearer-only тип. Заменяется на обычный Confidential с отключенными flows.

### Choosing type: практика

| Приложение | Тип |
|-----------|------|
| Server-side web app (Thymeleaf, JSP) с backend'ом | Confidential + Standard Flow |
| SPA (React/Angular) без backend'а | Public + Standard Flow + PKCE |
| Mobile app (iOS/Android) | Public + Standard Flow + PKCE |
| Backend API за SPA (только проверяет JWT) | Confidential + все flows off (bearer-only style) |
| Batch job / cron делает API calls | Confidential + Service Accounts (Client Credentials) |
| Microservice общается с другими microservices | Confidential + Service Accounts |

### Client settings детально

Открываешь client → много вкладок. Разберём главные.

**Settings** tab — основы:

- **Client ID** — уникальный идентификатор. Не меняется после создания (важно, приложения его хардкодят).
- **Name / Description** — human-readable для admin console.
- **Enabled** — активен ли (можно временно disable без удаления).
- **Root URL** — базовый URL приложения (`https://myapp.com`). Prefix для всех остальных URLs. Меняешь Root — остальные автоматически.
- **Home URL** — куда redirect после SSO logout.
- **Valid Redirect URIs** — **критически важно**. Куда Keycloak может redirect user'а после логина. **Строго проверяется**. Если приложение шлёт `redirect_uri=X` в auth request и `X` не в этом списке — Keycloak отвергает.
- **Web Origins** — CORS. Какие origins могут делать AJAX запросы к Keycloak (для проверки токенов, refresh, logout). Обычно `+` (auto — из Redirect URIs) или явно `https://myapp.com`.
- **Admin URL** — куда Keycloak шлёт backchannel notifications (logout, session invalidation).
- **Access Type** — Confidential / Public / Bearer-only (обсудили выше).
- **Standard Flow Enabled** — Authorization Code flow.
- **Implicit Flow Enabled** — **DEPRECATED, не включать**.
- **Direct Access Grants Enabled** — Password Grant. Only для legacy или CLI tools.
- **Service Accounts Enabled** — Client Credentials flow.

**Valid Redirect URIs — тонкости**:

- Точная строка проверяется. `https://myapp.com/callback` != `https://myapp.com/callback/` (trailing slash).
- Wildcard `*` в конце разрешён: `https://myapp.com/*` — принимает любой path.
- Wildcard в середине **нельзя** для безопасности: `https://*.myapp.com/callback` — не работает.
- `localhost` для dev: `http://localhost:*` разрешает любой port.

Небезопасный wildcard `https://*` — принимает **любой** сайт. Никогда так не делать — атакующий регистрирует свой домен, вставляет в redirect_uri, получает user's authorization code.

**Credentials** tab — client_secret (только для Confidential):
- **Client Authenticator** — как client аутентифицируется. `Client Id and Secret` (стандарт) или `Signed JWT` (client подписывает JWT своим приватным ключом, Keycloak проверяет публичным — mTLS-like).
- **Secret** — сам secret. Можно regenerate (rotation).

**Roles** tab — client roles (см. раздел ниже).

**Client Scopes** tab — какие scopes доступны:
- **Default Client Scopes** — автоматически включаются в каждый token.
- **Optional Client Scopes** — user может явно запросить (`scope=openid email offline_access`).

**Mappers** tab — что попадает в токен (см. раздел ниже).

**Sessions** tab — активные sessions пользователей этого client'а.

**Advanced** tab:
- **Access Token Lifespan** — переопределить realm-level. Например API с sensitive data — 5 минут, обычное — по realm defaults (30 мин).
- **Access Token Signature Algorithm** — RS256 / RS384 / ES256 / HS256. По умолчанию RS256.
- **Fine-grained OpenID Connect configuration** — override anything.

## Users: пользователи

**User** в Keycloak — пользователь realm'а. Хранится в БД Keycloak (в таблице `user_entity` и связанных). Или федерируется из LDAP/AD (см. ниже).

**Атрибуты user**:

**Standard**:
- **Username** — уникален внутри realm.
- **Email** — обычно уникален (настраивается).
- **First Name** / **Last Name**.
- **Email Verified** — подтвердил ли email.
- **Enabled** — активен. Disabled = не может login.

**Custom attributes** — key-value pairs. Например:
- `department: engineering`
- `employee_id: EMP-12345`
- `phone_number: +7-701-123-4567`
- `tenant_id: acme-corp` (multi-tenant SaaS)
- `feature_flags: beta_users,new_dashboard`

Кастомные атрибуты используются в mappers для добавления в JWT.

### Credentials — как user аутентифицируется

**Password** — стандарт. Хешируется PBKDF2 (крипто-стойкий, устойчив к brute-force даже на GPUs). Настраиваемое количество итераций.

**Temporary password** — user должен изменить при первом логине.

**OTP** (One-Time Password) — 2FA через приложение (Google Authenticator, Microsoft Authenticator, Authy). TOTP или HOTP.

**WebAuthn** — hardware security keys (YubiKey), biometrics (Windows Hello, Touch ID).

**X.509 certificate** — client certificate authentication. User имеет certificate установленный в browser, Keycloak проверяет.

Один user может иметь **несколько credentials** одновременно. Например password + OTP + WebAuthn — user выбирает при login.

### Sessions

Каждый залогиненный user имеет одну или несколько **sessions** в Keycloak. Session = login событие с одного устройства.

Admin может:
- Посмотреть все sessions user'а (User → Sessions tab).
- Посмотреть где залогинен (IP, User Agent).
- **Force logout** конкретную session (или все) — токены становятся невалидны.

Полезно при инцидентах — пользователь заявил compromise, admin делает Logout All Sessions.

### Consents

Если приложение просит scopes требующие consent — Keycloak показывает пользователю «Приложение X просит доступ к: email, profile. Разрешить?». После согласия — consent сохраняется в Keycloak.

User может отозвать consent через Account console — при следующем логине снова спросят.

### Federated Identity

Если user пришёл через social login (Google, Facebook) — здесь ссылка на внешний identity. Показывает «linked with Google, sub=xxx».

User может **link** несколько social identities с одним Keycloak user'ом. Полезно если user раньше зашёл через Facebook, потом хочет Google — можно связать.

## Roles: realm vs client, composite

Keycloak имеет **два уровня ролей**.

### Realm Roles

Глобальные для realm. Определяются в realm settings → Roles.

Примеры:
- `ADMIN` — админ приложения.
- `USER` — обычный пользователь.
- `MANAGER` — менеджер.
- `OFFLINE_ACCESS` — специальная роль для offline tokens.

Один user может иметь несколько realm ролей. Realm роли одинаково видны всем clients в realm.

### Client Roles

Специфичные для конкретного client. Определяются в client settings → Roles.

Примеры для client `orders-api`:
- `orders:read`
- `orders:write`
- `orders:admin`

Client роли **видны только своему client'у** (по умолчанию). Приложение `orders-api` видит `orders:admin`, приложение `users-api` — нет.

### Как выбирать между realm и client roles

**Realm role** — общая концепция роли для всей системы. Пример: `ADMIN` — админ везде, `USER` — обычный юзер везде.

**Client role** — специфичная для приложения. Пример: `orders-api:read`, `reports-api:generate`.

**Хороший подход**:
- Realm roles — для broad концепций: `USER`, `ADMIN`, `SUPPORT`.
- Client roles — для fine-grained permissions конкретного API: `orders:read`, `orders:write`, `orders:delete`.

Комбинация: user имеет realm role `USER` (базовый доступ) + client roles `orders:read` в orders-api (специфический доступ к orders).

### Composite Roles

Роль включающая другие роли. При назначении composite user автоматически получает все включённые роли.

**Пример иерархии**:
- `USER` — базовая, ничего не включает.
- `MANAGER` — composite, включает `USER` + `orders:read` + `reports:read`.
- `ADMIN` — composite, включает `MANAGER` + `users:manage` + `settings:write`.
- `SUPER_ADMIN` — composite, включает `ADMIN` + все `*:admin` в client'ах.

Назначил user'у `SUPER_ADMIN` — автоматически имеет все нижестоящие роли. Не нужно назначать каждую отдельно.

**Плюс**: единый point of change. Хочешь добавить всем менеджерам новую роль? Добавляешь в composite `MANAGER` — все существующие менеджеры получают автоматически.

**Минус — не для всех сценариев**. Composite подразумевает иерархию. Не подходит если роли независимые (пользователь может иметь любую комбинацию).

### Как роли попадают в JWT

По умолчанию Keycloak кладёт:

**Realm roles** → `realm_access.roles`:
```json
{
  "realm_access": {
    "roles": ["USER", "MANAGER"]
  }
}
```

**Client roles** → `resource_access.<client_id>.roles`:
```json
{
  "resource_access": {
    "orders-api": {
      "roles": ["orders:read", "orders:write"]
    },
    "reports-api": {
      "roles": ["reports:read"]
    }
  }
}
```

Приложение (Resource Server) извлекает роли из этих полей.

**Composite roles разворачиваются** — если user имеет `MANAGER` (composite = USER + orders:read + reports:read), в JWT будут все включённые роли.

## Groups: коллекции users

**Group** — коллекция users. Group может иметь **roles** — все members получают эти roles автоматически.

Пример: group `finance-team` имеет роли `finance:read`, `finance:write`, `reports:generate`. User добавлен в group — получил роли. Удалён — потерял.

**Nested groups** — иерархия. `company/finance/accounting`. Может наследовать роли от parent.

**Attributes on groups** — group имеет свои custom атрибуты (например `department: finance`). Можно mapping в JWT — user получит атрибут своей группы.

### Groups vs Roles: когда что

**Roles** — для доступа к функциям. Что user может делать.

**Groups** — для организационной структуры. Кто user такой (в каком отделе, команде).

Комбинация: **groups с прикреплёнными roles**. Отдел `finance` имеет соответствующие роли `finance:*`. При добавлении user в `finance` — автоматически получает роли.

**Реальный пример**. Компания имеет структуру:
- Company
  - Engineering
    - Backend Team
    - Frontend Team
    - DevOps Team
  - Finance
  - HR
  - Support

Каждая группа имеет соответствующие роли:
- Backend Team → `backend:*`, `deploy:staging`.
- DevOps Team → все `*:admin`, `deploy:*`.
- Finance → `finance:*`, `reports:*`.

HR добавляет нового engineer в `Backend Team` — автоматически получает нужные роли. Изменил team (перевели в DevOps) — роли поменялись.

## Mappers: контроль над JWT

**Mapper** определяет что и как попадает в JWT (или SAML assertion).

Каждый client имеет свой набор mappers. Разные clients могут получать разные claims для одного и того же user. Например `orders-api` получает `tenant_id`, а `public-api` — нет.

### Типы mappers

**User Property Mapper** — берёт built-in атрибут user (email, firstName, lastName) и кладёт в токен под указанным claim name.

Пример: mapper типа User Property, property = `email`, token claim = `email`. Токен получит `"email": "ivan@example.com"`.

**User Attribute Mapper** — берёт custom атрибут user и кладёт в токен.

Пример: у user'а custom атрибут `department = engineering`. Настраиваю mapper: User Attribute, user attribute = `department`, token claim = `department`. Токен получит `"department": "engineering"`.

**Role Name Mapper** — переименовывает роли в токене.

Пример: realm role `admin` хочу видеть в токене как `ROLE_ADMIN` (Spring Security convention). Настраиваю mapper с указанием `role: admin` → `token claim: ROLE_ADMIN`.

**Group Membership Mapper** — кладёт группы user в токен.

Пример: `"groups": ["finance-team", "engineering"]`. Полезно для приложений которые фильтруют по группам.

**Audience Mapper** — добавляет audience в токен.

Полезно когда токен должен работать для нескольких APIs — добавляем каждый в `aud`. Также важно для решения проблемы «default aud = account в Keycloak».

**Hardcoded Claim Mapper** — просто добавляет статичное значение.

Пример: `"realm": "myapp"` для всех токенов этого client'а.

**Script Mapper** — самый гибкий. JavaScript код который вычисляет значение claim.

Пример: `token.subject + '@example.com'` — динамически computed email.

**Full Name Mapper** — специальный, склеивает firstName + lastName в `name`.

**Address Mapper** — стандартный OIDC address claim (объединяет street, city, country).

### Реальный use case: multi-tenant SaaS

Приложение — SaaS с многими клиентами. Каждый user принадлежит одному tenant'у. API должно фильтровать данные по tenant.

**Setup**:

1. У каждого user custom attribute `tenant_id`.
2. В client создать User Attribute Mapper:
   - User Attribute: `tenant_id`
   - Token Claim Name: `tenant_id`
   - Claim JSON Type: `String`
   - Add to ID token: yes
   - Add to access token: yes
   - Add to userinfo: yes

3. В JWT появляется:
```json
{
  ...
  "tenant_id": "acme-corp"
}
```

4. Приложение в каждом запросе извлекает `tenant_id` из JWT, фильтрует SQL:

```java
@GetMapping("/orders")
public List<Order> orders(@AuthenticationPrincipal Jwt jwt) {
    String tenantId = jwt.getClaimAsString("tenant_id");
    return orderRepo.findByTenantId(tenantId);
}
```

Multi-tenancy обеспечивается на уровне claim в токене — приложение не может «случайно» показать чужие данные (JWT содержит tenant_id, приложение фильтрует).

## Endpoints Keycloak

**Стандартные OIDC endpoints** (per realm) уже перечислили выше:
- `/protocol/openid-connect/auth` — authorize (redirect для login).
- `/protocol/openid-connect/token` — token endpoint.
- `/protocol/openid-connect/userinfo` — user info.
- `/protocol/openid-connect/certs` — JWKS.
- `/protocol/openid-connect/logout` — logout.
- `/protocol/openid-connect/token/introspect` — introspection.
- `/protocol/openid-connect/revoke` — revoke.

**SAML endpoints** (если используется):
- `/protocol/saml/descriptor` — SAML metadata.
- `/protocol/saml` — SAML SSO.

**Admin REST API** (`/admin/realms/{realm}/...`) — управление всем через HTTP.

### Admin REST API: что доступно

Всё что делается через admin console доступно через API:

- **Users**: create, update, delete, search, reset password, force logout.
- **Clients**: create, update, delete, regenerate secret.
- **Roles**: create, assign, remove, list users with role.
- **Groups**: create, add/remove members.
- **Sessions**: list active, terminate.
- **Events**: audit log queries.
- **Realm settings**: update.

Полная автоматизация setup — infrastructure as code.

### Пример: создать user через Admin API

Аутентификация — Client Credentials flow с client имеющим admin роли:

```bash
# Получить admin token
TOKEN=$(curl -X POST \
  https://keycloak/realms/master/protocol/openid-connect/token \
  -d "grant_type=client_credentials" \
  -d "client_id=admin-cli" \
  -d "client_secret=SECRET" \
  | jq -r '.access_token')

# Создать user
curl -X POST \
  https://keycloak/admin/realms/myapp/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "ivan.ivanov",
    "email": "ivan@example.com",
    "enabled": true,
    "firstName": "Иван",
    "lastName": "Иванов",
    "attributes": {
      "department": ["engineering"],
      "tenant_id": ["acme-corp"]
    },
    "credentials": [
      {
        "type": "password",
        "value": "TempPass123!",
        "temporary": true
      }
    ]
  }'
```

Возвращает 201 Created с `Location` header содержащим URL нового user.

### Use cases для Admin API

**Provisioning** — создание users при onboarding из HR системы. HR добавила employee → automation создаёт Keycloak user с нужными атрибутами и ролями.

**Bulk operations** — массовое обновление атрибутов. Например «всем engineers добавить group `2024-hires`».

**Migrations** — при upgrade Keycloak или переносе на новую installation.

**Integrations** — CRM создаёт Keycloak user когда добавили нового customer.

**Testing** — Testcontainers Keycloak + auto-create users для integration tests.

**SDKs / Libraries**:
- **Keycloak Admin Java Client** — официальный.
- **python-keycloak**, **@keycloak/keycloak-admin-client** (Node.js) и другие.

## Authentication Flows: кастомизация login

**Authentication flow** — последовательность шагов при login. По умолчанию: username/password. Но можно кастомизировать.

**Built-in flows**:
- **Browser** — стандартный login через browser.
- **Direct Grant** — для Direct Access Grants (Password Grant).
- **Registration** — регистрация нового user.
- **Reset Credentials** — forgot password flow.
- **First Broker Login** — при первом login через external IdP (Google, etc).
- **Post Broker Login** — после login через external.

Каждый flow — граф steps:
- **Required** — обязательный шаг.
- **Alternative** — один из группы (например email OR SMS).
- **Optional** — можно пропустить.
- **Disabled** — отключён.

### Стандартный Browser flow

```
Cookie [Alternative]                    ← auto-login если session cookie есть
Kerberos [Alternative]                  ← SSO если Kerberos setup
Identity Provider Redirector [Alternative]  ← redirect на social IdP если выбрано
Forms:                                  ← если ничего выше не сработало
  Username Password Form [Required]     ← вводит login/password
  Browser - Conditional OTP [Conditional]  ← если у user OTP настроен
    OTP Form [Required]                    ← вводит код 2FA
```

### Пример кастомизации: добавить SMS 2FA

Хочу второй фактор через SMS вместо (или дополнительно к) OTP.

1. Копировать Browser flow → New copy `Browser with SMS`.
2. Добавить новый step после OTP Form: `SMS Verification` (custom execution).
3. Set Required.
4. В client → Advanced → Authentication Flow Overrides → Browser: выбрать `Browser with SMS`.

Реализация SMS execution — Keycloak SPI (Service Provider Interface). Java plugin.

### Реализация custom SPI

Keycloak расширяется через SPI — Java интерфейсы которые ты реализуешь.

Пример SPI для SMS verification:

```java
public class SmsAuthenticator implements Authenticator {
    
    @Override
    public void authenticate(AuthenticationFlowContext context) {
        // Генерировать код, отправить SMS
        String code = generateCode();
        sendSms(context.getUser().getFirstAttribute("phone_number"), code);
        
        // Сохранить в session
        context.getAuthenticationSession().setAuthNote("smsCode", code);
        
        // Показать форму ввода кода
        Response challenge = context.form()
            .createForm("sms-verification.ftl");
        context.challenge(challenge);
    }
    
    @Override
    public void action(AuthenticationFlowContext context) {
        // Проверить введённый код
        String enteredCode = context.getHttpRequest().getDecodedFormParameters().getFirst("code");
        String expectedCode = context.getAuthenticationSession().getAuthNote("smsCode");
        
        if (enteredCode.equals(expectedCode)) {
            context.success();
        } else {
            context.failureChallenge(AuthenticationFlowError.INVALID_CREDENTIALS, 
                context.form().setError("Invalid SMS code").createForm("sms-verification.ftl"));
        }
    }
    
    // ... другие методы
}
```

Компилируешь JAR, кладёшь в `providers/` директорию Keycloak, restart. Появляется как execution option в flow builder.

### Другие полезные custom flows

**Terms and Conditions** — user должен принять условия при первом login. Встроенный execution, добавить в Registration flow.

**Recaptcha** — защита от bot'ов на login/registration. Google reCAPTCHA integration встроенная.

**Conditional flows** — «сделай X только если Y». Например «требовать 2FA только для admin users».

**Password-less** — WebAuthn как единственный factor (без password).

## User Federation: интеграция с LDAP/AD

Enterprise часто уже имеет корпоративный LDAP или Active Directory с users. Не хочется дублировать в Keycloak.

**User Federation** — Keycloak **читает** users из внешнего LDAP/AD, показывает как своих, аутентифицирует против LDAP.

### Что даёт federation

- **Не дублировать users**. Users живут в LDAP, Keycloak их «показывает».
- **Auth против LDAP**. User вводит password в Keycloak — Keycloak делает LDAP bind с этими credentials. Если LDAP принимает — user authenticated.
- **Single source of truth** — HR добавила user в LDAP, автоматически доступен в Keycloak.
- **Legacy integration**. Существующая система работает на LDAP — Keycloak позволяет постепенно мигрировать на OAuth без нарушения LDAP-based приложений.

### Конфигурация LDAP provider

Realm → User Federation → Add provider → LDAP.

Основные настройки:

- **Connection URL**: `ldap://ldap.company.com:389` (или `ldaps://` для TLS).
- **Bind DN**: DN аккаунта Keycloak в LDAP (`cn=keycloak,ou=service,dc=company,dc=com`). Keycloak авторизуется в LDAP этим аккаунтом для поиска.
- **Bind Credentials**: password service account.
- **Users DN**: где искать users в LDAP tree (`ou=people,dc=company,dc=com`).
- **Username LDAP Attribute**: какой атрибут LDAP = username в Keycloak. `uid` для OpenLDAP, `sAMAccountName` для AD.
- **User Object Classes**: `person, inetOrgPerson, organizationalPerson` для OpenLDAP, `user` для AD.
- **RDN LDAP Attribute**: обычно тот же что Username.
- **UUID LDAP Attribute**: `entryUUID` (OpenLDAP), `objectGUID` (AD).

### Sync strategy

**Import Users** — стратегия:
- **ON** — users имортируются в Keycloak БД (создаётся запись в user_entity). Периодически sync с LDAP. Быстро работать с users, но данные могут «отставать» от LDAP.
- **OFF** — always fetch from LDAP при каждом запросе. Всегда актуально, но медленнее.

Обычно **ON** — периодический full sync (раз в час) + on-demand при первом login + import при auth.

**Sync Registrations**:
- **ON** — user регистрируется через Keycloak UI → создаётся в LDAP.
- **OFF** — регистрация только через LDAP (Keycloak read-only).

**Edit Mode**:
- **READ_ONLY** — Keycloak может только читать LDAP. Изменения (change password, update profile) — не работают.
- **WRITABLE** — Keycloak может писать в LDAP.
- **UNSYNCED** — user's local overrides в Keycloak, LDAP не меняется.

### LDAP Mappers

Как LDAP атрибуты mapping'ятся на Keycloak атрибуты.

Стандартные mappers создаются автоматически:
- `cn` → firstName + lastName
- `mail` → email
- `memberOf` → groups (при setup Group LDAP Mapper).

**Custom mappers** для custom атрибутов:
- LDAP атрибут `department` → Keycloak user attribute `department`.
- LDAP атрибут `employeeNumber` → Keycloak `employee_id`.

Всё что доступно в LDAP — можно замапить в Keycloak.

### Kerberos SSO

Если корпоративная сеть с Kerberos (Windows Domain), Keycloak может делать automatic SSO. User на domain-joined машине заходит на Keycloak URL — автоматически аутентифицирован через Kerberos ticket, никакого login screen'а.

Настройка:
- В LDAP provider — Allow Kerberos authentication.
- Kerberos realm settings — принципал Keycloak (`HTTP/keycloak.company.com@COMPANY.COM`).
- Keytab на Keycloak сервере с secret этого принципала.
- Browser user настроен на использование Kerberos ticket для этого домена.

Работает — user без ввода credentials попадает в приложение. Standard Windows Enterprise SSO.

## Identity Brokering: SSO через внешние IdPs

**Identity Brokering** — Keycloak как «мост» к внешним identity providers (Google, Facebook, LinkedIn, Twitter, Microsoft, GitHub, corporate SAML/OIDC).

Сценарий: приложение хочет предложить «Login with Google». Не хочет напрямую интегрироваться с Google API — использует Keycloak как посредника.

### Как работает

1. В Keycloak realm добавить Identity Provider (Google).
2. Задать `client_id` / `client_secret` Google (полученные при регистрации приложения в Google Cloud Console).
3. На login page Keycloak появляется кнопка «Google».

**Flow**:
1. User нажимает «Google» на Keycloak login page.
2. Keycloak делает redirect на Google OAuth (Keycloak — client Google).
3. Google login screen.
4. User логинится в Google.
5. Google redirect обратно к Keycloak с authorization code.
6. Keycloak обменивает code на Google id_token.
7. Keycloak проверяет id_token, извлекает user data (email, name).
8. Создаёт **локального Keycloak user** (или обновляет существующего) linked с Google identity.
9. Keycloak выдаёт **свой** access_token приложению (не Google's).

Приложение работает только с Keycloak — единый интерфейс, независимо от того как user залогинился. При смене IdP или добавлении новых (Facebook, Apple) — приложение не меняется.

**Supported protocols**: OIDC, OAuth 2.0, SAML 2.0. Support для Google, Facebook, LinkedIn, GitHub, Instagram, Microsoft, Twitter, Apple, PayPal, StackOverflow, Twitter, plus custom OIDC/SAML providers.

### First Broker Login flow

Что происходит при **первом** login через external. Настраивается через specific flow.

Опции:
- **Auto-link** — если email из external совпадает с существующим Keycloak user, автоматически link (тот же user, теперь через Google тоже).
- **Ask** — спросить user'а: «У нас уже есть аккаунт с этим email. Link or create new?»
- **Create new** — всегда создавать нового user'а, никогда не link.

Стандартное поведение — Ask, но настраивается.

### Практика Identity Brokering в enterprise

Часто enterprise:
- Own Keycloak realm для приложений.
- Keycloak связан с corporate Azure AD через SAML.
- Users не хранятся в Keycloak — federation с Azure AD.
- Приложения работают только с Keycloak через OIDC.

Плюсы:
- Employees используют existing Azure AD credentials.
- Onboarding/offboarding через AD (Keycloak автоматически синхронизирует).
- Приложения независимы от AD — говорят только с Keycloak по OIDC.
- Легко добавить social login для external users в тот же realm.

## Themes: кастомизация UI

Keycloak UI (login page, account console, admin console, emails) — кастомизируется через **themes**.

**Theme** — набор ресурсов:
- **Freemarker templates** (`.ftl` файлы) — HTML структура.
- **CSS** — стили.
- **Images** — logos, backgrounds.
- **JavaScript** — client-side логика.
- **Message properties** — i18n (`messages_en.properties`, `messages_ru.properties`, `messages_kk.properties`).

**Стандартные themes**: `keycloak` (дефолт), `base` (минимальный, для extension).

### Custom theme

1. Копировать `keycloak` theme как starting point.
2. Модифицировать templates под свой бренд.
3. Положить в `themes/` директорию Keycloak (или monter volume в Docker).
4. В realm settings → Themes → выбрать custom theme.

### Что кастомизируется

- **Login theme** — login/registration/forgot password/OTP pages.
- **Account theme** — user self-service console (change password, sessions, applications).
- **Admin theme** — admin console (обычно не кастомизируют).
- **Email theme** — email templates для verification, password reset, magic link, etc.

Для enterprise обычно — свой login и account themes с корпоративными цветами, логотипом. Email theme — company branding в письмах.

## HA setup: production deployment

Единичный Keycloak instance — single point of failure. В prod — cluster.

### Компоненты HA

**Multiple Keycloak nodes** — 2-3 nodes за load balancer'ом. При падении одного — трафик идёт на других.

**Shared database** — PostgreSQL (стандарт), MySQL, MariaDB, MSSQL, Oracle. Все Keycloak nodes используют одну БД. Realm config, users, roles, sessions (для offline sessions) — хранятся здесь.

**Distributed cache** — **Infinispan** (встроен в Keycloak). Keycloak кэширует много (users, sessions, tokens). В HA cache должен быть distributed — изменение на node A видно на node B. Работает через JGroups (UDP multicast или TCP unicast между nodes).

**Sticky sessions на load balancer** — рекомендуется. User залогинился на node A → его requests идут на node A (пока node жив). Уменьшает cache misses (session cached локально на node A). Session ID хранится в cookie, LB роутит по нему.

**Health checks**:
- **Liveness**: `/health/live` — процесс жив.
- **Readiness**: `/health/ready` — готов принимать трафик (БД доступна, cache initialized).

### Пример: 2-node cluster с docker-compose

```yaml
version: '3'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: xxx
    volumes:
      - pg_data:/var/lib/postgresql/data
  
  keycloak-1:
    image: quay.io/keycloak/keycloak:23.0
    command: start --optimized
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://db/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: xxx
      KC_HOSTNAME: keycloak.example.com
      KC_CACHE: ispn
      KC_CACHE_STACK: kubernetes  # или tcp для non-k8s
      KC_HTTP_ENABLED: 'true'
      KC_HOSTNAME_STRICT: 'false'
    depends_on: [db]
  
  keycloak-2:
    image: quay.io/keycloak/keycloak:23.0
    # same env
    depends_on: [db]
  
  nginx:
    image: nginx
    ports: ["443:443"]
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on: [keycloak-1, keycloak-2]
```

nginx.conf с sticky sessions:

```nginx
upstream keycloak {
    ip_hash;  # sticky sessions по IP
    server keycloak-1:8080;
    server keycloak-2:8080;
}

server {
    listen 443 ssl;
    server_name keycloak.example.com;
    
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    
    location / {
        proxy_pass http://keycloak;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Kubernetes deployment

Официальный **Keycloak Operator** для K8s — управляет cluster, HA, upgrades через CRDs.

Или Helm chart от Bitnami — популярный, кастомизируется.

## Мониторинг Keycloak

**Что мониторить**:

**Метрики** — Keycloak Quarkus версии экспозит Prometheus metrics на `/metrics`:
- HTTP request rate, latency, errors (по endpoint).
- Database connection pool usage.
- Active user sessions count per realm.
- Login rate (success / failure).
- Token issuance rate (access, refresh).
- JVM metrics (heap, GC, threads).

**Логи**:
- Успешные и failed logins.
- Errors при валидации токенов.
- Slow queries к БД.
- Admin actions (кто что менял).

**Events** — Keycloak умеет писать audit trail:
- Все actions users (login, logout, register, password change).
- Admin actions.
- Настраивается в realm settings → Events.

Storage событий:
- **JBOSS logging** (в файл или journal).
- **DB storage** (в основную БД, для admin console просмотра).
- **Custom SPI** — отправка в Kafka, ELK, custom analytics.

**Health checks**:
- `/health/live` — процесс жив.
- `/health/ready` — готов принимать трафик.
- `/metrics` для Prometheus scraping.

## Best practices для prod

**Отдельная БД для Keycloak** — не смешивать с приложенческой. Разные lifecycle, разные требования backup.

**HTTPS обязательно** — Keycloak передаёт tokens, passwords. Никакого HTTP в production. TLS termination на LB или Keycloak напрямую.

**Реалистичные token lifespan**:
- Access token: 5-15 минут (не час-два как дефолт).
- Refresh token: часы для SPA, недели для mobile.
- SSO session idle: 30 мин типично.
- SSO session max: 8-12 часов (рабочий день).

**Refresh Token Rotation** — включить `Revoke Refresh Token = ON` в client settings.

**Brute Force Protection** — включить в realm settings. После 5 failed attempts — временный lock user'а на 15 минут. Защищает от credential stuffing.

**Password Policy** — минимум 12 символов, разные символы, expiration через 90 дней для sensitive systems.

**MFA обязателен для admin ролей** — минимум. Желательно для всех.

**Backup БД ежедневно** — realm config, users, roles, sessions — всё в БД. Тестировать restore процедуру раз в квартал.

**Мониторинг** — Prometheus + Grafana dashboard. Alert на failed logins spike (brute force?), slow response times, DB connection issues.

**Обновления** — Keycloak регулярно выпускает патчи. Особенно security. Обновляться в течение месяца-двух от release.

**Sizing** (примерное):
- Development: 512 MB heap, 1 CPU.
- Production small (< 10K users, < 100 requests/sec): 2 GB heap, 2 CPU.
- Production medium: 4-8 GB heap, 4 CPU, HA cluster.
- Large enterprise (100K+ users): 8+ GB heap, 4+ nodes cluster.

## Общие ошибки в проде

**Дефолтный `master` realm для приложений** — master только для admin Keycloak. Всегда создавать отдельный realm для каждого приложения (или группы связанных).

**Public client для backend'а** — backend имеет secret storage, должен быть Confidential. Public только для SPA/mobile.

**Слишком долгий access_token TTL** — дефолт 5 минут, часто увеличивают до часа-двух «для удобства». Плохо — компрометация = 1-2 часа доступа. Держать 5-15 минут, использовать refresh token для UX.

**Отсутствие backchannel logout** — user logout из одного приложения → другие приложения не знают. Использовать front-channel или back-channel logout.

**Хранение client_secret в git** — никогда. Secrets в Vault / K8s secrets / environment variables только.

**Один Keycloak instance без HA в prod** — SPOF, потеря = весь identity недоступен, приложения не работают. Минимум 2 node cluster + shared DB.

**Забытые sessions пользователей при инциденте** — при компрометации сделать force logout всех sessions через Admin API. `POST /admin/realms/{realm}/users/{id}/logout`.

**LDAP sync без monitoring** — LDAP sync тихо ломается → новые users не появляются в Keycloak → они не могут login. Мониторить sync errors.

**Отсутствие backup БД** — потеря БД = потеря всего identity. Ежедневные backups минимум, регулярное тестирование restore.

**`Verify Email = OFF`** для user registration в prod — приводит к аккаунтам без валидных email. Включать.

**Слабая password policy** — «minimum 8 symbols» больше не безопасно. Минимум 12, требовать разные типы символов.

## Заключение

**Keycloak** — open-source IAM сервер: **Identity Provider** + **Authorization Server** + **user management** в одном. Стандарт для enterprise Java. Поддерживает OAuth 2.0, OIDC, SAML 2.0.

**Realm** — центральная концепция изоляции. Свои users, clients, roles, settings. Master realm только для admin Keycloak (никогда не для приложений!). Приложения — в своих realms. Multi-tenant SaaS = realm per tenant. Один продукт с несколькими приложениями → один realm с несколькими clients.

**Clients** — приложения работающие с Keycloak. **Confidential** (с secret, для backend/microservices) vs **Public** (без secret, для SPA/mobile, использует PKCE) vs **Bearer-only** (только проверка JWT, без login flow — deprecated).

**Client settings**: **Valid Redirect URIs** строго проверяются (никаких wildcards `https://*`!), Web Origins для CORS, Access Type + flow toggles, Access Token Lifespan (override realm default).

**Users** — хранятся в Keycloak или федерируются из LDAP/AD. Атрибуты (standard + custom), credentials (password/OTP/WebAuthn/x509), sessions, consents, federated identities. Custom attributes для business-specific данных → в JWT через mappers.

**Roles**:
- **Realm roles** — глобальные для realm (USER, ADMIN, MANAGER).
- **Client roles** — специфичные для клиента (orders:read, reports:write).
- **Composite roles** — включают другие роли (SUPER_ADMIN включает ADMIN + все `*:admin`). Автоматически разворачиваются в JWT.

В JWT: realm roles → `realm_access.roles`, client roles → `resource_access.<client>.roles`.

**Groups** — коллекции users с общими roles/attributes. Nested groups для иерархии. Groups для организационной структуры (отделы), roles для permissions.

**Mappers** — контроль что попадает в JWT. **User Property** (email, name), **User Attribute** (custom атрибуты), **Role Name** (переименование), **Group Membership** (список групп), **Audience** (добавить aud), **Hardcoded**, **Script** (JavaScript). Реальный use case: multi-tenant SaaS — `tenant_id` из user attribute в JWT для API фильтрации.

**Endpoints Keycloak**:
- Стандартные OIDC: `/auth`, `/token`, `/userinfo`, `/certs` (JWKS), `/logout`, `/introspect`, `/revoke`.
- Admin REST API `/admin/realms/{realm}/...` — управление users, clients, roles, groups, sessions.

**Authentication flows** — кастомизация login. Built-in: Browser, Direct Grant, Registration, Reset Credentials, First Broker Login. Кастомизация через SPI (Java plugins). Примеры: 2FA через SMS, Terms and Conditions, reCAPTCHA, WebAuthn passwordless.

**User Federation** с LDAP/AD — Keycloak читает users из LDAP, аутентифицирует против LDAP. Sync strategies (Import ON/OFF). LDAP mappers для атрибутов. **Kerberos SSO** для corporate networks с Windows Domain.

**Identity Brokering** — Keycloak как посредник к внешним IdPs (Google, Facebook, corporate SAML). Приложение работает только с Keycloak — единый интерфейс. First Broker Login flow — auto-link по email vs ask vs create new.

**Themes** — кастомизация UI login/account/admin/email. Freemarker templates + CSS + images + i18n. Для enterprise — свой брендовый theme.

**Admin REST API** — полная автоматизация setup. Provisioning users, bulk operations, migrations. Java Admin Client + другие SDK. Аутентификация через Client Credentials с service account имеющим admin роли.

**HA setup**: 2-3 Keycloak nodes + shared PostgreSQL + distributed Infinispan cache + sticky sessions на LB. Backup БД жизненно критичен. Kubernetes deployment через Operator или Helm chart.

**Мониторинг**: Prometheus metrics (`/metrics`), health checks (`/health/live`, `/health/ready`), events (login attempts, admin actions), Grafana dashboards.

**Best practices**: отдельная БД, HTTPS обязательно, короткие access token TTL (5-15 мин), refresh token rotation, brute force protection, password policy 12+ symbols, MFA для admin, ежедневный backup БД, регулярные обновления (security patches), HA cluster минимум 2 node.

OAuth/OIDC протокол — файл 117. JWT детально — 118. Spring Security + Keycloak integration полностью — 120. Здесь была глубина по Keycloak: realms/clients/users/roles/groups/mappers, все settings с реальными сценариями, federation с LDAP, brokering, admin API, HA production setup.
