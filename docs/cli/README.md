# SPEQ CLI

`speq-cli` — единственный runtime экосистемы. Любое поведение времени выполнения принадлежит ему;
`speq-github-runner` и `speq-vscode-extension` только вызывают его и отображают результат.

Этот документ описывает **фактическую** поверхность CLI. Источник истины — код в
[speq-cli](https://github.com/speq-tms/speq-cli) (`src/main.rs`, `src/cli/`). Расхождение документа с кодом
считается багом документа.

Актуальная версия: `1.0.0`.

---

## С чего начать

| Документ | О чём |
| --- | --- |
| [quickstart.md](quickstart.md) | Первый зелёный прогон с чистого каталога |
| [project-layout.md](project-layout.md) | Раскладка проекта `.speq` и справочник манифеста |
| [dsl.md](dsl.md) | Полный справочник YAML-DSL: тесты, шаги, ассерты, переменные, модули, фикстуры |

---

## Команды

| Команда | Назначение |
| --- | --- |
| `speq init [--mode in-repo\|test-repo]` | Создать скелет `.speq` в выбранном режиме репозитория |
| `speq list [--speq-root <path>] [--format json]` | Перечислить обнаруженные тесты и сьюты |
| `speq validate [--speq-root <path>] [--format json]` | Проверить манифест и YAML-спеки без выполнения запросов |
| `speq run [...]` | Выполнить тест-план и сформировать отчёты |
| `speq report [--speq-root <path>] [--format all\|summary\|allure] [--summary <summary.json>]` | Пересобрать отчёты из существующего summary |
| `speq doctor [--speq-root <path>] [--format json]` | Диагностика окружения и структуры проекта |
| `speq version` (`-V`, `--version`) | Версия бинарника |
| `speq help` (`-h`, `--help`) | Справка |

### `speq run`

```
speq run [--speq-root <path>] [--env <name>] [--test <file>|--suite <dir>] [--tags a,b]
         [--report all|summary|allure] [--output <summary.json>] [--coverage] [--openapi <spec.yaml>]
```

| Флаг | Поведение |
| --- | --- |
| `--speq-root <path>` | Корень `.speq`; по умолчанию определяется по текущему каталогу — см. [project-layout.md](project-layout.md#как-cli-находит-корень) |
| `--env <name>` | Имя окружения; по умолчанию `defaultEnvironment` из манифеста |
| `--test <file>` | Запустить один файл теста |
| `--suite <dir>` | Запустить одну сьюту |
| `--tags a,b` | Фильтр по тегам |
| `--report all\|summary\|allure` | Режим отчёта; **по умолчанию `allure`** |
| `--output <summary.json>` | Путь для summary; допустим только с `--report summary\|all` |
| `--coverage` | Включить contract coverage, даже если он выключен в манифесте |
| `--openapi <spec.yaml>` | Переопределить `coverage.openapi` из манифеста |

Отчёты пишутся в `<reportsDir>/results/summary.json` и `<reportsDir>/allure/`.

---

## Коды выхода

| Код | Значение |
| --- | --- |
| `0` | Успех: нет упавших тестов, нет ошибок, coverage-порог не нарушен |
| `1` | Есть упавшие тесты/ошибки выполнения, либо coverage ниже `coverage.fail_below` |
| `2` | Ошибка валидации, конфигурации или аргументов |
| `3` | Внутренняя ошибка (сообщение начинается с `internal:`) |

Тесты со статусом `pending` (см. [ATDD flow](features/atdd-flow.md)) не приводят к коду `1`.

---

## Манифест

Полный справочник манифеста и раскладки проекта — [project-layout.md](project-layout.md).
Краткая сводка полей, поддерживаемых runtime:

| Поле | Назначение |
| --- | --- |
| `version`, `project` | Идентификация проекта |
| `defaultEnvironment` | Окружение по умолчанию для `run` |
| `environmentsDir`, `suitesDir`, `reportsDir`, `schemasDir`, `modulesDir`, `fixturesDir` | Раскладка каталогов |
| `retry` | Политика повторов: `enabled`, `maxAttempts`, `delayMs`, `backoff`, `retryOn` |
| `coverage` | Contract coverage: `enabled`, `openapi`, `report`, `fail_below` |

> Известная несогласованность: все поля манифеста именуются в `camelCase`, кроме `coverage.fail_below`,
> который парсится только как `snake_case`. Документ описывает фактическое поведение парсера.

---

## Статус фич

| Фича | Спецификация | Статус в коде |
| --- | --- | --- |
| Data generation | [features/data-generation.md](features/data-generation.md) | реализовано (`src/generator/`) |
| Fixtures and builders | [features/fixtures-and-builders.md](features/fixtures-and-builders.md) | реализовано (`src/fixtures/`) |
| Retry / waiter policy | [features/retry-waiter-policy.md](features/retry-waiter-policy.md) | реализовано (`manifest.retry`) |
| Module outputs | [features/module-outputs.md](features/module-outputs.md) | реализовано (`use action` + `returns`) |
| ATDD flow | [features/atdd-flow.md](features/atdd-flow.md) | реализовано (`status: pending`) |
| API coverage | [features/coverage.md](features/coverage.md) | реализовано (`src/coverage/`, `--coverage`) |
| Parallel execution | [features/parallel-execution.md](features/parallel-execution.md) | **не реализовано** — спецификация опережает код |

Umbrella-документ по Phase 2: [phase2-core-expansion.md](phase2-core-expansion.md).

---

## Локальная проверка

```bash
cd speq-cli && cargo build --release && cargo test
cd speq-examples/test-repo-mode-jsonplaceholder && speq run --env ci
```

Перед каждым прогоном очищайте `reports/allure/` и `reports/results/`, чтобы не читать устаревший вывод.
Зелёный прогон даёт `"failed": 0` в summary. Регрессии в этом наборе не допускаются.

Каждая новая фича должна получить хотя бы один acceptance-пример в
`speq-examples/test-repo-mode-jsonplaceholder`. Багфиксы и доработки существующих фич нового примера не требуют.
