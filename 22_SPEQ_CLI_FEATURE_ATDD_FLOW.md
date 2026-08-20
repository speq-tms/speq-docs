# SPEQ ATDD Flow

## Цель

Описать поддержку Acceptance Test-Driven Development (ATDD) в экосистеме SPEQ: DSL-расширение, поведение CLI, интеграцию в lifecycle разработки.

ATDD в SPEQ — это процесс, при котором тест-артефакты создаются до реализации фичи и выступают исполняемой спецификацией. CLI умеет запускать такие "незавершённые" тесты, не ломая CI, и позволяет разработчику итерационно "зеленить" их по мере написания кода.

---

## Мотивация

Современный shift-left подход и рост AI-ассистированной разработки меняют порядок работы:

- Аналитик прорабатывает детали задачи (контракты, эндпоинты, валидации, тексты ошибок).
- Тестировщик на основе аналитического документа пишет декларативные АТ в SPEQ **до реализации**.
- Разработчик получает: ТЗ + исполняемые спецификации. Он пишет фичу и итерационно зеленит тесты.
- AI-агент при разработке ориентируется на прогон, а не на описание — тесты правятся аналитиком, код итерируется под тесты.

Это отличает ATDD от классического QA-подхода, где тесты пишутся после реализации.

---

## DSL: маркер `status: pending`

### Уровень теста

Тест может быть помечен как `pending` на уровне всего теста:

```yaml
name: create user returns 201
status: pending
request:
  method: POST
  url: "{{ base_url }}/users"
  body:
    name: John
    job: developer
expect:
  status: 201
  body:
    id: "{{ not_empty }}"
    name: John
```

### Уровень шага (step-level)

В multi-step тестах отдельный шаг может быть pending:

```yaml
name: user lifecycle
steps:
  - name: create user
    request:
      method: POST
      url: "{{ base_url }}/users"
      body:
        name: John
    expect:
      status: 201

  - name: delete user returns 204
    status: pending
    request:
      method: DELETE
      url: "{{ base_url }}/users/{{ steps.create_user.response.body.id }}"
    expect:
      status: 204
```

### Правила DSL

- `status: pending` является опциональным полем на уровне теста и шага.
- При отсутствии поля тест/шаг считается активным (поведение не меняется).
- Значение `pending` является единственным нестандартным статусом; другие значения (например `skip`) — вне scope v1.0.0.
- Поле `status` не конфликтует с полем `expect.status` (HTTP-статус ответа).

---

## Поведение CLI

### Выполнение pending-тестов

Когда CLI встречает тест или шаг с `status: pending`:

1. HTTP-запрос **не выполняется**.
2. Тест/шаг репортится как `pending` — отдельная категория, не `failed`, не `passed`.
3. Exit code процесса **не меняется** из-за pending-тестов. `0` остаётся `0`, если нет реальных failed.
4. В консольный вывод добавляется секция `PENDING` с перечислением тестов.

### Разграничение статусов в репорте

| Статус | Причина | Влияет на exit code |
|---|---|---|
| `passed` | Все assertions прошли | Нет |
| `failed` | Assertion упала или неожиданный HTTP-статус | Да |
| `error` | Connectivity error, timeout, invalid URL | Да |
| `pending` | `status: pending` в DSL | Нет |

`error` (недоступный эндпоинт) и `failed` (ответ не соответствует ожиданиям) — разные статусы. Это важно в ATDD: тестировщик пишет тест до реализации, и эндпоинт может физически не существовать. Использование `status: pending` позволяет явно отделить "ещё не реализовано" от "сломано".

### Summary output

```
Tests: 12 passed, 3 pending, 0 failed, 0 error
```

Pending-тесты перечисляются отдельно с именем теста/шага и именем файла:

```
PENDING:
  - [tests/user_delete.yaml] delete user returns 204
  - [tests/admin_flow.yaml] admin can revoke token
  - [tests/admin_flow.yaml#step:3] cleanup session
```

### JSON summary

Поле `pending` добавляется в результирующий JSON:

```json
{
  "passed": 12,
  "failed": 0,
  "error": 0,
  "pending": 3,
  "total": 15
}
```

