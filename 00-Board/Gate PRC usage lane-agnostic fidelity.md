---
title: Gate PRC usage lane-agnostic fidelity
type: report
permalink: tacticum/00-board/gate-prc-usage-lane-agnostic-fidelity
status: draft
verdict: FIDELITY-PASS
tags:
- gate
- fidelity
- ds-web-axis1
- skills
- verifier
---

# Gate PRC usage lane-agnostic fidelity

**Вердикт: FIDELITY-PASS** (1 nit, не блокер)

## Объект
- Дерево `/Users/bubblemac/tacticum/tacticum-dev-web-axis1`, ветка `feat/ds-web-axis1`, коммит `789dbde`.
- Переформулировка acceptance-строк навыка `angular-ds-component-usage` на LANE-AGNOSTIC форму (решение ГД, вариант «а»).
- `git show --stat`: 2 файла (обе копии usage: `iva-web-brownfield` + `iva-web-development-base`), 18 insertions / 14 deletions. Обе копии по-прежнему бит-в-бит идентичны (`diff` → IDENTICAL; общий blob `2d311a0`).

## Изменённые места (ровно 3, переформулировка unconditional → conditional)
1. Таблица навыков, строка `ui-mockup-match` (L37): было «It matches against HTML … The numeric Figma comparison … is its Figma numeric-compare mode — the acceptance path». Стало: «Use its **Figma numeric-compare mode** … **when the attached `ui-mockup-match` provides it**; otherwise fall back to its **HTML mode / design review**».
2. `## Step 7 — Acceptance`, буллет «Visual match to the frame» (L156-161): та же условная развилка (Figma numeric-compare mode когда профиль даёт, иначе HTML mode / design review).
3. `## Anti-patterns / guardrails`, буллет «No pixel matcher here» (L173-176): «Numeric Figma comparison is `ui-mockup-match`'s **Figma numeric-compare mode** when the attached profile provides it (else its **HTML mode / design review**) — this skill only references it, **never implements it**».

## Проверки (по существу)

**1. Пробел ТЗ Сц.2 ш7 остаётся закрытым — ДА.**
Секция `## Step 7 — Acceptance` сохранена, acceptance-путь макета присутствует и теперь условный: Figma numeric-compare mode ИЛИ HTML mode / design review. Шаг приёмки не сломан, путь есть в обоих лейнах.

**2. Достоверность ссылок (сверено с реальными файлами worktree + origin/main) — ПРАВДИВО.**
- `templates/iva-web-brownfield/.../ui-mockup-match/SKILL.md`: секция `## Figma numeric-compare mode (Scenario 2, step 7 — gap G5)` РЕАЛЬНО есть — worktree L164 (+ L8/L31/L47/L62), и на **origin/main** (grep → L8/L31/L47/L164, rc=0). Условная ссылка правдива для brownfield.
- `templates/tacticum-ui-base/.../ui-mockup-match/SKILL.md`: секции Figma numeric-compare mode НЕТ; L27 «**HTML mockups only** for this version. Figma URLs and PNG exports are out of [scope]», L135 «No pixel SSIM / pixel-diff». То же на origin/main. Условность («иначе HTML mode / design review») оправдана для dev-base.
- dev-base лейн **не имеет собственной копии** ui-mockup-match — использует из `tacticum-ui-base` (HTML-only). Lane-mapping в правке корректен.
- Термин «HTML mode» реален (brownfield ui-mockup-match L30 «HTML mode (original)»). Числовые атрибуты (ΔE, pixel-diff, size-deltas, tolerance) реально описаны в секции — не выдуманы.

**3. 0 сверх-ТЗ — ДА.**
Ровно 3 места, чистая переформулировка; новой функциональности/требований/полей/режимов не добавлено. Доктрина навыка не расширена. Обе копии идентичны (single-source сохранён).

**4. Консистентность с sibling — ДА.**
`angular-ds-component-authoring` (brownfield) использует тот же термин: L134 «`ui-mockup-match` Figma numeric-compare mode», L190 «numeric-compare mode». Терминологического дрейфа нет.

**5. Guardrail не перевёрнут — ДА (даже усилен).**
«usage только ссылается на ui-mockup-match, не реализует matcher» сохранён: «this skill only references it, **never implements it**» + «do not build a numeric matcher in this skill».

## Nit (не блокер)
`tacticum-ui-base` ui-mockup-match не именует себя «HTML mode» (у него единственный безымянный режим «HTML mockups only»); термин «HTML mode» — это именование из brownfield-копии. В dev-base-копии usage фраза «HTML mode / design review» строго-формально применяет brownfield-термин к безымянному режиму dev-base. Это НЕ over-claim (скорее под-claim), дизъюнкция «/ design review» делает формулировку безопасной, и обе копии usage намеренно идентичны (single-source doctrine). Косметика, не fidelity-нарушение.

## Сверял (команды)
- `git show 789dbde` (полный diff), `git show --stat`, `diff` двух копий usage → IDENTICAL.
- `grep -n -i "figma numeric-compare mode|HTML mode|HTML mockups only|ΔE|pixel-diff"` по обоим ui-mockup-match SKILL.md.
- `git show origin/main:…/iva-web-brownfield/…/ui-mockup-match/SKILL.md` и `…/tacticum-ui-base/…` (сверка на main).
- `grep` sibling `angular-ds-component-authoring/SKILL.md`.
- `sed -n '148,180p'` отредактированного usage (Step 7 + guardrails).

Подлинность доказательства — на аудит контролёру.