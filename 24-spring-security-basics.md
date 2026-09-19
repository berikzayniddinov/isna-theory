# 24. Spring Security основы

## Что делает Spring Security

Spring Security решает четыре фундаментальные задачи защиты web приложения. Authentication отвечает на вопрос «кто ты?» — проверка credentials пользователя через пароль, токен, сертификат или другой механизм. Authorization отвечает на вопрос «что тебе можно?» — определение прав пользователя на конкретные ресурсы и операции через роли, permissions, policies. Attack protection защищает от известных классов атак — CSRF, XSS, session fixation, brute force, clickjacking. Session management контролирует создание сессий, timeout, invalidation после logout.

Практические вопросы на которые Security отвечает в реальном приложении. Может ли пользователь berik вызвать POST /api/fno/submit? Валиден ли предоставленный JWT токен? Правильный ли пароль указан при логине? Есть ли у пользователя роль ADMIN необходимая для admin endpoint? На каждый входящий запрос Security выполняет цепочку проверок отвечающих на эти и подобные вопросы прежде чем разрешить или запретить операцию.

## Основа — цепочка фильтров

Архитектурная база Spring Security это цепочка Servlet-фильтров. Каждый входящий HTTP запрос проходит через последовательность фильтров до того как попадёт в DispatcherServlet и контроллер приложения. Каждый фильтр имеет специфическую роль в security обработке.

```
HTTP request
    │
    ▼
┌───────────────────────────────────────────────────┐
│ FilterChainProxy (главный оркестратор)             │
│                                                    │
│ ├─ SecurityContextPersistenceFilter               │
│ │  восстанавливает SecurityContext из session      │
│ │                                                  │
│ ├─ CsrfFilter                                     │
│ │  проверка CSRF token для state-changing         │
│ │                                                  │
│ ├─ LogoutFilter                                   │
│ │  обрабатывает /logout                           │
│ │                                                  │
│ ├─ UsernamePasswordAuthenticationFilter           │
│ │  form login POST /login                         │
│ │                                                  │
│ ├─ BasicAuthenticationFilter                      │
│ │  HTTP Basic Auth header                         │
│ │                                                  │
│ ├─ BearerTokenAuthenticationFilter                │
│ │  Bearer JWT (OAuth2)                            │
│ │                                                  │
│ ├─ RequestCacheAwareFilter                        │
│ ├─ SecurityContextHolderAwareFilter               │
│ │                                                  │
│ ├─ AnonymousAuthenticationFilter                  │
│ │  гость если нет других auth                     │
│ │                                                  │
│ ├─ SessionManagementFilter                        │
│ │  session creation policy, fixation protection   │
│ │                                                  │
│ ├─ ExceptionTranslationFilter                     │
│ │  ловит AccessDenied → 403, Unauthorized → login │
│ │                                                  │
│ └─ AuthorizationFilter                            │
│    финальная проверка авторизации                 │
└───────────────────────────────────────────────────┘
    │
    ▼
DispatcherServlet → @RestController
```

Порядок фильтров важен потому что каждый предполагает определённое состояние подготовленное предыдущими. SecurityContextPersistenceFilter первым потому что все последующие полагаются на SecurityContext. AuthorizationFilter в конце потому что должен работать с уже установленной authentication. Нарушение порядка приводит к неправильному поведению системы безопасности.

SecurityFilterChain это bean декларация цепочки в Spring Security 6. Заменяет старый WebSecurityConfigurerAdapter из более ранних версий. Декларируется через @Bean возвращающий SecurityFilterChain построенный из HttpSecurity builder. В builder указываются authorization rules через authorizeHttpRequests, session policy через sessionManagement, disabled features вроде CSRF для REST API, метод аутентификации вроде oauth2ResourceServer для JWT.

Возможно несколько SecurityFilterChain для разных частей приложения. @Order определяет какой применяется первым для конкретного запроса. Полезно для комплексных приложений где public endpoints, authenticated API, admin panel требуют разной конфигурации.

## Ключевые концепты аутентификации

