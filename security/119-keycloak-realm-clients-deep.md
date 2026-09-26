# 119. Keycloak глубоко: realm, clients, users, roles, federation, admin

## Зачем это знать

Keycloak — самый популярный open-source Identity and Access Management (IAM) сервер в enterprise. Реализует OAuth 2.0, OIDC, SAML 2.0, User Federation с LDAP/AD, admin console, REST API. Практически стандарт в Java-мире (Red Hat стоит за проектом). У КНП, у большинства enterprise-приложений — Keycloak как IDP.

Разница между «настроил Keycloak в docker-compose» и «понимаю Keycloak» — способность за минуту ответить: что такое realm и почему нельзя всё в одном; как правильно моделировать clients (public vs confidential, зачем разница); где хранить роли (realm-level или client-level и почему); как работает user federation с существующим LDAP; что такое authentication flow и как встроить custom step (например второй фактор через SMS); как правильно делать HA setup Keycloak в проде; как использовать Admin API для автоматизации.

Разберём: что такое Keycloak фундаментально (identity provider, authorization server, user management всё в одном). Realm — центральная концепция, изоляция users/clients/roles. Clients — типы (public, confidential, bearer-only), settings, redirect URIs, access types. Users — creation, credentials, federated. Roles — realm roles vs client roles, composite roles. Groups. Mappers — как контролировать что попадает в JWT. Protocol mappers для custom claims. Endpoints Keycloak (стандартные OIDC + admin). Authentication flows — как модифицировать логин (2FA, captcha, terms). User federation — LDAP/AD integration. Identity brokering — SSO через Google/Facebook. Themes — кастомизация UI. Admin REST API для автоматизации. HA setup — cluster, DB, cache. Мониторинг и troubleshooting. Best practices для prod.

OAuth/OIDC протокол — файл 117. JWT детально — 118. Spring Security integration — 120. Здесь фокус на самом Keycloak.

## Что такое Keycloak фундаментально

**Keycloak** — сервер централизованного управления identity и access. Три главные функции в одном:

**1. Identity Provider (IdP)** — где хранятся пользователи (username, password, email, атрибуты). Или федерация с внешним источником (LDAP/AD). Аутентификация — Keycloak проверяет credentials.

**2. Authorization Server (OAuth/OIDC AS)** — выдаёт токены. Клиенты (приложения) регистрируются в Keycloak, получают токены после успешной аутентификации пользователя. Все OAuth/OIDC endpoints (authorize, token, userinfo, jwks, discovery) — Keycloak.

**3. Access management** — роли, группы, permissions, что кому разрешено. Custom claims в токенах. Fine-grained authorization (Keycloak Authorization Services — специальный feature).

Вместо того чтобы каждое приложение имело свою систему пользователей + auth + roles — всё в Keycloak. Приложение просто «доверяет» Keycloak через OIDC.

**Плюсы централизации**:
- **SSO** (Single Sign-On) — залогинился один раз, работает во всех приложениях.
- **Управление пользователями в одном месте** — админ создаёт user'а в Keycloak, автоматически доступен во всех apps.
- **Стандартные протоколы** — приложения через OIDC/SAML, работают с любым IdP не только Keycloak.
- **Готовые UI** — login page, self-service (change password, forgot password), account management.
- **Enterprise features** — 2FA, brute force detection, session management, audit log.

**Стек Keycloak**:
- Написан на **Java** (WildFly до v17, Quarkus с v17+). Quarkus сделал startup быстрее в 10 раз.
- Хранит данные в БД: PostgreSQL, MySQL, MariaDB, MSSQL, Oracle. Дефолт для dev — embedded H2 (в prod — только внешняя БД).
- Cache через **Infinispan** (в HA — distributed cache между нодами).
- Frontend — Freemarker templates + JavaScript (можно кастомизировать через themes).

## Realm: центральная концепция изоляции

