---
title: qa-two-layers — два слоя QA, что готово, зачем ждём 3 субагента
type: report
permalink: tacticum/20-architecture/qa-two-layers-dva-sloia-qa-chto-gotovo-zachem-zhdiom-3-subagenta
status: current
role: lead (тимлид)
date: 2026-07-21
tags:
- qa
- role-presets
- architecture
- analysis
- tacticum-dev
---

# QA-профиль: два слоя, готовность, чего ждём и почему

Разбор по вопросу «можно ли уже сделать QA и зачем ждём данные». Синтез из [[explore-qa-autotest-skills]] + чтения 3 заблокированных SKILL.md + docs архива. База для ГД/Diaret и для запроса к QA-команде.

## Ключ: два РАЗНЫХ слоя QA 

**Слой 1 — тест-ДИЗАЙН (авторинг TC + покрытие).** Уже есть в tacticum-dev, полный:
- `tests-authoring` (в `iva-analysis-base`) — авторинг тест-кейсов контрактного уровня (GIVEN/WHEN/THEN).
- `helm-analyst.requirement_tests` — покрытие требования автотестами + статус из Allure.
- Данных хватает, работает сегодня. Делал возможным плейсхолдер `iva-role-qa = core+analysis+write`.
- **docs из архива описывают именно этот слой** (Qwen генерит тест-кейсы → IVA QA Agent → TestOps, PoC −75%).

**Слой 2 — автотест-КОД (писать/гонять/чинить Selenium-pytest).** 9 новых скиллов QA-команды. Другая, более глубокая способность — генерация и исполнение кода теста на живом UI one-web.

→ «Почему опять не хватает»: **не тот же QA стал сложнее — мы перешли от QA-авторинга (слой 1, готов) к QA-автоматизации кода (слой 2, новый)**, у которого свои зависимости. Данные не терялись — замах на более мощный профиль.

## Чего не хватает и зачем (3 субагента = движок генерации)
3 скилла (`write-autotest`, `batch-autotest`, `fix-failed-test`) делегируют всю умную работу 3 субагентам через `Task`; их defs (agent_spec) в архиве НЕТ:

| Субагент | Что делает | Без него |
|---|---|---|
| codebase-analyst | инвентарь репо one-web (page-objects/chunks/helpers/i18n/фикстуры) | тест не знает ГДЕ/КАК писаться |
| dom-explorer | реальные XPath/локаторы в живом UI (playwright-cli), держит AUT_OVERVIEW.md | нет локаторов — тест не кликает нужное |
| code-writer | пишет локаторы/методы/i18n в page-objects | код теста не создаётся |

`write-autotest` = оркестратор: `TC → codebase-analyst → dom-explorer → code-writer → tests/test_*.py`. Убери любого — генерация рушится. Это НЕ тестовые данные, а определения агентов. Есть только у QA-команды.

## Можно ли уже сделать QA — да, 3 варианта
- **Вариант 1 — QA авторинга/покрытия (слой 1), готов сейчас, ноль недостающего.** `core+analysis+write`. Пишет TC, меряет покрытие, публикует. Минус: это по сути работа аналитика (развилка «кто генерит TC») + переоснащён всем analysis.
- **Вариант 2 — QA автотестов частично (~6/9 скиллов), готов сейчас.** `core+iva-qa-autotest-base+write`. Работает без субагентов: прогон существующих тестов (run-tests, playwright-cli), MR-ветки (prepare-mr-branch), rebuild-autocore, retro, ручной поиск локаторов. НЕ работает: генерация (write/batch) и авто-починка (fix).
- **Вариант 3 — полный (все 9), нужны 3 субагента.** Генерация→прогон→починка→публикация→MR. Цель QA-команды: KPI автотест 8ч→≤1.5ч (5×), покрытие 20%→≥60%.

## Как QA-профиль ДОЛЖЕН работать (целевой end-to-end)
1. Аналитик авторит TC (tests-authoring) [или Qwen по docs] — «что».
2. QA: `TC` → write-autotest → [codebase-analyst → dom-explorer → code-writer] → `tests/test_*.py`.
3. run-tests / playwright-cli → прогон на стенде one-web (at-test-*-one).
4. fix-failed-test → воспроизводит, классифицирует (DOM_CHANGED/flaky/product-bug), чинит.
5. Публикация → allurectl / tools.testops → Allure TestOps (allure.iva.ru, project 5); jira-issue-autotest оркестрит e2e по Jira.
6. prepare-mr-branch → чистая MR-ветка (glab) → GitLab CI.
7. batch-autotest масштабирует; rebuild-autocore/retro — инфра.

## Серьёзная оговорка — репо-специфичность
Даже с субагентами этот QA работает ТОЛЬКО для one-web / web-стека: хардкод инфры (`/Users/oc4kxb/…`, пакет autocore, стенды at-test-*-one.ivcs.su, tools/testops, GitLab CI). iOS/KMP = отдельные лейны (см. [[qa-profile-model]]).

## ЗАПРОС К QA-КОМАНДЕ (точная формулировка)
Прислать **определения 3 субагентов** (то, что вызывается через `Task` из write-autotest/batch-autotest/fix-failed-test):
- `codebase-analyst`, `dom-explorer`, `code-writer` — их системные промпты / инструкции (как оформлены у них: `.claude/agents/*.md` или аналог), включая модель, набор инструментов, режимы (write/fix).
Без них 3 из 9 скиллов (генерация и авто-починка автотестов) нерабочи. Формат приёма в лейн готов: `ingredients/agents/<id>.md` + `kind: agent_spec` в manifest (слот описан в README лейна).

## Рекомендация лида
Главный анлок — **получить 3 agent_spec** от QA-команды → Вариант 3 + валидация 1-в-1 ([[qa-validation-plan]]). Промежуточно можно раздать Вариант 2 (прогон/поддержка существующих автотестов), если полезно тестировщикам.

## Связано
- [[qa-profile-model]] · [[qa-validation-plan]] · [[explore-qa-autotest-skills]] · [[qa-profil-trek-b-gotovo-built-verified-flagi-dlia-gd]] · [[plan-qa-profile-obogatit-iva-role-qa-realnymi-qa-skillami-trek-b-lidu]]