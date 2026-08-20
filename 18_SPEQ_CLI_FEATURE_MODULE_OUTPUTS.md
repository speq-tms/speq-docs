# speq cli feature: module outputs for use actions

## Цель

Расширить `use action`, чтобы модуль не только принимал параметры, но и возвращал значения в контекст теста для дальнейшего использования.

## Проблема сейчас

Текущий `use` удобен для композиции шагов, но не позволяет напрямую переиспользовать данные из response модуля (например токен, id сущности).

## MVP контракт

Добавить в шаг `use` поле `as` для bind результата:

```yaml
steps:
  - type: use
    action: auth.login
    properties:
      user: "john"
      pass: "secret"
    as: login
  - type: api
    method: GET
    url: "/users/me"
    headers:
      Authorization: "Bearer {{login.token}}"
```

И добавить в описание action поддержку `returns`:

```yaml
actions:
  login:
    properties:
      - name: user
        required: true
      - name: pass
        required: true
    steps:
      - type: api
        id: login_call
        method: POST
        url: "/auth/login"
        body: { user: "{{user}}", pass: "{{pass}}" }
    returns:
      token: "$steps.login_call.response.body.access_token"
      userId: "$steps.login_call.response.body.user.id"
```

## Runtime semantics

- `use` исполняет action steps как сейчас;
- после исполнения вычисляет `returns` выражения;
- если указан `as`, кладет объект в контекст (`login.token`, `login.userId`);
- если `returns` не описан, поведение legacy (ничего не возвращается).

## Ошибки и диагностика

- если `returns` expression не резолвится -> `module_return_resolution_error`;
- если `as` конфликтует с существующей переменной:
  - fail fast в MVP (без auto-override);
- `validate` проверяет, что `as` - валидный identifier.

## Backward compatibility

- существующие модули без `returns` и вызовы без `as` работают без изменений;
- `use.ref` legacy путь сохраняется.

## Acceptance criteria

- минимум 3 e2e кейса:
  - auth token flow;
  - create entity and reuse id;
  - module without returns (legacy).
- diagnostics дают точный путь ошибки в `returns`;
- VS Code diagnostics/completion обновлены под `as` и `returns`.
