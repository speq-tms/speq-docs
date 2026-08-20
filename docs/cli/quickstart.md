# Quickstart: первый зелёный прогон

С нуля до проходящего теста. Ничего знать заранее не нужно; открывать `speq-examples` не потребуется.

Тест пишется против [JSONPlaceholder](https://jsonplaceholder.typicode.com) — публичного тестового API,
доступного без ключей и регистрации. Нужен только интернет.

Всё, что ниже, копируется и вставляется как есть.

---

## 1. Установка

### macOS — Homebrew

```bash
brew tap speq-tms/tap https://github.com/speq-tms/homebrew-tap
brew install speq
```

Формула собрана только для macOS (Apple Silicon и Intel). На Linux ставьте бинарник напрямую.

### macOS и Linux — бинарник

```bash
# macOS, Apple Silicon
curl -L -o speq.tar.gz https://github.com/speq-tms/speq-cli/releases/download/v1.0.0/speq-darwin-aarch64.tar.gz

# macOS, Intel
curl -L -o speq.tar.gz https://github.com/speq-tms/speq-cli/releases/download/v1.0.0/speq-darwin-x86_64.tar.gz

# Linux, x86_64
curl -L -o speq.tar.gz https://github.com/speq-tms/speq-cli/releases/download/v1.0.0/speq-linux-x86_64.tar.gz
```

В архиве один файл — исполняемый `speq`. Распакуйте и положите в `PATH`:

```bash
tar -xzf speq.tar.gz
sudo mv speq /usr/local/bin/
```

### Windows

Скачайте `speq-windows-x86_64.zip` со
[страницы релиза](https://github.com/speq-tms/speq-cli/releases/tag/v1.0.0), распакуйте и добавьте
каталог с `speq.exe` в `PATH`.

### Проверка

```bash
speq version
```

```text
speq 1.0.0
```

---

## 2. Создание проекта

```bash
mkdir my-api-tests
cd my-api-tests
speq init --mode test-repo
```

```text
Initialized speq (test-repo) at /path/to/my-api-tests
```

> Флаг `--mode test-repo` обязателен, если тесты живут в отдельном репозитории. **Без него режим по
> умолчанию — `in-repo`**, и всё окажется в подкаталоге `.speq/`. Подробнее — в
> [project-layout.md](project-layout.md#два-режима-репозитория).

Получилось:

```text
my-api-tests/
├── manifest.yaml            # версия, имя проекта, окружение по умолчанию, раскладка каталогов
├── environments/
│   └── ci.yaml              # заготовка окружения
├── suites/
│   └── smoke.yaml           # тест-пример
├── modules/                 # пусто
├── schemas/                 # пусто
└── reports/                 # сюда пишутся результаты
```

Проект уже рабочий: заготовка настроена на httpbin.org. Мы заменим её на JSONPlaceholder, чтобы
написать содержательные проверки — у httpbin `/status/200` пустое тело, и проверять в нём нечего.

---

## 3. Настройка окружения

Замените `environments/ci.yaml`:

```yaml
name: ci
baseUrl: https://jsonplaceholder.typicode.com
```

`baseUrl` подставляется слева ко всем относительным URL в шагах. Имя окружения — это имя файла:
`ci.yaml` запускается как `--env ci`.

---

## 4. Первый тест

Замените `suites/smoke.yaml`:

```yaml
id: "posts.get-by-id"
title: "GET /posts/1 возвращает пост с ожидаемым содержимым"
tags: [smoke]
steps:
  - type: api
    name: "GET /posts/1"
    method: GET
    url: "/posts/1"
    assert:
      - type: status
        expected: 200
      - type: json
        path: "$.id"
        expected: 1
      - type: json
        path: "$.userId"
        expected: 1
      - type: exists
        path: "$.title"
```

Построчно:

| Строка | Что делает |
| --- | --- |
| `id` | Идентификатор теста в отчётах. Должен быть уникальным |
| `title` | Человекочитаемое название |
| `tags` | Метки для фильтрации: `speq run --tags smoke` |
| `type: api` | Шаг выполняет HTTP-запрос |
| `name` | Название шага в отчёте; обязательное поле |
| `url: "/posts/1"` | Относительный путь — превратится в `https://jsonplaceholder.typicode.com/posts/1` |
| `type: status` | Сравнивает HTTP-код с `expected` |
| `type: json` | Сравнивает значение по пути `path` с `expected`, строго по типу |
| `type: exists` | Проверяет, что поле в ответе есть, не заглядывая в значение |

Путь `$.id` — это ключ `id` в корне ответа. Вложенность записывается через точку: `$.address.geo.lat`.
К элементам массива обратиться нельзя — это ограничение описано в
[dsl.md](dsl.md#пути-к-полям-ответа).

Перед запросами полезно проверить, что всё разбирается:

```bash
speq validate
```

```text
Validation passed: mode=test-repo, root=/path/to/my-api-tests, tests=1
```

---

## 5. Прогон

```bash
speq run --env ci --report all
```

```json
{
  "ok": true,
  "reports": {
    "allure": "/path/to/my-api-tests/reports/allure",
    "summary": "/path/to/my-api-tests/reports/results/summary.json"
  },
  "status": "passed",
  "totals": {
    "error": 0,
    "failed": 0,
    "passed": 1,
    "pending": 0,
    "total": 1
  }
}
```

`"failed": 0` — прогон зелёный.

> `--report all` пишет и `summary.json`, и данные для Allure. **Режим по умолчанию — `allure`**, при нём
> `summary.json` не создаётся. Для `summary.json` нужен `--report summary` или `--report all`.

---

## 6. Чтение результата

```bash
cat reports/results/summary.json
```

```json
{
  "durationMs": 215,
  "startedAtMs": 1787247840182,
  "status": "passed",
  "tests": [
    {
      "durationMs": 214,
      "id": "posts.get-by-id",
      "status": "passed"
    }
  ],
  "totals": {
    "failed": 0,
    "passed": 1,
    "total": 1
  }
}
```

| Поле | Смысл |
| --- | --- |
| `status` | `passed`, если нет ни упавших, ни ошибочных тестов |
| `totals.passed` / `failed` / `total` | Счётчики по тестам |
| `totals.pending` / `error` | Появляются, только если такие тесты были |
| `tests[]` | По записи на тест: `id`, `status`, `durationMs` и `message` при падении |

### Как выглядит падение

Поменяйте `expected: 1` на `expected: 2` и запустите снова:

```json
{
  "status": "failed",
  "tests": [
    {
      "durationMs": 183,
      "id": "posts.get-by-id",
      "message": "json assertion failed at '$.id': expected 2, got 1; json assertion failed at '$.userId': expected 2, got 1",
      "status": "failed"
    }
  ],
  "totals": {
    "failed": 1,
    "passed": 0,
    "total": 1
  }
}
```

Все упавшие ассерты одного шага перечисляются в одном `message`.

### Коды выхода

Именно на них смотрит CI:

| Код | Когда |
| --- | --- |
| `0` | Зелено. `pending`-тесты код не портят |
| `1` | Есть упавшие тесты или ошибки выполнения, либо покрытие ниже `coverage.fail_below` |
| `2` | Ошибка валидации, конфигурации или аргументов |
| `3` | Внутренняя ошибка |

```bash
speq run --env ci --report all
echo $?      # 0 на зелёном прогоне, 1 на красном
```

---

## 7. Дальше

Схема есть — дальше растёт только содержание тестов.

| Что нужно | Куда смотреть |
| --- | --- |
| Все семь типов ассертов, включая проверку по JSON Schema | [dsl.md](dsl.md#ассерты) |
| Переменные и подстановка `{{...}}` | [dsl.md](dsl.md#переменные-и-подстановка) |
| Генерация данных: `uuid`, `email`, случайные строки и числа | [dsl.md](dsl.md#генерация-данных) |
| Фикстуры — заготовки тел запросов | [dsl.md](dsl.md#фикстуры) |
| Модули: переиспользуемые действия и передача значений между шагами | [dsl.md](dsl.md#модули) |
| Общие хуки для группы тестов | [dsl.md](dsl.md#сьюты-и-inityaml) |
| Ожидание нужного состояния перед проверкой | [dsl.md](dsl.md#условия-и-ожидание) |
| Пометка нереализованного через `status: pending` | [dsl.md](dsl.md#atdd-status-pending) |
| Повторы, покрытие по OpenAPI и остальные поля манифеста | [project-layout.md](project-layout.md#справочник-манифеста) |
| Все команды и флаги | [README.md](README.md) |

Готовый проект с тем же тестом — `speq-examples/quickstart`. Более крупный набор,
покрывающий все возможности, — `speq-examples/test-repo-mode-jsonplaceholder`.

Прежде чем вкладываться в большой набор, стоит прочитать
[чего DSL не умеет](dsl.md#чего-dsl-не-умеет): несколько ограничений выглядят неожиданно, и лучше
узнать о них сейчас, а не на середине пути.