SecurityContext это thread-local хранилище содержащее информацию о текущем аутентифицированном пользователе. SecurityContextHolder предоставляет статический доступ к контексту из любой точки кода. Метод getContext().getAuthentication() возвращает объект Authentication с деталями пользователя.

Thread-local природа означает что контекст привязан к thread обрабатывающему текущий запрос. При переключении thread — @Async, virtual threads, executor — контекст по умолчанию не пробрасывается. Это критическое ограничение о котором нужно помнить, специальные механизмы существуют для передачи контекста между threads когда это необходимо.

Authentication это интерфейс представляющий аутентифицированного пользователя. Метод getName возвращает username или subject identifier. getPrincipal возвращает сам объект пользователя, обычно UserDetails для локальной auth или Jwt для OAuth2. getCredentials возвращает credentials обычно пароль хотя после успешной аутентификации часто nullify для безопасности. getAuthorities возвращает коллекцию GrantedAuthority — ролей и permissions пользователя. isAuthenticated возвращает true когда пользователь успешно аутентифицирован.

GrantedAuthority это интерфейс представляющий одну единицу прав пользователя. Единственный метод getAuthority возвращает строку идентифицирующую right. По конвенции роли префиксятся ROLE_ (например ROLE_ADMIN), permissions без префикса (KNP_PERM_CREATE_NZ_N07). Это соглашение отражается в API методах — hasRole автоматически добавляет префикс ROLE_ при проверке, hasAuthority работает с полным значением без префикса.

UserDetails стандартный интерфейс представляющий пользователя. Включает getUsername и getPassword для credentials, getAuthorities для прав, boolean методы для состояния аккаунта — isAccountNonExpired, isAccountNonLocked, isCredentialsNonExpired, isEnabled. Все эти состояния проверяются при аутентификации, отрицательное значение любого блокирует вход.

UserDetailsService провайдит UserDetails для конкретного username. Единственный метод loadUserByUsername принимает username и возвращает UserDetails либо бросает UsernameNotFoundException. Реализация обычно загружает пользователя из базы данных, конвертирует entity в UserDetails через builder. Spring использует UserDetailsService в DaoAuthenticationProvider для password-based аутентификации.

## PasswordEncoder и хеширование

PasswordEncoder отвечает за преобразование plaintext паролей в хешированный формат для безопасного хранения. Пароли никогда не должны храниться в plaintext — это фундаментальное требование безопасности. BCrypt, Argon2, PBKDF2 — рекомендуемые алгоритмы с adaptive cost, что означает возможность увеличения computational cost со временем чтобы противостоять улучшениям hardware attackers.

BCryptPasswordEncoder стандартный выбор для большинства приложений. При регистрации пользователя пароль передаётся через encode метод, результат сохраняется в базу. При проверке пароля matches метод сравнивает plaintext с stored hash, автоматически извлекая соль и cost factor из hash формата.

Argon2 более современный вариант выигравший password hashing competition. Более memory-hard что делает brute force attacks дороже. Argon2PasswordEncoder доступен как альтернатива BCrypt для новых проектов.

DelegatingPasswordEncoder поддерживает несколько форматов одновременно через prefix. Хеш начинается с идентификатора алгоритма — {bcrypt}, {argon2}, {pbkdf2}. Позволяет миграцию с одного алгоритма на другой без принудительного reset паролей всех пользователей. Дефолтный encoder в Spring Boot использует delegating подход что даёт flexibility.

Соль автоматически включается в hash в современных algorithms. Encoder генерирует случайную соль при encode, включает её в результирующий hash. При matches соль извлекается из stored hash для правильного сравнения. Разработчику не нужно думать о соли явно.

## AuthenticationManager и провайдеры

AuthenticationManager это интерфейс с одним методом authenticate принимающим Authentication token и возвращающим fully authenticated Authentication или бросающим AuthenticationException. Стандартная реализация ProviderManager делегирует authentication provider chain — последовательности AuthenticationProvider каждый умеющий проверить конкретный тип аутентификации.

