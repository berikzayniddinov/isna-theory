# 26. Keycloak: realm, client, users, roles

## Что такое Keycloak

Keycloak это open source Identity and Access Management (IAM) solution от Red Hat. Предоставляет централизованное управление аутентификацией и авторизацией для множества приложений через стандартные протоколы. В экосистеме КНП Keycloak играет роль центрального IAM — все микросервисы валидируют выданные им токены для authentication и authorization.

Роль Keycloak в архитектуре покрывает несколько функций одновременно. OAuth2 и OpenID Connect provider — выдаёт tokens по стандартным flows, обслуживает discovery endpoint, публикует JWKS для валидации токенов. SAML 2.0 provider для enterprise интеграций где требуется этот протокол. User management — регистрация пользователей, пароли, сброс, атрибуты, роли. Federation с внешними user stores — LDAP, Active Directory, социальные сети через standard identity providers. Admin console — web UI для управления всеми аспектами системы. Adapters — client-side библиотеки для разных языков, хотя они deprecated в пользу стандартных OAuth2 клиентов.

Deployment типично включает несколько нод Keycloak в кластере плюс PostgreSQL как persistent store плюс Infinispan для sessions cache. Такая конфигурация обеспечивает высокую доступность IAM что критично поскольку падение Keycloak означает что все связанные с ним приложения теряют возможность authentication.

## Основные сущности

Keycloak организует данные в иерархическую структуру. Понимание этой иерархии критически важно для правильной настройки системы.

```
Keycloak Server
    │
    └── Realm (изолированный контейнер)
         │
         ├── Users              (пользователи с credentials)
         ├── Groups             (логические группы пользователей)
         ├── Realm Roles        (роли уровня realm)
         ├── Clients            (приложения-потребители)
         │    ├── Client Roles  (роли специфичные клиенту)
         │    ├── Mappers       (маппинг данных в claims)
         │    ├── Scopes        (scope конфигурация)
         │    └── Service Account (для M2M)
         ├── Identity Providers (federation с внешними IdP)
         ├── User Federation    (LDAP, Kerberos, custom stores)
         ├── Authentication Flows (customizable login sequences)
         ├── Client Scopes      (переиспользуемые наборы claims)
         ├── Roles Mapper       (правила маппинга ролей)
         └── Events             (audit trail)
```

Realm это фундаментальная единица изоляции в Keycloak. Каждый realm представляет собой независимый контейнер со своими пользователями, клиентами, ролями, keys для подписи tokens. Разные realm полностью изолированы — token выданный одним realm никогда не будет валиден в другом. Типичное использование включает master realm для администрирования самого Keycloak, отдельные realm для разных бизнес доменов или клиентских организаций, dev/staging/prod realm для разных окружений.

Практический пример из КНП. Master realm для админов Keycloak. Realm knp для пользователей налоговой системы — налогоплательщики, инспекторы, администраторы. Realm internal для сотрудников службы. Полная изоляция означает что compromise одного realm не влияет на другие, разные policies аутентификации применимы к разным аудиториям.

Users это записи о людях или service accounts. Каждый user имеет username, email, набор атрибутов включая custom fields определённые администратором. Пароль хранится в hashed виде через настроенный algorithm. Enabled флаг позволяет временно блокировать аккаунт без удаления. Атрибуты используются в mappers для наполнения claims в токенах.

Groups это логические объединения пользователей. Пользователь может входить в несколько групп. Groups наследуют роли — пользователь автоматически получает все роли своих групп. Полезно для управления большими наборами пользователей — вместо назначения ролей каждому по одному можно управлять членством в группе. Иерархические groups поддерживаются — subgroup наследует roles родительской group.

Roles разделяются на два типа. Realm roles определяются на уровне realm и применимы ко всем клиентам этого realm. Стандартный пример — admin, user, taxpayer как общие категории. Client roles определяются в контексте конкретного client и специфичны для этого приложения. Например client roles knpKnpIntegration CREATE_FNO, VIEW_FNO, DELETE_FNO для функциональности связанной с ФНО. Пользователь может иметь роли обоих типов одновременно, все они попадают в токен через соответствующие mappers.

