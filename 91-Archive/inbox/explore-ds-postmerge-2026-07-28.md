---
title: explore ДС post-merge 2026-07-28 — карта пяти правок
type: note
status: current
created: 2026-07-28 09:35
repo: tacticum-dev
tags:
- board
- design-system
permalink: tacticum/00-board/explore-ds-postmerge-2026-07-28-1
updated: 2026-07-28 09:35
archived-at: 2026-08-05 15:19
---

# Разведка: карта пяти правок по ДС (post-merge 27.07)

Репо: `/Users/bubblemac/tacticum/tacticum-dev`, ветка `main`. Только чтение.
Владелец ДС-лейна: `templates/iva-kmp-development-base` (v0.12.0), зеркало —
`templates/iva-kmp-brownfield`, роль — `templates/iva-role-kmp` (v0.2.1).

---

## 1. Обратные ссылки между навыками — ВЕРДИКТ: ни одной обратной ссылки нет

Все шесть SKILL.md живут у владельца лейна:
`/Users/bubblemac/tacticum/tacticum-dev/templates/iva-kmp-development-base/ingredients/skills/<id>/SKILL.md`

| навык | строк | секция «см. также» |
|---|---|---|
| `ds-component-adoption` | 313 | **есть** — `## 11. Related` (стр. 302–313), таблица «Skill \| Relationship» |
| `web-to-kmp-screen-port` | 454 | **есть** — `## 8. Reused in-repo skills (reference, do not duplicate)` (стр. 396–415), таблица «Skill (in-repo) \| Role here» |
| `ui-mockup-match` | 386 | **нет отдельной секции**; роутинг-врезка в §«Two modes» (стр. 40–50) + `## Tools (depends on)` (стр. 66–80) |
| `compose-multiplatform-ui` | 70 | **нет**; единственная ссылка — в теле §«Design system» (стр. 27) |
| `design-token-usage` | 263 | **нет** — ни одной ссылки на соседние навыки |
| `design-system-discovery` | 82 | **нет** — ни одной ссылки на соседние навыки |

### Матрица «ссылается → на кого» (по телу SKILL.md, включая frontmatter)

| ↓ из \ → на | ds-comp-adopt | design-token-usage | ui-mockup-match | web-to-kmp-screen-port | design-system-discovery | compose-mp-ui |
|---|---|---|---|---|---|---|
| **ds-component-adoption** | — | ✅ 6 | ✅ 3 | ✅ 9 | ✅ 2 | ✅ 6 |
| **design-token-usage** | ❌ | — | ❌ | ❌ | ❌ | ❌ |
| **ui-mockup-match** | ❌ | ✅ 4 | — | ✅ 2 | ✅ 2 | ❌ |
| **web-to-kmp-screen-port** | ❌ | ✅ 10 | ✅ 7 | — | ✅ 5 | ✅ 5 |
| **design-system-discovery** | ❌ | ❌ | ❌ | ❌ | — | ❌ |
| **compose-multiplatform-ui** | ❌ | ✅ 1 (стр. 27) | ❌ | ❌ | ❌ | — |

`ds-component-adoption` ссылается на все пять соседей; **обратно на него не ссылается ни один**.

### Недостающие обратные рёбра (5 штук, все ведут в `ds-component-adoption`)

1. `web-to-kmp-screen-port` → `ds-component-adoption` — куда: таблица §8 (стр. 402–415). Точка стыка уже названа с той стороны: `ds-component-adoption:30-34` цитирует «§1.M M5 и §5 говорят прямо: элемент без `Iva*`-эквивалента — остановись и сообщи». Со стороны `web-to-kmp-screen-port` этот стоп никуда не маршрутизирует.
2. `design-token-usage` → `ds-component-adoption` — в файле вообще нет ссылок на соседей; `ds-component-adoption:224,307` делегирует ему токены «wholesale».
3. `ui-mockup-match` → `ds-component-adoption` — куда: врезка «Route out before you start» (стр. 42–50), где уже разведены `web-to-kmp-screen-port` и `design-token-usage`.
4. `design-system-discovery` → `ds-component-adoption` — ссылок на соседей нет вовсе.
5. `compose-multiplatform-ui` → `ds-component-adoption` — есть только ссылка на `design-token-usage` (стр. 27), секции «см. также» нет.

### Где ещё упоминается `ds-component-adoption` (call-sites вне SKILL.md)

