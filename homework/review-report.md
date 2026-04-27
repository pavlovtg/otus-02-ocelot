# Итоговый отчёт по code review: Ocelot API Gateway

> **Студент:** Павлов Тимур Геннадьевич  
> **Репозиторий:** https://github.com/pavlovtg/otus-02-ocelot  
> **Дата:** 2026-04-27  

---

## 1. Описание проекта и области анализа

**Ocelot** — API Gateway с открытым исходным кодом на ASP.NET Core (.NET 8+). Реализован как конвейер из 18 middleware: входящий запрос последовательно проходит маршрутизацию, аутентификацию, балансировку нагрузки и проксируется к внутреннему сервису. Поддерживает маршрутизацию, rate limiting, кэширование, service discovery (Consul, Kubernetes, Eureka), агрегацию ответов (BFF), WebSockets.

**Область анализа: модуль LoadBalancer** — компонент, распределяющий запросы между несколькими экземплярами одного сервиса. Реализует паттерн Strategy: четыре алгоритма (`RoundRobin`, `LeastConnection`, `CookieStickySessions`, `NoLoadBalancer`) за единым интерфейсом `ILoadBalancer`. Каждый запрос проходит через `LoadBalancingMiddleware` → `LeaseAsync()` (выбор сервиса) → downstream → `Release()` (освобождение).

