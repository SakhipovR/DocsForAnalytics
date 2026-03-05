# ParsingPipelineApp

Веб-приложение для запуска и мониторинга pipeline аналитики. ASP.NET Core API + React TypeScript.

## Зависимости

- **BusinessAnalytics.Core** (проектная ссылка)
- **BusinessAnalytics.Verifier** (проектная ссылка)
- **BusinessAnalytics.PdfReport** (проектная ссылка)
- **SQLite** - хранение истории выполнений
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

## Архитектура

### PipelineOrchestrator

Управление выполнением pipeline:
- **Singleton lock** - только одно выполнение одновременно
- Последовательное выполнение 5 стадий
- Сохранение статуса и результатов в SQLite
- Обработка ошибок на каждом этапе

### Стадии pipeline (5 этапов)

| # | Стадия | Описание |
|---|--------|----------|
| 1 | DataLoadingStageAsync | Загрузка данных из Google Sheets. Статистика, топ-5 проектов, маржа по клиентам |
| 2 | ProjectsSyncStageAsync | Синхронизация проектов. Новые проекты, неполные поля, маржа по типам |
| 3 | VerificationStageAsync | Все 5 проверок: платежи, расходы, клиенты, доходы, P&L |
| 4 | ExportStageAsync | Генерация JSON, PDF, текстовых отчётов |
| 5 | HighlightStageAsync | Создание Excel с подсветкой расхождений |

### FileStorageService

Управление файлами результатов:
- Изолированные директории: `{BaseStoragePath}/execution-{id}/`
- Получение списка файлов
- Очистка старых выполнений (retention policy)

### Database (SQLite)

Таблица **ParsingResults**:

| Поле | Описание |
|------|----------|
| Id | Уникальный идентификатор выполнения |
| Status | Running / Completed / Failed |
| Timestamp | Время начала |
| CompletedAt | Время завершения |
| CurrentStage | Текущая стадия |
| CompletedStages / TotalStages | Прогресс |
| ConfigJson | Конфигурация запуска |
| Stage*ResultJson | Результаты каждой стадии (JSON) |
| GeneratedFilesJson | Список сгенерированных файлов |
| ErrorMessage | Сообщение об ошибке |

## DTO модели

| DTO | Назначение |
|-----|-----------|
| `PipelineExecutionRequest` | Параметры для запуска |
| `PipelineExecutionResponse` | Ответ: id выполнения |
| `PipelineStatusDto` | Статус с % прогресса |
| `PipelineResultDto` | Полные результаты |
| `StageResultDto` | Результат одной стадии |
| `GeneratedFileDto` | Информация о файле |

## Отличия от RunAnalytics (Console)

| Характеристика | RunAnalytics | ParsingPipelineApp |
|----------------|-------------|-------------------|
| Тип | Console App | Web App (API + React) |
| Стадий | 9 | 5 (объединённых) |
| Выполнение | Синхронное | Асинхронное |
| История | Нет | SQLite |
| UI | Консоль | React + TypeScript |
| Скачивание файлов | Локальные пути | REST endpoint |
| Параллельность | Нет | Singleton lock |