- `templates/iva-kmp-development-base/manifest.yaml:215-240` — объявление ingredient
- `templates/iva-kmp-development-base/CHANGELOG.md` (1 упоминание)
- `templates/iva-role-kmp/CHANGELOG.md` (3)
- `templates/iva-role-kmp/ingredients/repo-configs/claude-code/CLAUDE.md.fragment:32,35` и `.../codex/AGENTS.md.fragment` — секция-маршрутизатор «UI по эталону»
- `apps/backend/tests/e2e_install/test_install_flow.py` (3), golden `iva-role-kmp/{codex,claude-code}.json`
- `docs/user_manuals/iva-kmp-figma-mapping-quickstart.md:63`
- `docs/runbooks/prod-seed-2026-07-27-rollback.md`

### Важно про зеркало

`templates/_mirrors.yaml` (пара `iva-kmp-development-base` ↔ `iva-kmp-brownfield`) **не содержит**
`ds-component-adoption` и `web-to-kmp-screen-port` — они owner-only. Зато `ui-mockup-match`,
`compose-multiplatform-ui`, `design-token-usage`, `design-system-discovery` в списке есть →
байт-в-байт синхрон обязателен, правка в них требует того же PR в зеркале.
Сверка: `scripts/check_mirror_sync.py`, `apps/backend/tests/catalog/test_role_replacement_parity.py`.

---

## 2. Неверное утверждение о доставке в CHANGELOG 0.4.0 — ВЕРДИКТ: подтверждено, утверждение ложно

### (а) Файл и строка

`/Users/bubblemac/tacticum/tacticum-dev/templates/iva-kmp-development-base/CHANGELOG.md`,
запись `## [0.4.0] — 2026-07-24`, строки **352–361**. Дословно (стр. 356–358):

```
  `codex_target_path` унифицирован на тот же `'AI common/skills/{ingredient_id}/SKILL.md'` —
  KMP-репо потребляет навыки через opencode (`opencode.json`, `ai-skills` path=`AI common/skills`),
  т.е. и claude-code, и codex-агенты читают ОДИН репо-нативный файл. `body_path`
```

### (б) Манифест слоя

`templates/iva-kmp-development-base/manifest.yaml`, для трёх репо-нативных навыков — дословно
(`web-to-kmp-screen-port`, стр. 241–250; идентично у `ds-component-adoption` 215–224 и
`web-to-kmp-source-reference` 266–275):

```yaml
- ingredient_id: web-to-kmp-screen-port
  kind: skill_spec
  tier: full
  supports:
  - claude-code
  - codex
  install_scope: repo
  target_path_template: 'AI common/skills/{ingredient_id}/SKILL.md'
  codex_target_path: 'AI common/skills/{ingredient_id}/SKILL.md'
  body_path: ingredients/skills/web-to-kmp-screen-port/SKILL.md
```

### (в) Эталон установки — противоречит манифесту

`apps/backend/tests/e2e_install/golden/iva-role-kmp/codex.json`:

```
"repo/.agents/skills/ds-component-adoption/SKILL.md":   "126b22b1…",
"repo/.agents/skills/web-to-kmp-screen-port/SKILL.md":  "24dffa40…",
"repo/.agents/skills/web-to-kmp-source-reference/SKILL.md": "11315ad4…",
```

`apps/backend/tests/e2e_install/golden/iva-role-kmp/claude-code.json:37-39`:

```
"repo/AI common/skills/ds-component-adoption/SKILL.md":   "126b22b1…",
"repo/AI common/skills/web-to-kmp-screen-port/SKILL.md":  "24dffa40…",
"repo/AI common/skills/web-to-kmp-source-reference/SKILL.md": "11315ad4…",
```

Файлы одинаковые (совпадают хеши), но **лежат в двух разных местах**: claude-code — в
`AI common/skills/`, codex — в `.agents/skills/`.

### Почему так — код

- `apps/backend/src/backend/catalog/infrastructure/renderers/codex.py:79` — путь скилла **захардкожен**:
  `relative_path=f".agents/skills/{s.ingredient_id}/SKILL.md"`. `codex_target_path` тут не читается.
- `apps/backend/src/backend/catalog/domain/renderer.py:281-284` — `codex_target_path` используется
  только через `_CLI_BODY_KEYS` / `_cli_body_passthrough`, т.е. **только для агентов с
  рукописным `codex_body`**. У `skill_spec` `codex_body_path` нет → passthrough не срабатывает.