**Realm** — независимое «пространство» в Keycloak. Свои users, свои clients, свои roles, свои settings. Realms полностью изолированы — user из realm A не может login в realm B (даже с тем же паролем — они разные users).

Аналогия: realm — отдельный «tenant» или «организация». Multi-tenant Keycloak — много realms для разных клиентов/проектов.

**Master realm** — специальный, создаётся автоматически. Содержит admin пользователей самого Keycloak. **Не использовать для приложений**. Только для управления Keycloak.

**Приложенческие realms** — создаются админом. Например `myapp`, `enterprise`, `production`.

**Когда создавать разные realms**:

- **Полная изоляция data и users**. Например SaaS: каждый клиент = свой realm. Пользователи разных клиентов не знают друг о друге.
- **Разные политики**. Realm A требует MFA для всех, realm B — нет. Разные session timeouts, разные password policies.
- **Разные окружения**. dev / staging / prod — обычно отдельные realms (иногда отдельные Keycloak instances).
- **Разные проекты внутри компании**. Разработка A на своём realm, разработка B на своём. Users в обоих не пересекаются.

**Когда НЕ создавать realm на каждое приложение**:

- Если приложения одной команды используют общих users → один realm с несколькими clients. Правильный подход для микросервисной архитектуры одного продукта.

**Realm settings** — конфигурация:
- **Login**: registration allowed? Forgot password? Remember me? Email as username?
- **Tokens**: access_token lifespan (5 мин, 15 мин, 1 час?), refresh_token lifespan, SSO session (idle + max).
- **Security defenses**: brute force protection (после N failed attempts блок), CORS, headers, X-Frame-Options.
- **Password policy**: minimum length, digits, special chars, history (не повторять последние 3), expiration.
- **OTP policy**: TOTP или HOTP, digits, period, algorithm.
- **Themes**: login theme, account theme, admin theme, email theme.

**URLs** — Keycloak имеет много endpoints per realm:

```
https://keycloak.example.com/realms/myapp/                          — realm root
https://keycloak.example.com/realms/myapp/.well-known/openid-configuration
https://keycloak.example.com/realms/myapp/protocol/openid-connect/auth
https://keycloak.example.com/realms/myapp/protocol/openid-connect/token
https://keycloak.example.com/realms/myapp/protocol/openid-connect/userinfo
https://keycloak.example.com/realms/myapp/protocol/openid-connect/certs        — JWKS
https://keycloak.example.com/realms/myapp/protocol/openid-connect/logout
https://keycloak.example.com/realms/myapp/account/                             — user self-service
```

Discovery endpoint `/.well-known/openid-configuration` — стандартный OIDC. Клиент читает его, узнаёт все другие URLs автоматически.

## Clients: приложения работающие с Keycloak

**Client** в Keycloak = **приложение** которое использует Keycloak для аутентификации. Не пользователь, а именно приложение — web-app, mobile-app, backend API, microservice.

Каждое приложение регистрируется как client в Keycloak. Получает `client_id`, опционально `client_secret`.

**Client Types** (Access Type):

**Public** — не имеет secret. Не может безопасно хранить credentials. SPA (React app), mobile apps. Используют Authorization Code + PKCE flow.

**Confidential** — имеет secret. Backend server, batch jobs, microservices. Могут использовать все flows включая Client Credentials.

**Bearer-only** — только принимает tokens, не аутентифицирует пользователей. Например internal microservice который проверяет JWT приходящий от API Gateway. Нет login flow — просто validation.

**Client settings** — что настраивается:

**Basic**:
- **Client ID** — уникальный идентификатор (например `my-web-app`, `my-api`).
- **Name / Description** — для админа.
- **Root URL** — базовый URL приложения. Prefix для остальных URL.
- **Valid redirect URIs** — куда Keycloak может redirect пользователя после логина. **Обязательно** и **строго** проверяется. `https://myapp.com/callback`, wildcards ограниченно.
- **Web Origins** — CORS. Какие origins могут делать AJAX запросы к Keycloak.
- **Admin URL** — для backchannel logout уведомлений.

