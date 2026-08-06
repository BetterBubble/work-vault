---
title: recon-ds-skill-format-web-to-kmp
type: note
permalink: tacticum/00-board/recon-ds-skill-format-web-to-kmp-1
status: draft
role: explorer
repo: tacticum-dev/templates
tags:
- recon
- lead-ds
- kmp
- skill-format
- scenario-4
archived-at: 2026-07-31 17:27
---

# recon: формат навыка для `web-to-kmp-screen-port` (Сц.4)

status: draft · роль: explorer · репо: `~/tacticum/tacticum-dev/templates`
Вход для implementer'а (скелет навыка в worktree). Код НЕ правился — карта.

## TL;DR (что важно решить лиду)
- Навык = папка `ingredients/skills/<kebab-name>/SKILL.md` + запись `kind: skill_spec` в `manifest.yaml` пакета. Frontmatter навыка: **только `name` + `description`** (в description — trigger-фразы, вкл. русские). Всё остальное (tier/supports/paths/description_trigger) — в манифесте, НЕ в теле навыка. `depends_on`/`allowed_tools` в теле навыка НЕТ.
- **Два навыка, на которые ссылается спек, в репо ОТСУТСТВУЮТ**: `android-to-kmp-porting` и `angular-ds-component-usage`. Спек ошибочно считает их существующими. Риск: не ссылаться на мёртвые навыки.
- **Дом нового навыка = `iva-kmp-development-base`** (owner KMP-стек-навыков). `iva-kmp-brownfield` — это байт-в-байт зеркало (депрекируется по ADR-0059). Роль-пресет `iva-role-kmp` подхватит навык АВТОМАТически через `depends_on` — его манифест править НЕ нужно.
- Mapping-таблицы (Angular→Compose, code-bindings) оформляются как **markdown-таблицы в теле SKILL.md**. Отдельного yaml/json для code-bindings в навыках нет; резолв токенов — рантайм через `tacticum-mcp` (`design_resolve_token`).

## 1. Формат навыка внутри пакета

Расположение (owner-пакет): `templates/iva-kmp-development-base/ingredients/skills/<name>/SKILL.md`
Опционально рядом: `references/*.md` (deep-dive, если тело > ~150 строк).

Frontmatter тела (контракт из `local-skill-authoring`, подтверждён на реальных навыках):
```markdown
---
name: <kebab-case>            # = имя папки
description: >
  Use when <зона>. Triggers on "<фраза>", "<phrase>", "<русский триггер>".
---
# <Title>
<императивный текст для агента, таблицы > прозы, реальные пути/команды>
```
Правила: `name` = имя папки (kebab); `description` ≤ 1024 симв и ОБЯЗАН содержать trigger-фразы (агент маршрутизирует по ним); тело ≤ ~150 строк (overflow → `references/*.md`).

Пример существующего навыка целиком по структуре:
`templates/iva-kmp-brownfield/ingredients/skills/compose-multiplatform-ui/SKILL.md`
frontmatter: `name: compose-multiplatform-ui`, `description:` с триггерами ("Composable","Compose","экран","компонент"). Тело: разделы «Design system — Iva», «Non-negotiable Compose MP rules», «Stability/recomposition». Ссылается на репо-локальный `AI common/skills/compose-ui-patterns-*/SKILL.md`.

### Подключение к профилю — запись в `manifest.yaml`
Схема v2 (ADR-0023), блок `ingredients:`, `kind: skill_spec`. Реальный шаблон записи (из `iva-kmp-brownfield/manifest.yaml`):
```yaml
  - ingredient_id: compose-multiplatform-ui
    kind: skill_spec
    tier: full                 # trial | full
    supports: [claude-code, codex, copilot]
    install_scope: user
    target_path_template: ".claude/skills/{ingredient_id}/SKILL.md"
    copilot_target_path: ".github/skills/{ingredient_id}/SKILL.md"
    codex_target_path: ".agents/skills/{ingredient_id}/SKILL.md"
    body_path: ingredients/skills/compose-multiplatform-ui/SKILL.md
    metadata:
      description_trigger: "<короткая строка-триггер для каталога>"
```
Примечание: в `iva-kmp-development-base` записи чуть уже — `supports: [claude-code, codex]` (без copilot) и без `copilot_target_path`. Копировать стиль ЦЕЛЕВОГО пакета.

`skill_spec` по `_schema/ingredient.v1.schema.json`: `metadata.description_trigger` — **required**; опционально `assets: []`, `scripts: []`. Никаких `depends_on`/`allowed_tools` у skill_spec нет.

## 2. Референсные навыки, на которые Сц.4 ссылается

