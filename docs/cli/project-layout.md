# Раскладка проекта `.speq` и справочник манифеста

Что такое проект SPEQ, из каких файлов он состоит, как CLI находит его корень и какие поля понимает
манифест.

Источник истины — код: `speq-cli/src/cli/discovery.rs`, `src/cli/init.rs`, `src/manifest/mod.rs`.
Расхождение этого документа с кодом считается багом документа.

Справочник самого DSL и quickstart добавляются отдельно (#10, #11); ссылки на них появятся здесь, когда
эти документы будут существовать.

---

## Два режима репозитория

Проект существует в одном из двух режимов. Отличие только одно: где лежит корень `.speq`.

| Режим | Корень | Когда выбирать |
| --- | --- | --- |
| `in-repo` | каталог `.speq/` внутри репозитория продукта | тесты живут рядом с кодом, который проверяют |
| `test-repo` | корень самого репозитория | отдельный репозиторий, в котором нет ничего, кроме тестов |

Никакого другого поведения режим не меняет: набор артефактов, синтаксис DSL и команды одинаковы.

```text
in-repo                          test-repo
repo/                            api-tests/
├── src/                         ├── manifest.yaml
├── package.json                 ├── environments/
└── .speq/                       ├── suites/
    ├── manifest.yaml            ├── modules/
    ├── environments/            ├── fixtures/
    ├── suites/                  ├── schemas/
    ├── modules/                 └── reports/
    ├── fixtures/
    ├── schemas/
    └── reports/
```

---

## Как CLI находит корень

`discover_speq_root` (`src/cli/discovery.rs`) выполняет ровно три проверки:

1. Если передан `--speq-root <path>` — используется он. Относительный путь разрешается от текущего
   каталога. Режим в выводе команд при этом обозначается как `explicit`, а существование каталога на
   этом шаге не проверяется.
2. Иначе, если в **текущем каталоге** есть `.speq/manifest.yaml` и `.speq/suites/` — это `in-repo`.
3. Иначе, если в **текущем каталоге** есть `manifest.yaml` и `suites/` — это `test-repo`.

Если ни одна проверка не сработала:

```text
speq root not found; run 'speq init' or pass --speq-root
```

Если сработали обе (в каталоге есть и `manifest.yaml` + `suites/`, и `.speq/manifest.yaml` +
`.speq/suites/`), CLI отказывается угадывать:

```text
ambiguous speq layout: both .speq and repository root look valid, pass --speq-root
```

> **Поиск вверх по дереву не выполняется.** Вопреки распространённому ожиданию — и вопреки тому, как
> ведут себя `git`, `cargo` или `npm`, — CLI смотрит только в текущий каталог. Запуск из
> `<проект>/nested/deeper` завершится ошибкой `speq root not found`, хотя корень существует двумя
> уровнями выше. Запускайте команды из корня проекта либо передавайте `--speq-root`.
> Обсуждение изменения: [speq-tms/speq-docs#25](https://github.com/speq-tms/speq-docs/issues/25).

Признак корня — именно пара «манифест + каталог сьют». Одного `manifest.yaml` недостаточно.

---

## Артефакты проекта

| Артефакт | Расположение по умолчанию | Ключ манифеста | Обязателен |
| --- | --- | --- | --- |
| Манифест | `manifest.yaml` | — | да |
| Окружения | `environments/` | `environmentsDir` | для `run` — да, нужен файл выбранного окружения |
| Сьюты и тесты | `suites/` | `suitesDir` | да, каталог должен существовать |
| Модули | `modules/` | `modulesDir` | нет |
| Фикстуры | `fixtures/` | `fixturesDir` | нет |
| JSON-схемы для ассертов | `schemas/` | `schemasDir` | нет |
| Отчёты | `reports/` | `reportsDir` | создаётся при прогоне |

Значения ключей — пути относительно корня проекта.

**Манифест** — единственный файл с фиксированным именем и расположением. Всё остальное перемещается
соответствующим ключом.

**Окружения.** Один файл на окружение; имя файла без расширения и есть имя окружения для `--env`.
Ключ `baseUrl` задаёт префикс относительных URL, любой другой ключ верхнего уровня становится
переменной с тем же именем.

**Сьюты и тесты.** `suites/` — дерево произвольной глубины. Каждый `.yaml`/`.yml` в нём считается
тест-спекой, кроме файлов `init.yaml` и `init.yml`: это конфигурация сьюты, а не тест.

**Модули** — переиспользуемые действия, которые тест подключает через `imports` и вызывает шагом
`use`. **Фикстуры** — заготовки тел запросов для `bodyFromFixture`. **Схемы** — JSON-схемы, на которые
ссылается ассерт `type: schema` и `fixture.schemaRef`.

**Отчёты.** Прогон пишет `<reportsDir>/results/summary.json` и `<reportsDir>/allure/`. Каталог создаётся
сам; в системе контроля версий ему делать нечего.

Каталог, на который не ссылается ни один артефакт, можно не создавать: он нужен только тогда, когда в
него кто-то смотрит. `speq doctor` сообщает, каких каталогов нет, но это не ошибка сама по себе.

---

## Справочник манифеста

Разбирается структурой `Manifest` в `src/manifest/mod.rs`. Неизвестные ключи игнорируются молча.

### Обязательные поля

| Поле | Тип | Назначение |
| --- | --- | --- |
| `version` | строка | Версия формата манифеста. Единственное поддерживаемое значение — `"1"` |
| `project` | строка | Имя проекта; выводится в `speq doctor` и в отчётах |
| `defaultEnvironment` | строка | Окружение, используемое, когда `--env` не передан |

### Каталоги

Все необязательны, каждый со своим значением по умолчанию.

| Поле | По умолчанию |
| --- | --- |
| `environmentsDir` | `environments` |
| `suitesDir` | `suites` |
| `reportsDir` | `reports` |
| `schemasDir` | `schemas` |
| `modulesDir` | `modules` |
| `fixturesDir` | `fixtures` |

### `retry` — политика повторов

Применяется ко всем шагам `type: api`. Без блока повторов нет.

| Поле | Тип | По умолчанию | Назначение |
| --- | --- | --- | --- |
| `enabled` | boolean | `false` | Повторы происходят только при `true` |
| `maxAttempts` | integer | `3` | Общее число попыток, включая первую |
| `delayMs` | integer | `0` | Пауза перед первым повтором |
| `backoff` | `fixed` \| `exponential` | `fixed` | Как растёт пауза между попытками |
| `retryOn.networkErrors` | boolean | `false` | Повторять, когда запрос вообще не дошёл |
| `retryOn.statusCodes` | массив integer | `[]` | Коды ответа, вызывающие повтор |

Если `retryOn` не заполнен ни одним из двух способов, повторять нечего — даже при `enabled: true`.

```yaml
retry:
  enabled: true
  maxAttempts: 3
  delayMs: 300
  backoff: exponential
  retryOn:
    networkErrors: true
    statusCodes: [429, 502, 503, 504]
```

Подробности поведения — в [features/retry-waiter-policy.md](features/retry-waiter-policy.md).

### `coverage` — покрытие API по OpenAPI

| Поле | Тип | По умолчанию | Назначение |
| --- | --- | --- | --- |
| `enabled` | boolean | `false` | Включает анализ; то же делает флаг `--coverage` |
| `openapi` | строка \| null | `null` | Путь к OpenAPI-документу от корня проекта; переопределяется `--openapi` |
| `report` | boolean | `false` | Печатать списки покрытых и непокрытых эндпоинтов после прогона |
| `fail_below` | число \| null | `null` | Завершить прогон с кодом `1`, если покрытие ниже этого процента |

```yaml
coverage:
  enabled: true
  openapi: "./docs/openapi.yaml"
  report: true
  fail_below: null
```

> **Написание `fail_below`.** Все поля манифеста именуются в `camelCase` — кроме этого. `CoverageConfig`
> в `src/coverage/mod.rs` не имеет атрибута `rename_all`, поэтому поле читается только как
> `fail_below`. Написание `failBelow` молча игнорируется, и порог просто не срабатывает.
> Отслеживается в [speq-tms/speq-docs#1](https://github.com/speq-tms/speq-docs/issues/1).

Если `coverage.enabled: true`, но `openapi` не задан ни в манифесте, ни флагом, прогон продолжится с
предупреждением, а блок `coverage` в `summary.json` просто не появится.

Подробности — в [features/coverage.md](features/coverage.md).

### Полный пример

```yaml
version: "1"
project: "jsonplaceholder-api-tests"
defaultEnvironment: "ci"
environmentsDir: "environments"
suitesDir: "suites"
reportsDir: "reports"
schemasDir: "schemas"
modulesDir: "modules"
fixturesDir: "fixtures"

retry:
  enabled: true
  maxAttempts: 3
  delayMs: 300
  backoff: exponential
  retryOn:
    networkErrors: true
    statusCodes: [429, 502, 503, 504]

coverage:
  enabled: true
  openapi: "./docs/openapi.yaml"
  report: true
  fail_below: null
```

---

## Что создаёт `speq init`

```bash
speq init                      # in-repo: корень в ./.speq
speq init --mode test-repo     # test-repo: корень в текущем каталоге
```

**Режим по умолчанию — `in-repo`**, не `test-repo`.

Команда создаёт каталоги `environments/`, `suites/`, `reports/`, `schemas/`, `modules/` — и три файла:

`manifest.yaml` (ключи выводятся по алфавиту, `fixturesDir`, `retry` и `coverage` не создаются — они
необязательны и берутся из значений по умолчанию):

```yaml
defaultEnvironment: ci
environmentsDir: environments
modulesDir: modules
project: <имя текущего каталога>
reportsDir: reports
schemasDir: schemas
suitesDir: suites
version: '1'
```

`environments/ci.yaml`:

```yaml
name: ci
baseUrl: https://httpbin.org
```

`suites/smoke.yaml`:

```yaml
id: "smoke.health"
title: "Health endpoint smoke test"
steps:
  - type: api
    name: "GET health"
    method: GET
    url: "/status/200"
    assert:
      - type: status
        expected: 200
```

Полученный проект работоспособен сразу: `speq run --env ci` даёт зелёный прогон по httpbin.org.

Чего `init` **не** создаёт: каталог `fixtures/`, файлы `init.yaml`, модули и схемы. Всё это добавляется
вручную по мере надобности.

Существующие файлы не перезаписываются. `manifest.yaml` в корне — единственный случай, когда команда
завершается ошибкой:

```text
manifest already exists: <путь>
```

---

## Проверка раскладки

```bash
speq doctor --format json
```

Печатает найденный корень, режим, абсолютный путь каждого каталога, факт его существования и число
найденных тестов:

```json
{
  "environmentsDir": "<корень>/environments",
  "environmentsDirExists": true,
  "manifestExists": true,
  "mode": "test-repo",
  "modulesDir": "<корень>/modules",
  "modulesDirExists": true,
  "ok": true,
  "reportsDir": "<корень>/reports",
  "schemasDir": "<корень>/schemas",
  "schemasDirExists": true,
  "speqRoot": "<корень>",
  "suitesDir": "<корень>/suites",
  "suitesDirExists": true,
  "testsCount": 1
}
```

`speq validate` идёт дальше: разбирает манифест, все тест-спеки, все `init.yaml` и все модули.
Запросов при этом не выполняется.

Что именно проверяется по файлам, стоит знать точно — проверок меньше, чем кажется:

| Ссылка | Проверяется `validate` |
| --- | --- |
| `bodyFromFixture.ref` → файл в `fixturesDir` | да |
| `fixture.schemaRef` → файл в `schemasDir` | да |
| `assert: type: schema` с `ref` → файл в `schemasDir` | **нет** — отсутствие файла обнаружится только в прогоне |
| шаги внутри действий модуля | **нет** — у модулей проверяется только формат выражений `returns` |

Оба пробела отслеживаются в [speq-tms/speq-docs#23](https://github.com/speq-tms/speq-docs/issues/23).

---

## Раскладка на примере

`speq-examples/test-repo-mode-jsonplaceholder` — режим `test-repo`, все каталоги на местах:

| Путь | Что внутри |
| --- | --- |
| `manifest.yaml` | версия, проект, окружение по умолчанию, `retry`, `coverage` |
| `environments/ci.yaml`, `local.yaml` | `baseUrl` для JSONPlaceholder и для локального стенда |
| `suites/init.yaml` | переменные, импорты и хуки, общие для всех тестов |
| `suites/posts/`, `users/`, `comments/`, `todos/`, `albums/` | тесты по ресурсам, у каждого свой `init.yaml` |
| `modules/jsonplaceholder.yaml` | действия `getPostById`, `getUserById`, … с `returns` |
| `fixtures/post-create.yaml`, … | заготовки тел запросов с генерируемыми значениями |
| `schemas/jsonplaceholder/*.json` | JSON-схемы для ассертов `type: schema` |
| `docs/openapi.yaml` | спецификация, по которой считается покрытие |
| `reports/` | результат прогона; в репозитории не хранится |

---

## Дальше

- [README.md](README.md) — команды, флаги и коды выхода.
- [features/](features/) — спецификации отдельных фич: генерация данных, фикстуры, повторы, модули,
  ATDD, покрытие.
