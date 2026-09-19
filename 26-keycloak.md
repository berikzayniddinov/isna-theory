# 26. Keycloak: realm, client, users, roles

Что такое Keycloak, его сущности, как настроить, как выдаёт токены.

---

## 1. Что такое Keycloak

**Keycloak** — open-source Identity and Access Management (IAM) от RedHat.

Роль:
- **OAuth2 / OIDC provider** — выдаёт токены.
- **SAML 2.0 provider** (для enterprise).
- **User management** — регистрация, пароли, роли.
- **Federation** — LDAP, Active Directory, соцсети.
- **Admin console** — веб-морда для управления.
- **Adapters** — библиотеки для Java, JS, Python (deprecated).

В ИСНА — центральный IAM. Все микросервисы валидируют токены оттуда.

---

## 2. Основные сущности

Иерархия:
```
Keycloak Server
    │
    └── Realm  (изолированный контейнер)
         │
         ├── Users            (пользователи)
         ├── Groups           (группы пользователей)
         ├── Roles (Realm)    (realm-level roles)
         ├── Clients          (приложения)
         │    └── Client Roles (роли специфичные клиенту)
         ├── Identity Providers (federation: LDAP, Google, GitHub)
         ├── Authentication Flows
         ├── User Federation  (external stores)
         ├── Client Scopes    (переиспользуемые наборы claims)
         └── Events           (audit)
```

### 2.1 Realm

**Изолированный контейнер**. Свои users, clients, roles, keys. Разные realm — независимые «пространства».

Пример:
- `master` — для админов Keycloak.
- `knp` — для пользователей КНП.
- `internal` — для сотрудников.

Токен из одного realm НЕ валиден в другом.

### 2.2 Users

Пользователь realm'а:
- Username / email.
- Пароль (хешированный).
- Атрибуты (custom fields).
- Roles / groups.
- Enabled / disabled.

### 2.3 Groups

Логическая группа пользователей. Наследует roles.

### 2.4 Roles

**Realm role** — общая для realm (`admin`, `user`).
**Client role** — специфична клиенту (`isnaKnp:CREATE_FNO`).

У пользователя может быть много ролей: и realm, и client-специфичных.

### 2.5 Clients

Приложения, использующие Keycloak.

**Access type**:
- **Public** — SPA, mobile. Нет secret.
- **Confidential** — backend, есть client_secret.
- **Bearer-only** — только resource server (проверяет токены, не логинит).

**Standard flow enabled** — Authorization Code flow.
**Implicit flow** — deprecated.
**Direct access grants** — Password grant (для legacy).
**Service accounts enabled** — Client Credentials (M2M).

### 2.6 Client Scopes

Переиспользуемый набор claims, mapper'ов, ролей.

Например `email` scope добавляет claim `email`, `email_verified`. `profile` — `given_name`, `family_name`, `picture`.

---

## 3. Mappers

**Mapper** — как поле пользователя → в claim JWT.

Типы:
- **User Attribute** — атрибут → claim.
- **User Property** — стандартное поле (username, email) → claim.
- **Group Membership** — список групп → claim `groups`.
- **Role Mapper** — роли → claim `realm_access.roles` / `resource_access.<client>.roles`.
- **Hardcoded Claim** — фиксированное значение.
- **Script** — JS/Java для сложной логики.

По умолчанию Keycloak кладёт роли специфично:
```json
{
    "realm_access": {
        "roles": ["admin", "user"]
    },
    "resource_access": {
        "isna-knp-integration": {
            "roles": ["CREATE_FNO"]
        }
    }
}
```

Spring Security по умолчанию НЕ знает про эту структуру → надо кастомный `JwtAuthenticationConverter` (см. `27-spring-security-oauth2-keycloak.md`).

---

## 4. Endpoints Keycloak

Основные URL:
```
Base: https://keycloak.isna/realms/knp

.well-known:
  https://keycloak.isna/realms/knp/.well-known/openid-configuration

Authorization (для UI login flow):
  https://keycloak.isna/realms/knp/protocol/openid-connect/auth

Token:
  https://keycloak.isna/realms/knp/protocol/openid-connect/token

Userinfo:
  https://keycloak.isna/realms/knp/protocol/openid-connect/userinfo

JWKS (публичные ключи):
  https://keycloak.isna/realms/knp/protocol/openid-connect/certs

Logout:
  https://keycloak.isna/realms/knp/protocol/openid-connect/logout

Introspection (для opaque tokens):
  https://keycloak.isna/realms/knp/protocol/openid-connect/token/introspect

Revocation:
  https://keycloak.isna/realms/knp/protocol/openid-connect/revoke

Admin REST API:
  https://keycloak.isna/admin/realms/knp/users
```

