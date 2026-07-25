---
title: explore-2b-roles
type: note
permalink: tacticum/00-board/explore-2b-roles
status: draft
tags:
- explorer
- role-presets
- iva
---

# explore-2b-roles

Разведка грунта под сборку трёх ролей-пресетов 2B: `iva-role-architect`, `iva-role-qa`, `tacticum-role-techwriter`. Всё read-only. GUARDRAIL соблюдён: analysis-сущности только читал.

Репо: `/Users/bubblemac/tacticum/tacticum-dev` (branch `main`). Лейн `iva-write-base` — в отдельном worktree ветки `feat/iva-write-base`: `/Users/bubblemac/tacticum/tacticum-dev-iva-write`. В `main` его ЕЩЁ НЕТ.

## 1. Образцы роли-пресета (структура-эталон)

Каталог роли = 3 файла: `manifest.yaml` + `README.md` + `CHANGELOG.md` (см. `templates/iva-role-analyst/`, `templates/iva-role-go/`). Собственных `ingredients/` у роли нет.

**`iva-role-analyst/manifest.yaml`** (guardrail — читал, не правил) — header-поля роли-пресета:
- `schema_version: "2"` (стр. 10)
- `profile_id: iva-role-analyst` (11)
- `name:` (12), `version: "0.1.0"` (13), `maintainer: "mr.diaret@ya.ru"` (14), `license: "MIT"` (15), `deprecated: false` (16)
- `depends_on:` (20-22) = `[tacticum-core-base, iva-analysis-base]`
- `description:` (24-39), `persona.role: analyst` + `persona.scope` (41-43)
- `target_tasks:` (45-50), `stack.required/optional: []` (52-55)
- `ide_targets:` (58-63) claude-code/codex `full`, copilot/opencode/gemini `unsupported`
- `profiles.trial/full` (65-69)
- `ingredients: []` (72) — пустой, чистая композиция

**`iva-role-go/manifest.yaml`** (не-guardrail, второй образец): те же header-поля; `depends_on` (22-27) = `[tacticum-core-base, iva-analysis-base, iva-go-development-base, tacticum-bugfix-base, tacticum-documentation-base]`; `persona.role: developer` (50); `stack.required: [go]` (61-63); `ingredients: []` (82).

README (см. `iva-role-analyst/README.md`) несёт: заголовок, блок «Композиция» (depends_on), «Что умеет», таблицу «Провенанс» (лейн → что даёт), «Установка/CLI», «Связано» (ADR/эпик/US). CHANGELOG — Keep-a-Changelog, версия = `version` манифеста.

## 2. ingredient_id по лейнам + дизъюнктность

**tacticum-core-base** (6): `getting-started`, `kb-navigation`, `tacticum-context`, `conventional-git`, `context7`, `tacticum-mcp`.

**iva-analysis-base** (17, guardrail-read): `fr-authoring`, `api-contracts-discovery`, `brd-authoring`, `adr-authoring`, `multi-container-pin-authoring`, `pin-api-verification`, `pin-upstream-dependency-check`, `tests-authoring`, `system-analyst`, `tacticum-workflow`, `start-feature`, `update-feature`, `prepare-tz`, `start-task`, `approve-docs`, `helm-analyst`, `iva-read`.

**tacticum-documentation-base** (1): `doc-authoring`.

**iva-write-base** (1, ветка feat/iva-write-base): `iva-write` (mcp_server_spec, url `.../iva-write/mcp`, scope `iva-req-write`, allowed_tools: confluence_create_page/update_page, jira_create_issue/add_comment/transition_issue).

**Попарная дизъюнктность по ролям — ВСЕ чисты:**

| Роль | Лейны | Σ | union | disjoint |
|---|---|---|---|---|
| iva-role-architect | core+analysis+write | 6+17+1=24 | 24 | ✅ |
| iva-role-qa | core+analysis+write | 6+17+1=24 | 24 | ✅ |
| tacticum-role-techwriter | core+documentation+write | 6+1+1=8 | 8 | ✅ |

