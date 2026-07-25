---
title: impl-g1-authoring
type: report
permalink: tacticum/00-board/impl-g1-authoring
status: draft
role: implementer
for: lead-ds
tz: scratchpad/ds-scan/figma-ds-scenario-1-library.md (Сц.1 шаг 3 + приёмка)
worktree: ~/tacticum/tacticum-dev-web-sc12
date: 2026-07-24
tags:
- figma-ds
- sc1
- impl
- g1
- lead-ds
---

# impl G1 — навык `angular-ds-component-authoring`

Закрыт gap **G1** (Сц.1): анатомия/конвенции авторинга нового web-DS-компонента. Ранее такого навыка не было ни в одном пакете.

## Файл (единственный созданный)
`/Users/bubblemac/tacticum/tacticum-dev-web-sc12/templates/iva-web-brownfield/ingredients/skills/angular-ds-component-authoring/SKILL.md`

Границы соблюдены: тронут ТОЛЬКО этот файл. `manifest.yaml` / `CHANGELOG.md` / зеркала / роль — не трогал. `git commit` не делал. (`git status`: мой untracked каталог + соседний `angular-ds-component-usage/` — это параллельный implementer G4, не мой.)

## Содержание по секциям
- **Frontmatter** — `name: angular-ds-component-authoring` + `description` с триггерами (new DS component / IvaControlBase / ivaButton / iva-* / Storybook / Property=Value / gap report...).
- **Intro + разграничение** — это implementation/authoring-фаза; явно отделён от `design-system-discovery` (design-фаза, токены в PIN) и `design-token-usage` (резолв токена). Отсылка к репо-гайдам `libs/design-system/src/lib/docs/guides/{development,storybook}` и Nx-генератору `angular-library` как source of truth.
- **When to call** — триггер по gap-отчёту «есть в Figma, нет в коде»; сначала прочитать мастер из Figma MCP.
- **Component folder anatomy** — папка-файлы (component.ts/html/scss/types.ts/spec.ts/index.ts/stories/mdx/examples); standalone + OnPush + signals (`input()`/`model()`).
- **Selector shape** — директива на нативном элементе `button[ivaButton]`/`a[ivaButton]`/`input[ivaInput]` vs элемент `iva-*` для составных виджетов; правило выбора.
- **Form controls** — `IvaControlBase` (CVA регистрируется сам), не переписывать writeValue/registerOnChange.
- **Slots** — маркеры контент-проекции `ivaPrefix`/`ivaSuffix`/`ivaLabel`.
- **Styling — tokens only** — `main.ivaGetColor()` / `ivaFontPreset()`, НОЛЬ hex/raw; нет токена → стоп, не инлайнить.
- **Completeness** — все `Property=Value` варианты как union в `.types.ts`; Storybook-стори; `.mdx`-док (Quick Start, чтобы Сц.2 читал usage); зелёный spec.
- **Acceptance** — полнота вариантов; витрина↔Figma ЧИСЛАМИ (ссылка на числовой режим `ui-mockup-match` = PR-B, здесь НЕ реализуем); ноль hex; зелёный тест.
- **Guardrails** — не изобретать компоненты/API вне реальной DS-поверхности; незнакомое → свериться с репо/гайдами, не выдумывать; не хардкодить; не класть вне DS-lib; не расширять замороженный `ui-kit`; без опубликованного мастера — не цель.
- **Related skills** — перекрёстные ссылки (ниже).

## Навыки, на которые сослался (не дублирую)
`design-system-discovery`, `design-token-usage`, `ui-mockup-match`, `angular-ui-testing`, `tests-authoring`, `nx-workspace-discipline`, `ivcs-libs-contract`.

## Предлагаемый `metadata.description_trigger` (для манифеста — впишет лид)
> Authoring a NEW `@iva/design-system` Angular web component from a Figma master (gap-report case): component folder anatomy, directive `button[ivaX]` vs element `iva-*`, `IvaControlBase`+CVA, `ivaPrefix/ivaSuffix/ivaLabel` slots, token-only styling (zero hex), full variant coverage + Storybook + `.mdx` + spec.

## Заметки для лида
- Строго по ТЗ Сц.1 шаг 3 + приёмка; правил сверх ТЗ не добавлял (президентский принцип).
- Тон/формат — по образцу `design-system-discovery`/`design-token-usage` (английский, frontmatter name+description). Если пакет требует русский — скажи, переведу.
- Числовой Figma-матчинг сознательно оставлен ссылкой на PR-B (G5), не реализован в этом навыке.