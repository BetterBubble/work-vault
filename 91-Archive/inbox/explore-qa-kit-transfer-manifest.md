---
title: explore-qa-kit-transfer-manifest
type: note
permalink: tacticum/00-board/explore-qa-kit-transfer-manifest
status: draft
lead: explorer-qa
tags:
- explore
- qa
- kit
- transfer
- subagents
- lead-qa
- manifest
archived-at: 2026-07-31 04:54
---

# Манифест переноса kit → QA-лейн (кол ГД #1+#2)

**Когда:** 2026-07-23 · **Кто:** explorer-qa → lead-qa
**Основание:** [[recon-kit-full-qa-dorabotka]] (не дублирую — опираюсь).
**Источники читаны напрямую:** kit `.../scratchpad/kit/kit-main/`, target `~/tacticum/tacticum-dev-iva-write/templates/iva-qa-autotest-base/`.

Все пути ниже — абсолютные либо от корней:
- **K** = `/private/tmp/claude-501/-Users-bubblemac-tacticum/5bcc859c-bf80-431d-bfc0-adde0845932e/scratchpad/kit/kit-main`
- **T** = `~/tacticum/tacticum-dev-iva-write/templates/iva-qa-autotest-base`

---

## 0. Ключевой контекст структуры (важно для всех разделов)

- Наш лейн **T** — это **template-каталог Tacticum** (schema v2, `manifest.yaml`), а НЕ плагин. Единственный ingredient-контент — `T/ingredients/skills/` (9 skill_spec). **Нет** `agents/`, **нет** `stacks/`, **нет** `.py`-скриптов, **нет** `answers.toml`.
- Kit **K** — marketplace-плагин; craft-модуль ссылается на стек через `$CRAFT/stacks/{stack}/` (= `${CLAUDE_PLUGIN_ROOT}` в CC). **У нас плагин-рута нет** — все `$CRAFT/...`-ссылки надо разрешать в конкретные пути (см. §1).
- Схема ingredient (`T/../_schema/ingredient.v1.schema.json`) **поддерживает `kind: agent_spec`** (строка 12, блок стр. 55–71): требует `metadata.model` + `metadata.description`, опц. `metadata.tools`. → субагентов можно объявить штатно, и **model=opus прописывается на уровне ingredient** (во фронтматере дефов kit `model:` НЕТ).

---

## 1. #1 Перенос субагентов — таблица source → target

### 1a. Три дефа субагентов (kind: agent_spec, model=opus)

| Source (K) | Target (T) | target_path_template (в консьюмере) | tools (из фронтматера) |
|---|---|---|---|
| `craft/agents/codebase-analyst.md` | `ingredients/agents/codebase-analyst.md` | `.claude/agents/codebase-analyst.md` | `Read, Glob, Grep` (Write НЕТ) |
| `craft/agents/dom-explorer.md` | `ingredients/agents/dom-explorer.md` | `.claude/agents/dom-explorer.md` | `Bash, Read, Write, Glob, Grep` |
| `craft/agents/code-writer.md` | `ingredients/agents/code-writer.md` | `.claude/agents/code-writer.md` | `Read, Edit, Write, Glob, Grep, Bash` |

- Имя агента (`name:` во фронтматере) — **bare**: `codebase-analyst` / `dom-explorer` / `code-writer`. В файле CC агент резолвится по имени файла = bare.
- **model=opus** — задать в `metadata.model` каждого agent_spec (требование приёмки; у kit во фронтматере модели нет).

### 1b. Файлы стека pytest-playwright-canvas (8 файлов)

Все три дефа + оркестраторы ссылаются на `$CRAFT/stacks/pytest-playwright-canvas/{...}`. Нужен единый дом стека в консьюмере (напр. `.claude/skills/_craft-stack/pytest-playwright-canvas/` или assets-ingredient). Источник:

| Source (K) `craft/stacks/pytest-playwright-canvas/` | Роль |
|---|---|
| `recon.md` | модель поверхности canvas + инструмент разведки (нужен dom-explorer) |
| `rules/page-objects.md` | правила page-слоя/адресации/ассертов/i18n |
| `rules/tests.md` | сигнатура теста, маркеры, преамбулы, xfail |
| `test-templates.md` | single/multi-session шаблоны |
| `failure-taxonomy.md` | 7 категорий падения (нужен codebase-analyst fix + code-writer) |
| `fix-playbooks.md` | плейбуки правок по категориям |
| `allure-raw-parser.md` | разбор сырых результатов |
| `pytest-runner.md` | команды раннера |
| `batch-conventions.md` | матрица верификации, конвенции батча |

