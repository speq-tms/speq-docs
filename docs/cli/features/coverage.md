# SPEQ CLI Feature: API Coverage

## Цель

Описать механизм подсчёта и репортинга coverage эндпоинтов: какая доля задокументированных в Swagger/OpenAPI эндпоинтов покрыта SPEQ-тестами.

Coverage в SPEQ — это не code coverage. Это **contract coverage**: насколько API-контракт (описанный в OpenAPI-спецификации) покрыт существующими acceptance-тестами.

---

## Мотивация

SPEQ может находиться непосредственно в репозитории с кодом. Это даёт прямой доступ к Swagger/OpenAPI-документации проекта без внешних сервисов. Coverage-отчёт отвечает на вопрос: "Какие эндпоинты у нас протестированы, а какие нет?" — и является естественным продолжением [ATDD-флоу](atdd-flow.md): pending-тесты вносят вклад в coverage, а непокрытые эндпоинты становятся очевидными кандидатами для новых тестов.

Качество coverage напрямую зависит от качества OpenAPI-документации: чем полнее она ведётся — тем точнее отчёт.

---

## Scope v1.0.0 RC

Уровень coverage для v1.0.0 RC: **endpoint coverage** (path + method).

Единица измерения — пара `(HTTP method, path)`. Эндпоинт считается покрытым, если хотя бы один SPEQ-тест выполняет запрос на этот `method + path`.

Более детальные уровни (status codes, query parameters, request body fields) — out of scope для v1.0.0 RC и рассматриваются в следующем milestone.

---

## Конфигурация

### `speq.yaml` / project config

```yaml
coverage:
  enabled: true
  openapi: "./docs/openapi.yaml"   # путь к OpenAPI-спецификации относительно speq.yaml
  report: true                      # включать в summary-отчёт
  fail_below: null                  # опционально: минимальный порог в % (null = не фейлить)
```

Поле `openapi` обязательно при `enabled: true`. Поддерживаются форматы OpenAPI 3.x и Swagger 2.x (JSON и YAML).

### Альтернативный путь через CLI-флаг

```bash
speq run --coverage --openapi ./docs/openapi.yaml
```

CLI-флаги имеют приоритет над `speq.yaml`.

---

## Алгоритм подсчёта

### 1. Парсинг OpenAPI

CLI читает указанный файл и извлекает полный список пар `(method, path)` из секции `paths`. Игнорируются:

- `x-internal: true` — приватные/внутренние эндпоинты (опционально, через конфиг).
- deprecated-эндпоинты с явным `deprecated: true` — по умолчанию включены, но могут быть исключены конфигом.

Результат: список `CoveredEndpoint { method: String, path: String }`.

### 2. Нормализация путей

OpenAPI использует path templates: `/users/{id}`. SPEQ-тест использует реальные URL: `/users/42` или `{{ base_url }}/users/{{ user_id }}`.

Нормализация:

1. Из URL теста удаляется `base_url` (определяется через environment).
2. Сегменты пути, соответствующие шаблонным переменным OpenAPI, матчатся по regex.
3. Path templates из OpenAPI компилируются в regex: `/users/{id}` → `^/users/[^/]+$`.

Пример матчинга:

| OpenAPI path | Test URL | Match |
|---|---|---|
| `/users/{id}` | `/users/42` | ✓ |
| `/users/{id}` | `/users/42/posts` | ✗ |
| `/posts/{postId}/comments` | `/posts/1/comments` | ✓ |

### 3. Сбор покрытых эндпоинтов

По результатам прогона CLI собирает множество `executed_endpoints`: все пары `(method, normalized_path)` из фактически выполненных запросов.

Pending-тесты (`status: pending`) **не вносят вклад** в `executed_endpoints` — они не выполняют HTTP-запрос.

### 4. Вычисление coverage

```
covered = intersection(openapi_endpoints, executed_endpoints)
coverage_pct = len(covered) / len(openapi_endpoints) * 100
uncovered = openapi_endpoints - covered
```

---

## Репортинг

### Консольный вывод

```
API Coverage: 18/24 endpoints (75.0%)

Covered (18):
  ✓ GET    /users
  ✓ POST   /users
  ✓ GET    /users/{id}
  ✓ PUT    /users/{id}
  ...

Not covered (6):
  ✗ DELETE /users/{id}
  ✗ GET    /users/{id}/posts
  ✗ POST   /posts
  ✗ GET    /posts/{id}
  ✗ PUT    /posts/{id}
  ✗ DELETE /posts/{id}
```