---

## 5. Стандартный flow

### 5.1 Authorization Code (для UI)

Пользователь на knp.kgd.gov.kz кликает «Войти»:

1. **Redirect на Keycloak**:
   ```
   GET https://keycloak.isna/realms/knp/protocol/openid-connect/auth?
       client_id=isna-knp-front&
       redirect_uri=https://knp.kgd.gov.kz/callback&
       response_type=code&
       scope=openid profile email&
       state=<random>&
       code_challenge=<sha256(code_verifier)>&
       code_challenge_method=S256
   ```

2. **Пользователь вводит логин/пароль на Keycloak-странице**.

3. **Redirect обратно с code**:
   ```
   GET https://knp.kgd.gov.kz/callback?code=<code>&state=<state>
   ```

4. **Frontend/backend обменивает code на token**:
   ```
   POST https://keycloak.isna/realms/knp/protocol/openid-connect/token
   grant_type=authorization_code
   code=<code>
   redirect_uri=https://knp.kgd.gov.kz/callback
   client_id=isna-knp-front
   code_verifier=<original>
   ```

5. **Ответ**:
   ```json
   {
       "access_token": "eyJ...",
       "refresh_token": "eyJ...",
       "id_token": "eyJ...",
       "expires_in": 300,
       "refresh_expires_in": 1800,
       "token_type": "Bearer"
   }
   ```

6. Frontend хранит токены, использует access_token в API.

### 5.2 Client Credentials (M2M)

Один сервис (`isna-knp-fno`) вызывает другой (`isna-knp-user`).

1. **Получить токен**:
   ```
   POST https://keycloak.isna/realms/knp/protocol/openid-connect/token
   grant_type=client_credentials
   client_id=isna-knp-fno
   client_secret=<secret>
   ```

2. **Ответ**:
   ```json
   {
       "access_token": "eyJ...",
       "expires_in": 300
   }
   ```

3. **API вызов**:
   ```
   GET https://isna-knp-user/api/users/123
   Authorization: Bearer <access_token>
   ```

Реальный кейс из ИСНА `taxreport21-java21-runtime-regressions` — bearer-канал НЗ падал OAuth NoSuchMethodError (spring-security jose skew).

### 5.3 Refresh

```
POST /token
grant_type=refresh_token
refresh_token=<old>
client_id=isna-knp-front
```

Возвращается новый access + иногда rotated refresh.

---

## 6. ЭЦП flow (Казахстан-specific)

В ИСНА пользователи не только логин/пароль, но и **ЭЦП (Kalkan)**.

Обычно:
1. Frontend просит пользователя подписать «challenge» ЭЦП.
2. Отправляет подпись на backend.
3. Backend валидирует ЭЦП, извлекает данные пользователя (ИИН).
4. Обменивает у Keycloak на токен через custom flow (Direct Grant с custom authenticator).

Технически — свой Authentication Flow в Keycloak с custom Authenticator SPI.

---

## 7. Admin API

Полный REST для управления:

```
GET  /admin/realms/knp/users
POST /admin/realms/knp/users
PUT  /admin/realms/knp/users/{id}
GET  /admin/realms/knp/users/{id}/role-mappings/realm
POST /admin/realms/knp/users/{id}/role-mappings/realm
GET  /admin/realms/knp/clients
```

Требует admin-token. Обычно service account с client_credentials.

Использование:
- Автопровижнинг пользователей.
- Массовые операции (список активных, отзыв доступа).
- Custom-скрипты для интеграции.

Есть Java admin-client:
```java
Keycloak kc = KeycloakBuilder.builder()
    .serverUrl("https://keycloak.isna")
    .realm("master")
    .grantType(OAuth2Constants.CLIENT_CREDENTIALS)
    .clientId("admin-cli")
    .clientSecret("...")
    .build();

kc.realm("knp").users().search("berik");
```

---

## 8. Federation

Keycloak может подключаться к внешним источникам пользователей:
- **LDAP / Active Directory** — синхронизация users.
- **Kerberos**.
- **SAML / OIDC** — другие IdP (Google, GitHub).

Пользователи из LDAP появляются в Keycloak, могут логиниться под своими credentials.

---

## 9. Authentication Flows

Конфигурируемая цепочка шагов при логине. Стандартные:
- Cookie (SSO)
- Kerberos (SPNEGO)
- Identity provider redirect
- Forms (username/password)
- OTP
- WebAuthn
- Password reset

Можно построить свою:
- Password → OTP (mandatory) → captcha → success.

Или custom Authenticator (Java class) — например «войти по ЭЦП».

---

