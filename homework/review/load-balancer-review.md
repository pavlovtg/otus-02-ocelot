# Code Review: модуль LoadBalancer

> **Проект:** Ocelot — API Gateway на ASP.NET Core  
> **Модуль:** LoadBalancer  
> **Дата:** 2026-04-27  
> **Ревьюер:** AI-ассистент (Cline / Claude Sonnet 4.5)

---

## Контекст

Модуль LoadBalancer реализует паттерн Strategy: интерфейс `ILoadBalancer` с четырьмя реализациями — `RoundRobin`, `LeastConnection`, `CookieStickySessions`, `NoLoadBalancer`. Каждый входящий запрос к downstream-сервису проходит через `LoadBalancingMiddleware`, которая вызывает `LeaseAsync()` для получения адреса сервиса и `Release()` после завершения запроса. Модуль находится на критическом пути — от его корректности зависит доступность всего API Gateway.

---

## 1. Баги и ошибки выполнения

### 1.1 [Bug] Инвертированная логика истечения сессии в `CheckExpiry`

**Файл:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`, строки 56–68

```csharp
private void CheckExpiry(StickySession sticky)
{
    lock (Locker)
    {
        if (!Stored.TryGetValue(sticky.Key, out var session) || session.Expiry >= DateTime.UtcNow)
        {
            return; // ← выходим, если сессия ЕЩЁ НЕ истекла
        }

        Stored.Remove(session.Key);
        _loadBalancer.Release(session.HostAndPort);
    }
}
```

**Проблема:** Условие `session.Expiry >= DateTime.UtcNow` означает «время истечения ещё не наступило» — то есть сессия живая. Метод выходит (`return`) именно тогда, когда сессия живая, и удаляет её, когда `Expiry < UtcNow` — то есть когда она уже истекла. На первый взгляд это выглядит правильно, но логика `||` делает условие составным: метод возвращается если **либо** ключ не найден, **либо** сессия не истекла. Это корректно. Однако проблема в другом: `CheckExpiry` вызывается через `IBus<StickySession>` с задержкой `_keyExpiryInMs`, но в `Update()` публикуется **обновлённая** сессия с новым временем истечения, а не исходная. Это означает, что при каждом обращении к уже существующей сессии в шину публикуется новое сообщение с новым временем, но старые сообщения в шине остаются и будут обработаны позже — они попытаются удалить сессию, которая уже была обновлена. Реализация `InMemoryBus` не отменяет старые сообщения, что приводит к накоплению «мусорных» событий.

**Решение:** При обновлении сессии отменять предыдущее событие истечения или использовать версионирование сессий. Минимальное исправление — проверять в `CheckExpiry`, что `sticky.Expiry` совпадает с `session.Expiry` (т.е. сообщение актуально):

```csharp
if (!Stored.TryGetValue(sticky.Key, out var session)
    || session.Expiry != sticky.Expiry  // сессия уже обновлена
    || session.Expiry >= DateTime.UtcNow)
{
    return;
}
```

---

### 1.2 [Bug] `GetAwaiter().GetResult()` внутри `lock` — риск дедлока

**Файл:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`, строка 87

```csharp
lock (Locker)
{
    // ...
    var next = _loadBalancer.LeaseAsync(httpContext).GetAwaiter().GetResult(); // unfortunately the operation must be synchronous
    // ...
}
```

**Проблема:** Вызов `.GetAwaiter().GetResult()` блокирует текущий поток до завершения асинхронной операции. Если `_loadBalancer.LeaseAsync()` (например, `RoundRobin`) внутри пытается захватить тот же `Locker` или ожидает освобождения потока из пула (threadpool starvation), возникает дедлок. В ASP.NET Core синхронизационный контекст отсутствует, поэтому дедлок маловероятен, но threadpool exhaustion при высокой нагрузке реален: поток заблокирован внутри `lock`, не может вернуться в пул, новые запросы ждут потока — система деградирует.

**Решение:** Переделать `LeaseAsync` в `CookieStickySessions` на полностью асинхронный код с использованием `SemaphoreSlim` вместо `lock`:

