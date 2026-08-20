# speq ecosystem MVP v1.0.0 scope

## Цель

Зафиксировать минимальный, релизоспособный scope для `v1.0.0` по всей экосистеме `speq`:

- `speq-cli` как стабильный CLI runtime;
- `speq-github-runner` как простой способ запускать CLI в GitHub Actions;
- `homebrew-tap` как основной install path для macOS/Linux;
- `speq-docs` как публичная документация и quickstart;
- `speq-examples` как acceptance coverage для реальных сценариев;
- `speq-vscode-extension` как интерфейсный слой без собственного runtime.

MVP `v1.0.0` не расширяет продуктовую модель сверх необходимого для первого стабильного релиза. Все спорные улучшения уходят в следующий milestone, если они не блокируют установку, запуск, документацию или acceptance gates.

---

## Release flow

Для каждого затронутого репозитория используется общий release flow:

1. Не работать напрямую от `main`.
2. Использовать уже созданную release-candidate branch `v1.0.0`, созданную от `main` в каждом затронутом репозитории.
3. Создавать delivery branches только от `v1.0.0`:
  - `feat/<scope>-<name>`;
  - `fix/<scope>-<name>`;
  - `chore/<scope>-<name>`.
4. Открывать PR из delivery branch в `v1.0.0`.
5. После готовности открыть один финальный PR из `v1.0.0` в `main`.
6. Использовать Conventional Commits в commit messages и PR titles.

`v1.0.0` публикуется как стабильный release tag, не как prerelease. Если до публикации tag `v1.0.0` найден блокер, исправление остается в той же RC branch. Если tag уже опубликован, исправление идет через новый patch/release-candidate.

---

## Что требуется от пользователя / владельца до старта

Перед началом delivery work владелец подтвердил ключевые решения и доступы:

- GitHub Pages должен иметь clean URL `speq-tms` без custom domain для `v1.0.0`.
- Для clean org/user Pages URL обычно нужен отдельный Pages repository вида `speq-tms.github.io`. Если использовать только текущий `speq-docs`, GitHub Pages даст project pages URL, а не clean org URL. Так как для `v1.0.0` требуется clean URL, ручное действие: создать или подтвердить repository `speq-tms.github.io` под GitHub Pages и публиковать docs/site через него.
- Custom domain для docs/site в `v1.0.0` не нужен.
- Ownership и write/admin permissions подтверждены для:
  - `speq-cli`;
  - `speq-contracts`;
  - `speq-examples`;
  - `speq-github-runner`;
  - `speq-vscode-extension`;
  - `speq-docs`;
  - `homebrew-tap`.
- RC branch `v1.0.0` создана в каждом затронутом репозитории от `main`.
- Release policy подтверждена: `v1.0.0` публикуется как stable tag, не prerelease.
- Имена GitHub Secrets для CI examples и cookbook фиксируются в документации в рамках delivery work.
- Naming release artifacts для CLI по платформам фиксируется перед ручной публикацией и должен быть одинаковым в docs, runner и tap.
- Windows install MVP подтвержден: только zip/manual install, без package manager.
- Credentials/tokens для Homebrew tap, GitHub Releases, GitHub Pages и Marketplace не автоматизируются в рамках этого документа: публикации выполняются вручную владельцем.
- Assignment release owner/reviewer для Go/No-Go остается вне документа и управляется пользователем вручную.

Работы можно стартовать от зафиксированной RC branch и подтвержденных решений. Открытыми остаются только ручные операционные действия публикации: создание/подтверждение `speq-tms.github.io`, финальная публикация artifacts/site/listings и распределение owner/reviewer пользователем вне документа.

---

## MVP scope по репозиториям

### `speq-cli`

Scope:

- Стабильный build/test для Linux, macOS и Windows.
- Release workflow с бинарными artifacts для:
  - `linux-x86_64`;
  - `macos-aarch64`;
  - `macos-x86_64`;
  - `windows-x86_64`.
- Предсказуемые checksum files для release artifacts.
- Совместимость с `speq-examples/test-repo-mode-jsonplaceholder`.
- Базовые команды CLI UX:
  - `speq version`;
  - `speq help`.

`version` и `help` являются обязательным стандартом CLI для `v1.0.0`. Реализация должна быть максимально простой:

