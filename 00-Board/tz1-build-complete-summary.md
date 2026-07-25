---
title: 'ТЗ#1 BUILD-COMPLETE — сводка сделано vs ТЗ (figma-ds, template-предел)'
type: note
status: draft
permalink: tacticum/00-board/tz1-build-complete-summary
tags:
- board
- design-system
- lead-ds
- tz1
- build-complete
- acceptance
---

# ТЗ#1 (figma-ds) — BUILD-COMPLETE: сделано vs ТЗ

**Ведёт:** lead-ds. **Дата:** 2026-07-24. **Предел (утверждён президентом+ГД):** template-side capability = PR-A + PR-B + PR-C; всё за пределом = handoff (`handoff-tz1-deferred-remainder`). **Принцип:** строго по ТЗ Солонко, без раздувания.

## СТАТУС ДОСТАВКИ
| PR | Содержание | Статус |
|---|---|---|
| Сц.4 bundle (#149) | навык `web-to-kmp-screen-port` + `web-to-kmp-source-reference` + словарь Iva*↔web | ✅ в main |
| PR-A (#149) | `angular-ds-component-authoring` (G1) + `angular-ds-component-usage` (G4+G6) | ✅ в main (iva-web-brownfield 0.3.0) |
| CI-fix (#150) | врезка angular-ds-component-* в iva-web-development-base (покрытие роли) | ✅ в main (0.1.1) |
| PR-B (#154) | `ui-mockup-match` Figma numeric-compare mode (G5) | ✅ в main (0.4.0) |
| **PR-C** | ось-1: `design-system-discovery` framework/surface fix (G7) + `iva-core-design-system` скилл + web quickstart (G8) + Сц.3-правило + coverage | 🟢 ready, у ГД на гейте → push (0.5.0 / 0.1.2) |

## СВЕРКА ПО СЦЕНАРИЯМ/ОСЯМ ТЗ

**Сценарий 1 (наполнение библиотеки)** — ✅ template-часть закрыта: навык авторинга-конвенций web-компонента (`angular-ds-component-authoring`: анатомия, директива vs элемент, IvaControlBase+CVA, слоты, токены-only, полнота вариантов, Storybook+.mdx+spec). Отложено: G2/G3 (словарь v2 + авто-пересборка+CI) = RE.

**Сценарий 2 (новый экран по макету)** — ✅ template-часть закрыта: навык сборки из словаря (`angular-ds-component-usage`: pre-flight макета, резолв code-connect/словарь, не-в-словаре→стоп, сборка по selector, .mdx-first) + числовая приёмка (`ui-mockup-match` Figma numeric-mode, G5: ΔE/size/token-conformance, заякорено на токены). + web quickstart (G8). Отложено: 17 null figma_key + Figma-фреймы = дизайнеры.

**Сценарий 3 (миграция iva-one)** — 🟢 template-часть = ТОНКОЕ ПРАВИЛО (в `angular-ds-component-authoring §Migration`: 2 слоя батчами, не смешивать ДС, нет аналога→Сц.1, удаление легаси отдельно). Прогон миграции репо = команда iva-one (handoff).

**Сценарий 4 (перенос Angular→KMP)** — ✅ в main: навык-оркестратор `web-to-kmp-screen-port` (процедура чтения источника + Angular→Compose + маппинг состояния + гардрейлы + верификация + rewrite-vs-move) + reference-скилл двух деревьев + словарь Iva*↔web (resolved, 32 figma_key). Сухой пилот на ContactDetailScreen пройден (статприёмка); рантайм — у команды Легина (среда).

**Ось-1 (несколько ДС по поверхности)** — 🟢 в PR-C: `design-system-discovery` читает platform/framework_hint (снят хардкод iva-web) + surface-router (iva-one→@iva/design-system, конференц→iva-core) + тонкий скилл `iva-core-design-system`. Отложено: серверная ДС iva-core+словарь = server/RE; канон ui-base «unified» = ADR.

**Ось-2 (два репо: source read-only + target write)** — ✅ п.3 (reference-скилл) в main (Сц.4 bundle); п.1/2/4 (start task arg + workflow + гейт) — спека передана lead-modes, ретаргет на iva-*-brownfield (его лейн).

## КАЧЕСТВО/ДИСЦИПЛИНА
- Каждый юнит через батарею: git/controller + fidelity (по-ТЗ, 0 сверх-ТЗ) + critic. ~15 controller-гейтов суммарно.
- Достоверность: все figma_key/токен-якоря/поля словаря сверены с реальным `tokens.json`; выдуманного API нет (принцип президента).
- Серверы iva (adp/teststand) — read-only весь путь; разовое teststand-исключение (рантайм-пилот) закрыто teardown'ом.
- Урок регрессии PR-A (coverage-тест): зафиксирован — полный `pytest catalog/` в каждой батарее; стоп-предохранитель поймал coverage PR-C ДО коммита.
- 0 самовольных мержей/пушей (все через ГД+президента).

## ОТЛОЖЕНО (handoff/ADR — вне template-предела)
См. `handoff-tz1-deferred-remainder`: Сц.3-прогон=iva-one · iva-core-server+словарь=server/RE · G2/G3=RE · 17 null+фреймы=дизайнеры · канон ui-base + _mirrors DS-навыков=ADR/процесс.

## ВЕРДИКТ
**ТЗ#1 template-side = BUILD-COMPLETE** (по ТЗ Солонко, без отсебятины). В main: Сц.4 + Сц.1/2 + G5. Готов к мержу: PR-C (ось-1 + Сц.3-правило). Всё остальное — задокументированный handoff другим командам/ADR. Прод-предел достигнут.

## Связано
[[plan-tz-1-dizain-protsess-figma-kod-sts.4-perenos-form-one-kmp-lead-ds]] · `handoff-tz1-deferred-remainder` · `remainder-tz1-to-prod-limit` · `conclusion-sc4-by-tz-and-working`
