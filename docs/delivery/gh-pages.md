# speq GitHub Pages implementation

## Решение

Для MVP `v1.0.0` публичная документация публикуется через отдельный GitHub repository:

- owner/org: `speq-tms`;
- repository: `speq-tms.github.io`;
- production URL: `https://speq-tms.github.io/`.

Это решение нужно для clean org/user GitHub Pages URL без custom domain. Репозиторий `speq-docs` остается source/engineering docs и может быть источником контента, но не является production Pages repository для `v1.0.0`.

Альтернатива: включить GitHub Pages прямо в `speq-docs`. В таком случае URL будет project pages URL вида `https://speq-tms.github.io/speq-docs/`. Это не соответствует требованию clean URL `speq-tms`, поэтому для MVP не выбирается.

---

## Release flow

Если `speq-tms.github.io` ведется как отдельный delivery repository, для него применяется общий release flow:

1. Создать repository `speq-tms.github.io` вручную в GitHub owner/org `speq-tms`.
2. Создать branch `v1.0.0` от `main`.
3. Создавать delivery branches только от `v1.0.0`, например `chore/docs-gh-pages`.
4. Открывать PR из delivery branch в `v1.0.0`.
5. После готовности открыть финальный PR из `v1.0.0` в `main`.
6. Использовать Conventional Commits в commit messages и PR titles.

Если repository создается пустым, сначала нужно добавить минимальный scaffold в `main`, затем создать от него `v1.0.0`.

---

## MVP scaffold

Для `v1.0.0` не нужен сложный documentation generator. Рекомендуемый путь:

- начать со static site: `index.html`, несколько HTML/Markdown страниц и простой CSS;
- держать структуру максимально плоской и ручной;
- не добавлять MkDocs/Docusaurus до появления реальной потребности в навигации, версии документации или поиске.

Минимальная структура:

```text
/
  index.html
  quickstart.html
  install.html
  ci-secrets.html
  examples.html
  assets/
    styles.css
```

Минимальный контент:

- landing: что такое SPEQ и для кого он;
- quickstart: установка, первый YAML test, первый `speq run`;
- install: Homebrew, GitHub Release artifacts, Windows zip/manual path;
- ci/secrets: GitHub Actions example, documented secret names, generated environment YAML;
- examples: ссылка на canonical examples и JSONPlaceholder acceptance scenario.

Контент из `speq-docs` для MVP переносится вручную: curated copy/paste только тех частей, которые нужны пользователю quickstart/site. Автоматическую синхронизацию между `speq-docs` и `speq-tms.github.io` в `v1.0.0` не делаем.

---

## GitHub Pages setup

Для MVP предпочтителен `Deploy from branch`, потому что static site не требует build step, package manager или deploy token. Это снижает риск перед релизом и хорошо подходит для ручной публикации.

Настройка после merge/push в `main`:

1. Открыть repository `speq-tms.github.io`.
2. Перейти в `Settings` -> `Pages`.
3. В `Build and deployment` выбрать:
   - Source: `Deploy from a branch`;
   - Branch: `main`;
   - Folder: `/ (root)`.
4. Сохранить настройки.
5. Дождаться Pages deployment в `Actions` или в блоке статуса `Settings` -> `Pages`.
6. Проверить URL `https://speq-tms.github.io/`.

Если позже появится generator или build step, можно перейти на GitHub Actions deploy. Для `v1.0.0` это не требуется.

---

## Критерии готовности

- Repository `speq-tms.github.io` создан или подтвержден.
- Branch `v1.0.0` создан от `main`, если repository ведется по общему release flow.
- Site scaffold добавлен через delivery PR и влит в RC branch.
- Финальный PR из `v1.0.0` в `main` влит.
- GitHub Pages включен из `main` root.
- `https://speq-tms.github.io/` открывается без custom domain.
- Site содержит landing, quickstart, install, ci/secrets и examples.
- Quickstart можно пройти без знания внутренних engineering docs.

---

## Out of scope для `v1.0.0`

- Custom domain.
- Сложный documentation generator.
- Автоматическая синхронизация из `speq-docs`.
- Search, versioned docs и analytics.
- Package manager для Windows beyond zip/manual install.