Clients представляют приложения использующие Keycloak для authentication. Каждый client имеет client_id как identifier плюс возможно client_secret для confidential clients. Access type определяет тип клиента и влияет на доступные grant types. Public клиенты для SPA, mobile apps — не могут хранить secret. Confidential клиенты для backend приложений — имеют client_secret. Bearer-only клиенты для чистых resource servers которые только валидируют tokens без инициации login flow.

Client настройки определяют разрешённые OAuth2 flows. Standard flow enabled активирует Authorization Code flow. Implicit flow deprecated и обычно отключено. Direct access grants соответствует password grant, обычно отключено кроме legacy migrations. Service accounts enabled активирует Client Credentials flow для M2M сценариев.

Client Scopes представляют переиспользуемые конфигурации mappers и требуемых claims. Scope может быть default (всегда включён в токены client) или optional (включается только когда запрошен). Стандартные OIDC scopes включают profile для basic user info, email для email address, address для physical address. Custom scopes определяют application specific наборы claims. Использование scopes упрощает управление — общие sets claims определяются один раз и переиспользуются в разных clients.

## Mappers

Mapper определяет правило как данные из user record попадают в claims JWT токенов. Без mapper токен содержит только базовые claims — sub, exp, iat. Mappers добавляют custom claims делающие токен полезным для application.

Типы mappers покрывают разные источники данных. User Attribute Mapper берёт значение custom attribute пользователя и помещает в claim. User Property Mapper использует стандартные поля пользователя (username, email, firstName) как claims. Group Membership Mapper добавляет список групп пользователя как claim обычно groups. Role Mapper специализированный для ролей, помещает их в claim обычно как список.

Hardcoded Claim Mapper вставляет фиксированное значение независимо от пользователя. Полезно для marker claims идентифицирующих tenant или environment. Script Mapper использует JavaScript или Java для complex логики — вычисляемые claims, transformations, conditional inclusion. Мощный но требует care потому что скрипт выполняется на каждом token issuance.

Keycloak по умолчанию использует специфическую структуру для ролей в токенах. Realm roles попадают в realm_access.roles как array. Client roles идут в resource_access.<client_id>.roles также как array. Пример структуры claims:

```json
{
    "sub": "b3f2a1c4-...",
    "preferred_username": "berik",
    "email": "berik@example.com",
    "realm_access": {
        "roles": ["admin", "user"]
    },
    "resource_access": {
        "isna-knp-integration": {
            "roles": ["CREATE_FNO", "VIEW_FNO"]
        },
        "isna-knp-front": {
            "roles": ["UI_ACCESS"]
        }
    }
}
```

Spring Security по default не понимает эту структуру — читает только scope claim который стандартный OAuth2 scope. Для работы с Keycloak-специфичной структурой ролей нужен custom JwtAuthenticationConverter преобразующий эти claims в GrantedAuthorities. Это одна из основных задач при интеграции Spring Security с Keycloak.

## Endpoints Keycloak

Каждый realm имеет стандартный набор OIDC endpoints. Все URLs формируются от base URL Keycloak плюс имя realm.