Ключевой вопрос: **`iva-analysis-base` ∩ `iva-write-base` = ∅** ✅. analysis несёт read-канал `iva-read`, write-лейн несёт `iva-write` — разные ingredient_id (сделано намеренно во избежание single-owner-коллизии, см. коммент манифеста write-base стр. 12-13). Пересечений нет ни в одной паре. golden-parity (union == Σ) пройдёт для всех трёх.

Депт-1: ни один из 4 лейнов НЕ несёт `depends_on` (core/analysis/documentation — база; write-base leaf явно без depends_on). `test_lanes_are_depth1_bases` пройдёт.

## 3. persona.role enum — enum ОТСУТСТВУЕТ, правки схемы НЕ нужны

`templates/_schema/manifest.v2.schema.json` (актуальная, `schema_version:"2"`) — минимальная: `required: [schema_version]`, `properties` только `schema_version` / `depends_on` / `ide_targets`. **`persona` в схеме v2 вообще не описана и не enum'ится.** В v1 (`manifest.v1.schema.json`) `persona.role` — свободная строка (тоже без enum).

Подтверждение практикой: по репозиторию `persona.role` уже принимает произвольные значения — `tech-lead`, `developer`, `analyst`, `system-analyst`, `requirements-analyst`, `engineer`, `ui-engineer`, **`architect`** (уже используется в `tacticum-platform-dev`), `requirements-author` (в iva-write-base). Значит `architect` / `qa` / `techwriter` в `persona.role` пройдут без изменения схемы.

Helm-миграция `0026_membership_role_architect.py` — это enum РОЛЕЙ ЧЛЕНСТВА в helm (backend ИВА), к каталожной схеме tacticum-dev отношения не имеет. Для сборки ролей-пресетов её трогать не нужно.

## 4. QA-контент (собрать qa из уже существующего)

Отдельного qa-лейна/директории НЕТ (`templates/*qa*` пусто). QA-грунт уже лежит в `iva-analysis-base`:
- скилл **`tests-authoring`** (`templates/iva-analysis-base/ingredients/skills/tests-authoring/SKILL.md`) — TESTS/TC контрактного уровня (GIVEN/WHEN/THEN, UC/FT-трассировка, переиспользование Allure-кейсов).
- MCP **`helm-analyst`** несёт read-тул `requirement_tests` (покрытие требования автотестами + статус из Allure TestOps) — это источник Allure-данных для QA.

Вывод: **отдельный qa-lane не нужен** — `iva-role-qa = core + analysis + write` полностью покрывает QA из существующего (tests-authoring + helm-analyst.requirement_tests + write-канал публикации). Композиция qa совпадает с architect (см. риск ниже).

## 5. Техпис-пробел

`tacticum-documentation-base` несёт РОВНО ОДИН скилл `doc-authoring` (skill_spec, install_scope user) — «пост-dev документирование: что документировать (API/contract, README, changelog) + структура, стэк-агностично». Ни агента, ни команды, ни оркестратора write-публикации.

`iva-write-base` несёт ТОЛЬКО mcp_server_spec `iva-write` (write-ТУЛЫ), без скилла/агента/команды, которые бы направляли их использование.

Итог для techwriter (`core + documentation + write`): есть скилл «что писать» (`doc-authoring`) + write-ТУЛЫ (`iva-write`), но **нет скилла-моста «черновик документа → публикация в Confluence/Jira через iva-write»**. `fr-authoring` (в analysis, которого у техписа нет) техпису и не нужен — он не про FR. `doc-authoring` покрывает контент/структуру, но НЕ оркестрацию публикации. Это КОНТЕНТ-пробел, НЕ блокер сборки/тестов (тесты скиллы не проверяют). Решение (свой write-скилл техпису vs расширение doc-authoring) — за ответственной ролью, не мной.

## 6. Тест `apps/backend/tests/catalog/test_iva_role_presets.py`

`ROLE_LANES` (стр. 40-49) — dict `role_id → список depends_on точь-в-точь`. Рядом `ROLE_PERSONA` (стр. 51) — dict `role_id → ожидаемый persona.role`.

