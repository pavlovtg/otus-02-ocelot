# Active Context

## Текущая задача
Проведение code review выбранного модуля Ocelot.

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
- Проведён анализ открытых багов из GitHub Issues (label=bug, state=open)
- Создан отчёт `homework/description/bugs-analysis.md` с:
  - 11 открытыми issues с тегом bug
  - Анализом критичности (1 Critical, 6 High, 3 Medium, 1 Low)
  - Распределением по 7 модулям (Routing, LoadBalancer, Aggregation, Middleware, ServiceDiscovery, Authorization, Administration)
  - Выводами о наиболее проблемных модулях
- **Создан промт для выбора модуля code review** (`homework/prompts/06-choose-module.md`)
- **Выполнен скоринг и выбран модуль LoadBalancer** (Score=9.40)
- Результат сохранён в `homework/review/choose-module.md`:
  - Таблица скоринга 6 кандидатов
  - Финальный выбор: **LoadBalancer**
  - Обоснование выбора (3 пункта)
  - Список ключевых файлов для code review

## Следующие шаги
- Провести code review выбранного модуля LoadBalancer
- Создать промт для code review LoadBalancer
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

## Ключевые баги (из GitHub Issues, для быстрого доступа)
- **Critical**: #1252 — HttpContext теряется в DelegatingHandler (регрессия с v15.0.7)
- **High**: #2143, #2191 — спецсимволы в Routing (OData `$query`, query string)
- **High**: #714 — multipart/form-data не перенаправляется (404)
- **High**: #1041, #1513 — LoadBalancer не исключает упавшие узлы (нет Health Check)
- **High**: #2208 — Consul Node.Name может быть DNS-именем
- Наиболее проблемный модуль: **Routing** (3 бага)
- Наиболее критичный модуль: **Middleware** (содержит единственный Critical-баг)

## Последнее обновление
2026-04-27 — Анализ открытых багов Ocelot из GitHub Issues
