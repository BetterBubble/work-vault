---
title: 'Critic-ревью PR-B (ui-mockup-match Figma numeric-compare, G5) — с итогом правок'
type: note
status: resolved
permalink: tacticum/00-board/critic-prb
tags:
- board
- design-system
- lead-ds
- tz1
- critic
- sc12
---

# Critic-ревью PR-B (G5 — ui-mockup-match Figma numeric-compare)

**Ревизор:** critic-агент, 2026-07-24 (персистит lead-ds — критик без write-доступа). **Объект:** `iva-web-brownfield/ingredients/skills/ui-mockup-match/SKILL.md` (commit 00c6edb). **Итог:** образцово (аддитивность/anti-галлюцинация/токен-якоря/границы), **1 обязательная правка + 3 дешёвые** (внесены фикс-раундом).

## Сильные стороны
- Аддитивность безупречна: HTML-режим цел, Figma в отдельной секции, «two modes do not mix».
- Запрет слепого pixel не выброшен, а разведён принципом («neither mode does blind pixel; Figma adds numbers measured against resolved token»).
- Anti-галлюцинация сильная: node→token только из get_variable_defs (не code-bindings), «no anchor → unanchored, do not invent», ΔE требует активной темы.
- installation_id-дисциплина выдержана; токен-якоря сверены с tokens.json (реальны); границы идеальны (3 файла, другие 4 копии/зеркала/owner не тронуты, не в _mirrors → CI не сработает).

## Обязательная (ВНЕСЕНА)
1. **§Metrics цвет ΔE — снятие рантайм-цвета через `browser_evaluate` computed style, НЕ bbox-семпл скриншота** (bbox из get_metadata = координаты Figma, не рантайма; для text/icon/border вернёт фон). → исправлено на computed-style (как метрика размера), усиливает «не пиксель».

## Дешёвые (ВНЕСЕНЫ тем же коммитом)
2. Tolerance «uiMatch» — висячая ссылка → убрана/оперта на самодостаточное обоснование (анти-алиасинг→допуск).
3. Детект активной темы (light/dark по рантайму — класс/CSS-переменная).
4. CHANGELOG опечатка «veb»→«web».

## Сверх-ТЗ — НЕ обнаружено (метрики ΔE/size/token-conformance + допуск ровно по ш7).

## Связано
[[plan-tz-1-dizain-protsess-figma-kod-sts.4-perenos-form-one-kmp-lead-ds]] · `00-Board/gate-prb-git` · `00-Board/gate-prb-fidelity` · `00-Board/impl-prb-critic-fixes`
