# Tech Context

## Технологический стек

### Основной стек
- **Язык:** C# (.NET)
- **Фреймворк:** ASP.NET Core
- **Целевые платформы:** .NET 8+

### Зависимости
- `Microsoft.AspNetCore.*` — базовый фреймворк
- `Ocelot` — основная библиотека (этот репозиторий)
- `Consul` — service discovery (опционально)
- `Polly` — resilience и retry политики

### Тестирование
- **Фреймворк:** xUnit
- **Моки:** Moq / NSubstitute
- **Acceptance тесты:** Microsoft.AspNetCore.TestHost
- **Покрытие:** Coverlet

### Инструменты разработки
- **IDE:** Visual Studio / VS Code / Rider
- **Build:** `dotnet build`, `build.cake`
- **Версионирование:** GitVersion (SemVer)
- **Документация:** Sphinx (RST формат)

### CI/CD
- GitHub Actions
- Docker (Dockerfile.build, Dockerfile.release)

## Структура решения
```
Ocelot.sln
├── src/
│   ├── Ocelot/                    # Основная библиотека
│   ├── Ocelot.Provider.Consul/    # Consul провайдер
│   └── Ocelot.Provider.Kubernetes/ # Kubernetes провайдер
├── test/
│   ├── Ocelot.UnitTests/
│   ├── Ocelot.AcceptanceTests/
│   └── Ocelot.Benchmarks/
└── samples/                       # Примеры использования
```

## Команды разработки
```bash
# Сборка
dotnet build Ocelot.sln

# Тесты
dotnet test Ocelot.sln

# Конкретный проект
dotnet test test/Ocelot.UnitTests/
```
