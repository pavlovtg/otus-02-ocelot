# Анализ сильных и слабых сторон архитектуры Ocelot

> Данный отчёт является продолжением архитектурного анализа из [architecture.md](./architecture.md).
> Описание модулей, паттернов и C4-диаграммы не дублируются — здесь только критический анализ.

---

## 1. Сильные стороны архитектуры

### 1.1. Чистая реализация паттерна Middleware Pipeline

Конвейер из 18 middleware (`OcelotPipelineExtensions.BuildOcelotPipeline`) реализует паттерн «Цепочка обязанностей» в его каноническом виде для ASP.NET Core. Каждый компонент отвечает ровно за одну задачу, передаёт управление через `RequestDelegate next` и не знает о соседях. Это обеспечивает:

- **Предсказуемый порядок выполнения** — порядок middleware жёстко задан в одном месте и легко читается
- **Независимость компонентов** — каждый middleware тестируется изолированно
- **Точки расширения** — `OcelotPipelineConfiguration` позволяет заменить любой встроенный middleware пользовательским через `configuration.AuthenticationMiddleware`, `configuration.AuthorizationMiddleware` и т.д., не трогая ядро

### 1.2. Принцип открытости/закрытости через DI

`OcelotBuilder` регистрирует все сервисы через `TryAddSingleton`, что позволяет пользователю переопределить любой компонент до вызова `AddOcelot()`. Это классическое применение принципа Open/Closed: ядро закрыто для изменений, но открыто для расширений. Примеры точек расширения:

- `ILoadBalancer` — кастомные балансировщики через `AddCustomLoadBalancer<T>()`
- `IServiceDiscoveryProvider` — кастомные провайдеры через `ServiceDiscoveryFinderDelegate`
- `IResponseAggregator` / `IDefinedAggregator` — кастомная агрегация ответов
- `DelegatingHandler` — перехватчики HTTP-запросов через `AddDelegatingHandler<T>()`

### 1.3. Паттерн Strategy + Factory для балансировщиков и провайдеров

Балансировщики нагрузки (`RoundRobin`, `LeastConnection`, `CookieStickySessions`, `NoLoadBalancer`) реализуют единый интерфейс `ILoadBalancer`. `LoadBalancerFactory` выбирает нужную реализацию по строке из конфигурации. Аналогично устроены провайдеры Service Discovery. Это позволяет добавлять новые стратегии без изменения существующего кода.

### 1.4. Fluent API для конфигурирования (`IOcelotBuilder`)

`OcelotBuilder` предоставляет удобный fluent API для подключения провайдеров, балансировщиков, агрегаторов и delegating handlers. Это снижает порог входа для пользователей библиотеки и делает конфигурацию читаемой:

```csharp
services.AddOcelot()
    .AddConsul()
    .AddPolly()
    .AddDelegatingHandler<MyHandler>(global: true);
```

### 1.5. Чёткое разделение слоёв конфигурации

Конфигурация проходит через три чётко разделённых слоя:

1. `FileConfiguration` — внешнее представление (JSON-файл)
2. Валидация через `FileConfigurationFluentValidator` с понятными сообщениями об ошибках
3. `IInternalConfiguration` — внутреннее представление, оптимизированное для runtime

Такое разделение позволяет изменять формат конфигурационного файла независимо от внутренней логики.

### 1.6. Реактивное обновление конфигурации без перезапуска

Механизм `OcelotConfigurationChangeTokenSource` + `OcelotConfigurationMonitor` реализует паттерн Observer поверх стандартного `IOptionsMonitor<T>` ASP.NET Core. Изменение конфигурации через Administration API или Consul KV автоматически распространяется на все компоненты без перезапуска приложения.

### 1.7. Изоляция провайдеров в отдельные пакеты

Consul, Kubernetes, Polly и Eureka вынесены в отдельные NuGet-пакеты. Это снижает граф зависимостей базового пакета и позволяет подключать только нужные интеграции. Каждый провайдер регистрируется через собственный `OcelotBuilderExtensions`.

---

## 2. Слабые стороны и уязвимости

### 2.1. `HttpContext.Items` как неявная шина данных между middleware

**Модуль:** `src/Ocelot/Middleware/HttpItemsExtensions.cs`

Все middleware передают данные друг другу через `HttpContext.Items` — словарь `IDictionary<object, object>` со строковыми ключами-«магическими строками»:

```csharp
input.Upsert("DownstreamRequest", downstreamRequest);
input.Upsert("DownstreamRoute", downstreamRoute);
input.Upsert("IInternalConfiguration", config);
```

