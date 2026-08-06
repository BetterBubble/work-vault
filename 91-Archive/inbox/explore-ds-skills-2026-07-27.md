---
title: 'Разведка: что из дизайн-системы доезжает до разработчика через профиль'
type: note
status: draft
created: 2026-07-27
tags:
- board
- design-system
- explore
permalink: tacticum/00-board/explore-ds-skills-2026-07-27
archived-at: 2026-08-04 10:01
---

# Разведка: ДС → профиль → разработчик (уровни 4–5)

Репозиторий: `/Users/bubblemac/tacticum/tacticum-dev`, ветка `main`, HEAD `28e3109`
(последний коммит репо на момент разведки — `119ccb0 2026-07-26 Add BPMS process skills to analysis profile`).
Только чтение, правок нет.

---

## 0. Короткий ответ

Разработчик получает ДС **не как один канал, а как три несвязанных**:

1. **Серверный канал (MCP `tacticum-mcp`)** — `design_list_systems` / `design_get_tokens` /
   `design_get_theme_tokens` / `design_resolve_token`. Отдаёт DTCG-дерево токенов и (внутри
   `$extensions`) словарь Figma→код. Ни одного чтения из репозитория.
2. **Репо-локальный канал** — навыки, которые описывают ДС словами по коду конкретного репо
   (`iva-core-design-system`, `angular-ds-component-*`, `compose-multiplatform-ui`,
   `ios-module-integration`). Сервера не касаются вовсе (кроме шага «резолвни токен»).
3. **Figma-канал** — только через `figma-desktop` MCP и только в двух quickstart-доках
   (не в профиле): пользователь руками вставляет блок правил в `AGENTS.md` репозитория.

Покрытие поверхностей **резко неравномерное**: Web Angular — полный конвейер, KMP/Compose —
конвейер есть, но с открытыми TODO, iOS — одна строка про `Color.IV…`, Desktop Qt — **ноль**
(явно объявлено «Qt берёт веб-токены»).

---

## 1. Каталог ДС-навыков: где лежат и что делают

**8 профилей** содержат навыки, связанные с ДС/дизайном/Figma. Всего **9 уникальных
имён навыков** ДС-семейства (не считая копий).

| Навык | В каких профилях | Что делает | Источник данных | Последний коммит |
|---|---|---|---|---|
| `design-system-discovery` | `tacticum-ui-base`, `iva-analysis-base`, `iva-fr-analyst`, `iva-web-brownfield`, `iva-kmp-brownfield`, `iva-rn-brownfield`, `iva-brownfield-mail` (**7 копий, 6 разных тел**) | design-фаза: перечислить привязанные ДС и вытянуть токены | MCP `tacticum-mcp` | ui-base `7d6986b` 2026-07-15; analysis `57ec7cc` 2026-07-22; web `eb70dcb` 2026-07-24 |
| `design-token-usage` | `tacticum-ui-base`, `iva-web-brownfield`, `iva-kmp-brownfield`, `iva-rn-brownfield`, `iva-brownfield-mail` | impl-фаза: резолв пути токена → значение, привязка через theme bridge | MCP | ui-base `7d6986b` 2026-07-15 |
| `ui-mockup-match` | те же 5 профилей (**2 разных тела**) | цикл сверки runtime-UI с макетом, playwright, ≤3 итерации | playwright MCP + `design_*` | ui-base `7d6986b`; web-brownfield `eb70dcb` 2026-07-24 |
| `pin-ui-pipeline-check` | те же 5 профилей | аудит PIN на полноту UI-пайплайна | ничего (текстовый чеклист) | `7d6986b` 2026-07-15 |
| `angular-ds-component-usage` | `iva-web-development-base`, `iva-web-brownfield` | **Сценарий 2**: собрать экран из готовых компонентов `@iva/design-system` по Figma-фрейму | Figma MCP + `design_get_tokens` (словарь `$extensions`) | `789dbde` 2026-07-25 |
| `angular-ds-component-authoring` | `iva-web-development-base`, `iva-web-brownfield` | **Сценарий 1**: написать НОВЫЙ компонент в библиотеку ДС | репо + Figma | `2d919de` 2026-07-25 |
| `iva-core-design-system` | `iva-web-development-base`, `iva-web-brownfield` | вторая ДС (конференц/MCU, npm `iva-core`, Figma VCSWEB) | **только репо** (`core-demo`, `CHANGES.md`) | `eb70dcb` 2026-07-24 |
| `compose-multiplatform-ui` | `iva-kmp-development-base`, `iva-kmp-brownfield` | Compose MP UI: `AppColors`/`IvaTheme`, стабильность, Decompose | репо (`AI common/skills/compose-ui-patterns-…`) | `508854f` 2026-07-22 |
| `tacticum-design-tokens` | `tacticum-internal-base`, `tacticum-internal-dev` | токены СВОЕЙ ДС Tacticum (`tacticum-web`) для внутренних консолей | MCP | `7ba69ff` 2026-07-22 |
| `mockup-authoring` | `iva-analysis-base`, `iva-fr-analyst` | HTML-артборды для FR/ТЗ на токенах | зависит от `design-system-discovery` | `57ec7cc` 2026-07-22 |
| `web-to-kmp-screen-port` | `iva-kmp-development-base` (**owner-only, в зеркало не входит**) | перенос экрана iva-one Angular → KMP Compose | iva-one (read-only) + KMP-репо + Figma-фрейм | `39ae642` 2026-07-24 |
| `web-to-kmp-source-reference` | `iva-kmp-development-base` (owner-only) | модель доступа к двум деревьям (источник read-only / цель write) | — | `d715669` 2026-07-24 |

