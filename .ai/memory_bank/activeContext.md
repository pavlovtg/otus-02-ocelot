# Active Context

## Текущая задача
Анализ сильных и слабых сторон архитектуры Ocelot.

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
- Создан отчёт `homework/description/architecture-strengths-and-weaknesses.md` с:
  - 7 сильными сторонами (Middleware Pipeline, OCP через DI, Strategy+Factory, Fluent API, разделение конфигурации, Change Tracking, изоляция провайдеров)
  - 10 слабыми сторонами и уязвимостями (HttpContext.Items как шина, static state в CookieStickySessions, GetAwaiter().GetResult() в lock, глобальный ProcessLocker, StringBuilder JSON, один делегат SD, захват IServiceProvider, static поля WatchKube, TODO-долг)
  - 6 зонами технического долга
  - 10 приоритизированными рекомендациями (2 критичных, 4 важных, 4 желательных)

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

## Ключевые проблемы архитектуры (для быстрого доступа)
- `HttpContext.Items` со строковыми ключами — неявная шина данных между middleware
- `CookieStickySessions.Stored` — static Dictionary, не масштабируется горизонтально
- `GetAwaiter().GetResult()` внутри `lock` в `CookieStickySessions` и `PollConsul` — риск дедлока
- `RateLimiting.ProcessLocker` — static глобальный мьютекс, узкое место при нагрузке
- `ServiceDiscoveryProviderFactory` — поддерживает только один `ServiceDiscoveryFinderDelegate`
- Newtonsoft.Json вместо System.Text.Json в .NET 8+ проекте

## Последнее обновление
2026-04-27 — Анализ сильных и слабых сторон архитектуры Ocelot
