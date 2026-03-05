# Тесты

## BusinessAnalytics.Core.Tests

Фреймворк: NUnit

### Покрытие

| Тестовый класс | Кол-во | Что проверяет |
|---------------|--------|---------------|
| ExpensePLComparisonTests | 10 | Сравнение расходов с P&L: матчинг, регистронезависимость, пустые данные |
| ExpensesParserTests | 3 | Парсинг "Итоги месяцев": блоки, итоги, прямые расходы |
| PaymentsParserTests | 4 | Парсинг "Выплаты": валидные строки, пропуск пустых, все поля |
| ProjectsSheetParserTests | 6 | Парсинг "Проекты": все поля, пропуск пустых, недостающие поля |
| PLValidationTests | 5 | Валидация P&L: совпадения, расхождения, общие периоды |
| ClientGroupingTests | 3 | Группировка по клиентам, экспорт JSON, "Без клиента" |
| NameCleaningServiceTests | 11 | Нормализация: ключевые слова, версии, lowercase, спецсимволы |
| ConstantExpensesTests | - | Постоянные расходы |
| DebtorBalanceTests | - | Дебиторская задолженность |
| ExpenseAccountingTest | - | Учёт расходов |
| ProjectProfitTests | - | Расчёт прибыли проекта |
| ProjectNameNormalizationTest | - | Нормализация имён проектов |
| RegexFunctionalityTests | - | Регулярные выражения |
| ProjectDateCalculationServiceTests | - | Расчёт дат проектов |
| PeriodCalculationServiceTests | - | Расчёт общих периодов |
| ProjectRegistryTests | - | Реестр проектов |
| ProjectTypeMarginServiceTests | - | Маржа по типам проектов |

## BusinessAnalytics.Verifier.Tests

| Тестовый класс | Кол-во | Что проверяет |
|---------------|--------|---------------|
| IncomeDiscrepancyVerifierTests | 13 | Расхождения доходов: агрегирование, регистр, null, топ-N |
| PLClientsCompatibilityVerifierTests | 13 | Несопоставленные клиенты: поиск, сортировка по дате, граничные случаи |

## BusinessAnalytics.PdfReport.Tests

| Тестовый класс | Кол-во | Что проверяет |
|---------------|--------|---------------|
| ClientCalculationServiceTests | 4 | Топ по маржи, сортировка, пустые списки |
| ReportGenerationServiceTests | 2 | Генерация PDF из JSON, валидность PDF заголовка |

## Ключевые проверяемые сценарии

1. **Нормализация имён** - разные варианты одного проекта распознаются корректно
2. **Нечувствительность к регистру** - "Клиент" = "клиент" = "КЛИЕНТ"
3. **Обработка пустых данных** - null, пустые строки, пустые коллекции
4. **Агрегирование** - несколько платежей одного клиента суммируются правильно
5. **Фильтрация по периодам** - сравнение только в пересечении доступных периодов
6. **Генерация отчётов** - PDF содержит валидный заголовок (%PDF)
