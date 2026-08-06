---
title: explore-qa-autotest-skills
type: note
permalink: tacticum/00-board/explore-qa-autotest-skills-1
status: draft
role: explorer
track: B
tags:
- explorer
- qa
- role-presets
archived-at: 2026-08-03 11:16
---

# explore-qa-autotest-skills

Разведка исходников QA-команды под дизайн лейна `iva-qa-autotest-base` (Трек B). Read-only, ничего не правил.
Линк на план: [[plan-qa-profile — обогатить iva-role-qa реальными QA-скиллами (Трек B, лиду)]]

Источник: `/private/tmp/claude-501/-Users-bubblemac-tacticum/048dfc9a-e975-420e-80f0-d87e7b55d956/scratchpad/iva-qa/iva/` (2 docs + 9 skills, каталога агентов НЕТ).

## Пайплайн (важное расхождение docs vs skills)

**2 docs описывают ТЕСТ-ДИЗАЙН, а не автотесты.** `iva_qa_transformfation.html` (бизнес-кейс -60% штата) + `poc_results.html` (PoC на 2 боевых задачах IVA One: IVAONE-11392, IVAONE-11207). PoC-механика: `Требования → Qwen-3.5 (генерит тест-КЕЙСЫ, ~10-20 мин) → IVA QA Agent (сервис http://10.6.0.219:8000/, zero-touch импорт в Allure TestOps, ~20 мин) → QE валидирует (~35 мин)`. Цифры PoC: тест-дизайн 7ч → 1ч5м (**-75%**), при масштабе ~2500 ч/мес высвобождается. KPI-цели: автотест 8ч→≤1.5ч (5×), покрытие регрессии ~20%→≥60%. Allure TestOps на `allure.iva.ru` (project 5), интеграция TestOps↔Jira `integrationId=5`.

**9 skills описывают ДРУГОЙ слой — автотест-КОД** (pytest/Selenium на репо `one-web`, локаторы через `playwright-cli`, пакет `autocore` в venv). "IVA QA Agent" из PoC в скиллах НЕ используется. Цикл автотестов по скиллам:
- **Генерация:** `write-autotest` (1 TC) / `batch-autotest` (набор из TestOps-фильтра). Вход: CSV `.tcs`, URL TestOps, или текст → нормализуется в `.tasks/tc-{id}-input.md` → пишет `tests/test_<feature>.py`. Оба делегируют 3 субагентам.
- **Локаторы:** `playwright-cli` — обёртка над CLI-бинарём `playwright-cli` (open/run-code/snapshot/click/fill/tracing/state-save…), поиск XPath в живом UI.
- **Прогон:** `run-tests` — pytest в браузере, архивирует `./allure-raw` → `./allure-raw-<browser>`, при падениях зовёт `fix-failed-test`.
- **Починка:** `fix-failed-test` — вход: имя теста / stack trace / `allure-raw` дир / URL ланча Allure; batch по всем не-зелёным; правит локаторы/методы/импорты (не трогает `autocore` в site-packages). Самый тяжёлый: 15 references.
- **Оркестрация e2e:** `jira-issue-autotest` — `Jira ISSUE + VCSAT-* → batch-autotest → prepare-mr-branch → glab ci run (пайплайн) → резолв ланча → развилка fix`. Связь TC↔Jira резолвится ЧЕРЕЗ TestOps (AQL `issue="<ISSUE>"`), в саму Jira не ходит.
- **MR-ветки:** `prepare-mr-branch` — cherry-pick проектных коммитов из «грязной» агентской ветки в чистую MR-ветку (убирает `.claude/`, `.tasks/`, `.playwright-cli/`), выдаёт 3 блока для MR.
- **rebuild-autocore:** пересборка локального пакета `autocore` (build wheel → uninstall → install в venv one-web); триггерится PostToolUse-хуком после коммита в `$HOME/Projects/AT/autocore`.
- **retro:** антиэнтропийная чистка агентной инфры (ledger/metrics.jsonl, fix-report, MEMORY.md/CLAUDE.md бюджеты, дрейф веток) — чистый housekeeping, к автотестам прямого отношения не имеет.

Публикация в Allure: через `allurectl upload/launch` (в пайплайне) + `python -m tools.testops launches/launch`. НЕ через IVA QA Agent.

## 9 скиллов (таблица)

Frontmatter во ВСЕХ: `name`, `description`. Плюс `allowed-tools` у 8 из 9 (у `batch-autotest` его НЕТ). Формат — Anthropic Claude Code Skill (не формат ингредиента tacticum-dev).

