# Active Context

## Текущая задача
Архитектурный анализ проекта Ocelot с построением C4-диаграмм.

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

## Следующие шаги
- Выбрать модуль для code review
- Провести code review выбранного модуля Ocelot с помощью AI
- Оформить итоговый отчёт по code review

## Активные файлы
- `homework/prompts/` — промты для учебного задания
- `homework/description/` — описания проекта Ocelot
- `.ai/memory_bank/` — контекст для AI-ассистента
- `.ai/prompts/` — шаблоны промтов

## Ключевые архитектурные факты (для быстрого доступа)
- Ocelot = конвейер из 18 ASP.NET Core middleware
- Точка входа: `AddOcelot()` + `UseOcelot()`
- Конфигурация: `ocelot.json` → `FileConfiguration` → `IInternalConfiguration`
- Расширяемость: `IOcelotBuilder` (fluent API), `ILoadBalancer`, `IServiceDiscoveryProvider`, `IResponseAggregator`
- Провайдеры: `Ocelot.Provider.Consul`, `Ocelot.Provider.Kubernetes`
- QoS: отдельный пакет `Ocelot.QualityOfService.Polly`

## Последнее обновление
2026-04-27 — Архитектурный анализ Ocelot, C4-диаграммы (Context, Container, Component)
