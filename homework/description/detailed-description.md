# Подробное описание проекта Ocelot

## Краткое описание

Ocelot — это API Gateway с открытым исходным кодом, реализованный на платформе .NET. Проект предназначен для использования в микросервисных (сервис-ориентированных) архитектурах и обеспечивает единую точку входа в систему. Ocelot работает с любым сервисом, поддерживающим HTTP(S), и запускается на любой платформе, поддерживаемой ASP.NET Core.

---

## Цели, задачи и область применения

### Цели проекта
- Предоставить разработчикам на .NET готовое решение для построения API Gateway без необходимости писать собственную инфраструктуру маршрутизации.
- Обеспечить единую точку входа для клиентов в микросервисной архитектуре.
- Упростить интеграцию с популярными системами обнаружения сервисов и аутентификации.

### Задачи, решаемые проектом
- Маршрутизация входящих HTTP-запросов к нужным downstream-сервисам.
- Агрегация ответов от нескольких сервисов в один ответ (BFF-паттерн).
- Защита API: аутентификация, авторизация, ограничение частоты запросов.
- Балансировка нагрузки между экземплярами downstream-сервисов.
- Обнаружение сервисов через Consul, Kubernetes, Eureka, Service Fabric.
- Кэширование ответов, трассировка запросов, трансформация заголовков и методов.

### Область применения
- Микросервисные архитектуры на .NET, где требуется единая точка входа.
- Backend-for-Frontend (BFF) сценарии с агрегацией запросов.
- Системы с динамическим масштабированием сервисов и service discovery.
- Облачные и контейнерные среды (Docker, Kubernetes, Azure Service Fabric).

---

## Основные возможности

| Группа | Возможности |
|--------|-------------|
| **Первичные** | Конфигурация маршрутов, маршрутизация запросов |
| **Solid** | Кэширование, Delegating Handlers, Quality of Service (QoS), Rate Limiting |
| **Hybrid** | Администрирование, Агрегация (BFF), Аутентификация, Dependency Injection, Load Balancer |
| **Family** | Конфигурация, Маршрутизация, Логирование, Трансформации, Service Discovery |

### Детальный список возможностей

- **Маршрутизация** — гибкая настройка маршрутов с плейсхолдерами, приоритетами, catch-all, маршрутизацией по заголовкам и хосту.
- **Конфигурация** — JSON-конфигурация (`ocelot.json`), поддержка нескольких файлов, слияние конфигураций, хранение в Consul KV, горячая перезагрузка.
- **Аутентификация и авторизация** — интеграция с ASP.NET Core Identity, Bearer-токены, проверка claims.
- **Rate Limiting** — ограничение частоты запросов на уровне маршрута или глобально.
- **Load Balancing** — встроенные алгоритмы: `RoundRobin`, `LeastConnection`, `CookieStickySessions`, `NoLoadBalancer`; поддержка кастомных балансировщиков.
- **Service Discovery** — интеграция с Consul, Kubernetes, Netflix Eureka, Azure Service Fabric; динамическая маршрутизация.
- **Кэширование** — кэширование ответов downstream-сервисов.
- **Quality of Service** — политики повторных попыток через библиотеку Polly, настройка таймаутов.
- **Агрегация запросов** — объединение ответов нескольких маршрутов в один JSON-ответ (BFF).
- **Трансформации** — трансформация заголовков (upstream/downstream), claims, HTTP-методов.
- **WebSockets** — поддержка проксирования WebSocket-соединений.
- **Трассировка** — интеграция с OpenTracing.
- **Администрирование** — REST API для управления конфигурацией в runtime.
- **Metadata** — произвольные метаданные на уровне маршрута для расширения функциональности.
- **Delegating Handlers** — кастомные обработчики в pipeline запроса.
- **Middleware Injection** — внедрение собственных middleware в pipeline Ocelot.

---

## Архитектура и ключевые модули

Ocelot построен как набор ASP.NET Core middleware, выстроенных в определённом порядке. Входящий запрос проходит через pipeline, трансформируется согласно конфигурации, пересылается в downstream-сервис, а ответ возвращается клиенту.

### Ключевые модули (`src/Ocelot/`)

