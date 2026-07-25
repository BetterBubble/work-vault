---
title: impl-qa-mcp-thin-lanes
type: report
permalink: tacticum/00-board/impl-qa-mcp-thin-lanes
tags:
- lead-qa
- implementer
- qa-kit
- ADR-0057
- draft
---

# impl: три тонких per-role MCP-лейна + возврат ролей в чистую композицию (вариант B)

status: draft
worktree: ~/tacticum/tacticum-dev-qa-kit, ветка feat/qa-kit-subagents (НЕ мержил/не пушил)

## Что сделано

Вариант B из брифа: восстановлен инвариант ADR-0057 «роль = pure-composition leaf (ingredients==[])»,
сломанный переносом MCP прямо в роль-пресеты (15 падений test_iva_role_presets.py). MCP вынесен
в тонкие per-role лейны (прецедент структуры — tacticum-core-base).

### 1. Три новых MCP-лейна (manifest v2 + README + CHANGELOG, 0.1.0)
- **iva-qa-mcp** — 2 mcp_server_spec: `iva-atlassian-write` (stdio uvx mcp-atlassian, jira.iva.ru, личный PAT; allowed_tools = Jira defect/status) + `helm-analyst` (http helm.tacticum.ru/mcp/analyst, Bearer=phk_*; allowed_tools = READ-срез для ревью: requirement_tests, gap_questions, contradiction_check, nearest_spec, affected_systems, constraints, related_tasks). Закрывает сигнал lead-arch 13:00.
- **iva-architect-mcp** — 1 mcp_server_spec `iva-atlassian-write`: Confluence page-authoring + полный Jira issue-ops.
- **iva-techwriter-mcp** — 1 mcp_server_spec `iva-atlassian-write`: Confluence page-authoring + jira_add_comment.

### 2. Роли → чистая композиция
- iva-role-qa: own iva-atlassian-write убран → `ingredients: []`; depends_on += iva-qa-mcp (0.3.0→0.4.0). Из non_goals снято исключение покрытия requirement_tests — QA теперь ЧИТАЕТ факт-базу через helm-analyst (реверс Track-B-исключения).
- iva-role-architect: → `ingredients: []`; depends_on += iva-architect-mcp (0.2.0→0.3.0).
- tacticum-role-techwriter: → `ingredients: []`; depends_on += iva-techwriter-mcp (0.2.0→0.3.0).
- README/CHANGELOG всех трёх обновлены.

### 3. Тест-канон
test_iva_role_presets.py: ROLE_LANES — снесённый iva-write-base заменён на новые MCP-лейны. Все зелёные (pure-composition, depth1, single-owner, golden-parity).

### 4. Фикс моделей субагентов (Codex, профиль 0)
iva-qa-autotest-base: 3 agent_spec (codebase-analyst/dom-explorer/code-writer) — снят хардкод `model: opus` → `model: gpt-5.4` (конвенция codex-моделей репо, ветка chore/codex-models-gpt5x-migration). README/CHANGELOG/нарратив manifest обновлены.

## Проверка (пройдено)
- `pytest test_iva_role_presets.py` — 35/35 зелёные (venv 3.12 + jsonschema/pyyaml, --noconftest).
- `pytest test_manifest_schemas.py` — зелёные.
- Валидатор manifest.v2 + ingredient.v1: iva-role-qa/architect/techwriter + 3 MCP-лейна + iva-qa-autotest-base — ВСЕ VALID; каждая роль ingredients==[].
- `grep -rn iva-write-base templates/` = 0; helm-analyst больше не исключён в non_goals QA.
- DB-backed catalog тесты (test_seed_depends_on, golden и т.д.) НЕ прогнаны: требуют backend-окружения (sqlalchemy/alembic не в изолированном venv). Статически проверено: снесённый лейн / изменённые роли references только в test_iva_role_presets.py (обновлён). Прогнать при доступном backend venv.

## Развилки (разумные дефолты, помечено флагами)
- **ingredient_id MCP-ингредиентов НЕ префиксовал ролью** (бриф предлагал как fallback). Оставил канонические `iva-atlassian-write` / `helm-analyst`, потому что: (а) рендерер claude_code.py маппит mcpServers[ingredient_id] — id = имя MCP-сервера, скиллы зовут mcp__helm-analyst__*; префикс сломал бы неймспейс тулов; (б) переиспользование ingredient_id между профилями — штатная конвенция репо (context7 в 9 манифестах, helm-analyst/tacticum-mcp — в нескольких), НЕ глобально-уникальный ключ. Single-owner держится: каждая роль композит ровно один MCP-лейн, внутри роли пересечений id нет.
- **codex-модель = gpt-5.4** (worker-tier, как coder/tester в codex-миграции; orchestrator-tier = gpt-5.5). metadata.model в agent_spec — просто string; reasoning_effort в этой схеме не выражается (только в codex TOML) — для 3 worker-субагентов не требуется.
- **Нейминг techwriter-лейна = iva-techwriter-mcp** (не tacticum-*): write-канал IVA-специфичен (mcp-atlassian против wiki.iva.ru/jira.iva.ru), консистентно семейству iva-qa-mcp/iva-architect-mcp. Роль остаётся tacticum-role-techwriter.

## Коммиты (ветка feat/qa-kit-subagents)
- 54812e6 templates: три тонких per-role MCP-лейна
- ed8b74b roles: вернуть QA/architect/techwriter в чистую композицию
- fa7d62e test(catalog): актуализировать ROLE_LANES под вариант B
- b866c0f iva-qa-autotest-base: субагенты craft на Codex gpt-5.4 (снят хардкод opus)

git status: clean. Без AI-подписей, без мусора.

## Observations
- [invariant] ADR-0057 pure-composition leaf восстановлен для всех трёх write-ролей #adr-0057
- [convention] ingredient_id = имя MCP-сервера (рендерер mcpServers[id]); id переиспользуется между профилями, не глобально-уникален #mcp
- [decision] QA читает helm-analyst READ-срез для ревью (реверс Track-B-исключения) #qa

## Relations
- implements [[resheniia-po-qa-profiliu-trek-b-2026-07-21]]
