# BusinessAnalytics.Core

Ядро системы. Содержит модели данных, парсеры, сервисы аналитики, абстракции источников данных.

## Зависимости

- **Google.Apis.Auth** 1.72.0
- **Google.Apis.Sheets.v4** 1.72.0.3966
- Платформа: .NET 8.0

## Структура

```
BusinessAnalytics.Core/
├── Abstractions/         # Интерфейсы источников данных
├── Configuration/        # Конфигурация Google Sheets и парсинга
├── DataSources/          # Реализации чтения/записи данных
├── Models/               # Все модели и DTO
├── Services/             # Бизнес-логика
├── BaseTableParser.cs    # Базовый класс парсеров
├── ExcelDataLocator.cs   # Поиск данных в таблицах
└── Parsers (5 парсеров)
```

## Абстракции

| Интерфейс | Методы | Назначение |
|-----------|--------|-----------|
| `IDataSource` | `ReadDataAsync(source)` → `TableData` | Универсальный контракт чтения данных |
| `IExcelReader` | extends IDataSource + `SheetName` | Специализация для Excel/Sheets |
| `IProjectMatcher` | `AreMatched()`, `IsKnownProject()` | Контракт сопоставления проектов |

## Источники данных (DataSources)

| Класс | Описание |
|-------|----------|
| `GoogleSheetsDataSource` | Чтение из Google Sheets API v4 (публичный доступ по API key) |
| `JsonFileDataSource` | Чтение из JSON файлов (для тестов или предзагруженных данных) |
| `GoogleSheetsWriteService` | Запись в Google Sheets через Service Account (OAuth2) |

## Конфигурация

### GoogleSheetsConfig
Загружается из `google-sheets-config.json`:
- `ApiKey` - ключ для чтения
- `SpreadsheetUrl` / `ExpensesSpreadsheetUrl` / `PaymentsSpreadsheetUrl` / `ProjectsSpreadsheetUrl`
- `ServiceAccountCredentialsPath` - путь к JSON ключам для записи
- `SheetNames` - имена листов (IncomesSheet, PaymentSheets[], ExpensesSheet, PlSheet, ProjectsSheet)
- `OutputPaths` - пути для экспорта (ResultsText, ProjectMarginsJson, ExpensesJson)

### ParsingConfiguration
Маркеры для определения границ данных в таблице:
- Начало данных: "Проекты:"
- Конец данных: "Конец проектов"
- Маркер года: "Год"
- Валидные месяцы: список русских названий

## Модели данных

### Финансовые модели

| Модель | Ключевые поля | Назначение |
|--------|---------------|-----------|
| `ProjectData` | ProjectName, Client, MonthlyData[], DebtorBalance | Проект с ежемесячными данными |
| `ProjectProfitData` | TotalIncome, TotalExpense, Profit, DebtorBalance | Агрегированная прибыльность |
| `MonthlyData` | Year, Month, Income, Expense | Один месяц по проекту |
| `PLStructure` | Projects[], ConstantExpenses[] | Полная структура P&L |
| `TotalIncomes` | ProjectName, Income, OperationDate, Client | Запись о доходе |
| `PaymentData` | Recipient, PaymentType, Amount, PaymentDate, ProjectName | Исходящий платёж |
| `ProjectExpenseData` | ProjectName, TotalExpense, BdmBonus, DirectExpense | Расходы по проекту/месяцу |
| `MonthlyExpenseData` | Projects[], TotalMonthExpense | Месяц расходов |
| `ConstantExpenseData` | Name, Amount, Year, Month | Постоянный расход |
| `ProjectMetadata` | ProjectName, ClientName, ProjectType, SalesManager, StartPeriod, EndPeriod | Метаданные проекта |

### Аналитические модели

| Модель | Назначение |
|--------|-----------|
| `CustomerMarginMetric` | Маржинальность клиента: TotalMargin, Profitability%, MarginScore |
| `ProjectTypeMarginRow/Result` | Маржа по типам проектов с разбором по годам |
| `ClientMonthlyDetailRow` | Детали по клиенту за месяц |