### JSON summary

Поле `coverage` добавляется в `reports/results/summary.json`:

```json
{
  "passed": 12,
  "failed": 0,
  "error": 0,
  "pending": 3,
  "total": 15,
  "coverage": {
    "enabled": true,
    "total_endpoints": 24,
    "covered_endpoints": 18,
    "percentage": 75.0,
    "uncovered": [
      { "method": "DELETE", "path": "/users/{id}" },
      { "method": "GET",    "path": "/users/{id}/posts" },
      { "method": "POST",   "path": "/posts" },
      { "method": "GET",    "path": "/posts/{id}" },
      { "method": "PUT",    "path": "/posts/{id}" },
      { "method": "DELETE", "path": "/posts/{id}" }
    ]
  }
}
```

### Allure

Coverage-метрики добавляются как environment properties в Allure-репорт:

```
api_coverage_total=24
api_coverage_covered=18
api_coverage_pct=75.0
```

Uncovered-эндпоинты могут быть представлены как отдельный "fake" suite `[Coverage] Uncovered endpoints` со статусом `skipped` — для наглядности в Allure UI. Реализация этого опциональна для v1.0.0 RC.

---

## Fail threshold

Если задан `fail_below` в конфиге:

```yaml
coverage:
  fail_below: 80
```

CLI завершается с non-zero exit code, если `coverage_pct < fail_below`, выводя:

```
Coverage gate failed: 75.0% < 80% threshold
```

По умолчанию `fail_below: null` — coverage не влияет на exit code.

---

## Требования к OpenAPI-документации

Для корректной работы coverage необходимо:

- OpenAPI-файл должен быть валидным (OpenAPI 3.x или Swagger 2.x).
- Все эндпоинты, которые нужно отслеживать, должны быть описаны в секции `paths`.
- Чем полнее описание (ответы, параметры) — тем выше ценность для будущих уровней coverage.

CLI не валидирует корректность OpenAPI-схемы — только читает `paths`. При невалидном файле или отсутствии секции `paths` CLI выводит предупреждение и продолжает прогон без coverage-репорта.

---

## Взаимодействие с [ATDD](atdd-flow.md)

Coverage органично дополняет ATDD-флоу:

- Тестировщик видит непокрытые эндпоинты → создаёт pending-тесты для них.
- Pending-тесты не вносят вклад в coverage (нет реального запроса).
- Разработчик реализует фичу → тесты переходят в passed → coverage растёт.

Таким образом coverage является метрикой зрелости тестового покрытия, а не просто счётчиком выполненных запросов.

---

## Delivery

### Затронутые репозитории

- `speq-cli`: реализация парсинга OpenAPI, нормализации путей, подсчёта и репортинга.
- `speq-contracts`: схема поля `coverage` в `summary.json`.
- `speq-examples/test-repo-mode-jsonplaceholder`: AT-пример с OpenAPI-файлом и включённым coverage.
- `speq-docs`: обновление конфигурационного reference (поле `coverage` в `speq.yaml`).

### Delivery branches (от RC v1.0.0)

- `feat/cli-coverage-openapi`
- `chore/contracts-coverage-schema`
- `chore/examples-coverage-at`

---

## Acceptance criteria для v1.0.0 RC

- `speq run` с `coverage.enabled: true` и валидным `openapi` файлом выводит coverage-секцию.
- Coverage считается как endpoint coverage (method + path).
- Path templates из OpenAPI корректно матчатся против реальных URL тестов.
- Pending-тесты не вносят вклад в covered-множество.
- JSON summary содержит поле `coverage` с `total_endpoints`, `covered_endpoints`, `percentage`, `uncovered`.
- `fail_below` завершает CLI с non-zero exit code при не прохождении порога.
- При невалидном или отсутствующем OpenAPI-файле CLI выводит warning и продолжает без coverage.
- AT-пример в `speq-examples` демонстрирует coverage с реальным OpenAPI JSONPlaceholder (или mock-спецификацией).

---

## Out of scope для v1.0.0 RC

- Coverage по status codes (200, 404, 400...) — следующий milestone.
- Coverage по query parameters и request body fields — следующий milestone.
- Автоматическая генерация pending-тестов для непокрытых эндпоинтов — рассматривается отдельно.
- Diff coverage между прогонами (история покрытия во времени) — post-MVP.
- Поддержка remote OpenAPI URL (только local file path для v1.0.0 RC).
- Валидация корректности OpenAPI-схемы — вне scope CLI.