- `speq version` печатает текущую версию CLI и завершается успешно;
- `speq help` показывает краткую справку по доступным командам;
- без plugin system, auto-update, remote metadata, telemetry или сложного форматирования;
- поведение должно быть доступно в release artifact и проверяться smoke tests.

Delivery branches:

- `feat/cli-version-help`
- `chore/cli-release-matrix`
- `fix/cli-jsonplaceholder-regressions`

Acceptance criteria:

- `cargo build` и `cargo test` проходят.
- Release artifacts публикуются для всех MVP-платформ.
- `speq version` и `speq help` работают локально и из release binary.
- В `speq-examples/test-repo-mode-jsonplaceholder` прогон `speq run --environment ci` завершается с `"failed": 0`.

### `speq-docs`

Scope:

- Публичный docs/site через GitHub Pages с clean URL `speq-tms`, без custom domain.
- Для clean org/user Pages URL требуется создать или подтвердить отдельный repository `speq-tms.github.io`. Публикация из `speq-docs` напрямую допустима только если будет принят project pages URL; для `v1.0.0` это не целевое решение.
- Implementation details зафиксированы в `speq-docs/docs/delivery/gh-pages.md`.
- Quickstart для установки CLI и первого запуска.
- Cookbook для environment YAML в CI.
- Release/install инструкции для Homebrew, GitHub runner и Windows zip/manual path.
- Страница compatibility/version alignment для CLI, runner, examples и extension.

Delivery branches:

- `chore/docs-gh-pages`
- `chore/docs-install-quickstart`
- `chore/docs-ci-env-cookbook`

Acceptance criteria:

- Docs доступны по clean GitHub Pages URL через `speq-tms.github.io`.
- Quickstart можно пройти без знания внутренних репозиториев.
- В документации явно описаны секреты, environments и release artifacts.

### `speq-examples`

Scope:

- Canonical example `test-repo-mode-jsonplaceholder` остается главным AT gate.
- Добавить или обновить примеры CI, где environment YAML генерируется из GitHub Secrets.
- Зафиксировать expected reports paths для summary/results/allure.

Delivery branches:

- `chore/examples-ci-env`
- `fix/examples-at-gates`

Acceptance criteria:

- Existing examples не ломаются.
- JSONPlaceholder AT показывает `"failed": 0`.
- CI example не содержит реальных секретов и использует documented secret names.

### `speq-github-runner`

Scope:

- Runner устанавливает или использует нужную версию CLI предсказуемо.
- Reference workflow показывает:
  - setup/install;
  - generation of environment YAML from secrets;
  - `speq run`;
  - upload summary/allure artifacts.
- Runner docs aligned с `speq-cli` `v1.0.0`.

Delivery branches:

- `chore/runner-v1-cli-install`
- `chore/runner-ci-env-example`

Acceptance criteria:

- Workflow на GitHub-hosted runner проходит с published или locally built CLI.
- Artifact upload содержит summary/results/allure.
- README не обещает функций вне MVP.

### `homebrew-tap`

Scope:

- Formula для `speq` с macOS Apple Silicon, macOS Intel и Linux x86_64 install path.
- URL и sha256 aligned с release artifacts `speq-cli`.
- Formula test использует простой CLI smoke:
  - `speq help`;
  - `speq version`.

Delivery branches:

- `chore/tap-speq-v1`

Acceptance criteria:

- `brew install speq` проходит на macOS.
- Homebrew formula smoke не требует network API calls сверх скачивания artifact.
- Formula version соответствует CLI release version.

### `speq-vscode-extension`

Scope:

- Extension остается interface layer only.
- Docs/README явно указывают поддерживаемую CLI version.
- Diagnostics/run behavior не расходится с `speq-cli` `v1.0.0`.

Delivery branches:

- `chore/extension-v1-version-docs`
- `fix/extension-cli-compatibility`

Acceptance criteria:

- `npm run compile` проходит.
- Extension README объясняет, что CLI должен быть установлен отдельно или доступен в PATH.
- Нет собственного runtime behavior, дублирующего CLI.

### `speq-contracts`

Scope:

- Только alignment схем, если `v1.0.0` delivery меняет DSL/results contracts.
- Не добавлять новые contract features без прямой необходимости для MVP.

Delivery branches:

- `fix/contracts-v1-alignment`

Acceptance criteria:

