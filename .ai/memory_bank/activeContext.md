# Active Context

## Текущая задача
Итоговый отчёт по домашней работе сформирован и сохранён.

## Что было сделано
- Создана структура папок `.ai/memory_bank/` и `.ai/prompts/`
- Созданы шаблоны промтов:
  - `.ai/prompts/create-prompt.md` — шаблон создания нового промта
  - `.ai/prompts/commit-and-push.md` — шаблон для commit и push
- Инициализирован memory bank
- Созданы rules для Cline (`.clinerules`)
- Создано описание проекта Ocelot (`homework/description/detailed-description.md`, `short-description.md`)
- Проведён архитектурный анализ проекта Ocelot
- Создан отчёт `homework/description/architecture.md`
- Проведён анализ сильных и слабых сторон архитектуры
- Создан отчёт `homework/description/architecture-strengths-and-weaknesses.md`
- Проведён анализ открытых багов из GitHub Issues (label=bug, state=open)
- Создан отчёт `homework/description/bugs-analysis.md`
- Выполнен скоринг и выбран модуль LoadBalancer (Score=9.40)
- Результат сохранён в `homework/review/choose-module.md`
- Проведён полный code review модуля LoadBalancer
- Создан отчёт `homework/review/load-balancer-review.md` (22 замечания)
- **Создан промт для итогового отчёта** (`homework/prompts/08-review-report.md`)
- **Создан итоговый отчёт** `homework/review-report.md`

## Следующие шаги
- Оформить commit и push результатов

## Активные файлы
- `homework/review-report.md` — итоговый отчёт для сдачи домашней работы
- `homework/review/load-balancer-review.md` — детальный code review
- `homework/prompts/` — все промты (01–08)
- `homework/description/` — описания проекта Ocelot

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
2026-04-27 — Создан итоговый отчёт homework/review-report.md
