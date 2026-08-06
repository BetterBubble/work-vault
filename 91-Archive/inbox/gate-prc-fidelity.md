---
title: gate-prc-fidelity
type: report
permalink: tacticum/00-board/gate-prc-fidelity-1
role: controller
for: lead-ds
tz: figma-ds ТЗ ось-1 (§Ось 1)
object: worktree ~/tacticum/tacticum-dev-web-axis1, дельта cc016e6, ветка feat/ds-web-axis1
verdict: FIDELITY-PASS (с задокументированными обоснованными отклонениями)
tags:
- controller
- gate
- pr-c
- axis1
- figma-ds
- fidelity
archived-at: 2026-08-03 11:16
---

# Гейт PR-C ось-1 — FIDELITY-сверка с ТЗ

**ВЕРДИКТ: FIDELITY-PASS.** PR-C соответствует ТЗ figma-ds ось-1, покрывает реальный gap карты, факты/API реальны (сверено с tokens.json), 0 отсебятины сверх ось-1. Отклонения от буквального «object» задачи — все обоснованы и вынуждены (mirror/role-coverage), честно задокументированы в CHANGELOG.

## Дельта cc016e6 (что реально в коммите)
8 файлов. Кроме 3 названных в задаче (G7 discovery, iva-core skill, G8 quickstart в `iva-web-brownfield`) коммит также трогает **`iva-web-development-base`**: копия iva-core skill + manifest + CHANGELOG. Причина установлена (см. п.4) — вынужденное покрытие роли, не разрастание.

## 1. G7 (design-system-discovery, ось-1 п.1) — ПРОШЛО
Правка только web-копии `iva-web-brownfield`. Диф подтверждает всё требуемое:
- хардкод `iva-web for web and Electron` УБРАН, заменён на выбор по `platform`/`framework_hint` (сервер отдаёт через `design_list_systems`);
- идентичность репо (`package.json name`) — есть;
- опциональный `default_design_system_id` из `.tacticum/context.yaml` — есть;
- >1 ДС → спросить пользователя (ADR-0026) — есть;
- `installation_id` явно на `design_get_tokens` — есть;
- surface-split router: main iva-one UI→`@iva/design-system`; conference/MCU/iva-connect (`calls`/`mcu`/`conference`, `from 'iva-core'`)→`iva-core` через companion-скилл — есть;
- output-контракт `design_list_systems` дополнен `platform, framework_hint`.
- **«unified surfaces» НЕ введено** (в отличие от tacticum-ui-base SoT — тот НЕ тронут, канон-решение осознанно отложено лиду, см. карту §1/§5).

## 2. iva-core skill — ПРОШЛО
`iva-web-brownfield/.../iva-core-design-system/SKILL.md` (new). Все требования ТЗ:
- 3 факта separate DS: (1) пакет `from 'iva-core'` ≠ `@iva/design-system`; (2) токены `get-color('primary-color')`/`--primary-color` RGB-тройки руками, общий словарь невозможен; (3) Figma VCSWEB ≠ IVA DS UI KIT — все три дословно;
- каталог `iva-datepicker`/`iva-grid`/`iva-seeker`/`charts` — помечен **illustrative** (не выдан за резолвимый);
- серверная ДС + словарь code-bindings — секция «Deferred — not yet available (do not fake it)», честно отложено (server/RE/DS-команда), с указанием fallback на `core-demo`/`CHANGES.md`;
- companion → `design-system-discovery` как router. Витрина = `core-demo`+`CHANGES.md` (нет .mdx/AGENTS.md) — верно.

## 3. G8 quickstart — ПРОШЛО, факты реальны
`docs/user_manuals/iva-web-figma-mapping-quickstart.md` (new), по образцу существующего kmp-quickstart. Резолв по РЕАЛЬНОМУ словарю. Сверка с `design-systems/iva-web/tokens.json` (jq):
- `$extensions."dev.tacticum.code-bindings".components` = **49** ✓ (quickstart: «49»);
- non-null `figma_key` = **32** ✓ (quickstart/ds.yaml: «32 из 49», version 0.3.0);
- Chip→selector `iva-chip`, match `["chip"]` ✓; селекторы `iva-avatar`, `iva-form-field`, `button[ivaButton]` реально существуют ✓;
- name-алиасы (`match[]` >1) — 25 компонентов ✓; нормализация lowercase/убрать пробелы-дефисы-подчёркивания описана корректно;
- `installation_id` обязателен на каждом `design_*` — верно.
- API реальны: `design_get_tokens`, Figma MCP `get_metadata`/`get_code_connect_map`/`get_screenshot`/`get_variable_defs`/`get_design_context` — **выдуманного API нет**.

## 4. 0 сверх-ТЗ — ПРОШЛО (одно вынужденное отклонение, обосновано)
- **Отклонение:** iva-core skill добавлен также в `iva-web-development-base` (лейн-owner), не только в названный задачей brownfield. **Причина (CHANGELOG dev-base 0.1.2):** без копии в лейне падает `test_role_covers_replaced_profile` — роль `iva-role-web` через лейны не покрывает замещаемый профиль. Прецедент — 0.1.1 (angular-ds-component-*). Две копии **байт-идентичны** (сверено `diff` → IDENTICAL). Это дисциплина покрытия, НЕ разрастание, в рамках ось-1 (client-side skill).
- Серверная часть iva-core (регистрация ДС, словарь) — **НЕ реализована**, честно отложена. Верно (вне скоупа ось-1 клиента).
- Массовый фикс дрейфа (rn/mail хардкод), tacticum-ui-base «unified», seed `framework_hint: react` — НЕ тронуты. Верно (карта §5, вне PR-C).

## 5. Не дублирует — ПРОШЛО
G8 и discovery ссылаются на существующие `angular-ds-component-usage`/`-authoring`, `ui-mockup-match`, `design-token-usage`, `tacticum-context` — все реально присутствуют в профилях, не переписаны. iva-core → companion `design-system-discovery`.

## Git-чистота — ПРОШЛО
worktree чист (`git status` пуст), ветка `feat/ds-web-axis1` (не main), автор коммита «Александр Шульга», AI-подписей нет. Секретов/мусора в дельте нет (8 md/yaml файлов).

## Заметки лиду (не блокеры)
1. **Mirror-lock отсутствует** для `iva-core-design-system` и `design-system-discovery` в паре `iva-web-development-base ↔ iva-web-brownfield` (`_mirrors.yaml` строки 42-59 их не содержат). Две iva-core копии сейчас байт-идентичны, но НЕ под CI-локом → риск будущего дрейфа. Расширение `_mirrors.yaml` — осознанно отдельное решение лида/ГД (карта §5), не gap PR-C.
2. **G7-фикс только в физкопии brownfield.** Лейн `iva-web-development-base` получает `design-system-discovery` через `depends_on: tacticum-ui-base`, а там сохраняется «unified»-допущение и не читается framework_hint. Роль покрыта по присутствию, но контент канона (tacticum-ui-base) не выровнен — это канон-решение, отложенное лиду (карта §1). Осознанно вне PR-C.

Итог: цель «PR-C по ТЗ ось-1, только gap, факты реальны, без отсебятины» — достигнута.