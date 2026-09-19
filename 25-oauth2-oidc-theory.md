# 25. OAuth2 и OIDC теория

## Что такое OAuth2

OAuth2 это открытый стандарт для делегированной авторизации. Позволяет одному приложению получить доступ к ресурсам пользователя на другом сервисе без раскрытия пароля. Классические примеры — «Войти через Google», «Разрешить приложению X доступ к вашим файлам Google Drive», «Использовать GitHub для аутентификации в CI/CD системе».

Ключевая философия OAuth2 — user никогда не отдаёт свой пароль стороннему приложению. Вместо этого пользователь аутентифицируется у trusted authorization server, который выдаёт ограниченный по правам и времени токен для стороннего приложения. Приложение использует этот токен для доступа к API от имени пользователя без knowledge оригинального пароля.

Важное различие — OAuth2 сам по себе не про authentication. OAuth2 отвечает на вопрос «что тебе можно», выдавая tokens с определёнными правами. Он не отвечает на вопрос «кто ты» стандартизированным способом. Отсюда родилось OIDC — OpenID Connect как расширение OAuth2 для authentication. Часто в разговоре OAuth2 и OIDC используются как синонимы, но технически OIDC это надстройка над OAuth2.

## Четыре роли

OAuth2 определяет четыре роли участников. Понимание этих ролей критически важно для правильного применения OAuth2 в реальных системах.

Resource Owner это пользователь владеющий защищёнными ресурсами. Обычно человек с учётной записью в системе. Может быть также service account для machine-to-machine сценариев, хотя классически resource owner всегда user.

Client это приложение желающее получить доступ к ресурсам owner. Может иметь разные формы. Web application с backend где client secret хранится на server. Single-page application (SPA) в браузере где нет secure storage для secret. Native application на mobile или desktop. Machine-to-machine service работающий без user interaction.

Authorization Server (AS) выдаёт токены после проверки пользователя и его consent. Управляет sessions пользователей, регистрирует clients, определяет какие scopes может запросить каждый client. Примеры реальных AS — Keycloak как open source solution обычно self-hosted, Okta и Auth0 как SaaS решения, Google, GitHub, Microsoft как identity providers для social login.

Resource Server (RS) это API с защищёнными данными. Не занимается authentication пользователя — доверяет tokens выданным AS. При каждом запросе проверяет token — подпись, expiration, audience, scopes. Пример — Google Fit API это RS выдающий данные fitness, Auth Server Google выдаёт tokens к нему.

Практический пример иллюстрирует роли. Берик хочет использовать fitness tracker который анализирует данные из Google Fit. Берик это Resource Owner. Fitness tracker это Client. Google OAuth2 это Authorization Server. Google Fit API это Resource Server. Fitness tracker получает access token от Google OAuth2 после Berik's consent, использует токен для запросов к Google Fit API, обрабатывает полученные данные.

## Authorization Code Flow

Authorization Code это основной flow OAuth2 для web приложений с backend. Более безопасный чем предшественники потому что access token никогда не проходит через browser напрямую, только authorization code который сам по себе бесполезен без client secret.

