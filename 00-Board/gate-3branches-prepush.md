---
title: gate-3branches-prepush
type: note
permalink: tacticum/00-board/gate-3branches-prepush
tags:
- gate
- controller
- prepush
- us4
- modes
---

# Гейт: 3 ветки к push (controller, read-only)

Дата: 2026-07-24. origin/main = 6c1c0ea (зелёный, fix #151 внутри). Все 3 ветки смёржены с origin/main (merge-base = 6c1c0ea).

## Метод (урок регрессии PR-A)
На КАЖДОЙ ветке прогнан ПОЛНЫЙ `pytest apps/backend/tests/catalog/` (не подмножество). Каждое падение классифицировано. Docker/Postgres :5432 в среде НЕТ → DB-инфра-падения ожидаемы и не являются регрессией.

## Baseline падений (одинаков на всех 3 ветках И это DB-инфра)
- **120 errors** — fixture `postgres_url`: `docker run postgres:16-alpine` → exit 125 (docker недоступен). Модули: repository/admin/migrations/seed/tenant_visibility/composition/init/builtins/owner_amendment.
- **2 failed** — `test_admin_catalog_authoring.py::test_patch_profile_404` и `::test_create_draft_404_unknown_profile` — SQLAlchemy connection refused (500 вместо 404 без БД). Идентичны на всех ветках.
- Итого 549 passed / 2 failed / 120 errors на каждой ветке. Реальных (schema/role/parity/mirror/логика) падений — 0.

## Ветка 1 — feat/us4-passC-canon-starttask (C-канон) → GO
- Скоуп: ровно 3 файла в `tacticum-dev-base` (start-task.md, CHANGELOG.md, manifest.yaml). brd-authoring НЕ в диффе. bump 0.2.6→0.2.7 (CHANGELOG [0.2.7] совпадает).
- Не-DB критичный набор (parity/schemas/role_presets/install_smoke/dev_base_profile/stack_residue): **304 passed, 0 fail**.
- mirror-sync (`test_mirror_content_is_byte_identical`) PASS; version-discipline PASS.
- git чист: 0 секретов, 0 AI-подписей, feature-ветка (не main), worktree чист.

## Ветка 2 — feat/us4-passB2 (B2) → GO
- Скоуп: ровно 16 файлов в 4 профилях (firebird-web-brownfield / iva-brownfield-mail / iva-ios-brownfield / iva-rn-brownfield), каждый: pin-authoring/SKILL + tests-authoring/SKILL + CHANGELOG + manifest. Ничего вне (brd/start-task/base/analysis/web/kmp/роли не тронуты).
- Версии: mail 0.7.2→0.7.3, rn 0.5.2→0.5.3, ios 0.1.2→0.1.3, firebird 0.1.2→0.1.3 (все CHANGELOG top-entry совпадают).
- Не-DB набор (parity/schemas/role_presets/install_smoke + 3 профильных теста): **331 passed, 0 fail**. `test_role_covers_replaced_profile` (стр.157 parity) + coverage/mirror subset **84 passed** — контент-правки покрытие НЕ сломали.
- git чист (совпадение "token" в диффе = русский текст про токены К-2, не секрет). 0 AI-подписей.

## Ветка 3 — feat/workflow-modes-infra (§1.2) → GO
- Скоуп: ровно 3 файла в `tacticum-lite-base` (lite-task-workflow/SKILL.md, CHANGELOG.md, manifest.yaml). Ничего вне (analysis/bugfix/роли/ROLE_LANES/_mirrors не тронуты). bump 0.1.2→0.1.3 (CHANGELOG [0.1.3] совпадает).
- Полный catalog: те же 2 failed / 120 errors = DB-инфра. Реальных 0. (заявленные modes 213 passed — их профильный subset; полный catalog = 549 passed.)
- Не-DB набор: **290 passed, 0 fail**. parity/mirror PASS.
- git чист: 0 AI-подписей (несмотря на amend), feature-ветка, worktree чист.

## Итоговая таблица
| ветка | вердикт | не-DB тесты | падения (полный catalog) | скоуп | git |
|---|---|---|---|---|---|
| feat/us4-passC-canon-starttask | **GO** | 304 passed / 0 fail | 2 failed + 120 err = DB-инфра | чист (3 файла dev-base, brd не тронут) | чист |
| feat/us4-passB2 | **GO** | 331 passed / 0 fail (coverage OK) | 2 failed + 120 err = DB-инфра | чист (16 файлов, 4 профиля) | чист |
| feat/workflow-modes-infra | **GO** | 290 passed / 0 fail | 2 failed + 120 err = DB-инфра | чист (3 файла lite-base) | чист |

## Вердикт (дословно, по каждой ветке)
Все три: **оба гардрейла PASS, не-DB набор 100% зелёный, все падения = DB-инфра, скоуп чист, git чист.**

## Оговорка контролёра
Тесты гонялись через общий venv основного репо (editable-импорт `backend`); templates резолвятся из worktree (`REPO_ROOT = parents[4]` от тест-файла), src ветками не тронут — импорт из main корректен. DB-гейты (Postgres/docker) в этой среде проверить нельзя — считаю DB-инфра-падения не-регрессией по факту идентичности baseline. Финальное подтверждение DB-путей — на CI с Postgres.