**`figma-quickstart` как навыка НЕ существует.** Искал: `templates/*/ingredients/skills/`,
grep `figma-quickstart` по всему репо — ноль. Figma-quickstart'ы — это **два документа**, не навыки:

- `/Users/bubblemac/tacticum/tacticum-dev/docs/user_manuals/iva-web-figma-mapping-quickstart.md` (142 строки, `a07f504` 2026-07-24)
- `/Users/bubblemac/tacticum/tacticum-dev/docs/user_manuals/iva-kmp-figma-mapping-quickstart.md` (108 строк)

Оба помечены «**Статус: пилот**» и оба требуют, чтобы **пользователь руками** добавил блок
между маркерами `<!-- figma-mapping-pilot -->` в `AGENTS.md` своего репозитория (Шаг 3).
Т.е. Figma-канал профилем **не доставляется** — он доставляется инструкцией человеку.

---

## 2. `design-system-discovery` целиком — разбор

### 2.1 Что вызывает

Каноничная (ui-base) версия — `/Users/bubblemac/tacticum/tacticum-dev/templates/tacticum-ui-base/ingredients/skills/design-system-discovery/SKILL.md`:

- `whoami()` — проверка auth-контекста (строка 22)
- `design_list_systems(installation_id=...)` — один раз в начале design-фазы (строка 37)
- `design_get_theme_tokens(ds_id, mode="light", groups=[])` → зонд `available_groups`,
  затем `design_get_theme_tokens(..., groups=[нужные])` (строки 41–43)

**Внутреннее противоречие в самом файле:** таблица «Tools» (строка 24) заявляет
`design_get_tokens(design_system_id, version?)` как основной тул, а тело (строка 41) велит
звать `design_get_theme_tokens`. `manifest.yaml:75` (`description_trigger`) тоже говорит
`design_list_systems / design_get_tokens`. Т.е. заявленный и предписанный тул расходятся
внутри одного ингредиента.

### 2.2 Требование groups-фильтрации и почему

Строки 43–45, дословно: **«Never fetch without `groups`»** — полная ДС 150+ КБ, переполняет
лимит tool-result клиента, «the spilled file gets skipped and hex values end up hard-coded
in mockups». То есть мотив не экономия, а **прямая причина хардкода hex в макетах**.

**Числа подтверждаются кодом и данными:**