**Проблемы:**
- Нет типобезопасности — ошибка в ключе приведёт к `null` или `InvalidCastException` в runtime
- Скрытые зависимости между middleware — невозможно понять из сигнатуры метода, какие данные он ожидает
- Нарушение принципа явных зависимостей — middleware не декларируют свои входные данные через конструктор
- Сложность тестирования — для теста нужно вручную заполнять словарь Items

### 2.2. Статическое разделяемое состояние в `CookieStickySessions`

**Модуль:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`

```csharp
private static readonly Dictionary<string, StickySession> Stored = new(); // TODO Inject instead of static sharing
```

Словарь сессий объявлен как `static` — он разделяется между всеми экземплярами `CookieStickySessions` в рамках одного процесса. Это означает:

- **Утечка состояния между маршрутами** — сессии разных маршрутов хранятся в одном словаре
- **Невозможность горизонтального масштабирования** — при нескольких инстансах Ocelot сессии не синхронизируются
- **Сложность тестирования** — статическое состояние сохраняется между тестами
- Сам разработчик оставил `TODO: Inject instead of static sharing`, признавая проблему

### 2.3. Блокирующий вызов async-метода внутри `lock` в `CookieStickySessions`

**Модуль:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`

```csharp
lock (Locker)
{
    var next = _loadBalancer.LeaseAsync(httpContext).GetAwaiter().GetResult(); // unfortunately the operation must be synchronous
}
```

Вызов `GetAwaiter().GetResult()` внутри `lock` — классический антипаттерн, способный вызвать дедлок в ASP.NET Core при использовании `SynchronizationContext`. Комментарий «unfortunately the operation must be synchronous» указывает на архитектурный компромисс, который не был устранён.

### 2.4. Блокирующий вызов async-метода в `PollConsul`

**Модуль:** `src/Ocelot.Provider.Consul/PollConsul.cs`

```csharp
lock (_lockObject)
{
    _services = _consulServiceDiscoveryProvider.GetAsync().GetAwaiter().GetResult();
}
```

Та же проблема: синхронный вызов async-метода внутри `lock`. При высокой нагрузке это создаёт узкое место — все потоки, запрашивающие список сервисов, блокируются на одном объекте синхронизации.

### 2.5. Глобальная блокировка в `RateLimiting` не масштабируется

**Модуль:** `src/Ocelot/RateLimiting/RateLimiting.cs`

```csharp
private static readonly object ProcessLocker = new();

lock (ProcessLocker)
{
    var entry = _storage.Get(counterId);
    counter = Count(entry, rule, now);
    _storage.Set(counterId, counter, expiration);
}
```

`ProcessLocker` объявлен как `static` — это единственный мьютекс для всех запросов ко всем маршрутам. При высокой нагрузке это становится глобальным узким местом. Кроме того, по умолчанию используется `MemoryCacheRateLimitStorage` — счётчики хранятся в памяти одного инстанса, что делает rate limiting неэффективным при горизонтальном масштабировании.

### 2.6. `SimpleJsonResponseAggregator` строит JSON конкатенацией строк

**Модуль:** `src/Ocelot/Multiplexer/SimpleJsonResponseAggregator.cs`

```csharp
contentBuilder.Append('{');
contentBuilder.Append($"\"{responseKeys[k]}\":{content}");
contentBuilder.Append('}');
```

Агрегатор строит JSON-ответ через `StringBuilder` и конкатенацию строк, не валидируя содержимое ответов downstream-сервисов. Если downstream вернёт невалидный JSON или пустую строку, итоговый ответ будет невалидным JSON без какого-либо предупреждения. Также жёстко задана фраза `"cannot return from aggregate..which reason phrase would you use?"` в качестве `ReasonPhrase`.

### 2.7. `ServiceDiscoveryProviderFactory` поддерживает только один делегат

**Модуль:** `src/Ocelot/ServiceDiscovery/ServiceDiscoveryProviderFactory.cs`

```csharp
_delegates = provider.GetService<ServiceDiscoveryFinderDelegate>(); // только один!
```

Фабрика получает только **один** `ServiceDiscoveryFinderDelegate` через `GetService` (не `GetServices`). Если зарегистрировано несколько провайдеров (например, Consul и Kubernetes одновременно), будет использован только последний зарегистрированный. Это архитектурное ограничение, которое не очевидно из документации.

### 2.8. Захват `IServiceProvider` через поле в `OcelotBuilder`

**Модуль:** `src/Ocelot/DependencyInjection/OcelotBuilder.cs`

