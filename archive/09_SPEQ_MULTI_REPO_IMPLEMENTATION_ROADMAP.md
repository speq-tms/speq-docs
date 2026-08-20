# speq multi-repo implementation roadmap

## Цель документа

Зафиксировать пошаговый план реализации экосистемы `speq` как набора отдельных git-репозиториев внутри папки `./speq`, где каждый проект:

- имеет автономную кодовую базу;
- инициализирован как отдельный git repository;
- привязан к собственному remote (`origin`);
- публикуется и версионируется независимо.

Документ дополняет `08_SPEQ_EXECUTION_ROADMAP.md` и превращает его в исполнимый план по репозиториям, deliverables и критериям готовности.

## Базовые вводные (синхронизированы с текущими материалами)

- Product thesis: `one spec + one runner + many interfaces`.
- OSS-first: runtime и execution logic только в `speq-cli`.
- Целевой naming/layout:
  - `tms` -> `speq`;
  - `.tms_test` -> `.speq`;
  - поддержка двух режимов: `in-repo` и `test-repo`.
- CLI autodiscovery:
  - `.speq/manifest.yaml + suites/` -> `in-repo`;
  - `manifest.yaml + suites/` в repo root -> `test-repo`;
  - конфликт режимов -> hard error + требование `--speq-root`.

## Что уже можно переиспользовать из прототипа

На основе `speq-docs/03_*` и поведения UI из `/Users/stepankaziatko/tms-desktop/src` в MVP нужно сохранить:

- core-команды: `init`, `validate`, `list`, `run`, `report`, `doctor`;
- контракты результата:
  - JSON summary + устойчивые exit codes (`0/1/2/3`);
  - Allure-compatible artifacts;
- runtime-возможности:
  - YAML parse/validate;
  - `variables`, `{{var}}`, env substitution;
  - reusable steps (`type: use`);
  - assertions (`status`, `json`, `contains`, `notcontains`, `exists`, `regex`);
- операционные сценарии (из UI-флоу):
  - выбор/клонирование репозитория;
  - validate/save/run;
  - git status/diff/commit/pull/push;
  - детализированный run result (step-by-step status, duration, message).

## Целевая структура репозиториев в `./speq`

```text
./speq/
  speq-cli/
  speq-github-runner/
  speq-vscode-extension/
  speq-contracts/               # рекомендуется как отдельный OSS repo
  speq-examples/                # рекомендуется как отдельный OSS repo
  speq-docs/                    # текущий docs repo (может остаться как отдельный)
```

### Минимальный набор обязательных репозиториев

1. `speq-cli` (OSS, core runtime).
2. `speq-github-runner` (OSS, CI wrapper над CLI).
3. `speq-vscode-extension` (OSS, интерфейс поверх CLI).

### Рекомендуемые supporting-репозитории

4. `speq-contracts`:
   - JSON Schema (`manifest`, test YAML, JSON run model),
   - версия контрактов и changelog.
5. `speq-examples`:
   - шаблоны `.speq`,
   - демо для `in-repo` и `test-repo`,
   - reference CI workflows.

## Git model для каждого проекта

Для каждого каталога из roadmap:

1. `git init`.
2. Базовый branch: `main`.
3. Добавление remote:
   - `git remote add origin <repo-url>`.
4. Первый push:
   - `git push -u origin main`.
5. Версионирование:
   - SemVer;
   - release tags вида `vX.Y.Z`.

Рекомендуемые branch conventions:

- `main` — stable;
- `develop` (опционально) — интеграционная ветка;
- feature branches: `feat/<scope>-<short-name>`;
- fix branches: `fix/<scope>-<short-name>`.

## Поэтапный план реализации

## Phase 0 (Week 1): bootstrap и contract freeze

### Deliverables

- Созданы репозитории:
  - `speq-cli`,
  - `speq-github-runner`,
  - `speq-vscode-extension`,
  - `speq-contracts`,
  - `speq-examples`.
- Во всех репозиториях:
  - README + contribution guide + license;
  - issue/PR templates;
  - базовый CI (lint/test/build).
- В `speq-contracts`:
  - зафиксированы v1 схемы и exit-code contract.

### Acceptance criteria

- Каждый проект имеет собственный `origin`.
- Каждый проект проходит базовый CI на pull request.
- Контрактные схемы опубликованы и импортируются как source of truth.

## Phase 1 (Weeks 2-4): `speq-cli` extraction + alpha release

### Deliverables

- Перенос core-модулей из прототипа:
  - parser, runner, assertions, reporting.
- Реализация discovery логики:
  - `in-repo` + `test-repo` + `--speq-root`.
