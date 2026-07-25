---
title: 'ТЗ#1 Figma↔код — handoff-остаток для других команд'
type: note
permalink: tacticum/12-features/tz1-figma-ds-handoff-ostatok
status: current
created: '2026-07-25'
updated: '2026-07-25'
repo: /Users/bubblemac/tacticum/tacticum-dev
tags:
- feature
- tz1
- figma-ds
- handoff
- design-system
---

# ТЗ#1 (Figma↔код) — остаток вне template-предела (передача другим командам)

**Статус ТЗ#1:** template-side ЗАКРЫТ и В ПРОДЕ (Сц.4 + Сц.1/2 + G5 + ось-1/Сц.3-правило, деплой 2026-07-25 verify зелёный). Ниже — что осталось ЗА пределом профиля «Разработчик+ИИ-агент»: это ДРУГИЕ роли/команды по разделению самого ТЗ Солонко. Профиль даёт правило/дисциплину; фактическое исполнение на чужих репо/сервере/Figma — их зона. Формат: **что · кому · контекст · опора в ТЗ**.

## 1. Сц.3 — прогон миграции iva-one (легаси → новая ДС)
- **Что:** фактическая миграция репо iva-one — ~72 легаси-файла на замороженном `ui-kit`/`dsGetColor` → `@iva/design-system`/`ivaGetColor` (556 экранов уже на новом). 2 слоя батчами (токены по transition-таблице → компоненты по словарю), каждый батч — приёмка как Сц.2. Удаление легаси-слоя — по нулю в отчёте покрытия.
- **Кому:** **команда iva-one** (миграция их репо).
- **Контекст:** template даёт ТОЛЬКО правило-дисциплину (навык `angular-ds-component-authoring §Migration`), не codemod и не coverage-тулинг. Отчёт покрытия сейчас — вручную grep.
- **Опора:** ТЗ `figma-ds-scenario-3-migration.md`.

## 2. iva-core — серверная ДС + словарь code-bindings (VCSWEB-поверхность)
- **Что:** завести серверную ДС `iva-core` (как `iva-web`): `design-systems/iva-core/design-system.yaml` + токены + словарь `$extensions."dev.tacticum.code-bindings"` для компонентов iva-core (datepicker/grid/seeker/charts…), attach к workspace. Источник — Figma VCSWEB, пакет `iva-core`.
- **Кому:** **server/RE + DS-команда** (серверная инфра, не template-профиль).
- **Контекст:** template-часть (тонкий скилл `iva-core-design-system` + surface-router в `design-system-discovery`) УЖЕ в проде (PR-C); рантайм-резолв заработает, когда server/RE заведут ДС+словарь. До тех пор скилл — fallback на demo-app.
- **Опора:** ТЗ `figma-ds-multirepo-and-selection.md §Ось-1`.

## 3. G2/G3 — словарь v2 (доп-поля) + авто-пересборка + CI
- **Что:** (G2) добавить в code-bindings поля `mdx_path/host/requires/slots/import/category` (сейчас v1). (G3) авто-пересборка словаря из кода iva-one + CI-проверка «каждый компонент/пропс существует» (аналог `figma connect publish --dry-run`) + регуляризация gap-отчёта «есть в Figma — нет в коде».
- **Кому:** **RE-конвейер** (генерация словаря/токенов в dev-репо).
- **Контекст:** текущий словарь v1 (49 комп./32 ключа) собран вручную (pilot). Влияет на Сц.1 (пересборка) и Сц.2 (чтение доки).
- **Опора:** ТЗ `figma-ds-scenario-1-library.md` (G2/G3); карта `map-existing-vs-gap-sc12`.

## 4. Figma — 17 null figma_key + подтверждение фреймов
- **Что:** дозаполнить 17 `figma_key: null` в словаре (компоненты из других файлов мультифайловой ДС; сейчас обоснованный null, матч по имени) через Figma REST/MCP. Подтвердить auto-layout Figma-фреймы пилотных экранов (критерий приёмки Сц.2/4).
- **Кому:** **дизайнеры / владелец ДС (Д.Солонко)** + Figma-доступ (REST-токен уже выпущен).
- **Контекст:** 17 null — НЕ блокер (матч по имени работает); дозаполнение — когда реально нужен ключ.
- **Опора:** словарь `phase2-provisional-iva-web-dictionary`; `map-existing-vs-gap-sc12`.

## 5. Внутренние ADR/процесс (архитектура — решение президента, для полноты)
- **Канон `tacticum-ui-base`:** допущение «Iva DS unified across surfaces» → surface-split (наш G7-фикс — только в brownfield-копии; ui-base-канон держит «unified»). Уровень ADR (ADR-0056/0059). Следствие: ось-1 KMP-align — до ADR web-only.
- **`_mirrors` для DS-навыков:** `design-system-discovery` (7 копий/6 хэшей, CI-лок только пара analysis-base↔fr-analyst) + `ui-mockup-match` (5 копий, вне _mirrors). Решить: расширить CI-лок на web-копии ИЛИ признать независимыми. **Прод-риск:** правка в одном стеке не доходит до других (CI ловит только задекл. пары).

## Связано
[[handoff-tz1-deferred-remainder]] (исходная board-версия) · [[final-fidelity-tz1]] · [[prod-deploy-3tz-done-2026-07-25]] · [[plan-tz-1-dizain-protsess-figma-kod-sts.4-perenos-form-one-kmp-lead-ds]]