```
Base URL для realm:
  https://keycloak.isna/realms/knp

Discovery endpoint (все URLs автоматически):
  https://keycloak.isna/realms/knp/.well-known/openid-configuration

Authorization endpoint (UI login flow):
  https://keycloak.isna/realms/knp/protocol/openid-connect/auth

Token endpoint (обмен code, client credentials, refresh):
  https://keycloak.isna/realms/knp/protocol/openid-connect/token

Userinfo endpoint (OIDC user info):
  https://keycloak.isna/realms/knp/protocol/openid-connect/userinfo

JWKS endpoint (публичные ключи для валидации подписи):
  https://keycloak.isna/realms/knp/protocol/openid-connect/certs

Logout endpoint (RP-initiated logout):
  https://keycloak.isna/realms/knp/protocol/openid-connect/logout

Introspection endpoint (для opaque tokens):
  https://keycloak.isna/realms/knp/protocol/openid-connect/token/introspect

Revocation endpoint (отзыв tokens):
  https://keycloak.isna/realms/knp/protocol/openid-connect/revoke

Admin REST API (управление realm):
  https://keycloak.isna/admin/realms/knp/users
  https://keycloak.isna/admin/realms/knp/clients
  https://keycloak.isna/admin/realms/knp/roles
```

Discovery endpoint критически важен потому что через него клиенты автоматически конфигурируются. Одно поле issuer-uri в Spring Security достаточно — Spring читает discovery document и получает все другие URLs автоматически. Это standard OIDC practice работающий с любым compliant provider не только Keycloak.

## Стандартный Authorization Code flow

Практический пример прохождения полного flow для UI приложения КНП. Пользователь на knp.kgd.gov.kz кликает Войти. Дальше происходит следующая последовательность.

Redirect на Keycloak с параметрами:
```
GET https://keycloak.isna/realms/knp/protocol/openid-connect/auth?
    client_id=isna-knp-front&
    redirect_uri=https://knp.kgd.gov.kz/callback&
    response_type=code&
    scope=openid profile email&
    state=<random-csrf-protection>&
    code_challenge=<sha256-of-verifier-base64url>&
    code_challenge_method=S256
```

Пользователь вводит username и пароль на Keycloak login странице. Никакой пароль не проходит через client application — только на Keycloak. После successful authentication Keycloak redirect обратно на client:
```
GET https://knp.kgd.gov.kz/callback?
    code=<authorization-code>&
    state=<same-state-value>
```

Client проверяет state matches ожидаемый (защита от CSRF). Обменивает code на tokens:
```
POST https://keycloak.isna/realms/knp/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=<authorization-code>&
redirect_uri=https://knp.kgd.gov.kz/callback&
client_id=isna-knp-front&
code_verifier=<original-random-value>
```

Ответ содержит tokens:
```json
{
    "access_token": "eyJhbGci...",
    "refresh_token": "eyJhbGci...",
    "id_token": "eyJhbGci...",
    "expires_in": 300,
    "refresh_expires_in": 1800,
    "token_type": "Bearer"
}
```

Frontend сохраняет tokens securely (HttpOnly cookie для web) и использует access_token в Authorization header для всех API запросов к backend.

## Client Credentials flow

Machine-to-machine коммуникация между микросервисами КНП использует Client Credentials flow. Пример когда isna-knp-fno сервису нужно позвать isna-knp-user API от имени сервиса не пользователя.

Получение токена:
```
POST https://keycloak.isna/realms/knp/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&
client_id=isna-knp-fno-service&
client_secret=<securely-stored-secret>
```

Ответ:
```json
{
    "access_token": "eyJhbGci...",
    "expires_in": 300,
    "token_type": "Bearer",
    "scope": "profile email"
}
```

API вызов с полученным токеном:
```
GET https://isna-knp-user/api/users/123
Authorization: Bearer <access_token>
```

Client Credentials flow не выдаёт refresh_token — client может получить новый access_token в любой момент используя свои credentials. Обычно клиенты кэшируют access_token до его expiration и получают новый когда старый почти истёк, минимизируя нагрузку на Keycloak.

Реальный кейс из КНП — memory taxreport21-java21-runtime-regressions показал что bearer канал НЗ падал с OAuth NoSuchMethodError из-за skew версий spring-security-jose и spring-security-core. Урок — версии всех Spring Security компонентов должны быть согласованы через BOM.

## Refresh Token flow

При истечении access_token client обменивает refresh_token на новый access_token без re-authentication пользователя:
```
POST https://keycloak.isna/realms/knp/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&
refresh_token=<current-refresh-token>&
client_id=isna-knp-front
```

