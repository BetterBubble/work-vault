---
title: explore-qa-vs-analyst-final
type: explore
permalink: tacticum/00-board/explore-qa-vs-analyst-final
tags:
- explore
- lead-qa
- qa-profile
- analyst
- onboarding
---

# explore-qa-vs-analyst-final

status: draft
role: explorer → lead-qa
repo: /Users/bubblemac/tacticum/tacticum-dev-qa-kit
QA-ветка: feat/qa-kit-subagents · Аналитик-источники: origin/main
Проверено: чтение манифестов (Read) + git show origin/main. Модель субагентов QA = gpt-5.4 подтверждена в manifest.

## Источники (пути)
- QA роль: `templates/iva-role-qa/manifest.yaml` (ветка feat/qa-kit-subagents)
- QA MCP-лейн: `templates/iva-qa-mcp/manifest.yaml`
- QA автотест-лейн: `templates/iva-qa-autotest-base/manifest.yaml` (9 skill + craft-stack + 3 agent_spec + 2 config)
- QA субагенты: `templates/iva-qa-autotest-base/ingredients/agents/{codebase-analyst,dom-explorer,code-writer}.md`
- База: `templates/tacticum-core-base/manifest.yaml` (context7 + tacticum-mcp + 4 skill)
- Аналитик роль: `git show origin/main:templates/iva-role-analyst/manifest.yaml`
- Аналитик лейн: `git show origin/main:templates/iva-analysis-base/manifest.yaml`
- FR-аналитик: `git show origin/main:templates/iva-fr-analyst/manifest.yaml`

## Deliverable 1 — Карта финального QA-профиля (iva-role-qa v0.4.0)

**Композиция (depends_on, pure-composition leaf, ingredients: []):**
1. tacticum-core-base — репо-биндинг, KB-навигация, онбординг, git-конвенции, базовый MCP (context7 + tacticum-mcp)
2. iva-qa-autotest-base — автотест-лейн (9 скиллов + craft-stack + 3 субагента), привязан к репо one-web
3. iva-qa-mcp — MCP-лейн (iva-atlassian-write write Jira-дефектов + helm-analyst READ-срез)

**9 скиллов (iva-qa-autotest-base):**
- write-autotest (trial) — 1 TC (CSV .tcs / TestOps URL / текст) → pytest/Selenium; делегирует субагентам
- playwright-cli (trial, +codex) — поиск локаторов/XPath в живом UI через CLI-бинарь
- run-tests (trial) — прогон pytest в браузере + архив allure-raw, зовёт fix при падениях
- fix-failed-test (trial) — разбор pytest-падений, batch-правки локаторов/методов/импортов
- batch-autotest (full) — пачка TC из фильтра TestOps end-to-end (PROGRESS.md/metrics.jsonl)
- jira-issue-autotest (full) — e2e: Jira ISSUE → автотесты → пайплайн glab → резолв ланча Allure
- prepare-mr-branch (full, +codex) — cherry-pick в чистую MR-ветку, 3 блока для MR
- rebuild-autocore (full, +codex) — пересборка пакета autocore в venv (PostToolUse-хук)
- retro (full, +codex) — housekeeping агентной инфры (ledger/metrics/бюджеты)
- (+ craft-stack — референс-библиотека стека, не user-навык)

**3 субагента (agent_spec, model gpt-5.4 — Codex):**
- codebase-analyst — read-only инвентарь page-слоя/flows/i18n/enum + пробелы; tools [Read, Glob, Grep]
- dom-explorer — разведка живого UI, единственный с браузером, пишет locators.md; tools [Bash, Read, Write, Glob, Grep]
- code-writer — генерация кода тест-слоя по analysis.md+locators.md, py_compile; tools [Read, Edit, Write, Glob, Grep, Bash]

**MCP и скоуп:**
- iva-atlassian-write (stdio, uvx mcp-atlassian, Server/DC jira.iva.ru, личный PAT) — env только JIRA_URL+JIRA_PERSONAL_TOKEN (без CONFLUENCE_*). allowed_tools = jira_create_issue, jira_update_issue, jira_add_comment, jira_get_transitions, jira_transition_issue. Только дефекты/статусы Jira.
- helm-analyst (http helm.tacticum.ru/mcp/analyst, Bearer phk_*) — READ-срез, allowed_tools = requirement_tests, gap_questions, contradiction_check, nearest_spec, affected_systems, constraints, related_tasks (7). Без write, без полного C4 (arch_map/arch_container/requirement_coverage).
- context7 + tacticum-mcp (kb_*) — из tacticum-core-base.
- НЕТ iva-read/iva-mcp; в автотест-лейне MCP нет вовсе (Allure TestOps через tools/testops в репо one-web).

