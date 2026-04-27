# Анализ открытых багов Ocelot

> Источник: [https://github.com/ThreeMammals/Ocelot/issues](https://github.com/ThreeMammals/Ocelot/issues)  
> Фильтр: `state=open`, `label=bug`  
> Дата анализа: 2026-04-27  
> Всего найдено: 11 открытых issues с тегом `bug`

---

## Сводная таблица по модулям

| Модуль / Компонент | Количество багов | Критичность (макс.) |
|--------------------|-----------------|---------------------|
| Routing            | 3               | High                |
| LoadBalancer       | 2               | High                |
| Aggregation        | 2               | Medium              |
| Middleware         | 2               | Critical            |
| ServiceDiscovery   | 1               | High                |
| Authorization      | 1               | Medium              |
| Administration     | 1               | Low                 |

---

## Детальный анализ по модулям

### 🔴 Middleware (2 бага, максимальная критичность: Critical)

| Issue # / Заголовок | Ссылка | Критичность | Модуль | Краткое описание |
|---------------------|--------|-------------|--------|-----------------|
| #1252 — HttpContext lost between HttpClientWrapper and DelegatingHandler | [#1252](https://github.com/ThreeMammals/Ocelot/issues/1252) | **Critical** | Middleware / DelegatingHandler | После версии 15.0.7 кастомный `DelegatingHandler` получает пустой `HttpContext` — `IHttpContextAccessor.HttpContext` не содержит данных пользователя (claims). Регрессия, сломавшая существующую функциональность. |
| #1687 — HTTP status 499 seems inappropriate when gateway times out | [#1687](https://github.com/ThreeMammals/Ocelot/issues/1687) | **Medium** | Middleware / QoS | При таймауте ожидания ответа от downstream-сервиса Ocelot возвращает клиенту HTTP 499 (нестандартный код Nginx) вместо корректного HTTP 504 Gateway Timeout. Вводит в заблуждение клиентов и мониторинг. |

---

### 🟠 Routing (3 бага, максимальная критичность: High)

| Issue # / Заголовок | Ссылка | Критичность | Модуль | Краткое описание |
|---------------------|--------|-------------|--------|-----------------|
| #2143 — Routing not working with special character in UpstreamPathTemplate | [#2143](https://github.com/ThreeMammals/Ocelot/issues/2143) | **High** | Routing | Маршруты с символом `$` в `UpstreamPathTemplate` (например, `/v3/Orders/$query` в OData) не работают — Ocelot не может корректно обработать `$` как часть пути, что блокирует интеграцию с OData-сервисами. |
| #2191 — Follow up #2150: Special chars in values of query strings | [#2191](https://github.com/ThreeMammals/Ocelot/issues/2191) | **High** | Routing | Специальные символы в значениях query string (`(`, `)`, `*`, `+`, `?`, `|`, `{`, `}`, `[`, `]`, `^`, `$`, `.`, `#`, пробел) некорректно обрабатываются методом `MergeQueryStringsWithoutDuplicateValues` в middleware. Запросы с такими параметрами завершаются ошибкой. |
| #714 — Multipart/form-data is not rerouted (error 404) | [#714](https://github.com/ThreeMammals/Ocelot/issues/714) | **High** | Routing / Middleware | HTTP POST-запросы с `Content-Type: multipart/form-data` (загрузка файлов) не перенаправляются к downstream-сервису и возвращают 404. Функциональность загрузки файлов через API Gateway недоступна. |

---

### 🟠 LoadBalancer (2 бага, максимальная критичность: High)

| Issue # / Заголовок | Ссылка | Критичность | Модуль | Краткое описание |
|---------------------|--------|-------------|--------|-----------------|
| #1041 — How to avoid calling down service in load balancer option | [#1041](https://github.com/ThreeMammals/Ocelot/issues/1041) | **High** | LoadBalancer / HealthCheck | Load Balancer продолжает направлять запросы на недоступный (упавший) downstream-сервис, возвращая HTTP 500 на каждый запрос. Отсутствует механизм исключения нездоровых узлов из ротации. Помечен как `high` приоритет. |
| #1513 — Ocelot Load Balance using Web Socket Issue | [#1513](https://github.com/ThreeMammals/Ocelot/issues/1513) | **High** | LoadBalancer / WebSocket | При падении одного из backend-серверов WebSocket-соединения (SignalR) не переключаются на другой сервер — возвращается Bad Gateway. Load Balancer не поддерживает failover для WebSocket-соединений. |

---

### 🟡 Aggregation (2 бага, максимальная критичность: Medium)

| Issue # / Заголовок | Ссылка | Критичность | Модуль | Краткое описание |
|---------------------|--------|-------------|--------|-----------------|
| #2248 — Aggregates + RouteKeysConfig where the array of values does not work correctly | [#2248](https://github.com/ThreeMammals/Ocelot/issues/2248) | **Medium** | Aggregation | При использовании `RouteKeysConfig` с массивом значений в агрегированных маршрутах параметры пути (например, `userId`) не подставляются корректно в downstream-запросы. Агрегатор не извлекает значения из JSON-ответа первого сервиса для передачи во второй. |
| #2328 — Ensure correct mapping of RouteKeysConfig arrays in aggregates (PR) | [#2328](https://github.com/ThreeMammals/Ocelot/pull/2328) | **Medium** | Aggregation | Pull Request, исправляющий баг #2248. Агрегатор некорректно подставлял значение `JsonPath`-параметра в downstream-запрос, что приводило к пустым или неверным вызовам. PR находится в открытом состоянии (не смержен). |

---

### 🟠 ServiceDiscovery (1 баг, критичность: High)

| Issue # / Заголовок | Ссылка | Критичность | Модуль | Краткое описание |
|---------------------|--------|-------------|--------|-----------------|
| #2208 — Consul Node.Name might be a DNS name | [#2208](https://github.com/ThreeMammals/Ocelot/issues/2208) | **High** | ServiceDiscovery / Consul | Провайдер Consul Service Discovery не проверяет, является ли `Node.Name` DNS-именем хоста. В зависимости от конфигурации агента Consul `Node.Name` может быть hostname, а не IP-адресом, что приводит к некорректному формированию URL сервиса. |

---

### 🟡 Authorization (1 баг, критичность: Medium)

| Issue # / Заголовок | Ссылка | Критичность | Модуль | Краткое описание |
|---------------------|--------|-------------|--------|-----------------|
| #679 — Ocelot doesn't handle correctly RouteClaimsRequirement with a key as an Url | [#679](https://github.com/ThreeMammals/Ocelot/issues/679) | **Medium** | Authorization | `RouteClaimsRequirement` не работает корректно, когда ключ claim является URL (например, `http://schemas.microsoft.com/ws/2008/06/identity/claims/role` из `ClaimTypes.Role`). Ocelot не может сопоставить claim с URL-ключом, что блокирует авторизацию на основе стандартных .NET claim-типов. |

---

### 🟢 Administration (1 баг, критичность: Low)

| Issue # / Заголовок | Ссылка | Критичность | Модуль | Краткое описание |
|---------------------|--------|-------------|--------|-----------------|
| #989 — Hide FileConfiguration and OutputCache controller from Swagger | [#989](https://github.com/ThreeMammals/Ocelot/issues/989) | **Low** | Administration / Swagger | Внутренние контроллеры Ocelot (`FileConfigurationController`, `OutputCacheController`) отображаются в Swagger UI пользовательского API Gateway. Нет механизма их скрытия из документации. |

---

## Итоговая сводка

### Распределение по критичности

| Критичность | Количество | Issues |
|-------------|-----------|--------|
| Critical    | 1         | #1252  |
| High        | 6         | #2143, #2191, #714, #1041, #1513, #2208 |
| Medium      | 3         | #1687, #2248, #679 |
| Low         | 1         | #989   |

### Модули с наибольшим количеством критичных багов

1. **Routing** — 3 бага (2× High + 1× High) — наибольшее количество открытых проблем
2. **LoadBalancer** — 2 бага (2× High) — оба критичны для production-нагрузки
3. **Middleware** — 2 бага (1× Critical + 1× Medium) — содержит единственный Critical-баг
4. **Aggregation** — 2 бага (2× Medium) — функциональные ошибки в агрегации
5. **ServiceDiscovery** — 1 баг (1× High) — проблема с Consul в определённых конфигурациях
6. **Authorization** — 1 баг (1× Medium) — несовместимость со стандартными .NET claim-типами
7. **Administration** — 1 баг (1× Low) — косметическая проблема с Swagger

### Выводы

- **Наиболее проблемный модуль по количеству багов:** `Routing` (3 открытых бага, все High)
- **Наиболее критичный баг:** `#1252` (Critical) — регрессия в `DelegatingHandler`, ломающая доступ к `HttpContext`
- **Системная проблема:** `LoadBalancer` не имеет механизма Health Check для исключения недоступных узлов (баги #1041 и #1513 — разные проявления одной архитектурной проблемы)
- **Долгожители:** Баги #679 (2019), #714 (2019), #989 (2019), #1041 (2020), #1252 (2021) открыты более 3–6 лет, что свидетельствует о низком приоритете или сложности исправления
