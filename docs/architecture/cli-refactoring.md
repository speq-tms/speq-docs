Рассмотрим предложенные расширения подробнее, опираясь на текущую архитектуру `speq` и лучшие практики.

---

### 1. Ассерт `schema`: хранение и формат

**Проблема:** Встраивать JSON Schema в тело YAML-теста — это громоздко, снижает читаемость и затрудняет переиспользование.  
**Решение:** хранить схемы отдельно от тестов, а в шаге указывать на них ссылку.

#### 📁 Где хранить
В корне проекта предлагаю завести директорию `schemas/` (аналогично `environments/` и `suites/`). Её путь можно переопределить в манифесте:

```yaml
# manifest.yaml (фрагмент)
schemasDir: "contracts/schemas"
```

Внутри — произвольная вложенность, например, по доменам:
```
project/
  schemas/
    user.json
    order-v1.yaml
    common/
      error.yaml
      pagination.json
```

#### 📄 В каком формате
Поддерживаются обычные **JSON Schema**-файлы (`.json` или `.yaml`).  
Никаких обёрток не требуется: каждая схема — это полноценный JSON Schema-документ, готовый к использованию в валидаторах.

#### 🧩 Как использовать в тесте
Добавляем новый тип ассерта — `schema` — с полем `ref`, указывающим путь к файлу схемы относительно `schemasDir`:

```yaml
steps:
  - type: api
    method: GET
    url: "/users/{{userId}}"
    assert:
      - type: status
        expected: 200
      - type: schema
        ref: "user.json"
      - type: schema
        ref: "common/error.yaml"   # если ожидаем ошибку
```

Альтернативно, для простых случаев можно разрешить **inline-схему**:
```yaml
      - type: schema
        inline:
          type: object
          required: [id, name]
          properties:
            id:   { type: integer }
            name: { type: string }
```
Но основной сценарий для «чистого» теста — именно ссылка.

#### ⚙️ Детали реализации
* На этапе запуска `speq` загружает схему из указанного файла и валидирует тело ответа (или его часть, если задан `extract`).
* Если `ref` начинается с `speq://`, можно адресовать предопределённые схемы из `speq-contracts` (например, `speq://common/error-v1`).
* Кэширование схем на время выполнения сьюта для производительности.

---

### 2. Модули и импорты: переиспользование в YAML

Текущий механизм `use` c `ref` позволяет ссылаться на конкретный шаг — это уже простейшая форма модульности.  
Предлагаю развить её до полноценной системы без разрушения обратной совместимости.

#### 📦 Модули как файлы с переиспользуемыми сущностями
Директория `modules/` (настраивается в манифесте `modulesDir`), внутри — YAML-файлы, каждый из которых экспортирует именованные элементы:

```yaml
# modules/auth.yaml
actions:
  login:
    - type: api
      method: POST
      url: "/auth/login"
      body: { user: "{{user}}", pass: "{{pass}}" }
      extract:
        token: "$.access_token"
  logout:
    - type: api
      method: POST
      url: "/auth/logout"
      headers:
        Authorization: "Bearer {{token}}"

variables:
  user: "testuser"
  pass: "defaultpass"
```

#### 🔗 Импорт в тесте или на уровне сьюта
Два способа:

**1. Импорт на уровне теста (директива `imports`)**  
В начале файла теста перечисляем, какие модули нужны, и даём им алиасы:

```yaml
# suites/smoke/user.yaml
imports:
  - module: auth
    alias: auth
  - module: common/helpers
    alias: helpers
steps:
  - type: use
    action: auth.login
  - type: api
    method: GET
    url: "/users/me"
    ...
  - type: use
    action: auth.logout
```

**2. Импорт по умолчанию**  
В манифесте или в `init.yaml` сьюта можно указать модули, доступные всем тестам сьюта без явного импорта (через поле `globalImports`). Это уменьшит дублирование.

#### 🧠 Разрешение ссылок
- При парсинге теста `action: auth.login` ищется среди импортированных модулей (по алиасам), затем в текущем файле (если определён `steps:` с идентификаторами), затем в глобальных.
- Переменные модуля подключаются в контекст теста (и могут быть переопределены локально).
- Специальный тип шага `use` не меняется — он просто выполняет список шагов, определённый в модуле.

#### 🌟 Дополнительные возможности
- Импорт переменных: `imports: [module: common/env]` только ради переменных.
- Функции-генераторы: модуль `faker` с действием `faker.email` (позже).
- Возможность наследования: один модуль может импортировать другой.

Таким образом, тесты остаются декларативными, без «кода» в привычном смысле, но получают мощный механизм композиции.

---

### 3. Императивные хуки и `init.yaml`

Предложенное вами видение — файл `init.yaml` в корне сьюта — очень органично и логично продолжает архитектуру.

#### 📂 Структура сьюта с хуками
```
suites/
  my-suite/
    init.yaml
    test1.yaml
    test2.yaml
    nested-suite/
      init.yaml
      test3.yaml
```

