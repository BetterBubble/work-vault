---
title: explore-ds-angular-parity-2026-07-28
type: note
permalink: tacticum/00-board/explore-ds-angular-parity-2026-07-28-1
status: current
created: 2026-07-28 11:41
repo: tacticum-dev
tags:
- board
- design-system
- angular
updated: 2026-07-28 11:41
archived-at: 2026-08-05 15:19
---

# Разведка: что есть под Angular против KMP (ДС-паритет)

Read-only, ничего не менял. Репозиторий `/Users/bubblemac/tacticum/tacticum-dev`, ветка `main`,
`26f5301` (2026-07-27, merge PR #165). Serena активирована на `tacticum-dev`.
Даты — по `git log` последнего изменения файла/каталога.

---

## 0. Главный вывод в одну фразу

Под Angular **есть данные маппинга** (словарь `code-bindings` для веба богаче мобильного:
49 компонентов, 32 стабильных `figma_key`) — и **нет доработанных навыков**: вся работа
21–27.07 (числовая сверка с Figma, ревизия по смене значений токенов, adopt/promote/author
компонента, обновление существующего экрана по новому фрейму) сделана **только в KMP-лейне**.
Больше того — **роль `iva-role-web` получает старые общие веб-навыки из `tacticum-ui-base`
(15.07)**, а не улучшенные веб-версии из `iva-web-brownfield` (24–25.07): те живут только в
депрекируемом профиле и в роль не приезжают.

---

## 1. Что есть под Angular — инвентаризация

### 1.1 Профили и лейны

| Профиль/лейн | Версия | Последнее изменение | Роль |
|---|---|---|---|
| `templates/iva-role-web` | 0.1.1 | 2026-07-24 | роль-пресет, implement-only; содержимое — из лейнов |
| `templates/iva-web-development-base` | 0.1.2 | 2026-07-25 | стек-лейн Angular/Nx, 14 скиллов + 3 агента |
| `templates/iva-web-brownfield` | 0.5.1 | 2026-07-25 | СТАРЫЙ профиль (замещается ролью), 22 скилла |
| `templates/tacticum-ui-base` | 0.1.0 | **2026-07-15** | общий UI-лейн: 4 ДС-навыка + playwright MCP |

Для сравнения — KMP: `iva-role-kmp` **0.2.1** (27.07), `iva-kmp-development-base` **0.12.0**
(27.07, было 0.7.0 на 25.07), `iva-kmp-brownfield` 0.5.4 (27.07).

**Композиция `iva-role-web`** (`depends_on`, порядок = приоритет, later base wins):
`tacticum-core-base` → `tacticum-development-core` → `iva-web-development-base` →
`tacticum-bugfix-base` → `tacticum-lite-base` → `tacticum-research-base` → **`tacticum-ui-base`**.
`tacticum-ui-base` стоит **последним** ⇒ его версии ДС-навыков побеждают. У `iva-role-kmp`
он стоит **третьим**, ПЕРЕД стек-лейном — сознательно, чтобы Compose-версии перекрыли веб-версии
(комментарий в манифесте роли, строки 12–18).

### 1.2 Навыки про дизайн в веб-контуре

| Навык | Где лежит | Байт / строк | Изменён | Приезжает в `iva-role-web`? |
|---|---|---|---|---|
| `ui-mockup-match` | `tacticum-ui-base` | 6 910 / 149 | 15.07 | **ДА** |
| `ui-mockup-match` | `iva-web-brownfield` | 16 255 / 296 | 24.07 | нет |
| `design-token-usage` | `tacticum-ui-base` | 4 988 / 85 | 15.07 | **ДА** |
| `design-token-usage` | `iva-web-brownfield` | 5 509 / 93 | 15.07 | нет |
| `design-system-discovery` | `tacticum-ui-base` | 4 491 / 82 | 15.07 | **ДА** |
| `design-system-discovery` | `iva-web-brownfield` | 5 256 / 95 | 24.07 (ось-1, surface-router) | нет |
| `pin-ui-pipeline-check` | `tacticum-ui-base` | — | 15.07 | **ДА** |
| `angular-ds-component-usage` | `iva-web-development-base` + brownfield | 10 012 / 178 | 25.07 | да (по манифесту 0.1.1) |
| `angular-ds-component-authoring` | `iva-web-development-base` + brownfield | 9 667 / 195 | 25.07 | да (по манифесту 0.1.2) |
| `iva-core-design-system` | `iva-web-development-base` + brownfield | 4 357 / 83 | 24.07 | да (по манифесту 0.1.2) |

Для сравнения KMP-версии (все в `iva-kmp-development-base`, все 27.07):
`ui-mockup-match` 23 110 / 386 · `design-token-usage` 16 714 / 263 · `ds-component-adoption`
20 762 / 313 · `web-to-kmp-screen-port` 34 791 / 454 · `web-to-kmp-source-reference` 6 143 / 98 ·
`design-system-discovery` 4 535 / 82.

**Проверка по эталону установки (не по манифесту, а по тому, что реально доставляется):**
хэши в `apps/backend/tests/e2e_install/golden/iva-role-web/{claude-code,codex}.json` совпадают
с файлами **`templates/tacticum-ui-base/...`** для `ui-mockup-match`, `design-token-usage`,
`design-system-discovery`, `pin-ui-pipeline-check`. Для `iva-role-kmp` — с файлами
`iva-kmp-development-base`. Сверено побайтово (sha256 golden ↔ sha256 файлов в `templates/`).

### 1.3 Сценарии KMP → есть ли веб-аналог

| Сценарий (как он есть в KMP) | Веб-аналог | Где |
|---|---|---|
| Обновление **существующего** экрана по новому фрейму (`web-to-kmp-screen-port`, трек M: снимок текущего экрана → новый фрейм → диф → меняем только презентацию, поведение заморожено) | **НЕ НАЙДЕНО** | — |
| Ревизия экрана при **смене значений токенов** (`design-token-usage` §«When the token values change», 155 строк: baseline → план фетча → три вида мест → матч по ЗНАЧЕНИЮ → все режимы темы → отчёт «токенов не хватило») | **НЕ НАЙДЕНО** — в веб-версиях `design-token-usage` (85 и 93 строки) такого раздела нет вовсе | — |
| Замена самописного элемента на компонент ДС (`ds-component-adoption`, гейт «поиск в ДВУХ местах» → adopt / promote / author) | **частично**: `angular-ds-component-authoring` §«Migration (Scenario 3)» (17 строк) — миграция легаси `ui-kit`→`@iva/design-system` батчами; **гейта adopt/promote/author нет**, `angular-ds-component-usage` при незамапленном инстансе просто СТОПит | `iva-web-development-base` |
| Числовая сверка с Figma-фреймом (`ui-mockup-match` §«Figma numeric-compare mode», ΔE/dp/допуски) | **есть в `iva-web-brownfield`** (§«Figma numeric-compare mode (Scenario 2, step 7 — gap G5)», 24.07) — но **в роль `iva-role-web` не приезжает** | только старый профиль |
| Сборка НОВОГО экрана по фрейму из компонентов ДС | **есть и лучше вебовое**: `angular-ds-component-usage` (пред-проверка фрейма, Code Connect → словарь, стоп на незамапленном) | `iva-web-development-base` |
| Отдельная ДС конференц-поверхности | **есть только у веба**: `iva-core-design-system` (iva-core / VCSWEB) | `iva-web-development-base` |

---

## 2. Данные ДС

`design-systems/` — четыре системы, все с `design-system.yaml` + `tokens.json` (DTCG).

| ДС | Версия | Листовых токенов | `code-bindings` | С `figma_key` | Последнее изменение |
|---|---|---|---|---|---|
| `iva-web` | **0.3.0** | 1 021 | **49** | **32** (17 = `null`) | 2026-07-23 |
| `iva-mobile` | **0.2.0** | 966 | **34** | **0** (все `null`, pending) | 2026-07-23 |
| `iva-rn` | 0.1.0 | 120 | **нет блока** | — | 2026-06-18 |
| `tacticum-web` | 0.1.0 | 153 | **нет блока** | — | 2026-07-19 |

**ДС `iva-core` в репозитории НЕТ** — каталога `design-systems/iva-core/` не существует;
есть только навык `iva-core-design-system` (описывает поверхность словами, без словаря и
серверного seed). Это зафиксировано как отложенное в `12-Features/tz1-figma-ds-handoff-ostatok.md`
(пункт 2, владелец — server/RE + DS-команда).

### 2.1 Где живёт маппинг

`$extensions."dev.tacticum.code-bindings"` — **один блок в КОРНЕ** `tokens.json`, а не поле у
отдельных токенов. Ключи блока: `schema` (`tacticum-code-bindings/v0`), `description`,
`codebase`, `figma_file_key`, `usage`, `components` (список), плюс у веба `figma_file` и
`storybook_base`, у мобильного `stack: compose-mp` и `gallery`.

**Веб (`iva-web`) — привязка к Angular-исходникам, не к NgModule:**

```json
"codebase": {
  "repo": "https://git.hi-tech.org/iva/one/web/iva-one",
  "library": "@iva/design-system",
  "components_root": "libs/design-system/src/lib/components",
  "extracted_from_commit": "daf1424ad 2026-07-20"
},
"storybook_base": "https://iva-ui.ivcs.su/",
"figma_file_key": "AG11paSthGC7zSoovfjip0"
```

Запись компонента — дословно (первые три из 49):

```json
{ "name": "Chip", "match": ["chip"],
  "figma_key": "f0385470a591afc26b342916b89eb6c3c7ebf883",
  "selector": "iva-chip", "kind": "component",
  "source": "chips/chip/chip.component.ts",
  "storybook": "components-chips-chip",
  "inputs": { "appearance": ["cyan","red","green","gray","purple","orange","yellow",
    "lime","brown","pink","lavender","blue","neutral","warning","critical"],
    "size": ["s","m","l"], "closable": "boolean (default false)",
    "selected": "boolean", "disabled": "boolean" } }

{ "name": "Chip User", "match": ["chipuser","userchip"],
  "figma_key": "fe19a9302881c136bd43a7965aea9594b0843322",
  "selector": "iva-chip-user", "kind": "component",
  "source": "chips/chip-user/chip-user.component.ts",
  "storybook": "components-chips-chipuser",
  "inputs": { "appearance": ["neutral","warning","critical"],
    "closable": "boolean (default true)", "selected": "boolean", "disabled": "boolean" },
  "notes": "User chip with avatar slot; the go-to component for user lists as chips." }

{ "name": "Chip Button", "match": ["chipbutton"],
  "figma_key": "eda46a078192773a19cfebd871b2cc56943c9fce",
  "selector": "iva-chip-button", "kind": "component",
  "source": "chips/chip/chip-button/chip-button.component.ts",
  "storybook": null,
  "notes": "Internal close/action button of Chip; rarely used standalone." }
```

Единица привязки на вебе — **Angular selector + путь к файлу компонента** (`selector`,
`kind: "component"`, `source` относительно `components_root`) + страница Storybook + inline-список
`inputs`. Ни пакета npm-модуля, ни NgModule, ни CSS-класса в записях нет.

**Мобильный (`iva-mobile`) — привязка к Compose-composable:**

```json
"stack": "compose-mp",
"figma_file_key": null,
"codebase": {
  "repo": "https://git.hi-tech.org/iva-m/android/kmp/",
  "module": ":core:design-system",
  "package": "su.ivcs.messenger.designsystem",
  "components_root": "core/design-system/src/commonMain/kotlin/su/ivcs/messenger/designsystem",
  "extracted_from_commit": "8f65eea2ff 2026-07-23"
},
"gallery": "no Storybook; components carry @Preview functions next to the source"
```

```json
{ "name": "Avatar", "match": ["avatar","ava","userphoto","circlephoto"],
  "figma_key": null, "selector": "IvaAvatar", "kind": "composable",
  "source": "element/circle_photo/IvaAvatar.kt",
  "inputs": { "mode": "IvaAvatarMode", "sizeMode": "IvaAvatarSizeMode",
    "initials": "String", "linkPhoto": "String?", "userType": "UserAvatarType" },
  "notes": "Initials-only variant: IvaAvatarInitials (same folder)." }

{ "name": "Badge", "match": ["badge","counter"], "figma_key": null,
  "selector": "IvaBadge", "kind": "composable", "source": "element/badge/IvaBadge.kt",
  "notes": "Domain badges nearby: SystemBadge, RoomStatusBadge, CalendarEventStatusBadge." }

{ "name": "System Badge", "match": ["systembadge"], "figma_key": null,
  "selector": "SystemBadge", "kind": "composable", "source": "element/badge/SystemBadge.kt" }
```

Схема одна и та же (`tacticum-code-bindings/v0`), различаются `kind` (`component` ↔ `composable`),
наличие Storybook и заполненность `figma_key`. **По данным веб сильнее мобильного**: у веба есть
`figma_file_key` и 32 стабильных ключа, у мобильного нет ни файла, ни ключей — матч только по имени.

---

## 3. Чем проверяется «в бою»

### 3.1 Эталоны установки (`apps/backend/tests/e2e_install/golden/`)

Есть 18 каталогов. Для веба: **`iva-role-web`** (последнее изменение 23.07),
**`iva-web-brownfield`** (23.07), **`firebird-role-web`**, **`firebird-web-brownfield`**.
Для KMP: **`iva-role-kmp`** (27.07). Каждый — `claude-code.json` + `codex.json` с деревом
`relpath → sha256`.

**Эталон `iva-role-web` устарел.** В нём 34 (claude-code) / 33 (codex) записи, и в них **НЕТ**
`angular-ds-component-usage`, `angular-ds-component-authoring`, `iva-core-design-system` —
трёх скиллов, добавленных в `iva-web-development-base` 24–25.07 (версии лейна 0.1.1 и 0.1.2,
CHANGELOG подтверждает). Эталон с тех пор не перегенерировали (`git log` — 23.07). Тест
`test_install_flow_roles_generic` сверяет дерево с эталоном (`assert_tree_matches_golden`),
так что либо эталон надо регенерировать, либо скиллы не доезжают — прогнать я не могу
(нужен Docker/Postgres). Для сравнения эталон `iva-role-kmp` актуален: в нём есть
`AI common/skills/ds-component-adoption`, `web-to-kmp-screen-port`, `web-to-kmp-source-reference`.

### 3.2 Гейт подмены стека — только для KMP

`apps/backend/tests/catalog/test_role_replacement_parity.py`. Базовый тест
`test_role_covers_replaced_profile` сверяет **только идентификаторы** ингредиентов и для пары
`("iva-role-web","iva-web-brownfield")` проходит — id совпадают, хотя тела разные.
27.07 добавлен stack-fidelity-блок (`_STACK_MARKERS`, `_FOREIGN_MARKERS`,
`test_role_never_delivers_foreign_stack_body`) — и он заведён **только для `iva-role-kmp`**
(строки 244–262: словари содержат ровно один ключ `"iva-role-kmp"`). У веба тел никто не сверяет.

### 3.3 Зеркала (`templates/_mirrors.yaml`)

- Пара **`iva-kmp-development-base` ↔ `iva-kmp-brownfield`**: 18 ингредиентов, среди них
  **все четыре ДС-навыка** (`design-system-discovery`, `design-token-usage`,
  `pin-ui-pipeline-check`, `ui-mockup-match`) — CI держит их байт-в-байт.
- Пара **`iva-web-development-base` ↔ `iva-web-brownfield`**: 15 ингредиентов, и **ни одного
  ДС-навыка** (`angular-ds-*`, `iva-core-design-system`, `design-token-usage` и др. вне пары).
  Веб-копии расходятся свободно, CI-замка нет.

Фактический разброс копий (md5 тел):
`design-system-discovery` — 8 копий / 6 разных тел · `design-token-usage` — 6 копий / 4 тела ·
`ui-mockup-match` — 6 копий / 5 тел · `pin-ui-pipeline-check` — 6 копий / 4 тела.

### 3.4 Прогнать сценарий целиком

- **Локального чекаута `iva-one` нет.** В `~/tacticum` только `tacticum-dev` и его worktree.
  Единственный источник кода веб-ДС — repomix-снимок
  `~/tacticum/helm/data/real/git/repomix/iva-one.xml`, снят **2026-07-03**, `--compress`
  (у `.ts` сигнатуры без тел; `.md`/`.mdx`/`.scss` полные). 1 682 упоминания
  `figma-integration`; пути генераторов (`colors/color-tokens.mjs`, `icons/icon-components.mjs`,
  `icons/optimize-icons.mjs`, `shared/api.js`) в снимке видны.
- **Веб-пилота у нас не было.** Единственный прогон сценария — KMP:
  `00-Board/Пилот Сц.4 — ContactDetail (СУХОЙ прогон web-to-kmp-screen-port).md`, и он **сухой**
  (валидация навыка, ported-код остался черновиком).
- **Живая установка веба есть**: на проде `iva-web-brownfield` на репозитории iva-one,
  installation `7c5854f6-66e0-4dfe-a617-26ffc9fc946e`, пин **0.5.1** (снимок
  `docs/runbooks/prod-seed-2026-07-25-rollback.md`). То есть в поле веб сидит на **старом
  профиле**, не на роли. Выкатка 27.07 веб не трогала вообще
  (`prod-seed-2026-07-27-rollback.md`: только `iva-kmp-development-base` 0.7.0→0.12.0,
  `iva-kmp-brownfield` 0.5.2→0.5.4, `iva-role-kmp` 0.1.1→0.2.0).
- **Демо/песочницы под веб-ДС в репозитории нет.**

### 3.5 `docs/user_manuals/iva-web-figma-mapping-quickstart.md`

142 строки, последнее изменение **2026-07-24**. Описывает Сценарий 2 (сборка экрана по фрейму):
Шаг 0 включить MCP в Figma desktop → Шаг 1 обновить профиль → Шаг 2 подключить `figma-desktop`
MCP (`http://127.0.0.1:3845/mcp`) → **Шаг 3 вручную вставить блок правил между маркерами
`<!-- figma-mapping-pilot -->` в `AGENTS.md`** → Шаг 4 проверка `design_get_tokens` (ожидание «49»).

**Неактуален в двух местах:**
1. строки 22 и 43 требуют профиль **`iva-web-brownfield`** (депрекируемый) и его quickstart;
   роли `iva-role-web` в документе нет ни разу;
2. Шаг 3 — **ручная вставка правила**. У KMP этот шаг снят: `iva-role-kmp` 0.2.1 сама дописывает
   маршрутизатор ДС-сценариев в `AGENTS.md`/`CLAUDE.md`
   (`iva-kmp-figma-mapping-quickstart.md`, 110 строк, обновлён 27.07: «Шаг 3 — правило в
   репозитории приезжает САМО»). У `iva-role-web` в
   `ingredients/repo-configs/claude-code/CLAUDE.md.fragment` маршрутизатора нет — там одна
   строка: «UI-задачи: design-system-discovery → токены → ui-mockup-match (лейн ui)».
   У KMP-фрагмента на этом месте 5 строк маршрутизации по сценариям.

Также в веб-quickstart нет предупреждения про платный тариф/seat Figma Dev Mode — в KMP-версии
оно есть (добавлено 27.07).

---

## 4. Таблица паритета

| Сценарий | KMP | Angular |
|---|---|---|
| Данные маппинга Figma↔код | 34 компонента, **0** `figma_key`, `figma_file_key: null` | **49 компонентов, 32 `figma_key`**, файл-ключ есть, Storybook-ссылки |
| Единица привязки | composable `IvaAvatar` + `.kt` в `:core:design-system` | selector `iva-chip` + `.component.ts` в `libs/design-system` |
| Сборка НОВОГО экрана по фрейму | `compose-multiplatform-ui` + `ds-component-adoption` | **`angular-ds-component-usage`** (пред-проверка фрейма, Code Connect → словарь, стоп на незамапленном) — сильнее KMP |
| Обновление СУЩЕСТВУЮЩЕГО экрана по новому фрейму | **есть** — `web-to-kmp-screen-port` трек M (454 строки) | **нет** |
| Смена ЗНАЧЕНИЙ токенов, ревизия экрана | **есть** — `design-token-usage` §«token values change» (155 строк, матч по значению, все режимы темы) | **нет** (85/93 строки, раздела нет) |
| Замена самописного элемента на компонент ДС | **есть** — `ds-component-adoption`, гейт adopt/promote/author (313 строк) | **частично** — §Migration в `angular-ds-component-authoring` (17 строк), гейта нет |
| Числовая сверка с Figma (ΔE, dp, допуски) | **есть**, в роли | **есть в старом профиле, в роль НЕ приезжает** |
| Что реально приезжает в роль | Compose-версии из стек-лейна (перекрывают ui-base) | **общие версии `tacticum-ui-base` от 15.07** |
| Маршрутизатор ДС-сценариев в `AGENTS.md`/`CLAUDE.md` | **есть**, приезжает сам (роль 0.2.1) | **нет**, одна общая строка; правило вставляется руками (Шаг 3 quickstart) |
| Отдельная ДС конференц-поверхности (`iva-core`) | н/п | навык есть, **серверной ДС и словаря нет** |
| Гейт против подмены тел (stack-fidelity) | **есть** (27.07) | **нет** — сверяются только id |
| Зеркала под CI-замком | 4 ДС-навыка в паре зеркал | **0 ДС-навыков** в паре зеркал |
| Эталон установки | актуален (27.07), 39/38 записей | **устарел (23.07)** — нет 3 скиллов от 24–25.07 |
| Прогон на реальном репозитории | сухой пилот ContactDetail | **не было**; живая установка сидит на старом профиле 0.5.1 |
| Версия в проде | лейн 0.12.0, роль 0.2.0 (27.07) | лейн 0.1.2, роль 0.1.1, старый профиль 0.5.1 (25.07) |

### Что у Angular ЛУЧШЕ, чем у KMP

1. **Словарь.** 49 против 34, 32 стабильных `figma_key` против нуля, есть `figma_file_key`
   и `storybook_base`. У KMP матч только по имени/алиасам.
2. **Сборка нового экрана по фрейму.** `angular-ds-component-usage` содержит пред-проверку
   фрейма (auto layout + узнаваемые инстансы, иначе стоп к дизайнеру) и порядок резолва
   Code Connect → словарь. В KMP этого нет.
3. **Витрина.** Storybook `https://iva-ui.ivcs.su/` с ID страниц прямо в биндингах;
   у KMP — «no Storybook, смотри `@Preview` рядом с исходником».

---

## 5. Внешняя сторона

**На сервер `helm` я не ходил — доступа у меня нет, не проверял.** Всё ниже — из локальных
материалов.

- В vault есть готовая разведка: `00-Board/explore-ds-sheremetev-history-2026-07-27` — подход
  Шереметьева разобран по repomix-снимку `iva-one` (2026-07-03): библиотека `libs/design-system`
  = публикуемый npm-пакет `@iva/design-system`, каталог `src/lib/figma-integration/` (448 файлов)
  с тремя генераторами (цвета через плагин `var-exporter` → `colors/generated/*.scss`;
  типографика через Figma API, узел `4151:90`; иконки — ручной экспорт SVG + svgo + генерация
  Angular-компонентов), рантайм-переключение тем `data-iva-scheme`, единый SCSS-API
  `main.ivaGetColor(...)`, Storybook как единственная документация с писаными конвенциями
  (`src/lib/docs/guides/`), пофайловая миграция легаси по таблице `design-system-comparison.md`.
  Пути `figma-integration/*` я перепроверил грепом по снимку — совпадают.
- Другие релевантные заметки vault: `00-Board/map-existing-vs-gap-pr-c-axis1` (дрейф
  `framework_hint: react` в `iva-web/design-system.yaml` при том, что iva-one — Angular;
  подтверждаю, файл до сих пор так и лежит, строки 17–18),
  `00-Board/impl-prc-consolidation`, `00-Board/gate-prc-*`, `90-Materials/chat-ii-dizain-sistema-2026-07-27`,
  `12-Features/tz1-figma-ds-handoff-ostatok.md`.
- Снимков `iva-one` свежее 2026-07-03 в локальных материалах **не найдено**.

---

## 6. Риски (без предложений — они не моя зона)

1. **Роль `iva-role-web` отдаёт разработчику ДС-навыки от 15.07.** Улучшения 24–25.07 живут
   в депрекируемом `iva-web-brownfield` и в роль не попадают. Тест паритета это не ловит:
   он сверяет id, а тела — только для KMP.
2. **Эталон установки `iva-role-web` не перегенерирован** после 24–25.07 (три скилла).
3. **Дрейф `design-system.yaml` `iva-web`:** `platform: web` + `framework_hint: react` при
   Angular-репозитории — на этом поле стоит surface-router в `design-system-discovery`.
4. **Веб-ДС-навыки вне `_mirrors.yaml`** — правка в одной копии не доезжает до другой,
   CI не ловит.
5. **`figma_file_key` у `iva-mobile` пуст**, а у `iva-web` заполнен: числовая сверка с Figma
   на мобильном опирается на матч по имени.
6. **Веб-quickstart зовёт депрекируемый профиль** и требует ручной вставки правила в `AGENTS.md`.
7. **Живого прогона на Angular-репозитории не было ни разу.**

## Ограничения разведки

- Тесты не запускал (нужен Docker + Postgres) — выводы про эталоны и паритет сделаны сверкой
  файлов и хэшей, а не прогоном.
- На `helm` не ходил.
- Repomix-снимок `iva-one` статичен на 2026-07-03; что изменилось в июле — не видно.