Плюс стек-нейтральный канон, на который ссылаются дефы/скиллы:
- `craft/rules/invariants.md` → общий дом (напр. `_craft-stack/rules/invariants.md`)
- `craft/shared/{ledger-and-deviations.md, jira-candidates.md, coverage-ledger.template.md}` → `_craft-stack/shared/`
- `craft/references/product-api/research_user.py` → референс фабрики research-юзеров (только если включаем ось product-api)

> ⚠️ Частично эти файлы **уже** лежат в нашем лейне как per-skill references (byte-copy): напр. `fix-failed-test/references/{failure-taxonomy,fix-playbooks,allure-raw-parser,pytest-runner}.md`, `write-autotest/references/test-templates.md`. То есть либо (а) дедуплицировать в общий `_craft-stack/`, либо (б) оставить per-skill и переписать `$CRAFT`-ссылки на локальные. Решение — за lead-qa (это правка, не разведка).

### 1c. Оркестраторы (kind: skill_spec — заменяют тела наших 3 заблокированных)

| Source (K) | Наш ingredient_id (T) | body_path |
|---|---|---|
| `craft/skills/write/SKILL.md` + `references/*` | `write-autotest` | `ingredients/skills/write-autotest/SKILL.md` |
| `craft/skills/batch/SKILL.md` + `references/*` | `batch-autotest` | `ingredients/skills/batch-autotest/SKILL.md` |
| `craft/skills/fix/SKILL.md` + `references/*` | `fix-failed-test` | `ingredients/skills/fix-failed-test/SKILL.md` |

kit write/references: `csv-parsing.md, description-to-input.md, feature-mapping.template.md, input-sources.md, sanitization.md, tc-review.md` (у нас `tc-review.md` нет, `feature-mapping.md` вместо `.template`).
kit fix/references: `batch-discovery.md, batch-report.md, deferred-handling.md, failure-localization.md, input-sources.md, known-issues.md, launch-input.md, reproduce-phases.md, verification-loop.md, xfail-verification.md`.
kit batch/references: `conventions.md, phases.md, progress.md`.

### 1d. Разрешение `$CRAFT`/`${CLAUDE_PLUGIN_ROOT}` у нас

В нашем шаблоне плагин-рута НЕТ. Каждую ссылку `$CRAFT/stacks/{stack}/X`, `$CRAFT/rules/X`, `$CRAFT/shared/X` в переносимых дефах и SKILL.md надо **переписать на конкретный путь** дома стека в консьюмере (напр. `.claude/skills/_craft-stack/...`). Аналогично `$TMS/scripts/testops_cli.py` → путь к CLI TestOps в консьюмере. Параметр `stack` из `[craft]` у нас фиксирован = `pytest-playwright-canvas` (можно инлайнить, без переменной).

---

## 2. Как наши 3 заблокированных скилла зовут субагентов — контракт и РАСХОЖДЕНИЯ

Прочитаны `T/ingredients/skills/{write-autotest,batch-autotest,fix-failed-test}/SKILL.md`.

**Имена в Task:** наши скиллы зовут **bare**: `codebase-analyst`, `dom-explorer`, `code-writer`. Оркестраторы kit зовут **с префиксом модуля**: `craft:codebase-analyst`, `craft:dom-explorer`, `craft:code-writer`. → Если ставим агентов как `.claude/agents/<bare>.md` (наша template-модель, не плагин), то **правильны bare-имена**; при переносе тел kit-скиллов префикс `craft:` надо снять.

**РАСХОЖДЕНИЕ 1 (критично) — контракт записи `analysis.md`:**
- Наш `write-autotest` шаг 3: «Вызови `codebase-analyst` … **Запиши** `.tasks/tc-{allure_id}-analysis.md`» — т.е. ждёт, что САБАГЕНТ пишет файл.
- Деф kit `codebase-analyst` имеет tools `Read, Glob, Grep` — **Write у него НЕТ**, и он намеренно «на диск не пишет вовсе», возвращает контент текстом (обход эвристики CC «report file»). Файл `analysis.md` пишет **оркестратор** (kit write SKILL шаг 3: «Запиши возврат субагента … сам»).
- → наш текущий шаг 3 **несовместим** с дефом kit: агент физически не сможет записать analysis.md. Тела наших скиллов надо привести к kit-контракту (оркестратор персистит).