DaoAuthenticationProvider проверяет username/password credentials через UserDetailsService. Наиболее классический вариант для form-based или Basic auth. Загружает UserDetails по username, проверяет пароль через PasswordEncoder, возвращает successful Authentication с authorities пользователя.

JwtAuthenticationProvider проверяет JWT токены. Декодирует токен, проверяет подпись через public key, валидирует claims включая expiration, извлекает authorities из claims (обычно из scope или custom claim), создаёт JwtAuthenticationToken.

OAuth2LoginAuthenticationProvider обрабатывает OAuth2 authorization code flow. При callback от authorization server извлекает code, обменивает на tokens, получает user info, создаёт OAuth2AuthenticationToken.

Custom providers реализуются через AuthenticationProvider интерфейс когда стандартных не хватает — например для custom SSO, certificate-based auth, LDAP integration. Регистрируются в AuthenticationManager для использования.

Provider chain обрабатывает Authentication последовательно. Первый провайдер поддерживающий тип Authentication (метод supports) пытается аутентифицировать. Успех — возврат аутентифицированного объекта. Неудача — возможно следующий provider попробует или общая ошибка. Позволяет комбинировать разные типы auth в одном приложении.

## Различные механизмы аутентификации

HTTP Basic это простой механизм — client отправляет Authorization header с base64 encoded username:password. Фильтр BasicAuthenticationFilter декодирует header, создаёт UsernamePasswordAuthenticationToken, передаёт AuthenticationManager. При success кладёт Authentication в SecurityContext.

Плюсы простота, встроенная поддержка в браузерах, работа без cookies. Минусы — credentials передаются с каждым запросом, требует HTTPS для безопасности, отсутствие механизма logout кроме закрытия браузера. Используется обычно для internal APIs, monitoring endpoints, quick prototypes.

Form login это классический механизм для UI приложений. Server показывает форму, client POST username/password на /login endpoint, UsernamePasswordAuthenticationFilter обрабатывает credentials, создаёт session cookie при success, redirect на исходный URL или default success page.

Плюсы user-friendly UI, стандартный подход для web приложений, session-based позволяет invalidation через logout. Минусы CSRF protection обязательна, session storage требуется, менее подходит для API.

Bearer token это стандарт для API auth. Client отправляет Authorization header с Bearer prefix и token value. BearerTokenAuthenticationFilter извлекает токен, обычно JWT. JwtDecoder верифицирует подпись и claims. Создаёт JwtAuthenticationToken с authorities.

Плюсы stateless, легко масштабируется, стандарт OAuth2. Минусы токен нужно надёжно хранить на client, revocation сложнее чем session invalidation.

Cookies через session это классика для UI. Успешный login создаёт session на сервере в memory или Redis, browser получает session cookie, каждый последующий запрос отправляет cookie для идентификации session. SessionCreationPolicy контролирует поведение — always создаёт session даже для anonymous, if_required только по необходимости, never не создаёт но использует existing, stateless вообще не работает с session.

Для микросервисной архитектуры чаще предпочтителен stateless режим с JWT потому что sessions требуют либо sticky sessions либо shared session store что усложняет scaling. JWT позволяет любой instance service обрабатывать любой запрос без coordination.

## Stateless против stateful

Stateful architecture с session на сервере имеет свои преимущества и ограничения. Session store в memory или Redis хранит per-user данные. Cookie с session-id ссылается на записи в store. Server может хранить много данных в session — user preferences, cached data, workflow state.

Плюсы включают легкость invalidation — просто удалить session и все дальнейшие запросы будут unauthenticated. Возможность хранить больше данных чем помещается в token. Простота security model — session id одна короткая случайная строка.

Минусы связаны с масштабированием. Sticky sessions требуют direct routing от load balancer к конкретному instance имеющему session. Либо shared session store например Redis добавляющий infrastructure complexity. Failover при потере instance теряет session если не replicated. Cross-datacenter замедляется round trips к shared store.

Stateless architecture с JWT решает проблемы масштабирования. Каждый запрос несёт полный context в токене. Server ничего не хранит per-user — только проверяет подпись и claims. Любой instance может обработать любой запрос без coordination. Failover тривиален потому что нет per-instance state.

