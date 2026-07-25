---
title: impl-iva-core-skill
type: report
permalink: tacticum/00-board/impl-iva-core-skill
status: draft
role: implementer
for: lead-ds
tz: figma-ds ТЗ#1, PR-C = ось-1, iva-core template-side (тонкий скилл)
worktree: ~/tacticum/tacticum-dev-web-axis1 (ветка feat/ds-web-axis1)
autonomy: false
tags:
- ds-web
- figma-ds
- pr-c
- axis1
- iva-core
- implement
---

# impl-iva-core-skill — тонкий скилл ДС iva-core (ось-1, конференц-поверхность)

Готов НОВЫЙ тонкий скилл ДС `iva-core`. Один файл, строго по ТЗ (3 факта + каталог из спека). НЕ commit, manifest/CHANGELOG/др. НЕ трогал (консолидирует лид).

## Путь (создан)

`templates/iva-web-brownfield/ingredients/skills/iva-core-design-system/SKILL.md`

Имя папки/скилла: `iva-core-design-system` (предложенное лидом; понятно, симметрично соседям `design-system-discovery` / `design-token-usage`). Если лид хочет короче — `iva-core-ds`, но рекомендую оставить полное.

## Содержание (что внутри, по секциям)

- **Frontmatter** `name` + `description` (в стиле образца — с триггерами). Язык — английский (как все скиллы профиля).
- **Заголовок/интро** — iva-core = самостоятельная ДС (не форк), flat npm-пакет, одевает конференц/call/VCS-поверхность (`iva-connect` + `calls`/`mcu`/`conference` либы iva-one). Основной UI iva-one → `@iva/design-system`.
- **Surface-router строка** — main iva-one UI → `@iva/design-system`; конференц/MCU/VCS overlays → `iva-core`; не делят токены/пакет/Figma.
- **«Three facts that make it a separate DS»** (ровно из ТЗ):
  1. другой пакет/импорт — `import { ... } from 'iva-core'`, не `@iva/design-system`;
  2. другая архитектура токенов — `get-color('primary-color')` / `--primary-color`, RGB-тройки руками, не `ivaGetColor(...)` / `--iva-color-*`; общий словарь невозможен;
  3. другой Figma — VCSWEB, не IVA DS UI KIT.
- **«Where the design system is documented»** — у репо нет `.mdx` и `AGENTS.md`; витрина = demo-app `core-demo`, доки = `CHANGES.md`.
- **«Component catalog (illustrative)»** — примеры из ТЗ: `iva-datepicker`, `iva-grid`, `iva-seeker`, `charts`, «и др.». Явно помечено illustrative, не exhaustive; авторитетный резолв — из словаря code-bindings (отложено); до тех пор сверять по `core-demo`. НЕ выдумывал компоненты/API сверх спека.
- **«Deferred — not yet available (do not fake it)»** — честная пометка отложенного (см. ниже).
- **«Companion skill»** — ссылка на `design-system-discovery` как router по поверхности.

## Предлагаемый description_trigger (для манифеста — лид впишет)

> Use when the UI work is on the conference / call / VCS surface that pulls the `iva-core` design system — the npm package imported as `from 'iva-core'` (the `iva-connect` repo, and the `calls` / `mcu` / `conference` libs inside iva-one). A DIFFERENT design system from `@iva/design-system` (main iva-one UI): different package, token architecture (`get-color()` / `--primary-color`, RGB triples by hand), and Figma file (VCSWEB). Triggers on "iva-core", "iva-connect", "conference UI", "VKS UI", "VCSWEB", "get-color", "--primary-color", "iva-datepicker", "iva-grid", "iva-seeker", "conference design system".

(В самом SKILL.md frontmatter уже содержит рабочую версию этого description с триггерами.)

## Что помечено ОТЛОЖЕННЫМ (честно, не выдумано как готовое)

Секция «Deferred» прямо говорит: server/RE/DS-команда, НЕ этот скилл:
- серверная ДС `iva-core` (attach к workspace) — ещё НЕ зарегистрирована;
- словарь code-bindings для каталога iva-core — ещё НЕ построен.

Скилл описывает только *подход*; резолв по словарю (`design_*`) заработает КОГДА словарь загрузят. До тех пор — НЕ выдавать серверный резолв за готовый: fallback на `core-demo` / `CHANGES.md` + прямое чтение `get-color()` / `--primary-color`, и пометка в PIN, что резолв iva-core ручной до появления словаря.

## Границы соблюдены

- Один новый файл. manifest.yaml / CHANGELOG.md / соседние скиллы — НЕ трогал.
- НЕ создавал `design-systems/iva-core/` (серверный seed) и словарь — отложено.
- НЕ commit, НЕ push. Ветка `feat/ds-web-axis1`, worktree axis1.
- autonomy off — жду решения лида (принять / вернуть на переделку / имя).

## Открытый вопрос лиду (не блокер)

Router-note в `design-system-discovery` (конференц → iva-core; основной UI → @iva/design-system) в моём scope НЕ было (я авторю только новый скилл). Карта `map-existing-vs-gap-pr-c-axis1` относит router-note к G7-правке существующего `design-system-discovery`. В новом скилле companion-ссылка на `design-system-discovery` уже есть; обратную router-note в самом `design-system-discovery` пусть делает исполнитель G7 / решит лид.