---
title: gate-prb-fidelity
type: report
permalink: tacticum/00-board/gate-prb-fidelity
tags:
- controller
- gate
- PR-B
- G5
- fidelity
---

# Гейт controller — FIDELITY PR-B (G5) · ui-mockup-match Figma numeric-compare

**Вердикт: FIDELITY-PASS** (G5 по ТЗ Сц.2 ш7, аддитивно, числа реальны, без отсебятины).

Объект: `templates/iva-web-brownfield/ingredients/skills/ui-mockup-match/SKILL.md`
Worktree: `/Users/bubblemac/tacticum/tacticum-dev-web-mockup`, ветка `feat/ds-web-mockup-figma`, коммит `00c6edb`, tree clean.
Сверка токенов: `~/tacticum/tacticum-dev/design-systems/iva-web/tokens.json` (jq).

## Пункты

1. **G5/ТЗ ш7 покрыт — PASS.** Есть числовое сравнение витрина↔Figma: ΔE (CIEDE2000) по именованным цвет-токенам; signed px size-deltas по `radius.*/padding.*/gap.*/fontSize.*`; token-conformance; обязательный допуск (ΔE ≤ 2.0, ±2px, tunable из PIN). Заякорено на именованных токенах, НЕ слепой pixel-SSIM.

2. **Аддитивно — PASS.** HTML-режим цел (§Inputs/§Tools/§Loop/§1–5 на месте), добавлена таблица «Two modes» + отдельная Figma-секция. Diff SKILL.md +146/−8: из 8 удалённых — старое ограничение «HTML mockups only, Figma out of scope» (снято обоснованно) и анти-паттерн «No pixel SSIM» — НЕ удалён, а **переформулирован корректно**: запрет слепого whole-image pixel-diff держится в ОБОИХ режимах, token-anchored ΔE-числа разведены с blind pixel. Два режима явно разделены («they do not mix»).

3. **installation_id — PASS.** На `design_*`-вызовах Figma-режима installation_id обязателен (Tools, Inputs-таблица, Metrics, CHANGELOG). Урок PR-A учтён.

4. **Токен-якоря РЕАЛЬНЫ — PASS.** Сверено jq, совпадает точь-в-точь:
   - `radius.control-m` = 10 ✓
   - `padding.content-area-sidebar` = 16 ✓
   - `gap.mail-chips` = 6 ✓
   - `bg.primary` → `$type:color`, `{light:"{solid.gray.100}", dark:"{solid.steel.1500}"}` ✓ (алиас-цепочка и per-mode структура — как в навыке)
   - `solid.gray.100`, `fontSize`, цвет-токены `$type:color` с {light,dark} — существуют ✓
   Выдуманных токенов/метрик нет. Честные зависимости присутствуют («no token anchor / unanchored», «не выдумывать expected», ΔE в активной теме).

5. **0 сверх-ТЗ — PASS.** Три метрики + допуск = ровно то, что требует ТЗ ш4/7. Token-conformance не разрастание: покрывает формальный критерий ТЗ «только именованные токены, hex/px = не прошёл» числами. Manifest bump 0.3.0→0.4.0 + description_trigger + CHANGELOG — по задаче.

## Гит-чистота — PASS
Ветка явная, не main; 1 коммит ahead; tree clean; в diff только 3 файла (SKILL.md, manifest.yaml, CHANGELOG.md) — секретов/`.env`/мусора нет. Тело коммита однострочное, **без AI-подписей**.

## Нюанс (не дефект, к сведению)
ТЗ дословно перечисляет «pixel-diff + ΔE + расхождения размеров». Реализация осознанно заменяет blind pixel-diff на token-anchored ΔE + token-conformance и документирует почему (антиалиасинг/cross-OS ложные срабатывания). Это соответствует рамке приёмки G5 («НЕ слепой pixel-SSIM») и одобрено картой — расценено как корректная переформулировка, а не отклонение.