### Инфраструктурные модели

| Модель | Назначение |
|--------|-----------|
| `TableData` | Унифицированное представление таблицы |
| `RowData` | Строка таблицы (ячейки по адресам A1, B2...) |
| `DataLocation` | Координаты данных: год, начальная/конечная строка, колонки месяцев |

## Парсеры

Все наследуют от `BaseTableParser`:

| Парсер | Источник | Результат | Особенности |
|--------|----------|-----------|-------------|
| `ProjectPLParser` | Лист "Факт" | `List<ProjectProfitData>` | Многострочные проекты (доход + расход), множественные годы |
| `TotalIncomesParser` | Лист "Доходы" | `List<TotalIncomes>` | Колонки: Проект, Зачисление, Дата, Контрагент |
| `PaymentsParser` | Лист "Выплаты" | `List<PaymentData>` | Кому, Тип, Сумма, Проект, Дата |
| `ExpensesParser` | Лист "Итоги месяцев" | `List<MonthlyExpenseData>` | Блочная структура: маркер "Месяц Год" → данные → "Итого" |
| `ProjectsSheetParser` | Лист "Проекты" | `List<ProjectMetadata>` | Название, клиент, тип, менеджер, даты |

### BaseTableParser (базовые утилиты)
- Работа с адресами ячеек: `ExtractColumnLetter()`, `GetColumnIndex()`, `GetColumnLetter()`
- Парсинг значений: `ParseDecimal()`, `ParseDate()`
- Поиск колонок: `FindColumns()`

### ExcelDataLocator
- `FindYear()` - определение года в таблице
- `FindStartRow()` / `FindEndRow()` - границы данных
- `FindMonthColumns()` - колонки с месяцами
- Возвращает `DataLocation`

## Сервисы

### Нормализация и матчинг

| Сервис | Тип | Описание |
|--------|-----|----------|
| `NameCleaningService` | static | Нормализация имён: удаление ключевых слов → версии → lowercase → спецсимволы → пробелы |
| `ProjectMatchingService` | static | 3-уровневый матчинг: точное → substring → без числовых суффиксов |
| `ProjectRegistry` | instance | Реестр проектов. Union-Find кластеризация. Canonical/Display names |
| `AlgorithmicMatcher` | instance | IProjectMatcher через 3-уровневый алгоритм |
| `RegistryBasedMatcher` | instance | IProjectMatcher через ProjectRegistry |

### Аналитика

| Сервис | Описание |
|--------|----------|
| `CustomerMarginService` | Маржинальность по клиентам. Profitability% и MarginScore |
| `ProjectTypeMarginService` | Маржинальность по типам проектов с разбором по годам |
| `PLValidationService` | Валидация P&L: сравнение с Incomes/Expenses |
| `ExpensePLComparisonService` | Разделение расходов на постоянные и проектные |

### Периоды

| Сервис | Описание |
|--------|----------|
| `MonthNameHelper` | Конвертация названий месяцев (рус/англ) ↔ номера |
| `PeriodCalculationService` | Общий период всех 3 источников (PL, Incomes, Expenses) |
| `ProjectDateCalculationService` | Даты старта/конца проекта (приоритет: расходы → доходы → PL) |

### Экспорт

| Сервис | Описание |
|--------|----------|
| `ProjectResultsExporter` | Экспорт маржи проектов в JSON (группировка по клиентам) |
| `ExpensesExporter` | Экспорт расходов в JSON |

## Ключевая бизнес-логика

### Расчёт маржи проекта
```
Profit = TotalIncome - TotalExpense + DebtorBalance
```

### Маржинальность клиента
```
Profitability = TotalMargin / (TotalIncome + DebtorBalance) * 100%
```

### MarginScore
| Маржа | Score |
|-------|-------|
| < 150 000 | 2 |
| <= 800 000 | 5 |
| <= 3 000 000 | 7 |
| > 3 000 000 | 10 |