Плюсы масштабирования линейно с количеством instances. Не нужен shared session store. Easy proxy через load balancer без специальной logic. Cross-datacenter работает нормально потому что нет centralized state.

Минусы включают сложность invalidation. Если пользователь logged out или compromised token — token остаётся валидным до его expiration. Требуются mechanisms вроде short TTL с частым refresh, revocation list по jti или user_id, session versioning. Токен большой поскольку содержит все claims — размер запроса больше чем с session cookie. Rotation ключей требует careful handling потому что старые токены должны валидироваться пока не истекут.

Для микросервисной архитектуры stateless практически всегда правильный выбор. sessionCreationPolicy STATELESS плюс OAuth2 resource server с JWT — стандартный setup.

## CSRF защита

Cross-Site Request Forgery это атака когда атакующий сайт заставляет browser жертвы отправить запрос на защищённый ресурс от её имени. Классический пример — жертва logged in на bank.com, посещает evil.com который содержит iframe или image src на bank.com/transfer с параметрами. Browser автоматически прикладывает session cookie к запросу поскольку он идёт на bank.com, request выглядит как legitimate от жертвы.

CSRF token решает проблему. Сервер генерирует случайный токен для session или запроса. Legitimate формы включают токен как hidden field. Server проверяет что токен в запросе совпадает с ожидаемым. Атакующий не может узнать токен потому что same-origin policy запрещает читать response от bank.com с evil.com. Без токена запрос отклоняется.

CookieCsrfTokenRepository стандартная реализация. Хранит токен в cookie с HttpOnly false — необходимо чтобы JavaScript мог прочитать и добавить в X-CSRF-TOKEN header для AJAX запросов. Server проверяет соответствие header и cookie.

Правило применения. Session-based UI приложения обязательно с CSRF protection — атака реальна пока используются cookies для auth. REST API с Bearer token обычно без CSRF protection — атака не работает поскольку Bearer token не отправляется автоматически browser, требуется explicit JavaScript код, который защищён same-origin policy на предыдущем шаге.

Отключение через http.csrf.disable типично для REST API с JWT. Но это должно быть explicit решение с пониманием, не default для всего. Mixed приложения с и cookie session и API могут требовать различной настройки для разных paths.

## CORS

Cross-Origin Resource Sharing это механизм браузера для контроля AJAX запросов между разными origins. Origin определяется как комбинация scheme, host, port. Same-origin policy запрещает JavaScript от одного origin читать responses от другого. CORS предоставляет механизм для legitimate cross-origin запросов через опрос сервера.

Preflight OPTIONS request отправляется browser перед non-simple запросом (не GET/POST с определёнными content types). Server отвечает CORS headers указывающими какие origins, methods, headers разрешены. Browser разрешает основной запрос только если preflight успешен.

Настройка через CorsConfigurationSource bean в Spring Security. AllowedOrigins список origins которым разрешён доступ — обычно frontend URL. AllowedMethods HTTP methods которые могут быть использованы. AllowedHeaders какие custom headers frontend может отправлять. AllowCredentials разрешает cookies и Authorization headers в cross-origin запросах.

Активация через http.cors в SecurityFilterChain применяет CORS filter в цепочке до authentication. Это важно потому что preflight запросы не должны требовать authentication — они анонимные проверки политики.

Практическая настройка КНП — frontend на knp.kgd.gov.kz, API на api.knp.kgd.gov.kz. CORS разрешает knp.kgd.gov.kz как origin для API, все HTTP methods используемые в приложении, стандартные headers плюс Authorization для Bearer token, allowCredentials true для cookie session где применимо.

## Method Security

Method Security предоставляет тонкую авторизацию на уровне отдельных методов через аннотации. Активируется через @EnableMethodSecurity в конфигурации.

@PreAuthorize проверяет authorization до вызова метода. Принимает SpEL выражение оценивающееся с access к Authentication object как authentication переменная, method arguments по имени, custom bean references через @beanName. При false выражении бросается AccessDeniedException, метод не вызывается.

