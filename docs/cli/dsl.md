# Справочник DSL

Полное описание YAML-языка тестов SPEQ: тест-спека, шаги, ассерты, переменные, генерация данных,
сьюты, модули и фикстуры.

Источник истины — код: `speq-cli/src/parser/mod.rs` (что принимается), `src/runner/mod.rs` (что
исполняется), `src/generator/mod.rs` (генерация). Расхождение этого документа с кодом считается багом
документа.

Если вы здесь впервые, начните с [quickstart.md](quickstart.md) — первый зелёный прогон за несколько
минут. Раскладка проекта и манифест — в [project-layout.md](project-layout.md). Команды и коды выхода —
в [README.md](README.md).

---

## Оглавление

- [Тест-спека](#тест-спека)
- [Порядок выполнения](#порядок-выполнения)
- [Шаг `type: api`](#шаг-type-api)
- [Шаг `type: use`](#шаг-type-use)
- [Ассерты](#ассерты)
- [Пути к полям ответа](#пути-к-полям-ответа)
- [Условия и ожидание](#условия-и-ожидание)
- [Переменные и подстановка](#переменные-и-подстановка)
- [Генерация данных](#генерация-данных)
- [Сьюты и `init.yaml`](#сьюты-и-inityaml)
- [Модули](#модули)
- [Фикстуры](#фикстуры)
- [ATDD: `status: pending`](#atdd-status-pending)
- [Чего DSL не умеет](#чего-dsl-не-умеет)

---

## Тест-спека

Один файл `.yaml` или `.yml` внутри `suitesDir` — один тест. Исключение: файлы `init.yaml` и `init.yml`,
это конфигурация сьюты (см. [ниже](#сьюты-и-inityaml)).

```yaml
id: "posts.get-by-id"
title: "GET /posts/1 возвращает пост"
tags: [smoke, posts]
variables:
  postId: 1
steps:
  - type: api
    name: "GET /posts/{{postId}}"
    method: GET
    url: "/posts/{{postId}}"
    assert:
      - type: status
        expected: 200
      - type: json
        path: "$.id"
        expected: 1
```

| Поле | Тип | Обязательно | Назначение |
| --- | --- | --- | --- |
| `id` | строка | да | Идентификатор теста в отчётах. Непустой |
| `title` | строка | да | Человекочитаемое название. Непустое |
| `tags` | массив строк | нет | Фильтрация через `--tags` |
| `markers` | массив строк | нет | Устаревший псевдоним `tags` |
| `variables` | объект | нет | Переменные теста; значение — любой JSON, включая `gen` |
| `imports` | массив | нет | Подключаемые модули |
| `setup` | массив шагов | нет | Шаги подготовки |
| `steps` | массив шагов | нет | Основные шаги |
| `cleanup` | массив шагов | нет | Шаги завершения |
| `status` | `pending` | нет | Тест пропускается целиком |

Хотя бы один из `setup`, `steps`, `cleanup` должен быть непустым — иначе:

```text
add at least one step in 'setup', 'steps' or 'cleanup' in <файл>
```

Неизвестные ключи парсер игнорирует молча. Опечатка в имени поля не вызовет ошибку — поле просто не
подействует. Опубликованная схема
[`speq-contracts/schemas/test/v1.json`](https://github.com/speq-tms/speq-contracts/blob/main/schemas/test/v1.json)
намеренно строже и подсвечивает такие опечатки в редакторе.

### `imports`

```yaml
imports:
  - module: jsonplaceholder     # файл в modulesDir, обязателен и непуст
    alias: jp                   # необязательно
```

Без `alias` именем становится последний сегмент пути модуля: `api/users` → `users`.
Имя файла модуля можно писать без расширения — CLI пробует `<имя>`, `<имя>.yaml`, `<имя>.yml`.

Если два импорта дают одинаковый алиас, побеждает **последний**: поиск идёт с конца списка.

---

## Порядок выполнения

Для каждого теста:

```text
beforeAll   ── один раз на сьюту, только из ближайшего init.yaml
beforeEach  ── перед каждым тестом; накапливается от корня к ближайшему init.yaml
  setup     ┐
  steps     ├ шаги самого теста
  cleanup   ┘
afterEach   ── после каждого теста; накапливается и выполняется в обратном порядке
afterAll    ── один раз на сьюту, только из ближайшего init.yaml
```

Два правила, которые легко упустить:

- `beforeAll` и `afterAll` берутся **только из ближайшего** `init.yaml`, а не собираются по всей цепочке.
  `beforeAll` в `suites/init.yaml` не выполнится для теста в `suites/posts/`, если в `suites/posts/init.yaml`
  тоже есть блок `suite`.
- `beforeEach` накапливается от корня к ближайшему, `afterEach` — накапливается и затем **разворачивается**,
  то есть выполняется от ближайшего к корню.

Падение `beforeAll` помечает все тесты сьюты как `failed` без их выполнения. Падение шага внутри теста
прекращает выполнение группы: остальные шаги `steps` не запускаются.

---

## Шаг `type: api`

HTTP-запрос.

```yaml
- type: api
  id: created                     # необязательно
  name: "POST /posts"             # обязательно, непустое
  method: POST
  url: "/posts"
  headers:
    Content-Type: "application/json"
  body:
    title: "Заголовок"
    userId: 1
  assert:
    - type: status
      expected: 201
```

| Поле | Обязательно | Примечание |
| --- | --- | --- |
| `name` | да | Непустое; попадает в отчёт |
| `method` | да | См. ниже |
| `url` | да | Непустой |
| `id` | нет | Используется выражениями `returns` **внутри модуля**; в тест-спеке ни на что не влияет |
| `headers` | нет | Только строковые значения |
| `body` | нет | Любой JSON; взаимоисключающ с `bodyFromFixture` |
| `bodyFromFixture` | нет | См. [Фикстуры](#фикстуры) |
| `assert` | нет | Список ассертов |
| `condition` | нет | См. [Условия и ожидание](#условия-и-ожидание) |
| `status` | нет | `pending` — шаг пропускается |

### `method`

Принимаются семь методов: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`.

Регистр не важен — `valid_http_method` приводит значение к верхнему, так что `get` и `Get` работают.
Канонично писать в верхнем регистре.

**`TRACE` не поддерживается.**

### `url`

Если значение начинается с `http://` или `https://`, оно используется как есть. Иначе к нему слева
приписывается `baseUrl` окружения (без завершающего `/`).

```yaml
url: "/posts/1"                              # baseUrl + /posts/1
url: "https://example.com/health"            # baseUrl игнорируется
```

### `headers`

Значения должны быть строками. Числовое значение в YAML вызовет ошибку разбора — экранируйте кавычками:

```yaml
headers:
  X-Retry-Count: "3"      # не 3
```

Помимо этих заголовков на запрос попадают [заголовки окружения](#заголовки-окружения).

### Заголовки окружения

Блок `headers` в файле окружения отправляется с **каждым** запросом прогона:

```yaml
# environments/ci.yaml
name: ci
baseUrl: https://jsonplaceholder.typicode.com
headers:
  x-source: speq-examples-jsonplaceholder
  authorization: "Bearer {{token}}"
```

Значения раскрывают `{{...}}` по тем же правилам, что и заголовки шага, поэтому в них можно ссылаться на
переменные окружения.

`headers` самого шага перекрывает одноимённый заголовок окружения. Имена заголовков в HTTP
регистронезависимы, поэтому `X-Source` в шаге вытесняет `x-source` из окружения — на провод уходит одно
значение, а не два.

Значения должны быть строками: нестроковое значение — ошибка разбора файла окружения, а не молчаливое
игнорирование.

### `body` и `bodyFromFixture`

Одновременно указывать оба нельзя:

```text
step cannot have both 'body' and 'bodyFromFixture' in <файл> step[N]
```

Тело собирается в три приёма, именно в этом порядке:

1. фикстура материализуется (если использован `bodyFromFixture`);
2. раскрываются блоки `gen`;
3. подставляются `{{переменные}}` во всех строках.

---

## Шаг `type: use`

Вызов переиспользуемого кода. Два разных механизма, различаются полем:

```yaml
- type: use
  name: "Прогрев"
  action: "jp.getPostById"      # действие модуля
  properties:
    postId: 1
  as: post                      # привязать returns под этим именем

- type: use
  name: "Общие шаги"
  ref: "../_shared/login.yaml"  # файл с переиспользуемыми шагами
```

Нужен ровно один из `ref` или `action` — но хотя бы один:

```text
step action or ref is required for use step in <файл> step[N]
```

| Поле | Назначение |
| --- | --- |
| `action` | `<алиас>.<действие>`; алиас — из `imports` |
| `ref` | Путь к файлу с шагами, **относительно каталога самого теста** (не `modulesDir`) |
| `properties` | Параметры вызова действия; строковые значения проходят подстановку переменных |
| `as` | Имя, под которым привязываются `returns` действия |

`as` должен быть идентификатором — только латиница, цифры и `_`:

```text
'as' must be a valid identifier (alphanumeric and underscore only) in <файл> step[N]
```

Файл, подключаемый через `ref`, содержит список шагов и не может сам содержать `use`:

```yaml
# _shared/probe.yaml
steps:
  - type: api
    name: "health"
    method: GET
    url: "/health"
```

---

## Ассерты

Семь типов. Все они выполняются после получения ответа; шаг падает, если упал хотя бы один.

| Тип | На что смотрит | Поля |
| --- | --- | --- |
| `status` | HTTP-код | `expected` — число |
| `json` | значение по пути | `path`, `expected` — любой JSON |
| `exists` | наличие пути | `path` |
| `contains` | **сырое тело ответа как строка** | `expected` — строка |
| `notcontains` | то же, инвертировано | `expected` — строка |
| `regex` | **сырое тело ответа как строка** | `expected` — регулярное выражение |
| `schema` | тело как JSON против JSON Schema | `ref` или `inline` |

```yaml
assert:
  - type: status
    expected: 200
  - type: json
    path: "$.userId"
    expected: 1
  - type: exists
    path: "$.email"
  - type: contains
    expected: "sunt aut"
  - type: notcontains
    expected: "error"
  - type: regex
    expected: "\"id\"\\s*:\\s*1"
  - type: schema
    ref: "jsonplaceholder/post.schema.json"
```

Существенные подробности:

- **`contains`, `notcontains` и `regex` работают с телом как с текстом**, а не с JSON. Они не принимают
  `path` и ищут по всему ответу. Если `expected` не строка, подстрока считается пустой — и `contains`
  тогда проходит всегда.
- **`json` сравнивает строго.** Значение по пути должно быть равно `expected` как JSON-значение:
  `1` и `"1"` не равны.
- Если `path` не указан у `json` или `exists`, он равен `$`, то есть всему телу.
- Если тело не разбирается как JSON, `json`, `exists` и `schema` падают с сообщением
  `response is not json`.
- `schema` с `ref` ищет файл в `schemasDir`; расширение можно опустить — пробуются `<ref>`,
  `<ref>.json`, `<ref>.yaml`, `<ref>.yml`. С `inline` схема пишется прямо в шаге. Нужен ровно один из двух:

  ```text
  schema assertion requires 'ref' or 'inline' in <файл> assert[N]
  ```

> **`value` — синоним `expected`.** Как `markers` для `tags`. Писать оба в одном ассерте нельзя, это
> ошибка разбора:
>
> ```text
> YAML parse error in <файл>: steps[0].assert[0]: duplicate field `expected` at line N column M
> ```
>
> Строка и колонка указывают на `value:`. Канонический вариант — `expected`.

> **В `expected` подстановка выполняется** — с той же областью видимости, что у `url` и `body` шага,
> включая привязки ответов предыдущих шагов по `id`. Это то, что делает выразимой связку
> *создать → прочитать → сравнить*.
>
> **Тип сохраняется.** Если строка состоит ровно из одного плейсхолдера, подставляется само значение, а не
> его текст: `expected: "{{created.response.body.id}}"` сравнится с числом `101`, а не со строкой `"101"`.
> Плейсхолдер внутри текста остаётся текстом: `"post-{{id}}"` → `"post-101"`.
>
> Вложенные структуры в `expected` раскрываются поэлементно. Плейсхолдер, не привязанный ни к одной
> переменной, — это ошибка `unresolved_template`, а не сравнение с литеральным `{{…}}`. Сообщение упавшего
> ассерта показывает и разрешённое значение, и шаблон, из которого оно получено.
>
> `schema` не затрагивается: `ref` и `inline` — это путь и схема, а не сравниваемое значение.

```yaml
steps:
  - type: api
    id: created
    name: "создать"
    method: POST
    url: "/posts"
    body: { title: "{{newTitle}}", userId: 1 }
    assert:
      - type: json
        path: "$.title"
        expected: "{{newTitle}}"

  - type: api
    name: "прочитать и сравнить с захваченным"
    method: GET
    url: "/posts/{{created.response.body.userId}}"
    assert:
      - type: json
        path: "$.userId"
        expected: "{{created.response.body.userId}}"   # число, не "1"
```

---

## Пути к полям ответа

`path` в ассертах `json` и `exists` и в `condition.path` разбирается функцией `json_path_get`. Несмотря
на название `jsonpath`, поддерживается только очень узкое подмножество:

| Выражение | Работает |
| --- | --- |
| `$` | да — всё тело |
| `$.id` | да |
| `$.user.address.city` | да — любая вложенность объектов |
| `$.items[0]` | **нет** |
| `$.0.id` | **нет** — к элементам массива обратиться нельзя |
| `$.items[*].id`, фильтры, срезы | **нет** |

Разбор устроен так: путь должен начинаться с `$.`, остаток делится по точкам, и каждый сегмент читается
как **ключ объекта**. Обращения к массивам нет ни в каком виде, поэтому проверить ответ, который сам
является массивом, можно только целиком (`$`) или через `contains`/`regex`.

---

## Условия и ожидание

`condition` превращает шаг в ожидание: запрос повторяется, пока условие не выполнится.

```yaml
- type: api
  name: "Ждём готовности задачи"
  method: GET
  url: "/todos/1"
  condition:
    type: jsonpath
    path: "$.completed"
    equals: false
    wait:
      timeoutMs: 5000
      intervalMs: 500
  assert:
    - type: status
      expected: 200
```

| Поле | Обязательно | Примечание |
| --- | --- | --- |
| `type` | да | Единственное значение — `jsonpath` |
| `path` | да | Непустой; те же ограничения, что выше |
| `equals` | да | Любой JSON, в том числе `null` |
| `wait.timeoutMs` | да, если есть `wait` | Должен быть `>= intervalMs` |
| `wait.intervalMs` | да, если есть `wait` | Пауза между попытками |

- **Без блока `wait` условие проверяется один раз.** Не выполнилось — шаг падает с
  `condition not met: ...`.
- С `wait` шаг повторяет запрос каждые `intervalMs`, пока не истечёт `timeoutMs`; тогда —
  `wait_timeout: condition (...) not met within Nms`.
- Ассерты выполняются **только после того, как условие выполнилось**. При таймауте они не запускаются.
- **Сравнение строковое.** И фактическое значение, и `equals` приводятся к строке, поэтому `equals: true`
  и `equals: "true"` ведут себя одинаково.

Нарушение соотношения таймаутов ловится ещё до запуска:

```text
condition.wait.timeoutMs must be >= intervalMs in <файл> step[N]
```

---

## Переменные и подстановка

Синтаксис — `{{имя}}`. Пробелы внутри скобок допускаются: `{{ имя }}`.

### Источники и приоритет

Позже — важнее:

1. **Окружение.** Каждый ключ файла окружения, кроме `baseUrl` и `headers`, становится переменной с тем
   же именем. Сам `baseUrl` доступен как `{{baseUrl}}`. `headers` переменной не становится — это
   заголовки запроса, см. [Заголовки окружения](#заголовки-окружения).
2. **Сьюта.** `suite.variables` из всех `init.yaml` по пути от корня к тесту; ближний перекрывает дальний.
3. **Модуль.** `variables` подключённого модуля.
4. **Тест.** `variables` самой спеки; здесь же раскрываются блоки `gen`.
5. **Вызов действия.** `properties` шага `use` — только внутри этого действия.
6. **`returns` действия** — привязываются как `<as>.<поле>` и видны последующим шагам.
7. **Ответ шага.** Успешный шаг `api` с полем `id` привязывается как `<id>` и виден последующим шагам
   теста.

### Ответ шага в следующих шагах

Шаг `api` с полем `id` публикует свой ответ под этим именем — сразу после того, как шаг прошёл:

| Выражение | Значение |
| --- | --- |
| `{{<id>.response.status}}` | HTTP-статус, числом |
| `{{<id>.response.body.<путь>}}` | Значение в теле ответа; путь идёт через точку |
| `{{<id>.response.headers.<имя>}}` | Заголовок ответа |

```yaml
steps:
  - type: api
    name: "POST /posts"
    id: created
    method: POST
    url: "/posts"
    body: { title: "t", body: "b", userId: 1 }
    assert:
      - type: status
        expected: 201

  - type: api
    name: "GET /posts/{{created.response.body.id}}"
    method: GET
    url: "/posts/{{created.response.body.id}}"
    assert:
      - type: status
        expected: 200
```

Числовой сегмент пути индексирует массив: `{{created.response.body.tags.0}}`. Это работает только
в подстановках; `path` ассертов и выражения `returns` к элементам массива по-прежнему не обращаются.

Привязка появляется, **только если шаг прошёл**. Упавший шаг не публикует ничего, и следующий шаг,
который на него ссылается, падает с `unresolved_template` — а не отправляет запрос с мусором.

Область видимости — весь тест: привязка, сделанная в `beforeEach`, видна в `steps` и в `afterEach`.
Тело действия модуля выполняется в своём наборе переменных, поэтому `id` внутри действия наружу не
виден — значения оттуда отдаются только через `returns`.

### Где подстановка выполняется

| Место | Подставляется |
| --- | --- |
| `url` шага | да |
| Значения `headers` | да |
| Строки внутри `body` (на любой глубине, включая массивы) | да |
| Строковые значения `properties` шага `use` | да |
| `assert.expected`, `condition.equals`, `condition.path` | **нет** |
| Имена ключей в `body` и `headers` | **нет** |

### Неизвестное имя — ошибка шага

```yaml
url: "/posts/{{нетТакойПеременной}}"
```

Подстановка, которая никуда не привязана, остаётся в строке как есть — и шаг падает до отправки запроса:

```text
unresolved_template: '{{нетТакойПеременной}}' in url is not bound to any variable
```

Проверяются `url` (уже вместе с `baseUrl`), значения `headers` и тело запроса; в сообщении указывается
первая найденная подстановка и место. Запрос не выполняется, поэтому опечатка в имени переменной больше
не превращается в загадочный `404`.

То же правило действует в `expected` — с той разницей, что ответ к этому моменту уже получен, поэтому
падает не шаг целиком, а сам ассерт:

```text
unresolved_template: '{{нетТакойПеременной}}' in assert[0] of this step is not bound to any variable
```

### Нестроковые значения

Значение подставляется в строку так: строки — как есть, всё остальное — через JSON-представление.
`{{postId}}` при `postId: 1` даст `1`, при `postId: {a: 1}` — `{"a":1}`.

`expected` в ассертах — исключение, и намеренное. Тело запроса всегда уезжает на провод текстом, а ассерт
сравнивается с уже разобранным JSON, поэтому строка, состоящая ровно из одного плейсхолдера, подставляется
значением, а не его текстом: `expected: "{{postId}}"` при `postId: 1` — это число `1`. Без этого
`json`-ассерт, сравнивающий строго, не совпал бы никогда.

---

## Генерация данных

Объект `{ gen: { type: ... } }` заменяется сгенерированным значением.

```yaml
variables:
  email: { gen: { type: email } }
steps:
  - type: api
    name: "POST /users"
    method: POST
    url: "/users"
    body:
      id: { gen: { type: uuid } }
      name: { gen: { type: name } }
      age: { gen: { type: int, min: 18, max: 60 } }
```

| `type` | Параметры | Результат |
| --- | --- | --- |
| `uuid` | — | UUID v4 строкой |
| `date-time` | `format`: `rfc3339` (по умолчанию) или `iso8601` | Текущее время |
| `string` | `minLength` (по умолчанию `1`), `maxLength` (по умолчанию `32`) | Латиница и цифры |
| `int` | `min`, `max` | Целое в диапазоне |
| `bool` | — | `true` или `false` |
| `email` | — | Безопасный адрес вида `...@example.com` |
| `name` | — | Имя человека |

> **Тип называется `date-time`, а не `datetime`.** Перечисление `GeneratorConfig` объявлено с
> `rename_all = "kebab-case"`, поэтому написание без дефиса не распознаётся и даёт
> `invalid gen config`.

Значения по умолчанию для `int` — половина диапазона `i64` в каждую сторону, то есть без `min`/`max`
получится очень большое число. На практике их стоит задавать всегда.

### Где `gen` раскрывается

Только в трёх местах: `variables` теста, `body` шага и `fixture.build`. В `url`, `headers`, `properties`
и в `variables` сьюты или модуля объект `gen` останется обычным объектом.

### Блок должен быть единственным ключом

Генератором считается объект, у которого `gen` — **единственный** ключ:

```yaml
title: { gen: { type: string } }              # генератор
title: { gen: { type: string }, note: "x" }   # обычный объект, ничего не подставится
```

### Ошибки валидации

Проверяются до выполнения запросов:

```text
generation_error: unsupported date-time format 'unix' at 'at'; supported: rfc3339, iso8601
generation_error: minLength (10) > maxLength (5) at 'title'
generation_error: min (10) > max (5) at 'age'
generation_error: invalid gen config at 'x' in <файл>: ...
```

---

## Сьюты и `init.yaml`

Сьюта — это каталог внутри `suitesDir`. Её настройки лежат в `init.yaml` (или `init.yml`) этого каталога.

```yaml
suite:
  variables:
    healthPath: "/todos/1"
  imports:
    - module: jsonplaceholder
      alias: jp
  beforeAll:
    - type: use
      name: "Прогрев"
      action: "jp.getPostById"
      properties:
        postId: 1
  beforeEach:
    - type: api
      name: "Проба"
      method: GET
      url: "{{healthPath}}"
      assert:
        - type: status
          expected: 200
  afterEach: []
  afterAll: []
```

Все ключи необязательны; пустой файл допустим и ничего не добавляет. Шаги в хуках — те же самые шаги
`api` и `use`.

К тесту применяется **вся цепочка** `init.yaml` от `suitesDir` вниз до каталога теста. Правила наследования
описаны в разделе [Порядок выполнения](#порядок-выполнения); коротко: `variables` и `imports` накапливаются,
`beforeAll`/`afterAll` берутся только из ближайшего файла.

---

## Модули

Модуль — файл в `modulesDir` с набором именованных действий. Верхний уровень — сразу `actions`
и `variables`, **без обёртки `module:`**.

```yaml
# modules/jsonplaceholder.yaml
variables:
  defaultLimit: 10

actions:
  getPostById:
    properties:
      - postId
    steps:
      - type: api
        id: get_post
        name: "GET /posts/{{postId}}"
        method: GET
        url: "/posts/{{postId}}"
        assert:
          - type: status
            expected: 200
    returns:
      postId: "$steps.get_post.response.body.id"
      userId: "$steps.get_post.response.body.userId"
```

| Поле действия | Назначение |
| --- | --- |
| `properties` | Имена параметров, которые обязан передать вызывающий |
| `steps` | Тело действия; вложенный `use` не поддерживается |
| `returns` | Значения, возвращаемые вызывающему |

Действие можно записать сокращённо — просто списком шагов. Тогда у него нет ни параметров, ни `returns`:

```yaml
actions:
  warmup:
    - type: api
      name: "GET /"
      method: GET
      url: "/"
```

### `returns`

Формат выражения строгий:

```text
$steps.<id шага>.response.body.<путь>
```

`<id шага>` должен совпадать с `id` одного из шагов действия — это проверяется `speq validate`.
`<путь>` разбирается тем же `json_path_get`, то есть только по ключам объектов.

### Как получить результат

```yaml
imports:
  - module: jsonplaceholder
    alias: jp
steps:
  - type: use
    name: "Получить пост"
    action: "jp.getPostById"
    properties:
      postId: 1
    as: post

  - type: api
    name: "Получить автора"
    method: GET
    url: "/users/{{post.userId}}"
    assert:
      - type: status
        expected: 200
```

Каждое поле `returns` привязывается под именем `<as>.<поле>` — здесь `post.postId` и `post.userId`.
Без `as` возвращаемые значения просто теряются.

Ограничения и ошибки:

- Привязка происходит, **только если все шаги действия прошли**.
- Имя из `as` не должно быть занято: `module_output_conflict: '<имя>' is already bound in context`.
- Параметр из `properties` считается переданным, если переменная с таким именем уже есть в области
  видимости — явная передача необязательна. Иначе:
  `use action '<действие>' requires property '<имя>' in step '<шаг>'`.
- Неизвестный алиас: `use action '...' references unknown alias '...'; available imports: [...]`.

---

## Фикстуры

Фикстура — заготовка тела запроса в `fixturesDir`.

```yaml
# fixtures/post-create.yaml
fixture:
  schemaRef: "jsonplaceholder/post.schema.json"    # необязательно
  build:
    title: { gen: { type: string, minLength: 10, maxLength: 50 } }
    body:  { gen: { type: string, minLength: 20, maxLength: 200 } }
    userId: { gen: { type: int, min: 1, max: 10 } }
```

Использование:

```yaml
- type: api
  name: "POST /posts"
  method: POST
  url: "/posts"
  bodyFromFixture:
    ref: "post-create.yaml"
    overrides:
      userId: 5
```

- `ref` — путь внутри `fixturesDir`, **с расширением**: подстановки расширения здесь нет, в отличие от
  `schema.ref` и имён модулей.
- `build` обязателен и должен раскрыться **в объект**, иначе
  `fixture_resolution_error: fixture.build did not resolve to an object`.
- `overrides` — слияние **только по верхнему уровню**: ключ целиком заменяется, вложенные объекты не
  сливаются рекурсивно. Значение из `overrides` побеждает, в том числе перекрывая `gen`.
- `schemaRef` проверяется `speq validate` на существование файла.

---

## ATDD: `status: pending`

`status: pending` помечает ещё не реализованное. Единственное допустимое значение — `pending`.

На уровне теста — тест не выполняется вовсе и попадает в отчёт со статусом `pending`:

```yaml
id: "users.create"
title: "POST /users создаёт пользователя"
status: pending
steps: [...]
```

На уровне шага — пропускается шаг, остальные выполняются:

```yaml
- type: api
  name: "PUT /posts/1"
  status: pending
  method: PUT
  url: "/posts/1"
```

Каскад: шаг, который читает привязку пропущенного шага, тоже становится `pending` — без явного
`status`. Отслеживаются оба имени, которые вообще что-то привязывают: `id` шага `api` и `as` шага `use`.

```yaml
- type: api
  name: "PUT /posts/1"
  id: updated
  status: pending
  method: PUT
  url: "/posts/1"

- type: api
  name: "DELETE /posts/{{updated.response.body.id}}"   # pending по каскаду
  method: DELETE
  url: "/posts/{{updated.response.body.id}}"
```

`pending` не приводит к ненулевому коду выхода: прогон из одних только `pending`-тестов завершится
успешно. В `summary.json` они попадают в `totals.pending`, в Allure — как `skipped`.

Подробнее — в [features/atdd-flow.md](features/atdd-flow.md).

---

## Чего DSL не умеет

Список ограничений, о которые чаще всего спотыкаются. Все проверены на текущем коде.

**`id` внутри действия модуля наружу не виден.** Значение из действия отдаётся только через `returns`
и `as`.

**Каскад `pending` не переходит между группами шагов.** `beforeEach`, `steps` и `afterEach`
просматриваются на pending-зависимости по отдельности.

**К элементам массива обратиться нельзя** — ни в `path` ассертов, ни в `condition.path`, ни в выражениях
`returns`. В подстановках `{{...}}` — можно, числовым сегментом пути.

**Вложенный `use` не поддерживается** — ни в файле переиспользуемых шагов, ни внутри действия модуля.

**`speq validate` проверяет не всё.** Шаги внутри действий модуля и `ref` у ассерта `type: schema` не
проверяются ([speq-tms/speq-docs#23](https://github.com/speq-tms/speq-docs/issues/23)).

**Параллельного выполнения нет** — тесты идут последовательно
(см. [features/parallel-execution.md](features/parallel-execution.md), спецификация опережает код).

---

## Дальше

- [quickstart.md](quickstart.md) — первый зелёный прогон с чистого каталога.
- [project-layout.md](project-layout.md) — раскладка проекта и справочник манифеста.
- [README.md](README.md) — команды, флаги, коды выхода.
- [features/](features/) — спецификации отдельных фич.
