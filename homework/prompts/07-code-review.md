## Роль
Ты — опытный разработчик на C#/.NET, проводящий детальное code review.

## Контекст
Проект Ocelot — API Gateway на ASP.NET Core. Модуль LoadBalancer реализует паттерн Strategy (`ILoadBalancer` с реализациями `RoundRobin`, `LeastConnection`, `CookieStickySessions`, `NoLoadBalancer`). Модуль находится на критическом пути pipeline — каждый запрос к downstream-сервису проходит через балансировщик при включённом `LoadBalancerOptions`.

## Задача
Проведи code review указанных файлов модуля LoadBalancer. Выяви баги, архитектурные ограничения, проблемы читаемости, недостатки тестов и документации.

## Входные данные
- Основные файлы модуля:
  - `src/Ocelot/LoadBalancer/Balancers/CookieStickySessions.cs`
  - `src/Ocelot/LoadBalancer/Balancers/LeastConnection.cs`
  - `src/Ocelot/LoadBalancer/Balancers/RoundRobin.cs`
  - `src/Ocelot/LoadBalancer/Balancers/NoLoadBalancer.cs`
  - `src/Ocelot/LoadBalancer/LoadBalancerHouse.cs`
  - `src/Ocelot/LoadBalancer/LoadBalancerFactory.cs`
  - `src/Ocelot/LoadBalancer/ILoadBalancer.cs`
  - `src/Ocelot/LoadBalancer/ILoadBalancerHouse.cs`
- Связанные файлы:
  - `src/Ocelot/Middleware/LoadBalancingMiddleware.cs`
  - `src/Ocelot/ServiceDiscovery/ServiceDiscoveryProviderFactory.cs`
  - `src/Ocelot/WebSockets/`
- Тесты:
  - `test/Ocelot.UnitTests/LoadBalancer/`
  - `test/Ocelot.AcceptanceTests/LoadBalancer/`
- Дополнительно: открытые баги #1041, #1513 (отсутствие Health Check)
- Описание проекта, архитектура и открытые баги в `homework/description/`

## Ожидаемый результат
Структурированный отчёт по разделам:
1. Баги и ошибки выполнения
2. Архитектурные ограничения
3. Читаемость и удобство использования
4. Тесты
5. Документация

Каждое замечание должно содержать:
- название файла и номер строки / ссылку на код;
- фрагмент кода в котороме находится проблема;
- объяснение проблемы понятным языком (без жаргона, с минимальным контекстом);
- конкретное предложение по улучшению.

Результат ревью сохранен в файл `homework/review/load-balancer-review.md`.

## Ограничения
- Проверяющий — опытный инженер, но может не знать Ocelot или .NET: формулируй чётко, без жаргона, с необходимым контекстом.
- Не включать в результаты ревью код из других модулей (вне списка выше).
- Не предлагать изменения публичного API без крайней необходимости.
- Анализировать весь репозиторий при необходимости, но включать в отчёт только файлы из списка.