- `/Users/bubblemac/tacticum/tacticum-dev/design-systems/iva-web/tokens.json` — **366 677 байт**
- `/Users/bubblemac/tacticum/tacticum-dev/design-systems/iva-mobile/tokens.json` — **324 796 байт**
- `design_get_theme_tokens` имеет `groups` и `max_chars: int = 50_000`
  (`apps/backend/src/backend/design/interface/mcp/design_get_theme_tokens.py:24-26`),
  при перерасходе отдаёт `truncated: True` + `omitted_groups` (строки 86–88)
- **`design_get_tokens` параметра `groups` НЕ имеет вовсе**
  (`apps/backend/src/backend/design/interface/mcp/design_get_tokens.py:29-33`) и возвращает
  `tokens` целиком (строка 91)

**Отсюда фактический разрыв:** 3 из 7 копий навыка (`iva-web-brownfield`,
`iva-rn-brownfield`, `iva-brownfield-mail`) предписывают именно `design_get_tokens(design_system_id)`
без фильтра — то есть **ровно то, что канон запрещает**. Плюс `angular-ds-component-usage:94`
и оба Figma-quickstart'а (`iva-web-figma-mapping-quickstart.md:100`,
`iva-kmp-figma-mapping-quickstart.md:61`) тоже зовут `design_get_tokens` без фильтра.

Оговорка в пользу них: **словарь code-bindings живёт в `$extensions` и через
`design_get_theme_tokens` недоступен** — `available_groups` явно отсекает ключи на `$`
(`design_get_theme_tokens.py:51`). То есть безфильтровый вызов у Figma-сценария —
не небрежность, а единственный доступный путь. Но у `design-system-discovery` в
web/rn/mail это именно расхождение с каноном.

### 2.3 Поведение без подписки (402)

Все 7 копий говорят «402 Payment Required» и «surface the upgrade hint, do not fabricate
token values» (ui-base строки 78–82).

**Кода, который отдаёт 402, для design-тулов нет.** `require_premium_tier`
(`apps/backend/src/backend/design/interface/mcp/common.py:7-23`) бросает
`AuthError("seat_required", ...)` при `scope.tier not in ("full","trial")`.
Тест закрепляет именно это: `apps/backend/tests/design/test_mcp_design_tools.py:419` —
`assert exc.value.code == "seat_required"`. Единственное место в бэкенде, где реально
возвращается `HTTP_402_PAYMENT_REQUIRED`, — роутер другого контекста
(`apps/backend/src/backend/architecture/interface/api/router.py:156`, SKU `arch`).
Т.е. навык учит агента ловить ошибку, которой design-тулы не выдают.

### 2.4 Читает ли что-то из репозитория

**Нет, ничего.** Единственное обращение к файлу репо — чтение `installation_id` из
`.tacticum/context.yaml` (строки 26–29, 37). Ни исходников, ни библиотеки ДС, ни
`package.json`. Единственное исключение — **версия `iva-web-brownfield`** (95 строк,
самая большая): она добавляет repo-identity в правило выбора ДС — `package.json` `name`,
`default_design_system_id` из `.tacticum/context.yaml`, и surface-routing
(`libs/design-system` / `@ivcs/ui-kit` / импорты `from 'iva-core'`).

### 2.5 Шесть разных тел одного навыка

| Профиль | строк | Чем отличается |
|---|---|---|
| `tacticum-ui-base` | 82 | канон: groups-фильтрация, «Iva DS едина для всех поверхностей» |
| `iva-kmp-brownfield` | 82 | канон + слово «Compose» в трёх местах |
| `iva-analysis-base` / `iva-fr-analyst` | 76 | **полностью по-русски**, для мокапов FR/ТЗ; добавлено правило **zero-resolve** (после загрузки дерева — ноль вызовов `design_resolve_token`) |
| `iva-rn-brownfield` | 78 | `design_get_tokens` без groups; хардкод выбора: `iva-web` для web/desktop/Qt, `iva-mobile` для mobile |
| `iva-brownfield-mail` | 78 | то же, что rn, + «не требовать отдельную Qt-ДС» |
| `iva-web-brownfield` | 95 | `design_get_tokens` без groups + repo-identity + surface-routing на `iva-core` |

