# speq open-source and pro boundary

## Open Source scope

### 1) `speq-cli`

- YAML parser and validator.
- Manifest contract support.
- Environment loading.
- API test runner.
- Assertions and reusable helpers (`type: use`).
- JSON output + exit codes.
- Allure-compatible artifacts.

### 2) `speq-github-runner`

- GitHub Action wrapper around CLI.
- Standard workflow templates:
  - PR smoke
  - nightly/full regression
- Artifacts upload (`results`, `allure`, logs).

### 3) `speq-vscode-extension`

- YAML test visualization.
- Manifest/env quick preview.
- Run diagnostics (lint-like feedback from CLI).
- Test tree explorer.

## Paid scope (`speq-pro`)

### `speq-pro ci/cd`

- Parallel/multithread execution.
- Advanced scheduling and sharding.
- CI performance analytics.
- SLA support and enterprise policies.

### `speq-pro extension`

- Visual editing (no-code/low-code editor).
- Graph/project-level visualization.
- Cross-test refactoring helpers.
- Team governance features in editor.

## Boundary rules

- Core execution logic всегда в open-source CLI.
- Pro не дублирует runtime, а расширяет orchestration и UX.
- Контракт формата (manifest + test YAML) остается единым между OSS и Pro.
