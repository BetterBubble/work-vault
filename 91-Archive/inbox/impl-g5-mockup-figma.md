---
title: impl-g5-mockup-figma
type: report
permalink: tacticum/00-board/impl-g5-mockup-figma-1
status: draft
role: implementer
for: lead-ds
tz: ТЗ#1 figma-ds, PR-B, gap G5, Сц.2 шаг 7
worktree: ~/tacticum/tacticum-dev-web-mockup (ветка feat/ds-web-mockup-figma)
commit: 00c6edbd42db24fd400fa328b54481fa694fedcb
date: 2026-07-24
tags:
- figma-ds
- impl
- sc2
- gap-g5
- pr-b
- ui-mockup-match
- lead-ds
archived-at: 2026-08-03 11:16
---

# impl G5 — ui-mockup-match Figma numeric-compare mode

Расширил навык `ui-mockup-match` (веб-копия) числовым Figma-режимом (Сц.2 шаг 7). Аддитивно: HTML-режим не тронут.

## Изменённые файлы (3)
- `templates/iva-web-brownfield/ingredients/skills/ui-mockup-match/SKILL.md` (+146/-8)
- `templates/iva-web-brownfield/manifest.yaml` (version 0.3.0→0.4.0 + description_trigger)
- `templates/iva-web-brownfield/CHANGELOG.md` (+32, секция [0.4.0])

`git diff --stat`: 3 files changed, 174 insertions(+), 8 deletions(-).

## Что добавлено (Figma numeric-compare mode)
1. **Секция «Two modes»** сверху — явное разделение: HTML mode (semantic+DOM, оригинал, не тронут) vs Figma numeric-compare mode. Переформулировано старое примечание «HTML mockups only» → теперь ограничивает HTML-режим и указывает на Figma-режим (был прямой конфликт с ТЗ, снят).
2. **Inputs (Figma mode):** `get_screenshot`/`get_metadata`/`get_variable_defs`(node-id) + `design_get_tokens(design_system_id="iva-web", installation_id=<из .tacticum/context.yaml>)`. installation_id помечен **обязательным на КАЖДОМ** `design_*`-вызове (урок PR-A, иначе `installation_id_required`).
3. **Метрики — числами, все заякорены на токены:**
   - (а) **цвет — ΔE (CIEDE2000)** между рендером региона элемента и резолвнутым значением именованного color-токена, который фрейм биндит (`bg.*`/`text.*`/`border.*`/`icon.*`, `$type: color`, per-mode light/dark). Якорь = токен, НЕ соседний пиксель.
   - (б) **размеры/отступы — signed px-deltas** vs числовые токены фрейма (`radius.*`/`padding.*`/`gap.*`/`fontSize.*`, все `$type: number`).
   - (в) **token-conformance** — биндит ли рантайм тот же именованный токен или сырой hex/px (Tier-1 дельта даже в допуске — дрейф при смене темы).
4. **Tolerance (обязателен):** ΔE ≤ 2.0 (JND) и ±2px по умолчанию, по образцу uiMatch (0% недостижим — антиалиасинг), тюнится из PIN.
5. **Выход:** числовой delta-report (тот же `mockup-match-report.md`, тот же 3-итерационный цикл и hand-off в `/run-coder`, что и HTML-режим).
6. **«Honest dependencies» (не выдумывать):** frame→token биндинг берётся из Figma `get_variable_defs`, а НЕ из code-bindings словаря (тот маппит instance→selector, не node→token); нет биндинга/slot-токена → репортить «no token anchor / unanchored», не выдумывать. ΔE требует активного mode (light/dark).
7. **Anti-pattern уточнён:** «no blind pixel SSIM» держится в ОБОИХ режимах; Figma ΔE = заякоренное на токене число в регионе элемента, НЕ сырой whole-frame pixel-compare.

## Сверка с реальностью (принцип президента, как PR-A)
Все токен-якоря проверены по `~/tacticum/tacticum-dev/design-systems/iva-web/tokens.json`:
- color-токены `bg/text/border/icon` = `$type: color`, `{light,dark}` с алиасами `{solid.*}` — подтверждено.
- `radius.control-m` = 10, `padding.content-area-sidebar` = 16, `gap.mail-chips` = 6, `radius.l` = 12, `padding.btn-panel-leftright` = 24 — реальные значения, использованы в примерах.
- code-bindings — `$extensions."dev.tacticum.code-bindings"` (instance→selector), честно помечено что node→token оттуда НЕ берётся.
Выдуманных полей/метрик нет. Единственная иллюстративная величина в примере отчёта (`#f5f5f6`) явно помечена «e.g.».

## Границы скоупа — подтверждение
- HTML-режим не сломан (все существующие секции §Inputs→§5 на месте, только добавлено разделение и новый раздел после §5).
- Другие 4 копии (kmp/rn/mail-brownfield, ui-base) НЕ тронуты — `git status` показывает только 3 файла iva-web-brownfield.
- `ui-mockup-match` НЕ в `_mirrors.yaml` (0 вхождений) → копии независимы, зеркала не задеты.
- Owner/роль/зеркала/другие пакеты не трогал. Сверх-ТЗ нет.

## Валидаторы (venv `apps/backend/.venv`, создан через `uv sync`, .venv в .gitignore — подтверждено `git check-ignore`)
- **mirror-sync** (`scripts/check_mirror_sync.py`): `OK — 64 зеркальных ингредиентов в 6 парах синхронны`. ui-mockup-match — 0 вхождений в `_mirrors.yaml`, зеркала не задеты.
- **version-discipline static** (`scripts/check_profile_version_discipline.py`): `OK — 48 profile(s) clean`.
- **version-discipline --diff-against origin/main** (после коммита): `OK — 48 profile(s) clean`.
- **pytest** `apps/backend/tests/catalog/test_manifest_schemas.py`: 38 passed, exit 0.

## Commit
`00c6edb feat(ds-web): ui-mockup-match Figma numeric-compare mode (Сц.2 шаг 7, gap G5)` — ветка `feat/ds-web-mockup-figma`, НЕ запушен. Без AI-подписей.

## Самопроверка
Аддитивно ✅ · installation_id на каждом design_* ✅ · числа заякорены на токены, не слепой pixel ✅ · сверх-ТЗ ничего ✅ · другие копии не тронуты ✅.
</content>
</invoke>