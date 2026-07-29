---
title: explore-профили-ролей-tacticum-dev
type: note
permalink: tacticum/00-board/explore-profili-rolei-tacticum-dev
tags:
- explore
- tacticum-dev
- profiles
- draft
archived-at: 2026-07-29 18:12
---

# explore-профили-ролей-tacticum-dev

status: draft
Разведка (read-only + git pull) по устройству профилей ролей в `TacticumApps/tacticum-dev`. Цель — понять, как добавить профили: техпис, техподдержка, QA, дизайнер (mockup).

## Пул
- Репо `/Users/bubblemac/tacticum/tacticum-dev`, ветка `main`. HEAD отставал от `origin/main` на 76 коммитов; дерево чистое → сделал `git pull --ff-only origin main`, прошло чисто. Появились новые профили `tacticum-dev-base`, `tacticum-ui-base`, `tacticum-internal-dev`, `tacticum-platform-dev`.

## Где живут профили
- Каталог: `templates/<profile-id>/` (kebab-case). Один подкаталог = один профиль.
- Внутри: `manifest.yaml` (метаданные + список ингредиентов), `ingredients/` (тела: agents/, agents-codex/, agents-copilot/, commands/, skills/, rules/, repo-configs/), `tests/` (smoke .ps1), `README.md`, `CHANGELOG.md`.
- Схемы: `templates/_schema/manifest.v2.schema.json`, `templates/_schema/ingredient.v1.schema.json`.
- Существующие профили: iva-go-backend-brownfield, iva-web-brownfield, iva-rn-brownfield, iva-kmp-brownfield, iva-ios-brownfield, iva-brownfield-mail, iva-2-client-shell-dev, firebird-web-brownfield, iva-system-analyst, iva-fr-analyst, brownfield-task-workflow(deprecated); базы: tacticum-dev-base, tacticum-ui-base; tacticum-internal-dev, tacticum-platform-dev.

## Анатомия (manifest v2, ADR-0023)
Верхний уровень: schema_version:"2", profile_id, name, version, maintainer, license, deprecated, `depends_on`(опц), description, persona{role,scope}, target_tasks[], stack{required,optional}, ide_targets{claude-code/codex/copilot/opencode/gemini: full|best-effort|unsupported}, profiles{trial,full}, ingredients[], post_install_notes, non_goals.
- Manifest v2 schema минимальна (required только schema_version); реальная валидация ингредиентов — ingredient.v1.
- 9 kind'ов ингредиента (ADR-0020): instruction_pack, rule_set, agent_spec, skill_spec, mcp_server_spec, command_spec, hook_spec, permission_policy, repo_config.
- Ингредиент: ingredient_id, kind, tier(trial|full), supports[cli], install_scope(user|repo), body_path|body|codex_body_path|copilot_body_path, target_path_template + per-cli target_path, metadata{}.

## Как регистрируется
- Реестра-списка нет: профиль = наличие `templates/<id>/manifest.yaml`. CI на merge в main сидит в Postgres-каталог: `apps/backend/scripts/seed_community.py` → `backend.catalog.application.seed_profile.seed_profile`. `depends_on`-рёбра фризятся на сиде (ADR-0056). Раздаётся подписчикам через catalog-mcp (`tacticum_init`/`tacticum_fetch_action`).

## Композиция (ADR-0056)
- `depends_on: [tacticum-dev-base, tacticum-ui-base]` — один уровень (база сама без depends_on), shallow-merge по ingredient_id, профиль побеждает базу. Стековый профиль несёт только дельту.
- tacticum-dev-base = stack-agnostic ядро (KB-nav, BRD/ADR/PIN/TESTS, команды /start-task…/run-*, MCP: context7+serena+tacticum-mcp). tacticum-ui-base = UI/дизайн-кластер.

## Рецепт «добавить профиль»
Skill `profile-authoring` (`.claude/skills/profile-authoring/SKILL.md`, копия в `.agents/skills/`). Фазы: 0 клон+kb_discover → 1 досье фактов → 2 `<id>-task.md` + апрув owner → 3 US в Taiga, worktree, coder→tester, тест-инварианты (пересечение ingredient_id с базами == объявленные override; grep-гейт чужих стеков; identity-тест без пина; e2e_install golden; quickstart в docs/user_manuals; .gitattributes eol=lf) → 4 сид на VPS + provision + ручная приёмка owner.
Минимум для нового профиля: каталог templates/<id>/ + manifest.yaml (depends_on на базы, дельта-ингредиенты) + ingredients/ тела + README + CHANGELOG + tests + строка в .gitattributes.

## Дизайн/mockup — что уже есть
- tacticum-ui-base: skills design-system-discovery, design-token-usage, pin-ui-pipeline-check, ui-mockup-match (playwright MCP). MOCKUPS-артефакт генерит инлайн оркестратор tacticum-workflow в Phase 1 (BRD/MOCKUPS/ADR/PIN/TESTS). ui-mockup-match — СВЕРКА рантайм-UI с утверждёнными MOCKUPS, НЕ генерация с нуля.
- design_* MCP tools premium-gated (ADR-0028): design_list_systems, design_get_tokens, design_resolve_token, design_get_theme_tokens.
- Отдельного skill/agent «генерация макета из ТЗ» и Figma-интеграции в репо НЕТ. Профиль дизайнера — greenfield-дельта поверх ui-base.

## Связь с KB / гейт KB-discovery
- MCP `tacticum-mcp` (https://mcp.tacticum.dev/mcp, bearer TACTICUM_TOKEN) даёт kb_*-тулы. Гейт в агенте tacticum-workflow.md («Run Discovery»): читать `.tacticum/context.yaml`→installation_id → `kb_discover(installation_id)` РОВНО один раз → kb_run_id → все kb_* с этим id. Если kb_discover падает — STOP, просить подключить MCP, локального fallback НЕТ (hard rule). Это и есть блокер «KB discovery» со скринов.
- Ключевые файлы: `templates/iva-go-backend-brownfield/ingredients/agents/tacticum-workflow.md` (Run Discovery), `templates/tacticum-dev-base/ingredients/skills/kb-navigation/SKILL.md` (таблица kb_*), `.../repo-configs/claude-code/CLAUDE.md.fragment`.

## Для 4 новых профилей (гипотеза направления, не реализация)
- analyst-подобные (iva-system-analyst, iva-fr-analyst) НЕ используют depends_on/dev-base — это самостоятельные single-tier профили без coder/tester, MCP helm-analyst+iva-mcp+tacticum-mcp. Техпис/техподдержка/QA ближе к этому классу (аналитический, без кодового конвейера) либо дельта над dev-base.
- Дизайнер — дельта над tacticum-ui-base + новый skill генерации макетов (+ возможно Figma MCP, которого в репо пока нет).
