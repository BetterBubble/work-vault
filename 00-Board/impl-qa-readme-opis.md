---
title: impl-qa-readme-opis
type: note
permalink: tacticum/00-board/impl-qa-readme-opis
status: draft
role: implementer
track: B
tags:
- implementer
- qa
---

# impl-qa-readme-opis

Дополнил документацию лейна `iva-qa-autotest-base` (Трек B, задачи ГД 2 и 3). Только `.md` в лейне. Analysis не тронут, не провижнил/не пушил/не мержил.

## Worktree / ветка / коммит
- Worktree: `/Users/bubblemac/tacticum/tacticum-dev-iva-write`, ветка `feat/iva-write-base`
- Коммит: `8a7e854` — `docs(qa): опись скиллов, слот под 3 agent_spec, мульти-стэк в README iva-qa-autotest-base`
- 2 файла: `templates/iva-qa-autotest-base/README.md`, `.../CHANGELOG.md`

## Что дописал в README (3 новых раздела перед «Установка / поддержка CLI»)
1. **«Опись скиллов»** — таблица 6 рабочих ✅ (playwright-cli, run-tests, jira-issue-autotest, prepare-mr-branch, rebuild-autocore, retro) / 3 заблокированных ⛔ (write-autotest, batch-autotest, fix-failed-test — ждут defs) + строки: субагенты (3 отсутствуют), MCP (свой QA-MCP отсутствует; iva-write из iva-write-base + tacticum-mcp/context7 из core), Allure/TestOps (allurectl/glab/playwright-cli + tools/testops в one-web, локально не MCP), окружение (репо one-web: autocore-venv/tools.testops/.gitlab-ci). Синхронизировано с [[qa-profile-model]].
2. **«Слот под 3 agent_spec»** — таблица кто из каких скиллов зовёт codebase-analyst/dom-explorer/code-writer через Task; куда лягут файлы, что прописать в manifest, что после этого 3 скилла станут рабочими. Файлы НЕ создавал — только слот.
3. **«Мульти-стэк»** — web (one-web) = этот лейн; iOS/KMP = отдельные лейны (iva-qa-ios-autotest-base/iva-qa-kmp-autotest-base) со скиллами их команд; стэк-агностичное (tests-authoring/requirement_tests в analysis) не дублировать. Ссылка на [[qa-profile-model]].

## Формат agent_spec — СВЕРЕНО (не выдумка)
Сверил по существующим agent_spec в `templates/iva-analysis-base` (system-analyst, tacticum-workflow) и по `templates/_schema/ingredient.v1.schema.json` (kind:agent_spec → metadata.required = model, description; опц. tools/permissions_ref/delegation).
- Тело субагента = **один плоский markdown-файл** `ingredients/agents/<ingredient_id>.md` — НЕ каталог `<id>/AGENT.md` (в подсказке задачи была развилка — она снята, верный вариант плоский `.md`).
- Опц. codex-тело: `ingredients/agents-codex/<ingredient_id>.toml` + `codex_target_path` (нужно только если supports включает codex).
- В manifest.yaml — запись `kind: agent_spec` с `target_path_template: ".claude/agents/{ingredient_id}.md"`, `body_path: ingredients/agents/<id>.md`, `metadata: {model, tools, description}`.
- В README пример-заготовка дана; `model`/`tools`/`description` помечены **«требует сверки»** с присланными defs QA-команды.

## CHANGELOG
Вписал док-дополнение в **существующую 0.1.0** (подраздел «Docs»), а НЕ завёл 0.1.1. Причина: `version` живёт в `manifest.yaml`, а его трогать нельзя (guardrail «только .md»); лейн ещё не смержен/не зарелизен — весь WIP под 0.1.0, так нет рассинхрона changelog↔manifest.

## Вопрос по формату (для лида/пользователя)
- Патч-бамп версии (0.1.0 → 0.1.1) под док-дополнение **не сделан** намеренно — это правка `manifest.yaml` (не `.md`), вне моего скоупа. Если по конвенции репо нужен явный патч-бамп при доп-документации — это отдельная однострочная правка манифеста, оставляю решение лиду.

## Связано
- [[qa-profile-model]]
- [[plan-qa-profile-obogatit-iva-role-qa-realnymi-qa-skillami-trek-b-lidu]]
- [[impl-qa-autotest-base]]