Возвращается новый access_token и опционально rotated refresh_token. Rotation refresh_token это security best practice — каждый refresh инвалидирует старый refresh_token, обнаруживая кражу если атакующий использовал token до legitimate user.

## ЭЦП flow Казахстан-специфика

В КНП пользователи аутентифицируются не только логин/пароль но и электронной цифровой подписью через криптоплагин Kalkan. Это Kazakhstan-specific требование поддерживается через custom authentication flow в Keycloak.

Схема работы. Frontend просит пользователя подписать challenge через ЭЦП plugin. Пользователь подписывает ключом хранящимся в NCA storage или на USB. Frontend отправляет подпись на backend. Backend валидирует ЭЦП, извлекает данные из certificate — ИИН, ФИО. Обменивает у Keycloak на токен через custom flow используя Direct Grant с custom authenticator.

Технически это реализуется через custom Authentication Flow в Keycloak с custom Authenticator SPI написанным на Java. Authenticator реализует логику проверки ЭЦП, извлечения атрибутов пользователя, поиска или создания user record. Deployment требует упаковки Authenticator в JAR и разворачивания в Keycloak providers directory.

Такой подход даёт полную интеграцию ЭЦП с остальной OIDC инфраструктурой. Пользователь после ЭЦП авторизации получает стандартные OIDC tokens которые могут быть использованы во всех сервисах платформы без специальной обработки ЭЦП на каждом уровне.

## Admin REST API

Keycloak предоставляет полноценный REST API для управления realm. API покрывает все operations доступные в Admin console — создание/изменение/удаление users, clients, roles, group memberships, custom attributes.

Основные endpoints:
```
GET  /admin/realms/{realm}/users
POST /admin/realms/{realm}/users
PUT  /admin/realms/{realm}/users/{id}
DELETE /admin/realms/{realm}/users/{id}

GET  /admin/realms/{realm}/users/{id}/role-mappings/realm
POST /admin/realms/{realm}/users/{id}/role-mappings/realm

GET  /admin/realms/{realm}/clients
GET  /admin/realms/{realm}/roles
GET  /admin/realms/{realm}/groups
```

Аутентификация в API через bearer token. Обычно используется service account с client_credentials — специальный client настроенный с admin роли для управления realm. Никогда не использовать admin console credentials в автоматизации.

Использование Admin API покрывает несколько сценариев. Автопровижнинг пользователей при интеграции с внешними системами — новые сотрудники автоматически появляются в Keycloak из HR system. Массовые операции — генерация reports по активным пользователям, bulk revocation прав при security incidents. Custom скрипты для интеграции — например синхронизация atributes из external system периодически.

Java admin client предоставляет typed API вместо raw HTTP:
```java
Keycloak kc = KeycloakBuilder.builder()
    .serverUrl("https://keycloak.isna")
    .realm("master")
    .grantType(OAuth2Constants.CLIENT_CREDENTIALS)
    .clientId("admin-cli")
    .clientSecret("...")
    .build();

List<UserRepresentation> users = kc.realm("knp").users().search("berik");
```

## User Federation

Keycloak может интегрироваться с внешними user stores чтобы не дублировать данные пользователей. Federation полезна для организаций где existing directory (LDAP, AD) является source of truth для user identities.

LDAP и Active Directory поддерживаются напрямую через встроенный User Storage Provider. Настройка включает LDAP URL, bind DN и credentials, base DN для поиска пользователей, mappings атрибутов между LDAP и Keycloak model. Пользователи из LDAP автоматически появляются в Keycloak при первом успешном login, credentials проверяются через LDAP bind, atributes синхронизируются из LDAP.

Kerberos поддерживается через SPNEGO integration для seamless SSO в корпоративных environments где пользователи уже authenticated через domain login. SAML и OIDC federation позволяют использовать другие Identity Providers — Google, GitHub, Microsoft Azure AD как upstream аутентификация. Custom User Storage Providers через SPI позволяют интеграцию с proprietary user stores.

