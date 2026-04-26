# Project Brief

## Название проекта
Ocelot — API Gateway на .NET

## Репозиторий
- Fork: https://github.com/pavlovtg/otus-02-ocelot
- Upstream: https://github.com/ThreeMammals/Ocelot

## Цель использования репозитория
Учебное задание OTUS: code review opensource проекта с использованием AI-ассистента (Cline).

## Описание проекта
Ocelot — это API Gateway, реализованный на платформе .NET. Предоставляет функциональность:
- Маршрутизация запросов
- Аутентификация и авторизация
- Rate limiting
- Load balancing
- Service discovery (Consul, Kubernetes)
- Кэширование
- Трассировка (OpenTracing)
- WebSockets

## Структура репозитория
- `src/` — исходный код Ocelot и провайдеров
- `test/` — юнит и acceptance тесты
- `samples/` — примеры использования
- `docs/` — документация
- `homework/` — материалы учебного задания
- `.ai/` — конфигурация AI-ассистента

## Технологии
- Язык: C#
- Платформа: .NET
- Тесты: xUnit
- CI/CD: GitHub Actions
