# speq release and marketplace playbook

## Цель

Зафиксировать единый практический процесс релиза и публикации `speq` артефактов:

- `speq-cli` как installable package через Homebrew;
- `speq-vscode-extension` в VS Code Marketplace;
- `speq-github-runner` в GitHub Marketplace.

## Milestone summary (Apr 2026)

- `speq-cli`:
  - добавлен release workflow с упаковкой darwin/linux артефактов и checksum;
  - закрыты CI-регрессии на внешние пути (`speq-examples`, `speq-contracts`) через локальные fixtures;
  - зафиксирован alpha tag `v0.1.0-alpha.0`.
- `speq-github-runner`:
  - опубликован на GitHub Marketplace;
  - добавлен branding (`check-square`, `green`);
  - синхронизированы namespace/defaults на `speq-tms/*`.
- `speq-vscode-extension`:
  - опубликован OSS MVP scaffold (explorer/diagnostics/run/preview path через CLI).

---

## Daily git flow (team standard)

### Branching and merge

1. Разработка в feature branch (кроме экстренных hotfix).
2. PR в `main` с зелёным CI.
3. Merge только после review.

### Emergency hotfix in `main`

1. Прямой commit в `main` (если инцидент/блокер релиза).
2. Сразу запуск CI + smoke.
3. После стабилизации вернуть обычный PR flow.

### Commit style

- `feat:` — новая функциональность;
- `fix:` — исправление поведения/регрессии;
- `chore:` — infra/docs/техническое сопровождение без изменения продуктового поведения.

---

## 1) `speq-cli`: как релизить и отдавать через Homebrew

На первом этапе используем модель:

1. `speq-cli` публикует бинарные tar.gz артефакты в GitHub Releases;
2. Homebrew formula в отдельном tap (`speq-tms/homebrew-tap`) указывает на эти артефакты.

### 1.1 Release artifacts в `speq-cli`

Для каждого релиза `vX.Y.Z` публикуем минимум:

- `speq-darwin-aarch64.tar.gz`
- `speq-darwin-x86_64.tar.gz`

Опционально (для Linux через Homebrew on Linux):

- `speq-linux-x86_64.tar.gz`
- `speq-linux-aarch64.tar.gz`

Каждый tar.gz должен содержать исполняемый файл `speq` в корне или предсказуемом пути.

### 1.2 Homebrew tap

Создать отдельный публичный репозиторий:

- `speq-tms/homebrew-tap`

Структура:

```text
homebrew-tap/
  Formula/
    speq.rb
```

Пример formula:

```ruby
class Speq < Formula
  desc "Open-source CLI runtime for speq"
  homepage "https://github.com/speq-tms/speq-cli"
  version "0.1.0"

  on_macos do
    if Hardware::CPU.arm?
      url "https://github.com/speq-tms/speq-cli/releases/download/v0.1.0/speq-darwin-aarch64.tar.gz"
      sha256 "<sha256-darwin-aarch64>"
    else
      url "https://github.com/speq-tms/speq-cli/releases/download/v0.1.0/speq-darwin-x86_64.tar.gz"
      sha256 "<sha256-darwin-x86_64>"
    end
  end

  def install
    bin.install "speq"
  end

  test do
    assert_match "speq", shell_output("#{bin}/speq --help")
  end
end
```

### 1.3 Что делает пользователь

```bash
brew tap speq-tms/tap https://github.com/speq-tms/homebrew-tap
brew install speq
```

### 1.4 Release checklist для `speq-cli` + Homebrew

1. Выпустить GitHub Release в `speq-cli` с tar.gz и checksum.
2. Обновить `Formula/speq.rb` (version/url/sha256).
3. Запушить изменения в `homebrew-tap`.
4. Проверить clean install на macOS:
   - `brew update`
   - `brew install speq`
   - `speq --help`

---

## 2) `speq-vscode-extension`: публикация в VS Code Marketplace

### 2.1 Preconditions

В `speq-vscode-extension` должны быть:

- валидный `package.json` (`name`, `displayName`, `version`, `publisher`, `engines.vscode`);
- `README.md`, `LICENSE`;
- иконка extension (рекомендуется 128x128 PNG, поле `icon` в `package.json`).

### 2.2 Один раз: подготовка publisher и токена

1. Создать publisher в Visual Studio Marketplace.
2. Создать Personal Access Token (Azure DevOps) с правом `Marketplace (Manage)`.
3. Локально логин для `vsce`:

```bash
npx @vscode/vsce login <publisher-name>
```

### 2.3 Publish процесс

```bash
cd speq-vscode-extension
npm ci
npm run check
npx @vscode/vsce publish patch
```

Варианты версий:

- `publish patch` -> `x.y.(z+1)`
- `publish minor` -> `x.(y+1).0`
- `publish major` -> `(x+1).0.0`

### 2.4 Post-publish checks