**Access Type** (упомянуто выше).

**Authentication**:
- **Standard Flow Enabled** — Authorization Code flow.
- **Implicit Flow Enabled** — DEPRECATED, не использовать.
- **Direct Access Grants Enabled** — Password Grant. Только для legacy или CLI tools где нельзя browser.
- **Service Accounts Enabled** — Client Credentials flow. Для service-to-service.

**Advanced**:
- **Access Token Lifespan** — переопределить realm-level.
- **Access Token Signature Algorithm** — RS256 / RS384 / RS512 / ES256 / HS256. Обычно RS256.

**Roles** — можно определять роли на уровне client (в отличие от realm-level ролей).

**Mappers** — что попадает в токен (см. ниже).

## Users: пользователи

**User** в Keycloak — пользователь realm'а. Может быть создан вручную, зарегистрирован сам, или федерирован из LDAP/AD.

**Атрибуты user**:
- **Username** — уникальный внутри realm.
- **Email** — обычно уникальный, часто primary login.
- **First name / Last name**.
- **Email verified** — подтвердил ли пользователь email.
- **Enabled** — активен ли аккаунт (disabled = не может login).
- **Custom attributes** — key-value (department, phone, employee_id, что угодно).

**Credentials**:
- **Password** — hashed, PBKDF2 (стандарт Keycloak). Может быть temporary (заставит поменять при первом логине).
- **OTP** — TOTP через Google Authenticator, Microsoft Authenticator, etc.
- **WebAuthn** — hardware keys, biometrics.
- **X.509 certificates** — client cert authentication.

**Sessions** — activation sessions пользователя. Можно посмотреть где залогинен, force logout.

**Consents** — какие clients получили какие scopes от пользователя (для UI где пользователь может отозвать доступ).

**Federated identity** — если user пришёл через social login (Google/Facebook) — здесь ссылка на внешний identity.

## Roles: realm vs client, composite

Keycloak имеет **два уровня ролей**:

**Realm Roles** — глобальные для realm. Например `admin`, `user`, `manager`. Один пользователь может иметь несколько.

**Client Roles** — специфичные для client. Например для client `orders-api`: `orders:read`, `orders:write`, `orders:admin`. Роли одного client не видны другому.

**Как выбирать**:
- **Realm role** — общая концепция роли для всей системы. Пример: `ADMIN`, `USER`, `MANAGER`.
- **Client role** — специфичная для приложения роль. Пример: `orders-api:read`.

В простых случаях достаточно realm ролей. В сложных enterprise с многими приложениями — client roles для чистоты (разные приложения имеют свои роли, не пересекаются).

**Composite Roles** — роль включающая другие роли. Например `super-admin` — composite, включает `admin` + `orders:admin` + `users:admin`. При назначении super-admin пользователю — автоматически даются все composite roles.

Полезно для иерархии: `super-admin > admin > user`. Не назначать все roles руками — назначить одну composite.

**В JWT** роли попадают через mappers. По умолчанию Keycloak кладёт:
- Realm roles → `realm_access.roles`
- Client roles → `resource_access.<client_id>.roles`

Пример payload:

```json
{
  "realm_access": {
    "roles": ["ADMIN", "USER"]
  },
  "resource_access": {
    "orders-api": {
      "roles": ["orders:read", "orders:write"]
    },
    "users-api": {
      "roles": ["users:read"]
    }
  }
}
```

Приложение (Resource Server) извлекает роли из этих полей для проверки прав.

## Groups: группы пользователей

**Group** — коллекция users. Group может иметь **roles** — все members получают эти roles.

Пример: group `finance-team` имеет роли `finance:read`, `finance:write`. Добавили user в group — автоматически получил роли. Удалили — потерял.

**Nested groups** — иерархия. `company/finance/accounting`. Может наследовать роли от parent.