`_mirrors.yaml:15-23` держит байт-в-байт только пару `iva-analysis-base` ↔ `iva-fr-analyst`.
Остальные пять копий — **свободно разошедшиеся**, не под контролем `check_mirror_sync.py`.

---

## 3. Карта профиль → навык: кто что получает

### 3.1 Архитектура (ADR-0059): роль = core + lane'ы

`tacticum-ui-base` (v0.1.0, `7d6986b` 2026-07-15) — **общий UI-кластер**: 4 навыка +
playwright MCP. `manifest.yaml:5-11` прямо говорит: не смог войти в `tacticum-dev-base`,
потому что у Go-профиля нет UI.

Кто его подключает (`depends_on`, извлечено из всех 48 манифестов):

| Профиль | получает `tacticum-ui-base` |
|---|---|
| `iva-role-web` v0.1.1 | **да** |
| `iva-role-kmp` v0.1.1 | **да** |
| `iva-role-ios` v0.1.1 | **да** |
| `iva-role-mail` v0.1.1 | **да** |
| `firebird-role-web` v0.1.0 | **да** |
| `iva-ios-brownfield` v0.1.3, `firebird-web-brownfield` v0.1.3 | **да** |
| `iva-role-go`, `iva-role-java`, `iva-role-qa`, `iva-role-analyst`, `iva-role-architect` | нет |
| `tacticum-role-internal` / `-platform` / `-techwriter` | нет |

Монолиты (`iva-web-brownfield` v0.5.1, `iva-kmp-brownfield` v0.5.2,
`iva-rn-brownfield` v0.5.4, `iva-brownfield-mail` v0.7.4) **не имеют `depends_on`** — они
носят свои копии тех же 4 навыков внутри себя. Отсюда и разошедшиеся тела.

### 3.2 Что реально получает разработчик каждой поверхности

| Роль | Из `tacticum-ui-base` | Из своего стек-лейна | Итого по ДС |
|---|---|---|---|
| **`iva-role-web`** (Angular) | discovery, token-usage, pin-ui-check, **ui-mockup-match (HTML-only, 149 строк)** | `angular-ds-component-usage`, `angular-ds-component-authoring`, `iva-core-design-system` | Полный конвейер. **Но:** усечённый `ui-mockup-match` |
| **`iva-role-kmp`** | те же 4 | `compose-multiplatform-ui`, `web-to-kmp-screen-port`, `web-to-kmp-source-reference` | Конвейер есть, приёмка через Roborazzi/VLM (описана, не автоматизирована) |
| **`iva-role-ios`** | те же 4 | **ничего ДС-специфичного** | Только generic-навыки + одна строка `Color.IV…` в `ios-module-integration/SKILL.md:64` |
| **`iva-role-mail`** (Qt) | те же 4 | **ничего ДС-специфичного** | Только generic. Qt-навыки (`qt-ui-testing`, `qt-runtime-deployment`) про сборку и тесты, не про ДС |
| `iva-role-analyst` | — (нет ui-base) | `design-system-discovery` (рус.) + `mockup-authoring` из `iva-analysis-base` | Свой канал под FR/ТЗ |

**Разрыв, который стоит подсветить.** `iva-web-development-base` **не содержит**
`ui-mockup-match` — значит `iva-role-web` берёт его из `tacticum-ui-base`, а это версия
**без Figma numeric-compare** (149 строк, «HTML mockups only … Figma URLs and PNG exports
are out of scope», строки 27–28). Режим числовой приёмки по Figma (ΔE CIEDE2000 +
px size-deltas + token-conformance, ~112 добавленных строк) есть **только** в копии
`iva-web-brownfield` (296 строк). Поэтому `angular-ds-component-usage:37,157-161`
формулирует шаг 7 условно: «use its Figma numeric-compare mode **when the attached
`ui-mockup-match` provides it**». А `iva-web-figma-mapping-quickstart.md:129-130` говорит
про числовую приёмку уже утвердительно — и требует профиль `iva-web-brownfield` (строка 22).
То есть **числовая приёмка макета доезжает только до монолита, не до роли.**

