---
title: plan-qa-profile — обогатить iva-role-qa реальными QA-скиллами (Трек B, лиду)
type: note
permalink: tacticum/00-board/plan-qa-profile-obogatit-iva-role-qa-realnymi-qa-skillami-trek-b-lidu-1
status: APPROVED
created: 2026-07-21 12:40
updated: 2026-07-21 12:40
role: director→lead
autonomy: 'off'
scope: QA-профиль
project: tacticum-dev
tags:
- plan
- approved
- handoff
- qa-profile
- tacticum-dev
- role-presets
---

# plan-qa-profile — Трек B (ГД → лид)

Параллельно с валидацией профиля аналитика (её ведёт ГД с пользователем). autonomy: off — до controller-гейта, мерж — по OK пользователя. Guardrail прежний: **не трогать аналитиков** (iva-analysis-base / iva-role-analyst), работать в worktree, не мержить.

## Цель
Обогатить `iva-role-qa` (сейчас плейсхолдер из существующего контента, лежит в worktree `feat/iva-write-base`) **реальными QA-скиллами команды тестировщиков**.

## Источник скиллов
`~/tacticum/helm/data/iva-qa.rar` — распакуй. Внутри `iva/`:
- **skills/** (9): write-autotest, batch-autotest, playwright-cli, fix-failed-test, run-tests, jira-issue-autotest, prepare-mr-branch, rebuild-autocore, retro — каждый со своими `references/*.md`.
- **docs/**: `iva_qa_transformfation.html`, `poc_results.html` — **прочитай их первыми**: там пайплайн QA и результаты PoC (как они реально работают: Playwright + Allure/TestOps, автотесты).

## Что сделать
1. **Изучи** 2 docs + скиллы → пойми пайплайн QA (это автотест-автоматизация: генерация/прогон/починка автотестов через Playwright, публикация в Allure TestOps, MR-ветки).
2. **Дизайн композиции (ADR-0057, single-owner):** эти автотест-скиллы — QA-специфичны и НЕ должны дублироваться → оформи **отдельным лейном** (напр. `iva-qa-autotest-base`), которым владеет `iva-role-qa` через `depends_on`. НЕ клади в analysis/core.
   - Предлагаемый состав `iva-role-qa` = `[tacticum-core-base, iva-qa-autotest-base, iva-write-base]` (+ возможно analysis ради requirement_tests/tests-authoring — реши по факту, не нарушая single-owner).
3. **Скопируй** скиллы в `templates/iva-qa-autotest-base/ingredients/skills/<id>/SKILL.md` (+ references), напиши `manifest.yaml` лейна (по образцу iva-analysis-base), README, CHANGELOG.
4. **Перепиши** `iva-role-qa/manifest.yaml` на новую композицию; обнови `apps/backend/tests/catalog/test_iva_role_presets.py` (ROLE_LANES) — должны пройти single-owner-disjoint / golden-parity / depth1.
5. **Провижн-проверка** + отчёт на доску.

## Открытая развилка (НЕ решай сама — флагни)
«Кто генерит тест-кейсы — аналитик (tests-authoring в analysis) или QA?» На созвоне TBD. **Пока:** аналитик держит tests-authoring (постановка TC), QA-лейн = **исполнение/автоматизация** автотестов (Playwright/Allure). Не переноси tests-authoring в QA без решения.

## Замечания
- Многие QA-скиллы завязаны на реальные тулы (Playwright CLI, TestOps API, pytest, MR-ветки) — на уровне лейна это `skill_spec` (тексты); MCP/тулчейн QA (если нужен свой mcp_server_spec для TestOps) — отметь в отчёте, отдельно.
- Не выдумывай — если пайплайн из docs неясен, флагни вопросы в отчёте.

Отчёт: что за лейн собрал, состав iva-role-qa, зелёные ли тесты, открытые вопросы. Мержа не делай — жди OK.

---

## 🔄 ПОЛНАЯ ЗАДАЧА QA (обновление ГД, 2026-07-21) — читать лиду

Первый проход принят (ГД проверил): лейн `iva-qa-autotest-base` (9 скиллов) + `iva-role-qa` = `[core, iva-qa-autotest-base, iva-write-base]` (Вариант 1), оба флага задокументированы, **тест композиции 35 passed** (ГД прогнал). Ниже — весь оставшийся скоуп QA с контекстом.

### Опись (зафиксируй в README лейна + отчёте)
| Что | Состав | Статус |
|---|---|---|
| Скиллы рабочие (6) | playwright-cli, run-tests, jira-issue-autotest, prepare-mr-branch, rebuild-autocore, retro | ✅ |
| Скиллы заблокированы (3) | write-autotest, batch-autotest, fix-failed-test | ⛔ ждут defs |
| Субагенты (3, ОТСУТСТВУЮТ) | codebase-analyst, dom-explorer, code-writer | ❌ запрошены у QA-команды |
| MCP | iva-write (из iva-write-base) + tacticum-mcp/context7 (core) | отдельного QA-MCP нет |
| Allure/TestOps | локальные бинари allurectl/glab/playwright-cli + модуль tools/testops в one-web | локально, не MCP |
| Окружение | репо one-web (autocore-venv, tools/testops, .gitlab-ci) | внешняя зависимость лейна |

### Модель мульти-стэк (задокументируй — база под iOS/KMP QA)
- **web-only — НЕ ограничение дизайна**, а следствие: скиллы, что дала команда, жёстко на web-тулинг (Playwright/Selenium/pytest/autocore/glab). 
- **Один общий лейн исполнения невозможен** — браузерная автоматизация ≠ нативное мобильное тестирование, тулинг принципиально разный.
- **Другие стэки = отдельные лейны** со скиллами их команд: `iva-qa-ios-autotest-base` (XCUITest), `iva-qa-kmp-autotest-base` (Espresso/KMP-test). Роль per-stack: `iva-role-qa-ios = core + iva-qa-ios-autotest-base + iva-write` и т.д.
- **Общее/стэк-агностичное (НЕ дублировать):** авторинг тест-кейсов (`tests-authoring` в analysis) + покрытие (`requirement_tests`) — шарятся всеми QA-ролями.
- Расширение на iOS/KMP = **получить автотест-скиллы тех команд** (как получили web). Пока их нет → web-only = реальный скоуп.

### Задачи СЕЙЧАС (не ждут defs)
1. **Отчёт на доску** (его не хватало): что собрал, состав role-qa, тест 35 passed, 2 флага.
2. **Задокументировать опись + мульти-стэк-модель** (README лейна + короткая заметка `20-Architecture/qa-profile-model` — пригодится Diaret и под iOS/KMP).
3. **Подготовить слот под defs:** структура лейна готова принять 3 `agent_spec` (codebase-analyst/dom-explorer/code-writer) — опиши в README, куда именно они лягут и что после этого 3 скилла станут рабочими.
4. **Подготовить план валидации QA 1-в-1** (по образцу валидации аналитика, что ведёт ГД): как провижнить iva-role-qa, взять реальный тест-кейс, прогнать генерацию/прогон/починку автотеста на one-web, сверить достоверность. Готовый чек-лист — чтобы как придут defs, валидировать быстро.

### Когда придут defs — план
1. Добавить 3 defs как `agent_spec` в `iva-qa-autotest-base` (скопировать, прописать в manifest, single-owner — не коллидить).
2. Пересобрать + прогнать тест композиции.
3. **Валидация 1-в-1** (провижн → реальный TC → генерит/гоняет/чинит автотест → сверка «зелёное ≠ верное»).
4. Только тогда — раздача QA.

### Guardrail
worktree, autonomy off, **не мержить** (жди OK пользователя), аналитиков не трогать. Открытая развилка «кто генерит TC (аналитик vs QA)» — держим на аналитике, не переносить без решения Diaret.

Отчёт по каждому пункту — на доску.