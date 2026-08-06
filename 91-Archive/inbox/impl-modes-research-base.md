---
title: impl-modes-research-base
type: note
permalink: tacticum/00-board/impl-modes-research-base-1
tags:
- draft
- impl
- lead-modes
- research-lane
- adr-0057
archived-at: 2026-07-31 17:27
---

# impl-modes-research-base

status: draft
Implementer-отчёт. Направление lead-modes (ТЗ#2 Солонко). Создан новый процесс-лейн `tacticum-research-base` (режим research) с нуля, структурно 1:1 по образцу `tacticum-bugfix-base`.

Worktree: /Users/bubblemac/tacticum-worktrees/modes-workflow, ветка `feat/workflow-modes`, коммит **dbe976f** (только новый каталог, 5 файлов, +417 строк, без пуша).

## Что создано (пути)
- `templates/tacticum-research-base/manifest.yaml` — base-манифест (schema_version "2", **НЕТ depends_on**), version **0.1.0**, maintainer mr.diaret@ya.ru, ide_targets claude-code=full/codex=full/остальные=unsupported (как в bugfix). 2 ингредиента.
- `templates/tacticum-research-base/ingredients/commands/start-research.md` — тонкая команда входа `/start-research <вопрос>` (аналог fix-bug.md): запуск в top-level сессии без спавна суб-агента (та же Codex-MCP-мотивация), маркер завершения `RESEARCH_COMPLETE`.
- `templates/tacticum-research-base/ingredients/skills/research/SKILL.md` — процедура: Phase0 frame → Phase1 search KB cross-repo (no clone) → **Phase2 verify every claim against code (честное «не найдено, проверял там-то»)** → Phase3 два документа (research-report + adr-draft, шаблоны внутри) → Phase4 outcome/handoff. Стэк-агностично, кода не пишет.
- `templates/tacticum-research-base/README.md` + `CHANGELOG.md` — provenance (ТЗ#2 §1/§1.2 п.4/§3/§6 + сценарий /start-research), состав, выходы во 2-й слой.

## Ключевые решения
- **ID ингредиентов различны: skill `research` ≠ command `start-research`** — как `bug-fix`≠`fix-bug` у bugfix. Задача формулировала оба как «start-research», но два одинаковых ingredient_id в одном лейне = дубль (ломает within-lane uniqueness/golden-parity при попадании в роль). Слэш-команда осталась `/start-research` (command metadata.name="start-research"), skill назван `research` (dir skills/research/SKILL.md). Триггеры/тело ссылаются на /start-research.
- **Факт-инструменты опциональны** (helm-analyst/iva-read/arch_*): используются ЕСЛИ в роли, иначе от KB каталога — зафиксировано в манифесте, SKILL Phase1 и README (ТЗ §1.2 п.4).
- **Выход во 2-й слой** (§3): «кодить не надо» → стоп; известное решение → /lite-task с планом; нужна реализация → /start-task ADR-first (черновик ADR на вход, межролевой handoff через wiki/Jira) — в SKILL Phase4 таблицей.
- Guardrail соблюдён: лейн создан как **orphan**, НЕ добавлен ни в один depends_on/ROLE_LANES; никакие существующие манифесты/лейны/_mirrors/тесты не тронуты.

## Результат прогона тестов (docker DOWN, postgres 5432 недоступен)
- **Схема-валидация нового манифеста** (jsonschema Draft7, uv-venv): manifest.v2 OK; оба ингредиента против ingredient.v1 OK; id уникальны; depends_on отсутствует; body_path-файлы существуют — **всё OK**.
- **`tests/catalog/test_manifest_schemas.py` + `tests/catalog/test_iva_role_presets.py`: 129 passed за 1.17s** (0 failed). Второй — file-level валидация всех role-манифестов + инвариантов композиции: подтверждает, что новый лейн НЕ ломает роли (он в ROLE_LANES не значится — по дизайну).
- **Полный `tests/catalog/` (env-справка): 547 passed, 2 failed, 120 errors.** Все 120 errors + 2 failed — **средовые**: db-тесты требуют docker-postgres (5432 down; ошибки Connect call failed / httpx ASGI→starlette→db). Падающие 2 (`test_admin_catalog_authoring::test_patch_profile_404`, `::test_create_draft_404_unknown_profile`) — HTTP-интеграция к БД, к templates-файлам отношения не имеют. Ни одного падения, связанного с моими файлами.

## Находки / риски для тимлида
- **Orphan-лейн НЕ ломает file-level тесты.** В репо НЕТ теста, который свипает templates/ и требует, чтобы каждый лейн был в роли: `test_iva_role_presets.py` итерирует только явный dict `ROLE_LANES`, а `seed_depends_on`/composition-тесты используют синтетические payload'ы, не реальные templates. Поэтому orphan прошёл валидацию чисто. Добавление `tacticum-research-base` в depends_on ролей (ROLE_LANES в test_iva_role_presets.py:44, role-манифесты, тест-матрица) — **отдельный шаг тимлида/ГД**, я его не делал (guardrail).
- **Docker/postgres в среде воркера недоступны** — seed-тайм тесты (test_seed_depends_on, test_composition, admin_catalog HTTP) прогнать нельзя. Реальная seed-валидация нового манифеста против БД (canonical hash, edge-freeze) — при следующем прогоне на машине с docker. Схема-уровень я закрыл локально.

## Что осталось (вне моего объёма)
- Врезка лейна в роли (dev + analyst) через ROLE_LANES/role-манифесты + правка `test_iva_role_presets.py` (ROLE_LANES, ROLE_PERSONA) — тимлид.
- Seed-тесты с docker-postgres — при интеграции.
- Перевод/адаптация RU-промптов под поверхности CLI перед выкаткой (общее для ТЗ#2).