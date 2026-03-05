# Взаимосвязи компонентов

## Граф зависимостей

```
BusinessAnalytics.RunAnalytics (Console App - оркестратор)
  ├── BusinessAnalytics.Core
  ├── BusinessAnalytics.Verifier
  └── BusinessAnalytics.PdfReport

ParsingPipelineApp.Api (Web App - оркестратор)
  ├── BusinessAnalytics.Core
  ├── BusinessAnalytics.Verifier
  └── BusinessAnalytics.PdfReport

BusinessAnalytics.Verifier
  └── BusinessAnalytics.Core

BusinessAnalytics.PdfReport
  └── BusinessAnalytics.Core

BusinessAnalytics.Core
  └── (внешние: Google.Apis.Sheets.v4, Google.Apis.Auth)

First (независимый проект, без связей)
```

## Потоки данных

### 1. Загрузка и парсинг

```
Google Sheets
  ├── Лист "Факт"      → ProjectPLParser      → PLStructure (ProjectProfitData[])
  ├── Лист "Доходы"     → TotalIncomesParser   → TotalIncomes[]
  ├── Лист "Выплаты"    → PaymentsParser       → PaymentData[]
  ├── Лист "Итоги мес." → ExpensesParser       → MonthlyExpenseData[]
  └── Лист "Проекты"    → ProjectsSheetParser  → ProjectMetadata[]
```

### 2. Построение реестра проектов

```
PLStructure + TotalIncomes + PaymentData + MonthlyExpenseData
        |
        v
  ProjectRegistry.Build()
        |
        v
  Union-Find кластеризация → Canonical Names
```

Проблема: один проект может иметь разные названия в разных источниках.
Решение: `NameCleaningService` нормализует имена, `ProjectRegistry` группирует варианты через Union-Find.

### 3. Верификация (6 проверок)

| Проверка | Сервис | Что сравнивает |
|----------|--------|----------------|
| Выплаты vs Факт | PaymentsPLFactReconciliationService | Проекты из Выплат есть ли в P&L |
| Расходы vs Факт | ExpensesPLFactReconciliationService | Проекты из Расходов есть ли в P&L |
| Клиенты: Доходы vs Факт | PLClientsCompatibilityVerifier | Клиенты из Доходов есть ли в P&L |
| Расхождения доходов | IncomeDiscrepancyVerifier | Суммы доходов совпадают ли |
| Валидация P&L | PLValidationService | Внутренняя согласованность P&L |
| Метаданные проектов | ProjectsMetadataVerificationStage | Полнота полей и наличие новых проектов |

### 4. Расчёт дат проектов (3-уровневый fallback)

```
ProjectDateCalculationService.Calculate()
  1. Расходы (MonthlyExpenseData)  → даты старта/конца по месяцам расходов
  2. Доходы (TotalIncomes)         → даты по операциям зачисления (если нет расходов)
  3. P&L (ProjectProfitData)       → даты из MonthlyData с ненулевыми суммами (если нет 1 и 2)
```

### 5. Аналитика

```
ProjectProfitData + ProjectMetadata + ProjectRegistry
        |
        v
  ├── CustomerMarginService        → Маржинальность по клиентам
  ├── ProjectTypeMarginService     → Таблица маржи по типам проектов и годам
  │     ↑ ProjectMetadata.ProjectType + StartPeriod (год)
  │     ↑ ProjectRegistry (canonical name → связка с P&L)
  │     → ProjectTypeMarginRow[] (строки: тип проекта, колонки: годы, значения: сумма маржи)
  └── ExpensePLComparisonService   → Постоянные vs проектные расходы
```

### 6. Экспорт

```
Результаты аналитики
  ├── ProjectResultsExporter → JSON (маржа по проектам/клиентам)
  ├── ExpensesExporter       → JSON (расходы)
  ├── PdfReportService       → PDF (отчёт по клиентам, QuestPDF)
  ├── ExcelHighlighter       → XLSX (подсветка расхождений, NPOI)
  └── GoogleSheetsWriteService → Google Sheets (синхронизация проектов)
```

## Два оркестратора

### RunAnalytics (Console)
- Последовательный pipeline из 10 стадий (IPipelineStage)
- Каждая стадия в try-catch (продолжение при ошибке)
- Без сохранения истории
- Конфигурация из `google-sheets-config.json`

### ParsingPipelineApp (Web)
- Асинхронный pipeline из 5 стадий
- Singleton lock (одно выполнение за раз)
- История выполнений в SQLite
- REST API + React UI для мониторинга
- Скачивание сгенерированных файлов

## Общие паттерны

1. **Strategy** - `IDataSource` (Google Sheets / JSON)
2. **Template Method** - `BaseTableParser` для всех парсеров
3. **Pipeline** - `IPipelineStage` для последовательной обработки
4. **Repository** - `ProjectRegistry` как единый реестр проектов
5. **DI-ready** - все зависимости через конструкторы
