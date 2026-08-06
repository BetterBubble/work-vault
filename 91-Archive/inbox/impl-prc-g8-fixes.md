---
status: draft
type: report
role: implementer
task: ТЗ#1 figma-ds, PR-C, ось-1 / G8 web-quickstart
lead: lead-ds
worktree: /Users/bubblemac/tacticum/tacticum-dev-web-axis1
branch: feat/ds-web-axis1
commit: 5fe8163
date: 2026-07-24
permalink: tacticum/00-board/impl-prc-g8-fixes-1
archived-at: 2026-08-03 11:16
---

# impl PR-C G8 — should-fix критика в web-quickstart

Внесены should-fix критика в `docs/user_manuals/iva-web-figma-mapping-quickstart.md`.
Тронут ТОЛЬКО этот файл. Версия iva-web-brownfield остаётся 0.5.0 (правка в уже
добавленном на 0.5.0 доке; CHANGELOG не трогал — version-discipline на этот
non-seeded user-manual не распространяется, см. ниже).

Коммит `5fe8163`: `fix(ds-web): quickstart G8 — source not .mdx/slots + surface scope note (critic)` (без AI-подписей). НЕ запушен.

## Сверка с tokens.json (source of truth)

`design-systems/iva-web/tokens.json` → `$extensions."dev.tacticum.code-bindings"`:
- Реальные поля компонента: `figma_key, inputs, kind, match, name, notes, selector, source, storybook` (49 компонентов).
- **`.mdx` — НЕТ** (проверено: 0 вхождений в code-bindings). **`slots` — НЕТ** (0 вхождений).
- `source` — ЕСТЬ (путь от `components_root`). `storybook` — ЕСТЬ (docs-page id для `storybook_base`). `inputs` — ЕСТЬ, inline в биндинге.
- `cb.usage` дословно: «Before emitting code, read the `source` file (path relative to components_root) for the exact inputs/outputs — do not guess props» + «`storybook` is a docs page id…».
- Menu-биндинг: `selector=iva-menu`, `figma_key=null`, `source=overlays/menu/menu/menu.component.ts`, `notes` содержит «open via [ivaMenuTriggerFor] or [ivaMenuTriggerContextFor]».

Итог сверки: критик прав — агента вело за несуществующими `.mdx`/`slots`; правильный источник — `source`-файл, `inputs` inline, `storybook` доп-док.

## Изменения (до/после)

### п.1 — Шаг 3, пункт 3 (достоверность полей словаря)
До:
> 3. Перед использованием компонента прочитай его .mdx / source-файл из биндинга — точные inputs/outputs/slots оттуда, не угадывай. Собирай по selector: …

После:
> 3. Перед использованием компонента прочитай `source`-файл из биндинга (поле `source`, путь от `components_root`) — точные inputs/outputs, не угадывай. Часть `inputs` уже inline в самом биндинге (поле `inputs`); `storybook`-страница (поле `storybook`) — доп-док. Собирай по selector: …

`.mdx` и `slots` убраны полностью.

### п.2 — Скоуп + связь с G7-роутером (шапка)
Добавлена строка в шапку (после вводного абзаца, перед «Статус: пилот»):
> Этот пилот покрывает ВЕБ-поверхность (`@iva/design-system` / DS `iva-web`); конференц/MCU-UI → скилл `iva-core-design-system`. `design_system_id` здесь — та ДС, которую резолвит `design-system-discovery` (surface-router); в этом пилоте она = `iva-web`.

Цель — чтобы литерал `design_system_id="iva-web"` не стал ре-хардкодом в не-iva-web репо (ровно то, что ось-1/G7 снимает в design-system-discovery).

### п.3 — Menu (~стр.113, «Как пользоваться»)
До: `… формы внутри iva-form-field, меню через [ivaMenuTriggerFor]) → токены …`
После: `… формы внутри iva-form-field, меню iva-menu) → токены …`

Смягчено до селектора `iva-menu` для консистентности со списком селекторов (`button[ivaButton]` / `iva-*`). Примечание: `[ivaMenuTriggerFor]` реально присутствует, но в поле `notes` биндинга Menu (как механизм открытия), не как `selector`; в списке селекторов ему не место.

### п.4 — блок AGENTS.md (дедуп с навыком)
Добавлена строка сразу под заголовком блока `<!-- figma-mapping-pilot -->`:
> Канонический алгоритм — в навыке `angular-ds-component-usage`; этот блок — только триггер/напоминание.

## Валидаторы (venv `apps/backend/.venv/bin/python`)

- **Полный `pytest apps/backend/tests/catalog/ -q`:** не-DB тесты зелёные. Все ERROR/FAILED — исключительно DB-connection (Postgres `localhost:5432` / `docker run postgres:16-alpine` недоступны в среде): 120 ERROR + 2 FAILED (`test_patch_profile_404`, `test_create_draft_404_unknown_profile`) — оба падают на connect к 5432, не на ассертах. Правка — markdown, Python-тесты не задевает.
- **Coverage-тест `test_role_covers_replaced_profile`:** PASSED (exit 0).
- **Mirror-sync `test_mirror_content_is_byte_identical`:** PASSED (exit 0). У этого user-manual нет байт-копии-зеркала (единственная ссылка — CHANGELOG iva-web-brownfield), так что правка зеркал не рушит.
- **Version-discipline `test_different_content_same_version_is_rejected`:** не запускается без Docker/Postgres (setup ERROR, не фейл). Плюс не применим к этому доку: `docs/user_manuals/*` — не seeded/manifest-ingredient контент, version-discipline проверяет seeded ingredient-контент против версии. Поэтому CHANGELOG-помету не добавлял, версия 0.5.0 без изменений.

_Примечание по среде:_ conftest-репортёр глушит финальную summary-строку pytest при прогоне (артефакт среды); результаты подтверждены по exit-кодам и отсутствию F/E в прогресс-выводе.

## Самопроверка

- [x] `.mdx` и `slots` убраны из дока (0 вхождений после правки).
- [x] scope-строка про ВЕБ-поверхность / surface-router есть в шапке.
- [x] тронут ТОЛЬКО `docs/user_manuals/iva-web-figma-mapping-quickstart.md` (`git status --short` = 1 файл, +13/-4).
- [x] iva-core SKILL.md, design-system-discovery, manifest — НЕ тронуты.
- [x] версии 0.2.0/0.3.0, числа 49/32, fileKey, селекторы, installation_id — не тронуты.
- [x] коммит без AI-подписей, НЕ запушен.