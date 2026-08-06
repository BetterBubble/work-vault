---
title: impl-qa-role-atlassian-write
type: report
permalink: tacticum/00-board/impl-qa-role-atlassian-write
status: draft
role: implementer
lane: lead-qa
branch: feat/qa-kit-subagents
worktree: ~/tacticum/tacticum-dev-qa-kit
tags:
- qa
- architect
- techwriter
- iva-atlassian-write
- iva-write-base
- adr-0058
- adr-0057
- implementer
archived-at: 2026-07-31 04:54
---

# impl · Унификация write-канала: 3 роли off iva-write-base → own iva-atlassian-write + снос лейна

Задача (решение ГД / пользователь, расширена в ходе работы): ретайр лейна `iva-write-base` → все три write-роли (`iva-role-qa`, `iva-role-architect`, `tacticum-role-techwriter`) переведены на собственный `mcp_server_spec`-ингредиент `iva-atlassian-write` (локальный mcp-atlassian под личным Atlassian PAT), сужение write-тулов ПО РОЛИ, физический снос лейна. Изолированно в worktree, без merge/push.

## Что сделано

### Write-канал `iva-atlassian-write` (прод-образец из `iva-fr-analyst`, ADR-0058)
Единый `mcp_server_spec`: `transport: stdio`, `uvx mcp-atlassian` (sooperset, Server/DC против wiki.iva.ru / jira.iva.ru), аутентификация ЛИЧНЫМ Atlassian PAT через env, авторство родное. Сужение — через `allowed_tools` (та же форма, что несла прежний write-лейн — не выдумана). Скоуп РАЗНЫЙ по роли:

| Роль | version | allowed_tools | env_required |
|---|---|---|---|
| **iva-role-qa** | 0.2.0→0.3.0 | `jira_create_issue`, `jira_update_issue`, `jira_add_comment`, `jira_get_transitions`, `jira_transition_issue` (только Jira дефекты/статусы) | `JIRA_URL`, `JIRA_PERSONAL_TOKEN` |
| **iva-role-architect** | 0.1.0→0.2.0 | Confluence `create/update_page` + полный Jira issue-ops (create/update/comment/get/transition) | `JIRA_*` + `CONFLUENCE_*` |
| **tacticum-role-techwriter** | 0.1.0→0.2.0 | Confluence `create/update_page` + `jira_add_comment` (линковка); заведение/перевод issue НЕ дан | `CONFLUENCE_*` + `JIRA_*` |

- **QA** — Confluence page-authoring убран (`CONFLUENCE_*` env не выставляется) — авторинг FR не QA-задача.
- **architect** — публикует ADR/PIN/решения (Confluence) + ставит задачи (Jira) → оба семейства.
- **techwriter** — документация (Confluence) + линковка к задачам (Jira comment) → без issue-lifecycle.

Для каждой роли: `depends_on` без write-лейна; обновлены `name`/`description`/`persona.scope`/`target_tasks`/`profiles`/`post_install_notes`/`non_goals`/README/CHANGELOG.

### Автотест-лейн (`iva-qa-autotest-base`)
Ссылки на write-канал роли в README/manifest переключены на `iva-atlassian-write`. Состав самого лейна не тронут (mcp_server_spec = 0).

### Снос лейна
`git rm -r templates/iva-write-base/` (CHANGELOG/README/manifest). Все ссылки предварительно переключены.

## Проверка (прогнал)
- **Схемы (venv 3.12 + jsonschema 4.26):** `iva-role-qa`, `iva-role-architect`, `tacticum-role-techwriter`, `iva-qa-autotest-base` — все **VALID** (manifest.v2); ингредиент `iva-atlassian-write` во всех трёх ролях **VALID** (ingredient.v1); все 16 ингредиентов автотест-лейна VALID.
- **`grep -rn iva-write-base templates/` = 0** (лейн снесён, все ссылки переключены; литеральные упоминания снесённого лейна убраны и из нарратива).

## ⚠️ ФЛАГИ для lead-arch

### 1. Инвариант ADR-0057 «pure-composition leaf» сломан для ВСЕХ 3 write-ролей
Собственный ингредиент в role preset противоречит инварианту «zero own ingredients». Фактически прогнал `apps/backend/tests/catalog/test_iva_role_presets.py` — **15 упавших кейсов** (по 5 на каждую из qa/architect/techwriter):
- `test_role_is_pure_composition_leaf` — ожидает `ingredients == []`;
- `test_depends_on_is_the_declared_lanes_in_order` — `ROLE_LANES` хардкодит старый `depends_on` c `iva-write-base`;
- `test_lanes_are_depth1_bases`, `test_single_owner_lanes_are_pairwise_disjoint`, `test_golden_parity_union_equals_sum_of_lanes` — падают, т.к. `ROLE_LANES` ссылается на снесённый лейн `iva-write-base` (файла больше нет).

Тест **кросс-каттинг** (общий инвариант всех пресетов + `ROLE_LANES`/`ROLE_PERSONA` dict) — правка кодирует архитектурное решение (можно ли пресету нести MCP-ингредиент), поэтому НЕ трогал. **Нужно решение lead-arch:**
- **(A)** принять own-MCP в пресетах → обновить инвариант + `ROLE_LANES` (убрать `iva-write-base`, отразить own-ингредиент); либо
- **(B)** вернуть write отдельным тонким лейном (тогда переигрываю placement — верну `iva-atlassian-write` в лейн).

Бриф явно велел класть ингредиент в роли (вариант A де-факто) — так и сделано; инвариант — не моя зона.

### 2. Скоуп тулов architect/techwriter — разумный дефолт
Наборы даны по ТЕКУЩЕМУ использованию (architect: публикация ADR/PIN + постановка задач; techwriter: документация + линковка). Владельцы ролей / lead-arch пусть сверят, не блокировался (по указанию координатора).

### 3. Прочее
- Возможны другие тесты/сиды, ссылающиеся на `iva-write-base` вне `templates/` (напр. `test_seed_depends_on`, golden-фикстуры) — я их не проверял исчерпывающе (вне брифа-грепа `templates/`). Стоит прогнать полный бэкенд-набор.
- Ретайр-коммит лейна технически вошёл в коммит `a7659c2` (был staged через `git rm` до commit) — связно с миграцией architect/techwriter.

## Артефакты
- Ветка `feat/qa-kit-subagents`, коммиты: `3adff42` (qa ретайр), `f9b4cb0` (autotest ссылки), `a7659c2` (architect+techwriter + снос лейна), `69a8939` (qa нарратив/флаг под унификацию).
- Diff stat (мои 4 коммита): 14 файлов, +376/−285 (в т.ч. −3 файла снесённого лейна).
- Не мержил, не пушил.

## Границы
- Правки только в worktree. Основное дерево не тронуто.
- Тест-инвариант `test_iva_role_presets.py` не менял (решение за lead-arch).
</content>