```
Пользователь          Client              Auth Server         Resource Server
    │                   │                      │                     │
    │  клик "Login"     │                      │                     │
    ├──────────────────►│                      │                     │
    │                   │                      │                     │
    │                   │  Redirect на AS      │                     │
    │                   │  ?response_type=code │                     │
    │                   │  &client_id=xxx      │                     │
    │                   │  &redirect_uri=...   │                     │
    │                   │  &state=random       │                     │
    │◄──────────────────┤                      │                     │
    │                                          │                     │
    │  Пользователь логинится + consent        │                     │
    │  (пароль вводится ТОЛЬКО на AS)          │                     │
    ├─────────────────────────────────────────►│                     │
    │                                          │                     │
    │  Redirect обратно на Client с authcode   │                     │
    │  ?code=SplxlOBeZQQYbYS6WxSbIA            │                     │
    │  &state=random                           │                     │
    │◄─────────────────────────────────────────┤                     │
    │                   │                      │                     │
    ├──────────────────►│  authcode            │                     │
    │                   │                      │                     │
    │                   │  Обмен code +        │                     │
    │                   │  client_secret       │                     │
    │                   │  на tokens           │                     │
    │                   ├─────────────────────►│                     │
    │                   │                      │                     │
    │                   │  access_token,       │                     │
    │                   │  refresh_token,      │                     │
    │                   │  id_token (OIDC)     │                     │
    │                   │◄─────────────────────┤                     │
    │                   │                      │                     │
    │                   │  API request с access_token                │
    │                   │  Authorization: Bearer <token>             │
    │                   ├───────────────────────────────────────────►│
    │                   │                      │                     │
    │                   │                      │  Проверка токена   │
    │                   │                      │  (JWKS / introspect)│
    │                   │                      │◄────────────────────┤
    │                   │                      │                     │
    │                   │  Data response                             │
    │                   │◄───────────────────────────────────────────┤
```

Ключевые моменты этого flow. Client никогда не видит пароль пользователя — пользователь вводит credentials только на AS. AS выдаёт authorization code через redirect обратно на client. Client обменивает code на tokens используя свой client_secret — прямой server-to-server call с backend клиента. Access token используется для API вызовов на Resource Server.

Двойная защита — код + secret. Даже если атакующий перехватит authorization code (redirect идёт через browser, теоретически перехватываемо), он не сможет обменять его на tokens без client_secret. Client_secret хранится на server никогда не проходит через browser. Refresh token также остаётся на server — не отправляется client browser.

State параметр защищает от CSRF на OAuth flow. Client генерирует random state перед redirect на AS, сохраняет в session, проверяет при callback. Если state в callback не совпадает с ожидаемым — атака, отклонить запрос. Без state параметра атакующий может завершить OAuth flow от чужого имени.

## Authorization Code + PKCE

PKCE расшифровывается как Proof Key for Code Exchange. Расширение authorization code flow для клиентов которые не могут безопасно хранить client_secret. SPA в браузере, mobile apps — не имеют secure storage что означает secret был бы viewable для reverse engineering.

Механизм PKCE основан на дополнительной проверке при обмене code на tokens. Client генерирует случайный code_verifier — cryptographically random string 43-128 символов. Вычисляет code_challenge как base64url(SHA256(code_verifier)). При инициации authorize request отправляет только code_challenge — оригинальный verifier остаётся у client.

AS запоминает code_challenge связанным с authorization code. При обмене code на tokens client отправляет оригинальный code_verifier. AS вычисляет hash от verifier, сравнивает с сохранённым challenge. Совпадает — выдаёт tokens. Не совпадает — отклоняет запрос.

Гарантия PKCE — даже если атакующий перехватит authorization code (например через malicious app на устройстве получившим доступ к redirect URI), обмен на tokens не сможет выполнить потому что не знает code_verifier. Verifier никогда не проходит через network пока не пришло время обмена, к этому моменту атакующий не имеет доступа к сгенерированному client значению.

С 2024 года рекомендации сместились — PKCE рекомендуется даже для web app с client secret. Дополнительная защита имеет минимальные overhead и защищает от определённых attack vectors. Все современные OAuth2 clients должны использовать PKCE независимо от типа.

## Client Credentials Flow

Client Credentials это flow для machine-to-machine коммуникации. Нет пользователя как resource owner — один сервис вызывает другой от своего имени. Используется для backend integrations, scheduled jobs, cron tasks, service mesh internal calls.

Client делает POST запрос на token endpoint с grant_type client_credentials, client_id и client_secret как credentials. AS проверяет credentials и возвращает access_token без refresh_token и без user info. Токен содержит client_id как subject и permissions предоставленные этому client.