### 3.3 Почему ДС-навыки продублированы, а не зеркалятся

`iva-web-development-base/CHANGELOG.md:5-11,15-21` прямым текстом: «DS-навыки не зеркалятся
в паре `_mirrors.yaml`» — `angular-ds-component-*` (v0.1.1) и `iva-core-design-system`
(v0.1.2) были **скопированы** в лейн, чтобы прошёл `test_role_covers_replaced_profile`
(«main CI red»). Т.е. покрытие роли достигнуто копипастой под давлением красного CI,
а не переносом владения. `_mirrors.yaml:44-58` подтверждает: в паре
`iva-web-development-base` ↔ `iva-web-brownfield` перечислены 15 ингредиентов, ДС-навыков
среди них нет.

---

## 4. Четыре поверхности: что покрыто, а что упомянуто

### Web Angular — покрыто полнее всех
- Сценарий 1 (новый компонент ДС): `angular-ds-component-authoring`, 195 строк
- Сценарий 2 (экран по Figma-фрейму): `angular-ds-component-usage`, 178 строк — предполётная
  проверка фрейма (auto layout + инстансы мастер-компонентов, строки 61–77), резолв через
  Code Connect → словарь по `figma_key` → имя/алиас, стоп на незамапленном (строки 106–121),
  правила композиции Angular (директива `button[ivaButton]` vs элемент `iva-*`, `iva-form-field`,
  `[ivaMenuTriggerFor]`, строки 130–139)
- Вторая ДС конференции: `iva-core-design-system`, 83 строки. **Честно помечено как
  недоступное:** «server-side `iva-core` design system … **not yet registered**»,
  «code-bindings dictionary … **not yet built**» (строки 60–70)
- Данные: `design-systems/iva-web/` — ДС **v0.3.0**, **49 компонентов** в code-bindings,
  **32 с `figma_key`** (17 — `null`, «компонент в другом файле многофайловой ДС»)

### KMP / Compose Multiplatform — покрыто, но с открытыми хвостами
- `compose-multiplatform-ui` (70 строк) — тонкий фронт к репо-навыку
  `compose-ui-patterns-su-ivcs-messenger-kmp`, стек прибит: Compose MP 1.11.1, Material3
  jetbrains 1.9.0, Decompose 3.5.0
- `ui-mockup-match` в KMP-копии (158 строк) добавляет важное: у Compose Desktop/Android
  **нет DOM**, playwright неприменим → снимок берётся из **semantics tree**
  `runComposeUiTest` (строки 68–76). Для `:webApp` (Kotlin/JS) DOM-путь работает как есть
- Данные: `design-systems/iva-mobile/` — ДС **v0.2.0**, **34 компонента**, **0 с `figma_key`**
  (в KMP-quickstart, строки 66–68: «ключи Figma будут добавлены после подтверждения файла
  ДС дизайнерами»). Резолв — **только по имени**

### iOS — не покрыто
Ни одного ДС-навыка в `iva-ios-development-base` / `iva-ios-brownfield`. Grep по
`design.system|design token|SwiftUI|figma` даёт единственное содержательное попадание:
`ios-module-integration/SKILL.md:64` — «Colors: design-system colors `Color.IV…`
(e.g. `Color.IV.Text.mPrimaryInverse`), not raw `Color.white` — the `iva_use_IVA_color` rule».
Т.е. ДС на iOS доезжает как **правило линтера**, а не как канал токенов. Связи
`Color.IV.*` ↔ токены сервера / `design_resolve_token` в репо не найдено.
Роль `iva-role-ios` при этом получает `tacticum-ui-base` — четыре generic-навыка,
у которых примеры и theme-bridge'и написаны под CSS/Tailwind/styled-components
(`design-token-usage/SKILL.md:63-70`), не под Swift.

