# BusinessAnalytics.PdfReport

Генерация PDF-отчётов по клиентской аналитике.

## Зависимости

- **BusinessAnalytics.Core** (проектная ссылка)
- **QuestPDF** 2023.12.1 - генерация PDF (Community лицензия)
- **SixLabors.ImageSharp** 3.0.2 - обработка изображений (подключена, не используется)
- Платформа: .NET 8.0

## Классы

### ReportGenerationService

Координатор генерации. Вызывает JsonParser → ClientCalculation → PdfReport.

**Методы:**
- `GenerateReportFromJson(jsonFilePath)` → `byte[]` (PDF в памяти)
- `GenerateReportToFile(jsonFilePath, outputPdfPath)` - сохранение в файл

### JsonParserService

Чтение и агрегирование данных из JSON.

- Десериализация JSON → `List<ClientProfitData>`
- Агрегирование по клиенту: TotalIncome, TotalExpense, TotalProfit
- Расчёт MarginPercentage = (TotalProfit / TotalIncome) * 100%

### ClientCalculationService

Сортировка и фильтрация клиентов.

**Методы:**
- `GetTopClientsByMargin(clients, count=10)` - топ по % маржи
- `GetTopClientsByAbsoluteMargin(clients, count=10)` - топ по абсолютной марже
- `GetAllClientsSortedByMargin(clients)` - все, сортировка по прибыли

### PdfReportService

Формирование PDF через QuestPDF Fluent API.

**Структура отчёта:**
```
┌──────────────────────────────────┐
│ Аналитика по клиентам            │  ← Header (16pt, bold)
│ Отчет создан: dd.MM.yyyy HH:mm  │  ← (8pt, серый)
├──────────────────────────────────┤
│ Топ-10 клиентов по марже         │  ← Section (14pt, синий)
│ ┌────────────────────────────┐   │
│ │ 1. Клиент X                │   │  ← Block (рамка, фон)
│ │    Доход: 1 234 567,89     │   │
│ │    Расход: 987 654,32      │   │
│ │    Маржа абсол.: 246 913   │   │
│ │    Маржа: 20.00%           │   │
│ └────────────────────────────┘   │
│ ... (до 10 клиентов)             │
│ ─────── Разрыв страницы ──────── │
│ Все клиенты и их маржа           │  ← (если >10 клиентов)
├──────────────────────────────────┤
│ Страница X из Y                  │  ← Footer
└──────────────────────────────────┘
```

**Параметры форматирования:**
- Страница: A4, отступы 1 см
- Шрифт: Arial, 10pt
- Цвета: тёмно-синий (заголовки), серый (служебное)
- Числа: N2 (с разделителем тысяч), проценты F2

## Модели

| Модель | Поля |
|--------|------|
| `ClientProfitData` | ClientName, Projects[], TotalIncome, TotalExpense, TotalProfit, MarginPercentage |
| `ProjectData` | ProjectName, TotalIncome, TotalExpense, Profit, DebtorBalance |
