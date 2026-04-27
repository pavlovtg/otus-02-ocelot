# Результат выбора модуля для code review

## Таблица скоринга кандидатов

| Модуль | F (0.40) | A (0.30) | B (0.30) | Итоговый Score | Обоснование |
|--------|----------|----------|----------|----------------|-------------|
| **LoadBalancer** | 10 | 10 | 8 | **9.40** | Центральный модуль pipeline; 3 критичных архитектурных проблемы (static state, дедлок, глобальная блокировка); 2 High-бага об отсутствии Health Check |
| Middleware | 10 | 7 | 8 | 8.50 | Критический баг #1252 (DelegatingHandler), но проблемы размыты между подсистемами; HttpContext.Items — общая проблема всех middleware |
| Routing | 10 | 6 | 7 | 7.90 | 3 High-бага (OData, query string, multipart), но архитектурных проблем меньше — основные уязвимости в парсинге шаблонов |
| ServiceDiscovery | 8 | 7 | 7 | 7.40 | Важный модуль, но не все маршруты используют; PollConsul имеет дедлок, фабрика поддерживает только один делегат |
| RateLimiting | 8 | 9 | 0 | 5.90 | Сильные архитектурные проблемы (глобальный static мьютекс), но нет открытых багов — меньше практической пользы от review |
| Aggregation | 6 | 6 | 4 | 5.40 | BFF-функциональность опциональна; 2 Medium-бага; JSON-агрегатор конкатенирует строками |

## Финальный выбор: **LoadBalancer**

## Обоснование выбора

Модуль **LoadBalancer** набрал максимальный комплексный скор **9.40** и является оптимальным выбором для глубокого code review по трём причинам.

**Во-первых**, это центральный компонент pipeline — каждый запрос к downstream-сервису проходит через балансировщик при включённом `LoadBalancerOptions`. Модуль реализует паттерн Strategy (`ILoadBalancer` с реализациями `RoundRobin`, `LeastConnection`, `CookieStickySessions`, `NoLoadBalancer`) и напрямую влияет на надёжность и производительность всего шлюза.

**Во-вторых**, модуль содержит концентрированные и тяжёлые архитектурные проблемы: статический словарь `CookieStickySessions.Stored` без изоляции по маршрутам и с невозможностью горизонтального масштабирования; классический антипаттерн `GetAwaiter().GetResult()` внутри `lock` с риском дедлока в ASP.NET Core; глобальное состояние балансировщиков (`RoundRobin`, `LeastConnection`), не синхронизируемое между инстансами Ocelot. Все эти проблемы локализованы в одном модуле, что позволяет провести детальный анализ и предложить конкретные решения.

**В-третьих**, оба открытых High-бага (#1041 и #1513) указывают на одну системную проблему — отсутствие механизма Health Check для исключения нездоровых узлов из ротации. Это создаёт синергию: code review может сразу предложить архитектурное решение, которое закроет и архитектурные уязвимости, и открытые баги. Исправление даст прямой production-эффект: отказоустойчивость шлюза при падении downstream-сервисов и корректный failover WebSocket-соединений.

## Список ключевых файлов для code review

### Основные файлы модуля
- `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs` — static state, дедлок, истечение сессий
- `src/Ocelot/LoadBalancer/Balancers/LeastConnection.cs` — алгоритм + TODO
- `src/Ocelot/LoadBalancer/Balancers/RoundRobin.cs` — алгоритм, состояние не распределено
- `src/Ocelot/LoadBalancer/Balancers/NoLoadBalancer.cs` — fallback-реализация
- `src/Ocelot/LoadBalancer/LoadBalancerHouse.cs` — хранилище балансировщиков, TODO TryAdd
- `src/Ocelot/LoadBalancer/LoadBalancerFactory.cs` — фабрика создания балансировщиков
- `src/Ocelot/LoadBalancer/ILoadBalancer.cs` — контракт
- `src/Ocelot/LoadBalancer/ILoadBalancerHouse.cs` — контракт хранилища

### Связанные файлы
- `src/Ocelot/Middleware/LoadBalancingMiddleware.cs` — точка входа в pipeline
- `src/Ocelot/ServiceDiscovery/ServiceDiscoveryProviderFactory.cs` — поставщик списка узлов для балансировки
- `src/Ocelot/WebSockets/` — WebSocket-проксирование, связанное с багом #1513

### Тесты
- `test/Ocelot.UnitTests/LoadBalancer/` — unit-тесты балансировщиков
- `test/Ocelot.AcceptanceTests/LoadBalancer/` — acceptance-тесты
