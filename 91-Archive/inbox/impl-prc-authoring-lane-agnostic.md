---
title: impl-prc-authoring-lane-agnostic
type: report
permalink: tacticum/00-board/impl-prc-authoring-lane-agnostic-1
status: draft
role: implementer
task: lane-agnostic формулировка Figma numeric-compare mode в angular-ds-component-authoring
  (симметрия с usage)
branch: feat/ds-web-axis1
commit: 2d919de
base_commit: 789dbde
tags:
- ds-web
- axis1
- implementer
- angular-ds-component-authoring
- lane-agnostic
archived-at: 2026-08-03 11:16
---

# impl: authoring — lane-agnostic Figma numeric-compare mode

Снят безусловный over-claim про доступность Figma numeric-compare mode в навыке `angular-ds-component-authoring` (обе байт-идентичные копии). Формулировка приведена к той же УСЛОВНОЙ форме, что уже применена в sibling `angular-ds-component-usage` (789dbde). Верно в обоих лейнах: brownfield (co-located ui-mockup-match c Figma-mode) и dev-base (ui-mockup-match HTML-only через tacticum-ui-base).

## Файлы (ровно 2, правлены одинаково)
- `templates/iva-web-brownfield/ingredients/skills/angular-ds-component-authoring/SKILL.md`
- `templates/iva-web-development-base/ingredients/skills/angular-ds-component-authoring/SKILL.md`

Diff в обеих копиях идентичен: `index 89fb4fd..6b04d87`, +16/-12 суммарно (по 2 hunk на файл).

## Дельты (было → стало), одинаково в обеих копиях

### Место 1 — секция `## Acceptance (task is done when)`, буллет «Showcase ↔ Figma» (~стр.132-136)
БЫЛО (безусловно «This numeric Figma comparison **is** the ui-mockup-match Figma numeric-compare mode … reference it as the acceptance path»):
```
- **Showcase ↔ Figma, by numbers** — the Storybook page is compared to the
  master component numerically (size/color deltas), not "by eye". This numeric
  Figma comparison is the **`ui-mockup-match` Figma numeric-compare mode**
  (showcase ↔ Figma) — reference it as the acceptance path, do **not**
  implement pixel/ΔE matching here.
```
СТАЛО (условно — «use its Figma numeric-compare mode **when the attached profile provides it**; otherwise fall back to its **HTML mode / design review**»; guardrail «do **not** implement pixel/ΔE matching here» сохранён дословно):
```
- **Showcase ↔ Figma** — the Storybook page is checked against the master
  component. Acceptance via `ui-mockup-match`: use its **Figma numeric-compare
  mode** (showcase ↔ Figma — numeric size/color/ΔE deltas within a tolerance)
  **when the attached profile provides it**; otherwise fall back to its **HTML
  mode / design review**. Reference it as the acceptance path; do **not**
  implement pixel/ΔE matching here.
```

### Место 2 — секция `## Related skills`, буллет `ui-mockup-match` (~стр.189-190)
БЫЛО (безусловно «showcase↔Figma numeric compare via its Figma numeric-compare mode»):
```
- `ui-mockup-match` — showcase↔Figma numeric compare via its Figma
  numeric-compare mode.
```
СТАЛО (условно, «else HTML mode / design review» — формула из anti-patterns usage):
```
- `ui-mockup-match` — acceptance for the authored component: its **Figma
  numeric-compare mode** (showcase ↔ Figma) when the attached profile provides
  it, else its **HTML mode / design review**.
```

## Совпадение термина/формулы с usage (эталон 789dbde)
Использована ровно та же условная формула, что в usage:
- Step 7 usage: «use its **Figma numeric-compare mode** … **when the attached profile provides it**; otherwise fall back to its **HTML mode / design review**. Reference it as the acceptance path; do **not** build a numeric matcher».
- anti-patterns usage: «… **Figma numeric-compare mode** when the attached profile provides it (else its **HTML mode / design review**)».
Термин «Figma numeric-compare mode» / «HTML mode / design review» — дословно совпадает.

## Самопроверка (durably)
- `cmp` двух копий authoring → **IDENTICAL** (пусто, exit 0).
- grep устаревшей атрибуции `not.yet|not-yet-shipped|PR-B|gap G5|until it lands|future` по обеим копиям → единственное совпадение стр.16 «the code does not yet ship it» (про отсутствие компонента в коде, НЕ про Figma-mode; легитимно, не трогалось). Маркеров `PR-B / gap G5 / until it lands / future / not-yet-shipped` — НЕТ.
- Многострочная проверка (perl -0777) в ОБЕИХ копиях: guardrail «do **not** implement pixel/ΔE matching here» → FOUND; условная формула Acceptance → ACCEPT-COND; условная формула Related skills → RELATED-COND.
- Валидатор `check_mirror_sync.py` → EXIT 0 («OK — 64 зеркальных ингредиентов в 6 парах синхронны»).
- Валидатор `check_profile_version_discipline.py --diff-against origin/main` → EXIT 0 («OK — 48 profile(s) clean»). Бамп версии не потребовался (правка тела навыка).

## Коммит
- Новый коммит `2d919de` поверх `789dbde` на `feat/ds-web-axis1` (append, без force, без push). `git add` только 2 файла authoring. Дерево после коммита чистое.
- Сообщение: «authoring: lane-agnostic формулировка Figma numeric-compare mode (обе копии) — симметрия с usage, снятие over-claim в dev-base-лейне». Без AI-подписей/футеров.

## Подтверждения гардрейлов
- Guardrail «do **not** implement pixel/ΔE matching here» (authoring только ссылается на ui-mockup-match) — ЦЕЛ, не превращён в «реализуй тут».
- Устаревшая атрибуция (not-yet-shipped / PR-B / gap G5 / until it lands / future) — отсутствует.
- Тронуты ТОЛЬКО 2 места про acceptance/Figma-mode. Секция `## Migration (Scenario 3)` (в т.ч. «ui-mockup-match numeric compare, or design review when there is no mockup», ~стр.167) и остальная доктрина (анатомия, директива vs элемент, IvaControlBase+CVA, слоты, токены-only, Completeness, Storybook) — НЕ тронуты.
- Sibling `angular-ds-component-usage` (789dbde) — НЕ тронут (только читался как эталон).
- Байт-идентичность двух копий authoring сохранена (cmp IDENTICAL).
- Термин/формула совпадают с usage дословно по смыслу.
- НЕ пушил, НЕ мержил, НЕ деплоил.

## На ревью тимлиду
Замечание вне плана (НЕ правил, докладываю): в секции `## Migration (Scenario 3)` стр.~167 остаётся «(ui-mockup-match numeric compare, or design review when there is no mockup)» — это тоже мягкий over-claim для dev-base (условие завязано на наличие мокапа, а не на наличие Figma-mode). По прямому гардрейлу задачи Migration-секцию не трогал. Если ГД захочет симметрию и там — отдельная микроправка.