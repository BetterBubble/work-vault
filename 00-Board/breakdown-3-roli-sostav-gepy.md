---
title: breakdown-3-roli-sostav-gepy
type: note
permalink: tacticum/00-board/breakdown-3-roli-sostav-gepy
status: draft
repo: tacticum-dev
branch: feat/iva-write-base
worktree: /Users/bubblemac/tacticum/tacticum-dev-iva-write
created: 2026-07-22 14:30
author: explorer
tags:
- explore
- iva
- roles
- gaps
---

# Разбор 3 ролей ИВА: состав · что дают · зрелость · гэпы

Разведка leaf-профилей `iva-role-architect`, `iva-role-qa`, `tacticum-role-techwriter` против эталона `iva-role-analyst`. Только чтение. Все роли — тонкие пресеты (ADR-0057): `ingredients: []`, всё содержимое приходит по `depends_on` из base-лейнов.

## Эталон «готового профиля» — iva-role-analyst
- Состав: `[tacticum-core-base, iva-analysis-base]`, persona.role=analyst.
- Наполнение (что делает роль зрелой): **iva-analysis-base = 17 ингредиентов** — 8 skill_spec + 2 agent_spec + 5 command_spec + 2 mcp_server_spec.
  - Скиллы: fr-authoring, api-contracts-discovery, brd-authoring, adr-authoring, multi-container-pin-authoring, pin-api-verification, pin-upstream-dependency-check, tests-authoring.
  - Агенты: system-analyst (research as-is), tacticum-workflow (design-оркестратор требование→BRD/ADR/PIN/TC + Jira-декомпозиция).
  - Команды: start-feature, update-feature, prepare-tz, start-task, approve-docs.
  - MCP: **helm-analyst** (`https://helm.tacticum.ru/mcp/analyst`) + **iva-read** (`https://mcp.tacticum.ru/iva-read/mcp`).
- core добавляет 6: skills getting-started/kb-navigation/tacticum-context/conventional-git + MCP context7/tacticum-mcp.
- Итог: аналитик = ~23 ингредиента, живые агенты+команды+MCP, реальный конвейер постановки. Это планка зрелости.

## iva-role-architect
- **Состав**: `[tacticum-core-base, iva-analysis-base, iva-write-base]`, persona.role=architect.
- **Что даёт**: всё наполнение аналитика (те же 17 из analysis-base — ADR/PIN/UC/TC/контракт, агенты system-analyst+tacticum-workflow, 5 команд) + write-канал iva-write. Собственных arch-скиллов НЕТ.
- **Arch-тулы РЕАЛЬНО приходят**: helm-analyst входит в analysis-base, а значит arch_map / arch_container / affected_systems / constraints / contradiction_check / requirement_coverage / api_registry_check у архитектора ЕСТЬ (через MCP, не через отдельные скиллы).
- **Артефакты**: ADR, мульти-контейнерный PIN, API-контракт, UC, TC + публикация решений в Confluence/Jira (iva-write).
- **Зрелость**: высокая по read-части (= analyst), т.к. переиспользует analysis-base целиком. Дельта над аналитиком = только write-канал (1 MCP). По сути architect сейчас = «analyst + публикация», без собственной арх-специфики.
- **Гэпы до готового**: (1) нет ни одного arch-специфичного скилла/команды, отличающего роль от analyst (напр. C4-review, arch-decision-record поверх arch_map/contradiction_check) — роль содержательно дублирует analyst; (2) iva-write endpoint физически не поднят (ждёт Ф1 ADR-0058) — write не прогоняется вживую; (3) нет дым-прогона/провижн-проверки MCP; (4) CHANGELOG на 0.1.0.

