---
title: impl-g7-dsdiscovery
type: report
permalink: tacticum/00-board/impl-g7-dsdiscovery
status: draft
role: implementer
for: lead-ds
tz: figma-ds ТЗ#1, PR-C = ось-1, gap G7 (узко)
repo: worktree ~/tacticum/tacticum-dev-web-axis1 (ветка feat/ds-web-axis1)
autonomy: false
tags:
- figma-ds
- pr-c
- axis1
- g7
- implementer
- ds-web
---

# G7 — фикс design-system-discovery (веб-копия, ось-1) — реализовано

Узкий фикс строго по ТЗ ось-1 п.1+п.2. НЕ commit, НЕ manifest/CHANGELOG. Тронут ровно ОДИН файл.

## Файл
`templates/iva-web-brownfield/ingredients/skills/design-system-discovery/SKILL.md` (хэш был 8358daba)

## Что изменено (2 правки, 1 файл, +20/-7)

### Правка 1 — таблица Tools: `design_list_systems` output
**До:** `[{design_system_id, version, name, modes[]}]`
**После:** `[{design_system_id, version, name, modes[], platform, framework_hint}]` (+ формулировка «and selection hints»).
Причина: скилл теперь ЧИТАЕТ `platform`/`framework_hint`; они должны быть видимы в контракте output (сервер их уже отдаёт — ADR-0026, ось-1 §«что уже умеет сервер»).

### Правка 2 — шаг 2 «When to call»: снят хардкод + добавлен surface-split
**До (баг из ТЗ, бывшая стр.44):**
```
2. If the feature is UI-bearing, call `design_get_tokens(design_system_id)` ...
   Default selection rules:
  - `iva-web` for web and Electron/desktop surfaces
  - another explicitly attached compatible DS may override the default
  Do not block Electron/desktop mockups ... the browser and desktop shells share the web token base.
```
**После (выбор ДС, без хардкода):**
- выбор из `design_list_systems` по **`platform`/`framework_hint`** (сервер отдаёт) против стека целевого репо;
- **идентичность репо** — `package.json name` (уже резолвится workflow на Run Discovery);
- опциональный **`default_design_system_id`** из `.tacticum/context.yaml` (явный per-repo override, когда ключ присутствует);
- **>1 ДС подходит → спросить пользователя** (ADR-0026, не угадывать);
- `design_get_tokens(design_system_id, installation_id=<id from .tacticum/context.yaml>)` — **installation_id проброшен явно** (урок PR-A/B);
- Electron/desktop-нота сохранена, но переформулирована на «shells of the same repo» (без unified-обобщения между репозиториями).

**Surface-split router-note (ось-1 п.2):** основной UI iva-one → `@iva/design-system`; конференц/MCU/iva-connect-поверхность (`calls`/`mcu`/`conference` libs, `from 'iva-core'`) → отдельная ДС **`iva-core`** через companion iva-core skill, НЕ дефолтная веб-ДС репо. Маршрутизация по поверхности ВНУТРИ репо.

## Границы соблюдены
- ⛔ НЕ трогал `tacticum-ui-base` и 6 других копий design-system-discovery (mirror-пара analysis↔fr-analyst, rn/mail-дрейф — вне scope, решение ГД).
- ⛔ НЕ трогал manifest.yaml / CHANGELOG.md / `_mirrors.yaml` / `design-systems/iva-web/*.yaml` (консолидация/seed — лид/DS-команда).
- НЕ вводил «unified surfaces»-допущение — наоборот, ось-1 его снимает; surface-split явный.
- НЕ дублировал `design-token-usage` (шаг 2 остаётся design-фазой: выбор ДС + токены, не резолв компонентов).
- API реален: `design_list_systems`/`platform`/`framework_hint`/`design_system_id`/`installation_id` — из ТЗ ось-1 §«сервер».

## Предлагаемый апдейт description_trigger (на решение лида, НЕ внёс)
Текущие триггеры покрывают DS/токены/тему. Если PR-C хочет, чтобы скилл поднимался и на конференц-поверхности, можно ДОБАВить в `description` триггер-слова: `"iva-core"`, `"conference"`, `"MCU"`, `"iva-connect"`. Оставил на усмотрение лида — правка frontmatter выходит за узкий body-фикс, не вносил молча.

## Самопроверка
- ✅ Тронут только `iva-web-brownfield/.../design-system-discovery/SKILL.md` (`git diff --stat` = 1 файл, +20/-7).
- ✅ Хардкод `iva-web for web and Electron` удалён; выбор по platform/framework_hint + repo identity + default_design_system_id.
- ✅ >1 ДС → спросить пользователя (ADR-0026).
- ✅ `installation_id` проброшен на `design_get_tokens` явно.
- ✅ Surface-split router-note присутствует (iva-one main UI vs iva-core конференц).
- ✅ НЕ commit, НЕ manifest, НЕ CHANGELOG.

## Замечание для лида
В worktree присутствует **untracked** папка `templates/iva-web-brownfield/ingredients/skills/iva-core-design-system/` — я её НЕ создавал и НЕ трогал (вероятно companion iva-core skill от параллельного воркера/лида). Мой router-note ссылается на неё как «companion iva-core skill». Если её ещё нет в плане — подсветить.

## Overlap с PR-B (напоминание из карты)
PR-B (ui-mockup-match) и мой G7 — разные папки скиллов, конфликта тел нет. Пересечение только по `CHANGELOG.md`/`manifest.yaml` iva-web-brownfield — их я НЕ трогал (консолидирует лид). Рекомендация карты: коммитить G7 после закоммиченного PR-B.