---
title: gate-pra-fidelity
type: report
permalink: tacticum/00-board/gate-pra-fidelity-1
tags:
- figma-ds
- gate
- controller
- fidelity
- pr-a
- sc1
- sc2
archived-at: 2026-08-03 11:16
---

# Гейт PR-A — FIDELITY-сверка с ТЗ figma-ds Сц.1/2

**Роль:** controller (read-only). **Объект:** worktree `tacticum-dev-web-sc12`, ветка `feat/ds-web-sc12`, коммит `b8e1b5e`.
**ТЗ (истина):** scenario-1-library.md + scenario-2-new-screen.md. **Карта gap:** `00-Board/map-existing-vs-gap-sc12`.

## ВЕРДИКТ: FIDELITY-PASS

Сделано ровно по ТЗ Сц.1/2, покрыт реальный gap (G1/G4/G6), 0 сверх-ТЗ, ссылки реальны, задеплоенное не переписано. Одно косметическое наблюдение (не дефект) — ниже.

---

## Пункт 1 — G1 (Сц.1 шаг 3 + приёмка): ПРОШЛО
Навык `angular-ds-component-authoring` покрывает анатомию web-компонента полностью и дословно по ТЗ:
- standalone / OnPush / signals (`input()`/`model()`) — есть.
- Директива на нативном элементе (`button[ivaX]`/`input[ivaX]`) vs элемент `iva-*` для составных — есть, с правилом «match the existing family».
- `IvaControlBase` + авто-CVA (не переписывать writeValue/registerOnChange) — есть.
- Слоты `ivaPrefix`/`ivaSuffix`/`ivaLabel` — есть.
- Токены-only, ноль hex (`main.ivaGetColor()`/`ivaFontPreset()`) — есть.
- Полнота вариантов `Property=Value` через union-типы, «не добавлять вариант, которого нет в мастере, не пропускать существующие» — есть.
- Folder anatomy (ts/html/scss/types/spec/index/stories+mdx+examples) + Nx-генератор `angular-library` + гайды `docs/guides/{development,storybook}` — совпадает с ТЗ строки 54-70.
- Storybook + `.mdx` + Jest spec — есть.
- Приёмка (полнота вариантов, витрина↔Figma числами, ноль hex, зелёный spec) — есть; числовое сравнение **делегировано** `ui-mockup-match` (не реализовано тут).

## Пункт 2 — G4 (Сц.2 шаги 1-7): ПРОШЛО
Навык `angular-ds-component-usage` покрывает все 7 шагов:
- Step 1 pre-flight (G6): auto layout? инстансы распознаются? иначе СТОП дизайнеру с конкретикой — есть.
- Step 2 `get_metadata` → точечные под-вызовы, дисциплина квоты ~200/день — есть.
- Step 3 резолв: Code Connect first → словарь code-bindings (`design_get_tokens` → `$extensions."dev.tacticum.code-bindings"`), `figma_key` затем имя, нормализация lowercase/strip (Radio↔Radiobutton и т.д.) — есть, с реальными полями биндинга (name/match/figma_key/selector/kind/source/storybook/inputs).
- Step 4 не в словаре → СТОП по элементу, роутинг в authoring или возврат дизайнеру, «не выдумывать разметку» — есть.
- Step 5 сборка из готовых по `selector` (директива vs элемент, `iva-form-field`, `[ivaMenuTriggerFor]`) + **читать `.mdx` перед use** — есть.
- Step 6 токены-only, делегировано `design-token-usage` — есть.
- Step 7 приёмка: ноль самодельной разметки + скриншот-референс; **числовой matcher НЕ реализован** — явно помечен как PR-B/G5, только ссылка.

**G5 не залез:** оба навыка явно пишут «do not implement pixel/ΔE matching here — reference `ui-mockup-match` (PR-B / gap G5)». Границу держат.

## Пункт 3 — не дублирует задеплоенное: ПРОШЛО
Diff трогает ровно 4 файла (2 новых SKILL.md + manifest.yaml + CHANGELOG.md). Тела `design-system-discovery` / `design-token-usage` / `ui-mockup-match` **не изменены** — навыки на них ссылаются (таблицы «Depends on / hands off», «Related skills», делегирование Step 6/Step 7), а не переписывают. `ui-mockup-match` сохраняет «no pixel»-философию (не в diff). `_mirrors.yaml` не тронут; в CHANGELOG зафиксировано «brownfield-only, не зеркалятся в web-development-base» — консистентно с картой gap (эти DS-навыки brownfield-only, mirror-parity CI не ломается).

## Пункт 4 — 0 сверх-ТЗ: ПРОШЛО
Отсебятины/додуманных запретов не найдено. Все гардрейлы прослеживаются к ТЗ Сц.1 (строки 102-109) / Сц.2 (строки 83-89) либо к реальным фактам репо из карты gap:
- «не вне DS-модуля / не расширять замороженный ui-kit / без publish нет ключа» — ТЗ Сц.1.
- «не копировать промежуточный React+Tailwind / беречь квоту / не выдумывать незамапленное» — ТЗ Сц.2.
- «premium-gated per ADR-0028» на `design_get_tokens` — реальный факт репо (карта gap), не выдуманное ограничение.
- Навыки строго web/Angular — KMP-крипа нет (KMP = Сц.4, вне скоупа; корректно не залезли). Это НЕ недоделка.

## Пункт 5 — ссылки реальны: ПРОШЛО
Все навыки-ссылки существуют каталогами в `iva-web-brownfield/ingredients/skills/`: `design-system-discovery`, `design-token-usage`, `ui-mockup-match`, `angular-ui-testing`, `tests-authoring`, `nx-workspace-discipline`, `ivcs-libs-contract`, а также сами `angular-ds-component-authoring` / `angular-ds-component-usage`. Manifest регистрирует оба (skill_spec, tier trial, три target-пути claude/codex/copilot), счётчик 26→28, версия 0.2.1→0.3.0 — корректно.

---

## Наблюдение (косметика, НЕ дефект, НЕ основание для доработки)
Взаимо-ссылка authoring↔usage **односторонняя по имени**: `usage` ссылается на `authoring` трижды (counterpart, Step 4 hand-off, роутинг), а `authoring` на `usage` — только концептуально («Scenario 2 build», «so Scenario 2 can read»), без имени навыка в «Related skills». Фиделити ТЗ это не нарушает (ТЗ взаимо-ссылку не требует), скоуп не задет. Опционально: добавить `angular-ds-component-usage` в «Related skills» authoring для симметрии. На вердикт не влияет.

## Отложенное (осознанно вне скоупа PR-A, НЕ недоделка)
G2 (`mdx_path`/поля словаря) — сторона словаря/RE-конвейера. G3/G7/G8 — RE-конвейер / отдельные PR. G5 (числовой matcher) — PR-B. Всё корректно вынесено ссылками, не имитировано.

## Гейты чистоты (для полноты)
Ветка явная (`feat/ds-web-sc12`, не main), один осмысленный коммит по задаче, без AI-подписей в теле коммита, секретов/мусора в diff нет (только 2 SKILL.md + manifest + CHANGELOG). Деплой не в скоупе этого гейта.