```csharp
private readonly SemaphoreSlim _semaphore = new(1, 1);

public async Task<Response<ServiceHostAndPort>> LeaseAsync(HttpContext httpContext)
{
    await _semaphore.WaitAsync();
    try
    {
        // ... логика без GetAwaiter().GetResult()
        var next = await _loadBalancer.LeaseAsync(httpContext);
        // ...
    }
    finally
    {
        _semaphore.Release();
    }
}
```

---

### 1.3 [Bug] Статический `Stored` в `CookieStickySessions` — утечка между маршрутами и экземплярами

**Файл:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`, строка 43

```csharp
private static readonly Dictionary<string, StickySession> Stored = new(); // TODO Inject instead of static sharing
```

**Проблема:** Словарь `Stored` является статическим — он разделяется между **всеми** экземплярами `CookieStickySessions` в процессе. Это означает:
1. При наличии нескольких маршрутов с `CookieStickySessions` их сессии хранятся в одном словаре (ключ `$"{serviceName}:{cookie}"` частично защищает от коллизий, но не полностью).
2. При горизонтальном масштабировании (несколько pod-ов) сессии не синхронизируются между экземплярами — пользователь может попасть на другой pod и потерять привязку.
3. Словарь никогда не очищается полностью — при долгой работе и большом количестве уникальных cookie-значений он будет расти неограниченно (утечка памяти).

Сам код содержит TODO-комментарий, подтверждающий проблему.

**Решение:** Инжектировать хранилище сессий через DI как `IDistributedCache` или хотя бы как `IStickySessionStore` с реализацией по умолчанию на основе `ConcurrentDictionary` (не статической). Это также позволит использовать Redis для горизонтального масштабирования.

---

### 1.4 [Bug] Статический `SyncRoot` в `LeastConnection` — глобальный bottleneck

**Файл:** `src/Ocelot/LoadBalancer/Balancers/LeastConnection.cs`, строки 14–18

```csharp
#if NET9_0_OR_GREATER
private static readonly Lock SyncRoot = new();
#else
private static readonly object SyncRoot = new();
#endif
```

**Проблема:** `SyncRoot` объявлен как `static` — один объект блокировки на все экземпляры `LeastConnection`. Если в конфигурации Ocelot несколько маршрутов используют `LeastConnection`, все они будут конкурировать за один и тот же мьютекс. При высокой нагрузке это создаёт глобальный bottleneck: запросы к маршруту A блокируют запросы к маршруту B, хотя они полностью независимы.

**Решение:** Сделать `SyncRoot` экземплярным (убрать `static`):

```csharp
#if NET9_0_OR_GREATER
private readonly Lock _syncRoot = new();
#else
private readonly object _syncRoot = new();
#endif
```

Аналогичная проблема присутствует в `RoundRobin.cs` (строки 26–30) — там `SyncRoot` тоже статический.

---

### 1.5 [Bug] Лишний `await Task.FromResult()` и потенциальный `NullReferenceException` в `NoLoadBalancer`

**Файл:** `src/Ocelot/LoadBalancer/Balancers/NoLoadBalancer.cs`, строки 28–30

```csharp
var service = await Task.FromResult(services.FirstOrDefault());
return new OkResponse<ServiceHostAndPort>(service.HostAndPort);
```

**Проблема:** `await Task.FromResult(x)` — это бессмысленная операция: `Task.FromResult` создаёт уже завершённую задачу, `await` немедленно её разворачивает. Это лишняя аллокация объекта `Task` на каждый запрос (hot path). Кроме того, если `services` содержит элементы, но первый из них равен `null` (теоретически возможно при некорректном провайдере), `service.HostAndPort` выбросит `NullReferenceException`.

**Решение:**

```csharp
var service = services[0]; // список уже проверен на Count > 0
if (service == null)
{
    return new ErrorResponse<ServiceHostAndPort>(new ServicesAreNullError($"First service is null in {Type}!"));
}
return new OkResponse<ServiceHostAndPort>(service.HostAndPort);
```

---

## 2. Архитектурные ограничения

### 2.1 [Architecture] Отсутствие Health Check — балансировщики не исключают недоступные узлы

**Файлы:** `RoundRobin.cs` (строка 89), `LeastConnection.cs`, `NoLoadBalancer.cs`

```csharp
// RoundRobin.cs, строка 89:
// TODO Check real health status
while (next?.HostAndPort == null && stop-- > 0)
{
    index = last;
    next = readme[last];
    LastIndices[_serviceName] = (++last < length) ? last : 0;
}
```

**Проблема:** Все балансировщики получают список сервисов от Service Discovery и слепо доверяют ему. Если downstream-сервис упал, но Service Discovery ещё не успел это зафиксировать (или не поддерживает health check), балансировщик продолжает направлять запросы на недоступный узел. Это подтверждается открытыми багами #1041 и #1513. TODO-комментарий в `RoundRobin` прямо указывает на эту проблему.

**Решение:** Добавить в `ILoadBalancer` или в отдельный компонент механизм circuit breaker / health tracking: при получении ошибки от downstream помечать узел как временно недоступный и исключать его из ротации на настраиваемый период. Минимальный вариант — интеграция с Polly (уже используется в QoS-модуле).

---

### 2.2 [Architecture] `CookieStickySessions.Release()` — пустая реализация нарушает контракт

**Файл:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`, строки 108–110