**РАСХОЖДЕНИЕ 2 (критично) — каталог артефактов:**
- Наши скиллы: плоско `.tasks/tc-{id}-input.md`, `.tasks/tc-{id}-analysis.md`, `.tasks/tc-{id}-locators.md`; fix: `.tasks/batch-{ts}/`.
- kit: дом задачи `.tasks/work/tc-{id}/{input,analysis,locators}.md`; fix: `.tasks/work/batch-{ts}/`; batch-кампания: `.tasks/work/wave-<кампания>/`.
- Деф kit `codebase-analyst` (mode=write) **жёстко ждёт** `.tasks/work/tc-{id}/input.md`; dom-explorer пишет `.tasks/work/tc-{id}/locators.md`. → наша плоская раскладка не сойдётся с путями, зашитыми в дефы. Переходить на `.tasks/work/...`.

**СОВПАДЕНИЯ (перенос без правок контракта):**
- Имена артефактов **fix-режима идентичны**: `fix-{test}-analysis.md`, `fix-{test}-locators.md`, `fix-{test}-failure.md`, отчёт `fix-batch-{ts}-report.md`. Отличается только префикс каталога (`.tasks/` → `.tasks/work/`).
- Режимы субагентов (write/fix/adhoc), таблицы вывода analysis.md, категории падения (DOM_CHANGED/UX_FLOW_CHANGED/API_CONTRACT_CHANGED/FLAKY_TIMING/IMPORT_OR_REGISTRY/PRODUCT_BUG/UNKNOWN) — совпадают между нашими references и kit.
- batch: наши шаги 4–6 (codebase-analyst→dom-explorer→code-writer, тесты пишет оркестратор, не code-writer) — 1:1 с kit batch.

**Вывод по §2:** голая подстановка 3 дефов под НАШИ текущие тела скиллов НЕ заработает (расхождения 1 и 2). Чистый путь — перенести и тела оркестраторов kit (write/fix/batch) вместе с дефами, сохранив наши `ingredient_id`. Это и есть смысл «забрать комплектом».

---

## 3. `[craft]`-параметры

Источник схемы — **`K/craft/README.md` стр. 21–34** (в base `answers.template.toml` секции `[craft]` НЕТ — её несёт craft-модуль). Параметры и значения под наш проект:

| Параметр | Значение под one-web/наш лейн | Комментарий |
|---|---|---|
| `stack` | `"pytest-playwright-canvas"` | наш стек (kmp/canvas) |
| `default_env` | `"kmp-stage"` (уточнить актуальный) | env стенда по умолчанию для прогонов |
| `shared_users` | `["m.zuev","d.orlova","i.fedotov"]` | white-list общих тест-юзеров (сейчас хардкодом в `playwright-cli/SKILL.md:71-73` + `write-autotest/references/*`) |
| `research_user_cmd` | `""` или команда фабрики (референс kit: `./venv/bin/python .claude/scripts/research_user.py`) | пусто = фабрики нет |
| `vendored_dep` | `"autocore"` | имя vendored-библиотеки REST-обёрток (сейчас захардкожено в fix-playbooks/analysis) |
| `product_clone` | путь к read-only клону iva-one (сейчас хардкод `/Users/oc4kxb/Projects/iva/one/web/iva-one` в `write-autotest/SKILL.md:24`) | пусто = нет клона |

**Где прописать у нас:** в нашей template-модели нет `.kit/answers.toml`. Варианты (решение lead-qa): (а) новый ingredient (config/instruction_pack), кладущий `.claude/craft/answers.toml`-фрагмент в консьюмер; (б) инлайнить значения прямо в перенесённые SKILL/agent-файлы (мы фиксируем один стек). Сейчас эти значения размазаны хардкодом по скиллам — их надо собрать в один носитель.

---

## 4. #2 Санитизация — точные точки

### 4a. Хардкод `allure.iva.ru` (файл:строка) — всё в T/ingredients/skills

- `batch-autotest/SKILL.md:19`
- `batch-autotest/references/phases.md:225` (`allurectl upload --endpoint https://allure.iva.ru/ --token <TOKEN> ...`)
- `write-autotest/SKILL.md:3`, `:41`
- `write-autotest/references/input-sources.md:10`
- `write-autotest/references/testops-api.md:5, :12, :13, :35`
- `fix-failed-test/SKILL.md:3`, `:50`
- `fix-failed-test/references/launch-parser.md:3`
- `fix-failed-test/references/input-sources.md:79, :82`