**Attributes on groups** — custom attributes на группе (например `department: finance`), можно mapping в JWT для пользователей группы.

Когда использовать groups vs roles напрямую:
- **Roles** — для доступа к функциям.
- **Groups** — для организационной структуры (отделы, команды). Далее — roles на группу.
- Пример: пользователь user1 в group `finance-team`. Group имеет роль `finance:read`. User1 автоматически имеет `finance:read`.

## Mappers: контроль над содержимым токена

**Mapper** определяет что и как попадает в JWT (или SAML assertion).

Каждый client имеет свой набор mappers. Разные clients могут получать разные claims для одного и того же user.

**Типы mappers**:

**User Property Mapper** — берёт built-in атрибут user (email, firstName, lastName) и кладёт в токен под указанным claim name.

**User Attribute Mapper** — берёт custom атрибут user (department, phone) и кладёт в токен.

**Role Name Mapper** — переименовывает роли в токене. Например realm role `admin` в токене как `ROLE_ADMIN` (Spring Security convention).

**Group Membership Mapper** — кладёт группы user в токен. Обычно как `groups: ["finance-team", "developers"]`.

**Audience Mapper** — добавляет audience в токен. Полезно когда токен должен работать для нескольких API.

**Script Mapper** — самый гибкий. JavaScript код evaluating claim value. Может делать сложную логику.

**Пример**: хочу чтобы в JWT попадал `tenant_id` из custom attribute user.

Настройка mapper:
- Type: **User Attribute**
- User Attribute: `tenant_id`
- Token Claim Name: `tenant_id`
- Claim JSON Type: `String`
- Add to ID token: yes
- Add to access token: yes

Результат — при login user'а с `tenant_id=acme-corp` в JWT появляется `"tenant_id": "acme-corp"`.

## Endpoints Keycloak: OIDC + admin

Keycloak экспозит стандартные OIDC endpoints + свои admin endpoints.

**Стандартные OIDC** (per realm):

- `/.well-known/openid-configuration` — discovery.
- `/protocol/openid-connect/auth` — authorization endpoint (redirect для login).
- `/protocol/openid-connect/token` — token endpoint (обмен code на token, refresh).
- `/protocol/openid-connect/userinfo` — user info endpoint.
- `/protocol/openid-connect/certs` — JWKS (публичные ключи для проверки JWT).
- `/protocol/openid-connect/logout` — logout endpoint.
- `/protocol/openid-connect/token/introspect` — token introspection (для opaque tokens).
- `/protocol/openid-connect/revoke` — revocation endpoint.

**SAML endpoints** (если используется SAML):
- `/protocol/saml/descriptor` — SAML metadata.
- `/protocol/saml` — SAML SSO.

**Admin REST API** (`/admin/realms/...`) — управление всем через HTTP:
- Users: create, update, delete, search, reset password.
- Clients: create, update, delete, list secrets.
- Roles: create, assign, list.
- Groups: create, add/remove members.
- Sessions: list, revoke.
- Events: audit log.

**Пример вызова Admin API** — создать пользователя:

```http
POST https://keycloak/admin/realms/myapp/users
Authorization: Bearer <admin_access_token>
Content-Type: application/json

{
  "username": "ivan",
  "email": "ivan@example.com",
  "enabled": true,
  "firstName": "Иван",
  "lastName": "Иванов",
  "credentials": [
    {
      "type": "password",
      "value": "TempPass123!",
      "temporary": true
    }
  ]
}
```

Для этого нужен admin token — Client Credentials flow с service account имеющим admin роли.

## Authentication Flows: кастомизация логина

**Authentication flow** — последовательность шагов при login. По умолчанию: username/password + optional 2FA. Но можно кастомизировать.

**Built-in flows**:
- **Browser** — стандартный login через browser (username + password).
- **Direct Grant** — для Direct Access Grants (Password Grant).
- **Registration** — регистрация нового user.
- **Reset Credentials** — forgot password.
- **First Broker Login** — при первом login через external IdP (Google, etc).

