# speq cli feature: data generation

## Цель

Добавить простой и предсказуемый механизм генерации тестовых данных в YAML без написания кастомного кода.

Фокус первого этапа: примитивные генераторы без условий и сложных зависимостей.

## MVP требования

- поддержать генерацию:
  - `uuid` (`v4`);
  - `date-time` (`rfc3339`, `iso8601`);
  - `string` (случайная строка по длине);
  - `int` (min/max);
  - `bool`;
  - `email`;
  - `name`.
- генерация должна быть доступна в:
  - `test.variables`;
  - `step.body`;
  - fixtures (см. feature 16).
- дефолтно значения генерируются на каждый запуск теста заново.

## Предлагаемый DSL (MVP)

```yaml
variables:
  userId:
    gen:
      type: uuid
  createdAt:
    gen:
      type: date-time
      format: rfc3339
  randomName:
    gen:
      type: string
      minLength: 8
      maxLength: 16
  randomAge:
    gen:
      type: int
      min: 18
      max: 60
```

Встраивание в body:

```yaml
steps:
  - type: api
    method: POST
    url: "/users"
    body:
      id:
        gen: { type: uuid }
      email:
        gen: { type: email }
      createdAt:
        gen: { type: date-time, format: rfc3339 }
```

## Библиотеки (предпочтительно готовые)

Рекомендуется использовать готовые crate'ы:

- `uuid` - генерация UUID;
- `chrono` - date-time форматы;
- `fake` - email/name/random string шаблоны;
- `rand` - числовые диапазоны и базовая случайность.

Принцип: не писать собственный генератор там, где есть стабильная библиотека.

## Runtime semantics

- генераторы вычисляются перед выполнением шага, где они используются;
- сгенерированные значения попадают в общий контекст шаблонизации;
- в отчете можно маскировать/не логировать полностью динамические значения (по политике безопасного логирования);
- ошибки генерации должны падать как `generation_error` с указанием пути поля.

## Валидация

`speq validate` должен проверять:

- наличие `gen.type`;
- корректность параметров по типу генератора;
- диапазоны (`min <= max`, `minLength <= maxLength`);
- поддерживаемые форматы `date-time`.

## Backward compatibility

- существующие тесты без `gen` работают без изменений;
- `gen` - опциональное расширение DSL.

## Acceptance criteria

- генераторы работают в variables и body;
- deterministic validation errors при некорректной конфигурации;
- есть минимум 1 e2e test на каждый тип генератора;
- документация примеров добавлена в `speq-examples`.
