# speq cli mvp status and next roadmap

## Статус update (April 2026)

- `speq-github-runner` реализован как composite action:
  - modes: `setup|run|custom`;
  - reference workflows: PR smoke, nightly regression;
  - artifacts: summary/allure/logs;
  - release channel и checklist: `@v1`.
- `speq-vscode-extension` реализован как OSS MVP:
  - suites explorer;
  - diagnostics через `speq validate --format json`;
  - run action через `speq run`;
  - quick preview manifest/environment;
  - без собственного runtime.
- Добавлена сквозная parity-проверка CLI + Runner + Extension на canonical fixtures (`speq-cli/.github/workflows/mvp-integration.yml`).

## Что уже сделано по `speq-cli` (MVP)

## Core and layout

- CLI ядро переведено на Rust.
- Поддержаны оба режима структуры:
  - `in-repo` (`.speq/manifest.yaml + suites/`)
  - `test-repo` (`manifest.yaml + suites/` в root)
- Реализован autodiscovery + `--speq-root` override.

## Команды MVP

- `speq init --mode in-repo|test-repo`
- `speq validate [--speq-root <path>] [--format json]`
- `speq list [--speq-root <path>] [--format json]`
- `speq run [--speq-root <path>] [--env <name>] [--test <file>|--suite <dir>] [--tags a,b] [--report all|summary|allure] [--output <summary.json>]`
- `speq report [--speq-root <path>] [--format all|summary|allure] [--summary <summary.json>]`
- `speq doctor [--speq-root <path>] [--format json]`

## Run/report capabilities

- Запуск:
  - одного теста (`--test`)
  - группы тестов из папки (`--suite`)
  - фильтрация по тегам (`--tags`)
- Отчеты:
  - `summary`
  - `allure`
  - `all`
- Дефолт `run` без `--report` -> `allure`.
- `run --output` для кастомного пути summary (для `summary|all`).
- `report` умеет генерировать Allure из уже сохраненного summary.

## Контракты и тесты

- В `speq-contracts` добавлены v1 схемы:
  - `manifest`
  - `test`
  - `results`
- В `speq-examples` добавлены canonical fixtures для `in-repo` и `test-repo`.
- В `speq-cli/tests` добавлены regression suites:
  - discovery
  - run filters/options
  - report generation
  - contract check: summary vs `results/v1.json`

## CI and quality gates

- CI `speq-cli` выполняет:
  - `cargo build`
  - `cargo test`
  - smoke `validate` на canonical fixture
  - smoke `doctor` на canonical fixture

## Куда двигаться дальше по roadmap

## Priority 1: `speq-github-runner` (закрыто)

1. Реализовать `action.yml` и режимы:
   - `setup`
   - `run`
   - `custom`
2. Добавить reference workflows:
   - PR smoke
   - nightly regression
3. Гарантировать upload artifacts:
   - summary/results
   - allure
   - logs

### Exit criteria (status: met)

- Runner стабильно запускает `speq-cli` на GitHub-hosted runner.
- Для 2-3 demo проектов есть рабочие шаблоны workflow.

## Priority 2: `speq-vscode-extension` OSS MVP (закрыто)

1. Explorer для `suites/`.
2. Diagnostics через `speq validate --format json`.
3. Run action через `speq run`.
4. Quick preview для manifest/environment.

### Exit criteria (status: met)

- Extension не содержит собственного runtime.
- Диагностика и запуск повторяют поведение CLI.

## Priority 3: CLI hardening and alpha release (активный следующий этап)

1. CI matrix (ubuntu/macos/windows).
2. Release packaging (binary artifacts).
3. Alpha tag and release notes (`v0.1.0-alpha`).
4. Security базовый hardening:
   - secret masking в логах
   - safe logging defaults