Каждый flow — граф steps:
- **Required** — обязательный шаг.
- **Alternative** — один из группы (например email OR SMS).
- **Optional** — можно пропустить.
- **Disabled** — отключён.

**Пример кастомизации Browser flow**:

Стандарт:
```
Cookie (auto-login if session exists) [Alternative]
Kerberos (SSO if available) [Alternative]
Identity Provider Redirector (SSO to external) [Alternative]
Forms:
  Username Password Form [Required]
  Browser - Conditional OTP [Conditional]
    OTP Form [Required]
```

Хочу добавить second factor через SMS. Создать новый flow (или скопировать Browser), добавить шаг:

```
...
Forms:
  Username Password Form [Required]
  Browser - Conditional OTP [Conditional] → OTP Form
  SMS Verification [Required]              ← новый шаг через custom SPI
```

Custom steps реализуются через **Keycloak SPI** (Service Provider Interface) — Java plugins. Компилируешь JAR, кладёшь в `providers/`, Keycloak подхватывает.

**Terms and Conditions**. Добавить `Terms and Conditions` шаг в flow — пользователь при первом login должен принять условия.

**Recaptcha**. Добавить recaptcha на forms — защита от bot'ов.

## User Federation: интеграция с LDAP/AD

Enterprise часто уже имеет LDAP или Active Directory с корпоративными пользователями. Не хочется дублировать в Keycloak.

**User Federation** — Keycloak может **читать** users из внешнего LDAP/AD, показывать как своих, аутентифицировать против LDAP.

**Конфигурация LDAP provider** в Keycloak:
- Connection URL: `ldap://ldap.company.com:389`
- Bind DN / Password (Keycloak авторизуется в LDAP).
- Users DN: `ou=people,dc=company,dc=com` (где искать users).
- Username LDAP Attribute: `uid` или `sAMAccountName`.
- User Object Classes: `person, inetOrgPerson, organizationalPerson`.
- Sync Registrations: yes/no (создавать в LDAP при регистрации в Keycloak).
- Import Users: yes/no (import в Keycloak DB как локальные или always fetch from LDAP).
- Edit Mode: `READ_ONLY` / `WRITABLE` / `UNSYNCED`.

**LDAP Mappers** — как атрибуты LDAP mapping'ятся на атрибуты Keycloak user:
- `cn` → firstName + lastName
- `mail` → email
- `memberOf` → groups
- Custom атрибуты → custom Keycloak attributes.

**Sync strategy**:
- **Full sync** — периодически (раз в час например) fetch все users из LDAP в Keycloak.
- **On-demand** — пользователь заходит → Keycloak fetches его данные из LDAP.

Комбинация: on-demand для authentication + periodic full sync для admin console видимости.

**Kerberos SSO** — если корпоративная сеть с Kerberos, Keycloak может делать automatic SSO. Пользователь на domain-joined машине — заходит на Keycloak, автоматически аутентифицирован через Kerberos ticket.

## Identity Brokering: SSO через внешние IdPs

**Identity Brokering** — Keycloak как «мост» к внешним IdPs (Google, Facebook, LinkedIn, corporate SAML).

Сценарий: у тебя своё приложение. Пользователи хотят «Login with Google». Не хочешь напрямую интегрироваться с Google API — используешь Keycloak как посредника.

Настройка:
1. В Keycloak realm добавить Identity Provider (Google).
2. Задать client_id / secret Google (полученные при регистрации приложения в Google Cloud).
3. На login page Keycloak появляется кнопка «Google».

Flow:
1. User нажимает «Google» на Keycloak login page.
2. Keycloak делает OAuth flow с Google (как client).
3. Google аутентифицирует user, redirect обратно к Keycloak.
4. Keycloak получает Google id_token, создаёт (или обновляет) локального user (linked с Google identity).
5. Keycloak выдаёт **свой** токен приложению.

