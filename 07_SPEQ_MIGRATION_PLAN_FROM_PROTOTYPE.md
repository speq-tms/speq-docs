# speq migration plan from current prototype

## Контекст

Текущий проект был тех-обкаткой (`tms-desktop`) и уже содержит полезный CLI-core baseline.
Новый этап: выделение полноценного продукта `speq`.

## Что переносим в первую очередь

## P0: Contracts and naming

- Переименование:
  - `tms` -> `speq`
  - `.tms_test` -> `.speq` (фиксируем как целевой стандарт).
- Режимы хранения тестов (оба обязательны):
  - `in-repo mode`: тесты лежат в приложенческом репозитории в папке `.speq/`.
  - `test-repo mode`: отдельный репозиторий только для тестов, где содержимое `.speq` является корнем репозитория (без вложенной папки `.speq`).
- Freeze contracts:
  - manifest schema
  - test schema
  - run JSON model
  - exit codes

### Repository layout contract (new)

#### A) In-repo mode

```text
service-repo/
  src/...
  .speq/
    manifest.yaml
    environments/
    suites/
    reports/
```

#### B) Dedicated test-repo mode

```text
service-tests-repo/
  manifest.yaml
  environments/
  suites/
  reports/
```

### CLI discovery behavior

- CLI должен уметь автоматически определять оба режима:
  1. если найдена `.speq/` — работаем с ней;
  2. если `.speq/` нет, но в корне есть `manifest.yaml` + `suites/` — это `test-repo mode`.
- Добавить явный override:
  - `--speq-root <path>` для ручного задания корня.

### Final detection rule (fixed)

Определяем `speq root` и режим только по файловой структуре:

1. **Explicit override first**
   - Если передан `--speq-root <path>`, используем его как источник истины.

2. **In-repo mode**
   - Если в `cwd` (или в найденном repo root) существует директория `.speq/`
   - и внутри `.speq/` есть `manifest.yaml` и `suites/`,
   - тогда режим = `in-repo`, а `speq root = <repo>/.speq`.

3. **Test-repo mode**
   - Если `.speq/` отсутствует,
   - но в корне есть `manifest.yaml` и `suites/`,
   - тогда режим = `test-repo`, а `speq root = <repo root>`.

4. **Ambiguous config (hard error)**
   - Если одновременно валидны:
     - `<repo>/.speq/manifest.yaml + suites/`
     - и `<repo>/manifest.yaml + suites/`,
   - CLI завершает выполнение с ошибкой конфигурации и просит указать `--speq-root`.

5. **Not initialized**
   - Если ни один режим не найден, возвращаем error:
     - `speq root not found; run 'speq init' or pass --speq-root`.

### Why this rule

- Детерминированность и одинаковое поведение в local/CI/extension.
- Отсутствие магии по имени репозитория или remote URL.
- Простая отладка и предсказуемое переключение между режимами.

## P1: CLI extraction

- Создать отдельный репозиторий `speq-cli`.
- Перенести модули:
  - parser
  - runner
  - assertions
  - report writer (Allure)
- Добавить release packaging (mac/linux/windows).
- Добавить migration command:
  - `speq migrate-layout` (поддержка `.tms_test -> .speq`, и подготовка dedicated test-repo).

## P2: CI runner extraction

- Создать `speq-github-runner`.
- Базовый setup action + run templates.
- Документация по CI contract.

## P3: VS Code extension MVP

- Создать `speq-vscode-extension`.
- Реализовать explorer + diagnostics + run actions.

## Риски и контроль

- Риск: расхождение форматов между модулями.
  - Mitigation: единый contracts doc + versioning.
- Риск: “вторая логика исполнения” в extension/runner.
  - Mitigation: все execution path только через CLI.
- Риск: migration friction у ранних пользователей.
  - Mitigation: alias/compat layer и миграционные команды.
- Риск: путаница двух режимов хранения (in-repo vs dedicated test-repo).
  - Mitigation: четкие правила autodiscovery + `--speq-root` + единый docs/example pack.
