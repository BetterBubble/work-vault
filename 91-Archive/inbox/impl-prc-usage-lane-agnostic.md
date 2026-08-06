---
title: impl-prc-usage-lane-agnostic
type: report
permalink: tacticum/00-board/impl-prc-usage-lane-agnostic-1
status: draft
role: implementer (lead-ds)
branch: feat/ds-web-axis1
commit: 789dbde34c954ac8774279513df6dcb3437e8382
prev_commit: 290f448
tags:
- ds-web
- axis1
- implementer
- skill-usage
archived-at: 2026-08-03 11:16
---

# usage: lane-agnostic формулировка Figma numeric-compare mode

Мелкая doc-правка в 2 байт-идентичных копиях навыка `angular-ds-component-usage`. Снят over-claim «Figma numeric-compare mode доступен/shipped» — заменён на условную (lane-agnostic) форму, верную в обоих template-лейнах: brownfield (co-located ui-mockup-match с Figma-mode) и dev-base (UI делегирован в tacticum-ui-base, ui-mockup-match HTML-only).

## Файлы (оба правлены ОДИНАКОВО)
- `templates/iva-web-brownfield/ingredients/skills/angular-ds-component-usage/SKILL.md`
- `templates/iva-web-development-base/ingredients/skills/angular-ds-component-usage/SKILL.md`

## Дельты — 3 места в каждой копии

### Место 1 — depends-on таблица (~стр.37)
- **Было:** «It matches against **HTML** mockups (…) in its HTML mode. The **numeric Figma comparison** (…) **is its Figma numeric-compare mode — the acceptance path for Step 7**. Reference it; do **not** implement pixel matching in this skill.»
- **Стало:** «**Use its Figma numeric-compare mode** (screenshot vs frame with pixel-diff / ΔE / size-deltas + tolerance) **when the attached `ui-mockup-match` provides it**; otherwise fall back to its **HTML mode / design review** (semantic-token + DOM, no pixels). Reference it; do **not** implement pixel matching in this skill.»

### Место 2 — Step 7 acceptance (~стр.156-161)
- **Было:** «The **numeric** comparison against the frame (…) **is the `ui-mockup-match` Figma numeric-compare mode** — reference it as the acceptance path, do **not** build a numeric matcher in this skill. Its HTML mode (semantic + DOM) covers the HTML mockup path.»
- **Стало:** «Acceptance via `ui-mockup-match`: **use its Figma numeric-compare mode** (numeric comparison against the frame — pixel-diff + ΔE + size-deltas within a tolerance) **when the attached profile provides it**; otherwise fall back to its **HTML mode / design review** (semantic + DOM). Reference it as the acceptance path; do **not** build a numeric matcher in this skill.»

### Место 3 — anti-patterns (~стр.175-176)
- **Было:** «Numeric Figma comparison **is `ui-mockup-match`'s Figma numeric-compare mode** — this skill only references it (step 7).»
- **Стало:** «Numeric Figma comparison is `ui-mockup-match`'s **Figma numeric-compare mode when the attached profile provides it (else its HTML mode / design review)** — this skill only references it, never implements it (step 7).»

## Самопроверка
- `cmp` двух файлов → **IDENTICAL** (пусто). Байт-идентичность сохранена.
- `grep -niE "not.yet|not-yet-shipped|PR-B|gap G5|until it lands|shipped|future"` → только легитимная строка 23 в обеих копиях («If an instance is **not yet** in the dictionary») — НЕ трогалась (другой смысл). Устаревшей атрибуции (not-yet-shipped/PR-B/gap G5/until it lands/shipped/future) НЕТ.
- `check_mirror_sync.py` → EXIT=0 (OK — 64 зеркальных ингредиента в 6 парах синхронны).
- `check_profile_version_discipline.py --diff-against origin/main` → EXIT=0 (OK — 48 profile(s) clean). Бамп версии НЕ требуется — подтверждено.

## Гарантии
- **Guardrail цел:** во всех 3 местах сохранён запрет реализовывать pixel/numeric matcher в самом навыке («do not implement pixel matching in this skill», «do not build a numeric matcher in this skill», «this skill only references it, never implements it»). Usage только ссылается на ui-mockup-match.
- **Sibling не тронут:** `angular-ds-component-authoring` не редактировался; терминология Figma-mode консистентна.
- **Тронуты только 3 места:** доктрина (Step 1-6, резолв словаря, токены) и всё прочее не изменены.
- **Байт-идентичность:** обе копии правлены одинаково, cmp IDENTICAL.

## Коммит
- Ветка `feat/ds-web-axis1`, новый коммит `789dbde` (append, без force/push/merge).
- `2 files changed, 18 insertions(+), 14 deletions(-)`.
- `git add` только 2 файла usage.
- НЕ пушено, НЕ мержено, НЕ деплой.