### Desktop Qt — не покрыто, и это объявлено намеренно
Qt-навыки существуют, но про сборку и запуск, не про ДС: `qt-ui-testing` (238 строк,
Catch2 + QTest + ApprovalTests), `qt-runtime-deployment` (QML import paths, `windeployqt`).
ДС-позиция сформулирована как отказ от отдельной Qt-ДС —
`iva-brownfield-mail/…/design-system-discovery/SKILL.md:70-72`: «Do not require a separate
Qt design system for desktop work. The designers' rule is a unified IVA design system:
desktop/Qt can use `iva-web` token base». Единственное место, где Qt вообще упомянут в
общем UI-кластере, — триггер-список `pin-ui-pipeline-check` (строка 32: «QML/QWidget/React/Vue
component»). **Ни словаря компонентов, ни Figma-моста, ни theme-bridge для Qt нет.**

### RN — поверхность, о которой в постановке не спрашивали, но она есть
`iva-rn-brownfield` v0.5.4 несёт полный набор из 4 ДС-навыков. **Роли `iva-role-rn` не
существует** — значит новая архитектура ролей эту поверхность не покрывает вовсе.
`design-systems/iva-rn/tokens.json` — 13 160 байт, **0 code-bindings** (пустой заглушечный
набор против 366 КБ у `iva-web`).

---

## 5. `web-to-kmp-screen-port` — зрелость и пилот

Файл: `/Users/bubblemac/tacticum/tacticum-dev/templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-screen-port/SKILL.md`, 287 строк.

**Что делает.** Оркестратор переноса одного экрана iva-one (Angular) в KMP-репо
su.ivcs.messenger. Это **rewrite-port, не move-port** (§0): шаблон Angular + `signalStore`
переписываются в Decompose-компонент + Compose UI. Два источника правды — Figma-фрейм
(внешний вид) и экран iva-one (логика/структура/состояние). Содержит: фиксированный порядок
чтения источника (§1, 8 шагов), таблицу соответствий Angular→Compose (§2, 14 строк),
маппинг состояния signalStore → Decompose (§3), guardrail'ы приземления (§4), правила против
AI-ошибок порта (§5), что НЕ портировать (§6: routing/DOM/CSS/RxJS/DI/WebRTC/Electron),
верификацию (§7).

**История (6 коммитов, все 2026-07-24):**
`7882902` скелет → `0e52b6a` перелинковка на реальные in-repo навыки →
`62bc27c` repo-native delivery (v0.4.0) → `d715669` навык-компаньон
`web-to-kmp-source-reference` (v0.5.0) → `65e4fbf` правки по пилоту (v0.6.0) →
`39ae642` правки по critic (v0.7.0).

**Пилот был.** `iva-kmp-development-base/CHANGELOG.md:33-35` — «уточнения по
grounded-фидбэку **первого реального прогона** (сухой пилот `ContactDetailScreen`,
доска `00-Board/pilot-sts.4-contact-detail-…`)». Прогон был **сухой** (dry), не боевой.

**Чем закончился — 4 вскрытых пробела** (CHANGELOG строки 36–51):
- **F-1**: читать не только `*.component.ts`, но и `*.component.html` + `data-access` +
  shell/route — без них Transloco-ключи и REST-контракт неизвлекаемы
- **F-2**: `LazyColumn`+key только для потенциально длинных списков; короткие вложенные —
  keyed `forEach` в `Column` (нельзя вкладывать `LazyColumn` в `LazyColumn`)
- **F-3**: добавлен триаж «мелкий UI-фикс vs структурная правка» для случая fix-parity
- **F-5**: статическая приёмка должна читать `*Widget.kt` / `*PartComponent`, а не только
  `*Screen.kt` — сырьё (`Color`/`dp`) чаще в них
- **F-4 (словарь `figma_key`) не трогался — «на Figma-паузе у президента»**