- Команды:
  - `init`, `validate`, `list`, `run`, `report`, `doctor`,
  - `migrate-layout` (`.tms_test` -> `.speq`).
- Packaging:
  - Linux/macOS/Windows artifacts.
- CI matrix + smoke e2e на фикстурах из `speq-examples`.

### Acceptance criteria

- Детеминированные результаты local vs CI.
- Exit code contract стабилен и покрыт тестами.
- Alpha release опубликован и документирован.

## Phase 2 (Weeks 4-6): `speq-github-runner` OSS

### Deliverables

- `action.yml` и режимы:
  - `setup`,
  - `run`,
  - `custom`.
- Reference workflows:
  - PR smoke,
  - nightly regression.
- Upload artifacts:
  - `results`,
  - `allure`,
  - logs.
- Совместимость минимум с 2-3 демонстрационными проектами.

### Acceptance criteria

- Runner использует только `speq-cli` (без дублирования runtime).
- Workflow стабильно работает на чистом GitHub-hosted runner.
- Документация закрывает onboarding до первого зеленого CI-прохождения.

## Phase 3 (Weeks 6-9): `speq-vscode-extension` OSS MVP

### Deliverables

- Test explorer (`.speq/suites` или `suites/` для `test-repo mode`).
- Diagnostics через `speq validate --format json`.
- Run action для теста/сьюты через CLI.
- Manifest/env quick preview.
- UX-совместимость с operational flow из прототипа:
  - открыть проект,
  - валидировать,
  - запустить,
  - перейти к артефактам.

### Acceptance criteria

- Extension не содержит собственного execution engine.
- Ошибки валидации отображаются как actionable diagnostics.
- MVP готов к OSS-публикации.

## Phase 4 (Weeks 9-10): hardening + v1.0 readiness

### Deliverables

- Security hardening:
  - secret masking,
  - safe logs policy.
- Полный Allure lifecycle (история/trends/categories/environment) или документированный план v1.x.
- Совместимость версий:
  - матрица `cli <-> runner <-> extension`.
- Release playbooks:
  - кто и как режет релизы,
  - обратная совместимость/minimum supported versions.

### Acceptance criteria

- Публичный v1.0 релиз CLI + совместимые runner/extension версии.
- Time-to-first-test < 15 минут по onboarding check-list.
- Базовые KPI adoption/reliability/DX измеряются автоматически.

## Dependency graph и порядок работ

1. `speq-contracts` (source of truth).
2. `speq-cli` (runtime implementation).
3. `speq-examples` (fixtures + onboarding).
4. `speq-github-runner` (CI wrapper).
5. `speq-vscode-extension` (editor UX).

Критическое правило: `runner` и `extension` зависят от публичного CLI-контракта и версии `speq-cli`, но не копируют его runtime.

## Release и совместимость между репозиториями

- Ввести compatibility table в `speq-docs`:
  - `speq-cli vA.B` поддерживает `runner vX.Y`, `extension vM.N`.
- Для breaking changes в schema:
  - major bump контракта,
  - миграционные подсказки и `speq migrate-layout`.
- Для minor updates:
  - backward compatibility by default.

## Operational checklist на запуск каждого нового репозитория

1. Создать remote repository.
2. Создать локальную папку в `./speq/<repo-name>`.
3. `git init`, настроить `origin`, выполнить initial commit.
4. Подключить CI workflow.
5. Создать `v0.1.0-alpha` release tag.
6. Добавить ссылку на репозиторий в `speq-docs/00_INDEX.md`.

## Быстрый старт bootstrap

В корне `./speq` добавлен скрипт `bootstrap_speq_repos.sh` и шаблон `speq-remotes.env.example`.

Минимальный запуск:

```bash
cd ./speq
cp speq-remotes.env.example speq-remotes.env
# заполнить реальные URL remote-репозиториев
bash ./bootstrap_speq_repos.sh
```

## KPI и контрольные точки

- Adoption:
  - количество внешних репозиториев, где `speq` запускается в CI.
- Reliability:
  - процент совпадения local/CI результатов.
- DX:
  - median time-to-first-test.
- Quality:
  - доля падений с actionable diagnostics (JSON + Allure + extension diagnostics).

## Основные риски и mitigation

- Расхождение контрактов между репозиториями:
  - единый `speq-contracts`, contract tests в CI.
- Дублирование runtime в runner/extension:
  - архитектурное правило + codeowners review gate.
- Сложность миграции legacy `.tms_test`:
  - `speq migrate-layout`, примеры до/после, rollback guide.
- Путаница `in-repo` vs `test-repo`:
  - строгая autodiscovery логика и явный `--speq-root`.