- claude-code берёт путь из `target_path_template`:
  `apps/backend/src/backend/catalog/domain/renderer.py:207-212` (боевой путь, тело пишется
  verbatim) и `.../renderers/claude_code.py:47`.

**Итог:** `codex_target_path` у `skill_spec` — мёртвая метадата. Ложны обе стороны: и фраза
в CHANGELOG, и объявление в манифесте (объявлено `AI common/…`, ставится `.agents/…`).
CHANGELOG 0.4.0 — историческая запись (текущая версия 0.12.0), поправка логичнее отдельной
записью, а не переписыванием истории.

---

## 3. Русские триггеры соседним навыкам — ВЕРДИКТ: у обоих навыков русская поверхность бедная

### Откуда берётся триггерная поверхность

**Доставляется frontmatter самого SKILL.md, а не `description_trigger` из манифеста.**
Доказательство: в golden-деревьях (`e2e_install/golden/iva-role-kmp/*.json`) хеш установленного
файла = хеш исходного `SKILL.md` байт-в-байт (`shasum -a 256` совпадает). Механика:
- claude-code — `renderer.py:207-212`, ветка `_WRITE_FILE_KINDS`: `content = body` без обёртки;
- codex — `renderers/codex.py:71-72`: если тело начинается с `---\n`, оно берётся as-is.

`description_trigger` в манифесте — витрина каталога (поиск/листинг), в установленный файл не попадает.
Правку триггеров надо делать в **SKILL.md**; `description_trigger` желательно править синхронно,
чтобы каталог не расходился с файлом.

### `ui-mockup-match` — текущий description (SKILL.md, стр. 1–18)

```yaml
description: >
  Use after the coder has rendered UI to close the gap between the live runtime
  UI and the approved reference. Snapshots the runtime UI via playwright (DOM,
  `:webApp`) or the Compose semantics tree (`runComposeUiTest`, Desktop/Android),
  diffs against the mockup at semantic-token + DOM granularity, emits a structured
  delta list as a PIN-patch for the next `/run-coder` iteration. Also runs a
  **Figma numeric-compare mode**: against one Figma frame (`node-id`) it emits
  token-anchored numbers — ΔE on named colour tokens, dp size/inset deltas,
  token-conformance — inside a stated tolerance, and names in the report what a
  semantics tree cannot measure at all. Never a blind pixel diff. Triggers on
  "UI doesn't match mockup", "doesn't look like the design", "натянуть макет",
  "compare with mockup", "UI deviates from MOCKUPS", "fix UI to mockup",
  "сверь экран с макетом по числам", "числовая сверка с макетом", "экран не
  соответствует макету", "compare with the Figma frame", "numeric mockup match",
  "ΔE / size-delta acceptance".
```

Русских триггеров 4, анти-триггеров нет (роутинг «куда не сюда» есть в теле, стр. 42–50,
но в поверхность не вынесен).

### `compose-multiplatform-ui` — текущий description (SKILL.md, стр. 1–9)

```yaml
description: >
  Use when implementing Compose Multiplatform UI in the KMP repo — Screen/Content
  composables, the Iva design system theme, stability/immutable state, Decompose wiring,
  compose-resources. Routes to the repo compose-ui-patterns skill. Triggers on "Composable",
  "Compose", "Screen", "design system", "IvaTheme", "AppColors", "theme", "LazyColumn",
  "UI", "экран", "компонент".
```

Русских триггеров 2 («экран», «компонент»), оба — одиночные слова, пересекаются с
`ds-component-adoption` и `web-to-kmp-screen-port`; анти-триггеров нет.

### Образец стиля — `web-to-kmp-screen-port` (SKILL.md, стр. 1–29)

