---
title: 'HANDOFF — отложенный остаток ТЗ#1 (figma-ds) для других команд'
type: note
status: draft
permalink: tacticum/00-board/handoff-tz1-deferred-remainder
tags:
- board
- design-system
- lead-ds
- tz1
- handoff
---

# HANDOFF: остаток ТЗ#1 (figma-ds) вне template-предела

**Контекст:** template-side ТЗ#1 закрыт capability-профилями (Сц.4 + Сц.1/2 PR-A + G5 PR-B + ось-1/Сц.3-правило PR-C). Ниже — что осталось ЗА пределом template-профиля: чужая зона, с контекстом для передачи. Формат: **что осталось · кому · контекст/зачем · артефакт-опора**.

---

## 1. Сц.3 — прогон миграции iva-one (легаси → новая ДС)
- **Что осталось:** фактическая миграция репо iva-one: ~72 легаси-файла на замороженном `ui-kit`/`dsGetColor` → `@iva/design-system`/`ivaGetColor` (556 экранов/файлов уже на новом). 2 слоя батчами (сначала токены по transition-таблице, потом компоненты по словарю), каждый батч — приёмка как Сц.2. Удаление легаси-слоя — отдельная задача по нулю в отчёте покрытия.
- **Кому:** **команда iva-one** (миграция самого репо). Дисциплина/правило — уже в навыке `angular-ds-component-authoring` (§Migration, Сц.3), агент им следует; но прогон и «сколько осталось» — работа команды.
- **Контекст:** template даёт ТОЛЬКО правило-дисциплину (не codemod, не coverage-тулинг). Отчёт покрытия iva-one сейчас считается вручную grep-ом.
- **Опора:** навык `angular-ds-component-authoring §Migration (Scenario 3)`; ТЗ `figma-ds-scenario-3-migration.md`.

## 2. iva-core — серверная ДС + словарь code-bindings (конференц/VCSWEB-поверхность)
- **Что осталось:** завести серверную ДС `iva-core` (как `iva-web`): `design-systems/iva-core/design-system.yaml` + токены + словарь `$extensions."dev.tacticum.code-bindings"` для компонентов iva-core (iva-datepicker/grid/seeker/charts…), attach к workspace (сервер умеет N:M). Источник — Figma VCSWEB (не IVA DS UI KIT), пакет `iva-core` (`get-color()`/`--primary-color` RGB).
- **Кому:** **server/RE + DS-команда** (серверная ДС и её словарь — не template-профиль).
- **Контекст:** template-часть (тонкий скилл `iva-core-design-system` + surface-router в `design-system-discovery`) уже в PR-C; рантайм-резолв по словарю заработает, когда server/RE заведут ДС+словарь. До тех пор скилл fallback на demo-app `core-demo`/`CHANGES.md`.
- **Опора:** навык `iva-core-design-system` (§Deferred); ТЗ `figma-ds-multirepo-and-selection.md §Ось-1`.

## 3. G2/G3 — словарь v2 (доп-поля) + авто-пересборка словаря + CI-проверка
- **Что осталось:** (G2) добавить в code-bindings поля `mdx_path/host/requires/slots/import/category` (сейчас v1: name/match/figma_key/selector/kind/source/storybook/inputs/notes). (G3) авто-пересборка словаря из кода iva-one + CI-проверка «каждый компонент/пропс существует» (аналог `figma connect publish --dry-run`); регуляризация gap-отчёта «есть в Figma — нет в коде».
- **Кому:** **RE-конвейер** (генерация словаря/токенов в dev-репо, вне веб-профиля).
- **Контекст:** влияет и на Сц.1 (авто-пересборка после наполнения), и на Сц.2 (чтение доки перед сборкой). Текущий словарь (v1, 49 комп./32 ключа) собран вручную (pilot).
- **Опора:** карта `map-existing-vs-gap-sc12` (G2/G3); ТЗ `figma-ds-scenario-1-library.md`.

## 4. Figma — 17 null figma_key + подтверждение фреймов экранов
- **Что осталось:** дозаполнить 17 `figma_key: null` в словаре (компоненты из ДРУГИХ файлов мультифайловой ДС — сейчас обоснованный null, матч по имени) через Figma REST (`GET /v1/files/:key/components`) или Figma MCP. Подтвердить auto-layout Figma-фреймы для пилотных экранов (критерий приёмки Сц.2/4).
- **Кому:** **дизайнеры / владелец ДС (Д.Солонко)** + доступ к Figma (REST-токен уже выпущен на iva-web-brownfield installation, либо локальный Figma MCP).
- **Контекст:** 17 null — НЕ блокер (обоснованный null, матч по имени работает); дозаполнение — только когда реально нужен ключ. Figma-фрейм экрана — критерий (в) ТЗ, отложен (нужен Figma-доступ/дизайнер).
- **Опора:** словарь `phase2-provisional-iva-web-dictionary` (resolved, 17 null помечены); `map-existing-vs-gap-sc12`.

---

## 5. Внутренние ADR/процесс (решение президента/архитектурное — не другие команды, но для полноты предела)
- **Канон `tacticum-ui-base`:** допущение «Iva DS unified across surfaces» → surface-split (роль тянет discovery через ui-base-копию, где «unified» держится; наш G7-фикс — только в brownfield-копии). Уровень ADR (ADR-0056/0059). ГД фиксирует президенту.
- **`_mirrors` для DS-навыков:** design-system-discovery (7 копий/6 хэшей, CI-лок только пара analysis-base↔fr-analyst) + ui-mockup-match (5 копий, вне _mirrors). Решить: расширить CI-лок на web-копии ИЛИ признать независимыми. Прод-риск: правка в одном стеке не доходит до других (CI ловит только задекл. пары).

## Связано
[[План ТЗ-1 Дизайн-процесс Figma↔код — Сц.4 перенос форм one→kmp (lead-ds)]] · `00-Board/remainder-tz1-to-prod-limit` · `map-existing-vs-gap-sc12` · `map-existing-vs-gap-pr-c-axis1` · `map-sc3-and-remainder`
