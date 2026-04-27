# Шаблон: Commit и Push

## Задача
Закоммить изменения и запушь.

## Шаги

```bash
git status
git add .  # или git add <файл>
git commit -m "<тип>: <описание на русском>"
git push
git log --oneline -5
```

Типы: `feat`, `fix`, `docs`, `refactor`, `chore`, `test`.

## Примечания
- Коммит на русском
- Не коммить секреты и временные файлы