Примеры использования включают hasRole для проверки роли, hasAuthority для permission, комбинации через and или or, сложные условия сравнивающие arguments с authentication. Особенно полезно для owner-based access — @PreAuthorize("#userId == authentication.name") позволяет только владельцу изменять свой профиль.

@PostAuthorize проверяет после выполнения метода на основе returnObject. Полезно для сценариев где нельзя определить authorization до получения объекта — например ownership check на сущности возвращаемой методом. При false — метод уже выполнился с side effects что может быть проблемой для write operations, обычно применяется только к read операциям.

@Secured старый стиль поддерживающий только authority list без SpEL. Менее гибкий чем PreAuthorize, редко используется в новых проектах. @RolesAllowed из jakarta security стандарта аналогичен Secured.

Реализация Method Security через AOP proxy. Это означает что self-invocation имеет ту же проблему что и @Transactional — вызов annotated метода через this из того же класса минует proxy и проверка не срабатывает. Метод должен быть вызван через injected reference или через другой bean чтобы proxy применил security check.

## Обработка ошибок

Security errors бывают двух типов требующих разной обработки. Unauthorized означает что пользователь не аутентифицирован — не предоставил credentials или предоставил невалидные. HTTP 401 стандартный response. Forbidden означает что пользователь аутентифицирован но не имеет прав на конкретную операцию. HTTP 403 стандартный response.

AuthenticationEntryPoint обрабатывает unauthorized. Для UI приложений redirect на login page стандартный подход. Для REST API — возврат 401 с JSON error body. Конфигурация через exceptionHandling.authenticationEntryPoint в SecurityFilterChain.

AccessDeniedHandler обрабатывает forbidden. Стандартный ответ 403 с error page для UI или JSON error для API. Custom handler может добавить logging, метрики, специфический формат ответа.

Consistent error responses важны для API consumers. HTTP status codes должны точно соответствовать типу ошибки. Body должен содержать stable schema для программной обработки. Timestamps, trace IDs, error codes полезны для debugging и support.

## Attack protection

Spring Security защищает от нескольких классов атак через различные механизмы.

CSRF защита через token как описано выше.

Session fixation защищает от attacks где attacker заставляет victim использовать known session id. Spring регенерирует session id после successful authentication предотвращая использование pre-auth session. Активно по умолчанию в session-based auth.

Clickjacking защищается через X-Frame-Options header. Значение DENY запрещает загружать страницу в iframe что предотвращает overlay attacks. Значение SAMEORIGIN разрешает только same-origin iframe.

HSTS Strict-Transport-Security принуждает browser использовать HTTPS. После первого визита browser запоминает что site требует HTTPS и не позволяет downgrade на HTTP. Защищает от man-in-the-middle attacks пытающихся downgrade connection.

Content Security Policy позволяет declarative ограничения на loaded resources. Указывает откуда можно загружать scripts, styles, images. Строгий CSP значительно снижает impact XSS уязвимостей.

X-Content-Type-Options nosniff предотвращает browser от MIME type sniffing что закрывает определённые классы XSS через wrongly detected content types.

Headers конфигурация в SecurityFilterChain позволяет настроить все эти headers централизованно.

## Тестирование Security

Spring Security Test предоставляет utilities для тестирования security функциональности. @WithMockUser аннотация на test метод создаёт mock authenticated пользователя с указанными username и authorities. Полезно для unit тестов Service методов с @PreAuthorize.

Параметры включают username, password, roles для стандартных ROLE_ authorities, authorities для полного списка прав без префикса. Тест выполняется в context authenticated user.

MockMvc интеграция через with(user(...)) fluent API позволяет тестировать HTTP endpoints с mock authenticated user. Особенно полезно для integration tests контроллеров с security constraints.

@WithUserDetails загружает real UserDetails через UserDetailsService используя указанный username. Полезно когда тест требует реальные authorities из database не mock.

Custom @WithSecurityContext позволяет создать полностью custom security context с любой authentication. Для сложных сценариев вроде OAuth2 tokens с specific claims.

