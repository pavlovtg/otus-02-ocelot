# System Patterns

## Архитектурные паттерны Ocelot

### Middleware Pipeline
Ocelot построен на ASP.NET Core middleware pipeline. Каждый запрос проходит через цепочку middleware:
`Request → Authentication → Authorization → Rate Limiting → Routing → Load Balancing → Downstream → Response`

### Configuration-driven
Вся конфигурация маршрутов хранится в `ocelot.json`. Паттерн: декларативная конфигурация вместо кода.

### Provider Pattern
Расширяемость через провайдеры:
- `Ocelot.Provider.Consul` — service discovery через Consul
- `Ocelot.Provider.Kubernetes` — service discovery через Kubernetes

### Dependency Injection
Все компоненты регистрируются через DI-контейнер ASP.NET Core.

## Паттерны в коде
- **Repository Pattern** — для работы с конфигурацией
- **Strategy Pattern** — для load balancing алгоритмов
- **Decorator Pattern** — для middleware цепочки
- **Factory Pattern** — для создания downstream запросов

## Соглашения по именованию
- Интерфейсы: `I<Name>` (например, `IRouter`, `ILoadBalancer`)
- Тесты: `<ClassName>Tests` в папке `test/`
- Acceptance тесты: используют `Steps` паттерн
