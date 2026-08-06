---
title: gate-prb-g5
type: note
permalink: tacticum/00-board/gate-prb-g5-1
tags:
- gate
- controller
- prb
- g5
archived-at: 2026-08-03 11:16
---

# Гейт PR-B (G5) — ui-mockup-match Figma numeric-compare

Ветка: `feat/ds-web-mockup-figma`, tip `ff7589c` (merge origin/main + `09e523a` + `00c6edb`). Базлайн origin/main = `6c1c0ea`. Read-only аудит.

## ВЕРДИКТ: GO

Оба гардрейла PASS, не-DB набор 100% зелёный, все падения = DB-инфра (не регрессия), скоуп 3 файла чист, git чист.

## По пунктам

1. **Скоуп (3 файла) — PASS.** Дифф origin/main..HEAD = ровно 3 файла, ВСЕ в `templates/iva-web-brownfield`: `CHANGELOG.md` (+48), `ingredients/skills/ui-mockup-match/SKILL.md` (+153/−8), `manifest.yaml` (+4/−4). Ничего вне профиля: владелец/зеркала (`_mirrors.yaml`)/роли/другие профили/development-base не тронуты. Версия `0.3.0→0.4.0` + запись `[0.4.0]` в CHANGELOG есть. Новых навыков не добавлено (правка содержимого существующего ui-mockup-match) → coverage не затронут.

2. **Полный catalog/ + классификация — PASS (нет новых реальных падений).** `pytest tests/catalog/` = **2 failed, 549 passed, 120 errors**. Классификация КАЖДОГО падения: все 122 = **DB-инфра** (Postgres :5432 недоступен — `OSError: Connect call failed 127.0.0.1:5432 / ::1:5432`). 120 ERROR — падение в DB-фикстуре; 2 FAILED (`test_admin_catalog_authoring::test_patch_profile_404`, `::test_create_draft_404_unknown_profile`) — тот же корень 5432, но всплывает как 500-вместо-404 внутри хендлера (SQLAlchemy connect в request). НИ ОДНОГО падения по schema/role/parity/mirror/логике. PR-B трогает только markdown/yaml одного профиля — путей регрессии нет.

3. **Не-DB структурный набор — PASS (100% зелёный).** parity + role_replacement_parity + iva_role_presets + manifest_schemas + role_install_smoke + все profile-тесты (firebird_web_brownfield, mail, ios, ui-base, dev-base) = **355 passed, 0 failed, 0 error**.

4. **Гардрейлы — PASS.** `check_mirror_sync` = `OK — 64 зеркальных ингредиентов в 6 парах синхронны` (64/6). `check_profile_version_discipline` = `OK — 48 profile(s) clean` (48 clean) — подтверждает дисциплину bump+CHANGELOG.

5. **Git-чистота — PASS.** 0 секретов/.env/ключей/токенов (grep пуст), 0 AI-подписей (claude/generated/co-authored/anthropic — grep пуст), 0 мусора (только md/yaml). Автор коммитов = Александр Шульга. Ветка явная, не main.

6. **Строго-по-ТЗ (реальные поля, не выдуманный API) — PASS.** API реален: `design_get_tokens`/`design_resolve_token`/`get_variable_defs`/`installation_id` встречаются в 6 sibling-навыках профиля (tacticum-context, design-token-usage, design-system-discovery, kb-navigation, angular-ds-component-usage). Токены-якоря реальны в `design-systems/iva-web/tokens.json` и числа в SKILL точны: `gap.mail-chips=6`, `radius.control-m=10`, `padding.content-area-sidebar=16`, `bg.primary→{solid.gray.100}` (light). Граница «не blind pixel-diff»: каждое число (ΔE CIEDE2000 ≤2.0, ±2px) измеряется против resolved-значения именованного токена; критик-фикс убрал bbox-семпл из скриншота в пользу computed-style (`browser_evaluate`) + детект активной темы. Where no token bound → «no token anchor / unanchored», не выдумывает.

## Урок PR-A учтён
Прогнан ПОЛНЫЙ catalog/, каждое из 122 падений классифицировано → все DB-инфра, не регрессия.