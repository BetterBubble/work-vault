---
title: 'verify-факты: ADR модель профилей'
type: note
permalink: tacticum/00-board/verify-fakty-adr-model-profilei
tags:
- board
- draft
- adr
- verify
archived-at: 2026-07-31 04:54
---

# verify-факты: ADR модель профилей (аналитик↔QA↔dev + скоупинг MCP)

status: draft
verifier: worker-verifier (read-only)
дата: 2026-07-23
источники: git-репо /Users/bubblemac/tacticum/tacticum-dev (origin/main, origin/feat/profiles-lanes-roles-architecture, feat/qa-kit-subagents) + helm-сервер /Users/bubblemac/tacticum/helm/src/helm/interface/mcp/analyst_server.py

Проверено по источнику (не по тексту ADR). Итог: 6 из 8 пунктов **confirmed**; 1 пункт (тул-реестр) confirmed с полным перечнем; **1 материальное расхождение** по QA-пресету requirement_tests.

## Таблица: утверждение → вердикт → доказательство

| # | Утверждение ADR | Вердикт | Доказательство (команда/источник) |
|---|---|---|---|
| 1 | `allowed_tools`+`required_scopes` в схеме `ingredient.v1.schema.json` на origin/main | **confirmed** | `git show origin/main:templates/_schema/ingredient.v1.schema.json` — в блоке `mcp_server_spec.metadata` поля `allowed_tools` (array of string) и `required_scopes` (array of string), строки 102–103 |
| 2 | Механизм РЕАЛЬНО используют `iva-brownfield-mail`/`iva-kmp-brownfield`/`iva-rn-brownfield`/`tacticum-platform-dev` (сужают серверы) | **confirmed** | `git grep -l allowed_tools origin/main -- templates/*` → ровно эти 4 manifest.yaml. Сужают: три iva-*-brownfield → сервер **codegraph** (`allowed_tools: ["codegraph_explore"]`); tacticum-platform-dev → **arch-MCP** (`get_applicable_standards`/`check_compliance`/`search_artifacts`). ⚠️ Уточнение: сужают codegraph/arch, а НЕ helm-analyst — но ADR так и не утверждает; как доказательство «механизм в проде» — верно |
| 3 | helm-analyst в origin/main (iva-analysis-base) объявлен БЕЗ `allowed_tools` (весь сервер) | **confirmed** | `git show origin/main:templates/iva-analysis-base/manifest.yaml` — ingredient `helm-analyst` (стр.245) metadata: transport/url/env_required/auth_type, **без** allowed_tools. `iva-read` (стр.257) — тоже без. Ни один профиль main не скоупит helm-analyst |
| 4 | `iva-atlassian-write` реально в origin/main (прод write-канал, личный PAT) | **confirmed** | `git grep iva-atlassian-write origin/main` — реальный `ingredient_id: iva-atlassian-write` в iva-fr-analyst/manifest.yaml (стр.161): mcp_server_spec, `command: uvx args:[mcp-atlassian]`, env `JIRA_PERSONAL_TOKEN`/`CONFLUENCE_PERSONAL_TOKEN` (личный PAT). Референсится и в iva-system-analyst |
| 5 | ADR-0059 (go implement-only) НЕ в main; в main iva-role-go — полного цикла, несёт analysis; на branch — implement-only без analysis | **confirmed** | main: `docs/adr/` = 0055,0056,0057 (нет 0059). iva-role-go main `depends_on:` включает **iva-analysis-base** (полный цикл → helm-analyst). Ветка profiles-lanes-roles: есть `0059-single-axis-...md` («Решение 6 — dev implement-only»), iva-role-go `depends_on` = core+**tacticum-development-core**+go-dev+bugfix (**без** iva-analysis-base) |
| 6 | `iva-write`/`iva-write-base` нигде в origin (только локально; ветка ГД удалена) | **confirmed** | Скан всех refs/remotes/origin/*: `ingredient_id: iva-write*` — 0 совпадений. В origin/main «iva-write» встречается только как ПРОЗА («целевой iva-write MCP — follow-up Taiga #712»). Как реальный ингредиент/лейн — существует лишь локально (`feat/qa-kit-subagents:templates/iva-write-base/manifest.yaml`), что ADR и признаёт |
| 7 | Список тулов helm-analyst в §2 (19 тулов; универсальный слой + кластеры) — имена реальны, группировка корректна | **confirmed** | `grep @mcp.tool() analyst_server.py` = ровно **19** тулов; все 19 имён из ADR существуют, **выдуманных нет**: analyst_search, analyst_context, docs_ask, requirement_coverage, who_to_involve, my_todo, nearest_spec, gap_questions, affected_systems, related_tasks, constraints, contradiction_check, api_registry_check, contract_check, arch_map, arch_container, arch_drift, effort_hint, requirement_tests. `requirement_tests` — реально Allure TestOps coverage (детерм., без LLM, только чтение). `arch_*` существуют. `effort_hint` — «справка о сроках / lead time, НЕ оценка трудоёмкости» (совпадает с ADR) |
| 8 | Пресеты §2 вычислительно достижимы (тулы есть, роль получает через лейн) | **уточнение** | Аналитик: helm-analyst через iva-analysis-base (main) — достижимо, все тулы реальны ✓. go: пресет из реальных тулов; на main go несёт analysis→helm-analyst есть; на branch (0059) analysis убран→helm-analyst недостижим без предлагаемого read-лейна (ADR §6 п.2 это признаёт) ✓. **QA: НЕ достижимо на живой QA-ветке как есть** — см. расхождение ниже |

## Неточности / расхождения (явно)

**Р1 (материальное) — QA-пресет `requirement_tests` противоречит живой QA-роли.**
ADR §2 утверждает «у QA один свой тул — `requirement_tests`» и даёт QA-пресет `requirement_tests + related_tasks + nearest_spec + affected_systems + constraints` над helm-analyst (также §4.1 шаг 4, §6 п.3). Но живая ветка `feat/qa-kit-subagents` (lead-qa, которую ADR явно считает легитимным источником истины) **прямо исключает** requirement_tests и helm-analyst из QA-роли:
- `iva-role-qa/manifest.yaml`: «покрытие требований (helm-analyst.requirement_tests) **держит аналитик — в эту роль НЕ входят**»;
- non_goals: «покрытие requirement_tests — это лейн iva-analysis-base, роль iva-role-analyst»;
- `iva-qa-autotest-base` non_goals: «requirement_tests — это лейн iva-analysis-base, не этот»;
- `iva-role-qa` `depends_on` = только `tacticum-core-base` + `iva-qa-autotest-base` — **helm-analyst не приходит ни через один лейн**; единственный собственный MCP роли — `iva-atlassian-write`, сужённый до `jira_*`.

Итог: чтобы QA-пресет из §2 стал достижим, ADR должен ДОБАВИТЬ (сужённый) helm-analyst в QA-роль — что противоречит осознанному решению живого QA-дизайна отдать requirement_tests аналитику. ADR подаёт это как факт («один свой тул QA»), хотя источник говорит обратное. Требует согласования с lead-qa.

**Р2 (мелочь, не ошибка) — «сужают серверы» в §1/§2.** Опубликованные профили сужают codegraph и arch-MCP, не helm-analyst. Формулировки ADR корректны (helm-analyst «умеет, но не сужён»), просто стоит держать в уме, что прод-прецедент скоупинга — на других серверах.

## Подтверждено дополнительно (в пользу ADR)
- §3.1/§6 п.1: QA write-канал = прод `iva-atlassian-write` (сужён `jira_*`), `iva-write-base` ретайрится — подтверждено живой QA-веткой.
- helm-analyst на main несут аналитик и iva-role-go (оба через iva-analysis-base) — подтверждено.
- «19 тулов», характеристики requirement_tests (Allure) и effort_hint (lead time, не трудоёмкость) — совпадают с реестром сервера.

## Замечание об аутентичности
Доказательства воспроизводимы указанными read-only командами (`git show/grep/ls-tree`, `grep` по analyst_server.py). Подлинность подтверждает контролёр.
