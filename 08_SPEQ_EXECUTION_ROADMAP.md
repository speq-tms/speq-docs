# speq execution roadmap

## Phase 1 (2-4 weeks): foundation

- Finalize `speq` naming and contracts.
- Extract and stabilize CLI in separate repo.
- Add test fixtures and smoke e2e in CI.
- Publish first OSS alpha release.

## Phase 2 (2-3 weeks): github runner

- Create `speq-github-runner`.
- Ship official workflow templates.
- Validate compatibility on 2-3 sample projects.

## Phase 3 (3-5 weeks): vscode extension OSS MVP

- Explorer + diagnostics + run.
- Install/update story and docs.
- Stabilize telemetry-free OSS behavior (or explicit opt-in telemetry).

## Phase 4: pro planning and go-to-market

- Define `speq-pro ci/cd` feature set and pricing hypothesis.
- Define `speq-pro extension` visual editor scope.
- Build private alpha with design partners.

## KPIs for first milestone

- CLI adoption: first external repositories running in CI.
- Reliability: deterministic local vs CI results.
- DX: time-to-first-test under 15 minutes.
- Report quality: actionable failure diagnostics in Allure and JSON.