### 4b. Токен — где именно

**В нашем лейне отдельного `client.py` НЕТ** (T `.py`-файлов не содержит вовсе). Токен присутствует как **упоминание/установка в доках**:
- `write-autotest/references/testops-api.md:29` — проблемная формулировка: «Дефолты endpoint/token/project **зашиты** в `tools/testops/client.py` … Токен — внутреннего стенда, **не секрет**.» ← ровно анти-модель, что kit убирает.
- `fix-failed-test/references/launch-parser.md:84` — «Токен и endpoint — **дефолтные** в `tools/testops/client.py`.»
- `fix-failed-test/references/fix-playbooks.md:71` — шаблон autocore-функции с `headers = {"Authorization": f"Bearer {token}"}` (пример кода теста, не креды — по контексту, не хардкод-секрет).

Реальный `client.py` с зашитыми endpoint+token **живёт в репо one-web** (`tools/testops/client.py`), НЕ в шаблоне. → «вырезать токен из нашей копии» = переписать доки, ссылающиеся на зашитые дефолты, на env-модель; сам client.py байт-переносится в консьюмер (или заменяется kit-версией) — см. 4d.

### 4c. Хардкод кред тест-юзеров (для shared_users, §3)
- `playwright-cli/SKILL.md:71-73, :81` — `m.zuev/d.orlova/i.fedotov` `testtest`
- `write-autotest/references/description-to-input.md:9, :52, :53, :60, :78`, `input-sources.md:52`

### 4d. Env-модель kit (что байт-переносить)

**Файл:** `K/tms/scripts/testops/client.py` (150 строк, читан целиком). Ключевое:
- `resolve_settings()` (стр. 47–66): `endpoint/token/project_id` = **env `TESTOPS_ENDPOINT`/`TESTOPS_TOKEN`/`TESTOPS_PROJECT_ID`** → иначе секция `testops:` из **gitignored `./secrets.yaml`** (`_SECRETS_FILE="secrets.yaml"`, `_SECRETS_SECTION="testops"`, стр. 21–22) → иначе `TestOpsError` с инструкцией. **Порядок: env первичен** (для CI masked vars / удалённого хостинга), secrets.yaml — fallback. Литералов endpoint/token в коде НЕТ.
- `_secrets_section()` (стр. 31–44): читает `os.getcwd()/secrets.yaml` (вызывать из корня проекта), `yaml.safe_load`, секция `testops:`.
- Обмен токена (стр. 91–107): `POST {endpoint}/api/uaa/oauth/token`, `data={grant_type:"apitoken", scope:"openid", token:<apitoken>}` → `access_token` → `Authorization: Bearer <jwt>`. Ленивая авторизация + один авто-повтор при 401.
- Только GET (+ этот POST для auth). `verify=False`, warnings приглушены.

Сопутствующие модули пакета (при переносе всего клиента): `K/tms/scripts/testops/{__init__.py, __main__.py, launch.py, params.py, testcase.py}` + CLI `K/tms/scripts/testops_cli.py`.

**`secrets.yaml.example`:** в снапшоте kit **ОТСУТСТВУЕТ** (искал — нет; см. §6). client.py и SKILLы ссылаются на «форму — secrets.yaml.example», но самого файла в архиве нет — **пробел снапшота**.

**`.gitignore`:** `K/.gitignore` минимальный (`__pycache__/`, `.idea/`, `.vscode/`) — **`secrets.yaml` в нём НЕ игнорируется** (пробел kit). Что дописать у консьюмера (и в наш шаблон, если появится scripts-ingredient): строку `secrets.yaml`.

---

## 5. Приёмка (доказать живую работу 3 скиллов с субагентами на opus)

Прогон в консьюмер-репо one-web (наш лейн осмыслен только там — `manifest.yaml` явно: привязка к autocore/venv/tools.testops/glab/CI):

