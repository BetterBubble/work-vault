---
title: 'recon-adr-profiles: РЕ-разведка на актуальных ветках (0057/0059 + qa-kit)'
type: note
permalink: tacticum/00-board/recon-adr-profiles-re-razvedka-na-aktualnykh-vetkakh-0057-0059-qa-kit
tags:
- board
- draft
- adr
---

# recon-adr-profiles: РЕ-разведка на актуальных ветках

status: draft
Репо: /Users/bubblemac/tacticum/tacticum-dev
Ветки: `origin/feat/profiles-lanes-roles-architecture` (каноника архитектуры), `feat/qa-kit-subagents` (живая QA-работа).
Чтение через `git show <ref>:<path>` — worktrees не трогались.

## ЧАСТЬ 1 — ADR-фундамент (arch-ветка)

### ADR-0059 (одна ось: процессы-лейны + паки уровня роли), Proposed, 2026-07-21
- **Инвариант:** `Роль = core + процесс-лейны + стек-лейн(ы) + способность-лейны`. Механизм — ADR-0056 (depth-1, single-owner, override, frozen-ребро), лейны в лейны НЕ вкладываются.
- **4 типа кирпича:** core (ровно один: `tacticum-core-base`) · процесс (`iva-analysis-base`, `tacticum-development-core`, `tacticum-bugfix-base`, `tacticum-documentation-base`) · стек (`iva-*-development-base`) · способность (`tacticum-ui-base`). Роль = единица установки (`iva-role-*`).
- **Решение 5 (уточнение инварианта 0057):** роль несёт РОВНО пак-ингредиенты (`claude-md-fragment`, `codex-agents-md`, `codex-config-toml`, `claude-settings`) с маркером `tacticum:<role>` — по одному на роль. «роль = 0 ингредиентов» → «роль = только пак-kinds». Весь контент — из лейнов.
- **Решение 6 (implement-only):** dev-роли собираются БЕЗ analysis и БЕЗ documentation. `iva-role-go` сведена к `[core, development-core, development-go, bugfix]` — analysis удалён (22.07). Полный цикл — отдельной fullcycle-ролью по надобности. **Это отменяет матрицу ADR-0057 D3, где iva-role-go имел analysis+documentation.**
- Реализация — dev#119: стек-лейны kmp/web/mail/ios 0.1.0, роли 0.1.0/0.1.1, development-core 0.1.0.