```csharp
public void Release(ServiceHostAndPort hostAndPort)
{
    // пустое тело
}
```

**Проблема:** `CookieStickySessions` является декоратором над `ILoadBalancer` (внутри хранится `_loadBalancer`). Когда `LoadBalancingMiddleware` вызывает `Release()` после завершения запроса, вызов поглощается и не передаётся во внутренний балансировщик (`RoundRobin`). Это означает, что счётчик соединений в `RoundRobin._leasing` никогда не уменьшается — `RoundRobin` думает, что все соединения активны, хотя они давно завершены. При долгой работе это приводит к некорректному распределению нагрузки.

**Решение:**

```csharp
public void Release(ServiceHostAndPort hostAndPort)
{
    _loadBalancer.Release(hostAndPort);
}
```

---

### 2.3 [Architecture] `ILoadBalancer.LeaseAsync` принимает `HttpContext` — нарушение SRP

**Файл:** `src/Ocelot/LoadBalancer/Interfaces/ILoadBalancer.cs`, строка 11

```csharp
Task<Response<ServiceHostAndPort>> LeaseAsync(HttpContext httpContext);
```

**Проблема:** Балансировщик нагрузки — это алгоритм выбора сервиса из списка. Он не должен знать о HTTP-контексте. Передача `HttpContext` нарушает принцип единственной ответственности: `CookieStickySessions` использует `httpContext` для чтения cookie и маршрута, но `RoundRobin`, `LeastConnection` и `NoLoadBalancer` игнорируют его полностью. Это делает интерфейс неоднородным и затрудняет тестирование (нужно создавать `HttpContext` даже для балансировщиков, которым он не нужен).

**Решение:** Выделить необходимые данные в отдельный объект контекста балансировщика:

```csharp
public interface ILoadBalancer
{
    Task<Response<ServiceHostAndPort>> LeaseAsync(LoadBalancerContext context);
    void Release(ServiceHostAndPort hostAndPort);
    string Type { get; }
}

public record LoadBalancerContext(string ServiceName, string? StickySessionCookie);
```

> Примечание: это изменение публичного API, поэтому требует major-версии. Как минимум, стоит добавить перегрузку.

---

### 2.4 [Architecture] `LoadBalancerHouse` кэширует балансировщики без TTL и инвалидации

**Файл:** `src/Ocelot/LoadBalancer/LoadBalancerHouse.cs`, строки 30–33

```csharp
return _loadBalancers.TryGetValue(route.LoadBalancerKey, out var loadBalancer) &&
        loadBalancer.Type.Equals(route.LoadBalancerOptions.Type, StringComparison.OrdinalIgnoreCase)
    ? new OkResponse<ILoadBalancer>(loadBalancer)
    : GetResponse(route, config);
```

**Проблема:** Балансировщик кэшируется навсегда и заменяется только при смене **типа** балансировщика. Если изменились параметры (например, `ExpiryInMs` для `CookieStickySessions` или `Key` для cookie), кэшированный экземпляр продолжит работать со старыми параметрами. Также нет механизма очистки кэша при горячей перезагрузке конфигурации Ocelot.