| Навык | В репо | Путь (owner) | О чём |
|---|---|---|---|
| `compose-multiplatform-ui` | ЕСТЬ | `iva-kmp-development-base/ingredients/skills/compose-multiplatform-ui/SKILL.md` | целевые Compose-идиомы: Screen/Content, IvaTheme/AppColors, стабильность, Decompose. Новый навык ССЫЛАЕТСЯ. |
| `design-system-discovery` | ЕСТЬ | owner `tacticum-ui-base/ingredients/skills/design-system-discovery/SKILL.md` (+зеркала в kmp/web/rn/mail-brownfield, analysis) | design-фаза: `design_list_systems`/`design_get_theme_tokens` через `tacticum-mcp`; токены DTCG, Figma «вне охвата». |
| `design-token-usage` | ЕСТЬ | owner `tacticum-ui-base/.../design-token-usage/SKILL.md` | impl-фаза: `design_resolve_token` → биндинг в `AppColors`/`IvaTheme`, не хардкодить hex/px. |
| `ui-mockup-match` | ЕСТЬ | owner `tacticum-ui-base/.../ui-mockup-match/SKILL.md` | пост-impl цикл (≤3 итер): snapshot(playwright) ↔ HTML-mockup diff → PIN-patch. **Только HTML-mockup, Figma/PNG вне охвата.** |
| `android-to-kmp-porting` | **НЕТ** | — (find по всему репо пусто) | спек считает существующим — ЭТО НЕВЕРНО. Такого навыка нет ни в одном пакете. |
| `angular-ds-component-usage` | **НЕТ** | — | тоже нет. Web-DS-правила размазаны по `ngrx-signals-state`/`surgical-change-discipline` + общие DS-навыки. |

Владелец UI-навыков — лейн `tacticum-ui-base` (UI-способность). KMP-стек-навыки — лейн `iva-kmp-development-base`. Реестр зеркал: `templates/_mirrors.yaml`.

## 3. Где физически жить новому навыку `web-to-kmp-screen-port`

Навык KMP-стек-специфичный (Compose/Decompose/`feature/<name>/impl/commonMain`/`Iva*`-гардрейлы) → **owner = `iva-kmp-development-base`**:
- создать тело: `templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-screen-port/SKILL.md`
- добавить `kind: skill_spec` запись в `templates/iva-kmp-development-base/manifest.yaml` (стиль записей этого пакета: supports `[claude-code, codex]`, без copilot).
- bump `version` + запись в `templates/iva-kmp-development-base/CHANGELOG.md`.

Подключение в KMP-профиль: **править `iva-role-kmp/manifest.yaml` НЕ нужно** — роль-пресет тянет содержимое из лейнов через `depends_on: [tacticum-core-base, tacticum-development-core, iva-kmp-development-base, tacticum-bugfix-base, tacticum-ui-base]` (ADR-0056/0057/0059). Навык из `iva-kmp-development-base` приезжает автоматически; DS/токен/mockup-навыки — из `tacticum-ui-base`.

Зеркало `iva-kmp-brownfield` (legacy, депрекируется): mirror-контракт (`_mirrors.yaml`) требует байт-в-байт для УЖЕ задекларированных ингредиентов; НОВЫЙ ингредиент можно оставить owner-only. Дублировать в brownfield — на усмотрение лида (профиль замораживается по гейтам ADR-0059 §8). Рекомендация разведки: owner-only.

## 4. Формат таблиц соответствия / code-bindings
- Маппинг-таблицы = **markdown-таблицы прямо в теле SKILL.md**. Примеры формата:
  - `design-system-discovery/SKILL.md` — таблица «Tool | Purpose | Output».
  - `ui-mockup-match/SKILL.md` — таблица «Field | Required | Source».
  - Целевая таблица Angular→Compose из спека (Сц.4, строки 68-84) кладётся так же — markdown-таблица в теле нового навыка.
- Отдельного yaml/json «словаря code-bindings» внутри навыков НЕ найдено (grep по всем `skills` пуст, кроме несвязанного go-навыка). Резолв «токен → значение» — рантайм через `tacticum-mcp` (`design_resolve_token`/`design_get_theme_tokens`), не файл. Словарь `Iva*`↔веб-мастер-компонент (Сц.4 п.2 открытых вопросов) как артефакта в репо ПОКА НЕТ.

## 5. `_schema` и `README.md`
- `templates/_schema/`: `manifest.v2.schema.json` (профиль, ADR-0023), `ingredient.v1.schema.json` (ингредиент, ADR-0020, 9 kind'ов), `manifest.v1.schema.json` (старьё).
- Для навыка релевантен `ingredient.v1.schema.json` → ветка `kind: skill_spec`: `metadata.description_trigger` required; опц. `assets`/`scripts`. `supports` ∈ {claude-code,codex,opencode,gemini,copilot}.
- `templates/README.md`: структура пакета (`manifest.yaml` + `ingredients/` + `tests/`). Процесс добавления — `CONTRIBUTING.md`. CI-сверка зеркал: `scripts/check_mirror_sync.py` (workflow `profile-version-discipline.yml`) + `tests/catalog/test_role_replacement_parity.py`.

## Риски / замечания для implementer'а
1. **Не ссылаться на `android-to-kmp-porting` и `angular-ds-component-usage`** как на существующие навыки — их нет. Либо описать поведение внутри нового навыка, либо (для DS-usage) ссылаться на `compose-multiplatform-ui`.
2. Frontmatter тела навыка держать МИНИМАЛЬНЫМ (`name`+`description`); всё остальное — в манифесте. Частая ошибка — тащить tier/supports в тело.
3. Стиль `skill_spec`-записи копировать из ЦЕЛЕВОГО пакета `iva-kmp-development-base` (там нет copilot-полей), не из brownfield.
4. `ui-mockup-match`/`design-system-discovery` сейчас Figma-агностичны (HTML-mockup, DTCG). Трёхсторонний паритет и Figma-мост из спека — это РАСШИРЕНИЕ тех навыков (owner `tacticum-ui-base`), отдельная задача; новый навык на них лишь ссылается.
5. Тело ≤ ~150 строк — большие процедуры чтения источника/верификации выносить в `references/*.md`.