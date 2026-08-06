---
title: Разведка прод-БД — legacy-ключи в metadata ингредиентов
type: note
permalink: tacticum/00-board/explore-prod-legacy-metadata-2026-08-06
status: draft
---

# Мины того же класса, что `assets`: карта по всей прод-БД

Продолжение находки по #750. Вопрос: `assets` был один такой, или в БД лежат другие ключи, которых
модель с `extra="forbid"` не знает.

**Короткий ответ: не один. Таких строк 804 — в 800 раз больше, чем `assets`.** Они не падают
сегодня, но держатся не на гарантии, а на совпадении трёх условий.

Источник: `tacticum_prod` (159.194.224.59), БД `tacticum_catalog`, только `SELECT` и
`docker inspect`. Сверка со схемой и моделями — в отдельном worktree `scan-750` на `origin/main`
78a7031a (создан и снят мной; ветку `fix/750-skill-assets-schema` не трогал).

## Что вообще лежит в проде

| | Число |
|---|---|
| версий профилей | 503 (500 `active`, 3 `retracted`) |
| ингредиентов | 9347 (9335 в active, 12 в retracted) |
| ингредиентов без версии профиля | 0 |
| `metadata` = NULL / `{}` / не-объект | **0 / 0 / 0** — все metadata непустые объекты |
| kind-ов в наличии | **7 из 9**: `hook_spec` и `permission_policy` не встречаются ни разу |
| уникальных «форм» metadata (kind + ключи + типы) | **22**, покрывают все 9347 строк |

## 1. Ключи в БД по kind — и чего модель не знает

Свернул все 9347 строк в 22 формы и прогнал **каждую** через ту же дискриминированную union,
которой пользуется `renderer._orm_row_to_pydantic`. Значения брал реальные, из базы (заменены
только тела `codex_body`/`copilot_body` — они по 15 КБ и на валидацию не влияют, это extra-ключи).

**Ключ есть в БД, но модель его не знает (`extra="forbid"` ⇒ мина):**

| kind | ключ | строк | есть в схеме? |
|---|---|---|---|
| `agent_spec` | `codex_body` | 773 | нет |
| `agent_spec` | `codex_target_path` | 773 | нет |
| `agent_spec` | `copilot_body` | 576 | нет |
| `agent_spec` | `copilot_target_path` | 576 | нет |
| `skill_spec` | `codex_body` | 31 | нет |
| `skill_spec` | `codex_target_path` | 31 | нет |

Уникальных строк: **773 `agent_spec` + 31 `skill_spec` = 804**. Ни один из этих ключей не
объявлен ни в модели, ни в схеме авторинга — они попали в базу мимо обоих контрактов.

Результат прогона форм через модель:

```
ИТОГО строк: 9347, модель принимает: 8543, отвергает: 804
```

Все остальные ключи (`description`, `model`, `tools`, `name`, `marker_id`, `merge_strategy`,
`target_file`, `allowed_tools`, `args`, `auth_type`, `command`, `env_required`, `transport`,
`url`, `trigger`, `description_trigger`) модель знает.

**Обратный класс — поле объявлено, но в БД не встречается** (не мина, для полноты):
`agent_spec`: `delegation`, `permissions_ref` · `command_spec`: `args_schema` · `hook_spec`: все 5 ·
`mcp_server_spec`: `required_scopes` · `permission_policy`: все 6 · `rule_set`: `scope_globs` ·
`skill_spec`: `scripts`.

**Обязательные поля**: ни одной строки без обязательного поля своей модели. Отдельно проверил
`agent_spec` без `model` — 3 строки (`iva-frontend-base` 0.1.0), но `model` в модели необязателен,
обязателен только `description`, и он есть у всех 856.

## 2. Зеркало схема ↔ модель — расхождений 0

По всем 9 kind набор полей `metadata` в схеме и в pydantic-модели совпадает **точно**
(на `origin/main`). Второй класс расхождений из постановки в репозитории отсутствует.