| Модуль | Назначение |
|--------|-----------|
| `Middleware/` | Основной pipeline Ocelot, точка входа `UseOcelot()` |
| `Configuration/` | Загрузка, валидация и хранение конфигурации (`ocelot.json`) |
| `DependencyInjection/` | Регистрация сервисов Ocelot в DI-контейнере ASP.NET Core |
| `DownstreamRouteFinder/` | Поиск подходящего маршрута для входящего запроса |
| `DownstreamUrlCreator/` | Формирование URL для downstream-запроса |
| `Requester/` | Отправка HTTP-запроса к downstream-сервису (`HttpMessageInvoker`) |
| `Responder/` | Маппинг ответа downstream-сервиса в `HttpResponse` |
| `LoadBalancer/` | Балансировка нагрузки между экземплярами сервиса |
| `ServiceDiscovery/` | Обнаружение сервисов (Consul, Kubernetes, Eureka, Service Fabric) |
| `Authentication/` | Аутентификация запросов (Bearer-токены, ASP.NET Core Identity) |
| `Authorization/` | Авторизация на основе claims |
| `RateLimiting/` | Ограничение частоты запросов |
| `Cache/` | Кэширование ответов |
| `QualityOfService/` | Политики QoS (таймауты, retry через Polly) |
| `Multiplexer/` | Агрегация запросов (BFF, бывший Request Aggregation) |
| `Headers/` | Трансформация заголовков |
| `Claims/` | Трансформация claims |
| `WebSockets/` | Поддержка WebSocket-соединений |
| `Logging/` | Логирование, обработка ошибок, трассировка |
| `Administration/` | REST API администрирования |
| `Metadata/` | Метаданные маршрутов |
| `Security/` | Управление IP-фильтрацией (allow/block lists) |
| `Infrastructure/` | Вспомогательные утилиты и базовые абстракции |
| `Request/` | Построение downstream HTTP-запроса |
| `QueryStrings/` | Обработка query string параметров |
| `RequestId/` | Управление идентификаторами запросов |

### Провайдеры (отдельные пакеты)

| Пакет | Назначение |
|-------|-----------|
| `src/Ocelot.Provider.Consul/` | Интеграция с Consul (service discovery + KV store) |
| `src/Ocelot.Provider.Kubernetes/` | Интеграция с Kubernetes |

---

## Технологический стек

| Компонент | Технология |
|-----------|-----------|
| Язык | C# |
| Платформа | .NET 8 (LTS), .NET 9 (STS), .NET 10 (LTS) |
| Веб-фреймворк | ASP.NET Core (Minimal API) |
| Тесты | xUnit |
| QoS / Retry | Polly (`Ocelot.QualityOfService.Polly`) |
| Service Discovery | Consul, Kubernetes, Netflix Eureka (Steeltoe), Azure Service Fabric |
| Трассировка | OpenTracing |
| Документация | reStructuredText (Read the Docs) |
| CI/CD | GitHub Actions |
| Покрытие кода | Coveralls, Codecov |
| Пакетный менеджер | NuGet |
| Лицензия | MIT |

---

## Структура репозитория

```
/
├── src/
│   ├── Ocelot/                        # Основной проект Ocelot (API Gateway)
│   ├── Ocelot.Provider.Consul/        # Провайдер Consul
│   └── Ocelot.Provider.Kubernetes/    # Провайдер Kubernetes
├── test/
│   ├── Ocelot.UnitTests/              # Юнит-тесты
│   ├── Ocelot.AcceptanceTests/        # Acceptance-тесты
│   ├── Ocelot.Benchmarks/             # Бенчмарки производительности
│   └── Ocelot.ManualTest/             # Ручные тесты
├── testing/                           # Вспомогательные утилиты для тестов
├── samples/                           # Примеры использования
│   ├── Basic/                         # Базовый пример
│   ├── Configuration/                 # Пример с несколькими файлами конфигурации
│   ├── Kubernetes/                    # Пример с Kubernetes
│   ├── ServiceDiscovery/              # Пример с service discovery
│   ├── Eureka/                        # Пример с Netflix Eureka
│   ├── GraphQL/                       # Пример с GraphQL
│   ├── OpenTracing/                   # Пример с трассировкой
│   ├── Metadata/                      # Пример с метаданными
│   ├── ServiceFabric/                 # Пример с Azure Service Fabric
│   └── Web/                           # Веб-пример
├── docs/                              # Документация (reStructuredText)
│   ├── introduction/                  # Введение и быстрый старт
│   ├── features/                      # Документация по каждой фиче
│   └── building/                      # Инструкции по сборке и разработке
├── docker/                            # Docker-файлы для сборки и деплоя
├── homework/                          # Материалы учебного задания OTUS
│   ├── prompts/                       # Промты для AI-ассистента
│   └── description/                   # Описания проекта (этот файл)
└── .ai/                               # Конфигурация AI-ассистента (Cline)
    ├── memory_bank/                   # Memory bank для контекста
    └── prompts/                       # Шаблоны промтов
```