```yaml
description: >
  Use for screen-level work in the su.ivcs.messenger KMP repo that has an outside reference to match.
  TWO tracks. Track W — port an existing iva-one Angular screen into KMP: read the Angular source
  (page-route, signalStore, view-state, DS component names, Transloco keys, REST contract), rewrite it
  into a Decompose component + Compose UI on the Iva* design system, verify parity against the iva-one
  sample and the Figma frame. Track M — a screen that ALREADY EXISTS in this repo was built from a
  design frame that has since been REPLACED: snapshot the current screen, read the new frame, diff
  them, and change only the presentation (added / changed / removed elements) while its behaviour,
  state holder and use-cases stay frozen. Track W is a rewrite-port (Angular template + signalStore →
  Decompose + Compose), NOT a Kotlin move-port. Three things this is NOT: it is not the token-value
  revision pass ("the token values changed, go over the screen" → design-token-usage); not building a
  screen that has no reference at all — no existing screen and no web sample ("собери новый экран по
  макету" from scratch → compose-multiplatform-ui); and not the post-implementation verification
  loop against the reference the screen was built from ("после реализации не похоже на макет",
  "сверь экран с макетом по числам" → ui-mockup-match). This skill is the AUTHORING pass against a
  reference, and it ends in that loop rather than replacing it.
  Orchestrates the repo's own skills — decompose-su-ivcs-messenger-kmp,
  mvi-state-machine-su-ivcs-messenger-kmp, compose-ui-patterns-su-ivcs-messenger-kmp,
  compose-multiplatform-ui, clean-architecture-su-ivcs-messenger-kmp, design-system-discovery,
  design-token-usage, ui-mockup-match, kmp-ui-testing, iva-web-ecosystem, and
  android-to-kmp-porting-su-ivcs-messenger-kmp (as a contrast, not a template) — without duplicating them.
  Triggers on "port screen", "Angular to Compose", "iva-one screen", "signalStore to Decompose",
  "web to KMP", "перенос экрана", "перенести экран из веба", "Angular в Compose", "экран iva-one",
  "новый макет", "экран по новому макету", "привести экран к новому макету", "обновить экран по макету",
  "экран написан по старому макету", "редизайн экрана", "перерисовать экран", "new design frame",
  "screen redesign", "update an existing screen to the new design", "screen built from the old frame".
```

Приём образца: 13 русских триггеров-фраз + блок «Three things this is NOT» с явными
анти-триггерами и стрелками в конкретные навыки.

### Копии в репозитории (правка синхронно!)

**`ui-mockup-match` — 6 копий, 5 разных содержимых:**

| путь | sha256 |
|---|---|
| `templates/iva-kmp-development-base/ingredients/skills/ui-mockup-match/SKILL.md` | `cb50a5e6…` ← **владелец** |
| `templates/iva-kmp-brownfield/ingredients/skills/ui-mockup-match/SKILL.md` | `cb50a5e6…` ← **зеркало, байт-в-байт обязательно** |
| `templates/iva-brownfield-mail/ingredients/skills/ui-mockup-match/SKILL.md` | `0c1eff93…` (свой лейн, свой текст) |
| `templates/iva-rn-brownfield/ingredients/skills/ui-mockup-match/SKILL.md` | `3d283d65…` |
| `templates/iva-web-brownfield/ingredients/skills/ui-mockup-match/SKILL.md` | `c8bdbd1a…` |
| `templates/tacticum-ui-base/ingredients/skills/ui-mockup-match/SKILL.md` | `149031cc…` |

Плюс `description_trigger` в манифестах: `iva-kmp-development-base/manifest.yaml:194-214`
и `iva-kmp-brownfield/manifest.yaml:221-231`.

**`compose-multiplatform-ui` — 2 копии, обе идентичны (`1703cd67…`):**
- `templates/iva-kmp-development-base/ingredients/skills/compose-multiplatform-ui/SKILL.md` (владелец)
- `templates/iva-kmp-brownfield/ingredients/skills/compose-multiplatform-ui/SKILL.md` (зеркало)

Манифесты: `iva-kmp-development-base/manifest.yaml:131-143`, `iva-kmp-brownfield/manifest.yaml:233-243`.

**`web-to-kmp-screen-port` / `ds-component-adoption` — по 1 копии** (owner-only, зеркала нет).

---

## 4. Квикстарт зовёт старый профиль — ВЕРДИКТ: правка уже сделана, задача неактуальна

`docs/user_manuals/iva-kmp-figma-mapping-quickstart.md` в текущем `main` **строка 15**:

```
| Роль `iva-role-kmp` установлена **и обновлена до последней версии** (≥ 0.2.1) | установка — [quickstart роли](iva-role-kmp-profile-quickstart.md); обновление — Шаг 1 |
```

Единственное упоминание `iva-kmp-brownfield` во всём файле — **строка 77**, и оно корректное
(врезка про переход, а не требование старого профиля):