1. Extension находится в Marketplace.
2. Установка в VS Code через `Extensions` UI проходит без ошибок.
3. Smoke в IDE:
   - `speq: Validate Workspace`
   - `speq: Run Suite`
   - `speq: Preview Manifest`

---

## 3) `speq-github-runner`: публикация в GitHub Marketplace

### 3.1 Обязательные условия

1. Публичный репозиторий.
2. В `action.yml` заданы:
   - `name`
   - `description`
   - `runs`
   - `branding.icon`
   - `branding.color`
3. Стабильный major tag (`v1`) для потребителей action.

### 3.2 Текущие branding значения

- `icon: check-square`
- `color: green`

### 3.3 Publish процесс

1. Убедиться, что `main` green по CI.
2. Создать release tag (`v1.0.0`, далее `v1.0.1` и т.д.).
3. Обновить major tag (`v1`) на latest stable.
4. При создании release включить опцию публикации action в Marketplace.

### 3.4 Consumer usage

```yaml
- uses: speq-tms/speq-github-runner@v1
  with:
    mode: run
    speq-root: .speq
    env: ci
```

---

## 4) Unified release order (recommended)

Для предсказуемости экосистемы:

1. Сначала `speq-cli` release (артефакты + Homebrew update).
2. Затем `speq-github-runner` release (с pin на новый `cli-version`, если нужно).
3. Затем `speq-vscode-extension` release.
4. Обновить `speq-docs` changelog/roadmap статус.

Так мы избегаем ситуации, когда runner/extension ссылаются на несуществующий CLI release.

---

## 5) Release day operations checklist

Ниже минимальный runbook для одного релизного дня.

### Роли

- **Release owner**: запускает релизы и контролирует порядок.
- **Reviewer**: подтверждает smoke и availability в marketplace.

### Шаги (в порядке выполнения)

1. Проверить, что CI green во всех 3 репозиториях:
   - `speq-cli`
   - `speq-github-runner`
   - `speq-vscode-extension`
2. Выпустить `speq-cli`:
   - создать tag/release;
   - проверить наличие tar.gz + checksum;
   - обновить `homebrew-tap` formula.
3. Выпустить `speq-github-runner`:
   - tag/release;
   - обновить `v1` major tag;
   - подтвердить listing в GitHub Marketplace.
4. Выпустить `speq-vscode-extension`:
   - `vsce publish`;
   - проверить доступность extension в VS Code Marketplace.
5. Smoke после релиза:
   - `brew install speq` + `speq --help`;
   - workflow с `uses: speq-tms/speq-github-runner@v1`;
   - установка extension из Marketplace и базовый `Validate Workspace`.
6. Обновить `speq-docs`:
   - changelog/roadmap status;
   - зафиксировать выпущенные версии (`cli`, `runner`, `extension`).

### Go/No-Go критерии

- **Go**: все публикации доступны, smoke прошел, rollback не требуется.
- **No-Go**: любой из артефактов недоступен/некорректен -> остановить цепочку, откатить до последней стабильной версии, завести инцидент issue.

---

## 6) Command cookbook (copy-paste)

### 6.1 `speq-cli` release

```bash
cd /path/to/speq-cli
git checkout main
git pull
git tag v0.1.0-alpha.0
git push origin v0.1.0-alpha.0
```

Если нужен manual run:

- workflow: `release`
- inputs:
  - `tag: v0.1.0-alpha.0`
  - `prerelease: true`

### 6.2 Update Homebrew tap after CLI release

```bash
curl -fsSL https://github.com/speq-tms/speq-cli/releases/download/v0.1.0-alpha.0/speq-darwin-aarch64.tar.gz.sha256
curl -fsSL https://github.com/speq-tms/speq-cli/releases/download/v0.1.0-alpha.0/speq-darwin-x86_64.tar.gz.sha256

cd /path/to/speq-cli
DARWIN_ARM_SHA256="<arm_sha256>" \
DARWIN_AMD_SHA256="<x86_sha256>" \
./scripts/generate_homebrew_formula.sh 0.1.0-alpha.0 > /tmp/speq.rb

cd /path/to/homebrew-tap
mkdir -p Formula
cp /tmp/speq.rb Formula/speq.rb
git add Formula/speq.rb
git commit -m "speq 0.1.0-alpha.0"
git push origin main
```

### 6.3 Validate Homebrew installation

```bash
HOMEBREW_NO_AUTO_UPDATE=1 brew tap speq-tms/tap https://github.com/speq-tms/homebrew-tap
HOMEBREW_NO_AUTO_UPDATE=1 brew reinstall speq
speq --help
```

### 6.4 `speq-github-runner` release

```bash
cd /path/to/speq-github-runner
git checkout main
git pull
git tag v1.0.0
git push origin v1.0.0
git tag -f v1
git push origin v1 --force
```

### 6.5 `speq-vscode-extension` release

```bash
cd /path/to/speq-vscode-extension
npm ci
npm run check
npx @vscode/vsce publish patch
```
