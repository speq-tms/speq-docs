# speq cli feature: parallel test execution

## Цель

Сократить общее время прогона при большом количестве тестов за счет параллельного выполнения.

## MVP требования

- добавить аргумент CLI: `--parallel <n>`;
- default: `2` (если аргумент не передан);
- максимум: `10` (значения выше ограничиваются или дают validation error);
- единица параллелизма: файл теста;
- порядок в summary остается детерминированным.

## CLI UX

Примеры:

```bash
speq run
speq run --parallel 4
speq run --parallel 10 --suite suites/regression
```

Ошибки:

- `--parallel 0` -> invalid argument;
- `--parallel > 10` -> invalid argument (или auto-cap до 10, решение фиксируется до реализации).

## Runtime модель

- планировщик формирует очередь тестов;
- worker pool размера `parallel` выполняет тесты конкурентно;
- каждый тест имеет изолированный runtime context;
- aggregate reporter потокобезопасно собирает результаты.

## Интеграция с hooks/retry/waiter

- retry/waiter работают внутри каждого worker независимо;
- suite-level hooks, которые имеют side-effects, требуют прозрачной политики:
  - `beforeAll/afterAll` выполняются один раз на suite;
  - тесты из одного suite не стартуют до завершения `beforeAll`.
- в MVP важно сохранить корректность > максимальной скорости.

## Ограничения MVP

- без sharding между процессами/машинами;
- без приоритетов по длительности тестов;
- без fail-fast отключения всех workers (опционально для следующих фаз).

## Валидация и наблюдаемость

- `validate` не меняется;
- `run` логирует уровень параллелизма;
- summary содержит:
  - `parallelism`;
  - `totalDurationMs`;
  - per-test duration.

## Acceptance criteria

- на наборе из 20+ тестов время прогона заметно снижается относительно `parallel=1`;
- результаты и статусы идентичны последовательному запуску;
- нет data-race в отчетах/логах;
- regression test покрывает `parallel=1`, `parallel=2`, `parallel=10`.