### ADR-0057 (process-lanes + role-presets), Accepted, 2026-07-18
- **D1:** единица композиции — процесс-лейн (base, один процесс) + роль (тонкий leaf-пресет, `depends_on` набора лейнов + core).
- **D2 лейны:** core (context7+tacticum-mcp+conventional-git), analysis (постановка + analyst-MCP helm-analyst/iva-read), go-development (impl-агенты+serena+go-skills), bugfix, documentation, ui (playwright).
- **D4:** артефакты постановки single-owner в `analysis` → нет override.
- **Addendum 2026-07-20:** мост helm↔Jira фазный. MVP — линк через iva-read + interim write-канал (E-FRQA #699). **Целевка — отдельный `iva-write` MCP с личным PAT** (артефакты под своей учёткой). Taiga #712.
- **Механизм скоупинга тул-поверхности ЕСТЬ в схеме:** `templates/_schema/ingredient.v1.schema.json` для `mcp_server_spec` поддерживает `allowed_tools: [string]` и `required_scopes: [string]`. То есть сузить сервер под роль можно на уровне ингредиента — делить сам mcp-сервер не требуется, скоупинг делается декларативно.

## ЧАСТЬ 2 — карта role→base→MCP→allowed_tools

### MCP-объявления по базам
| База (ветка) | MCP-сервер | allowed_tools? |
|---|---|---|
| tacticum-core-base (обе) | context7 | НЕТ (весь сервер) |
| tacticum-core-base (обе) | tacticum-mcp | НЕТ (весь сервер) |
| iva-analysis-base (arch) | helm-analyst | **НЕТ** (весь сервер) |
| iva-analysis-base (arch) | iva-read | **НЕТ** (весь сервер) |
| iva-analysis-base (arch) | iva-atlassian-write (interim, mcp-atlassian stdio) | НЕТ (весь сервер) |
| iva-analysis-base (QA) | helm-analyst, iva-read ТОЛЬКО | НЕТ (весь сервер) — write вынесен |
| iva-write-base (QA) | **iva-write** (http) | **ДА**: `[confluence_create_page, confluence_update_page, jira_create_issue, jira_add_comment, jira_transition_issue]` + `required_scopes: [iva-req-write]` |
| tacticum-development-core (arch) | serena (uvx stdio) | НЕТ (весь сервер) |
| iva-kmp-development-base (arch) | codegraph | **ДА**: `[codegraph_explore]` |
| iva-mail-development-base (arch) | codegraph | **ДА**: `[codegraph_explore]` |
| iva-go/web/ios-development-base | (нет своего MCP — serena из development-core) | — |
| tacticum-ui-base (arch) | playwright (@playwright/mcp) | НЕТ (весь сервер) |
| tacticum-platform-base (arch) | arch (mcp.tacticum.ru/arch) | **ДА**: `[get_applicable_standards, check_compliance, search_artifacts]` + env TACTICUM_MCP_TOKEN |
| iva-qa-autotest-base (QA) | **нет MCP вообще** (9 skills + агенты codebase-analyst/dom-explorer/code-writer + команды, привязан к репо one-web) | — |

### Роли → результирующий набор MCP
| Роль | ветка | depends_on | MCP (скоуп) |
|---|---|---|---|
| iva-role-analyst | arch | core + analysis | context7, tacticum-mcp, helm-analyst(весь), iva-read(весь), iva-atlassian-write(весь) |
| iva-role-go | arch | core + development-core + go-dev + bugfix | context7, tacticum-mcp, serena(весь). **Analysis-MCP НЕТ** (implement-only) |
| iva-role-kmp | arch | core + development-core + kmp-dev + bugfix + ui | +serena, +codegraph(scoped), +playwright(весь) |
| iva-role-web/ios | arch | core + development-core + <stack>-dev + bugfix + ui | context7, tacticum-mcp, serena, playwright |
| iva-role-mail | arch | core + development-core + mail-dev + bugfix + ui | +codegraph(scoped) |
| tacticum-role-internal | arch | core + internal-base + bugfix | context7, tacticum-mcp |
| tacticum-role-platform | arch | core + platform-base + bugfix | +arch(scoped) |
| **iva-role-qa** | QA | core + iva-qa-autotest-base + **iva-write-base** | context7, tacticum-mcp, **iva-write(scoped)**. Автотест-лейн без MCP |
| **iva-role-architect** | QA | core + iva-analysis-base + iva-write-base | context7, tacticum-mcp, helm-analyst(весь), iva-read(весь), iva-write(scoped) |
| **tacticum-role-techwriter** | QA | core + tacticum-documentation-base + iva-write-base | context7, tacticum-mcp, iva-write(scoped) |

## ЧАСТЬ 3 — асимметрия скоупинга: ПОДТВЕРЖДЕНА
- **helm-analyst и iva-read — весь сервер** (без allowed_tools) в `iva-analysis-base` на ОБЕИХ ветках.
- **iva-write — сужен** через `allowed_tools` (5 тулов) + `required_scopes: [iva-req-write]` в `iva-write-base` (QA-ветка).
- Паттерн шире: read-знания (helm-analyst, iva-read) и локальные dev-инструменты (serena, playwright) — весь сервер; write/действие-каналы и внешние compliance (iva-write, platform arch) — сужены. Исключение: codegraph в kmp/mail тоже сужен до `codegraph_explore`.

## Дельта vs старая разведка
- **Подтвердилось:** iva-role-qa = core + qa-autotest + write (`depends_on: tacticum-core-base, iva-qa-autotest-base, iva-write-base`); helm-analyst живёт в iva-analysis-base без allowed_tools.
- **Поехало (главное):** `iva-write` теперь — first-class ОПУБЛИКОВАННЫЙ leaf `iva-write-base` на ветке `feat/qa-kit-subagents` (не «неопубликованный worktree»). Полный набор новых ролей (architect/qa/techwriter) — там же.
- **Разъезд по write-каналу между ветками:** arch-ветка всё ещё несёт interim `iva-atlassian-write` (mcp-atlassian, весь сервер) ВНУТРИ `iva-analysis-base`; QA-ветка вынесла write ИЗ analysis в отдельный скоупленный `iva-write-base`. analysis на QA-ветке = только helm-analyst + iva-read.
- **iva-qa-autotest-base НЕ несёт MCP** — это 9 скиллов + агенты (codebase-analyst/dom-explorer/code-writer) + команды (write-autotest/batch-autotest/run-tests/fix-failed-test/jira-issue-autotest/prepare-mr-branch/rebuild-autocore/retro), привязан к репо one-web. «MCP QA» как отдельного сервера нет; QA пишет через iva-write-base.
- **tc-review — НЕ команда/не MCP:** только reference-документ `iva-qa-autotest-base/ingredients/skills/write-autotest/references/tc-review.md`.
- **iva-role-go стал implement-only** (ADR-0059 Решение 6, 22.07): `[core, development-core, go-dev, bugfix]`, analysis+documentation убраны — матрица ADR-0057 D3 (go с analysis) на arch-ветке уже неактуальна.
