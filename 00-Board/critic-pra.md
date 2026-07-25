---
title: 'Critic-ревью PR-A (angular-ds-component-authoring + -usage) — с итогом правок'
type: note
status: resolved
permalink: tacticum/00-board/critic-pra
tags:
- board
- design-system
- lead-ds
- tz1
- critic
- sc12
---

# Critic-ревью PR-A (Сц.1/2 gap — 2 новых web-DS-навыка)

**Ревизор:** critic-агент, 2026-07-24 (не смог записать сам — персистит lead-ds; ТЗ-специи/карту не читал, т.к. они транзиентные — проверял связность+логику). **Объект:** `angular-ds-component-authoring` (G1) + `angular-ds-component-usage` (G4+G6). **Итог:** инструктивны/честны/дисциплинированы; **2 критичных до PR** (внесены фикс-раундом).

## Сильные стороны
- STOP-правила честные по-элементные (usage pre-flight→СТОП дизайнеру, не-в-словаре→СТОП; authoring: без published-мастера не авторить).
- Дисциплина не-дублирования: ссылаются на design-system-discovery/design-token-usage/ui-mockup-match, не переписывают.
- Граница G5 жёсткая (числовой matcher НЕ реализован, только ссылка).
- Anti-галлюцинация (bind only declared inputs, квота 200/день).

## Критичные до PR (ВНЕСЕНЫ фикс-раундом)
1. **usage: `design_get_tokens` без `installation_id`** — соседние навыки требуют его из `.tacticum/context.yaml` на каждом `design_*`-вызове (иначе `installation_id_required`). Дыра исполнимости. → добавлена оговорка.
2. **Шов authoring↔usage по биндингу словаря** — usage приписывал создание биндинга навыку authoring, а в authoring шага нет (и не должно: словарь=RE-конвейер G2/G3, вне скоупа). Противоречие. → usage указывает реального продюсера (словарный/RE-конвейер), authoring — короткий pointer.

## Верифицировано ЛИДОМ против tokens.json (РЕАЛЬНО, не выдумка — НЕ трогать)
- `#3` имя пакета **`@iva/design-system`** = реальная `codebase.library` словаря ✅.
- `#4` поля биндинга (name/match/figma_key/selector/kind/source/storybook/inputs) + правило нормализации «lowercase, strip spaces/dashes/underscores» — точь-в-точь из tokens.json ✅.

## Косmetика (внесена заодно)
- #5 единое имя forward-ref числового сравнения (PR-B/G5) в обоих навыках.
- #6 подрезка повторов в authoring.
- (fidelity-косметика) authoring→usage именная ссылка для симметрии.

## Связано
[[plan-tz-1-dizain-protsess-figma-kod-sts.4-perenos-form-one-kmp-lead-ds]] · `00-Board/gate-pra-git` · `00-Board/gate-pra-fidelity` · `00-Board/impl-pra-critic-fixes`