| id | назначение | реальные тулы/окружение | refs | allowed-tools |
|---|---|---|---|---|
| write-autotest | 1 TC (CSV/TestOps-URL/текст) → `tests/test_*.py` | `python -m tools.testops get`, pytest, субагенты codebase-analyst/dom-explorer/code-writer, venv one-web, `.tasks/`, AUT_OVERVIEW.md | 7 | Read,Write,Edit,Glob,Grep,Bash,Task |
| batch-autotest | набор TC из TestOps-фильтра, сквозной | те же + `tools.testops search`, PROGRESS.md/metrics.jsonl | 3 | (нет поля) |
| playwright-cli | браузерные взаимодействия / поиск локаторов | CLI-бинарь `playwright-cli` (open/run-code/snapshot/state/tracing/route…) | 7 | `Bash(playwright-cli:*)` |
| fix-failed-test | разбор падений pytest, авто-правки, batch | pytest, `allure-raw`, URL ланча Allure, `tools.testops`, субагенты, AskUserQuestion | 15 | Read,Write,Edit,Glob,Grep,Bash,Task,AskUserQuestion |
| run-tests | прогон pytest в браузере + архив allure-raw | pytest, browser, `./allure-raw`, зовёт fix-failed-test | 0 | Read,Glob,Grep,Bash,Skill,AskUserQuestion |
| jira-issue-autotest | e2e: Jira ISSUE → автотесты → ланч пайплайна | `tools.testops cases/get/launches`, `glab ci run`, `allurectl launch add-issue`, оркестрит batch/prepare-mr/fix | 0 | Read,Glob,Grep,Bash,Skill,AskUserQuestion,Task |
| prepare-mr-branch | cherry-pick проектных коммитов в MR-ветку | git (cherry-pick/amend), git log/diff | 3 | Read,Write,Edit,Glob,Grep,Bash,AskUserQuestion |
| rebuild-autocore | пересборка пакета autocore в venv | build wheel, pip uninstall/install, `$HOME/Projects/AT/autocore`, PostToolUse-хук | 0 | Bash,Read,Glob |
| retro | чистка агентной инфры (ledger/память/беклог) | Read/Grep/Bash по `.tasks/`, MEMORY.md, metrics.jsonl | 0 | Read,Glob,Grep,Bash,Write,Edit |

Итого references: 7+3+7+15+3 = **35 файлов** в `references/` (у 4 скиллов refs нет).

## mcp_server_spec: НЕ нужен

Вывод однозначный: **лейн = только skill_spec ×9, mcp_server_spec НЕ требуется.**
- Обращение к Allure TestOps идёт НЕ через MCP, а через собственный read-only Python-модуль `tools/testops` (пакет в репо one-web, `python -m tools.testops get/search/cases/launches/launch`). В `write-autotest/references/testops-api.md` прямо: «MCP-сервер `testops` больше НЕ используется» (модуль точнее — раскрывает shared steps). Токен зашит в `tools/testops/client.py`, внутренний стенд.
- Остальное — внешние CLI, которые ПРЕДПОЛАГАЮТСЯ установленными в окружении репо: `pytest`, `playwright-cli` (бинарь), `allurectl`, `glab`, `git`, venv+`autocore`. Лейн их не провиженит — это среда one-web.
- Ни один скилл не дергает helm-analyst/iva-read/iva-write MCP.

Т.е. автотест-лейн — чисто текстовые skill_spec + внешние CLI. Контраст с текущим placeholder iva-role-qa, который завязан на MCP (helm-analyst/iva-read/iva-write) через analysis+write лейны.

## Композиция + развилка

**Single-owner / коллизии id — ЧИСТО, пересечений НЕТ.** Сверил 9 id со всеми ingredient_id core/analysis/write/dev/ui лейнов. Ни write-autotest, ни batch-autotest, ни playwright-cli, ни fix-failed-test, ни run-tests, ни jira-issue-autotest, ни prepare-mr-branch, ни rebuild-autocore, ни retro нигде не встречаются. Флагну два БЛИЗКИХ по имени (НЕ коллизии, разные id, но семантически рядом — стоит держать в голове при неймингах):
- `playwright-cli` (наш skill, обёртка CLI-бинаря) vs `playwright` (mcp_server_spec Playwright MCP в tacticum-ui-base / *-brownfield). Разные вещи.
- `run-tests` (наш skill) vs `test-runner`/`tester` (agent_spec) + `run-test-runner` (command_spec) в dev-лейнах. Разные id, но концепт «прогон тестов» пересекается.

**Композиция.** Текущий placeholder `iva-role-qa` = `[tacticum-core-base, iva-analysis-base, iva-write-base]` (контрактные TC через tests-authoring + helm-analyst.requirement_tests + публикация через iva-write). Предложенная ГД для Трека B = `[tacticum-core-base, iva-qa-autotest-base, iva-write-base]` — заменяет analysis-base на новый автотест-лейн.