**Решение:** При сравнении кэшированного балансировщика учитывать не только тип, но и хэш параметров `LoadBalancerOptions`. Добавить метод `Invalidate(string key)` в `ILoadBalancerHouse` для поддержки горячей перезагрузки.

---

### 2.5 [Architecture] WebSocket не вызывает `Release()` на балансировщике

**Файл:** `src/Ocelot/WebSockets/WebSocketsProxyMiddleware.cs`, строки 138–143

```csharp
await client.ConnectAsync(destinationUri, context.RequestAborted);
using var server = await context.WebSockets.AcceptWebSocketAsync(client.SubProtocol);
await Task.WhenAll(
    PumpAsync(client.ToWebSocket(), server, DefaultWebSocketBufferSize, context.RequestAborted),
    PumpAsync(server, client.ToWebSocket(), DefaultWebSocketBufferSize, context.RequestAborted));
// Release() нигде не вызывается
```

**Проблема:** `LoadBalancingMiddleware` вызывает `Release()` в блоке `finally` после `await _next.Invoke(httpContext)`. Для WebSocket-соединений `_next` — это `WebSocketsProxyMiddleware`, которая держит соединение открытым на протяжении всей сессии (минуты, часы). Когда соединение закрывается, `Release()` вызывается корректно. Однако если `WebSocketsProxyMiddleware` выбрасывает исключение (например, при `ConnectAsync`), `Release()` всё равно вызывается — это корректно. Но проблема в том, что при использовании `LeastConnection` счётчик соединений увеличивается при `Lease`, а уменьшается при `Release`. Для долгоживущих WebSocket-соединений это означает, что один сервис может накопить большой счётчик и перестать получать новые соединения. Это связано с багом #1513.

**Решение:** Для WebSocket-соединений использовать отдельную стратегию балансировки или добавить в `ILoadBalancer` метод `ReleaseAsync` с поддержкой отмены.

---

## 3. Читаемость и удобство использования

### 3.1 [Suggestion] Двойная блокировка в `CookieStickySessions.Update()`

**Файл:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`, строки 77–106

```csharp
public Task<Response<ServiceHostAndPort>> LeaseAsync(HttpContext httpContext)
{
    // ...
    lock (Locker)          // ← первый захват Locker
    {
        // ...
        Update(key, ss);   // ← вызов Update
        // ...
    }
}

protected void Update(string key, StickySession value)
{
    lock (Locker)          // ← второй захват того же Locker
    {
        Stored[key] = value;
        _bus.Publish(value, _keyExpiryInMs);
    }
}
```

**Проблема:** `Update()` захватывает `Locker`, который уже захвачен в `LeaseAsync`. В .NET `lock` реентрантен (один поток может захватить один и тот же объект несколько раз), поэтому дедлока нет. Но это запутывает читателя: непонятно, зачем `Update()` сам берёт блокировку, если он всегда вызывается из уже заблокированного контекста. Кроме того, `Update()` объявлен как `protected`, хотя класс не предназначен для наследования (нет `virtual`-методов, нет документации о расширяемости).

**Решение:** Убрать `lock` из `Update()` и сделать метод `private`. Если метод должен быть вызываем извне без блокировки — задокументировать это явно.

---

### 3.2 [Suggestion] Дублирование метода `Update(ref Lease, bool)` в `RoundRobin` и `LeastConnection`

**Файлы:** `src/Ocelot/LoadBalancer/Balancers/RoundRobin.cs` (строки 108–113), `src/Ocelot/LoadBalancer/Balancers/LeastConnection.cs` (строки 67–73)

```csharp
// RoundRobin.cs
private int Update(ref Lease item, bool increase)
{
    var index = _leasing.IndexOf(item);
    _ = increase ? item.Connections++ : item.Connections--;
    _leasing[index] = item;
    return index;
}

