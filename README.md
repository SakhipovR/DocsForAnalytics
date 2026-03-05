# BusinessAnalytics - Документация проекта

## Обзор

**BusinessAnalytics** - система финансовой аналитики для анализа проектов, клиентов и P&L.
Загружает данные из Google Sheets, выполняет парсинг, валидацию, сверку и генерирует отчёты.

## Репозитории

| Репозиторий | Описание | Технологии |
|-------------|----------|------------|
| **AnalyticsTC** | Основной monorepo (solution .NET 8.0) | C#, .NET 8.0, Google Sheets API |
| **First** | Утилита извлечения ставок из текстов вакансий | C#, .NET 8.0, NUnit |

## Проекты внутри AnalyticsTC

| Проект | Тип | Назначение |
|--------|-----|-----------|
| [BusinessAnalytics.Core](services/core.md) | Библиотека | Ядро: модели, парсеры, сервисы, источники данных |
| [BusinessAnalytics.Verifier](services/verifier.md) | Библиотека | Верификация и сверка данных между источниками |
| [BusinessAnalytics.PdfReport](services/pdf-report.md) | Библиотека | Генерация PDF-отчётов (QuestPDF) |
| [BusinessAnalytics.RunAnalytics](services/run-analytics.md) | Console App | Консольный оркестратор pipeline |
| [ParsingPipelineApp](services/parsing-pipeline-app.md) | Web App | Веб-приложение (ASP.NET Core API + React) |
| [BusinessAnalytics.Core.Tests](services/tests.md) | Тесты | Unit-тесты ядра |
| [BusinessAnalytics.Verifier.Tests](services/tests.md) | Тесты | Unit-тесты верификации |
| [BusinessAnalytics.PdfReport.Tests](services/tests.md) | Тесты | Unit-тесты PDF-отчётов |
| [First](services/first.md) | Console App | Утилита поиска ставок в вакансиях |

## Архитектура высокого уровня

```
Google Sheets (Доходы, Выплаты, Расходы, Факт, Проекты)
        |
        v
  IDataSource / GoogleSheetsDataSource
        |
        v
  Парсеры (PL, Incomes, Payments, Expenses, Projects)
        |
        v
  Модели данных (ProjectProfitData, TotalIncomes, PaymentData, ...)
        |
        v
  ProjectRegistry (нормализация + кластеризация имён)
        |
        v
  +-----------+-----------+-----------+
  |           |           |           |
  v           v           v           v
Аналитика  Верификация  Экспорт    PDF-отчёты
(Margin,   (Сверки,     (JSON,     (QuestPDF)
 P&L)       Highlight)   TXT)
```

## Ключевые документы

- [Взаимосвязи компонентов](relationships.md) - как проекты зависят друг от друга
- [Глоссарий](glossary.md) - термины и определения
- [Сервисы](services/) - детальная документация по каждому проекту

## Технологический стек

- **.NET 8.0** - платформа
- **Google Sheets API v4** - источник данных (чтение + запись)
- **QuestPDF** - генерация PDF
- **NPOI** - работа с Excel (подсветка расхождений)
- **NUnit** - тестирование
- **ASP.NET Core** - веб-API
- **React + TypeScript** - фронтенд (ParsingPipelineApp)
- **SQLite** - хранение истории выполнений pipeline
