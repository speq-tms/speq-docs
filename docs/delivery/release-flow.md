# Release flow

Единый процесс доставки изменений для всех репозиториев экосистемы SPEQ, включая `speq-docs`.

## Правила ветвления

1. Не доставлять фичи и фиксы напрямую из `main`.
2. Создать release-candidate ветку от `main` — например `v1.0.0` или `v1.1.0-alpha.1`.
3. Создавать delivery-ветки **только от RC**:
   - `feat/<scope>-<name>`
   - `fix/<scope>-<name>`
   - `chore/<scope>-<name>`
   - `docs/<scope>-<name>`
4. Открывать PR из delivery-ветки в RC.
5. По готовности открыть один финальный PR из RC в `main`.
6. Использовать Conventional Commits в сообщениях коммитов и заголовках PR.

Если блокер найден до публикации тега — исправление остаётся в той же RC-ветке. Если тег уже опубликован —
исправление идёт через новый patch или новую RC.

## Conventional Commits

| Префикс | Когда |
| --- | --- |
| `feat:` | Новое поведение runtime или новая возможность DSL |
| `fix:` | Исправление некорректного поведения |
| `chore:` | Инфраструктура, зависимости, служебные изменения |
| `docs:` | Изменения только в документации |
| `refactor:` | Изменение структуры кода без изменения поведения |
| `test:` | Изменения только в тестах |

## Acceptance gates

Перед мержем RC в `main`:

- `cargo build && cargo test` в `speq-cli` — зелёные;
- прогон `speq-examples/test-repo-mode-jsonplaceholder` даёт `"failed": 0`;
- каждая новая фича покрыта хотя бы одним acceptance-примером в `speq-examples`;
- документация в `speq-docs` описывает фактическое поведение изменённого кода;
- при изменении DSL синхронизированы `speq-cli`, `speq-contracts` и `speq-vscode-extension`.

## Смежные документы

- [release-and-marketplace-playbook.md](release-and-marketplace-playbook.md) — публикация артефактов и дистрибуция.
- [gh-pages.md](gh-pages.md) — публичный сайт документации.
- [../product/mvp-v1.0.0-scope.md](../product/mvp-v1.0.0-scope.md) — scope релиза `v1.0.0`.
