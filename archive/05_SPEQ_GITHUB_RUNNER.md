# speq github runner (open source)

## Назначение

`speq-github-runner` — open-source action, который запускает `speq-cli` в CI/CD GitHub.

## Минимальный execution contract

1. `speq validate --env ci`
2. `speq run --env ci --format json --output .tms_test/reports/results/summary.json`
3. `speq report --format allure`
4. upload `.tms_test/reports`

## Action modes

- `mode: setup` — только устанавливает CLI.
- `mode: run` — setup + validate/run/report.
- `mode: custom` — user управляет командами сам.

## Пример workflow

```yaml
name: speq tests

on:
  pull_request:
  push:
    branches: [main]

jobs:
  speq:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: speq/setup-cli@v1
      - run: speq validate --env ci
      - run: speq run --env ci --format json --output .tms_test/reports/results/summary.json
      - run: speq report --format allure
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: speq-report
          path: .tms_test/reports
```

## Future (`speq-pro ci/cd`)

- Multi-thread execution.
- Sharding across CI workers.
- Advanced retry/flake control.
- Performance and trend analytics.
