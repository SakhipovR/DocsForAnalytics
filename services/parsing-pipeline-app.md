# ParsingPipelineApp

Веб-приложение для запуска и мониторинга pipeline аналитики. ASP.NET Core API + React TypeScript.

## Зависимости

- **BusinessAnalytics.Core** (проектная ссылка)
- **BusinessAnalytics.Verifier** (проектная ссылка)
- **BusinessAnalytics.PdfReport** (проектная ссылка)
- **Microsoft.EntityFrameworkCore.Sqlite** 8.0.0 - ORM и хранение истории
- **Serilog.AspNetCore** 8.0.3 - структурированное логирование
- **Swashbuckle.AspNetCore** 6.5.0 - Swagger/OpenAPI
- **Vite.AspNetCore** - интеграция с React dev-сервером
- Платформа: .NET 8.0, ASP.NET Core

## REST API

### ParsingController

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/api/parsing/execute-pipeline` | Запуск pipeline |
| GET | `/api/parsing/status/{id}` | Статус выполнения (с % прогресса) |
| GET | `/api/parsing/result/{id}` | Полные результаты |
| GET | `/api/parsing/download/{id}/{file}` | Скачивание сгенерированного файла |
| GET | `/api/parsing/executions` | Список выполнений (пагинация) |
| GET | `/api/parsing/latest` | Результаты последнего выполнения |
| GET | `/api/parsing/default-config` | Конфигурация по умолчанию |

### ParsingTestController

Управление тест-кейсами для валидации результатов pipeline.

| Метод | Endpoint | Описание |
|-------|----------|----------|
| GET | `/api/parsing-tests` | Список всех тест-кейсов |
| POST | `/api/parsing-tests` | Создание нового тест-кейса |
| PUT | `/api/parsing-tests/{id}` | Обновление тест-кейса |
| DELETE | `/api/parsing-tests/{id}` | Удаление тест-кейса |
| POST | `/api/parsing-tests/execute/{executionId}` | Запуск тестов против результатов выполнения |
| GET | `/api/parsing-tests/results/{executionId}` | Результаты тестов для выполнения |

## Архитектура

### PipelineOrchestrator

Управление выполнением pipeline:
- **Singleton lock** (SemaphoreSlim) - только одно выполнение одновременно
- Последовательное выполнение 5 стадий
- Создание изолированной директории для каждого запуска
- Сохранение статуса и результатов каждой стадии в SQLite как JSON
- Обработка ошибок на каждом этапе

### Стадии pipeline (5 этапов)

Каждая стадия реализует `IPipelineStageAsync` и оборачивает синхронные стадии из RunAnalytics через `Task.Run()`.

| # | Стадия | Описание |
|---|--------|----------|
| 1 | DataLoadingStageAsync | Загрузка данных из Google Sheets. Статистика, топ-5 проектов, маржа по клиентам |
| 2 | ProjectsSyncStageAsync | Синхронизация проектов. Новые проекты, неполные поля, маржа по типам |
| 3 | VerificationStageAsync | Все 5 проверок: платежи, расходы, клиенты, доходы, P&L |
| 4 | ExportStageAsync | Генерация JSON, PDF, текстовых отчётов |
| 5 | HighlightStageAsync | Создание Excel с подсветкой расхождений |

### FileStorageService

Управление файлами результатов:
- Базовый путь: `{temp}/pipeline-results/` (настраивается)
- Изолированные директории: `execution-{id}/`
- Методы: `CreateExecutionDirectory()`, `GetFileAsync()`, `GetExecutionFiles()`, `CleanupOldExecutions()`

### ParsingTestService

Подсистема тестирования результатов pipeline:
- Управление тест-кейсами (CRUD)
- Выполнение тестов: сравнение фактических значений с ожидаемыми
- Настраиваемый допуск (tolerance, по умолчанию 0.01)
- Типы метрик: margin, income, expense и др.

### GlobalExceptionHandlerMiddleware

Централизованная обработка ошибок:
- Перехват необработанных исключений
- Структурированное логирование через Serilog
- Формирование JSON-ответа с кодом 500

### Database (SQLite)

**ApplicationDbContext** — EF Core контекст с 3 таблицами.

#### Таблица ParsingResults

| Поле | Описание |
|------|----------|
| Id | Уникальный идентификатор выполнения |
| Status | Running / Completed / Failed |
| Timestamp | Время начала (UTC) |
| CompletedAt | Время завершения |
| CurrentStage | Текущая стадия |
| CompletedStages / TotalStages | Прогресс |
| ConfigJson | Конфигурация запуска (GoogleSheetsConfig) |
| DataLoadingResult | Результат стадии загрузки (JSON) |
| ProjectsSyncResult | Результат стадии синхронизации (JSON) |
| VerificationResult | Результат стадии верификации (JSON) |
| ExportResult | Результат стадии экспорта (JSON) |
| HighlightResult | Результат стадии подсветки (JSON) |
| GeneratedFilesJson | Список сгенерированных файлов |
| ErrorMessage | Сообщение об ошибке |

#### Таблица ParsingTestCases

| Поле | Описание |
|------|----------|
| Id | PK |
| ProjectName | Имя проекта для проверки |
| Year, Month | Период |
| MetricType | Тип метрики (margin, income, expense и т.д.) |
| ExpectedValue | Ожидаемое значение |
| Tolerance | Допуск (по умолчанию 0.01) |
| IsActive | Флаг активности |
| Description | Описание тест-кейса |
| CreatedAt | Дата создания |

#### Таблица ParsingTestResults

| Поле | Описание |
|------|----------|
| Id | PK |
| TestCaseId | FK → ParsingTestCases |
| ExecutionId | FK → ParsingResults |
| Passed | Результат (true/false) |
| ActualValue | Фактическое значение |
| ExpectedValue | Ожидаемое значение |
| ErrorMessage | Сообщение об ошибке |
| VerifiedAt | Время проверки |

## DTO модели

| DTO | Назначение |
|-----|-----------|
| `PipelineExecutionRequest` | Параметры для запуска |
| `PipelineExecutionResponse` | Ответ: id выполнения |
| `PipelineStatusDto` | Статус с % прогресса |
| `PipelineResultDto` | Полные результаты |
| `StageResultDto` | Результат одной стадии |
| `GeneratedFileDto` | Информация о файле |

## Фронтенд (React + TypeScript)

Расположение: `ParsingPipelineApp.Api/ClientApp/`
Сборка: **Vite** (HMR dev-сервер на порту 3000)

### Компоненты

| Компонент | Назначение |
|-----------|-----------|
| `App.tsx` | Маршрутизация приложения |
| `PipelineExecutor.tsx` | Ввод конфигурации и запуск pipeline |
| `ExecutionStatus.tsx` | Отображение прогресса в реальном времени |
| `PipelineResults.tsx` | Визуализация результатов |
| `ExecutionDetails.tsx` | Детали выполнения |
| `ExecutionsList.tsx` | История выполнений |
| `ParsingTests.tsx` | Управление тест-кейсами |
| `Metrics.tsx` | Ключевые метрики |
| `CustomerMargins.tsx` | Анализ маржинальности клиентов |
| `ProjectTypeMargins.tsx` | Анализ маржинальности по типам проектов |
| `ClientDetails.tsx` | Детали по клиенту (drill-down) |

### HTTP клиент

`api.ts` — централизованный сервис для запросов к backend API.

## Запуск

### Development (`start-dev.cmd`)
1. Остановка старых процессов (dotnet, node)
2. Проверка/установка frontend-зависимостей
3. Backend: `dotnet watch run` (авто-перезапуск при изменении .cs)
4. Frontend: `npm run dev` (Vite HMR, порт 3000)
5. Открытие http://localhost:3000

### Production (`start.cmd`)
1. Установка frontend-зависимостей
2. Сборка React-бандла: `npm run build` → wwwroot/
3. Запуск ASP.NET с встроенным фронтендом
4. Доступ: http://localhost:5134

## Отличия от RunAnalytics (Console)

| Характеристика | RunAnalytics | ParsingPipelineApp |
|----------------|-------------|-------------------|
| Тип | Console App | Web App (API + React) |
| Стадий | 10 | 5 (объединённых) |
| Выполнение | Синхронное | Асинхронное (Task.Run) |
| История | Нет | SQLite |
| UI | Консоль | React + TypeScript |
| Скачивание файлов | Локальные пути | REST endpoint |
| Параллельность | Нет | Singleton lock |
| Тестирование | Нет | Подсистема тест-кейсов |
| Логирование | Console.WriteLine | Serilog |
