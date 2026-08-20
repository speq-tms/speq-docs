# speq vscode extension (open source first)

## Назначение

`speq-vscode-extension` дает удобный вход в YAML-тесты внутри VS Code.

## Open Source MVP

- Test tree explorer (`.tms_test/suites`).
- YAML preview с подсветкой ключевых блоков.
- Manifest/env quick view.
- Diagnostics:
  - ошибки валидации через `speq validate --format json`.
- Run from editor:
  - запуск `speq run` для текущего теста/сьюта.

## UX flow

1. User открывает репозиторий.
2. Extension обнаруживает `.tms_test`.
3. Показывает дерево тестов.
4. При сохранении YAML запускает validation diagnostics.
5. По команде Run показывает статус и ссылку на artifacts.

## Future (`speq-pro extension`)

- Visual editor (no-code blocks/canvas).
- Cross-test reuse refactoring tools.
- Project topology view.
- Governance and team-level policy controls.

## Integration principles

- Extension не содержит собственного runner.
- Любое исполнение/валидация идет через CLI.
- Единый контракт данных с `speq-cli`.