Прогоняемые параметризованные проверки (по ключам ROLE_LANES):
- `test_manifest_validates_against_v2_schema` (68) — Draft7 против v2-схемы;
- `test_role_is_pure_composition_leaf` (73) — `profile_id`, `schema_version=="2"`, `deprecated is False`, `ingredients==[]`, `persona.role == ROLE_PERSONA[role_id]`;
- `test_depends_on_is_the_declared_lanes_in_order` (86) — `depends_on` строго == список + каждый лейн `manifest.yaml` существует на диске;
- `test_lanes_are_depth1_bases` (93) — ни один лейн не несёт `depends_on`;
- `test_single_owner_lanes_are_pairwise_disjoint` (104) — попарная дизъюнктность ingredient_id;
- `test_golden_parity_union_equals_sum_of_lanes` (115) — union == Σ размеров лейнов;
- `test_ide_targets_claude_and_codex_full` (131) — claude-code/codex `full`, copilot `unsupported`.

**Что добавить в тест:** 3 записи в `ROLE_LANES` (architect/qa → `[tacticum-core-base, iva-analysis-base, iva-write-base]`; techwriter → `[tacticum-core-base, tacticum-documentation-base, iva-write-base]`) + 3 записи в `ROLE_PERSONA` (значения persona.role новых ролей). Больше нигде роли перечислять не нужно — глобального теста-энумератора всех шаблонов нет (проверил `test_manifest_schemas.py` — не итерирует templates/; `scripts/check_profile_version_discipline.py` итерирует, но владеет версиями).

## Что нужно для каждой из 3 ролей

Каждой роли: каталог из 3 файлов (`manifest.yaml` + `README.md` + `CHANGELOG.md`) по образцу `iva-role-analyst`, `ingredients: []`, `ide_targets` claude-code/codex=full, copilot/opencode/gemini=unsupported.

- **iva-role-architect** — `depends_on: [tacticum-core-base, iva-analysis-base, iva-write-base]`, `persona.role: architect`.
- **iva-role-qa** — `depends_on: [tacticum-core-base, iva-analysis-base, iva-write-base]`, `persona.role: qa` (QA-контент = tests-authoring + helm-analyst.requirement_tests из analysis).
- **tacticum-role-techwriter** — `depends_on: [tacticum-core-base, tacticum-documentation-base, iva-write-base]`, `persona.role: techwriter`.
- Тест: +3 записи в `ROLE_LANES` и `ROLE_PERSONA`.

## Риски

- **Дизъюнктность** — чисто для всех трёх (в т.ч. analysis∩write=∅). Блокеров нет.
- **enum persona** — enum'а нет ни в v2, ни в v1; правки схемы НЕ нужны. Не блокер.
- **qa-контент** — отдельного qa-lane нет и не нужен; покрыто analysis (tests-authoring + helm-analyst). Не блокер.
- **техпис-скилл** — контент-пробел: у техписа есть write-ТУЛЫ, но нет скилла-моста публикации; `doc-authoring` его не покрывает. Не блокер сборки/тестов; решение за ответственной ролью.
- **⚠️ СЕКВЕНС-БЛОКЕР (главный):** `iva-write-base` живёт только в ветке `feat/iva-write-base`, в `main` его НЕТ. Все три роли ссылаются на него в `depends_on`. `test_depends_on_is_the_declared_lanes_in_order` требует наличия `templates/iva-write-base/manifest.yaml` на диске. Пока write-base не влит в рабочее дерево — тесты новых ролей упадут (missing lane). Сборку ролей делать в ветке/дереве, где iva-write-base уже присутствует.
- **Замечание (не риск):** architect и qa имеют ИДЕНТИЧНУЮ композицию лейнов (обе = core+analysis+write) → ingredient-идентичны, различаются только `persona.role`, `name`, README. Тесты это не ломает (роли проверяются независимо). Соответствует плану; если разведение ожидалось иным — подсветить владельцу.

## Связано

- [[План- три деливеравла ИВА (iva-write - 3 роли - my_todo) — привязка к системе]]
- [[Дизайн-спека iva-write-base (Вариант A) — на апрув]]