Приложение работает только с Keycloak — единый интерфейс, независимо от того как user залогинился. При смене IdP или добавлении новых (Facebook, Apple) — приложение не меняется.

**Supported protocols**: OIDC, OAuth 2.0, SAML 2.0. Support для Google, Facebook, LinkedIn, GitHub, Instagram, Microsoft, Twitter, etc. plus custom OIDC/SAML providers.

**First Broker Login flow** — что происходит при первом login через external. Создать локального user? Спросить дополнительную info? Auto-link с existing user по email? Настраивается.

## Themes: кастомизация UI

Keycloak UI (login page, account console, admin console, emails) — кастомизируется через **themes**.

Theme — набор:
- **Freemarker templates** (`.ftl` файлы) — HTML структура.
- **CSS** — стили.
- **Images**.
- **JavaScript**.
- **Message properties** (i18n).

**Стандартные themes**: `keycloak` (дефолт), `base` (минимальный).

**Custom theme**:
1. Скопировать стандартный theme.
2. Модифицировать templates под свой brand.
3. Положить в `themes/` директорию Keycloak.
4. В realm settings выбрать custom theme.

**Что кастомизируется**:
- **Login theme** — login/registration/forgot password pages.
- **Account theme** — user self-service (change password, sessions, applications).
- **Admin theme** — admin console.
- **Email theme** — email templates (verification, password reset, magic link).

Для enterprise почти всегда — свой login theme с корпоративными цветами и логотипом.

## Admin REST API: автоматизация

Всё что делается через admin console — можно через REST API. Автоматизация setup, migrations, integrations.

**Аутентификация**:

Создать confidential client с service account роль `realm-management/manage-users` (и другие нужные). Получить token через Client Credentials.

```bash
# Получить admin token
TOKEN=$(curl -X POST \
  https://keycloak/realms/master/protocol/openid-connect/token \
  -d "grant_type=client_credentials" \
  -d "client_id=admin-cli" \
  -d "client_secret=SECRET" \
  | jq -r '.access_token')

# Использовать
curl -H "Authorization: Bearer $TOKEN" \
  https://keycloak/admin/realms/myapp/users
```

**Основные endpoints**:

```
# Users
GET    /admin/realms/{realm}/users                    — list
GET    /admin/realms/{realm}/users/{id}               — get
POST   /admin/realms/{realm}/users                    — create
PUT    /admin/realms/{realm}/users/{id}               — update
DELETE /admin/realms/{realm}/users/{id}               — delete
GET    /admin/realms/{realm}/users?username=xxx       — search

# Roles
POST   /admin/realms/{realm}/roles
PUT    /admin/realms/{realm}/users/{id}/role-mappings/realm    — assign role

# Groups
POST   /admin/realms/{realm}/groups
PUT    /admin/realms/{realm}/users/{id}/groups/{groupId}       — add to group

# Sessions
GET    /admin/realms/{realm}/users/{id}/sessions
POST   /admin/realms/{realm}/users/{id}/logout                 — force logout

# Clients
GET    /admin/realms/{realm}/clients
POST   /admin/realms/{realm}/clients
```

**Использование**:
- **Provisioning** — создание users при onboarding из HR системы.
- **Bulk operations** — массовое обновление attributes.
- **Migrations** — перенос users при apgrade.
- **Integrations** — CRM создаёт user когда добавили нового customer.

**SDKs / Libraries**:
- **Keycloak Admin Java Client** — официальный Java SDK.
- **python-keycloak** — Python.
- **@keycloak/keycloak-admin-client** — Node.js.

## HA setup: production deployment

Единичный Keycloak — SPOF. В prod — cluster.

**Компоненты HA**:

**Multiple Keycloak nodes** — за load balancer'ом. Обычно 2-3 nodes.

**Shared database** — PostgreSQL (или другая). Все Keycloak nodes используют одну БД. Realm data, users, sessions — хранятся здесь.