## Authentication Flows

Authentication flow в Keycloak это настраиваемая последовательность шагов при login. Стандартные встроенные flows покрывают типичные сценарии — Browser flow для web login, Direct Grant flow для password grant, Reset Credentials для password recovery, Registration для sign-up.

Каждый flow состоит из последовательных executions каждый выполняющий specific проверку. Cookie execution проверяет existing SSO session. Identity provider redirect позволяет federated login. Forms представляет username/password form. OTP execution требует one-time password. WebAuthn поддерживает passwordless authentication через platform authenticators.

Каждый execution имеет requirement — REQUIRED (обязательное успешное завершение), ALTERNATIVE (одно из группы должно пройти), DISABLED (пропустить), CONDITIONAL (выполнить условно).

Custom flows позволяют комбинировать executions создавая специфические scenarios. Например flow «password обязательно, потом OTP обязательно» реализует two-factor authentication. Flow «либо ЭЦП либо password + OTP» даёт выбор пользователю с fallback.

Custom Authenticators расширяют возможности — Java class реализующий Authenticator interface, deployed как JAR в providers directory. Позволяет arbitrary логику включая integration с внешними системами вроде ЭЦП верификации в КНП.

## Themes

Themes управляют визуальным представлением user-facing страниц Keycloak — login page, registration, password reset, error pages. Также email templates для системных email включая password reset, email verification, account confirmation.

Structure themes использует FreeMarker templates для HTML, отдельные CSS для styling, JavaScript для интерактивности. Themes организованы в директории со specific структурой которую Keycloak scans при старте.

В КНП обычно custom themes под corporate branding — цвета, лого, welcome текст, formatted email templates. Custom themes упаковываются как JAR и deployment в Keycloak themes directory.

## Adapters deprecated

Раньше Keycloak предоставлял client-side adapters для интеграции с приложениями — keycloak-spring-boot-starter, keycloak-spring-security-adapter, keycloak-tomcat-adapter и другие. Adapters делали всё автоматически — интеграция с Spring Security, парсинг токенов, извлечение ролей.

С 2022 adapters помечены deprecated. Рекомендуемый подход — стандартный Spring Security OAuth2 Resource Server работающий с любым OIDC-compliant provider. Причины deprecation включают дублирование функциональности со стандартами, сложность поддержки для новых версий Spring, привязка приложения к Keycloak-специфичному API вместо стандартов.

Плюсы миграции на стандартный Spring Security подход. Portability — приложение работает с любым OIDC provider не только Keycloak. Проще обновлять Spring Boot версии — не нужно ждать пока Keycloak выпустит совместимый adapter. Better integration с остальной Spring экосистемой — standard patterns и practices.

Минусы миграции — Keycloak-specific структура ролей в realm_access и resource_access не понимается Spring по default. Требуется custom JwtAuthenticationConverter преобразующий эти claims в GrantedAuthorities. Некоторая работа для миграции но подробно описано в файле 27.

В КНП постепенный переход или уже переход на Spring Security OAuth2 Resource Server от Keycloak adapters. Отдельные модули могут ещё использовать adapters но новые модули и мигрирующие делают через стандартный подход.

## Настройка realm пример

Полный setup нового realm в Keycloak включает несколько шагов. Создание realm через Admin console — Add realm, указать name (например knp), enabled true.

Создание клиентов для разных use cases. Bearer-only client для backend API — Client ID isna-knp-integration, protocol openid-connect, access type bearer-only, standard flow off, direct access grants off (только валидация tokens). Public client для UI SPA — Client ID isna-knp-front, access type public, valid redirect URIs со wildcard для https://knp.kgd.gov.kz/*, web origins + для CORS, standard flow on, PKCE required. Confidential client для M2M — Client ID isna-knp-fno-service, access type confidential, service accounts enabled, standard flow off, direct access grants off.

