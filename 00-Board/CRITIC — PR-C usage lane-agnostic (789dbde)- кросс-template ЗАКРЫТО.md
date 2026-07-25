---
title: 'CRITIC — PR-C usage lane-agnostic (789dbde): кросс-template ЗАКРЫТО'
type: note
permalink: tacticum/00-board/critic-pr-c-usage-lane-agnostic-789dbde-kross-template-zakryto
status: draft
tags:
- board
- design-system
- lead-ds
- tz1
- critic
- pr-c
---

# CRITIC — PR-C usage lane-agnostic (789dbde)

**Ревизор:** critic (read-only, тула записи на доску не имел → фиксирует lead-ds). **Дата:** 2026-07-25. **Объект:** `tacticum-dev-web-axis1`, ветка `feat/ds-web-axis1`, HEAD `789dbde` поверх `290f448`. Правка 2 байт-идентичных копий `angular-ds-component-usage/SKILL.md` (iva-web-brownfield + iva-web-development-base), 3 места.

## ВЕРДИКТ: замечание кросс-template ЗАКРЫТО, к пушу претензий нет

Премиса подтверждена фактами дерева:
- `tacticum-ui-base/.../ui-mockup-match/SKILL.md:27` — «HTML mockups only … Figma URLs and PNG exports out of scope» → в dev-base-лейне Figma numeric-compare mode физически НЕТ.
- `iva-web-brownfield/.../ui-mockup-match/SKILL.md:8,30-31,164` — Figma numeric-compare mode ЕСТЬ.
- Прежняя безусловная формулировка в dev-base usage = реальный over-claim/висячая ссылка. Новая условная («use Figma numeric-compare mode when the attached provides it; otherwise HTML mode / design review») верна в ОБОИХ лейнах. Висячей ссылки больше нет.

По пунктам: (1) over-claim снят во всех 3 местах (depends-on 37, Step 7 154-161, anti-patterns 175-178); (2) guardrail цел, даже усилен («only references it, never implements it»); (3) не пере-исправлено — acceptance-путь чёткий (numeric-mode ИЛИ HTML/design-review), пробел Сц.2 ш7 заново не открыт; (4) байт-идентичность (md5 `c40f25c6…`, diff пуст), sibling authoring термин единый, дрейфа нет; (5) устарелость (not-yet/PR-B/gap G5/future/shipped) не вернулась, правка узкая +18/−14, доктрина не тронута.

**Критично-до-пуша: НЕТ.**

## Остаточные флаги (НЕ этот PR)
- **[для ГД — stance-асимметрия] `angular-ds-component-authoring` (dev-base) делает БЕЗУСЛОВНОЕ claim Figma-mode** — тот же класс over-claim, что закрыт в usage. Фикс создал асимметрию (usage условный ↔ authoring безусловный). Критик относит к отложенному mirror-drift/ADR-риску (риск #1), рекомендует привести authoring к той же lane-agnostic формуле для симметрии, но «вне скоупа этого PR по решению ГД, не расширять». → lead-ds поднимает ГД как решение (симметричная правка authoring сейчас vs строго ADR).
- **[гигиена]** `iva-web-brownfield/.../ui-mockup-match/SKILL.md:236` — «## Iteration 1 — Figma numeric-compare (2026-07-24…)»: хроника-итерации в теле спеки (нарушение высоты «хроника в норме»). Отдельная гигиена, не блок.
- Nice-to-have (микро): разнобой условия — depends-on «when the attached ui-mockup-match provides it» vs Step 7/anti-patterns «when the attached profile provides it». Смысл один, единообразие опционально.

## Связано
`00-board/gate-prc-usage-lane-agnostic-fidelity` (FIDELITY-PASS) · `00-board/gate-prc-usage-git` · [[plan-tz-1-dizain-protsess-figma-kod-sts.4-perenos-form-one-kmp-lead-ds]]