**Покрытие этапов workflow:**
- Генерация ТС→код · прогон · починка · публикация (Allure+MR) · дефекты (Jira) = ЕСТЬ (полный автотест-конвейер)
- Валидация + дополнение ТС = read-опора есть (helm-analyst.requirement_tests + write-autotest/references/tc-review.md), flow в парке
- Покрытие требований = только ЧТЕНИЕ (helm-analyst.requirement_tests), не владение

**Провайдер/модель:** субагенты craft — Codex gpt-5.4. Роль CLI: claude-code=full, codex=full (⚠ автотест-лейн на codex реально best-effort — Task-субагенты/Skill()/хуки Claude-специфичны).

**Чего профиль НЕ делает (non_goals):** авторинг тест-кейсов/тест-дизайн (это аналитик); разработка продуктового кода; постановка требований (BRD/FR/ADR); авторинг FR-страниц в Confluence (write сужён до Jira-дефектов); владение покрытием.

## Deliverable 2 — Сравнение QA ↔ Аналитик

| Ось | QA (iva-role-qa, feat/qa-kit-subagents) | Аналитик (iva-role-analyst + iva-fr-analyst, origin/main) |
|---|---|---|
| Назначение | Исполнитель: автоматизация автотест-кода + заведение дефектов + ревью-сверка | Тест-дизайн/авторинг ТС + постановка требований (BRD/UC/ADR/PIN/TC/API-контракт) |
| Композиция | core-base + iva-qa-autotest-base + iva-qa-mcp | core-base + iva-analysis-base (роль); FR-агент = standalone |
| Скиллы | 9 автотест: write/batch-autotest, playwright-cli, run-tests, fix-failed-test, jira-issue-autotest, prepare-mr-branch, rebuild-autocore, retro | analysis-base 8: fr-authoring, api-contracts-discovery, brd/adr/multi-container-pin/tests-authoring, pin-api-verification, pin-upstream-dependency-check · fr-analyst 4: fr-authoring, api-contracts-discovery, design-system-discovery, mockup-authoring |
| Владелец ТС | ЧИТАЕТ покрытие (tests-authoring НЕ входит) | ВЛАДЕЕТ tests-authoring / requirement_tests |
| Субагенты | 3 craft (Codex gpt-5.4): codebase-analyst, dom-explorer, code-writer | analysis-base 2 (opus): system-analyst, tacticum-workflow (design-оркестратор) · fr-analyst — 0 субагентов |
| helm-analyst | READ-срез, 7 тулов (requirement_tests/gap_questions/contradiction_check/nearest_spec/affected_systems/constraints/related_tasks) | ПОЛНЫЙ сервер (без allowed_tools): +C4 arch_map/arch_container, requirement_coverage, docs_ask, RAG#2 |
| atlassian-write | iva-atlassian-write, только Jira (5 jira_* тулов), env без CONFLUENCE_* → дефекты/статусы | iva-atlassian-write, Jira+Confluence (env +CONFLUENCE_*), без allowed_tools → авторинг+публикация FR |
| iva-read/iva-mcp | НЕТ | ЕСТЬ (iva-read в analysis-base; iva-mcp read в fr-analyst) |
| tacticum-mcp | из core-base | из core-base (роль); свой в fr-analyst (standalone) |
| Модель/провайдер | Codex gpt-5.4 (субагенты); роль codex=full (лейн best-effort) | субагенты opus; fr-analyst без субагентов; роль codex=full |
| Ownership | владеет автотест-кодом + дефектами Jira; читает ТС/покрытие | владеет ТС/тест-дизайном/покрытием/требованиями + публикацией FR (Confluence) |

**Стык:** аналитик авторит ТС и владеет покрытием (requirement_tests) → QA берёт готовые ТС, автоматизирует их в код, а покрытие/контекст лишь ЧИТАЕТ через тот же helm-analyst (узкий срез). Дефекты — зона QA (Jira write); Confluence/FR-публикация — зона аналитика.

## Не подтверждено / флаги
- iva-role-analyst В origin/main ЕСТЬ (подтверждено git show) — компонует core-base + iva-analysis-base, НЕ iva-fr-analyst. fr-analyst — отдельный standalone-профиль (без core-base, свой tacticum-mcp).
- «Ёмкость/модель аналитика»: analysis-base agent_spec model: opus (2 агента). fr-analyst субагентов не несёт — работает скиллами+командами+MCP.
- helm-analyst — физически один и тот же сервер (helm.tacticum.ru/mcp/analyst) у обеих ролей; разница ТОЛЬКО в allowed_tools (QA сужён до 7 read-тулов, аналитик — полный).