Создание ролей. Realm roles общие — admin, user, taxpayer. Client roles на isna-knp-integration специфичные для операций — CREATE_FNO, VIEW_FNO, DELETE_FNO, KNP_PERM_CREATE_NZ_N07 для конкретных permissions.

Создание пользователя. Username berik, email, first name, last name. Set password с temporary flag чтобы пользователь сменил при первом login. Role mapping — assign taxpayer realm role, CREATE_FNO client role на isna-knp-integration.

Настройка mappers. Default mappers для realm roles и client roles уже присутствуют — realm_access.roles и resource_access.<client>.roles. Custom mapper для group membership если groups используются. Custom attribute mappers если применимы — tenant_id, department, любые бизнес-specific атрибуты.

Настройка token settings — access token lifespan 5 минут, refresh token lifespan 30 минут, SSO session idle timeout 30 минут, SSO session max lifespan 10 часов. Настройка client scopes для переиспользуемых конфигураций между клиентами.

## HA и мониторинг

Keycloak stateful приложение — держит sessions в memory. Persistent данные (users, roles, clients) хранятся в external database обычно PostgreSQL. Sessions cache используется Infinispan.

Standalone deployment для development и небольших installations — один Keycloak instance с embedded H2 database. Простой но не подходит для production.

Cluster deployment для production — несколько Keycloak nodes, external database (PostgreSQL), Infinispan для sessions с replication между nodes. Обычно 3 или больше nodes для HA. Load balancer перед nodes для распределения трафика. Sticky sessions предпочтительны хотя не обязательны с полной sessions replication.

Мониторинг критически важен потому что Keycloak central IAM — его failure ломает authentication во всех связанных приложениях. Metrics endpoint /metrics в Prometheus формате экспортирует все ключевые показатели — request rates, latencies, error rates, session counts, cache hit rates.

Events log audit trail всех security-relevant событий. Login success, login failure, logout, token refresh, admin actions — все логируются с deltas of context. Storage events возможно в database или external system. Alerting на suspicious patterns — многочисленные failed login attempts, admin actions outside business hours.

Admin events отдельно логируют changes конфигурации — user create/delete, role assignments, client updates, realm configuration changes. Critical для audit compliance и security investigations.

## Итоги

Keycloak это open source IAM платформа предоставляющая OAuth2/OIDC provider, user management, federation, admin capabilities. В КНП играет роль центрального IAM для всех микросервисов.

Иерархия сущностей — Realm как контейнер изоляции, внутри users, groups, roles (realm и client), clients, mappers, scopes. Клиенты трёх типов — public для SPA/mobile, confidential для backend, bearer-only для чистых resource servers.

Mappers определяют как user data попадает в JWT claims. Keycloak specific структура ролей в realm_access и resource_access требует custom JwtAuthenticationConverter в Spring Security для правильного парсинга.

Стандартный набор OIDC endpoints per realm — discovery, authorization, token, userinfo, JWKS, logout, introspection, revocation. Discovery endpoint позволяет автоматическую конфигурацию клиентов через одну строку issuer-uri.

Authorization Code flow стандарт для UI приложений. Client Credentials для M2M. Refresh Token для обновления. Password grant deprecated. ЭЦП flow в КНП через custom Authenticator SPI.

Admin REST API для программного управления. User Federation для integration с external directories вроде LDAP. Authentication Flows customizable для complex scenarios. Themes для branding. Custom Authenticators через SPI для extension.

Adapters deprecated в пользу стандартного Spring Security OAuth2 Resource Server. Portability и integration benefits перевешивают need в custom JwtAuthenticationConverter.

HA deployment через cluster с external database и Infinispan sessions replication. Мониторинг через Prometheus metrics плюс events log критически важен потому что Keycloak central IAM.

Следующий файл — Spring Security плюс OAuth2 плюс Keycloak в production, конкретная реализация всех этих концепций в микросервисах.
