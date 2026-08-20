# speq cli feature: fixtures and builders

## Цель

Добавить декларативные fixtures/builders в отдельной директории, чтобы тесты были короче и читабельнее, а генерация данных и schema references переиспользовались централизованно.

## MVP концепция

- новая директория по умолчанию: `fixtures/`;
- переопределение в манифесте: `fixturesDir`;
- fixture описывается YAML-файлом;
- fixture можно:
  - использовать в шаге API body;
  - переопределять точечно через `overrides`;
  - строить на основе `schema` + генераторов.

## Предлагаемый manifest extension

```yaml
fixturesDir: "fixtures"
```

## Предлагаемый формат fixture

```yaml
fixture:
  schemaRef: "user-create.yaml"
  build:
    id:
      gen: { type: uuid }
    email:
      gen: { type: email }
    name:
      gen: { type: name }
    createdAt:
      gen: { type: date-time, format: rfc3339 }
    isActive: true
```

`schemaRef` в MVP используется как ссылка на структуру/контракт (валидация результата materialization перед отправкой запроса).

## Использование fixture в тесте

```yaml
steps:
  - type: api
    method: POST
    url: "/users"
    bodyFromFixture:
      ref: "users/create-default.yaml"
      overrides:
        email: "custom@example.com"
```

## Runtime semantics

- `bodyFromFixture.ref` резолвится относительно `fixturesDir`;
- сначала собирается fixture `build`, затем применяются `overrides`;
- после merge выполняется optional schema validation (если указан `schemaRef`);
- итоговый JSON отправляется как `body`.

## Интеграция с генерацией данных

- fixtures напрямую используют `gen` блок из feature 15;
- генерация значений в fixture выполняется на каждый вызов шага;
- одинаковая fixture в двух шагах может возвращать разные значения (кроме явно фиксированных полей).

## Валидация

`speq validate` должен проверять:

- существование файла fixture по `ref`;
- корректность структуры `fixture.build`;
- корректность `overrides` (валидный YAML object);
- доступность `schemaRef` (если указан).

## Backward compatibility

- отсутствие `fixturesDir` в manifest = default `fixtures`;
- тесты без `bodyFromFixture` работают как сейчас.

## Acceptance criteria

- fixture подключается в `body` минимум в 3 e2e кейсах;
- `overrides` корректно переопределяют вложенные поля;
- ошибка в fixture не ломает весь run silently, а дает явный `fixture_resolution_error`;
- документация и примеры добавлены в `speq-examples`.