Простой flow подходит для straightforward сценариев. Batch job публикует уведомления — использует client credentials получая token с scope send_notifications, вызывает notification API. Analytics service забирает данные из main service — client credentials с read_analytics scope. Backup service синхронизирует данные — client credentials с data_export scope.

Refresh token обычно не выдаётся для client credentials потому что client всегда может получить новый access token используя те же credentials. Простое обновление без стороннего state management.

Регистрация client в AS для client credentials требует определения scopes которые client может запрашивать. Обычно принцип минимальных привилегий — каждый client получает только необходимые для его функции scopes. Изменение scopes требует изменения регистрации в AS.

## Refresh Token Flow

Access tokens короткоживущие обычно 5-60 минут для security. Refresh token долгоживущий обычно дни или недели. Когда access token истёк client использует refresh token для получения нового без re-authentication пользователя.

Механизм — client отправляет POST на token endpoint с grant_type refresh_token и refresh_token value. AS проверяет что refresh token валиден и не revoked. Возвращает новый access_token и опционально rotated refresh_token.

Refresh token rotation это security best practice. При каждом использовании refresh token AS выдаёт новый refresh token invalidating старый. Это позволяет обнаружить кражу — если атакующий использовал refresh token, при следующем legitimate использовании старый уже invalid, что triggers session termination и alert. Не все AS поддерживают rotation, но современные implementations всё чаще используют её.

Client должен securely хранить refresh token. HttpOnly cookie для web app скрытый от JavaScript снижает impact XSS. Secure keychain для mobile защищает от других приложений. Encrypted local storage с careful implementation для desktop apps. Никогда plain localStorage в web браузере где XSS может прочитать.

Interceptor pattern часто реализует automatic refresh. При получении 401 от Resource Server client интерцептор пытается refresh, если удачно — retry исходный запрос с новым access token, если refresh упал — redirect на login. Прозрачно для application code.

## Password Grant deprecated

Password grant позволял client получить token напрямую отправляя username/password пользователя на AS. Client сам собирает credentials — пользователь вводит их в интерфейс client, client отправляет на AS.

Проблема очевидна — client видит и потенциально сохраняет пароль. Противоречит фундаментальной философии OAuth2 где пользователь никогда не должен доверять credentials стороннему client. Оставлено только для legacy migrations где нужно временно поддержать старые интеграции.

Не использовать в новых системах. Для API access использовать client credentials. Для user authentication использовать authorization code с PKCE. Password grant deprecated в OAuth 2.1 draft.

## Implicit Grant deprecated

Implicit grant предназначался для SPA где не было client secret. AS возвращал access token прямо в URL fragment после authentication. Client (JavaScript в браузере) читал token из URL и использовал для API запросов.

Проблемы включают токен в URL что риск утечки через logs, browser history, referrer headers. Отсутствие refresh token (иначе он бы также попал в URL) означает что при истечении access token нужен re-authentication. Vulnerability к token substitution attacks определённых сценариях.

Заменён на Authorization Code + PKCE который решает все эти проблемы. Implicit grant removed из OAuth 2.1. Не использовать в новых системах.

## Device Code Flow

Device Code это специальный flow для устройств без клавиатуры или ограниченного user input. Smart TV, IoT devices, CLI tools. Устройство показывает short code на своём экране, пользователь идёт на browser (обычно на телефоне) вводит код на special URL AS, аутентифицируется там.

Механизм — устройство запрашивает device_code и user_code у AS. Показывает user_code и URL на экране. Пользователь заходит по URL на другом устройстве, вводит user_code, аутентифицируется. Устройство параллельно polls AS с device_code. Когда пользователь завершил authentication на browser, AS отвечает с tokens на очередной poll устройства.

Использование — Android TV apps авторизующиеся в streaming service, CLI tools авторизующиеся в cloud provider, IoT devices регистрирующиеся в management platform. Специфический но важный flow для правильных use cases.

## Access Token

