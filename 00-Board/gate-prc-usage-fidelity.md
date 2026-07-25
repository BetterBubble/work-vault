---
title: gate-prc-usage-fidelity
type: note
permalink: tacticum/00-board/gate-prc-usage-fidelity
tags:
- gate
- controller
- ds-web
- fidelity
- usage
- pr-c
---

# Гейт controller — FIDELITY PR-C usage (Сц.2 ш7)

**Вердикт: FIDELITY-PASS**

Объект: `templates/iva-web-brownfield/ingredients/skills/angular-ds-component-usage/SKILL.md`
Коммит: `290f448` (ветка `feat/ds-web-axis1`, не main). Дерево чистое, `git diff HEAD` пуст — фикс закоммичен.
ТЗ: figma-ds-scenario-2-new-screen.md, шаг 7 (приёмка скриншот↔макет числами).

## 1. Пробел Сц.2 ш7 закрыт — PASS
Три места, где раньше было «not-yet-shipped / PR-B / gap G5», теперь называют режим ДОСТУПНЫМ (путь приёмки ш7):
- **Depends-on** (стр.37): «…is its **Figma numeric-compare mode** — the acceptance path for Step 7».
- **Step 7** (стр.156-161): «…is the **`ui-mockup-match` Figma numeric-compare mode** — reference it as the acceptance path».
- **Anti-patterns** (стр.175-176): «Numeric Figma comparison is `ui-mockup-match`'s Figma numeric-compare mode — this skill only references it».

Агент, читая §7, вызовет доступный matcher.

## Достоверность референса — PASS (не self-cert)
Проверено, что цель ссылки реальна: `templates/iva-web-brownfield/ingredients/skills/ui-mockup-match/SKILL.md` содержит секцию «## Figma numeric-compare mode (Scenario 2, step 7 — gap G5)» (стр.164+), заявлена в description (стр.8-10) и в таблице режимов (стр.31). G5-режим действительно в main → ссылка не на «фантом».

## 2. Консистентность с authoring — PASS
`angular-ds-component-authoring/SKILL.md` (стр.132-136): «This numeric Figma comparison is the **`ui-mockup-match` Figma numeric-compare mode** (showcase ↔ Figma) — reference it as the acceptance path, do **not** implement pixel/ΔE matching here». Related-skills (стр.189) — «via its Figma numeric-compare mode». Обе формулировки зовут G5-режим доступным, совпадают по смыслу (mode + acceptance path + «не реализуется здесь»).

## 3. 0 сверх-ТЗ / доктрина цела — PASS
- Guardrail «числовой matcher НЕ здесь, он в ui-mockup-match» сохранён (стр.37, 160, 175-176).
- installation_id (стр.48-53), токен-якоря, процедура резолва Step 3 (стр.86-104) — НЕ тронуты.
- Diff = ровно 3 хунка, только устранение устаревшей атрибуции. Хирургично, без разрастания.

## 4. Остаточная устарелость — PASS
Grep по usage: `not-yet-shipped|future ui-mockup|PR-B|gap G5|planned|will ship` → пусто. Единственный матч «does not yet ship it» в authoring (стр.16) относится к «компонент ещё не в коде» (Сц.1 gap-report), НЕ к ui-mockup-match — легитимно.

## Гит-чистота — PASS
Ветка `feat/ds-web-axis1` (не main); коммит по задаче; тело коммита без AI-подписей (проверено grep: claude/co-authored/generated/🤖 — пусто).

---
Итог: пробел Сц.2 ш7 закрыт, ui-mockup-match Figma numeric-compare доступен, консистентно с authoring, доктрина цела, без сверх-ТЗ. Правок не вносил (read-only).