#### 🎬 Содержимое `init.yaml`
Файл определяет хуки, общие переменные и, возможно, локальные вспомогательные действия для сьюта:

```yaml
# suites/my-suite/init.yaml
suite:
  variables:
    baseUrl: "http://api.example.com/v1"
    defaultTimeout: 10s

  beforeAll:
    - type: use
      action: auth.login
    - type: api
      method: POST
      url: "/testdata/reset"

  afterAll:
    - type: api
      method: DELETE
      url: "/testdata/cleanup"

  beforeEach:
    - type: use
      action: auth.login

  afterEach:
    - type: use
      action: auth.logout

  # необязательный импорт модулей для всех тестов сьюта
  imports:
    - module: auth
```

#### 🧮 Порядок выполнения
Обработчик сьюта (в `speq`) перед запуском каждого теста выполняет:
1. `beforeAll` (однократно для сьюта, если ещё не выполнялся)
2. Для каждого теста:
   - `beforeEach` (из текущего сьюта и **всех родительских**, начиная с корня)
   - сам тест
   - `afterEach` (в обратном порядке: сначала текущий, потом родительские)
3. `afterAll` (после всех тестов сьюта)

При вложенности хуки наследуются:
- `beforeAll`/`afterAll` выполняются только для того сьюта, где определены, но не для родительского.
- `beforeEach`/`afterEach` каскадируются: запускаются все, от самого внешнего к внутреннему (и обратно для after).

#### 🔧 Интеграция с существующими механизмами
- Хуки — это те же списки шагов (`actions`), их можно писать так же, как и тесты: с ассертами (если ассерт упадёт — выполнение прервётся), с вызовом `use`, с эсктрактами переменных.
- Переменные, определённые в `init.yaml`, становятся глобальными для всех тестов сьюта, но могут быть перекрыты локальными.
- Если `init.yaml` отсутствует, сьют работает как раньше — без хуков.

#### 📝 Альтернативный взгляд: отдельная папка для хуков
Если хуки становятся слишком объёмными, можно вынести их в отдельный файл `hooks.yaml` рядом с `init.yaml`, но предложенный подход с единым `init.yaml` — самый простой и прозрачный для начала.

---

Все три предложения сохраняют «чистоту» YAML-тестов, используют существующие концепции фреймворка (пути, `use`, манифест) и могут быть реализованы итерационно. Если нужна более детальная проработка какого-то из направлений (например, проектирование контракта для `init.yaml` в `speq-contracts`) — готов обсудить.

---

## Финальные решения для v1 (зафиксировано)

Ниже зафиксированы согласованные рамки и правила внедрения для первой версии расширения `speq`.

### 1) Scope v1

Входит в реализацию:

- хранение схем отдельно от тестов и проверка через `assert: schema`;
- обновленная структура `suites` с поддержкой `init.yaml` и suite hooks;
- более нативные импорты модулей и использование действий через `use action`.

Явно вне scope v1:

- интеграции с DB/brokers;
- advanced data generators.

### 2) DSL contracts freeze

Контракт DSL для v1 считается зафиксированным на основе данного документа (`docs/architecture/cli-refactoring.md`):

- `schema` assert (`ref` как основной сценарий, `inline` как опциональный);
- `modules` + `imports` + `use action`;
- `use.properties` для параметризации module action на уровне вызова;
- `init.yaml` для suite-level hooks/variables/imports (`suite.imports`).

Изменения DSL во время реализации допускаются только как bugfix-уточнения без расширения scope.

### 3) Execution semantics (v1)

#### Приоритет переменных

Порядок приоритета значений (от более сильного к менее сильному):

1. параметры вызова action (`use.properties`);
2. переменные теста (`test.variables`);
3. переменные импортированных модулей;
4. переменные сьюта (`init.yaml`);
5. переменные окружения (`environments/*.yaml`).

#### Поведение хуков при ошибках

- Ошибки в `beforeAll` и `beforeEach` трактуются как setup failures и приводят к fail соответствующих тестов.
- Ошибки в `afterEach` и `afterAll` трактуются как teardown failures и также приводят к fail соответствующих тестов.
- В отчетах и диагностике причина должна явно маркироваться как `setup` или `teardown`.

### 4) Migration strategy

Для v1 принимается стратегия immediate GA:

- без experimental flags;
- без режима "preview only";
- с сохранением обратной совместимости там, где это уже гарантировано текущим контрактом.

### 4.1) Tooling sync note (CLI + VS extension)

Чтобы не потерять согласованность между движком и IDE-плагином, для `vs-extension` нужно адаптировать:

- parser/validator схемы YAML под новые поля:
  - `step.properties` для `type: use`;
  - `suite.imports` в `init.yaml`;
  - action-контракт модуля: `actions.<name>.properties[]` + `actions.<name>.steps[]` (с сохранением legacy-формата `actions.<name>: Step[]`);
