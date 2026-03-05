# Зависимости и версии

Все проекты: **.NET 8.0**

## Матрица NuGet-пакетов

### Основные библиотеки

| Пакет | Версия | Проекты |
|-------|--------|---------|
| Google.Apis.Auth | 1.72.0 | Core |
| Google.Apis.Sheets.v4 | 1.72.0.3966 | Core |
| NPOI | 2.7.1 | Verifier |
| QuestPDF | 2023.12.1 | PdfReport |
| SixLabors.ImageSharp | 3.0.2 | PdfReport |

### Веб-инфраструктура (ParsingPipelineApp)

| Пакет | Версия | Назначение |
|-------|--------|-----------|
| Microsoft.EntityFrameworkCore.Sqlite | 8.0.0 | ORM + хранение истории pipeline |
| Microsoft.EntityFrameworkCore.Tools | 8.0.0 | Миграции EF Core |
| Serilog.AspNetCore | 8.0.3 | Структурированное логирование |
| Swashbuckle.AspNetCore | 6.5.0 | Swagger / OpenAPI документация |

### Тестирование

| Пакет | Версия | Проекты |
|-------|--------|---------|
| NUnit | 3.13.3 | Core.Tests, Verifier.Tests, E2ETests |
| NUnit | 3.14.0 | First |
| NUnit3TestAdapter | 4.5.0 | Core.Tests, Verifier.Tests, E2ETests, First |
| xunit | 2.4.2 | PdfReport.Tests |
| xunit.runner.visualstudio | 2.4.3 | PdfReport.Tests |
| Microsoft.NET.Test.Sdk | 17.8.0 | Core.Tests, Verifier.Tests, E2ETests |
| Microsoft.NET.Test.Sdk | 17.11.1 | First |
| Microsoft.NET.Test.Sdk | 17.3.2 | PdfReport.Tests |
| coverlet.collector | 6.0.0 | Core.Tests, Verifier.Tests, E2ETests |
| coverlet.collector | 3.1.2 | PdfReport.Tests |
| NUnit.Analyzers | 3.6.1 | Core.Tests, Verifier.Tests, E2ETests |
| Moq | 4.20.70 | Core.Tests |
| Microsoft.Playwright.NUnit | 1.41.0 | E2ETests |

### Конфигурация (E2ETests)

| Пакет | Версия | Назначение |
|-------|--------|-----------|
| Microsoft.Extensions.Configuration | 8.0.0 | Чтение конфигурации |
| Microsoft.Extensions.Configuration.EnvironmentVariables | 8.0.0 | Переменные окружения |

## Проекты без NuGet-зависимостей

- **BusinessAnalytics.RunAnalytics** — только проектные ссылки (Core, Verifier, PdfReport)

## Тестовые фреймворки по проектам

| Проект | Фреймворк | Причина |
|--------|-----------|---------|
| Core.Tests | NUnit | Основной фреймворк проекта |
| Verifier.Tests | NUnit | Согласованность с Core |
| PdfReport.Tests | xUnit | Исторически выбран при создании |
| E2ETests | NUnit + Playwright | Browser-тестирование React UI |
| First | NUnit | Независимый проект |

## Примечания

- **Два тестовых фреймворка**: NUnit (основной) и xUnit (только PdfReport). При создании новых тестов — использовать NUnit
- **Расхождение версий**: NUnit 3.13.3 vs 3.14.0 (First), coverlet 6.0.0 vs 3.1.2 (PdfReport) — кандидаты на выравнивание
- **Vite** используется для сборки React-фронтенда, но не является NuGet-пакетом (npm)