---

## ATDD lifecycle

### Роли и артефакты

```
Аналитик
  │
  │  Документ задачи:
  │  - контракты (request/response shapes)
  │  - эндпоинты (метод, путь, параметры)
  │  - статус-коды и тексты ошибок
  │  - бизнес-правила и граничные случаи
  ▼
Тестировщик
  │
  │  Пишет SPEQ-тесты на основе документа.
  │  Все новые тесты получают status: pending.
  │  Прогон: speq run → все pending, 0 failed → CI зелёный.
  │
  │  Передаёт разработчику:
  │  - ссылку на PR с тестами
  │  - документ задачи
  ▼
Разработчик
  │
  │  Пишет реализацию.
  │  Итерационно убирает status: pending из тестов.
  │  Прогон: speq run → тесты переходят в passed или failed.
  │  Цель: все тесты passed, 0 pending в финальном PR.
  ▼
Review / CI Gate
  │
  │  Финальный прогон без pending-тестов.
  │  Go/No-Go: passed == total, pending == 0, failed == 0.
```

### Пример CI pipeline

```yaml
# .github/workflows/atdd-gate.yml

- name: Run SPEQ (ATDD gate)
  run: speq run --environment ci
  # Прогон не падает на pending тестах.
  # Падает только если failed > 0 или error > 0.

- name: Check no pending in release gate
  run: |
    PENDING=$(cat reports/results/summary.json | jq '.pending')
    if [ "$PENDING" -gt "0" ]; then
      echo "Release gate failed: $PENDING pending tests remain"
      exit 1
    fi
  # Этот шаг опционален — только для release-gate, не для feature branch.
```

Таким образом:
- Feature branch: pending-тесты допустимы, CI зелёный.
- Release gate: отдельный check на `pending == 0`.

---

## Интеграция с репортингом

### Allure

Pending-тесты маппятся в Allure-статус `skipped` с аннотацией `[ATDD: pending]` в названии или в description. Это позволяет видеть их в Allure UI как незавершённые спецификации.

### JSON summary

Файл `reports/results/summary.json` расширяется полем `pending` (см. выше).

### Консольный вывод

Pending-тесты не подавляются — они явно перечислены, чтобы разработчик видел, что именно предстоит реализовать.

---

## DSL-схема (speq-contracts)

`speq-contracts` расширяется следующим образом:

```yaml
# Для test-level
TestCase:
  properties:
    status:
      type: string
      enum: [pending]
      description: >
        Помечает тест как незавершённый (ATDD pending).
        HTTP-запрос не выполняется. Тест репортится как pending, не failed.

# Для step-level
Step:
  properties:
    status:
      type: string
      enum: [pending]
      description: >
        Помечает шаг как незавершённый.
        Шаг пропускается. Последующие шаги, зависящие от его вывода,
        также получают статус pending.
```

Зависимые шаги (использующие `{{ steps.pending_step.response.* }}`) автоматически становятся pending, так как их входные данные недоступны.

---

## Acceptance criteria для v1.0.0 RC

- `status: pending` на уровне теста признаётся CLI и не выполняет HTTP-запрос.
- `status: pending` на уровне шага пропускает шаг и каскадирует pending на зависимые шаги.
- Pending-тесты не влияют на exit code.
- Summary (консоль и JSON) содержит отдельный счётчик `pending`.
- Allure-репорт показывает pending-тесты как `skipped` с ATDD-меткой.
- `speq-contracts` обновлён схемой поля `status`.
- AT-пример в `speq-examples/test-repo-mode-jsonplaceholder` демонстрирует pending-тест.

---

## Out of scope для v1.0.0 RC

- `status: skip` (явный skip без ATDD-семантики) — отдельная фича следующего milestone.
- `--atdd` флаг запуска (изменение exit code поведения через CLI-флаг) — не нужен при наличии `status: pending` в DSL.
- Генерация pending-тестов из Swagger/OpenAPI автоматически — рассматривается в контексте Coverage фичи (doc 23).
- ATDD-метрики и трекинг динамики pending → passed во времени — post-MVP.
