---
title: 'recon-adr-profiles: карта профилей и MCP по ролям'
type: note
permalink: tacticum/00-board/recon-adr-profiles-karta-profilei-i-mcp-po-roliam
tags:
- board
- draft
- adr
archived-at: 2026-07-31 04:54
---

# recon-adr-profiles: карта профилей и MCP по ролям

status: draft · для lead-arch (ADR «модель взаимодействия профилей»)
Источник (полный набор): `/Users/bubblemac/tacticum/tacticum-dev-iva-write/templates/`
main-репо (`/Users/bubblemac/tacticum/tacticum-dev/templates/`) — ПОДМНОЖЕСТВО: нет iva-role-qa, iva-role-architect, tacticum-role-techwriter, iva-qa-autotest-base, iva-write-base. Есть legacy tacticum-dev-base / tacticum-ui-base / tacticum-platform-dev / tacticum-internal-dev.

## Модель композиции (ADR-0057 + ADR-0056)
- **Роль** = единица УСТАНОВКИ: тонкий leaf-профиль, `ingredients: []`, всё содержимое через `depends_on` (depth-1, порядок = приоритет при коллизии).
- **База (лейн)** = единица АВТОРИНГА: несёт skills/agents/commands/mcp; `depends_on` НЕ имеет (она сама база). Лейны single-owner (непересекающиеся).
- Роль = core + N лейнов.

## Роль → базы (depends_on)
| Роль | depends_on |
|---|---|
| iva-role-analyst | tacticum-core-base, iva-analysis-base |
| iva-role-architect | tacticum-core-base, iva-analysis-base, iva-write-base |
| iva-role-go | tacticum-core-base, iva-analysis-base, iva-go-development-base, tacticum-bugfix-base, tacticum-documentation-base |
| iva-role-qa | tacticum-core-base, iva-qa-autotest-base, iva-write-base |
| tacticum-role-techwriter | tacticum-core-base, tacticum-documentation-base, iva-write-base |

## База → MCP (где объявлен сервер)
| База | MCP серверы (ingredient) | url | allowed_tools в манифесте |
|---|---|---|---|
| tacticum-core-base | context7, tacticum-mcp | mcp.context7.com/mcp · mcp.tacticum.dev/mcp (kb_*) | нет |
| iva-analysis-base | helm-analyst, iva-read | helm.tacticum.ru/mcp/analyst · mcp.tacticum.ru/iva-read/mcp | **нет (весь сервер целиком)** |
| iva-write-base | iva-write | mcp.tacticum.ru/iva-write/mcp | **ДА: 5 тулов + required_scopes [iva-req-write]** |
| iva-go-development-base | serena | локальный uvx (oraios/serena, ide-assistant) | нет |
| iva-qa-autotest-base | — НЕТ MCP — | TestOps через tools/testops в репо + внешние CLI | — |
| tacticum-documentation-base | — НЕТ MCP — | | — |
| tacticum-bugfix-base | — НЕТ MCP — | использует helm-analyst/iva-read из analysis (в роли go) | — |
| tacticum-dev-base (legacy) | context7, serena, tacticum-mcp, helm-analyst, iva-read | монолит до раскола на лейны | нет |
| tacticum-ui-base (legacy) | playwright | depends_on tacticum-dev-base | нет |

iva-write allowed_tools: `confluence_create_page, confluence_update_page, jira_create_issue, jira_add_comment, jira_transition_issue`.

## Эффективный MCP по РОЛИ (после композиции)
| Роль | context7+tacticum-mcp (core) | helm-analyst | iva-read | iva-write | serena |
|---|---|---|---|---|---|
| iva-role-analyst | ✓ | ✓ | ✓ | ✗ | ✗ |
| iva-role-architect | ✓ | ✓ | ✓ | ✓ | ✗ |
| iva-role-go | ✓ | ✓ (analysis+bugfix) | ✓ | ✗ | ✓ |
| iva-role-qa | ✓ | ✗ | ✗ | ✓ | ✗ |
| tacticum-role-techwriter | ✓ | ✗ | ✗ | ✓ | ✗ |

## КЛЮЧЕВОЙ ВОПРОС ADR: helm-analyst (mcp-analyst) — кто подключает и какие тулы
helm-analyst живёт ТОЛЬКО в iva-analysis-base → доступен ролям, которые несут analysis:
- **Подключают: analyst, architect, go.**
- **НЕ подключают: qa, techwriter** (у qa — только iva-write + core; qa-autotest вообще без MCP; у techwriter — documentation + write).

Тулы helm-analyst (по инструкции сервера, полный набор): analyst_search, analyst_context, docs_ask, arch_map, arch_container, affected_systems, requirement_coverage, related_tasks, nearest_spec, who_to_involve, effort_hint, gap_questions, constraints, contradiction_check, api_registry_check, requirement_tests.

Реальная потребность в тулах по ролям (из target_tasks манифестов):
- **analyst / architect** — по сути весь аналитический набор: affected_systems, arch_map/arch_container (C4-сверка), constraints, requirement_coverage, requirement_tests (переиспользование Allure), related_tasks, nearest_spec, who_to_involve, gap_questions, contradiction_check, api_registry_check.
- **go** — узко и через bugfix-лейн: requirement_coverage, related_tasks, constraints (факт-база дефекта). При самостоятельной постановке — те же, что у аналитика.

## Наблюдение (для ADR, не вердикт)
- helm-analyst и iva-read объявлены БЕЗ `allowed_tools` — каждая композирующая роль получает ВЕСЬ сервер целиком (analyst + go видят одинаковый полный toolset, хотя go нужен узкий срез).
- Контраст: iva-write СУЖЕН через `allowed_tools` (5) + `required_scopes`. Асимметрия скоупинга read-факт-базы vs write-канала — центральная точка для ADR о модели взаимодействия профилей.
- helm-analyst и iva-read аутентифицируются одним ключом TACTICUM_TOKEN (phk_*, bearer), как и iva-write; актор для write резолвится gateway (X-Auth-User-Id, ADR-0051/0058).

## Ингредиенты по лейнам (кратко)
- **core**: skills getting-started, kb-navigation, tacticum-context, conventional-git; mcp context7, tacticum-mcp. Агентов/команд НЕ несёт.
- **analysis** (17 ingr): skills fr-authoring, api-contracts-discovery, brd/adr-authoring, multi-container-pin-authoring, pin-api-verification, pin-upstream-dependency-check, tests-authoring; agents system-analyst (opus), tacticum-workflow (opus, design-only); commands start-feature/update-feature/prepare-tz/start-task/approve-docs; mcp helm-analyst, iva-read.
- **write** (1 ingr): mcp iva-write (5 тулов). Skills/agents НЕТ.
- **go-development** (31 ingr): 5 go-skills + 17 vendored cc-skills-golang + golang-documentation; agents coder/tester/test-runner; commands run-implementation/run-coder/run-tester/run-test-runner/setup-code-intelligence; mcp serena.
- **qa-autotest** (9 ingr, все skill): write-autotest, playwright-cli, run-tests, fix-failed-test, batch-autotest, jira-issue-autotest, prepare-mr-branch, rebuild-autocore, retro. MCP НЕТ (привязка к репо one-web). ⚠️ на codex best-effort (Task-субагенты/Skill/хуки — Claude-специфика).
- **documentation** (1 ingr): skill doc-authoring. MCP НЕТ.
- **bugfix** (2 ingr): skill bug-fix + command fix-bug. MCP НЕТ (берёт helm-analyst/iva-read из analysis).
