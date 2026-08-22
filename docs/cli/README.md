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
| `--speq-root <path>` | Корень `.speq`; без него ищется от текущего каталога вверх по дереву — см. [project-layout.md](project-layout.md#как-cli-находит-корень) |
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

## Статусы теста

Тест заканчивается одним из четырёх статусов. Границу между `failed` и `error` проводит один вопрос:
**ответил ли сервис?**

| Статус | Когда | Что это говорит разработчику |
| --- | --- | --- |
| `passed` | Ответ пришёл, все assertions выполнились | — |
| `failed` | Ответ пришёл и не совпал с ожиданием | Чинить код сервиса |
| `error` | Ответа нет и судить нечего | Поднять сервис или починить спеку |
| `pending` | `status: pending` в DSL — см. [ATDD flow](features/atdd-flow.md) | Ещё не реализовано |

`error` назначается в двух случаях, и оба означают «до ответа дело не дошло»:

- **запрос не завершился** — соединение отклонено, DNS не разрезолвился, TLS не установился, истёк
  `timeoutMs`, не ответил прокси;
- **из спеки не удалось собрать запрос** — не читается файл схемы, не разрешается фикстура, `{{плейсхолдер}}`
  не привязан ни к одной переменной, неизвестный метод, неизвестный тип assertion.

Ответ, который пришёл, но не устроил — всегда `failed`, включая `retry_exhausted` по статус-коду: сервис
ответил `503` три раза, это его ответ. Схема, которая **загрузилась** и не совпала с телом, — тоже `failed`;
`error` даёт только схема, которую не удалось прочитать или скомпилировать.

**Приоритет при сборке статуса теста: `failed` перекрывает `error`.** Если в тесте один шаг получил
неверный ответ, а другой не дозвонился, тест — `failed`. Иначе находка о сервисе была бы замаскирована
инфраструктурной проблемой соседнего шага. Хук (`beforeAll`, `beforeEach`) со статусом `error` прекращает
тест так же, как `failed`, и тест наследует статус хука.

В отчётах:

- `summary.json` — счётчик `totals.error` и `status` каждого теста;
- поле `status` самого прогона остаётся `passed`/`failed`: контракт результатов допускает только эти два
  значения, а различие живёт в счётчиках;
- Allure — `broken`, а не `failed`. Это ровно та же граница: `broken` в Allure означает «тест не удалось
  выполнить», а не «ожидание не сошлось».

---

## Коды выхода

| Код | Значение |
| --- | --- |
| `0` | Успех: нет упавших тестов, нет ошибок, coverage-порог не нарушен |
| `1` | Есть упавшие тесты/ошибки выполнения, либо coverage ниже `coverage.failBelow` |
| `2` | Ошибка валидации, конфигурации или аргументов |
| `3` | Внутренняя ошибка (сообщение начинается с `internal:`) |

Тесты со статусом `pending` (см. [ATDD flow](features/atdd-flow.md)) не приводят к коду `1`.

`error` и `failed` дают один и тот же код `1`: оба означают, что прогон не прошёл, и CI должен остановиться
в обоих случаях. Отличить их можно по отчёту — `totals.error` в `summary.json` и `broken` в Allure, — а не
по коду выхода.

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
| `coverage` | Contract coverage: `enabled`, `openapi`, `report`, `failBelow` |

> `coverage.fail_below` — устаревшее написание, оставленное рабочим: именно оно вышло в v1.0.0. Писать
> оба варианта в одном манифесте нельзя, это ошибка разбора:
>
> ```text
> invalid manifest <файл>: coverage: duplicate field `failBelow` at line N column M
> ```

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