// LeastConnection.cs — идентичный код
private int Update(ref Lease item, bool increase)
{
    var index = _leases.IndexOf(item);
    _ = increase ? item.Connections++ : item.Connections--;
    _leases[index] = item;
    return index;
}
```

**Проблема:** Полностью идентичный код в двух классах. При исправлении бага в одном нужно не забыть исправить в другом.

**Решение:** Вынести общую логику в базовый класс `LoadBalancerBase` или в статический метод-расширение для `List<Lease>`.

---

### 3.3 [Suggestion] TODO-комментарии в production-коде

**Файлы:**
- `CookieStickySessions.cs`, строка 43: `// TODO Inject instead of static sharing`
- `CookieStickySessions.cs`, строка 58: `// TODO Get test coverage for this`
- `RoundRobin.cs`, строка 89: `// TODO Check real health status`
- `LeastConnection.cs`, строка 42: `//todo - maybe this should be moved somewhere else...?`
- `LoadBalancerHouse.cs`, строка 52: `// TODO TryAdd ?`

**Проблема:** TODO-комментарии в production-коде — это технический долг, который не отслеживается системой задач. Они создают ложное ощущение, что проблема «известна и будет исправлена», хотя на практике такие комментарии живут годами.

**Решение:** Каждый TODO должен быть оформлен как GitHub Issue с ссылкой в комментарии: `// TODO: #XXXX — Inject instead of static sharing`. Комментарии без привязки к задаче следует удалить.

---

### 3.4 [Suggestion] Статический `LastIndices` в `RoundRobin` — скрытое разделяемое состояние

**Файл:** `src/Ocelot/LoadBalancer/Balancers/RoundRobin.cs`, строка 25

```csharp
private static readonly Dictionary<string, int> LastIndices = new();
```

**Проблема:** `LastIndices` — статический словарь, разделяемый между всеми экземплярами `RoundRobin`. Ключ — `_serviceName`. Если два маршрута используют один и тот же `serviceName` (что возможно при определённой конфигурации), их индексы будут конфликтовать. Кроме того, словарь никогда не очищается — при создании и уничтожении экземпляров `RoundRobin` записи в `LastIndices` остаются навсегда (утечка памяти при динамической конфигурации).

**Решение:** Сделать `LastIndices` экземплярным полем (просто `private int _lastIndex`), защищённым тем же `SyncRoot`. Это упростит код и устранит скрытое разделяемое состояние.

---

## 4. Тесты

### 4.1 [Bug] Статический `Stored` не очищается между тестами — потенциальные flaky-тесты

**Файл:** `test/Ocelot.UnitTests/LoadBalancer/CookieStickySessionsTests.cs`

**Проблема:** `CookieStickySessions.Stored` — статический словарь. Тесты создают новые экземпляры `CookieStickySessions`, но `Stored` сохраняет данные между тестами. Если тест `Should_return_same_host_and_port` записал сессию с ключом `"Should_return_same_host_and_port:321"`, следующий тест, использующий тот же cookie-значение, найдёт эту запись и получит неожиданный результат. Порядок выполнения тестов в xUnit не гарантирован.

**Решение:** Добавить очистку `Stored` в `Dispose()` тестового класса или сделать `Stored` инжектируемым (что также решит архитектурную проблему 1.3). Временное решение для тестов:

```csharp
public void Dispose()
{
    // Очищаем статическое состояние после каждого теста
    typeof(CookieStickySessions)
        .GetField("Stored", BindingFlags.Static | BindingFlags.NonPublic)
        ?.GetValue(null)
        ?.GetType()
        .GetMethod("Clear")
        ?.Invoke(...);
}
```

---

### 4.2 Отсутствие тестов на `CheckExpiry` — явно указано в TODO

