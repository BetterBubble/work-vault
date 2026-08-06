---
title: gate-fix-role-web-coverage
type: report
permalink: tacticum/00-board/gate-fix-role-web-coverage-1
tags:
- gate
- controller
- fix
- role-web
- go
archived-at: 2026-08-03 11:16
---

# Гейт-вердикт: fix/role-web-coverage (чинит красный main)

Контролёр, read-only. Ветка `fix/role-web-coverage` в worktree `/Users/bubblemac/tacticum/tacticum-dev-fix-rolecov` (HEAD e1f8394), НЕ пушено. Целевой профиль-инцидент: PR-A #149 добавил 2 DS-навыка в `iva-web-brownfield`, роль `iva-role-web` их не покрывала → падал `test_role_covers_replaced_profile[iva-role-web<-iva-web-brownfield]`.

## ВЕРДИКТ: GO (push разрешён)

Оба гардрейла PASS, целевой тест зелёный, все 122 падения (2 failed + 120 errors) = DB-инфра (docker postgres не стартует / asyncpg localhost:5432 недоступен) — не регрессия; не-DB структурный набор 100% зелёный; git чист.

## По пунктам

**1. Скоуп/дифф — PASS.** Против **origin/main** (истинная цель интеграции): ровно **4 файла**, ВСЕ в `templates/iva-web-development-base/`, **+381/−1**. Manifest `version: 0.1.0→0.1.1`, запись CHANGELOG `[0.1.1] — 2026-07-24` есть. Brownfield/другие роли/lite/bugfix/analysis не тронуты. _Нюанс:_ дифф против **локального** main даёт 10 файлов, т.к. локальный main отстал на PR#150 (brd-sync mail+rn); origin/main = c6be10a уже содержит PR#150 → merge-base(origin/main,HEAD)=c6be10a, чистая 4-файловая дельта. Ложной тревоги нет.

**2. Fidelity навыков — PASS.** Оба SKILL.md (`angular-ds-component-authoring`, `angular-ds-component-usage`) **байт-в-байт** идентичны версиям в `iva-web-brownfield` (sha256 совпадают: 59a65e6f… и cb1a2e36…). Manifest skill_spec — base-формат `supports: [claude-code, codex]` (без copilot) — ожидаемо для base-лейна, ок.

**3. Целевой тест — PASS.** `test_role_covers_replaced_profile -k "iva-role-web and iva-web-brownfield"` → 2 passed, 82 deselected.

**4. Полный catalog/ — PASS (нет регрессии).** Итог: **549 passed, 2 failed, 120 errors**. Все 120 ERROR = fixture `postgres_url` (docker run postgres:16-alpine → exit 125, нет docker-демона). Оба FAILED (`test_admin_catalog_authoring::test_patch_profile_404`, `test_create_draft_404_unknown_profile`) = asyncpg connect localhost:5432 отказано. Один корень — нет Postgres. Ни одного (schema/role/parity/mirror/логика). Не-DB структурный набор (parity+install_smoke+manifest_schemas+role_presets) — 290 passed, 100% зелёный.

**5. Валидаторы — PASS.** `check_mirror_sync` → OK, 64 зеркальных ингредиента в 6 парах синхронны. `check_profile_version_discipline --diff-against origin/main` → OK, 48 профилей clean.

**6. Git-чисто — PASS.** 0 секретов (grep-хиты «token» = дизайн-токены/zero-hex, не ключи). 0 AI-подписей (хиты `- claude-code` = значения поля `supports:`, легитимно; тело коммита пустое). 0 мусора в диффе. Дерево чистое (`git status` пуст). Автор коммита = идентичность Президента (Александр Шульга). Ветка `fix/role-web-coverage`, не main. _Примечание:_ `.serena/` трекается по всему репо (пред-существующее), но НЕ в диффе этого фикса — не блокер для этого PR.

## Без-протечки (доп.проверка)
Лейн `iva-web-development-base` composes **ТОЛЬКО** `iva-role-web` (grep по всем manifest.yaml). 2 навыка достигают ровно iva-role-web, в другие роли не утекают. `_mirrors.yaml` не тронут → mirror-дисциплина 64/6 без изменений.

## Дальше
Тимлид → OK Президента (через ГД) на push. Push формой `git push origin fix/role-web-coverage`, PR против origin/main. Merge — отдельное ручное действие.