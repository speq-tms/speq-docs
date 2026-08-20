# speq cli feature: retry policy and waiter

## Цель

Снизить flaky-падения и добавить управляемое ожидание eventual consistency сценариев в API тестах.

## MVP требования

- один общий retry-конфиг на весь проект;
- retry применяется к API шагам при сетевых/временных сбоях и configurable статусах;
- waiter подключается на уровне шага через `condition.wait` и переиспользует ту же retry-механику;
- без сложных per-step retry стратегий в первой версии.

## Глобальный retry в manifest

```yaml
retry:
  enabled: true
  maxAttempts: 3
  delayMs: 500
  backoff: exponential   # fixed|exponential
  retryOn:
    networkErrors: true
    statusCodes: [429, 502, 503, 504]
```

Если блок отсутствует:

- retry disabled по умолчанию (или `maxAttempts: 1` как эквивалент).

## Waiter в шаге с condition

```yaml
steps:
  - type: api
    method: GET
    url: "/jobs/{{jobId}}"
    condition:
      type: jsonpath
      path: "$.state"
      equals: "done"
      wait:
        timeoutMs: 30000
        intervalMs: 1000
```

Семантика:

- выполняется запрос;
- проверяется condition;
- если condition не выполнен, запускается цикл wait/retry до `timeoutMs`;
- успех при первом выполненном condition, иначе fail `wait_timeout`.

## Поведение retry + waiter

- retry применяется к каждому HTTP вызову;
- waiter управляет количеством повторных вызовов во времени до timeout;
- итоговый fail reason должен быть явным:
  - `retry_exhausted` (ошибка запроса);
  - `wait_timeout` (condition не достигнут вовремя).

## Валидация

`speq validate` проверяет:

- retry числа > 0;
- `backoff` в допустимом enum;
- `timeoutMs >= intervalMs`;
- корректность `condition` структуры.

## Отчетность и UX

- в summary фиксировать:
  - `attemptsUsed`;
  - `waitDurationMs`;
  - финальную причину fail;
- в логах отображать сокращенно: попытка, задержка, статус.

## Acceptance criteria

- flaky API сценарий стабилизируется с retry;
- eventual consistency сценарий проходит через waiter;
- присутствуют тесты на:
  - успех на N-ой попытке;
  - исчерпание retry;
  - timeout waiter;
- нет регрессий для тестов без retry/condition.wait.
