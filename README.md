# Vulnerability Tracker API

> Тестовое задание Developer C# — **КарповаВ.Г._Тестовое задание_DevC#**
> Репозиторий: `https://github.com/exewer/-.-._-_DevC-./edit/master/README.md`

REST API на ASP.NET Core (.NET 8, minimal API) для учёта уязвимостей и запуска фоновых задач сканирования кода.

## Запуск

Требуется [.NET 11 SDK]

```bash
dotnet run --project src/VulnerabilityTracker.Api     # http://localhost:5080
dotnet test                                           # unit + интеграционные тесты
```

Примеры запросов — в `requests.http` (VS / Rider / VS Code REST Client) или через curl:

```bash
curl -X POST localhost:5080/api/vulnerabilities -H 'Content-Type: application/json' \
     -d '{"name":"XSS","severity":"High"}'
curl -X POST localhost:5080/api/scans -H 'Content-Type: application/json' \
     -d '{"repositoryUrl":"https://gitlab.com/group/project"}'
```

## Эндпоинты

| Метод | URL | Описание | Ответ |
|---|---|---|---|
| POST | `/api/vulnerabilities` | Тело: `{name, severity}`; `severity` = `Low\|Medium\|High\|Critical`. Дата добавления ставится сервисом | `201 {id}` |
| GET | `/api/vulnerabilities/{id}` | Полная информация | `200 {id,name,severity,createdAt}` / `404` |
| POST | `/api/scans` | Тело: `{repositoryUrl}` (абсолютный http/https URL). Запускает сканирование в фоне | `202 {id}` |
| GET | `/api/scans/{id}` | Статус: `Queued\|Running\|Completed\|Failed\|Cancelled`, `updatedAt`, `result` (если завершено), `error` (если ошибка) | `200` / `404` |
| POST | `/api/scans/{id}/cancel` | Отмена (в очереди или уже идущего). Возвращает `{id, success, cancelledAt, status}`; `success=false`, если задача уже завершена | `200` / `404` |

Ошибки возвращаются в формате RFC 7807 (`application/problem+json`): `400` — валидация, `404` — не найдено, `500` — непредвиденная ошибка (детали только в логе).

## Конфигурация (`appsettings.json` / переменные окружения)

| Параметр | Env | По умолчанию | Смысл |
|---|---|---|---|
| `Scanning:MaxConcurrentScans` | `Scanning__MaxConcurrentScans` | `2` | Сколько сканирований идёт параллельно; остальные ждут в статусе `Queued` |
| `Scanning:SimulatedScanDurationMs` | `Scanning__SimulatedScanDurationMs` | `10000` | Длительность имитации сканирования |

Порт: `ASPNETCORE_URLS=http://localhost:8080 dotnet run --project src/VulnerabilityTracker.Api`.

## Где хранятся данные и файлы

Постоянного хранилища нет: уязвимости и задачи сканирования лежат **только в памяти** процесса, при перезапуске всё сбрасывается. Файлов сервис не создаёт. Логи пишутся в консоль (stdout), каждый запрос логируется строкой `METHOD /path -> status in N ms`, плюс события жизненного цикла сканов.

## Устройство

- `Services/ScanManager` — очередь и выполнение сканов: `SemaphoreSlim` ограничивает параллелизм, у каждой задачи свой `CancellationTokenSource` — отмена снимает задачу с ожидания слота либо прерывает сканер. Переходы состояний атомарны (`ScanJob.TryMoveTo`), завершённые состояния финальны, поэтому гонка «отмена vs завершение» разрешается корректно.
- `Services/IScanner` — точка расширения. Встроенный `SimulatedScanner` только имитирует работу (детерминированные «находки» по URL); URL, оканчивающийся на `/fail`, даёт статус `Failed`. Реальный сканер (клонирование репозитория, запуск анализатора) подключается заменой реализации в DI.
- Ошибки — единый обработчик в `Program.cs`; логирование запросов — middleware там же.

## Допущения

- Сканирование имитируется: в задании не указано, каким инструментом искать уязвимости.
- Отмена — `POST /api/scans/{id}/cancel` (а не DELETE), т.к. запись о задаче сохраняется со статусом `Cancelled`.