Модуль выбран как наиболее проблемный: концентрация архитектурных проблем (статическое состояние, блокирующие вызовы, отсутствие failover) + два открытых бага высокого приоритета (#1041, #1513).

---

## 2. Найденные проблемы

### 2.1 Баги и ошибки выполнения

#### 2.1.1 Накопление устаревших событий при обновлении сессии

**Файл:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`, строки 56–68

```csharp
private void CheckExpiry(StickySession sticky)
{
    lock (Locker)
    {
        if (!Stored.TryGetValue(sticky.Key, out var session) || session.Expiry >= DateTime.UtcNow)
            return;

        Stored.Remove(session.Key);
        _loadBalancer.Release(session.HostAndPort);
    }
}
```

**Проблема:** При каждом обращении к сессии в очередь публикуется новое событие истечения, но старые события не отменяются. Когда они обрабатываются, `CheckExpiry` пытается удалить уже обновлённую сессию. Реализация `InMemoryBus` не отменяет устаревшие сообщения — они накапливаются в памяти.

**Решение:** Проверять актуальность события по времени истечения:

```csharp
if (!Stored.TryGetValue(sticky.Key, out var session)
    || session.Expiry != sticky.Expiry  // событие устарело — сессия уже обновлена
    || session.Expiry >= DateTime.UtcNow)
    return;
```

---

#### 2.1.2 Блокирующий вызов асинхронного кода внутри `lock` — риск деградации

**Файл:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`, строка 87

```csharp
lock (Locker)
{
    var next = _loadBalancer.LeaseAsync(httpContext).GetAwaiter().GetResult();
    // unfortunately the operation must be synchronous
}
```

**Проблема:** `.GetAwaiter().GetResult()` блокирует поток внутри `lock`. При высокой нагрузке потоки пула исчерпываются — система перестаёт обрабатывать новые запросы (threadpool exhaustion). Комментарий в коде подтверждает, что проблема известна.

**Решение:** Заменить `lock` на `SemaphoreSlim` и использовать `await`:

```csharp
private readonly SemaphoreSlim _semaphore = new(1, 1);

public async Task<Response<ServiceHostAndPort>> LeaseAsync(HttpContext httpContext)
{
    await _semaphore.WaitAsync();
    try
    {
        var next = await _loadBalancer.LeaseAsync(httpContext);
        // ...
    }
    finally { _semaphore.Release(); }
}
```

---

#### 2.1.3 Статический словарь сессий — утечка памяти и несовместимость с масштабированием

**Файл:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`, строка 43

```csharp
private static readonly Dictionary<string, StickySession> Stored = new();
// TODO Inject instead of static sharing
```

**Проблема:** Один словарь на весь процесс — разделяется между всеми маршрутами. При горизонтальном масштабировании (несколько pod-ов) сессии не синхронизируются. Словарь никогда не очищается полностью — при большом числе уникальных пользователей растёт неограниченно. Разработчик сам оставил TODO.

**Решение:** Инжектировать хранилище через DI. По умолчанию — `ConcurrentDictionary` (не статический), для масштабирования — `IDistributedCache` (Redis).

---

#### 2.1.4 Статический `SyncRoot` в `LeastConnection` — глобальный bottleneck

**Файл:** `src/Ocelot/LoadBalancer/Balancers/LeastConnection.cs`, строки 14–18

```csharp
private static readonly object SyncRoot = new();
```

**Проблема:** Один объект блокировки на все экземпляры `LeastConnection`. При нескольких маршрутах с этим алгоритмом они конкурируют за один мьютекс — запросы к независимым маршрутам блокируют друг друга. Та же проблема в `RoundRobin.cs` (строки 26–30).

**Решение:** Убрать `static`:

```csharp
private readonly object _syncRoot = new();
```

---

#### 2.1.5 Лишняя аллокация и потенциальный `NullReferenceException` в `NoLoadBalancer`

**Файл:** `src/Ocelot/LoadBalancer/Balancers/NoLoadBalancer.cs`, строки 28–30

```csharp
var service = await Task.FromResult(services.FirstOrDefault());
return new OkResponse<ServiceHostAndPort>(service.HostAndPort);
```

**Проблема:** `await Task.FromResult(x)` — бессмысленная операция, создаёт лишний объект `Task` на каждый запрос (hot path). Если `FirstOrDefault()` вернёт `null`, следующая строка выбросит `NullReferenceException`.

**Решение:**

```csharp
var service = services[0];
if (service == null)
    return new ErrorResponse<ServiceHostAndPort>(new ServicesAreNullError(...));
return new OkResponse<ServiceHostAndPort>(service.HostAndPort);
```

---

### 2.2 Архитектурные ограничения

#### 2.2.1 Нет Health Check — балансировщики не исключают недоступные сервисы

**Файлы:** `RoundRobin.cs` (строка 89), `LeastConnection.cs`, `NoLoadBalancer.cs`

```csharp
// RoundRobin.cs, строка 89:
// TODO Check real health status
while (next?.HostAndPort == null && stop-- > 0) { ... }
```

**Проблема:** Все балансировщики слепо доверяют списку от Service Discovery. Если сервис упал, но SD ещё не зафиксировал это, запросы продолжают идти на недоступный узел. Подтверждается открытыми багами:
- **#1041** — балансировщик направляет запросы на упавший сервис, клиент получает 500
- **#1513** — WebSocket-соединения не переключаются при падении backend

TODO-комментарий в `RoundRobin` прямо указывает на проблему.

**Решение:** Добавить circuit breaker: при ошибке от сервиса временно исключать его из ротации. Минимальный вариант — интеграция с Polly (уже используется в QoS-модуле).

---

#### 2.2.2 `CookieStickySessions.Release()` — пустая реализация нарушает контракт

**Файл:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`, строки 108–110

```csharp
public void Release(ServiceHostAndPort hostAndPort)
{
    // пустое тело
}
```

**Проблема:** `CookieStickySessions` — обёртка над `ILoadBalancer`. Вызов `Release()` поглощается и не передаётся во внутренний балансировщик. Счётчик соединений в `RoundRobin._leasing` никогда не уменьшается — при долгой работе распределение нагрузки становится некорректным.

**Решение:** Одна строка:

```csharp
public void Release(ServiceHostAndPort hostAndPort) => _loadBalancer.Release(hostAndPort);
```

---

#### 2.2.3 `ILoadBalancer.LeaseAsync` принимает `HttpContext` — нарушение SRP

**Файл:** `src/Ocelot/LoadBalancer/Interfaces/ILoadBalancer.cs`, строка 11

```csharp
Task<Response<ServiceHostAndPort>> LeaseAsync(HttpContext httpContext);
```

**Проблема:** Балансировщик — алгоритм выбора сервиса из списка. Он не должен знать о HTTP-контексте. `CookieStickySessions` использует его для чтения cookie, но `RoundRobin`, `LeastConnection`, `NoLoadBalancer` полностью игнорируют. Интерфейс неоднороден, тестирование усложнено.

**Решение:** Выделить необходимые данные в отдельный объект (breaking change — требует major-версии):

```csharp
public record LoadBalancerContext(string ServiceName, string? StickySessionCookie);
Task<Response<ServiceHostAndPort>> LeaseAsync(LoadBalancerContext context);
```

---

#### 2.2.4 Кэш балансировщиков без инвалидации по параметрам

**Файл:** `src/Ocelot/LoadBalancer/LoadBalancerHouse.cs`, строки 30–33

```csharp
return _loadBalancers.TryGetValue(route.LoadBalancerKey, out var loadBalancer) &&
        loadBalancer.Type.Equals(route.LoadBalancerOptions.Type, ...)
    ? new OkResponse<ILoadBalancer>(loadBalancer)
    : GetResponse(route, config);
```

**Проблема:** Балансировщик заменяется только при смене типа. Если изменились параметры (например, `ExpiryInMs` для `CookieStickySessions`), кэшированный экземпляр продолжает работать со старыми значениями. Нет поддержки горячей перезагрузки конфигурации.

**Решение:** Учитывать хэш `LoadBalancerOptions` при сравнении. Добавить `Invalidate(string key)` в `ILoadBalancerHouse`.

---

#### 2.2.5 WebSocket-соединения некорректно учитываются в `LeastConnection`

**Файл:** `src/Ocelot/WebSockets/WebSocketsProxyMiddleware.cs`, строки 138–143

**Проблема:** WebSocket-соединения могут длиться часами. За это время `LeastConnection` накапливает большой счётчик для одного сервера и перестаёт направлять на него новые соединения, хотя реальная нагрузка минимальна. Связано с багом #1513.

**Решение:** Для WebSocket использовать отдельную стратегию балансировки или добавить поддержку долгоживущих соединений в интерфейс.

---

### 2.3 Читаемость и удобство использования

#### 2.3.1 Двойная блокировка в `CookieStickySessions.Update()`

**Файл:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`, строки 77–106

```csharp
public Task<Response<ServiceHostAndPort>> LeaseAsync(HttpContext httpContext)
{
    lock (Locker)          // первый захват
    {
        Update(key, ss);   // вызов Update
    }
}

protected void Update(string key, StickySession value)
{
    lock (Locker)          // второй захват того же объекта
    {
        Stored[key] = value;
    }
}
```

**Проблема:** `Update()` захватывает блокировку, которая уже захвачена в `LeaseAsync`. В .NET это работает (реентрантность), но запутывает читателя. `Update()` объявлен `protected`, хотя класс не предназначен для наследования.

**Решение:** Убрать `lock` из `Update()`, сделать метод `private`.

---

#### 2.3.2 Дублирование метода `Update(ref Lease, bool)` в двух классах

**Файлы:** `RoundRobin.cs` (строки 108–113), `LeastConnection.cs` (строки 67–73)

```csharp
// Идентичный код в обоих файлах:
private int Update(ref Lease item, bool increase)
{
    var index = _leasing.IndexOf(item);
    _ = increase ? item.Connections++ : item.Connections--;
    _leasing[index] = item;
    return index;
}
```

**Проблема:** Полное дублирование — при исправлении бага в одном классе легко забыть про второй.

**Решение:** Вынести в базовый класс `LoadBalancerBase` или метод-расширение для `List<Lease>`.

---

#### 2.3.3 TODO-комментарии без привязки к задачам

| Файл | Строка | Комментарий |
|------|--------|-------------|
| `CookieStickySessions.cs` | 43 | `// TODO Inject instead of static sharing` |
| `CookieStickySessions.cs` | 58 | `// TODO Get test coverage for this` |
| `RoundRobin.cs` | 89 | `// TODO Check real health status` |
| `LeastConnection.cs` | 42 | `//todo - maybe this should be moved somewhere else...?` |
| `LoadBalancerHouse.cs` | 52 | `// TODO TryAdd ?` |

**Проблема:** TODO без ссылки на задачу — технический долг, который не отслеживается и живёт годами.

**Решение:** Оформить каждый TODO как GitHub Issue, добавить ссылку в комментарий: `// TODO: #XXXX`.

---

#### 2.3.4 Статический `LastIndices` в `RoundRobin` — скрытое разделяемое состояние

**Файл:** `src/Ocelot/LoadBalancer/Balancers/RoundRobin.cs`, строка 25

```csharp
private static readonly Dictionary<string, int> LastIndices = new();
```

**Проблема:** Разделяется между всеми экземплярами `RoundRobin`. При совпадении имён сервисов в разных маршрутах индексы конфликтуют. Словарь никогда не очищается — при динамической конфигурации записи накапливаются.

**Решение:** Заменить на `private int _lastIndex` — экземплярное поле.

---

### 2.4 Тесты

#### 2.4.1 Статический `Stored` не очищается между тестами — нестабильные тесты

**Файл:** `test/Ocelot.UnitTests/LoadBalancer/CookieStickySessionsTests.cs`

**Проблема:** `CookieStickySessions.Stored` — статический словарь. Данные из одного теста остаются для следующего. Порядок выполнения тестов в xUnit не гарантирован — тесты могут случайно проходить или падать (flaky tests).

**Решение:** Добавить очистку `Stored` в `Dispose()` тестового класса. Долгосрочно — сделать `Stored` инжектируемым.

---

#### 2.4.2 Нет тестов на логику истечения сессий

**Файл:** `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`, строка 58

```csharp
private void CheckExpiry(StickySession sticky)
{
    // TODO Get test coverage for this
```

**Проблема:** Ключевая логика истечения сессий не покрыта тестами. Существующий тест проверяет только факт вызова `Release()`, но не проверяет корректность удаления/сохранения сессий.

**Решение:** Добавить параметризованные тесты: истёкшая сессия → удаляется; живая сессия → не удаляется; отсутствующий ключ → нет ошибки.

---

#### 2.4.3 Нет теста на конкурентный доступ к `CookieStickySessions`

**Файл:** `test/Ocelot.UnitTests/LoadBalancer/CookieStickySessionsTests.cs`

**Проблема:** В `LeastConnectionTests` и `RoundRobinTests` есть тесты на параллельный доступ, а для `CookieStickySessions` — нет. При этом именно этот класс наиболее уязвим к race condition из-за статических блокировок.

**Решение:** Добавить тест с 50+ параллельными запросами, проверяющий отсутствие ошибок и корректность привязки сессии.

---

#### 2.4.4 Тест обращается к приватным методам через Reflection

**Файл:** `test/Ocelot.UnitTests/LoadBalancer/RoundRobinTests.cs`, строки 167–191

```csharp
var method = typeof(RoundRobin).GetMethod(nameof(TryScanNext),
    BindingFlags.Instance | BindingFlags.NonPublic);
var field = typeof(RoundRobin).GetField("LastIndices",
    BindingFlags.Static | BindingFlags.NonPublic);
```

**Проблема:** Тест сломается при переименовании метода или поля. Проверяет детали реализации, а не публичный контракт.

**Решение:** Тестировать поведение через публичный `LeaseAsync`.

---

#### 2.4.5 Нет проверки вызова `Release()` при исключении в следующем middleware

**Файл:** `test/Ocelot.UnitTests/LoadBalancer/LoadBalancerMiddlewareTests.cs`

```csharp
// LoadBalancingMiddleware.cs:
try { await _next.Invoke(httpContext); }
finally { loadBalancer.Data.Release(hostAndPort.Data); } // должен вызываться всегда
```

**Проблема:** Тест `ShouldNot_LogDebug_WhenNextMiddlewareThrownException` проверяет проброс исключения, но не проверяет, что `Release()` вызывается в `finally`. Если нет — счётчики `LeastConnection` растут бесконечно.

**Решение:** Добавить `Verify(x => x.Release(...), Times.Once)` в тест с исключением.

---

### 2.5 Документация

#### 2.5.1 Интерфейсы без XML-документации

**Файлы:** `ILoadBalancer.cs`, `ILoadBalancerHouse.cs`, `ILoadBalancerFactory.cs`, `ILoadBalancerCreator.cs`

```csharp
public interface ILoadBalancer
{
    Task<Response<ServiceHostAndPort>> LeaseAsync(HttpContext httpContext); // нет документации
    void Release(ServiceHostAndPort hostAndPort);
    string Type { get; }
}
```

**Проблема:** Разработчик, реализующий кастомный балансировщик через `AddCustomLoadBalancer`, не знает: что делать при отсутствии сервисов, когда вызывается `Release`, должна ли реализация быть потокобезопасной.

**Решение:** Добавить XML-документацию к каждому методу с описанием контракта и требований к потокобезопасности.

---

#### 2.5.2 Нет документации о требованиях к потокобезопасности

**Файлы:** `CookieStickySessions.cs`, `RoundRobin.cs`, `LeastConnection.cs`

**Проблема:** Все балансировщики используют блокировки, но нигде не документируют это. Один экземпляр обслуживает все параллельные запросы к маршруту — реализация кастомного балансировщика без этого знания может быть небезопасной.

**Решение:** Добавить в `ILoadBalancer` комментарий: _«Реализации должны быть потокобезопасными»_.

---

#### 2.5.3 Несогласованное использование типов ошибок

**Файлы:** `ServicesAreNullError.cs`, `ServicesAreEmptyError.cs`

```csharp
// LeastConnection.cs — ServicesAreNullError используется для пустого списка:
return new ErrorResponse<ServiceHostAndPort>(new ServicesAreNullError(
    $"Services were null/empty in {Type}..."));
```

**Проблема:** `ServicesAreNullError` используется в `LeastConnection` для пустого списка, хотя для этого есть `ServicesAreEmptyError`. В `RoundRobin` — наоборот. Несогласованность затрудняет диагностику.

**Решение:** `ServicesAreNullError` — когда список `null`; `ServicesAreEmptyError` — когда список пуст.

---

## 3. Предложения улучшений

### 🔴 Критично

1. **Добавить Health Check / circuit breaker** — интегрировать Polly для исключения недоступных узлов из ротации (закроет баги #1041, #1513)
2. **Исправить `Release()` в `CookieStickySessions`** — добавить `_loadBalancer.Release(hostAndPort)` (одна строка, высокий эффект)
3. **Устранить `.GetAwaiter().GetResult()` внутри `lock`** — заменить на `SemaphoreSlim` + `await`

### 🟡 Важно

4. **Сделать `Stored` инжектируемым** — `IDistributedCache` вместо статического словаря
5. **Сделать `SyncRoot` и `LastIndices` экземплярными** — убрать `static` в `LeastConnection`, `RoundRobin`
6. **Исправить нестабильные тесты** — очищать `Stored` в `Dispose()` тестового класса

### 🟢 Желательно

7. Добавить тесты на `CheckExpiry` и конкурентный доступ
8. Добавить XML-документацию к интерфейсам
9. Оформить TODO-комментарии как GitHub Issues

---

## 4. Итоговый вывод

Модуль LoadBalancer реализует базовую функциональность балансировки, но содержит накопленный технический долг с реальными production-рисками.

**Критические находки:**
- Балансировщики не исключают упавшие сервисы — два открытых бага (#1041, #1513) с 2020–2021 года
- `CookieStickySessions.Release()` — пустая реализация, нарушающая контракт
- Статический `Stored` — несовместимость с горизонтальным масштабированием

**Системная проблема:** широкое использование `static`-полей (`Stored`, `SyncRoot`, `LastIndices`) делает модуль несовместимым с горизонтальным масштабированием и создаёт скрытые зависимости между независимыми маршрутами.

**Приоритеты:** исправить `Release()` (одна строка), устранить блокирующий вызов, добавить Health Check. Остальное — технический долг для планового рефакторинга.

---

## 5. Список использованных промптов

| # | Промт | Назначение |
|---|-------|-----------|
| 01 | [01-init-ai.md](homework/prompts/01-init-ai.md) | Инициализация AI-ассистента, настройка контекста |
| 02 | [02-project-description.md](homework/prompts/02-project-description.md) | Описание проекта Ocelot |
| 03 | [03-analyze-architecture.md](homework/prompts/03-analyze-architecture.md) | Архитектурный анализ: паттерны, модули, C4-диаграммы |
| 04 | [04-architecture-strengths-weaknesses.md](homework/prompts/04-architecture-strengths-weaknesses.md) | Сильные и слабые стороны архитектуры |
| 05 | [05-analyze-bugs.md](homework/prompts/05-analyze-bugs.md) | Анализ открытых багов из GitHub Issues |
| 06 | [06-choose-module.md](homework/prompts/06-choose-module.md) | Выбор модуля для code review (скоринг) |
| 07 | [07-code-review.md](homework/prompts/07-code-review.md) | Code review модуля LoadBalancer |
| 08 | [08-review-report.md](homework/prompts/08-review-report.md) | Формирование итогового отчёта |