1. **Установка:** после переноса — агенты видны как `.claude/agents/{codebase-analyst,dom-explorer,code-writer}.md`, каждый с `model: opus`. Проверка: `/agents` в CC показывает три с моделью opus.
2. **write-autotest (1 TC):** дать URL TC (env `TESTOPS_TOKEN` задан, `allure.iva.ru`-хардкода в доках нет). PASS = создан `.tasks/work/tc-{id}/{input,analysis}.md` (analysis записан **оркестратором**, не агентом), при пробелах — `locators.md` от dom-explorer, тест-функция в `tests/test_<feature>.py`, `--collect-only` зелёный. В логе Task — спавн `codebase-analyst`/`dom-explorer`/`code-writer` на opus.
3. **batch-autotest:** URL фильтра TestOps → `.tasks/work/wave-<кампания>/PROGRESS.md` + ≥1 TC-дом, веер research-агентов реально стартовал.
4. **fix-failed-test:** имя упавшего теста / ланч-URL → `.tasks/work/batch-{ts}/fix-{test}-analysis.md` от codebase-analyst(fix), при DOM_CHANGED — `fix-{test}-locators.md` от dom-explorer, правка от code-writer, верификационный прогон зелёный.
5. **Секреты:** grep по перенесённой копии — `allure.iva.ru` и литералов токена нет; `TestOpsClient()` без аргументов резолвит из env/secrets.yaml; при снятом env и без secrets.yaml — `TestOpsError` с инструкцией (не молчаливый фейл).
6. **opus-инвариант:** в трейсе Task у всех трёх субагентов модель = opus (не дефолт сессии).

---

## 6. Риски / пробелы

1. **Структурный разрыв template↔plugin (главный).** Наш лейн — skills-only Tacticum-template без plugin-рута, `$CRAFT`/`$TMS`/`${CLAUDE_PLUGIN_ROOT}` не резолвятся. Требуется: (а) объявить дом стека (`_craft-stack/`) и переписать все `$CRAFT/...`-ссылки; (б) объявить `agent_spec`×3; (в) решить носитель `[craft]`-параметров и client.py (scripts-ingredient в шаблоне сейчас нет).
2. **`manifest.yaml` явно декларирует НЕ-стек-агностичность** и привязку к one-web (autocore/venv/tools.testops/glab/CI). Противоречит стек-нейтральной модели kit — перенос дефов/оркестраторов kit частично разворачивает эту декларацию. Нужна ревизия manifest (описание, `stack.required`, добавление agent_spec/ingredient'ов). Это правка — за lead-qa.
3. **`secrets.yaml.example` отсутствует в снапшоте kit** — а client.py и все инструкции на него ссылаются. Нельзя байт-перенести. Достать из upstream (`git.hi-tech.org/ivaqa/kit`) либо воссоздать по форме (секция `testops: {endpoint, token, project_id}`).
4. **`K/.gitignore` не игнорирует `secrets.yaml`** — пробел самого kit; при переносе client.py обязательно дописать `secrets.yaml` в .gitignore консьюмера.
5. **Три несовместимости** (§2): имена Task (`craft:`-префикс vs bare), раскладка `.tasks/` (`.tasks/work/tc-{id}/` vs плоско), контракт записи analysis.md (оркестратор vs агент) — из-за них «голая» подстановка дефов не сработает. Лечится переносом тел оркестраторов kit целиком.
6. **model=opus** во фронтматере дефов kit не задан — проставить на уровне `agent_spec.metadata.model` (иначе субагенты пойдут на дефолтной модели сессии, приёмка §5.6 провалится).
7. **Дублирование stack-файлов**: часть kit-стека уже лежит per-skill в наших references (byte-copy, возможно дрейфующая). Не оставить две расходящиеся копии.

---

## Открытые вопросы к lead-qa

1. **Модель переноса скиллов:** заменяем тела наших `write-autotest`/`batch-autotest`/`fix-failed-test` телами kit `craft:write/batch/fix` (сохранив `ingredient_id`) — или патчим наши тела точечно под контракт дефов? Расхождения §2 по сути требуют замены тел.
2. **Дом стека и `[craft]`-параметров:** какой носитель в нашей template-модели — новый общий ingredient (`_craft-stack/` + `answers.toml`-фрагмент) или инлайн значений (один фиксированный стек)?
3. **client.py и секреты:** байт-переносим весь пакет `tms/scripts/testops/` в наш шаблон как scripts-ingredient (нужен новый kind/ingredient), или это ответственность one-web, а мы чистим только доки от `allure.iva.ru`/«токен зашит»?
4. **secrets.yaml.example** — доставать из upstream kit (нужен доступ к `git.hi-tech.org/ivaqa/kit`) или воссоздать по форме из client.py?