**Нужен ли ещё iva-analysis-base?** Это ровно упирается в развилку «кто генерит тест-кейсы». Фиксирую зависимость (НЕ решаю):
- `iva-qa-autotest-base` даёт КОД автотестов (берёт УЖЕ существующие TC из Allure/CSV на вход). Он НЕ авторит тест-кейсы и НЕ даёт статус покрытия.
- `iva-analysis-base` даёт `tests-authoring` (авторинг TC контрактного уровня, GIVEN/WHEN/THEN) + `helm-analyst.requirement_tests` (детерминированный статус покрытия автотестами из Allure).
- Если QA-роль ДОЛЖНА и авторить TC, и мерить покрытие — тогда роль = `[core, iva-analysis-base, iva-qa-autotest-base, iva-write-base]` (4 лейна). Если авторинг TC остаётся у аналитика (analysis-роль), а QA только кодит автотесты по готовым TC — analysis-base можно убрать, роль = `[core, iva-qa-autotest-base, iva-write-base]`.
- Single-owner это НЕ ломает в любом варианте: id автотест-лейна и analysis-лейна не пересекаются (проверено). depth-1 композиция ADR-0056 позволяет и 4 лейна.
- **Развилку «аналитик vs QA генерит TC» оставляю лиду** — от неё зависит только наличие/отсутствие iva-analysis-base в depends_on роли, сам автотест-лейн от неё не меняется.

## Что править при переносе в `templates/iva-qa-autotest-base/ingredients/skills/<id>/`

Формат ингредиента (образец `iva-analysis-base/ingredients/skills/*/SKILL.md`) — frontmatter только `name` + `description` (без `allowed-tools`); триггер идёт в `manifest.yaml` → `metadata.description_trigger`, `kind: skill_spec`.

1. **Frontmatter:** убрать `allowed-tools` из SKILL.md (в образце analysis его нет); при желании сохранить allow-list — это уровень permission_policy/claude-settings, не тела скилла. `name`+`description` оставить. У `batch-autotest` description — YAML-блок `|` (многострочный) — валиден, но проверить, что каталог его парсит (в analysis используют `>` folded).
2. **Manifest:** на каждый скилл — `ingredient_id`, `kind: skill_spec`, `tier`, `supports: [claude-code, codex]` (проверить codex-совместимость: скиллы завязаны на Claude-специфику — субагенты Task, Skill(), хуки PostToolUse), `body_path`, `codex_target_path`, `metadata.description_trigger`.
3. **references — тянуть как есть** (35 файлов), они самодостаточны. НО: пути внутри абсолютные/окруженческие (`$HOME/Projects/AT/autocore`, `./venv/bin/python`, `.tasks/`, `allure.iva.ru/project/5`, стенды `at-test-1-one.ivcs.su`) — это хардкод среды one-web. Для лейна-шаблона решить: оставить как есть (лейн только для one-web) или параметризовать. Санитизация в скиллах уже частично есть (креды `m.zuev/testtest`, `{stand_url}`), но пути autocore/venv — нет.

## Риски / вопросы

1. **КРИТИЧНО — 3 субагента отсутствуют в источнике.** `write-autotest`, `batch-autotest`, `fix-failed-test` делегируют через Task субагентам `codebase-analyst` (20 упоминаний), `dom-explorer` (47), `code-writer` (30). Их определений в распакованном источнике НЕТ (только docs+skills). Без них write/batch/fix НЕ работают. Нужны как **agent_spec** в лейне ИЛИ отдельно получить их исходники от QA-команды. **Вопрос лиду:** где взять defs этих 3 агентов?
2. **Скиллы жёстко привязаны к репо one-web и его инфре:** пакет `autocore`, venv, `tools/testops`, `AUT_OVERVIEW.md`, `.tasks/`-конвенция, PostToolUse-хук, `glab`/`allurectl`/`playwright-cli`-бинари, GitLab CI (`.gitlab-ci.yml`, `glab ci run`). Это НЕ стэк-агностичный лейн (в отличие от analysis) — он для конкретного web-репо. Уточнить у лида, ок ли это для шаблона.
3. **docs ≠ skills:** PoC описывает тест-дизайн через IVA QA Agent (сервис 10.6.0.219:8000), которого в скиллах нет. Скиллы — про автотест-код. Если лейну нужен и тест-дизайн-слой (Qwen→Allure), IVA QA Agent — отдельная интеграция, в 9 скиллах её нет.
4. **codex-совместимость под вопросом:** скиллы используют Claude-специфику (Task-субагенты, Skill()-кросс-вызовы, хуки). Analysis-лейн заявляет `supports: [claude-code, codex]` — для автотест-скиллов это надо проверить, вероятно многое только claude-code.
5. **`retro`** — housekeeping агентной инфры, а не QA-навык; стоит ли вообще включать в автотест-лейн или он относится к общей инфре — на решение лида.

## Готовность грунта
Грунт к сборке лейна — **частично готов.** 9 SKILL.md + 35 references читаемы и переносимы, id бесконфликтны, mcp_server_spec не нужен, формат адаптации понятен. **Блокер:** отсутствуют 3 субагента (codebase-analyst/dom-explorer/code-writer) — без них половина лейна нерабочая. До их получения лейн собрать можно, но 3 из 9 скиллов (write/batch/fix) будут неполными.