## 10. Themes

Кастомизация UI (login page, email templates). HTML/CSS/JS + FreeMarker templates.

В ИСНА обычно свои темы под брендинг.

---

## 11. Adapters (DEPRECATED)

Раньше Keycloak давал client-адаптеры для Java (`keycloak-spring-boot-starter`). Они делали всё сами: интегрировались с Spring Security, парсили токены, извлекали роли.

**С 2022 адаптеры deprecated**. Рекомендуется — **Spring Security OAuth2 Resource Server** (стандартный).

Плюсы миграции:
- Стандартный подход, не Keycloak-специфичный.
- Легче обновлять Spring Boot.
- Работает с любым OIDC provider.

Минусы:
- Роли Keycloak в claim `realm_access` / `resource_access` — Spring не знает про это по умолчанию. Нужен кастомный `JwtAuthenticationConverter`.

В ИСНА постепенно перешли или переходят с adapter на Spring Security Resource Server (см. `27-spring-security-oauth2-keycloak.md`).

---

## 12. Настройка в realm (пример)

### 12.1 Создать realm

Admin console → Add realm → `knp`.

### 12.2 Создать клиент

- Client ID: `isna-knp-integration`
- Client protocol: `openid-connect`
- Access type: `bearer-only` (только проверяет токены)

Для UI:
- Client ID: `isna-knp-front`
- Access type: `public`
- Valid redirect URIs: `https://knp.kgd.gov.kz/*`
- Web Origins: `+` (для CORS)
- Standard flow: on
- PKCE: on

Для M2M:
- Client ID: `isna-knp-fno-service`
- Access type: `confidential`
- Service accounts: enabled
- Standard flow: off
- Direct access grants: off

### 12.3 Создать роли

Realm roles: `admin`, `user`, `taxpayer`.
Client roles на `isna-knp-integration`: `CREATE_FNO`, `VIEW_FNO`, `KNP_PERM_CREATE_NZ_N07`.

### 12.4 Создать пользователя

- Username: berik
- Email, name.
- Set password (temporary).
- Role mapping: `taxpayer`.

### 12.5 Mappers

- Realm roles → `realm_access.roles` (default).
- Client roles → `resource_access.<client>.roles` (default).
- Group membership → `groups` (custom).

---

## 13. HA и мониторинг

### 13.1 HA

Keycloak — stateful. Session хранится в:
- Infinispan cache (embedded).
- External (кластерная конфигурация).

В production — cluster с 3+ node + внешний БД + Infinispan replication.

### 13.2 БД

Persistent store — обычно PostgreSQL. Хранит:
- Users.
- Realms.
- Clients.
- Roles.
- Sessions (опционально).

### 13.3 Мониторинг

- Metrics endpoint `/metrics` (Prometheus).
- Events log (login, failed, token refresh).
- Admin events (user create/delete).

---

## 14. Собесные вопросы

1. **Что такое Keycloak?** — OAuth2/OIDC provider, IAM, open-source.
2. **Что такое realm?** — Изолированный контейнер users/clients/roles.
3. **Разница realm role и client role?** — Realm role — общая; client role — специфична приложению.
4. **Access types клиента?** — Public (SPA), confidential (backend с secret), bearer-only (только validation).
5. **Что такое mapper?** — Правило как поле пользователя попадает в claim JWT.
6. **Где Keycloak кладёт роли в JWT?** — `realm_access.roles` и `resource_access.<client>.roles`.
7. **Что такое discovery endpoint?** — `/realms/<r>/.well-known/openid-configuration`.
8. **JWKS в Keycloak?** — `/realms/<r>/protocol/openid-connect/certs` — публичные ключи.
9. **Как получить токен для M2M?** — Client Credentials flow с client_secret.
10. **Что такое Client Scope?** — Переиспользуемый набор mappers / claims.
11. **Adapters deprecated — что взамен?** — Spring Security OAuth2 Resource Server.
12. **Как интегрировать с LDAP?** — User Federation → LDAP; users синхронизируются.
13. **Что такое Authentication Flow?** — Настраиваемая цепочка шагов при логине (пароль → OTP → ...).

---

## Итог

- **Keycloak** = центральный IAM в ИСНА.
- **Realm** = изоляция пространства.
- **Client** = приложение (public / confidential / bearer-only).
- **Realm + Client roles** — иерархия ролей.
- **Mappers** — поля пользователя → JWT claims.
- **Discovery + JWKS** — auto-конфиг Spring Security.
- **Admin REST API** для управления.
- **Adapters deprecated** → Spring Security OAuth2 Resource Server.

Следующий — `27-spring-security-oauth2-keycloak.md`.