```
> Остался на старом профиле `iva-kmp-brownfield`? Маршрутизатора там нет — он
> появился только у роли. Переходи на роль по её quickstart, раздел «Если раньше
> был установлен старый профиль».
```

Починено коммитом `2a514c4` «docs(kmp): квикстарты — роль вместо старого профиля, платный тариф
Figma, ручная вставка не нужна» (2026-07-27, ветка `feat/kmp-agents-router`, уже в `main`).
До него строка 15 читалась: `| Профиль `iva-kmp-brownfield` установлен **и обновлён до последней версии** | …`.

Образец формулировки перехода — `docs/user_manuals/iva-role-kmp-profile-quickstart.md:74-90`,
секция `## Если раньше был установлен старый профиль (`iva-kmp-brownfield`)`: 5 пунктов —
токен тот же, `installation_id` новый, одноимённые ингредиенты перезапишутся, секцию старого
маркера убрать, MCP дозакинуть.

**Побочная находка:** та же болезнь жива в веб-близнеце —
`docs/user_manuals/iva-web-figma-mapping-quickstart.md:22` и `:43` требуют профиль
`iva-web-brownfield`, роли `iva-role-web` там нет.

---

## 5. Платный гейт в квикстарте роли — ВЕРДИКТ: про Tacticum-премиум (ADR-0028) нет ни слова

`docs/user_manuals/iva-role-kmp-profile-quickstart.md` — поиск по
`402|seat_required|подписк|premium|тариф|subscription|Payment Required|платн` даёт **4 попадания,
и все про платный тариф Figma, а не про подписку Tacticum**:

- стр. 35 — таблица «Что понадобится»: «**Платный** аккаунт Figma и seat уровня **Dev или Full**…»
- стр. 64–68 — «**Figma MCP — платный тариф.** Локальный сервер Figma desktop живёт в Dev Mode…»
- стр. 293 — таблица «Если что-то пошло не так»: «В Figma Dev Mode нет секции «MCP» … Тариф и seat: нужен платный Figma…»

Про `402` от `design_*` / premium-гейт `tacticum-mcp` — **ничего**.

### Готовая формулировка для переиспользования (есть в 5 других квикстартах, дословно одна и та же)

```
| `design_*` возвращает 402 | Подписка без premium — design-инструменты закрыты server-side. Это не ошибка установки. |
```

Строки-источники:
- `docs/user_manuals/iva-web-brownfield-profile-quickstart.md:417`
- `docs/user_manuals/iva-ios-brownfield-profile-quickstart.md:444`
- `docs/user_manuals/iva-rn-brownfield-profile-quickstart.md:401`
- `docs/user_manuals/iva-brownfield-mail-profile-quickstart.md:395`
- `docs/user_manuals/firebird-web-brownfield-profile-quickstart.md:426`

Вариант базы (другая формулировка, про отсутствие скиллов):
- `docs/user_manuals/tacticum-dev-base-profile-quickstart.md:415` — «База не ставит design-скиллы; design-инструменты — premium и стек-профильная тема. Это не ошибка установки.»

Куда вставлять в квикстарте роли: таблица `## Если что-то пошло не так` (стр. 282–294),
рядом со строкой 293 про Figma-тариф — чтобы два разных «платно» стояли рядом и не путались.

**Побочная находка:** `docs/user_manuals/iva-kmp-brownfield-profile-quickstart.md` тоже не
упоминает 402/premium — то есть в KMP-ветке доков этой строки не было никогда, а не потеряли при переезде.

### Первоисточник гейта

`docs/adr/0028-tacticum-design-two-phase-rollout.md`:
- стр. 52 — «**MCP design tools (read side, MVP)** — premium gate strict. AuthScope check: `subscription.status='active' AND plan IN ('trial','team','enterprise')`. Free tier → 402 Payment Required. **Downgrade semantics**: при понижении до free агент перестаёт видеть design tools…»
- стр. 51 — «**design_web (Phase 2)** — premium **corporate** only…»

В навыках лейна гейт уже описан:
- `design-system-discovery/SKILL.md:14-16` — «premium-gated per ADR-0028; falls back to 402 Payment Required if subscription is missing»
- `design-system-discovery/SKILL.md:78-82` — секция `## When `design_*` returns 402 Payment Required`
- `design-token-usage/SKILL.md:20-21` — «(premium-gated per ADR-0028)»

То есть агент про гейт знает, а человек, читающий квикстарт роли, — нет.