```csharp
private IServiceProvider _serviceProvider; // TODO Reuse ActivatorUtilities factories?

public IOcelotBuilder AddCustomLoadBalancer<TLoadBalancer>(
    Func<IServiceProvider, DownstreamRoute, IServiceDiscoveryProvider, TLoadBalancer> loadBalancerFactoryFunc)
{
    ILoadBalancer Create(DownstreamRoute route, IServiceDiscoveryProvider discoveryProvider)
        => loadBalancerFactoryFunc(_serviceProvider, route, discoveryProvider);
    ILoadBalancerCreator implementationFactory(IServiceProvider provider)
    {
        _serviceProvider = provider; // захват через замыкание
        return new DelegateInvokingLoadBalancerCreator<TLoadBalancer>(Create);
    }
    Services.AddSingleton<ILoadBalancerCreator>(implementationFactory);
    return this;
}
```

`_serviceProvider` захватывается через замыкание при первом разрешении сервиса. Это нарушает принцип явных зависимостей и может привести к неожиданному поведению при использовании нескольких кастомных балансировщиков.

### 2.9. Статические поля конфигурации в `WatchKube`

**Модуль:** `src/Ocelot.Provider.Kubernetes/WatchKube.cs`

```csharp
public static int FailedSubscriptionRetrySeconds { get; set; } = 1;
public static int FirstResultsFetchingTimeoutSeconds { get; set; } = 1;
```

Параметры поведения провайдера вынесены в статические свойства класса, а не в конфигурацию. Это означает, что их нельзя задать через `ocelot.json` или DI — только через прямое присвоение в коде. Такой подход нарушает принцип конфигурируемости и усложняет тестирование.

### 2.10. Накопленный технический долг в виде `TODO`-комментариев

По всему коду рассыпаны `TODO`-комментарии, указывающие на известные проблемы, которые не были устранены:

| Файл | Комментарий |
|------|-------------|
| `LoadBalancerHouse.cs` | `// TODO TryAdd ?` |
| `LeastConnection.cs` | `//todo - maybe this should be moved somewhere else...?` |
| `CookieStickySessions.cs` | `// TODO Inject instead of static sharing` |
| `CookieStickySessions.cs` | `// TODO Get test coverage for this` |
| `OcelotBuilder.cs` | `// TODO Reuse ActivatorUtilities factories?` |
| `RateLimiting.cs` | `// TODO: The expiry approach doesn't make much sense in practice` |
| `RateLimitingMiddleware.cs` | `/// <summary>TODO: Produced Ocelot's headers don't follow industry standards.</summary>` |

---

## 3. Технический долг

### 3.1. Rate limiting не работает в multi-instance сценарии

По умолчанию `MemoryCacheRateLimitStorage` хранит счётчики в памяти одного процесса. При горизонтальном масштабировании (несколько инстансов Ocelot за балансировщиком) каждый инстанс ведёт свои счётчики независимо. Реальный лимит для клиента будет `N × limit`, где `N` — количество инстансов. `DistributedCacheRateLimitStorage` существует, но не является дефолтным и требует явной настройки.

### 3.2. Собственная реализация rate limiting вместо платформенной

В .NET 7 появился `Microsoft.AspNetCore.RateLimiting` с алгоритмами Fixed Window, Sliding Window, Token Bucket и Concurrency. Ocelot использует собственную реализацию, которая уступает платформенной по функциональности (нет sliding window, нет token bucket) и производительности. Миграция на платформенный rate limiting снизила бы технический долг и улучшила производительность.

### 3.3. Newtonsoft.Json вместо System.Text.Json

`ConsulFileConfigurationRepository` использует `Newtonsoft.Json` для сериализации/десериализации конфигурации. `OcelotBuilder` принудительно добавляет `AddNewtonsoftJson()` для MVC. Проект таргетирует .NET 8+, где `System.Text.Json` является предпочтительным и более производительным вариантом. Зависимость от Newtonsoft.Json увеличивает размер деплоя и замедляет сериализацию.

### 3.4. Отсутствие покрытия тестами критичных участков

Комментарий `// TODO Get test coverage for this` в `CookieStickySessions.CheckExpiry` указывает на непокрытый тестами код в критичном компоненте балансировщика. Логика истечения сессий не тестируется автоматически, что создаёт риск регрессий при изменениях.

### 3.5. Конфигурация `IInternalConfiguration` передаётся через `HttpContext.Items`

`IInternalConfiguration` — центральный объект конфигурации — передаётся между middleware через `HttpContext.Items` (ключ `"IInternalConfiguration"`), а не через прямую инъекцию. Это означает, что middleware не могут получить конфигурацию через конструктор и вынуждены обращаться к словарю Items в методе `Invoke`. Такой подход скрывает зависимости и усложняет тестирование.

