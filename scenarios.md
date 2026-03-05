# Сквозные сценарии

## Сценарий: Обнаружение и синхронизация нового проекта

Описывает полный путь от появления проекта в финансовых данных до его автоматической записи в лист "Проекты" Google Sheets.

### Предпосылка

Проект "Restart #2" появился в трёх источниках:
- Лист "Факт" (P&L) — как `"Restart #2"`
- Лист "Доходы" — как `"restart #2 nrd"`
- Лист "Итоги мес." (расходы) — как `"restart (2)"`

В листе "Проекты" записи о нём нет.

### Шаг 1: Загрузка данных (DataLoadingStage)

```
Google Sheets → 5 парсеров → модели данных
  ├── ProjectPLParser       → ProjectProfitData[]   (содержит "Restart #2")
  ├── TotalIncomesParser    → TotalIncomes[]         (содержит "restart #2 nrd")
  ├── ExpensesParser        → MonthlyExpenseData[]   (содержит "restart (2)")
  ├── PaymentsParser        → PaymentData[]
  └── ProjectsSheetParser   → ProjectMetadata[]      (нет записи!)
```

### Шаг 2: Построение реестра (ProjectRegistry.Build)

```
Все имена проектов из 5 источников
        |
        v
NameCleaningService.Normalize()
  "Restart #2"      → "restart"
  "restart #2 nrd"  → "restart nrd"
  "restart (2)"     → "restart"
        |
        v
ProjectMatchingService.AreMatched() — 3 уровня:
  1. Точное совпадение:     "restart" == "restart" ✓
  2. Substring:             "restart nrd" содержит "restart" ✓
  3. Без суффиксов:         (не потребовался)
        |
        v
Union-Find: все 3 варианта → один кластер
  Canonical Name:  "restart" (самое частое нормализованное)
  Display Name:    "Restart #2" (самый частый оригинал)
```

### Шаг 3: Поиск недостающих проектов (ProjectsSyncStage)

```
Для каждого canonical name из ProjectRegistry:
  → Нормализовать все имена из листа "Проекты"
  → Проверить: есть ли хотя бы один alias в множестве?

"restart" — ни один alias не найден → ОТСУТСТВУЕТ
```

### Шаг 4: Определение дат (ProjectDateCalculationService)

3-уровневый fallback по приоритету:

```
1. Расходы (MonthlyExpenseData):
   "restart (2)" → TotalExpense ≠ 0 в январь-март 2024
   → Найдены! Используем.

2. Доходы (TotalIncomes):
   (пропущен — расходы уже дали результат)

3. P&L (ProjectProfitData):
   (пропущен)

Результат:
  StartPeriod = "2024 январь" (первый месяц активности)
  EndPeriod   = "2024 март"   (последний, не текущий → конкретная дата)
  // Если бы последний месяц был текущим → "В работе"
```

### Шаг 5: Определение клиента

```
1. Поиск в P&L: "Restart #2" → ClientName = "Acme Corp" ✓
2. Fallback — Доходы: искать alias → Client
   (не потребовался)
```

### Шаг 6: Запись в Google Sheets (GoogleSheetsWriteService)

```
Формируется строка:
  ["", "Restart #2", "Acme Corp", "", "", "2024 январь", "2024 март"]
         ↑              ↑          ↑   ↑       ↑              ↑
    DisplayName     Client     Type  Mgr    Start           End
                              (пусто — заполнит пользователь)

GoogleSheetsWriteService.AppendRowsAsync()
  → Service Account OAuth2
  → InsertDataOption: INSERT_ROWS
  → ValueInputOption: USER_ENTERED
```

### Шаг 7: Верификация (ProjectsMetadataVerificationStage)

```
Анализ ProjectMetadata:
  ├── "Restart #2" — IsNew = true → помечен как новый
  ├── Отсутствуют: ProjectType, SalesManager
  └── Включён в топ-3 замечаний

Результат:
  NewProjectsCount: +1
  IncompleteProjectsCount: +1
  Причина: "Новый проект (добавлен автоматически)"
```

### Итог

```
┌────────────────┐    ┌──────────────┐    ┌──────────────────┐
│  5 источников  │───→│ ProjectRegi- │───→│ ProjectsSyncStage│
│  Google Sheets │    │ stry.Build() │    │ (find missing)   │
└────────────────┘    └──────────────┘    └────────┬─────────┘
                                                   │
                      ┌──────────────┐    ┌────────v─────────┐
                      │ DateCalcula- │◄───│ Build metadata   │
                      │ tionService  │    │ (client, dates)  │
                      └──────────────┘    └────────┬─────────┘
                                                   │
                      ┌──────────────┐    ┌────────v─────────┐
                      │ Google Sheet │◄───│ WriteService     │
                      │ "Проекты"   │    │ AppendRows()     │
                      └──────────────┘    └────────┬─────────┘
                                                   │
                      ┌──────────────┐    ┌────────v─────────┐
                      │ Verification │◄───│ MetadataVerifi-  │
                      │ Results      │    │ cationStage      │
                      └──────────────┘    └──────────────────┘
```

### Участвующие классы

| Класс | Роль в сценарии |
|-------|----------------|
| `ProjectRegistry` | Кластеризация имён, canonical/display name |
| `NameCleaningService` | Нормализация для сравнения |
| `ProjectMatchingService` | 3-уровневый матчинг |
| `ProjectDateCalculationService` | Определение дат с fallback |
| `GoogleSheetsWriteService` | Запись в Google Sheets через Service Account |
| `ProjectsSheetParser` | Чтение существующих проектов |
| `ProjectsSyncStage` | Оркестрация синхронизации |
| `ProjectsMetadataVerificationStage` | Проверка полноты метаданных |