**Файл:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`, строка 58

```csharp
private void CheckExpiry(StickySession sticky)
{
    // TODO Get test coverage for this
```

**Проблема:** Метод `CheckExpiry` — ключевая логика истечения sticky-сессий — не покрыт тестами. Единственный тест `Should_expire_sticky_session` проверяет только факт вызова `Release()`, но не проверяет:
- что сессия действительно удаляется из `Stored`
- что живые сессии не удаляются
- что при отсутствии ключа в `Stored` метод не падает

**Решение:** Добавить параметризованные тесты для `CheckExpiry`:
- сессия с истёкшим временем → удаляется, `Release()` вызывается
- сессия с актуальным временем → не удаляется, `Release()` не вызывается
- ключ отсутствует в `Stored` → метод завершается без ошибок

---

### 4.3 Нет теста на конкурентный доступ к `CookieStickySessions`

**Файл:** `test/Ocelot.UnitTests/LoadBalancer/CookieStickySessionsTests.cs`

**Проблема:** В отличие от `LeastConnectionTests` (тест `Should_be_able_to_lease_and_release_concurrently`) и `RoundRobinTests` (тест `Lease_LoopThroughIndexRangeIndefinitelyUnderHighLoad_ShouldDistributeIndexValuesUniformly`), для `CookieStickySessions` нет ни одного теста на конкурентный доступ. При этом `CookieStickySessions` использует статический `Locker` и статический `Stored`, что делает её наиболее уязвимой к race conditions.

**Решение:** Добавить тест аналогичный `LeastConnectionTests.Should_be_able_to_lease_and_release_concurrently`:

```csharp
[Fact]
public async Task Should_be_thread_safe_under_concurrent_load()
{
    Arrange();
    GivenTheLoadBalancerReturns();
    GivenTheDownstreamRequestHasSessionId("concurrent-session");

    var tasks = Enumerable.Range(0, 50)
        .Select(_ => _stickySessions.LeaseAsync(_httpContext));
    var results = await Task.WhenAll(tasks);

    results.ShouldAllBe(r => !r.IsError);
    results.Select(r => r.Data.DownstreamHost).Distinct().Count().ShouldBe(1); // все на один хост
}
```

---

### 4.4 Тест использует Reflection для доступа к приватным методам — хрупкий тест

**Файл:** `test/Ocelot.UnitTests/LoadBalancer/RoundRobinTests.cs`, строки 167–191

```csharp
[Fact]
public void TryScanNext()
{
    var method = typeof(RoundRobin).GetMethod(nameof(TryScanNext), BindingFlags.Instance | BindingFlags.NonPublic);
    var field = typeof(RoundRobin).GetField("LastIndices", BindingFlags.Static | BindingFlags.NonPublic);
    // ...
    bool success = (bool)method.Invoke(roundRobin, parameters);
}
```

**Проблема:** Тест обращается к приватному методу `TryScanNext` и приватному статическому полю `LastIndices` через Reflection. Такой тест:
1. Сломается при переименовании метода или поля
2. Не проверяет публичный контракт — только детали реализации
3. Сложно читается и поддерживается

**Решение:** Тестировать поведение через публичный API (`LeaseAsync`), а не через приватные методы. Если логика `TryScanNext` достаточно сложна для отдельного тестирования — вынести её в отдельный класс с публичным интерфейсом.

---

### 4.5 Нет теста на случай, когда `Release()` не вызывается при исключении в следующем middleware

**Файл:** `test/Ocelot.UnitTests/LoadBalancer/LoadBalancerMiddlewareTests.cs`

**Проблема:** Тест `ShouldNot_LogDebug_WhenNextMiddlewareThrownException` проверяет, что исключение из `_next` пробрасывается наружу, но не проверяет, что `Release()` всё равно вызывается (в блоке `finally`). Это критично: если `Release()` не вызывается при исключении, счётчики соединений в `LeastConnection` будут расти бесконечно.

```csharp
// LoadBalancingMiddleware.cs, строки 58–66:
try
{
    await _next.Invoke(httpContext);
}
finally
{
    loadBalancer.Data.Release(hostAndPort.Data); // ← должен вызываться всегда
}
```

**Решение:** Добавить тест:

```csharp
[Fact]
public async Task Should_call_release_even_when_next_throws()
{
    Arrange();
    _next = _ => throw new Exception("downstream error");
    _loadBalancer.Setup(x => x.LeaseAsync(It.IsAny<HttpContext>()))
        .ReturnsAsync(new OkResponse<ServiceHostAndPort>(new ServiceHostAndPort("host", 80)));

    _middleware = new LoadBalancingMiddleware(_next, _loggerFactory.Object, _loadBalancerHouse.Object);
    await Assert.ThrowsAsync<Exception>(() => _middleware.Invoke(_httpContext));

    _loadBalancer.Verify(x => x.Release(It.IsAny<ServiceHostAndPort>()), Times.Once);
}
```

---

## 5. Документация

### 5.1 Интерфейсы без XML-документации

**Файлы:**
- `src/Ocelot/LoadBalancer/Interfaces/ILoadBalancer.cs` — методы `LeaseAsync` и `Release` не задокументированы
- `src/Ocelot/LoadBalancer/Interfaces/ILoadBalancerHouse.cs` — нет документации
- `src/Ocelot/LoadBalancer/Interfaces/ILoadBalancerFactory.cs` — нет документации
- `src/Ocelot/LoadBalancer/Interfaces/ILoadBalancerCreator.cs` — нет документации

```csharp
// ILoadBalancer.cs
public interface ILoadBalancer
{
    Task<Response<ServiceHostAndPort>> LeaseAsync(HttpContext httpContext); // ← нет документации
    void Release(ServiceHostAndPort hostAndPort);                           // ← нет документации
    string Type { get; }
}
```

**Проблема:** Интерфейсы — это публичный контракт модуля. Разработчик, реализующий кастомный балансировщик (что поддерживается через `AddCustomLoadBalancer`), не знает:
- что должен делать `LeaseAsync` при отсутствии сервисов
- когда вызывается `Release` и что должно произойти
- что означает `Type` и как он используется

**Решение:** Добавить XML-документацию к каждому методу интерфейса:

```csharp
/// <summary>
/// Выбирает следующий доступный downstream-сервис для обработки запроса.
/// </summary>
/// <param name="httpContext">Контекст текущего HTTP-запроса.</param>
/// <returns>
/// <see cref="OkResponse{T}"/> с адресом сервиса при успехе;
/// <see cref="ErrorResponse{T}"/> если нет доступных сервисов.
/// </returns>
Task<Response<ServiceHostAndPort>> LeaseAsync(HttpContext httpContext);

/// <summary>
/// Освобождает ресурсы, связанные с завершённым запросом к указанному сервису.
/// Вызывается гарантированно после каждого <see cref="LeaseAsync"/>, в том числе при исключениях.
/// </summary>
void Release(ServiceHostAndPort hostAndPort);
```

---

### 5.2 Отсутствие документации о потокобезопасности

**Файлы:** `CookieStickySessions.cs`, `RoundRobin.cs`, `LeastConnection.cs`

**Проблема:** Все балансировщики используют блокировки (`lock`/`SyncRoot`) для обеспечения потокобезопасности, но нигде не документируют это явно. Разработчик, реализующий кастомный балансировщик, не знает, должна ли его реализация быть потокобезопасной. Это особенно важно, так как `LoadBalancerHouse` кэширует экземпляры и один экземпляр обслуживает все параллельные запросы.

**Решение:** Добавить в `ILoadBalancer` XML-комментарий:

```csharp
/// <remarks>
/// Реализации должны быть потокобезопасными: один экземпляр балансировщика
/// обслуживает все параллельные запросы к маршруту.
/// </remarks>
```

---

### 5.3 Несоответствие между именем ошибки и её смыслом

**Файлы:** `src/Ocelot/LoadBalancer/Errors/ServicesAreNullError.cs`, `src/Ocelot/LoadBalancer/Errors/ServicesAreEmptyError.cs`

```csharp
// LeastConnection.cs, строка 37:
return new ErrorResponse<ServiceHostAndPort>(new ServicesAreNullError(
    $"Services were null/empty in {Type} for '{_serviceName}'..."));
```

**Проблема:** `ServicesAreNullError` используется в `LeastConnection` для случая, когда список сервисов пуст (`Count == 0`), хотя для этого есть отдельный `ServicesAreEmptyError`. В `RoundRobin` используется `ServicesAreEmptyError` для пустого списка и `ServicesAreNullError` для `null`-элемента внутри списка. Это несогласованное использование двух похожих типов ошибок затрудняет диагностику проблем.

**Решение:** Унифицировать использование: `ServicesAreNullError` — когда список `null`, `ServicesAreEmptyError` — когда список пуст или содержит только `null`-элементы. Обновить сообщения об ошибках соответственно.

---

## Итоговая таблица замечаний

| # | Категория | Файл | Строка | Серьёзность | Краткое описание |
|---|-----------|------|--------|-------------|-----------------|
| 1.1 | Bug | `CookieStickySessions.cs` | 56–68 | High | Накопление устаревших событий в шине при обновлении сессии |
| 1.2 | Bug | `CookieStickySessions.cs` | 87 | High | `GetAwaiter().GetResult()` внутри `lock` — риск дедлока |
| 1.3 | Bug | `CookieStickySessions.cs` | 43 | High | Статический `Stored` — утечка памяти и несовместимость с горизонтальным масштабированием |
| 1.4 | Bug | `LeastConnection.cs` | 14–18 | Medium | Статический `SyncRoot` — глобальный bottleneck для всех маршрутов |
| 1.5 | Bug | `NoLoadBalancer.cs` | 28–30 | Low | Лишний `await Task.FromResult()` — ненужная аллокация |
| 2.1 | Architecture | `RoundRobin.cs`, `LeastConnection.cs`, `NoLoadBalancer.cs` | — | Critical | Нет Health Check — балансировщики не исключают упавшие узлы (#1041, #1513) |
| 2.2 | Architecture | `CookieStickySessions.cs` | 108–110 | High | `Release()` не передаётся во внутренний балансировщик |
| 2.3 | Architecture | `ILoadBalancer.cs` | 11 | Medium | `LeaseAsync` принимает `HttpContext` — нарушение SRP |
| 2.4 | Architecture | `LoadBalancerHouse.cs` | 30–33 | Medium | Кэш балансировщиков без TTL и инвалидации по параметрам |
| 2.5 | Architecture | `WebSocketsProxyMiddleware.cs` | 138–143 | Medium | WebSocket не освобождает соединение корректно (#1513) |
| 3.1 | Suggestion | `CookieStickySessions.cs` | 77–106 | Low | Двойная блокировка в `Update()` — запутывает читателя |
| 3.2 | Suggestion | `RoundRobin.cs`, `LeastConnection.cs` | 108–113, 67–73 | Low | Дублирование метода `Update(ref Lease, bool)` |
| 3.3 | Suggestion | Несколько файлов | — | Low | TODO-комментарии без привязки к задачам |
| 3.4 | Suggestion | `RoundRobin.cs` | 25 | Medium | Статический `LastIndices` — скрытое разделяемое состояние |
| 4.1 | Bug | `CookieStickySessionsTests.cs` | — | High | Статический `Stored` не очищается между тестами — flaky |
| 4.2 | Tests | `CookieStickySessionsTests.cs` | — | Medium | Нет тестов на `CheckExpiry` (есть TODO в коде) |
| 4.3 | Tests | `CookieStickySessionsTests.cs` | — | Medium | Нет теста на конкурентный доступ |
| 4.4 | Tests | `RoundRobinTests.cs` | 167–191 | Low | Тест через Reflection — хрупкий |
| 4.5 | Tests | `LoadBalancerMiddlewareTests.cs` | — | Medium | Нет теста на вызов `Release()` при исключении в `_next` |
| 5.1 | Docs | `Interfaces/*.cs` | — | Medium | Интерфейсы без XML-документации |
| 5.2 | Docs | `CookieStickySessions.cs`, `RoundRobin.cs`, `LeastConnection.cs` | — | Low | Нет документации о требованиях к потокобезопасности |
| 5.3 | Docs | `Errors/*.cs` | — | Low | Несогласованное использование `ServicesAreNullError` и `ServicesAreEmptyError` |

---

## Приоритеты исправлений

### Критично (исправить немедленно)
1. **2.1** — Добавить Health Check или circuit breaker для исключения недоступных узлов
2. **1.2** — Устранить `GetAwaiter().GetResult()` внутри `lock` в `CookieStickySessions`
3. **2.2** — Исправить `Release()` в `CookieStickySessions` — передавать вызов во внутренний балансировщик

### Важно (исправить в ближайшем релизе)
4. **1.3** — Сделать `Stored` инжектируемым (не статическим)
5. **1.4** — Сделать `SyncRoot` в `LeastConnection` и `RoundRobin` экземплярным
6. **3.4** — Сделать `LastIndices` в `RoundRobin` экземплярным
7. **4.1** — Исправить flaky-тесты из-за статического `Stored`

### Желательно
8. **4.5** — Добавить тест на вызов `Release()` при исключении
9. **4.2, 4.3** — Добавить тесты на `CheckExpiry` и конкурентный доступ
10. **5.1** — Добавить XML-документацию к интерфейсам