Access token это credential presented к Resource Server для доступа к защищённым ресурсам. Прикладывается к каждому API запросу через Authorization header с Bearer prefix.

Свойства access token включают короткий срок жизни обычно 5-60 минут. Присутствие в каждом request потенциально проходит через много logs и infrastructure что делает short lifetime security best practice. Содержит authorities/scopes определяющие что client может делать.

Формат может быть JWT или opaque. JWT это self-contained token где всё нужное для validation находится внутри токена. Resource Server проверяет подпись и claims локально без обращения к AS. Быстро — no round trip к AS для каждого запроса.

Opaque token это случайная строка не содержащая meaningful information. Resource Server должен обратиться к AS для валидации через introspection endpoint. Медленнее — network round trip к AS для каждого запроса. Плюс — легко revoke, AS может немедленно отклонить token без ожидания expiration.

Выбор JWT vs opaque зависит от требований. JWT предпочтителен для performance при большом количестве запросов. Opaque предпочтителен когда revocation критически важен, latency less critical.

## Refresh Token

Refresh token используется только для получения новых access tokens. Долгоживущий обычно от нескольких дней до месяцев или пока не revoked. Никогда не отправляется к Resource Server — только на token endpoint AS.

Хранение refresh token критически важно. HttpOnly cookie для web app защищает от XSS. Secure storage платформы для mobile — Keychain на iOS, EncryptedSharedPreferences на Android. Encrypted при at rest для desktop apps.

Rotation refresh token на каждое использование security best practice. Обнаруживает кражу — использованный украденный refresh token invalidates legitimate token, при следующем использовании обнаруживается несоответствие.

Revocation refresh token останавливает further использование. При logout или compromise — call revocation endpoint AS. Все связанные access tokens должны быть invalidated если возможно.

## ID Token OIDC

ID token это концепт из OpenID Connect не OAuth2 proper. Содержит информацию о пользователе — sub identifier, name, email, custom claims. Всегда в формате JWT для стандартизированного парсинга.

Использование только для authentication — узнать «кто пользователь». Не должен быть отправлен к Resource Server — resource servers должны использовать access token, не id token. Common ошибка — отправлять id token как Bearer token к API, это неправильное использование.

Client валидирует id token при получении. Проверяет подпись через JWKS из AS. Проверяет expiration, issuer, audience. Extracts user claims для отображения в UI или сохранения в session.

## JWT формат

JSON Web Token это открытый стандарт для self-contained tokens. Формат — три части через точки: header, payload, signature. Каждая часть base64url encoded.

Пример JWT:
```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImtleS0xIn0.
eyJzdWIiOiJiZXJpayIsImlzcyI6Imh0dHBzOi8va2V5Y2xvYWsuaXNuYSIsImV4cCI6MTcwMDAwMDAwMH0.
aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890...
```

Header содержит metadata о токене. Поля alg указывает алгоритм подписи (RS256, ES256, HS256), typ обычно JWT, kid identifier ключа для проверки подписи (используется когда AS имеет несколько ключей для rotation).

Payload содержит claims — утверждения о subject токена. Стандартные claims определены в RFC. iss (issuer) кто выдал токен. sub (subject) идентификатор пользователя или клиента. aud (audience) для кого предназначен токен. exp (expiration) unix timestamp истечения. iat (issued at) когда выдан. nbf (not before) с какого момента действителен. jti (JWT ID) уникальный идентификатор для revocation.

Custom claims могут быть добавлены — roles, permissions, tenant_id, email, name, любые данные необходимые приложению. Размер токена растёт с количеством claims, что важно учитывать поскольку токен передаётся с каждым запросом.

Signature подписывает header и payload криптографически. RS256 использует RSA private key для подписи, public key для проверки. HS256 использует symmetric HMAC с shared secret. RS256 предпочтителен для распределённых систем — public key может быть shared со всеми resource servers без риска подделки, private key остаётся только на AS.

