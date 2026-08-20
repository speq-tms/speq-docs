# speq cli phase 2 core expansion overview

## Контекст

После закрытия OSS MVP (`init/validate/list/run/report/doctor`) следующий этап для `speq-cli` - усиление core-runtime по пяти направлениям:

1. генерация данных;
2. fixtures/builders;
3. retry policy + waiter;
4. возврат данных из `use`/modules;
5. параллельный запуск тестов.

Этот документ фиксирует общий scope, порядок внедрения и delivery-модель через release candidate.

## Scope Phase 2

Входит в Phase 2:

- базовые генераторы (`uuid`, `date-time`, `string`, `int`, `bool`, `email`, `name`);
- `fixtures` директория и YAML-описание билдеров;
- глобальная retry policy на проект + step-level waiter;
- `use action` с возвращаемыми значениями;
- `run --parallel` (default `2`, максимум `10`).

Не входит в этот этап:

- условная генерация (`if/else`, rules engine);
- property-based/fuzzy testing;
- распределенный запуск по нескольким процессам/машинам.

## Предлагаемая версия-кандидат

По release-flow конвенции работа выполняется не из `main`.

Рекомендуемая цель релиз-кандидата:

- `v0.2.0-alpha.1` (или следующая свободная alpha-версия).

Ветвление:

1. создать RC-ветку от `main`: `v0.2.0-alpha.1`;
2. для каждой фичи создавать delivery-ветки от RC:
   - `feat/cli-data-generation`
   - `feat/cli-fixtures-builders`
   - `feat/cli-retry-waiter`
   - `feat/cli-module-returns`
   - `feat/cli-parallel-run`
3. PR только в RC-ветку;
4. после CI-green всех PR открыть один финальный PR `v0.2.0-alpha.1 -> main`.

## Фазы реализации

## Phase A - DSL and contracts

- расширить schema/contracts для новых блоков:
  - генераторы;
  - fixtures;
  - retry/waiter;
  - module returns;
  - run options (`parallel`).
- синхронизировать `speq-cli`, `speq-contracts`, `speq-vscode-extension` diagnostics.

Definition of done:

- JSON/YAML contracts зафиксированы;
- `validate` отлавливает некорректные конфиги;
- есть migration notes по backward compatibility.

## Phase B - Runtime implementation

- реализовать runtime для генераторов и fixture materialization;
- реализовать retry + waiter в execution engine;
- расширить `use` для output bindings;
- добавить worker-pool для parallel execution.

Definition of done:

- `run` стабильно работает на canonical fixtures;
- без регрессий для старого DSL;
- детерминированные отчеты при parallel run.

## Phase C - Hardening and release readiness

- интеграционные и regression тесты по каждой фиче;
- e2e наборы в `speq-examples`;
- обновление CI gates и release notes.

Definition of done:

- CI green для RC ветки;
- готова финальная PR в `main`;
- документированы новые возможности и ограничения.

## Порядок внедрения фич (зависимости)

1. Data generation.
2. Fixtures/builders (используют генерацию).
3. Retry + waiter (runtime policy).
4. Module returns (context propagation).
5. Parallel run (в конце, после стабилизации runtime).

## Список детальных документов

- `features/data-generation.md`
- `features/fixtures-and-builders.md`
- `features/retry-waiter-policy.md`
- `features/module-outputs.md`
- `features/parallel-execution.md`