**Distributed cache** — Infinispan. Keycloak кэширует много (users, sessions, tokens). В HA cache должен быть распределённый (изменение на node A → cache updated на node B). Infinispan работает через JGroups (UDP multicast или TCP unicast).

**Sticky sessions на load balancer** — рекомендуется. User залогинился → его requests идут на тот же node (пока node доступен). Уменьшает cache misses и session invalidations.

**Health checks**:
- Liveness: `/health/live` — процесс жив.
- Readiness: `/health/ready` — готов принимать трафик (БД доступна, cache initialized).

**Backup БД** — жизненно критично. Users, roles, config, sessions — всё в БД. Потеряли БД = потеряли identity infrastructure. Ежедневные backups минимум.

**Пример docker-compose для 2-node cluster**:

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
      KC_CACHE_STACK: kubernetes
      # ... TLS, etc.
    depends_on: [db]
  
  keycloak-2:
    # same config
  
  nginx:
    image: nginx
    # LB config with sticky sessions
```

**Kubernetes deployment** — есть официальные Helm charts. Работает через **Operator** для сложных сценариев.

## Мониторинг Keycloak

Что мониторить:

**Метрики** — Keycloak Quarkus версии экспозит Prometheus metrics на `/metrics`. Основные:
- HTTP request rate, latency, errors.
- Database connection pool usage.
- Active sessions count.
- Login rate (success / failure).
- Token issuance rate.

**Логи**:
- Успешные и failed logins.
- Errors при валидации токенов.
- Slow queries к БД.

**Events**:

Keycloak умеет писать **events** — audit trail всех действий. Login, logout, register, password change, etc. Настраивается в realm settings → Events.

Storage:
- **JBOSS logging** (в файл или journal).
- **DB storage** (в основную БД, для admin console просмотра).
- **Custom SPI** — отправка в Kafka, ELK, custom analytics.

**Health checks**:
- `/health/live`, `/health/ready`.
- `/metrics` для Prometheus.

## Best practices для prod

**Отдельная БД для Keycloak** — не смешивать с приложенческой. Разные lifecycle, разные требования backup.

**Внешний cache для sessions** — не полагаться на in-memory Infinispan для критичных данных. Опция — offline sessions в БД.

**HTTPS обязательно** — Keycloak передаёт tokens, passwords. Никакого HTTP в production. TLS termination на LB или Keycloak напрямую.

**Реалистичные token lifespan**:
- Access token: 5-15 минут (не час-два как дефолт).
- Refresh token: часы для SPA, недели для mobile.
- SSO session idle: 30 мин типично.
- SSO session max: 8-12 часов (рабочий день).

**Brute force protection** — включить. После 5 failed attempts блок на 15 минут.

**Password policy** — минимум 12 символов, разные символы, expiration.

**MFA обязателен для admin ролей** — минимум. Желательно для всех.

**Backup БД ежедневно**. Тестировать restore процедуру раз в квартал.

**Мониторинг** — Prometheus + Grafana dashboard. Alert на failed logins spike (brute force?), slow response, DB connection issues.

**Обновления** — Keycloak регулярно выпускает патчи. Особенно security. Обновляться в течение месяца-двух от release.

**Sizing**:
- Development: 512 MB heap, 1 CPU.
- Production small (< 10K users, < 100 requests/sec): 2 GB heap, 2 CPU.
- Production medium: 4-8 GB heap, 4 CPU, HA cluster.
- Large enterprise (100K+ users): 8+ GB heap, 4+ nodes cluster.

## Общие ошибки в проде

**Дефолтный `master` realm для приложений**. Master для admin only. Создать отдельный realm для каждого приложения (или группы).

**Public client для backend'а**. Backend имеет secret storage → должен быть confidential client. Public только для SPA/mobile.

**Слишком долгий access_token TTL**. Дефолт Keycloak — 5 минут, часто увеличивают до часа-двух «для удобства». Плохо — компрометация токена = 1-2 часа доступа. Держать 5-15 минут, использовать refresh.

**Отсутствие backchannel logout**. Пользователь logout из одного приложения → другие приложения не знают. Использовать front-channel или back-channel logout.

**Хранение client_secret в git**. Never. Secrets в Vault / K8s secrets / environment variables только.

**Один Keycloak instance без HA в prod**. SPOF, потеря — весь identity недоступен, приложения не работают. Минимум 2 node cluster + shared DB.

**Забытые sessions пользователей**. При инциденте (компрометация) — force logout всех sessions пользователя. `POST /users/{id}/logout` через Admin API.

**LDAP sync без monitoring**. LDAP sync тихо ломается → новые users не появляются в Keycloak → они не могут login. Мониторить sync errors.

## Заключение

**Keycloak** — open-source IAM сервер: Identity Provider + Authorization Server + user management в одном. Стандарт для enterprise Java. Поддерживает OAuth 2.0, OIDC, SAML 2.0.

**Realm** — центральная концепция изоляции. Свои users, clients, roles. Master realm только для admin Keycloak, приложения — в своих realms. Multi-tenant SaaS = realm per tenant.

**Clients** — приложения работающие с Keycloak. Public (SPA, mobile — без secret) vs Confidential (backend, service — с secret) vs Bearer-only (только validation, no login flow). Каждый клиент имеет свои redirect URIs, roles, mappers.

**Users** — хранятся в Keycloak или федерируются из LDAP/AD. Атрибуты, credentials (password, OTP, WebAuthn, x509), sessions, consents.

**Roles**:
- **Realm roles** — глобальные для realm.
- **Client roles** — специфичные для клиента.
- **Composite roles** — включают другие роли (иерархия).

В JWT: realm roles в `realm_access.roles`, client roles в `resource_access.<client>.roles`.

**Groups** — коллекции users с общими roles/attributes. Иерархические, для организационной структуры.

**Mappers** — контроль что попадает в JWT. User property, user attribute, role name, group membership, audience, script mappers. Custom claims — через user attribute mapper + custom атрибут user.

**Endpoints Keycloak**:
- Стандартные OIDC: authorize, token, userinfo, jwks, discovery, logout, introspect, revoke.
- Admin REST API — управление users, clients, roles, groups, sessions через HTTP.

**Authentication flows** — кастомизация login. Добавить 2FA, terms, recaptcha, custom SPI через Java plugins.

**User Federation** — интеграция с LDAP/AD. Keycloak читает users из LDAP, authenticates against LDAP. Mappers для атрибутов. Sync strategies (full periodic + on-demand). Kerberos SSO для corporate networks.

**Identity Brokering** — Keycloak как «мост» к внешним IdPs (Google, Facebook, corporate SAML). Приложение работает только с Keycloak, независимо от того как user залогинился.

**Themes** — кастомизация UI login, account, admin, email. Freemarker templates + CSS. Для enterprise — свой брендовый theme.

**Admin REST API** — полная автоматизация. Provisioning users, bulk operations, migrations. Java Admin Client + другие SDK. Аутентификация через Client Credentials с service account имеющим admin роли.

**HA setup**: multiple nodes + shared DB (PostgreSQL) + distributed cache (Infinispan) + sticky sessions на LB. Backup БД жизненно критичен.

**Мониторинг**: Prometheus metrics (`/metrics`), health checks (`/health/live`, `/health/ready`), events (login attempts, admin actions), Grafana dashboards.

**Best practices**: отдельная БД, HTTPS обязательно, короткие access token TTL (5-15 мин), refresh token rotation, brute force protection, password policy, MFA для admin, ежедневный backup, регулярные обновления.

OAuth/OIDC протокол — файл 117. JWT детально — 118. Spring Security integration — 120. Здесь была глубина по Keycloak как реализации identity provider.