Важное свойство JWT — payload читаем всеми через base64 decode. Не encryption. Signature гарантирует authenticity — токен не был подменён. Кто угодно может прочитать claims. Не помещать sensitive data в JWT без дополнительного encryption через JWE.

JWS vs JWE. JWS (JSON Web Signature) подпись без шифрования, payload читаем. JWE (JSON Web Encryption) шифрованный payload доступный только получателю с private key. 99 процентов практических случаев JWS — signature достаточна для authentication, encryption обычно излишен.

Проверка JWT в Resource Server состоит из нескольких шагов. Разобрать токен на header, payload, signature. Проверить alg — критически не принимать alg none, атака где attacker подписывает JWT с alg none и signature пустая. Найти public key по kid из JWKS endpoint AS. Проверить signature с public key и header плюс payload. Проверить exp — токен не истёк. Проверить nbf — токен уже действителен. Проверить iss — issuer соответствует ожидаемому. Проверить aud — audience содержит идентификатор этого resource server. Extract claims в GrantedAuthority для использования Spring Security. Spring делает это через JwtDecoder и NimbusJwtDecoder.

## OpenID Connect

OIDC это расширение OAuth2 для authentication. OAuth2 говорит про делегированный доступ к ресурсам — authorization. OIDC добавляет стандартизированный способ узнать кто пользователь — authentication.

Ключевые добавления OIDC. ID token как JWT с standardized claims о пользователе. UserInfo endpoint как REST для запроса user info у AS. Standard scopes включая openid обязательный для получения id_token, profile для basic user info, email для email adress, address для physical address, phone для phone number. Discovery document на well-known URL описывающий возможности AS.

Discovery endpoint это стандартный URL /.well-known/openid-configuration возвращающий JSON описание AS. Содержит все URLs — authorization_endpoint, token_endpoint, userinfo_endpoint, jwks_uri, revocation_endpoint, end_session_endpoint. Также supported grant_types, scopes, response_types, algorithms.

Spring Security читает discovery автоматически при настройке через issuer-uri. Одна строка конфигурации, Spring делает all rest — получает discovery document, определяет URLs, настраивает JWT decoder с правильным JWKS URI.

Keycloak, Okta, Auth0, Google, Microsoft — все являются OIDC-compliant providers. Стандартизация означает что client написанный по OIDC работает со всеми ними без специфичной для каждого логики.

## Scopes

Scopes ограничивают права выданного токена. Client запрашивает конкретные scopes при authorize request, пользователь одобряет consent screen (или скрыто когда consent предварительно разрешён), AS выдаёт токен только с одобренными scopes.

Стандартные OIDC scopes включают openid обязательный для получения id_token, profile для basic user attributes, email для email, offline_access для получения refresh_token.

Custom scopes определяются API. read:orders для чтения заказов, write:orders для создания и изменения. Convention — глагол:ресурс, но конкретный формат определяется системой. Некоторые системы используют более granular scopes.

Client Credentials обычно использует service scopes отличные от user scopes. service:notifications, service:analytics, service:audit — идентифицируют функциональность backend integration.

Resource Server проверяет scopes при обработке запроса. Endpoint для чтения заказов проверяет что токен содержит read:orders scope. Endpoint для создания — write:orders. Insufficient scope возвращает 403 Forbidden.

## JSON Web Key Set

JWKS это стандартный формат для публичных ключей AS используемых при проверке подписи JWT. Standard endpoint /realms/{realm}/protocol/openid-connect/certs (для Keycloak) или jwks_uri из discovery document возвращает JSON с ключами.

Формат — array объектов keys каждый описывающий один ключ. Поля включают kid идентификатор ключа matched с kid в JWT header, kty тип ключа обычно RSA, alg алгоритм обычно RS256, n modulus и e exponent для RSA public key.