## Реальные грабли

Забытый disable CSRF в REST API создаёт классическую проблему. POST /api/fno возвращает 403 Forbidden с сообщением Invalid CSRF token в логах. Причина — Spring по default включает CSRF защиту, для REST API с Bearer token она не нужна и должна быть отключена явно.

Порядок SecurityFilterChain важен когда их несколько. @Order определяет который применяется первым. Первый матчащий SecurityFilterChain для конкретного path выигрывает. Ошибочный порядок может привести к тому что общая цепочка применится к специфическим paths, игнорируя более специфическую конфигурацию.

Self-invocation в @PreAuthorize имеет ту же проблему что и @Transactional. Метод с @PreAuthorize вызывается через this из того же класса — proxy не активируется, проверка не срабатывает. Fix — вынести метод в другой bean или вызывать через injected reference на self.

Забытый ROLE_ prefix в конфигурации создаёт confusion. hasRole ADMIN проверяет ROLE_ADMIN — Spring автоматически добавляет префикс. hasAuthority ADMIN проверяет буквально ADMIN без префикса. Смешение приводит к неработающим security constraints. Правило — hasRole для ролей, hasAuthority для permissions, консистентно применять.

SecurityContext не пробрасывается автоматически в другие threads. @Async, virtual threads, executors по default не наследуют контекст. DelegatingSecurityContextAsyncTaskExecutor или подобные wrappers необходимы. Альтернатива — SecurityContextHolder MODE_INHERITABLETHREADLOCAL но осторожно с thread pools где inherited context из pool creation может быть неверным.

## Полная базовая конфигурация

Собранная воедино security конфигурация для типичного REST API. Отключенный CSRF потому что Bearer token. Stateless session policy. Authorization rules по path — public paths permitAll, admin paths hasRole ADMIN, остальные authenticated. OAuth2 resource server с JWT для authentication. Exception handling с 401 для unauthorized. Headers включая X-Frame-Options DENY.

Password encoder bean обязателен если приложение имеет любую password-based auth. BCryptPasswordEncoder стандартный выбор. DelegatingPasswordEncoder для поддержки миграции между алгоритмами.

Не забывать что security конфигурация это не только SecurityFilterChain. Также включает CORS если применимо, method security для тонкой авторизации на service level, exception handlers для consistent error responses, тестовое покрытие security paths.

## Итоги

Spring Security это цепочка Servlet-фильтров обрабатывающая каждый HTTP запрос до контроллера. SecurityFilterChain bean декларирует цепочку с authorization rules, session policy, authentication mechanism, exception handling.

SecurityContext thread-local хранит current Authentication. SecurityContextHolder предоставляет статический доступ. GrantedAuthority представляет право пользователя, конвенция ROLE_ prefix для ролей.

UserDetails и UserDetailsService абстракции для локальной auth. PasswordEncoder для безопасного хранения паролей — BCrypt, Argon2 стандартные choices. AuthenticationManager с provider chain обрабатывает разные типы аутентификации.

Различные механизмы аутентификации — HTTP Basic, form login, Bearer/JWT, session cookies — каждый подходит для своих сценариев. Микросервисы обычно stateless с JWT.

CSRF защита обязательна для session-based UI, обычно отключается для REST API с Bearer. CORS настраивает cross-origin AJAX permissions.

Method Security через @PreAuthorize даёт fine-grained authorization на уровне методов. Self-invocation caveat как у @Transactional.

Attack protection покрывает CSRF, session fixation, clickjacking, HSTS, CSP, MIME sniffing. Стандартные headers настраиваются через SecurityFilterChain.

Тестирование через @WithMockUser, @WithUserDetails, MockMvc utilities. Custom scenarios через @WithSecurityContext.

Реальные грабли — забытый disable CSRF, неправильный порядок фильтр цепочек, self-invocation, забытый ROLE_ prefix, missing propagation SecurityContext в другие threads. Все имеют предсказуемые причины и стандартные fixes.

Дальше — OAuth2 и OIDC теория как основа для understanding modern authentication и authorization patterns.
