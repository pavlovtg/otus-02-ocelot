# Active Context

## Текущая задача
Code review модуля LoadBalancer завершён.

## Что было сделано
- Создана структура папок `.ai/memory_bank/` и `.ai/prompts/`
- Созданы шаблоны промтов:
  - `.ai/prompts/create-prompt.md` — шаблон создания нового промта
  - `.ai/prompts/commit-and-push.md` — шаблон для commit и push
- Инициализирован memory bank
- Созданы rules для Cline (`.clinerules`)
- Создано описание проекта Ocelot (`homework/description/detailed-description.md`, `short-description.md`)
- Проведён архитектурный анализ проекта Ocelot
- Создан отчёт `homework/description/architecture.md` с:
  - Описанием 8 архитектурных паттернов (Middleware Pipeline, Strategy, Factory, Repository, Builder, Decorator, Observer, BFF)
  - Картой модулей (22 модуля с назначением, интерфейсами и зависимостями)
  - C4-диаграммами в формате PlantUML (Context, Container, Component)
- Проведён анализ сильных и слабых сторон архитектуры
- Создан отчёт `homework/description/architecture-strengths-and-weaknesses.md`
- Проведён анализ открытых багов из GitHub Issues (label=bug, state=open)
- Создан отчёт `homework/description/bugs-analysis.md`
- Создан промт для выбора модуля code review (`homework/prompts/06-choose-module.md`)
- Выполнен скоринг и выбран модуль LoadBalancer (Score=9.40)
- Результат сохранён в `homework/review/choose-module.md`
- **Создан промт для code review LoadBalancer** (`homework/prompts/07-code-review.md`)
- **Проведён полный code review модуля LoadBalancer**
- **Создан отчёт** `homework/review/load-balancer-review.md` с:
  - 5 багами и ошибками выполнения
  - 5 архитектурными ограничениями
  - 4 замечаниями по читаемости
  - 5 замечаниями по тестам
  - 3 замечаниями по документации
  - Итоговой таблицей из 22 замечаний с приоритетами

## Следующие шаги
- Оформить commit и push результатов code review
- Подготовить финальный отчёт для учебного задания

## Активные файлы
- `homework/review/load-balancer-review.md` — итоговый отчёт code review
- `homework/prompts/` — промты для учебного задания
- `homework/description/` — описания проекта Ocelot
- `.ai/memory_bank/` — контекст для AI-ассистента

## Ключевые архитектурные факты (для быстрого доступа)
- Ocelot = конвейер из 18 ASP.NET Core middleware
- Точка входа: `AddOcelot()` + `UseOcelot()`
- Конфигурация: `ocelot.json` → `FileConfiguration` → `IInternalConfiguration`
- Расширяемость: `IOcelotBuilder` (fluent API), `ILoadBalancer`, `IServiceDiscoveryProvider`, `IResponseAggregator`
- Провайдеры: `Ocelot.Provider.Consul`, `Ocelot.Provider.Kubernetes`
- QoS: отдельный пакет `Ocelot.QualityOfService.Polly`

## Ключевые проблемы LoadBalancer (из code review)
- `CookieStickySessions.Stored` — static Dictionary, не масштабируется горизонтально (утечка памяти)
- `GetAwaiter().GetResult()` внутри `lock` в `CookieStickySessions` — риск дедлока
- `CookieStickySessions.Release()` — пустая реализация, не передаёт вызов во внутренний балансировщик
- `LeastConnection.SyncRoot` и `RoundRobin.SyncRoot` — статические локеры, глобальный bottleneck
- `RoundRobin.LastIndices` — статический словарь, скрытое разделяемое состояние
- Нет Health Check — балансировщики не исключают упавшие узлы (#1041, #1513)
- Flaky-тесты из-за статического `Stored` в `CookieStickySessionsTests`

## Последнее обновление
2026-04-27 — Завершён code review модуля LoadBalancer