Но `additionalProperties` в схеме **не выставлен ни у одного из 9 kind** — то есть схема
авторинга сегодня пропустит любой незаявленный ключ. Именно так `codex_body` и попал в базу:
запретить его было нечем. (Правка #750 закрывает это у `skill_spec`; у остальных 8 остаётся.)

## 3. `scripts` — подтверждаю 0, включая архив

| статус версии | версий | ингредиентов | со `scripts` | с `assets` |
|---|---|---|---|---|
| `active` | 500 | 9335 | **0** | 1 |
| `retracted` | 3 | 12 | **0** | 0 |

**`metadata.scripts` в проде нет ни одной строки — ни в активных версиях, ни в отозванных.**
Поле мёртвое на 100%: объявлений в каталоге 0 (проверено по манифестам в прошлой задаче) и записей
в базе 0. Для задачи про `scripts` это и есть честный факт: снятие поля ничего в проде не заденет —
в отличие от `assets`, где строка нашлась.

## 4. Почему 804 мины ещё не рванули

Прогнал те же 22 формы через **реальные рендереры install-пути** — с фильтром `supports` и
passthrough, как в бою:

```
строк, падающих хотя бы на одном достижимом CLI: 0
```

Спасают три вещи, и ни одна из них не является гарантией:

1. **`_cli_body_passthrough`** перехватывает строку с `codex_body`+`codex_target_path` ДО pydantic
   и делает `continue`. Докстринг это признаёт прямо: «The body bypasses the Pydantic
   discriminated-union validation path entirely, so the extra metadata keys never reach the strict
   `AgentSpecMetadata` model». То есть лишние ключи в базе — не случайность, на них построена
   доставка per-CLI тел.
2. **Фильтр `supports`** отсекает 197 `agent_spec` и 31 `skill_spec` (у них `copilot_body` нет, и
   `copilot` не в `supports`) раньше, чем они дойдут до модели.
3. **На install-пути достижимы ровно 3 CLI** — `claude-code`, `codex`, `copilot`
   (`_RENDERERS` в `pull_installation_content.py` и `tacticum_init.py`). `opencode` и `gemini`
   рендереры имеют, но на install-путь не выведены; строк, где лишние ключи сочетались бы с
   `opencode`/`gemini` в `supports`, — **0**.

**Насколько это близко к срыву** (прогнал каждый сценарий):

| сценарий | результат |
|---|---|
| как сейчас: 197 `agent_spec` под copilot | OK, действий=0 — спасает **фильтр supports**, не модель |
| **добавить `copilot` в `supports`** этим строкам | **ПАДАЕТ** `ValidationError: codex_body, codex_target_path` |
| **потерять парный `codex_target_path`** (passthrough требует ОБА непустыми) | **ПАДАЕТ** `ValidationError: codex_body` |
| убрать/обойти passthrough | падают все 804 |

То есть одна правка `supports` в манифесте — и 197 строк ложатся. Пустых значений в парах сейчас
нет: у всех 773 `agent_spec` и 31 `skill_spec` и `codex_body`, и `codex_target_path` непустые
(проверено отдельным SELECT).

**Расхождение нормы с данными:** докстринг passthrough утверждает «Only agent ingredients carry a
per-CLI body (skills/mcp/commands share one body across CLIs)». В базе **31 `skill_spec` несёт
`codex_body`** — 3 профиля (`iva-qa-autotest-base`, `iva-qa-delivery-base`,
`tacticum-autotest-core`), 11 активных версий. Код kind не проверяет, поэтому работает; но
комментарий описывает не то, что есть.

## 5. Установки: эти мины, в отличие от `assets`, не бесхозные

Строки с `codex_body`/`copilot_body`/`assets` несут **27 профилей**. У 8 из них есть установки:

| профиль | версий | строк-мин | установок | активных |
|---|---|---|---|---|
| `iva-kmp-brownfield` | 36 | 144 | 2 | 2 |
| `iva-rn-brownfield` | 25 | 100 | 2 | 1 |
| `iva-brownfield-mail` | 23 | 92 | 2 | 2 |
| `iva-web-brownfield` | 19 | 76 | 1 | 1 |
| `iva-go-backend-brownfield` | 15 | 60 | 1 | 1 |
| `iva-ios-brownfield` | 6 | 24 | 1 | 1 |
| `firebird-web-brownfield` | 6 | 24 | 1 | 1 |
| `iva-system-analyst` | 4 | 4 | 1 | 1 |
| остальные 19 профилей | — | 380 | **0** | 0 |

Итого 11 установок, 10 активных — и они живые: `iva-kmp-brownfield` 181 синхронизация, последняя
**2026-08-06**; `iva-web-brownfield` 63, сегодня; `iva-ios-brownfield` 17, сегодня. Это прямой
контраст с `assets`, где установок было 0.

**Чем реально пользуются** (`telemetry_events.target_cli` за всё время):

| target_cli | событий | период |
|---|---|---|
| (пусто) | 39337 | 09.06 – 06.08 |
| `iva-read` | 37349 | 10.07 – 06.08 |
| **`codex`** | **125** | 13.07 – **06.08** |
| **`copilot`** | **1** | **28.05 – 28.05** |

По затронутым профилям: `codex` — 15 событий у `iva-kmp-brownfield`, 2 у `iva-web-brownfield`;
`copilot` — 1 событие у `iva-brownfield-mail` за 28.05. **codex-путь живой и используется сегодня,
copilot-путь практически мёртв.** Это снижает вероятность сценария «добавили copilot в supports»,
но не закрывает его.

## 6. Побочная находка: `{ingredient_id}` в пути passthrough не подставляется

Passthrough отдаёт путь **как есть**: `return {"action": "write_file", "path": path, ...}` — без
`.format(ingredient_id=...)`, которое есть в обычной ветке `_WRITE_FILE_KINDS` (там его добавили
по #658 ровно от этой ошибки). Прогнал реальную строку:

```
path = .github/agents/{ingredient_id}.agent.md
```

Сколько строк несут шаблон в пути:

| kind | `codex_target_path` с `{ingredient_id}` | `copilot_target_path` с `{ingredient_id}` |
|---|---|---|
| `agent_spec` | 0 из 773 | **576 из 576 — все** |
| `skill_spec` | **8 из 31** | — |

То есть **584 строки** уедут клиенту с фигурными скобками в пути. Подставляет ли их клиент
(`tacticum_cli`) — я проверить не могу, это client-side; со стороны сервера действие уходит
буквальным. Если не подставляет, то у copilot-установок файлы агентов ложатся в
`.github/agents/{ingredient_id}.agent.md` вместо имени. Учитывая, что copilot-путь почти не
используется (1 событие), это может быть не замечено до сих пор именно поэтому. **Вне скоупа
задачи — выношу как отдельный вопрос, не как вывод.**

---

```
ВЕРДИКТ: частично

Проверено: все 9347 ингредиентов прод-БД tacticum_catalog, свёрнутые в 22 уникальные формы
  metadata (kind + ключи + типы значений); каждая форма прогнана через дискриминированную union
  Ingredient и через все 3 достижимых на install-пути рендерера с реальными supports; сверка
  ключей БД с 9 pydantic-моделями и с ingredient.v1.schema.json на origin/main; обязательные поля;
  зеркало схема↔модель; scripts и assets по всем статусам версий; установки и телеметрия
  затронутых профилей; хрупкость passthrough в 4 сценариях.

Данные: 503 версии (500 active, 3 retracted), 9347 ингредиентов, 7 kind из 9 (hook_spec и
  permission_policy — 0 строк), 22 формы metadata, metadata NULL/пустых/не-объектов 0 ·
  модель отвергает 804 строки из 9347 (773 agent_spec + 31 skill_spec) по ключам codex_body,
  codex_target_path, copilot_body, copilot_target_path — ни один не объявлен ни в модели, ни в
  схеме · через реальные рендереры install-пути падений 0 · зеркало схема↔модель: расхождений 0
  по всем 9 kind, но additionalProperties не выставлен ни у одного из 9 · scripts: 0 строк в
  active и 0 в retracted · assets: 1 в active, 0 в retracted · профилей с минами 27, с
  установками 8, установок 11 (10 активных, синхронизации сегодня 06.08) · телеметрия: codex 125
  событий по 06.08, copilot 1 событие за 28.05 · шаблон {ingredient_id} в пути passthrough:
  576 из 576 copilot_target_path и 8 из 31 codex_target_path, не форматируется.

Подтверждение: SELECT по kind × jsonb_object_keys → 27 пар kind/ключ · свёртка в формы через
  string_agg(key||':'||jsonb_typeof) → 22 формы, сумма rows = 9347 · прогон форм скриптом →
  «ИТОГО строк: 9347, модель принимает: 8543, отвергает: 804» и «строк, падающих хотя бы на одном
  достижимом CLI: 0» · сценарии хрупкости → «добавить copilot в supports: ПАДАЕТ ValidationError:
  codex_body, codex_target_path», «потерять codex_target_path: ПАДАЕТ ValidationError:
  codex_body» · шаблон → «path = .github/agents/{ingredient_id}.agent.md» · SELECT по
  profile_versions.status → active 500/9335/scripts 0/assets 1, retracted 3/12/0/0 · SELECT по
  installations и telemetry_events (таблицы выше). Дата прогона 2026-08-06.

НЕ проверено: значения полей проверены только там, где их проверяет модель (enum/Literal взяты
  реальные из базы) — семантическую осмысленность значений не оценивал, это не валидация.
  Тела codex_body/copilot_body при прогоне заменены на "X": на набор полей и типы это не влияет,
  но прогон с полными телами (по 15 КБ, 773 строки) я не делал. Клиентскую сторону
  (tacticum_cli) не проверял вовсе — подставляет ли клиент {ingredient_id} в пути из passthrough,
  остаётся открытым, и вывод про 584 строки поэтому сформулирован как вопрос, а не как дефект.
  Не гонял тесты и линтеры — задача read-only по проду, кода я не менял. Не проверял, достижим ли
  admin render-preview по строкам ИЗ БД (он валидирует ингредиент из тела запроса), поэтому
  opencode/gemini рассматривал только как недостижимые на install-пути. Не смотрел, откуда
  codex_body попал в базу (какой сид/скрипт его пишет) — это вопрос авторства, а не состояния.
```