### 3.6. Ограниченная поддержка нескольких провайдеров Service Discovery одновременно

Архитектура `ServiceDiscoveryProviderFactory` предполагает использование только одного провайдера Service Discovery на весь шлюз. Использование Consul для одних маршрутов и Kubernetes для других в рамках одного инстанса Ocelot не поддерживается из коробки.

---

## 4. Приоритизированный план улучшений

### 🔴 Критичные (блокирующие — риск дедлоков и некорректного поведения)

**П1. Устранить `GetAwaiter().GetResult()` внутри `lock`**

- **Где:** `CookieStickySessions.LeaseAsync()`, `PollConsul.GetAsync()`
- **Проблема:** Блокирующий вызов async-метода внутри `lock` — классический источник дедлоков
- **Решение:** Переработать `CookieStickySessions.LeaseAsync()` как полностью асинхронный метод с использованием `SemaphoreSlim` вместо `lock`; в `PollConsul` использовать `SemaphoreSlim` и `await`

**П2. Заменить статический словарь в `CookieStickySessions` на инжектируемое хранилище**

- **Где:** `CookieStickySessions.Stored`
- **Проблема:** Статическое состояние не масштабируется горизонтально, утечка между маршрутами
- **Решение:** Инжектировать `IDistributedCache` или `IMemoryCache` через конструктор; ключи изолировать по маршруту

### 🟡 Важные (улучшающие поддержку и надёжность)

**П3. Типобезопасная передача данных между middleware**

- **Где:** `HttpItemsExtensions.cs`, все middleware
- **Проблема:** Строковые ключи в `HttpContext.Items`, скрытые зависимости
- **Решение:** Ввести типизированные ключи через `HttpContext.Features` (`IFeatureCollection`) или создать специализированный `IOcelotContext` с явными свойствами, передаваемый через DI как scoped-сервис

**П4. Устранить глобальную блокировку в `RateLimiting`**

- **Где:** `RateLimiting.ProcessLocker`
- **Проблема:** Единый `static` мьютекс для всех запросов — узкое место при нагрузке
- **Решение:** Использовать `ConcurrentDictionary` с блокировкой на уровне ключа счётчика; рассмотреть миграцию на `Microsoft.AspNetCore.RateLimiting`

**П5. Миграция на `System.Text.Json`**

- **Где:** `ConsulFileConfigurationRepository`, `OcelotBuilder.AddDefaultAspNetServices()`
- **Проблема:** Зависимость от Newtonsoft.Json в .NET 8+ проекте
- **Решение:** Заменить `JsonConvert` на `JsonSerializer`; убрать `AddNewtonsoftJson()` из дефолтной конфигурации

**П6. Исправить `ServiceDiscoveryProviderFactory` для поддержки нескольких делегатов**

- **Где:** `ServiceDiscoveryProviderFactory`
- **Проблема:** `GetService<ServiceDiscoveryFinderDelegate>()` возвращает только один делегат
- **Решение:** Использовать `GetServices<ServiceDiscoveryFinderDelegate>()` и выбирать нужный делегат по типу провайдера из конфигурации маршрута

### 🟢 Желательные (оптимизационные и качественные улучшения)

**П7. Перенести параметры `WatchKube` в конфигурацию**

- **Где:** `WatchKube.FailedSubscriptionRetrySeconds`, `WatchKube.FirstResultsFetchingTimeoutSeconds`
- **Решение:** Добавить эти параметры в `KubeRegistryConfiguration` и передавать через конструктор

**П8. Улучшить `SimpleJsonResponseAggregator`**

- **Где:** `SimpleJsonResponseAggregator.MapAggregateContent()`
- **Решение:** Использовать `System.Text.Json.Utf8JsonWriter` для построения JSON; добавить валидацию JSON-ответов downstream; убрать хардкод `ReasonPhrase`

**П9. Устранить накопленные `TODO`-комментарии**

- **Где:** `LoadBalancerHouse`, `LeastConnection`, `CookieStickySessions`, `OcelotBuilder`, `RateLimiting`, `RateLimitingMiddleware`
- **Решение:** Завести задачи в трекере для каждого `TODO`; устранить в рамках соответствующих рефакторингов

**П10. Добавить тестовое покрытие для `CookieStickySessions.CheckExpiry`**

- **Где:** `CookieStickySessions.CheckExpiry()`
- **Решение:** Написать unit-тесты для логики истечения сессий с использованием `FakeTimeProvider` (.NET 8+)