## iva-role-qa
- **Состав**: `[tacticum-core-base, iva-qa-autotest-base, iva-write-base]`, persona.role=qa.
- **Что даёт**: iva-qa-autotest-base = **9 skill_spec** (живой набор, привязан к репо one-web): write-autotest (1 TC→pytest/Selenium), batch-autotest (пачка TC из фильтра Allure), playwright-cli (локаторы/XPath в живом UI), run-tests (прогон + allure-raw), fix-failed-test (разбор падений/починка), jira-issue-autotest (e2e Jira→автотесты→glab-пайплайн→Allure ланч), prepare-mr-branch (чистые коммиты в MR), rebuild-autocore, retro. + write-канал iva-write (дефекты/статусы в Jira/Confluence).
- **Артефакты**: pytest/Selenium автотесты, прогоны в Allure TestOps, MR-ветки, дефекты в Jira.
- **requirement_tests (helm-analyst) — ЕГО НЕТ.** helm-analyst живёт только в analysis-base; QA композит его НЕ включает. Значит покрытие требований автотестами (requirement_tests) в QA-роли отсутствует. В манифесте это заявлено намеренно (Трек B): авторинг TC + requirement_tests отданы аналитику, в non_goals QA прямо прописано.
- **Allure**: чтение TC — да, но не через helm, а как ВХОД скиллов write-autotest/batch-autotest (Allure TestOps URL / .tcs CSV) + AQL в jira-issue-autotest; запись прогонов в Allure TestOps — да (run-tests → allure-raw, ланчи). Т.е. Allure покрыт со стороны исполнения автотестов, а не со стороны аналитики покрытия.
- **Зрелость**: высокая по автотест-части (9 живых скиллов с references) — по объёму сопоставима с analyst. Это самый наполненный из трёх новых лейнов.
- **Флаг из манифеста**: codex заявлен full, но реально автотест-лейн best-effort на codex (опирается на Claude-специфику — Task-субагенты, Skill(), хуки). Стэк жёстко привязан к one-web (autocore/venv/glab/CI) — вне него бессмыслен.
- **Гэпы до готового**: (1) требование созвона «дать QA тул requirement_tests» — НЕ выполнено (helm-analyst не в композите); решить продуктово: оставить в non_goals или добавить helm-analyst read-подмножество в qa-роль; (2) iva-write endpoint не поднят (ADR-0058 Ф1); (3) codex-паритет под вопросом (флаг в манифесте); (4) нет дым-прогона/провижн.

## tacticum-role-techwriter
- **Состав**: `[tacticum-core-base, tacticum-documentation-base, iva-write-base]`, persona.role=techwriter.
- **Что даёт**: documentation-base = **1 skill_spec** doc-authoring (что документировать + структура, стэк-агностично) + write-канал iva-write.
- **Артефакты**: API/contract-доки, README, changelog, арх-заметки + публикация в Confluence/Jira.
- **3 типа доков — покрыты ЧАСТИЧНО.** doc-authoring перечисляет: Контракт/API, README/how-to, Changelog, арх-заметка/ADR, трассировка FT/UC. Мэппинг на «release notes / пользовательская / техническая»: техническая = API+README+арх (есть); release notes ≈ changelog (частично, changelog≠release notes для пользователя); **пользовательская документация как отдельный тип — явно НЕ выделена**.
- **Зрелость**: скелет-минимум. 1 скилл против 8–9 у analyst/qa. Самый недонаполненный профиль: нет команд, нет агента-оркестратора документирования, нет типизации по 3 видам доков.
- **Гэпы до готового**: (1) наполнить documentation-base до паритета — минимум развести 3 типа доков (release notes / user guide / technical) отдельными разделами или скиллами; (2) нет команд/агента (у analyst есть start-*, tacticum-workflow) — техпис не имеет оркестрации; (3) iva-write endpoint не поднят; (4) нет провижн/дым-прогона.

## Сквозной блокер всех трёх ролей
**iva-write-base** несёт РОВНО 1 ингредиент — mcp_server_spec `iva-write` (`https://mcp.tacticum.ru/iva-write/mcp`, scope iva-req-write, allowed_tools: confluence_create_page/update_page, jira_create_issue/add_comment/transition_issue). Никаких skills/agents/commands. По post_install_notes endpoint **физически ждёт Ф1 ADR-0058**; до подъёма — только PoC атрибуции. То есть write-часть всех трёх новых ролей пока не прогоняется вживую.

## Что делать сейчас (по ролям)
- **architect**: (1) добавить в analysis-base или роль хотя бы 1 arch-специфичный скилл/команду поверх arch_map/contradiction_check/constraints, чтобы роль ≠ analyst; (2) дым-прогон helm-analyst arch-тулов (arch_map/arch_container) — они уже доступны; (3) дождаться/смокнуть iva-write.
- **qa**: (1) решение по requirement_tests — включить read-подмножество helm-analyst в qa-роль или зафиксировать отказ (сейчас в non_goals); (2) прогнать реальный автотест-цикл на one-web (write-autotest→run-tests→fix); (3) свести codex-паритет (флаг); (4) смок iva-write для дефектов.
- **techwriter**: (1) нарастить documentation-base — развести 3 типа доков (release notes / пользовательская / техническая) явно; (2) добавить команду/агент документирования (паритет с analyst); (3) смок iva-write.
- **общее**: поднять/смокнуть iva-write endpoint (ADR-0058 Ф1); прогнать провижн (claude mcp list / codex) по каждой роли; синхронизировать CHANGELOG перед мержем.

- [ ] to:director from:explorer 3 роли — тонкие пресеты; architect дублирует analyst (нет своих arch-скиллов), qa зрелый но БЕЗ requirement_tests (helm нет в композите), techwriter скелет (1 скилл, 3 типа доков не разведены); общий блокер — iva-write endpoint не поднят (ADR-0058 Ф1)