---
title: gate-prc-final-fidelity
type: report
permalink: tacticum/00-board/gate-prc-final-fidelity
status: draft
tags:
- fidelity
- gate
- ds-web
- axis1
- verifier
---

# Fidelity-гейт: authoring lane-agnostic (коммит 2d919de)

status: draft
verifier: read-only, вердикт (не правил код)

## Объект
- Дерево: `/Users/bubblemac/tacticum/tacticum-dev-web-axis1`, ветка `feat/ds-web-axis1`
- HEAD `2d919de` поверх `789dbde` (usage), baseline перед правкой = `290f448`/`789dbde`
- Правка: `angular-ds-component-authoring/SKILL.md` — 2 места, 14 строк на каждую из 2 байт-идентичных копий (brownfield + dev-base). Обе копии подтверждены `diff -q` → IDENTICAL.

## Вердикт: **FIDELITY-PASS**

### 1. Достоверность ссылок — PASS (сверено чтением)
- Brownfield `ui-mockup-match/SKILL.md`: секция **«Figma numeric-compare mode»** РЕАЛЬНО есть — выделенная секция стр.164 (Scenario 2 step 7, gap G5), с ΔE (CIEDE2000, стр.194-199), px size-deltas (стр.205-210), tolerance ΔE≤2.0 / ±2px (стр.222-224). Термин и режим не выдуманы.
- ui-base `ui-mockup-match/SKILL.md`: Figma-mode **НЕТ** — стр.27 «**HTML mockups only** for this version. Figma URLs and PNG exports are out of scope». HTML-mode эмитит structured delta list at semantic-token + DOM granularity, NOT pixels (стр.6, 46, 75).
- → Условность в authoring («Figma numeric-compare mode когда профиль даёт, иначе HTML mode / design review») оправдана фактами обоих лейнов. Выдуманных полей/режимов нет.

### 2. Zero сверх-ТЗ — PASS
Diff трогает ровно 2 места: §Acceptance (стр.129-137) и §Related skills (стр.187-192). Новой функциональности/требований не добавлено, доктрина не расширена. Формулировка «do not implement pixel/ΔE matching» усилена, не ослаблена.

### 3. Консистентность usage↔authoring — PASS
- usage (789dbde) и authoring (2d919de) теперь используют ОДИН термин **«Figma numeric-compare mode»** и ОДНУ условную формулу: «...when the attached profile provides it; otherwise fall back to its HTML mode / design review».
- Прежняя асимметрия (usage условный ↔ authoring безусловный «This numeric Figma comparison is the ...mode») **снята**. Терминологического/stance-дрейфа в §Acceptance и §Related skills между навыками больше нет.

### 4. Guardrail цел — PASS
- §Acceptance стр.136-137: «Reference it as the acceptance path; do **not** implement pixel/ΔE matching here».
- §Related skills стр.190-192: `ui-mockup-match` — «acceptance ... reference».
- authoring только ссылается на ui-mockup-match, matcher не реализует. Guardrail сохранён.

## ★ Отдельный вердикт по §Migration (Scenario 3, стр.160-169)

Точная строка (обе копии идентичны):
> Each batch is verified like **Scenario 2** (`ui-mockup-match` numeric compare, or design review when there is no mockup).

**Вердикт: ЗАЩИТИМО (defensible), НЕ over-claim. Верно в ОБОИХ лейнах.**

Обоснование:
1. Строка НЕ говорит «Figma numeric-compare mode» — только generic «numeric compare». Брендированный Figma-режим здесь не заявлен, поэтому недоступность его в dev-base не делает строку ложной.
2. Условие фолбэка («or design review when there is no mockup») корректно для обоих лейнов — при отсутствии мокапа уход в design review истинен везде.
3. «numeric compare» фактически истинно и для HTML-mode ui-base: HTML-mode эмитит token-anchored числовые дельты (напр. стр.112 «padding — mockup expects spacing.md (16px), runtime has 12px», hex-дельты стр.84). Это численное сравнение против мокапа, просто без ΔE/pixel-diff против Figma-фрейма.
4. Строка НЕ входит в scope коммита 2d919de (правка только §Acceptance + §Related skills) — она pre-existing и остаётся корректной.

**Caveat (не блокер, стилистика):** §Migration — самое «мягкое» место: его условность привязана к оси «есть/нет мокап», а не «профиль даёт/не даёт Figma-mode» как в двух приведённых к симметрии местах. Термин ui-mockup-match закрепляет слово «numeric» за Figma-режимом. Полной терминологической симметрии со §Acceptance/§Related нет. При желании — тонкий follow-up (условить так же), но по существу строка правдива в обоих лейнах и FAIL по ней не ставлю.

## Сверенные файлы/строки
- `templates/iva-web-brownfield/.../angular-ds-component-authoring/SKILL.md` стр.129-137, 160-169, 187-192
- `templates/iva-web-development-base/.../angular-ds-component-authoring/SKILL.md` (идентична)
- `templates/iva-web-brownfield/.../ui-mockup-match/SKILL.md` стр.8-10, 31, 34, 164, 194-224
- `templates/tacticum-ui-base/.../ui-mockup-match/SKILL.md` стр.6-7, 27, 46, 75-90, 112
- usage-симметрия: коммит `789dbde` (angular-ds-component-usage, обе копии)

## Итог
FIDELITY-PASS по коммиту 2d919de. §Migration-строка — защитима, истинна в обоих лейнах (не over-claim); остаточная стилистическая асимметрия — опциональный follow-up, не блокер.
