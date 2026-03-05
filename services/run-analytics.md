# BusinessAnalytics.RunAnalytics

Консольное приложение - оркестратор pipeline аналитики.

## Зависимости

- **BusinessAnalytics.Core** (проектная ссылка)
- **BusinessAnalytics.Verifier** (проектная ссылка)
- **BusinessAnalytics.PdfReport** (проектная ссылка)
- Платформа: .NET 8.0, OutputType: Exe

## Архитектура

Последовательный pipeline из 10 стадий (`IPipelineStage`), каждая выполняется в try-catch.
Данные передаются через `PipelineContext`.

## PipelineContext

Контейнер, передаваемый между стадиями:

| Поле | Тип | Описание |
|------|-----|----------|
| `Config` | GoogleSheetsConfig | Конфигурация подключения |
| `Incomes` | List\<TotalIncomes\> | Загруженные доходы |
| `Payments` | List\<PaymentData\> | Загруженные выплаты |
| `Expenses` | List\<MonthlyExpenseData\> | Загруженные расходы |
| `PlStructure` | PLStructure | P&L структура |
| `SortedProjects` | List\<ProjectProfitData\> | Отсортированные проекты |
| `AllDiscrepancies` | List\<IncomeDiscrepancyInfo\> | Найденные расхождения |
| `ProjectRegistry` | ProjectRegistry | Реестр проектов |
| `VerificationResults` | VerificationResults | Результаты всех проверок |
| `ProjectsMetadata` | List\<ProjectMetadata\> | Метаданные проектов |

## Стадии pipeline (порядок выполнения)

### 1. DataLoadingStage - "Загрузка данных"
- Загрузка Incomes, Payments, Expenses, PLStructure из Google Sheets
- Построение ProjectRegistry из всех источников
- Вывод статистики в консоль

### 2. ExportStage - "Экспорт и отчёты"
- Текстовый отчёт (детали по проектам, постоянные расходы, чистая прибыль)
- JSON маржин проектов
- PDF-отчёт через ReportGenerationService

### 3. PaymentsVerificationStage - "Сверка: Выплаты vs Факт"
- Находит проекты из Выплат, отсутствующие в P&L
- Топ-5 по количеству выплат

### 4. ExpensesVerificationStage - "Сверка: Расходы vs Факт"
- Находит проекты из Расходов, отсутствующие в P&L
- Топ-5 по сумме расходов

### 5. ClientsIncomesVerificationStage - "Сверка: Клиенты Доходы vs Факт"
- Находит клиентов из Доходов, отсутствующих в P&L
- Топ-10 по сумме доходов

### 6. IncomeDiscrepancyVerificationStage - "Проверка расхождений доходов"
- Сравнивает суммы доходов между источниками
- Топ-3 по величине расхождения

### 7. PLValidationStage - "Валидация PL"
- Проверяет внутреннюю согласованность P&L
- Общий период данных через PeriodCalculationService
- Топ-5 расхождений

### 8. ProjectsMetadataVerificationStage - "Проверка метаданных проектов"
- Проверяет полноту метаданных проектов (ProjectMetadata)
- Находит проекты с отсутствующими полями (тип, менеджер, даты)
- Определяет новые проекты без записи в листе "Проекты"

### 9. ProjectsSyncStage - "Синхронизация проектов"
- Загрузка существующих проектов из Google Sheets
- Определение недостающих проектов из ProjectRegistry
- Предзаполнение данных (клиент, даты)
- Запись новых проектов в Google Sheets

### 10. HighlightStage - "Подсветка в Excel"
- `output_incomes_highlighted.xlsx` - красная подсветка доходов
- `output_fact_highlighted.xlsx` - жёлтая подсветка в Факте

## Конфигурация

Файл: `google-sheets-config.json` (в директории сборки)

Аргументы командной строки не используются.

## Выходные файлы

| Файл | Содержимое |
|------|-----------|
| `config.OutputPaths.ResultsText` | Текстовый отчёт |
| `config.OutputPaths.ProjectMarginsJson` | JSON маржин |
| `config.OutputPaths.ExpensesJson` | JSON расходов |
| `Reports/ClientAnalysisReport.pdf` | PDF-отчёт |
| `output_incomes_highlighted.xlsx` | Excel с подсветкой доходов |
| `output_fact_highlighted.xlsx` | Excel с подсветкой факта |

## Обработка ошибок

- Каждая стадия в try-catch
- Вывод: `[!] Ошибка в стадии '{name}': {message}`
- Выполнение продолжается при ошибке одной стадии
