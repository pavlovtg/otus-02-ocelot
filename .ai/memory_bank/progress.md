# Progress

## Статус проекта
🟡 В процессе — учебное задание по code review

## Выполнено
- [x] Форк репозитория Ocelot
- [x] Инициализация AI-инфраструктуры (`.ai/` папка, memory bank, rules)
- [x] Создание шаблонов промтов
- [x] Описание проекта Ocelot (`detailed-description.md`, `short-description.md`)
- [x] Архитектурный анализ Ocelot с C4-диаграммами (`architecture.md`)
- [x] Анализ сильных и слабых сторон архитектуры (`architecture-strengths-and-weaknesses.md`)
- [x] Анализ открытых багов из GitHub Issues (`bugs-analysis.md`): 11 багов, 7 модулей, 1 Critical / 6 High / 3 Medium / 1 Low
- [x] Создание промта для выбора модуля code review (`homework/prompts/06-choose-module.md`)
- [x] Выполнение скоринга и выбор модуля LoadBalancer, результат в `homework/review/choose-module.md`

## В работе
- [x] Выбор модуля для code review — выбран **LoadBalancer** (Score=9.40)
- [ ] Проведение code review с AI-ассистентом
- [ ] Сохранение промтов в `homework/prompts/`

## Запланировано
- [ ] Оформление итогового отчёта
- [ ] Сдача домашнего задания

## Известные проблемы / Заметки
- Репозиторий является форком ThreeMammals/Ocelot
- Для code review рекомендуется выбрать один модуль (не весь проект)
- Все промты должны быть сохранены и приложены к отчёту
- Состояние балансировщиков (RoundRobin, LeastConnection) не распределено между экземплярами Ocelot
- Administration API зависит от устаревшего IdentityServer4

## История изменений
| Дата | Изменение |
|------|-----------|
| 2026-04-27 | Инициализация AI memory bank и структуры `.ai/` |
| 2026-04-27 | Создание описания проекта Ocelot (detailed + short) |
| 2026-04-27 | Архитектурный анализ Ocelot: паттерны, карта модулей, C4-диаграммы |
| 2026-04-27 | Анализ сильных и слабых сторон архитектуры: 7 сильных сторон, 10 уязвимостей, 6 зон техдолга, 10 рекомендаций |
| 2026-04-27 | Анализ открытых багов GitHub Issues: 11 багов, 7 модулей, наиболее проблемный — Routing (3 бага), Critical — #1252 (DelegatingHandler) |
| 2026-04-27 | Скоринг модулей для code review: выбран LoadBalancer (Score=9.40), промт `06-choose-module.md`, результат `homework/review/choose-module.md` |