Resource Server кэширует JWKS для эффективности. При первом запросе fetches и кэширует. При появлении JWT с новым kid не в кэше — refreshes JWKS. Периодический refresh даже без новых kid для обновления при rotation.

Key rotation важная security practice. AS периодически меняет ключи. Новые токены подписываются новым ключом с новым kid. Старые токены остаются валидными пока не истекут — старые ключи оставляются в JWKS до истечения последнего токена подписанного ими. Смена ключа происходит без прерывания сервиса.

## Token Introspection

Introspection endpoint позволяет Resource Server запросить у AS статус opaque access token. POST на introspect endpoint с token value и client credentials RS. AS отвечает JSON с active true или false, если active — plus claims like sub, exp, scope.

Использование для opaque tokens где нельзя валидировать локально. Или для scenarios где нужно verify актуальный статус токена — не был ли revoked. JWT токены обычно не требуют introspection если только не нужна immediate revocation, поскольку могут быть валидированы локально.

Trade-off — introspection даёт immediate revocation но требует network round trip к AS для каждого API запроса. JWT избегает round trip но revocation сложнее. Выбор зависит от требований производительности vs revocation.

## Refresh Token Flow детально

```
Client                                       Auth Server
  │                                                │
  │  API request с access_token                    │
  │                                                │
  │  ◄─── 401 Unauthorized                         │
  │       (access_token expired)                   │
  │                                                │
  │  POST /token                                   │
  │  grant_type=refresh_token                      │
  │  refresh_token=xxx                             │
  │  client_id=my-app                              │
  │  (client_secret если confidential client)     │
  ├───────────────────────────────────────────────►│
  │                                                │
  │  Проверка refresh_token                        │
  │  (не revoked, не expired)                      │
  │                                                │
  │  Response:                                     │
  │  {                                             │
  │    "access_token": "new-jwt-...",              │
  │    "refresh_token": "new-refresh-..."          │
  │       (если rotation)                          │
  │    "expires_in": 3600                          │
  │  }                                             │
  │◄───────────────────────────────────────────────┤
  │                                                │
  │  Retry API request с новым access_token        │
  │  (прозрачно для user)                          │
```

Client обычно имеет interceptor реализующий этот flow автоматически. При 401 response от Resource Server — попытка refresh. Успех — retry исходного запроса. Failure refresh — redirect пользователя на login.

## Logout

Logout в OAuth2/OIDC не так прост как в session-based auth. У пользователя есть tokens распределённые по разным клиентам, session на AS может использоваться multiple clients через SSO. Правильный logout должен terminate session у AS и invalidate all associated tokens.

RP-initiated logout это OIDC extension. Client отправляет пользователя на end_session_endpoint AS с параметрами включая id_token_hint. AS завершает user session, invalidates tokens, редиректит обратно на client с confirmation.

Backchannel logout это более сложный механизм. AS шлёт notification всем зарегистрированным clients когда user session завершена. Каждый client должен terminate свою локальную session. Не все AS поддерживают, не все clients реализуют. Сложность реализации compared with practical benefit.

Client-side logout простейший подход. Просто удалить tokens на client — clear cookies, clear storage, redirect на login. Работает для sessions конкретного client, не решает cross-client SSO logout. Часто достаточно для simple scenarios.

Access token остаётся valid до его natural expiration после logout. Real revocation требует либо short-lived access tokens (обычно правильный подход) либо blacklist по jti или user_id (сложнее).

## Common pitfalls

Ряд ошибок регулярно проявляется при работе с OAuth2 и OIDC. Понимание этих pitfalls помогает избегать их в реальных проектах.

Token в URL это серьёзная ошибка. GET /api?token=xxx попадает в browser history, server logs, proxy logs, referrer headers. Токен эффективно compromised поскольку viewable многим системам. Всегда использовать Authorization header с Bearer scheme.

