# speq current implementation baseline (from prototype)

Этот документ фиксирует, что уже готово в текущем проекте и может быть перенесено в `speq`.

## Уже реализовано

## CLI skeleton

- Отдельный бинарь CLI (`tms` в текущем прототипе).
- Команды:
  - `init`
  - `validate`
  - `list`
  - `run`
  - `report`
  - `doctor`

## Базовый контракт структуры

- `.tms_test` layout с `manifest`, `environments`, `suites`, `reports`.
- Детализация output-директорий:
  - `reports/results`
  - `reports/allure`
  - `reports/logs`

## YAML parser + runner (API)

- Парсинг API YAML.
- Переменные на уровне теста (`variables`).
- Шаблоны `{{var}}` + env substitution.
- Reusable steps через `type: use`.
- Assertions:
  - `status`
  - `json`
  - `contains`
  - `notcontains`
  - `exists`
  - `regex`

## JSON output + exit codes

- Машиночитаемый JSON output для ключевых команд.
- Exit codes:
  - `0` success
  - `1` test failures
  - `2` validation/config
  - `3` internal/runtime

## Allure-compatible output

- Генерация `*-result.json` и `executor.json`.
- Включены test steps.
- Вложены request/response attachments.
- Assertion details представлены как nested steps (`expected`/`actual`).

## Документация

- Сформированы канонические docs по стратегии и контрактам.
- Сформирован user guide по CLI-first подходу.

## Что еще не доведено до production

- Полноценный package/distribution `speq` CLI (cross-platform release).
- Отдельный репозиторий для `speq-github-runner`.
- Отдельный репозиторий для `speq-vscode-extension`.
- Строгая schema-driven валидация (JSON Schema + version migration engine).
- Полный Allure lifecycle (history/trends/categories/environment).
- Security hardening (secret masking, safe logs, policy controls).