**Что осталось открытым (§TODO, строки 263–286):**
- Первый экран `ContactDetailScreen` — **Figma-фрейм отложен**, паритет гонится по коду
- Словарь `Iva*`↔web: **RESOLVED на доске** `00-Board/phase2-provisional-iva-web-dictionary`
  (32 `figma_key` + 17 обоснованных `null`), но **«awaits repo-native delivery»** в
  `AI common/skills/`. Ключён `Iva*`→web, а порт делает обратный lookup
- «[TODO: confirm with designers] — мобильная Figma имеет только токены, компонентной
  библиотеки нет; мобильные экраны рисуются на веб-UI-KIT компонентах»
- **«[TODO: Figma bridge in DS skills] — `design-system-discovery` / `design-token-usage`
  currently Figma-agnostic (DTCG via MCP); Figma-мост — отдельное расширение, не построено»**

**Зрелость: скелет, доведённый до пригодности одним сухим прогоном.** Доставка —
`install_scope: repo`, путь `AI common/skills/{ingredient_id}/SKILL.md` (репо-нативный
вариант). Owner-only: в зеркало `iva-kmp-brownfield` не входит (CHANGELOG:135).

---

## 6. Риски и расхождения (для решения ответственной ролью)

1. **Семь копий `design-system-discovery`, шесть разных тел, одна пара под контролем.**
   Три копии предписывают запрещённый каноном безфильтровый `design_get_tokens` на
   366-килобайтном дереве. Ровно тот сценарий, из-за которого, по тексту навыка, hex
   утекает в макеты хардкодом.
2. **«402 Payment Required» в 7 навыках — ошибки, которой код не выдаёт.** Реально
   `AuthError("seat_required")`, закреплено тестом.
3. **Внутреннее противоречие в каноне:** таблица тулов и `manifest.yaml` называют
   `design_get_tokens`, тело предписывает `design_get_theme_tokens`.
4. **Числовая приёмка Figma есть только в монолите.** `iva-role-web` её не получает;
   quickstart обещает её утвердительно.
5. **`$extensions` (словарь Figma→код) недостижим через фильтрованный тул** —
   `available_groups` отсекает `$`-ключи. Значит правило «никогда без groups» и
   Figma-сценарий структурно несовместимы; нужен либо третий тул, либо явное исключение.
6. **iOS и Qt получают UI-кластер, написанный под веб.** Theme-bridge примеры —
   CSS-переменные / Tailwind / styled-components. Для Swift и Qt эквивалента нет.
7. **RN осталась без роли.** Монолит `iva-rn-brownfield` v0.5.4 живой и с ДС-навыками,
   роли `iva-role-rn` нет.
8. **ДС-навыки в лейны скопированы под красный CI**, а не перенесены по владению — новая
   точка расхождения на будущее.

---

## 7. Где искал и не нашёл

- **`figma-quickstart` как навык** — искал: `find templates -type d -name skills`,
  `ls templates/*/ingredients/skills/*/`, `grep -ril figma-quickstart` по всему репо. Нет.
  Есть два `docs/user_manuals/*-figma-mapping-quickstart.md`.
- **Навык под iOS/SwiftUI-ДС** — искал grep `design.system|design token|SwiftUI|figma` по
  `iva-ios-development-base/` и `iva-ios-brownfield/`. Нет.
- **Навык под Qt-ДС / QML-токены** — искал grep `qml|qt quick|design.system|design token`
  по `iva-mail-development-base/` и `iva-brownfield-mail/`. Нет (кроме generic-копий).
- **Маппинг `Color.IV.*` (iOS) на токены сервера** — искал grep по `Color.IV` в templates.
  Только правило линтера, связи с `design_*` нет.
- **Следы боевого (не сухого) прогона `web-to-kmp-screen-port`** — искал grep
  `pilot|пилот` по `templates/iva-kmp-development-base`, `iva-web-development-base`,
  `tacticum-ui-base` + `git log` по путям навыка. Только сухой пилот `ContactDetailScreen`.
  Доски (`00-Board/pilot-sts.4-…`, `00-Board/phase2-provisional-iva-web-dictionary`) — в
  vault, не в репо; по постановке в vault не ходил.