Долгий access token создаёт revocation window. Access token валиден 1 час означает что revocation не может быть applied в течение часа — уволенный сотрудник, compromised device, известный leak токена. Держать короткими — 5-15 минут, использовать refresh для получения новых. Compromise окно ограничено short период.

Неправильное хранение токенов на client. Web app стандарт — HttpOnly cookie недоступный для JavaScript, secure flag для HTTPS-only transmission. SPA — sessionStorage или memory storage acceptable но vulnerable к XSS. localStorage категорически не подходит — легко доступен XSS payload. Mobile — secure keychain платформы. Consistent secure storage критически важно.

Не проверять alg приводит к serious vulnerability. Атакующий может подписать JWT с alg none и empty signature. Some libraries принимают такой токен как valid. Всегда явно specify allowed algorithms при JWT validation. Reject unknown alg.

Не проверять iss и aud открывает confused deputy attack. Токен от одного AS может пройти validation на service которое не ожидает от него токенов. Или токен для одного audience используется для другого. Always check iss matches expected AS, aud contains identifier of this resource server.

CSRF на OAuth flow через отсутствующий state параметр. Attacker может initiate OAuth flow, получить authorization code, use victim's session для завершения flow linking его account с victim's identity. State параметр — random значение сохранённое в session client, проверяемый при callback. Без state — vulnerability. PKCE тоже помогает но state рекомендуется даже с PKCE.

Confidential vs public client неправильно классифицированы. Confidential client имеет client_secret безопасно хранимый на server. Public client не может секретно хранить secret — SPA, mobile app. Использование confidential flow для public client компрометирует secret. Всегда использовать public client type для SPA и mobile с PKCE.

Consent management часто игнорируется. Consent screen пользователю какие permissions запрашивает client. Не все AS показывают consent для internal apps. Правильное применение защищает пользователей от authorization без их knowledge.

## Итоги

OAuth2 предоставляет стандарт делегированной авторизации. Четыре роли — Resource Owner, Client, Authorization Server, Resource Server — образуют базовую architecture. OIDC как надстройка над OAuth2 добавляет стандартизированную authentication.

Authorization Code с PKCE это стандарт для UI приложений включая web apps, SPA, mobile. Client Credentials для machine-to-machine backend integrations. Refresh Token flow для обновления expired access tokens без re-authentication.

Deprecated grant types — Password grant и Implicit — не должны использоваться в новых системах. Password grant противоречит философии OAuth2. Implicit заменён на Authorization Code с PKCE.

Access tokens короткоживущие для security, refresh tokens долгоживущие для UX. ID tokens отдельно для authentication information в OIDC.

JWT это standard формат для self-contained tokens. Три части header.payload.signature, base64url encoded, подпись RSA или HMAC. Стандартные claims iss, sub, aud, exp, iat дают core semantics. Custom claims для application-specific data. Signature validation через JWKS public keys.

OIDC добавляет id_token, userinfo endpoint, discovery document, стандартные scopes. Позволяет client написанный по стандарту работать с любым OIDC provider.

Scopes ограничивают права токена. Standard OIDC scopes plus custom application scopes. Resource server проверяет наличие требуемых scopes на каждый защищённый endpoint.

Introspection для opaque tokens и immediate revocation. JWT для performance при большом количестве запросов. Choice зависит от specific requirements.

Logout complex в OAuth2. RP-initiated logout стандартный OIDC подход. Backchannel logout для sophisticated scenarios. Client-side logout для simple cases.

Common pitfalls — token в URL, long access token TTL, insecure client storage, alg none accepted, missing iss/aud check, missing state parameter. Все имеют предсказуемые причины и standard mitigations.

Правильное применение OAuth2 и OIDC требует attention к details — все standard mechanisms существуют для конкретных security threats, каждый пропущенный элемент создаёт vulnerability. С другой стороны consistent следование стандартам делает систему секьюрной без сложных custom mechanisms.

Дальше — Keycloak как конкретная реализация OIDC provider используемая в КНП и многих других enterprise системах.
