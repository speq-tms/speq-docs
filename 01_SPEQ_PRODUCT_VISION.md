# speq product vision

## Что такое speq

`speq` — это тестовый фреймворк для команд, который позволяет внедрять автотесты без привязки к языку программирования и стеку технологий.

Ключевая идея:

- тесты и контракты описываются в YAML;
- исполнение единообразно через CLI;
- UI/CI/Editor — только интерфейсы к одному ядру.

## Product thesis

`one spec + one runner + many interfaces`

## Компоненты платформы

- `speq CLI` (open source) — ядро.
- `speq GitHub runner` (open source) — CI/CD запуск.
- `speq VS Code extension` (open source) — визуализация YAML-тестов.

## Ценность для команд

- Единый стандарт АТ для polyglot микросервисов.
- Повторяемый запуск локально и в CI.
- Быстрый onboarding QA/Dev/DevOps.
- DRY-подход через переиспользуемые блоки и helpers.

## Бизнес-направление

- Open Source: базовая разработка, запуск, валидация, отчеты.
- Paid (`speq-pro`): расширенные CI/CD и визуальный редактор.

## PAID (late phase, after OSS traction)

После запуска и роста трафика на open-source рельсах добавляется следующий слой `speq-pro`:

- Cloud storage результатов прогонов из CI/CD.
- История и сравнение запусков между ветками/релизами.
- Метрики уровня TestOps:
  - pass/fail trends,
  - flaky tests analytics,
  - duration and bottleneck tracking,
  - suite health score и стабильность релизных веток.

Важно: этот блок реализуется в самом конце, после подтверждения adoption OSS-части.
