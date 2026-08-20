# speq next steps execution plan

## Status update (April 2026)

Закрыто в OSS MVP delivery:

- `speq-github-runner`:
  - реализованы action modes `setup|run|custom`;
  - добавлены reference workflows (PR smoke, nightly);
  - гарантирован upload artifacts (summary/allure/logs);
  - оформлен release channel через `@v1`.
- `speq-vscode-extension` MVP:
  - suites explorer;
  - diagnostics через `speq validate --format json`;
  - run actions через `speq run`;
  - quick preview manifest/environments;
  - extension не содержит execution runtime.
- Сквозная проверка parity добавлена в `speq-cli/.github/workflows/mvp-integration.yml`
  (CLI + Runner + Extension на canonical fixtures из `speq-examples`).

## Текущий статус

Закрыто в `speq-cli`:

- `init` (in-repo/test-repo);
- `validate`;
- `list`;
- `run` (test/suite/tags/report mode/output);
- `report` (allure from summary);
- regression tests в `speq-cli/tests`.

## Следующий фокус (Phase 1 completion)

## Sprint A: Stabilize CLI contracts

1. Финализировать контракт результата прогона (`results/v1`) и привести вывод `run`/`report` к единому model.
2. Добавить contract-check тест:
   - summary JSON из `run --report summary` валидируется против `speq-contracts/schemas/results/v1.json`.
3. Зафиксировать CLI errors catalog:
   - стандартизированные ошибки и когда возвращается код `2` vs `3`.

### Definition of done

- Один стабильный JSON output contract.
- Автотесты ловят любые breaking changes в структуре summary.

## Sprint B: Runtime parity with prototype

1. Расширить `run` до паритета с текущим Rust-прототипом:
   - step timeouts;
   - improved template/env substitution;
   - richer assertion diagnostics;
   - `type: use` edge-cases and nested validation errors.
2. Добавить локальные e2e тесты с mock server (без внешней сети).
3. Добавить `doctor` command (проверка структуры, env, доступности конфигов).

### Definition of done

- Поведение `run` детерминированно на mock e2e.
- Функции prototype-core покрыты migration tests.

## Sprint C: Release readiness

1. CI matrix для `speq-cli`:
   - ubuntu, macos, windows.
2. Release packaging:
   - бинарники для 3 платформ;
   - release notes и install инструкции.
3. Tag first alpha:
   - `v0.1.0-alpha`.

### Definition of done

- Артефакты релиза доступны.
- Проверен smoke install + run на минимум 2 внешних репозиториях.

## После Phase 1

Переход к следующему фокусу после OSS MVP:

1. CLI hardening and alpha release:
   - CI matrix (ubuntu/macos/windows);
   - release packaging (binary artifacts);
   - alpha tag + release notes (`v0.1.0-alpha`);
   - security hardening (secret masking, safe logging defaults).
2. Compatibility/version governance:
   - matrix `cli <-> runner <-> extension`;
   - policy для upgrade paths.

## Deferred scope

Миграция legacy-форматов (`.tms_test -> .speq`) отложена:

- нет активных legacy пользователей;
- текущий приоритет — delivery OSS ядра, CI runner и extension MVP.
