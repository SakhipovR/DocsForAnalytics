# BusinessAnalytics.Verifier

Модуль верификации и сверки финансовых данных между различными источниками.

## Зависимости

- **BusinessAnalytics.Core** (проектная ссылка)
- **NPOI** 2.7.1 - работа с Excel XLSX
- Платформа: .NET 8.0

## Классы

### IncomeDiscrepancyVerifier

Сравнение сумм доходов между листом "Доходы" и листом "Факт".

**Алгоритм:**
1. Проверяет наличие дат в данных (`HasDateInformation`)
2. Если есть даты - определяет общие периоды и сравнивает только внутри пересечения
3. Если нет дат - агрегирует по клиентам и сравнивает суммарно (обратная совместимость)
4. Нормализует имена клиентов через `NameCleaningService`
5. Группирует по составному ключу (клиент, год, месяц)

**Методы:**
- `FindDiscrepancies(incomes, plProjects)` → `List<IncomeDiscrepancyInfo>`
- `GetTopDiscrepancies(discrepancies, top=3)` - топ по величине расхождения

### PLClientsCompatibilityVerifier

Поиск клиентов из "Доходов", отсутствующих в "Факте".

**Методы:**
- `FindUnmatchedClients(incomes, plProjects)` → `List<UnmatchedClientInfo>`
- `GetTopUnmatchedClients(clients, top=10)` - сортировка по дате последнего платежа

### ExpensesPLFactReconciliationService

Поиск проектов из "Расходов", не найденных в P&L.

**Зависимости:** `IProjectMatcher`

**Методы:**
- `FindUnmatchedExpenseProjects(expenses)` → `List<UnmatchedExpenseProjectInfo>`
- `GetTopUnmatchedExpenseProjects(projects, top=5)` - сортировка по сумме расходов

### PaymentsPLFactReconciliationService

Поиск проектов из "Выплат", не найденных в P&L.

**Зависимости:** `IProjectMatcher`

**Методы:**
- `FindUnmatchedPaymentProjects(payments)` → `List<UnmatchedPaymentProjectInfo>`
- `GetTopUnmatchedPaymentProjects(projects, top=5)` - сортировка по сумме

### ExcelDiscrepancyHighlighter

Визуализация расхождений в Excel (NPOI).

**Методы:**
- `HighlightIncomeDiscrepancies(xlsxPath, discrepancies)` - красная подсветка в листе "Доходы"
- `HighlightFactDiscrepancies(xlsxPath, discrepancies)` - жёлтая подсветка в листе "Факт"

## Модели

| Модель | Поля | Назначение |
|--------|------|-----------|
| `IncomeDiscrepancyInfo` | ClientName, Year, Month, IncomeFromIncomes, IncomeFromFact, Discrepancy | Расхождение доходов |
| `UnmatchedClientInfo` | ClientName, DisplayClientName, LatestPaymentDate, TotalIncome | Несопоставленный клиент |
| `UnmatchedExpenseProjectInfo` | ProjectName, DisplayProjectName, TotalExpense | Несопоставленный проект расходов |
| `UnmatchedPaymentProjectInfo` | ProjectName, DisplayProjectName, TotalAmount, PaymentCount | Несопоставленный проект выплат |

## Сводка проверок

```
Доходы (TotalIncomes[])  ──┬── vs ── PLStructure ──── IncomeDiscrepancyVerifier
                           │                           (расхождения сумм)
                           │
                           └── vs ── PLStructure ──── PLClientsCompatibilityVerifier
                                                       (отсутствующие клиенты)

Расходы (MonthlyExpenseData[]) ── vs ── ProjectRegistry ── ExpensesPLFactReconciliationService
                                                            (неизвестные проекты)

Выплаты (PaymentData[]) ── vs ── ProjectRegistry ── PaymentsPLFactReconciliationService
                                                      (неизвестные проекты)
```
