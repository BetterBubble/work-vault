---
title: 'Critic-ревью PR-C (ось-1: G7 discovery + iva-core + G8 quickstart) — с итогом правок'
type: note
status: resolved
permalink: tacticum/00-board/critic-prc
tags:
- board
- design-system
- lead-ds
- tz1
- critic
- axis1
---

# Critic-ревью PR-C (ось-1)

**Ревизор:** critic-агент, 2026-07-24 (персистит lead-ds). **Объект:** design-system-discovery (G7 fix) + iva-core-design-system (new, 2 байт-идент. копии) + web figma quickstart (G8). **Итог:** блокеров нет, фундамент образцовый; **2 should-fix в G8** (внесены).

## Сильные стороны
- G7: хардкод iva-web снят чисто (выбор по platform/framework_hint + repo identity + default_design_system_id + мультиматч→спросить ADR-0026); «unified surfaces» НЕ введён; installation_id везде; surface-router → iva-core.
- iva-core честен: 3 факта разведены; серверная ДС+словарь в §Deferred «do not fake it»; каталог `(illustrative)`.
- G8 фактологичен: 49/32/fileKey/селекторы/поля/тулы сверены с tokens.json; делегирует навыкам.
- Coverage-копия iva-core **байт-идентична** (cmp/sha подтвердил).

## Should-fix до PR (G8, ВНЕСЕНЫ)
1. **Достоверность:** G8 ссылался на несуществующие поля словаря `.mdx`/`slots` → заменено на `source` (реальное поле; inputs inline; storybook=доп-док). Сверено с tokens.json.
2. **Связность/ре-хардкод:** G8 хардкодит `design_system_id="iva-web"` (литерал, который ось-1/G7 снимает) → добавлена строка в шапку о скоупе (веб-поверхность) + связь с design-system-discovery-роутером/iva-core.
3. (дешёвое) `[ivaMenuTriggerFor]`→`iva-menu`; 4. пометка что канон-алгоритм в навыке angular-ds-component-usage (не дублировать).

## Verify (подтверждено лидом): версии 0.2.0/0.3.0 — реальны (CHANGELOG code-bindings + design-system.yaml 0.3.0).

## Отложено (осознанно вне PR-C, не блокер)
- Канон tacticum-ui-base («unified»→surface-split) — отдельный ADR (ГД→президент).
- _mirrors для 2 копий iva-core (drift-риск) — отдельное решение.
- Серверная ДС iva-core + словарь — server/RE/DS-команда.
- VCS/VKS терминологический микс в iva-core — косметика на потом (не трогаем, чтобы не дрейфить 2 копии).

## Связано
[[plan-tz-1-dizain-protsess-figma-kod-sts.4-perenos-form-one-kmp-lead-ds]] · `00-Board/gate-prc-git` · `00-Board/gate-prc-fidelity` · `00-Board/impl-prc-g8-fixes`
