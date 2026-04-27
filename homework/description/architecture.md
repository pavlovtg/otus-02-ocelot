# Архитектурный анализ проекта Ocelot

## Содержание

1. [Введение](#введение)
2. [Архитектурные паттерны](#архитектурные-паттерны)
3. [Карта модулей](#карта-модулей)
4. [C4-диаграммы](#c4-диаграммы)
   - [C4 Context — уровень системы](#c4-context--уровень-системы)
   - [C4 Container — уровень контейнеров](#c4-container--уровень-контейнеров)
   - [C4 Component — уровень компонентов](#c4-component--уровень-компонентов)

---

## Введение

> Подробное описание проекта, его возможностей и технологического стека см. в [detailed-description.md](./detailed-description.md).

С архитектурной точки зрения Ocelot реализован как **конвейер ASP.NET Core middleware**, выстроенных в строго определённом порядке. Каждый входящий HTTP-запрос последовательно проходит через 18 middleware-компонентов — от маршрутизации и аутентификации до балансировки нагрузки и проксирования. Ответ от downstream-сервиса возвращается обратно через тот же конвейер.

Ключевая архитектурная особенность — **высокая расширяемость через DI**: практически каждый компонент (балансировщик, провайдер service discovery, агрегатор, middleware) заменяется или дополняется без изменения ядра.

---

## Архитектурные паттерны

### 1. Middleware Pipeline (Цепочка обязанностей)

Основной архитектурный паттерн проекта. Все запросы обрабатываются последовательно через 18 middleware-компонентов, выстроенных в строгом порядке в методе `BuildOcelotPipeline`. Каждый middleware выполняет свою задачу и передаёт управление следующему через `RequestDelegate next`.

```
ConfigurationMiddleware
  → ExceptionHandlerMiddleware
    → ResponderMiddleware
      → DownstreamRouteFinderMiddleware
        → MultiplexingMiddleware
          → SecurityMiddleware
            → HttpHeadersTransformationMiddleware
              → DownstreamRequestInitialiserMiddleware
                → RateLimitingMiddleware
                  → RequestIdMiddleware
                    → AuthenticationMiddleware
                      → ClaimsToClaimsMiddleware
                        → AuthorizationMiddleware
                          → ClaimsToHeadersMiddleware
                            → ClaimsToQueryStringMiddleware
                              → LoadBalancingMiddleware
                                → DownstreamUrlCreatorMiddleware
                                  → OutputCacheMiddleware
                                    → HttpRequesterMiddleware
```

### 2. Strategy (Стратегия)

Применяется в нескольких ключевых точках расширяемости:

- **Балансировщики нагрузки** — интерфейс `ILoadBalancer` с реализациями `RoundRobin`, `LeastConnection`, `CookieStickySessions`, `NoLoadBalancer` и возможностью подключения кастомных балансировщиков
- **Провайдеры Service Discovery** — интерфейс `IServiceDiscoveryProvider` с реализациями `Consul`, `PollConsul`, `Kube`, `WatchKube`, `PollKube`, `Eureka` и кастомными провайдерами
- **Агрегаторы ответов** — интерфейс `IResponseAggregator` с реализацией `SimpleJsonResponseAggregator` и поддержкой пользовательских агрегаторов через `IDefinedAggregator`

### 3. Factory (Фабрика)

Используется для создания объектов с выбором конкретной реализации в runtime:

- `LoadBalancerFactory` — создаёт нужный балансировщик по типу из конфигурации
- `ServiceDiscoveryProviderFactory` — создаёт провайдер service discovery по типу
- `InMemoryResponseAggregatorFactory` — выбирает агрегатор для маршрута
- `QoSFactory` — создаёт политику Quality of Service (Polly pipeline)
- `DelegatingHandlerFactory` — создаёт цепочку `DelegatingHandler` для HTTP-клиента

### 4. Repository (Репозиторий)

Абстрагирует хранение и получение конфигурации:

- `IInternalConfigurationRepository` / `InMemoryInternalConfigurationRepository` — хранение внутренней конфигурации в памяти
- `IFileConfigurationRepository` / `DiskFileConfigurationRepository` — чтение конфигурации с диска
- `ConsulFileConfigurationRepository` — хранение конфигурации в Consul KV Store

### 5. Builder (Строитель)

Fluent API для конфигурирования Ocelot при старте приложения:

- `IOcelotBuilder` / `OcelotBuilder` — регистрация всех сервисов, добавление провайдеров, балансировщиков, агрегаторов, delegating handlers
- `OcelotPipelineConfiguration` — конфигурирование конвейера middleware с возможностью переопределения отдельных шагов

### 6. Decorator (Декоратор)

Применяется для расширения функциональности без изменения базовых классов:

- `ConfigAwarePlaceholders` — декорирует `IPlaceholders`, добавляя поддержку плейсхолдеров из конфигурации
- `DefaultConsulServiceBuilder` — базовый класс с виртуальными методами, который можно переопределить для кастомизации построения downstream-хоста из данных Consul

### 7. Observer / Change Tracking (Наблюдатель)

Реализует реактивное обновление конфигурации без перезапуска приложения:

- `OcelotConfigurationChangeTokenSource` — источник токенов изменения конфигурации
- `OcelotConfigurationMonitor` — реализует `IOptionsMonitor<IInternalConfiguration>` для отслеживания изменений
- `IOptionsMonitor<FileConfiguration>` — ASP.NET Core механизм отслеживания изменений файла конфигурации с автоматической перезагрузкой

### 8. BFF / Gateway Aggregation (Агрегация на шлюзе)

Паттерн Backend for Frontend реализован через:

- `MultiplexingMiddleware` — разветвляет один входящий запрос на несколько параллельных запросов к downstream-сервисам (`Task.WhenAll`)
- `IResponseAggregator` — объединяет ответы в единый JSON-ответ
- `IDefinedAggregator` — позволяет реализовать кастомную логику агрегации

---

## Карта модулей

| Модуль | Назначение | Ключевые интерфейсы | Зависимости |
|--------|-----------|---------------------|-------------|
| **Middleware** | Конвейер обработки запросов, точки расширения | `OcelotMiddleware`, `OcelotPipelineConfiguration` | Все остальные модули |
| **Configuration** | Загрузка, валидация, преобразование и хранение конфигурации | `IInternalConfiguration`, `IInternalConfigurationCreator`, `IFileConfigurationRepository` | DependencyInjection |
| **DependencyInjection** | Регистрация всех сервисов, fluent API для расширений | `IOcelotBuilder`, `OcelotBuilder` | Все модули |
| **DownstreamRouteFinder** | Поиск downstream-маршрута по URL, методу, заголовкам | `IDownstreamRouteProvider`, `IDownstreamRouteProviderFactory` | Configuration |
| **LoadBalancer** | Балансировка нагрузки между экземплярами downstream-сервиса | `ILoadBalancer`, `ILoadBalancerFactory`, `ILoadBalancerHouse` | ServiceDiscovery, Configuration |
| **ServiceDiscovery** | Обнаружение downstream-сервисов | `IServiceDiscoveryProvider`, `IServiceDiscoveryProviderFactory` | Configuration |
| **Authentication** | Аутентификация входящих запросов | `AuthenticationMiddleware` | ASP.NET Core Authentication |
| **Authorization** | Авторизация по claims и scopes | `IClaimsAuthorizer`, `IScopesAuthorizer` | Authentication, Claims |
| **Claims** | Трансформация claims из токена в заголовки, query string, path | `IAddClaimsToRequest`, `IAddHeadersToRequest`, `IAddQueriesToRequest` | Authentication |
| **RateLimiting** | Ограничение частоты запросов | `RateLimitingMiddleware` | Configuration, Infrastructure |
| **Cache** | Кэширование ответов downstream-сервисов | `OutputCacheMiddleware`, `IOcelotCache<T>` | Configuration |
| **QualityOfService** | Circuit breaker через Polly | `IQoSFactory`, `QoSOptions` | Configuration, Polly |
| **Multiplexer** | Агрегация ответов нескольких downstream-сервисов (BFF) | `MultiplexingMiddleware`, `IResponseAggregator`, `IDefinedAggregator` | Configuration, Requester |
| **Requester** | Отправка HTTP-запросов к downstream-сервисам | `IHttpRequester`, `IMessageInvokerPool` | Configuration, QoS |
| **Responder** | Формирование HTTP-ответа клиенту | `IHttpResponder`, `ResponderMiddleware` | Errors |
| **Headers** | Трансформация заголовков запроса и ответа | `IHttpResponseHeaderReplacer`, `IHttpContextRequestHeaderReplacer` | Configuration |
| **Security** | IP-фильтрация (whitelist/blacklist) | `ISecurityPolicy`, `IPSecurityPolicy` | Configuration |
| **WebSockets** | Проксирование WebSocket-соединений | `WebSocketsProxyMiddleware`, `IWebSocketsFactory` | DownstreamRouteFinder, LoadBalancer |
| **Administration** | HTTP API для управления конфигурацией в runtime | `IFileConfigurationSetter`, `FileConfigurationController` | Configuration, Authentication |
| **Logging** | Структурированное логирование | `IOcelotLogger`, `IOcelotLoggerFactory` | Все модули |
| **Ocelot.Provider.Consul** | Service discovery и хранение конфигурации через Consul | `Consul`, `PollConsul`, `ConsulFileConfigurationRepository` | ServiceDiscovery, Configuration |
| **Ocelot.Provider.Kubernetes** | Service discovery через Kubernetes Endpoints API | `Kube`, `WatchKube`, `PollKube` | ServiceDiscovery |

---

## C4-диаграммы

### C4 Context — уровень системы

Диаграмма показывает Ocelot в контексте внешних систем и пользователей.

```plantuml
@startuml C4_Context
!include <C4/C4_Context>

title Ocelot API Gateway — C4 Context Diagram

Person(client, "Клиент", "Браузер, мобильное приложение\nили другой сервис, обращающийся\nк API через HTTP/WebSocket")

System(ocelot, "Ocelot API Gateway", "API Gateway на базе ASP.NET Core.\nЕдиная точка входа в систему микросервисов.\nМаршрутизация, аутентификация, балансировка,\nrate limiting, кэширование, агрегация.")

System_Ext(downstream_services, "Downstream-сервисы", "Микросервисы бизнес-логики:\nREST API, gRPC, WebSocket-сервисы.\nРаботают по HTTP/HTTPS.")

System_Ext(identity_provider, "Identity Provider", "Сервер аутентификации:\nIdentityServer, Auth0, Okta и др.\nВыдаёт JWT Bearer токены.")

System_Ext(consul, "HashiCorp Consul", "Service Discovery и\nхранилище конфигурации (KV Store).\nРегистрирует и отслеживает\nдоступность сервисов.")

System_Ext(kubernetes, "Kubernetes", "Оркестратор контейнеров.\nПредоставляет Endpoints API\nдля обнаружения подов.")

System_Ext(eureka, "Netflix Eureka", "Service Discovery провайдер\nдля экосистемы Spring/Steeltoe.")

Rel(client, ocelot, "Отправляет HTTP/WebSocket запросы", "HTTPS")
Rel(ocelot, downstream_services, "Проксирует запросы к сервисам", "HTTP/HTTPS")
Rel(ocelot, identity_provider, "Валидирует JWT токены", "HTTPS")
Rel(ocelot, consul, "Обнаруживает сервисы,\nчитает/пишет конфигурацию", "HTTP")
Rel(ocelot, kubernetes, "Обнаруживает сервисы\nчерез Endpoints API", "HTTPS")
Rel(ocelot, eureka, "Обнаруживает сервисы", "HTTP")

SHOW_LEGEND()
@enduml
```

---

### C4 Container — уровень контейнеров

Диаграмма показывает основные контейнеры (развёртываемые единицы) Ocelot и их взаимодействие.

```plantuml
@startuml C4_Container
!include <C4/C4_Container>

title Ocelot API Gateway — C4 Container Diagram

Person(client, "Клиент", "Браузер, мобильное приложение\nили другой сервис")

System_Boundary(ocelot_system, "Ocelot API Gateway") {

    Container(ocelot_core, "Ocelot Core", "ASP.NET Core Application\n(.NET 8+)", "Основное приложение шлюза.\nОбрабатывает входящие запросы,\nвыполняет маршрутизацию, аутентификацию,\nбалансировку и проксирование.\nКонфигурируется через ocelot.json.")

    Container(consul_provider, "Ocelot.Provider.Consul", "NuGet Package\n(C# Class Library)", "Расширение для интеграции с Consul.\nОбеспечивает service discovery\n(Consul, PollConsul) и хранение\nконфигурации в Consul KV Store.")

    Container(k8s_provider, "Ocelot.Provider.Kubernetes", "NuGet Package\n(C# Class Library)", "Расширение для интеграции с Kubernetes.\nОбеспечивает service discovery\nчерез Endpoints API\n(Kube, WatchKube, PollKube).")

    Container(polly_provider, "Ocelot.QualityOfService.Polly", "NuGet Package\n(C# Class Library)", "Расширение для Quality of Service.\nРеализует circuit breaker\nи retry-политики через Polly v8.")

    Container(config_file, "ocelot.json", "JSON Configuration File", "Файл конфигурации маршрутов.\nОпределяет Routes, Aggregates,\nGlobalConfiguration.\nМожет храниться в Consul KV.")
}

System_Ext(downstream_services, "Downstream-сервисы", "Микросервисы бизнес-логики")
System_Ext(identity_provider, "Identity Provider", "JWT-сервер аутентификации")
System_Ext(consul_server, "Consul Agent/Server", "HashiCorp Consul")
System_Ext(k8s_api, "Kubernetes API Server", "K8s Control Plane")

Rel(client, ocelot_core, "HTTP/WebSocket запросы", "HTTPS :443")
Rel(ocelot_core, downstream_services, "Проксированные запросы", "HTTP/HTTPS")
Rel(ocelot_core, identity_provider, "Валидация токенов", "HTTPS")
Rel(ocelot_core, config_file, "Читает конфигурацию при старте\nи при изменении файла", "File I/O")

Rel(ocelot_core, consul_provider, "Использует для service discovery\nи хранения конфигурации", "In-process")
Rel(consul_provider, consul_server, "Запросы к Health API\nи KV Store", "HTTP :8500")

Rel(ocelot_core, k8s_provider, "Использует для service discovery", "In-process")
Rel(k8s_provider, k8s_api, "Запросы к Endpoints API,\nWatch-подписки", "HTTPS :6443")

Rel(ocelot_core, polly_provider, "Применяет QoS-политики\nдля каждого маршрута", "In-process")

SHOW_LEGEND()
@enduml
```

---

### C4 Component — уровень компонентов

Диаграмма показывает ключевые компоненты внутри Ocelot Core и их взаимодействие.

```plantuml
@startuml C4_Component
!include <C4/C4_Component>

title Ocelot Core — C4 Component Diagram

Person(client, "Клиент", "HTTP/WebSocket")
System_Ext(downstream, "Downstream-сервисы", "Микросервисы")
System_Ext(identity, "Identity Provider", "JWT")
System_Ext(service_registry, "Service Registry", "Consul / K8s / Eureka")

Container_Boundary(ocelot_core, "Ocelot Core (ASP.NET Core Application)") {

    Component(di_container, "DI Container\n(OcelotBuilder)", "C# / ASP.NET Core DI", "Регистрирует все сервисы.\nFluent API для подключения\nпровайдеров и расширений.\nТочка входа: AddOcelot().")

    Component(config_subsystem, "Configuration Subsystem", "C# / IOptions<T>", "Загружает ocelot.json с диска.\nВалидирует конфигурацию.\nПреобразует FileConfiguration\nв InternalConfiguration.\nОтслеживает изменения файла.")

    Component(middleware_pipeline, "Middleware Pipeline", "ASP.NET Core Middleware", "Конвейер из 18 middleware.\nОбрабатывает каждый запрос\nпоследовательно.\nТочка входа: UseOcelot().")

    Component(routing_engine, "Routing Engine\n(DownstreamRouteFinder)", "C# / Regex", "Сопоставляет входящий URL\nс шаблонами маршрутов.\nПоддерживает плейсхолдеры,\nприоритеты, upstream host\nи upstream headers.")

    Component(security_module, "Security Module", "C# / IPAddressRange", "IP-фильтрация запросов.\nWhitelist и blacklist\nпо IP-адресам и CIDR-диапазонам.")

    Component(auth_module, "Authentication &\nAuthorization", "ASP.NET Core Auth", "Аутентификация через\nJWT Bearer / IdentityServer.\nАвторизация по claims и scopes.\nТрансформация claims.")

    Component(rate_limiter, "Rate Limiter", "C# / In-Memory", "Ограничение частоты запросов.\nФиксированное окно и\nгибридный алгоритм.\nПо клиентскому заголовку.")

    Component(load_balancer, "Load Balancer", "C# / Strategy Pattern", "Балансировка нагрузки:\nRoundRobin, LeastConnection,\nCookieStickySessions,\nNoLoadBalancer.\nПоддержка кастомных балансировщиков.")

    Component(service_discovery, "Service Discovery\nFactory", "C# / Factory Pattern", "Создаёт провайдер обнаружения\nсервисов по типу из конфигурации.\nПоддерживает Consul, K8s,\nEureka, кастомные провайдеры.")

    Component(multiplexer, "Multiplexer /\nAggregator (BFF)", "C# / Task.WhenAll", "Разветвляет запрос на несколько\nparallel downstream-запросов.\nАгрегирует ответы в один JSON.\nПоддержка кастомных агрегаторов.")

    Component(cache_module, "Output Cache", "C# / CacheManager", "Кэширование ответов\ndownstream-сервисов по URL.\nTTL-based инвалидация.\nПоддержка внешних кэшей.")

    Component(qos_module, "Quality of Service\n(Polly)", "C# / Polly v8", "Circuit breaker для\ndownstream-сервисов.\nTimeout-политики.\nRetry-стратегии.")

    Component(http_requester, "HTTP Requester\n(MessageInvokerPool)", "C# / HttpClient", "Отправляет HTTP-запросы\nк downstream-сервисам.\nПул HttpMessageInvoker.\nПоддержка DelegatingHandlers\nи трассировки.")

    Component(responder, "Responder", "C# / ASP.NET Core", "Формирует HTTP-ответ клиенту.\nМаппинг ошибок на HTTP-коды.\nТрансформация заголовков ответа.")

    Component(admin_api, "Administration API", "ASP.NET Core MVC\nController", "REST API для управления\nконфигурацией в runtime.\nЗащищён через IdentityServer.")

    Component(websocket_proxy, "WebSocket Proxy", "ASP.NET Core\nWebSockets", "Проксирование WebSocket-соединений.\nОтдельная ветка конвейера\n(MapWhen).")
}

' Внешние взаимодействия
Rel(client, middleware_pipeline, "HTTP/WebSocket запрос", "HTTPS")
Rel(middleware_pipeline, downstream, "Проксированный запрос", "HTTP/HTTPS")
Rel(auth_module, identity, "Валидация JWT токена", "HTTPS")
Rel(service_discovery, service_registry, "Получение списка\nэкземпляров сервиса", "HTTP/HTTPS")

' Внутренние взаимодействия
Rel(di_container, config_subsystem, "Инициализирует при старте")
Rel(di_container, middleware_pipeline, "Регистрирует все middleware")

Rel(middleware_pipeline, routing_engine, "1. Поиск маршрута")
Rel(middleware_pipeline, security_module, "2. IP-фильтрация")
Rel(middleware_pipeline, auth_module, "3. Аутентификация\nи авторизация")
Rel(middleware_pipeline, rate_limiter, "4. Rate limiting")
Rel(middleware_pipeline, multiplexer, "5. Агрегация (BFF)")
Rel(middleware_pipeline, load_balancer, "6. Балансировка нагрузки")
Rel(middleware_pipeline, cache_module, "7. Кэширование")
Rel(middleware_pipeline, http_requester, "8. Отправка запроса")
Rel(middleware_pipeline, responder, "9. Формирование ответа")
Rel(middleware_pipeline, websocket_proxy, "WebSocket ветка")

Rel(load_balancer, service_discovery, "Получает список\nдоступных экземпляров")
Rel(http_requester, qos_module, "Применяет QoS-политику")
Rel(config_subsystem, middleware_pipeline, "Предоставляет\nInternalConfiguration")
Rel(admin_api, config_subsystem, "Обновляет конфигурацию\nв runtime")

SHOW_LEGEND()
@enduml
```