- подсказки и completion:
  - для `action` в `use`;
  - для ключей `properties` на основе `actions.<name>.properties`;
- diagnostics:
  - ошибка при отсутствии required properties для action;
  - ошибка при неразрешимом `action` alias;
  - отдельная маркировка setup/teardown failures для hook execution.

### 5) CI gates

Обновление CI quality gates выполняется после реализации функционального слоя:

- contract/regression проверки для нового DSL;
- e2e на hooks/imports/schema;
- обновленные smoke сценарии для `in-repo` и `test-repo`.

---

## Execution checklist (issue-ready)

Ниже — декомпозиция задач для реализации в `speq-cli` в порядке выполнения. Каждый пункт можно выносить в отдельный issue/PR.

### Epic A: Manifest и базовые директории

**A1. Extend manifest contract**

- Добавить поля:
  - `schemasDir` (default: `schemas`);
  - `modulesDir` (default: `modules`).
- Обновить чтение/валидацию манифеста в `src/manifest/mod.rs`.
- Acceptance:
  - старые манифесты без новых полей валидны;
  - новые поля корректно подхватываются в runtime.

### Epic B: Parser/DSL расширение

**B1. Add schema assertion fields**

- Расширить `Assertion`:
  - `ref` для внешней схемы;
  - `inline` для опциональной inline-схемы.
- Валидация parser:
  - для `type: schema` должен быть задан `ref` или `inline`;
  - для остальных assert-типов поведение без изменений.
- Acceptance:
  - `speq validate` корректно валидирует `schema` assert;
  - ошибочные комбинации дают точные сообщения.

**B2. Add imports and use-action support**

- Расширить `TestSpec` полем `imports`.
- Расширить `Step`:
  - `action` для `type: use` (новый нативный путь);
  - legacy `ref` сохранить для обратной совместимости.
- Acceptance:
  - parser принимает оба режима `use` (`action` и legacy `ref`);
  - явные ошибки при отсутствии target action/ref.

### Epic C: Suite planner и hooks

**C1. Implement suite init model**

- Добавить модель `init.yaml`:
  - `suite.variables`;
  - `suite.imports`;
  - `beforeAll`, `beforeEach`, `afterEach`, `afterAll`.
- Acceptance:
  - `init.yaml` читается/валидируется как отдельный тип файла.

**C2. Exclude init.yaml from test collection**

- Обновить file collection (`src/cli/files.rs` и потребители) так, чтобы `init.yaml` не считался тестом.
- Acceptance:
  - `list/validate/run` не пытаются парсить `init.yaml` как `TestSpec`.

**C3. Build execution plan with nested hooks**

- Реализовать planner для дерева suites:
  - наследование `beforeEach/afterEach`;
  - локальные `beforeAll/afterAll` на уровне конкретного suite.
- Acceptance:
  - соблюден порядок исполнения hooks для вложенных suites;
  - порядок documented и покрыт regression tests.

### Epic D: Runtime (runner) поддержка

**D1. Implement schema assertion runtime**

- Добавить в runner обработку `type: schema`:
  - загрузка схем из `schemasDir` по `ref`;
  - поддержка `inline`;
  - кэш схем в рамках одного run.
- Acceptance:
  - schema validation работает на JSON-ответах;
  - ошибки schema-check содержат путь и причину.

**D2. Implement native action resolution**

- Добавить резолв `use action` через imports/modules.
- Сохранить legacy `use ref` для совместимости.
- Acceptance:
  - `use action` исполняется через модульный реестр;
  - legacy сценарии продолжают работать.

**D3. Hook failure classification**

- Внести в результаты классификацию причин:
  - setup failure (`beforeAll/beforeEach`);
  - teardown failure (`afterEach/afterAll`).
- Fail policy:
  - setup/teardown ошибки фейлят соответствующие тесты.
- Acceptance:
  - в summary/diagnostics видно тип причины (`setup`/`teardown`).

### Epic E: CLI команды и UX

**E1. Update validate/list/run/report integration**

- `validate`: добавить проверки hooks/imports/schema refs.
- `list`: строить список тестов через suite planner.
- `run`: исполнять тесты через execution plan (hooks + test body).
- `report`: убедиться, что новые failure-reasons отражаются в артефактах.
- Acceptance:
  - команды работают детерминированно на старых и новых фикстурах.

### Epic F: Examples, tests, CI gates (финальный этап)

**F1. Add canonical fixtures**

- Добавить фикстуры:
  - nested suite hooks;
  - modules/imports/use action;
  - schema success/failure;
  - backward compatibility (`use ref`).

**F2. Add regression/contract coverage**

- Unit + integration tests на parser/planner/runner.
- Contract tests для summary с новыми failure semantics.

**F3. Refresh CI gates**

- Обновить CI jobs после функциональной реализации:
  - новые regression suites;
  - e2e hooks/imports/schema;
  - smoke сценарии `in-repo` и `test-repo`.