- CLI и examples используют одну версию схем.
- Любое изменение contract сопровождается обновлением docs и fixtures.

---

## Parallelizable workstreams

Ниже пункты, которые можно брать параллельно после создания RC branches и фиксации ручных решений владельца:

- Docs/site: GitHub Pages, quickstart, install pages, release notes draft.
- Env/CI cookbook: documented secret names, generated environment YAML, workflow snippets.
- CLI `version`/`help`: простые команды, smoke tests, release binary check.
- Multi-platform CLI release: CI matrix, artifacts, checksums, naming.
- Homebrew tap update: formula, sha256 wiring, formula smoke.
- Runner examples/setup: install CLI, run examples, upload reports.
- Extension/version docs: compatibility note, README updates, no runtime expansion.
- Examples/AT gates: JSONPlaceholder acceptance run, reports cleanup, CI examples.

Ограничения параллельности:

- `homebrew-tap` зависит от финальных CLI artifact names и checksums.
- Runner install docs зависят от CLI release URL и version policy.
- Docs quickstart зависит от выбранного public URL и Windows install stance.
- Extension compatibility docs зависят от финального CLI version alignment.

---

## Environment and secrets MVP

Для `v1.0.0` environment/secrets model в CI должен быть практичным и прозрачным:

- В репозиторий не коммитятся реальные secrets.
- CI workflow генерирует environment YAML из GitHub Secrets во время job.
- Secret names документируются в `speq-docs`.
- Example workflow показывает минимальный happy path.
- Generated environment file удаляется или остается только в ephemeral runner workspace.

Acceptance criteria:

- Пользователь может скопировать workflow snippet и заменить только secret values.
- Логи не печатают secret values.
- Example не требует дополнительных сервисов кроме JSONPlaceholder и GitHub Actions.

---

## Manual publication MVP

Для `v1.0.0` публикации остаются ручными и не требуют автоматизированных credentials/tokens в репозиториях:

- GitHub Releases публикуются вручную после успешных RC gates.
- Homebrew tap обновляется вручную по финальным release artifact URLs и checksums.
- GitHub Pages site публикуется вручную через подтвержденный repository `speq-tms.github.io`.
- Marketplace publication для extension выполняется вручную, если extension входит в конкретный release pass.
- Документ не назначает owner/reviewer names для Go/No-Go; распределение остается вне документа и управляется пользователем вручную.

---

## Version alignment

Для релиза `v1.0.0` нужно зафиксировать единую таблицу совместимости:

- `speq-cli`: `v1.0.0`
- `speq-github-runner`: `v1` или `v1.0.0`, по ручному release decision перед публикацией
- `speq-vscode-extension`: версия, совместимая с CLI `v1.0.0`
- `speq-contracts`: schema version, используемая CLI и examples
- `speq-examples`: commit/tag, проверенный acceptance gates
- `homebrew-tap`: formula version `1.0.0`

Release не считается готовым, если runner, docs или tap указывают на несуществующий CLI artifact.

---

## Go/No-Go checklist

### Go

- Все затронутые PR влиты в RC branches.
- CI green в каждом затронутом репозитории.
- CLI artifacts доступны для Linux/macOS/Windows.
- `speq help` и `speq version` проходят smoke из release binary.
- Homebrew install проходит минимум на macOS.
- Runner workflow успешно запускает example suite.
- JSONPlaceholder acceptance run показывает `"failed": 0`.
- Docs опубликованы и содержат install, CI env и version alignment instructions.
- Go подтвержден пользователем вручную; assignment owner/reviewer остается вне документа.

### No-Go

- Не создан или не подтвержден repository `speq-tms.github.io` для clean Pages URL.
- Нет финального решения по artifact naming.
- Любой release artifact отсутствует или имеет неверный checksum.
- Runner/docs/tap ссылаются на разные версии CLI.
- Acceptance examples падают.
- Документация требует ручных знаний, которых нет в quickstart/cookbook.
- `speq help` или `speq version` не работают в release binary.

---

## Out of scope для `v1.0.0`

- Сложная installer story для Windows beyond zip/manual install.
- Auto-update в CLI.
- Telemetry/usage analytics.
- Plugin system.
- Pro-only CI/CD возможности.
- Новые DSL features, если они не нужны для текущих acceptance gates.
- Полная документационная платформа сверх минимального публичного site/quickstart/cookbook.

