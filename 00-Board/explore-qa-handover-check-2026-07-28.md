---
title: explore-qa-handover-check-2026-07-28
type: note
status: current
created: 2026-07-28 11:50
tags:
- board
- qa
- handover
permalink: tacticum/00-board/explore-qa-handover-check-2026-07-28
updated: 2026-07-28 11:50
repo: tacticum-dev
project: QA-профиль
---

# Сверка репо-стороны QA-передачи (по факту main, не по заметкам)

**Метод:** read-only, `~/tacticum/tacticum-dev` @ `main` = `26f5301` (2026-07-27), ветки не переключались, ничего не правилось. Все утверждения — с `file:line`. Где не проверено — сказано прямо.

---

## 1. Версии на main сейчас

| Профиль | version | Строка | Последний коммит по каталогу |
|---|---|---|---|
| `iva-role-qa` | **0.5.1** | `templates/iva-role-qa/manifest.yaml:22` | `1c6b7a9` 2026-07-24 «iva-role-qa 0.5.1 — композиция на iva-qa-autotest-base 0.3.0 (переморозка depends_on)» |
| `iva-qa-autotest-base` | **0.3.0** | `templates/iva-qa-autotest-base/manifest.yaml:22` | `ad1377a` 2026-07-24 «README/CHANGELOG — репо one-web-kmp, Мульти-стэк на KMP, секция [0.3.0]» |
| `iva-qa-mcp` | **0.1.0** | `templates/iva-qa-mcp/manifest.yaml:19` | `20ff9b8` 2026-07-24 «QA-профиль iva-role-qa + автотест-лейн + per-role MCP-лейны (#133)» |
| `tacticum-core-base` | **0.1.1** | `templates/tacticum-core-base/manifest.yaml:14` | `ccd247e` 2026-07-22 (#124, kb-navigation) |

Композиция роли: `depends_on: [tacticum-core-base, iva-qa-autotest-base, iva-qa-mcp]` — `templates/iva-role-qa/manifest.yaml:29-32`. Совпадает с заметками.

**Прод не проверялся** — задача была про репо-сторону, на прод-сервер не ходил. Цепочка «installation `b258bb6b` pin 0.5.1» из `checkpoint-lead-qa.md:24` НЕ подтверждена и НЕ опровергнута.

---

## 2. Правка метаданных стека 24.07 — доехала ЧАСТИЧНО

**Доехала в лейн `iva-qa-autotest-base` (0.3.0), НЕ доехала в роль `iva-role-qa` (0.5.1).** CHANGELOG роли сам это фиксирует: `templates/iva-role-qa/CHANGELOG.md:11` — *«Роль и её ингредиенты не менялись — bump нужен, чтобы ребро `depends_on` переморозилось»*. Правка была осознанно ограничена лейном, а роль осталась с прежним ошибочным обрамлением.

Лейн — чисто: `stack.required: [one-web-kmp]` (`iva-qa-autotest-base/manifest.yaml:62`), `name`/`description`/`persona.scope` на `one-web-kmp` и `pytest/Playwright`.

### Остатки в РОЛИ (это то, что видит инженер после установки)

**manifest роли:**
- `templates/iva-role-qa/manifest.yaml:75` — `stack.optional: [one-web]` (в лейне уже `one-web-kmp`)
- `:8` — «генерация/прогон/починка pytest/Selenium» (шапка)
- `:42` — «АВТОТЕСТ-лейн (9 скиллов): генерация pytest/Selenium»
- `:47` — «Привязан к репо one-web»
- `:60` — `persona.scope`: «Исполнение/автоматизация автотестов **one-web** (генерация pytest/**Selenium** …)»
- `:63` — `target_tasks`: «сгенерировать pytest/Selenium автотест»
- `:111` — `post_install_notes`: «⚠️ Автотест-лейн привязан к репо **one-web**»

**Тела пак-ингредиентов роли — приземляются в целевой репо как `CLAUDE.md` / `AGENTS.md` / `.codex/config.toml`:**
- `ingredients/repo-configs/claude-code/CLAUDE.md.fragment:15` — «генерация pytest/Selenium по TC»
- то же, `:25` — «автотест-конвенции **one-web**»
- то же, `:40` — «`kb_*` — intent/архитектура/топология **one-web**»
- то же, `:41` — «Прогон/сборка: **pytest + Selenium** через autocore-venv репо **one-web**»
- `ingredients/repo-configs/codex/AGENTS.md.fragment:23` — «разведка автотест-кодовой базы **one-web**»
- то же, `:25` — «`code-writer` — генерация/правка **pytest+Selenium** по тест-кейсу»
- то же, `:30` — «автотест-конвенции **one-web**»
- то же, `:41` — «топология **one-web**»
- то же, `:42` — «Прогон: **pytest + Selenium** (autocore-venv **one-web**)»
- `ingredients/repo-configs/codex/config.toml.template:2` — «autotest repo **one-web**»
- то же, `:28` — «Serena … по автотест-коду (**pytest + Selenium**)»

**Итого 18 остатков** (7 в манифесте роли + 11 в телах паков). Значимость: 11 из них — в файлах, которые физически приземляются в `one-web-kmp` и являются первым, что читает агент в сессии. Инструкция команде (`12-Features/iva-role-qa — установка и работа (для команды).md:84`) говорит «стек pytest + Playwright (canvas), репо one-web-kmp» — установленный `AGENTS.md` скажет обратное.

**Висячие `_selenium-ref` — вычищены.** `grep -rn '_selenium-ref' templates/` даёт 3 попадания, все в CHANGELOG'ах (описание самой правки): `iva-qa-autotest-base/CHANGELOG.md:29,30` и `iva-role-qa/CHANGELOG.md:11`. Живых ссылок нет.

**Контрастные «референс Selenium: …» в телах скиллов/субагентов — оставлены осознанно** (`iva-qa-autotest-base/CHANGELOG.md:22-24`): это dual-stack teaching, не описание стека. Как остаток НЕ считаю. Исключение — `run-tests`, см. п.3 грабля #2.

---

## 3. Грабли первого прогона в `one-web-kmp`

### #1 (killer) — 45 файлов канона стека и references НЕ доставляются вообще

Механики доставки вспомогательных файлов скилла в платформе **нет**, и это задокументировано в самом коде:

`apps/backend/src/backend/catalog/domain/ingredients/skill_spec.py:25-26`:
> «**Доставки этих файлов сейчас нет** — рендер ставит только ``body``. Не выдавать наличие поля за наличие механики.»

Поле `metadata.assets` в схеме есть (`templates/_schema/ingredient.v1.schema.json:81`) и `tacticum-lite-base` его объявляет (`templates/tacticum-lite-base/manifest.yaml:86-87`), но даже там это только декларация. **`iva-qa-autotest-base` не объявляет `assets` ни у одного из 10 `skill_spec`** — `grep -n 'assets\|scripts:' templates/iva-qa-autotest-base/manifest.yaml` пусто.

Подтверждается golden-снапшотом установленного дерева `apps/backend/tests/e2e_install/golden/iva-role-qa/claude-code.json` — 22 ключа, ни одного `references/`, у `craft-stack` только `repo/.claude/skills/craft-stack/SKILL.md`. Ни в одном golden репо нет `references/` (`grep -l 'references/' apps/backend/tests/e2e_install/golden/*/*.json` пусто).

Что теряется:
- **30 файлов `references/`**: `write-autotest/references/{csv-parsing,testops-api,input-sources,sanitization,tc-review,description-to-input,feature-mapping.template}.md`, `fix-failed-test/references/*` (10 файлов), `playwright-cli/references/*` (7), `batch-autotest/references/{phases,conventions,progress}.md`, `prepare-mr-branch/references/*` (3)
- **15 файлов `craft-stack/`**: `recon.md`, `rules/{page-objects,tests,invariants}.md`, `test-templates.md`, `failure-taxonomy.md`, `fix-playbooks.md`, `allure-raw-parser.md`, `pytest-runner.md`, `batch-conventions.md`, `shared/{ledger-and-deviations,pipeline-gate,jira-candidates,coverage-ledger.template}.md`, `craft-answers.example.toml`

Сколько ссылок в ДОСТАВЛЯЕМЫХ телах ведёт в пустоту: **53** на `references/` в claude-телах скиллов, **41** в codex-телах, **98** на `$CRAFT/…` в скиллах+субагентах.

Доставленный `craft-stack/SKILL.md` — буквально оглавление ненаступившего: `templates/iva-qa-autotest-base/ingredients/craft-stack/SKILL.md:19-37` — таблица «Карта файлов» из 15 строк, ни одного из этих файлов рядом не будет. Там же `:41-45` посылает читать `craft-answers.example.toml` «в этом же доме» — его тоже нет.

Практический эффект: `write-autotest` на шаге разбора входа идёт в `references/input-sources.md` / `csv-parsing.md` / `testops-api.md`; `code-writer` и `dom-explorer` стартуют с чтения `$CRAFT/rules/page-objects.md` и `$CRAFT/recon.md`; `fix-failed-test` — с `$CRAFT/failure-taxonomy.md`. Ни один не установится. Скиллы «видны» (что и проверил Level-3 smoke), но их канон — нет.

### #2 — `run-tests` остался Selenium-скиллом, противоречит объявленному стеку

`templates/iva-qa-autotest-base/ingredients/skills/run-tests/SKILL.md:36-45` — таблица браузеров: `browser_win_chrome` / `browser_win_firefox` / `browser_win_msedge` / `browser_win_ya` / `browser_mac_safari` / `browser_lin_chrome`, дефолт `lin_chrome`.
`:69` — команда: `./venv/bin/python -m pytest … -m "<combined-markers>" --env <env> [--browser-tag current | --local]`
`:64` — параллель «только lin_chrome: один **lin-selenoid** держит несколько сессий»; `:66` — «при `NoSuchSessionException`/таймаутах коннекта к **selenoid**».

Прямо противоречит собственному канону лейна: `ingredients/skills/fix-failed-test/references/reproduce-phases.md:20` — *«В одноконфигурационном стеке (референс canvas one-web-kmp: единственный headless chromium) маркеров нет — `-m` опускается»*, и `craft-stack/rules/page-objects.md:153` — *«Не использовать Selenium/XPath — их в этом стеке нет»*.

В отличие от остальных мест это НЕ оформлено как «референс Selenium: …» — это операционная инструкция по сборке команды. Правка 24.07 (`2b7746c`) прошла по `run-tests` только по репо-нейму и мёртвым ссылкам, логику браузерной матрицы не тронула.

Эффект: «прогони smoke» → агент соберёт `-m "suite_smoke and browser_win_chrome" --env at-test-1-one --browser-tag current`. Если в `one-web-kmp` таких маркеров/опций нет — pytest падает на unknown marker / unrecognized argument на первой же команде.

**Плюс расхождение стендов:** `run-tests/SKILL.md:52-55` резолвит env в `at-test-1-one` / `at-test-3-one` по ветке, а `craft-stack/craft-answers.example.toml:10` даёт «референс one-web-kmp: `kmp-stage`». Два разных ответа на «какой env».

### #3 — `tms:read` и `aql-notes.md`: скилл из kit не перенесён, ссылки висят

`batch-autotest/SKILL.md:16-17` — *«Чтение TMS — **только** через CLI модуля tms: операции (`get`, `search`, `cases`, `cfv`, `list`) и форма вызова — скилл `tms:read`»*. Скилла `tms:read` в лейне нет: `grep -n 'ingredient_id:' templates/iva-qa-autotest-base/manifest.yaml` даёт 15 id, `tms` среди них нет.

Синтаксис AQL целиком отдан наружу, и там ничего нет: `batch-autotest/references/phases.md:15,16` — *«Формат кодирования … `tms:read` `references/aql-notes.md`»*, *«грабли конкретного языка … `tms:read` `references/aql-notes.md`; синтаксис здесь не дублируем»*; также `:249`. Файла `aql-notes.md` нет нигде в репо (`find . -name 'aql-notes*'` пусто).

Ещё ссылки на `tms:read`: `batch-autotest/SKILL.md:37,68`, `phases.md:6,28,34`, `skills-codex/write-autotest/SKILL.md:100,135`, `skills-codex/fix-failed-test/SKILL.md:104`.

Эффект: `batch-autotest` (вход «пачка из фильтра TestOps») не может раскодировать URL-фильтр по инструкции — она вся в отсутствующем файле. Одиночный `write-autotest` живёт: у него есть прямая команда `./venv/bin/python -m tools.testops get <id>` (`write-autotest/references/testops-api.md:23`) — но и этот файл не доставляется (грабля #1).

Отдельно: скиллы ссылаются на `.kit/schema.md` проекта, секция «Роли» — контракт fallback обязательного делегирования (`write-autotest/SKILL.md:17`, `batch-autotest/SKILL.md:21`, `fix-failed-test/SKILL.md:63`). Лейн этот файл не поставляет; есть он в `one-web-kmp` или нет — **не проверено** (доступа к репо потребителя нет).

### #4 — `rebuild-autocore` зашит на абсолютную раскладку `$HOME/Projects/AT/…` + POSIX-venv

`ingredients/skills/rebuild-autocore/SKILL.md:29-31`:

```
AUTOCORE_DIR   = $HOME/Projects/AT/autocore
ONE_WEB_DIR    = $HOME/Projects/AT/one-web-kmp
VENV_PYTHON    = $HOME/Projects/AT/one-web-kmp/venv/bin/python
```

Дальше `:41-117` — все команды на этих абсолютных путях, включая `rm -rf $HOME/Projects/AT/autocore/{build,dist,src/autocore.egg-info,tests}` (`:113-116`) и `rm -f $HOME/Projects/AT/one-web-kmp/autocore-*.whl` (`:117`). Гарды на существование есть (`:41-42` — явная ошибка вместо тихого фейла), так что молча не сломается, но у Евгения раскладка совпадёт только случайно.

`venv/bin/python` — POSIX. При этом сам лейн в примерах трейса предполагает Windows: `fix-failed-test/references/failure-localization.md:15` — `File "venv\Lib\site-packages\selenium\webdriver\remote\webdriver.py"`. На Windows пути `venv/bin/python` не существует (там `venv/Scripts/python.exe`) — и это не только `rebuild-autocore`: `./venv/bin/python` захардкожен в `run-tests/SKILL.md:69`, `jira-issue-autotest/SKILL.md:41,46,79`, `retro/SKILL.md:40`, `craft-stack/pytest-runner.md:9`, `write-autotest/references/testops-api.md:23`. **На какой ОС работает Евгений — не проверено.** Если Windows — падают все команды прогона, не только пересборка.

Плюс `rebuild-autocore` на Codex не имеет авто-триггера: `iva-qa-autotest-base/manifest.yaml:65-66` — «авто-триггер rebuild-autocore на Codex отсутствует — PostToolUse-хука нет, только ручной вызов/git-hook репо».

### #5 — обязательные внешние зависимости, которые лейн не ставит

`post_install_notes` лейна их честно перечисляет (`manifest.yaml:306-321`), но это ровно тот список, где что-то не совпадёт:

1. venv `one-web-kmp` с установленным локальным `autocore` **и read-only модулем `tools/testops`** — вся работа с TestOps идёт через `python -m tools.testops` (`:307-308`). Модуль живёт в репо потребителя, лейн его не несёт. Нет модуля → `write-autotest` по URL, `batch-autotest`, `jira-issue-autotest`, `fix-failed-test` по ланчу не работают вообще.
2. В PATH: `pytest`, `playwright-cli` (бинарь), `allurectl`, `glab`, `git` (`:311-312`).
3. Секреты — env-модель, зашитых нет (проверил): `TESTOPS_ENDPOINT` / `TESTOPS_TOKEN` / `TESTOPS_PROJECT_ID` → fallback gitignored `secrets.yaml` секция `testops:` (`ingredients/config/secrets.yaml.example:1-17`, порядок резолва `:5-10`, apitoken→JWT `:11-12`). Форма `secrets.yaml.example` приземляется в корень репо и `.gitignore` дополняется — подтверждено golden: `repo/secrets.yaml.example` и `repo/.gitignore` в `golden/iva-role-qa/claude-code.json`. Здесь всё в порядке.
4. `[craft]` в `.kit/answers.toml` (`:319`) — форма в `craft-answers.example.toml`, который **не доставляется** (грабля #1). Скиллы обещают дефолты при отсутствии секции (`batch-autotest/SKILL.md:16-17`, `write-autotest/SKILL.md:9`), но у `shared_users` дефолта нет: `write-autotest/references/sanitization.md:23` — «Параметр `shared_users` пуст → спроси пользователя». Санитизация кредов при пустом параметре встанет вопросом.
5. `glab` должен быть установлен И авторизован, иначе деградация вручную: `jira-issue-autotest/SKILL.md:75`.
6. Конвенции репо, которые лейн предполагает готовыми: `.tasks/work/tc-{id}/`, `.tasks/work/batch-{ts}/`, `AUT_OVERVIEW.md` (`manifest.yaml:313-314`), причём `playwright-cli/SKILL.md:10` требует конкретно `.tasks/aut-research/AUT_OVERVIEW.md` с картой приложения по версиям, а `run-tests/SKILL.md:54` ссылается на секцию `CLAUDE.md → «Ветки и линии разработки»` репо потребителя. Есть ли всё это в `one-web-kmp` — **не проверено**.
7. Захардкоженные стенды ИВА: `playwright-cli/SKILL.md:10,56,64` — `https://at-test-1-one.ivcs.su/` (дефолт) и `at-test-3-one.ivcs.su` (для ветки `at-creation-crc`); `craft-stack/fix-playbooks.md:87` и `failure-taxonomy.md:179` — `local.ivcs.su:8443`. Нужен доступ до контура ИВА. Захардкоженных логинов/паролей нет — креды через `shared_users` и фабрику `research_user_cmd`.

### Прочее, не в топе, но реальное

**`claude-settings` роли режет Claude Code до трёх тулов и форсит opus** — `templates/iva-role-qa/manifest.yaml:166`:

```
body: '{"defaultModel":"opus","tools":{"allowed":["Read","Edit","Bash"]}}'
```

Приземляется как `repo/.claude/settings.json` (есть в golden). Два следствия: (а) `defaultModel: opus` — единственный живой `opus` во всём QA-комплекте, при том что решение зафиксировано обратное (`checkpoint-lead-qa.md:43` «opus НЕ форсить») и в лейне хардкод снят (`iva-qa-autotest-base/manifest.yaml:237`, `CHANGELOG.md:149`); (б) allow-list из `Read`/`Edit`/`Bash` не содержит `Write`, `Glob`, `Grep`, `Task` — а Claude-путь всех трёх оркестраторов построен на `Task`, и субагентам объявлены `tools: [Read, Edit, Write, Glob, Grep, Bash]` (`iva-qa-autotest-base/manifest.yaml:277`). На Claude Code это блокирует спавн субагентов. Для Евгения не блокер (он на Codex), но Claude-путь закрыт этой настройкой независимо от известного блокера с моделью субагентов.

**`.codex/config.toml` — `create_if_missing`** (`iva-role-qa/manifest.yaml:158`). Если в `one-web-kmp` этот файл уже есть, профильный конфиг (в т.ч. `[agents] max_threads=4 / max_depth=1` и запись `tacticum-mcp`) не приземлится. Мит: инструкция команде велит добавить блок руками (`12-Features/iva-role-qa — установка и работа (для команды).md:101`), так что как процесс закрыто — но «поставил и работает» не выполняется.

---

## 4. Соответствие отданного тому, что в репо

**Quickstart-а роли на main НЕТ.** `docs/user_manuals/` содержит 25 файлов, quickstart есть у каждой сестринской роли (`iva-role-analyst`, `iva-role-go`, `iva-role-ios`, `iva-role-java`, `iva-role-kmp`, `iva-role-mail`, `iva-role-web`, `tacticum-role-internal`, `tacticum-role-platform`), у `iva-role-qa` — нет. Файла `docs/user_manuals/iva-role-qa-profile-quickstart.md` не существует. Роль передана без репо-доки; вся инструкция живёт только в vault-заметке.

**Комплект передачи vs репо — сходится по всему, кроме одного.** Ссылки `:N` в колонке «Инструкция» — строки `12-Features/iva-role-qa — установка и работа (для команды).md`.

| Что | Инструкция команде | Факт на main | Сходится |
|---|---|---|---|
| Версии | роль 0.5.1, лейн 0.3.0, MCP 0.1.0, core 0.1.1 (`:24`, `:38`) | 0.5.1 / 0.3.0 / 0.1.0 / 0.1.1 | да |
| Стек | pytest + Playwright (canvas), `one-web-kmp` (`:47`, `:82`, `:84`) | лейн — да; **роль и её паки — one-web/Selenium** | **нет**, см. п.2 |
| Модель субагентов | `gpt-5.4`, не модель сессии (`:93`, `:135`) | `gpt-5.4` в manifest (`:248,262,276`) и в трёх `.codex/agents/*.toml:11`; `opus` в комплекте нигде, кроме `claude-settings` роли | да для субагентов; `defaultModel: opus` у Claude-сессии инструкцией не покрыт |
| Состав лейнов | core + автотест + qa-mcp | `depends_on:29-32` | да |
| 9 скиллов + 3 субагента | (`:202` и др.) | 9 skill_spec + craft-stack + 3 agent_spec + 2 config = 15 ingredients (`manifest.yaml:98-302`) | да |
| Env для секретов | `TESTOPS_ENDPOINT`/`TESTOPS_TOKEN`/`TESTOPS_PROJECT_ID` (`:145-147`), `JIRA_*` (`:218`) | те же (`secrets.yaml.example:7`); Jira — `JIRA_URL` + `JIRA_PERSONAL_TOKEN` (`iva-qa-mcp/manifest.yaml:110`) | да |
| «Окружение не поднимает» | (`:46`, `:92`) | `non_goals` лейна (`manifest.yaml:327`) | да |
| Канон стека / references | инструкция про них не говорит вообще | 45 файлов не доставляются | **умолчание** — инструкция не предупреждает |

**Codex-golden устарел и не покрывает Codex-путь.** `golden/iva-role-qa/codex.json` — 11 ключей: 8 скиллов в `.agents/skills/`, `AGENTS.md`, два `config.toml`. В нём **нет ни одного из трёх субагентов** `.codex/agents/*.toml` и **нет 5 скиллов** (`write-autotest`, `batch-autotest`, `fix-failed-test`, `jira-issue-autotest`, `craft-stack`) — то есть ровно того, что переделывали под Codex в 0.2.0. Это известный тех-долг (`checkpoint-lead-qa.md:38` — «golden codex.json реген: нужен Docker»), но следствие надо назвать: **ведущий для команды CLI-путь не покрыт golden-проверкой**. Живой smoke на стенде его частично закрыл (`checkpoint-lead-qa.md:27`), golden — нет.

---

## 5. Статус #119 (матрица ролей Глеба) — СМЕРЖЕН

- `1061d10` «Merge PR #119: переезд на архитектуру профилей лейны+роли (ADR-0059)» — в истории `main`.
- `_GENERIC_ROLES` — есть: `apps/backend/tests/e2e_install/test_install_flow.py:835`, параметризация `:899`, использование `:909`.
- `iva-role-qa` **подключён** в матрицу: `test_install_flow.py:870-874` (`present`: `write-autotest`, `run-tests`, `codebase-analyst`, `dom-explorer`, `iva-atlassian-write`, `helm-analyst`; `absent`: `coder`, `run-implementation`, `fr-authoring`, `bug-fix`), с комментарием `:869` «qa из main #133». Подключён коммитом `3473cbc` «merge main #133 (QA-профиль) + разрешение конфликтов тестовой матрицы».
- `test_role_install_smoke.py` — есть: `apps/backend/tests/catalog/test_role_install_smoke.py`, последний коммит `0420959` 2026-07-27 «паритет ролей сверяет тела, а не только идентификаторы».
- `golden/iva-role-go` — есть (`apps/backend/tests/e2e_install/golden/iva-role-go/`), `golden/iva-role-qa` — есть (оба файла от 24.07).

Тесты **не запускал** (read-only задача, backend-venv не поднимал) — это факт наличия символов и файлов, не факт зелёности.

---

## 6. Что блокировано отсутствием фидбека, а что нет

### Нельзя закрыть без прогона Евгения

1. **R1 «первый pull»** — `sync_count` 0→1 на прод-installation. Только его действие.
2. **Совпадение окружения с допущениями лейна:** наличие `tools/testops` в `one-web-kmp`, `autocore` в venv, `playwright-cli`/`allurectl`/`glab` в PATH и авторизованность `glab`, ОС и форма пути к python.
3. **Валидность `TESTOPS_*` и личных токенов** (Atlassian PAT, `phk_*`) на его контуре, доступность стендов `at-test-*.ivcs.su` из его сети.
4. **Работает ли `run-tests` на реальном `pytest.ini`** — существуют ли маркеры `browser_*`/`suite_*` и опции `--env`/`--browser-tag`/`--local` (грабля #2 подтверждается или снимается только его прогоном).
5. **Наличие в его репо предполагаемых конвенций:** `.kit/answers.toml`, `.kit/schema.md`, `.tasks/aut-research/AUT_OVERVIEW.md`, секция «Ветки и линии разработки» в `CLAUDE.md`.
6. **Есть ли уже `.codex/config.toml`** (влияет на `create_if_missing`).
7. **Приёмка «профиль включён в работу»** — по определению его прогон.

### Можно закрыть без него (проверено сейчас, по репо)

1. **Версии и цепочка композиции на main** — п.1, сходятся с комплектом передачи.
2. **Правка стека доехала частично** — п.2, 18 конкретных остатков с `file:line`. Правится в репо, фидбек не нужен.
3. **Отсутствие quickstart-а роли в `docs/user_manuals/`** — п.4. Пишется без него.
4. **Недоставка 45 файлов канона и references** — п.3 грабля #1. Подтверждена кодом платформы и golden-снапшотом; воспроизводится локально. Это дефект комплекта, не окружения.
5. **`tms:read` / `aql-notes.md` — висячие ссылки** — п.3 грабля #3. Проверяется grep-ом по нашему репо.
6. **Противоречие `run-tests` объявленному стеку** — п.3 грабля #2. Внутреннее противоречие лейна (`run-tests` vs `reproduce-phases.md:20` vs `rules/page-objects.md:153`) видно без его прогона; от Евгения нужен только ответ «падает или нет», не сам факт противоречия.
7. **`defaultModel: opus` и allow-list из 3 тулов в `claude-settings`** — п.3 «прочее». Расхождение с зафиксированным решением, читается из манифеста.
8. **Codex-golden не покрывает Codex-путь** — п.4. Факт состава `codex.json`; реген блокирован Docker'ом, а не фидбеком.
9. **Статус #119** — п.5, смержен.

---

## Не проверено (и почему)

- **Прод-каталог** (installation pin, `sync_count`, здоровье сервиса) — на прод-серверы не ходил, задача была про репо-сторону.
- **Зелёность тестов** (288 статики, e2e-матрица, smoke) — backend-venv не поднимал, тесты не запускал.
- **Содержимое репо `one-web-kmp`** — доступа нет. Всё про «есть ли там `tools/testops` / `.kit/` / `AUT_OVERVIEW.md` / маркеры pytest» — допущения лейна, а не проверенные факты.
- **ОС и раскладка каталогов у Евгения** — неизвестны; отсюда условность граблей #4.
---

# Дополнение (запрос lead-qa после проверки прода): 4 вопроса

## В1. Прод отстал или правки не было? — **(B)**

**Правка 24.07 применена только к лейну, роль забыли.** Прод НЕ отстал: на main та же 0.5.1 с тем же чужим стеком. Ре-сид ничего не исправит — исправлять надо сам шаблон роли.

Те же пять мест на main, `templates/iva-role-qa/manifest.yaml`:

| Место | Строка | Что стоит на main |
|---|---|---|
| `stack.optional` | `:75` | `[one-web]` — не `one-web-kmp` |
| `persona.scope` | `:60` | «автотестов **one-web** (генерация pytest/**Selenium**…)» |
| `description` | `:42`, `:47` | «генерация pytest/**Selenium**» · «Привязан к репо **one-web**» |
| `target_tasks` (usage) | `:63` | «сгенерировать pytest/**Selenium** автотест» |
| `post_install_notes` | `:111` | «⚠️ Автотест-лейн привязан к репо **one-web** (autocore/venv/tools.testops/glab/CI)» |
| `claude-settings` (пак) | `:166` | `{"defaultModel":"opus","tools":{"allowed":["Read","Edit","Bash"]}}` |

Плюс шапка-комментарий `:8` («pytest/Selenium») и **11 таких же мест в телах паков роли** (`CLAUDE.md.fragment`, `AGENTS.md.fragment`, `config.toml.template`) — перечислены в п.2 выше. Прод их не показал, потому что тела паков в манифесте не видны, но приземляются они в репо потребителя.

Прямое подтверждение, что это забывчивость, а не рассинхрон: `templates/iva-role-qa/CHANGELOG.md:9-13` — bump 0.5.1 сделан ТОЛЬКО ради переморозки `depends_on`, там дословно *«Роль и её ингредиенты не менялись»*. Лейн `iva-qa-autotest-base` 0.3.0 на main действительно чист (`stack.required: [one-web-kmp]` — `manifest.yaml:62`).

## В2. Версия роли на main — **0.5.1, ровно как в проде**

`templates/iva-role-qa/manifest.yaml:22`. Выше нет, ре-сид нечего давать. Последний коммит по каталогу роли — `1c6b7a9` от 2026-07-24, он же и есть тот самый bump. CHANGELOG роли: `[0.5.1]` — только переморозка ребра, `[0.5.0]` (23.07) — переход на пак уровня роли (ADR-0060).

## В3. `defaultModel: opus` — только Claude Code; субагенты `gpt-5.4` НЕ затронуты

**Подтверждаю по файлам.**

Задаётся: `templates/iva-role-qa/manifest.yaml:161-169` — ingredient `claude-settings`, `kind: repo_config`, **`supports: [claude-code]`**, `target_file: .claude/settings.json`, `merge_strategy: deep_merge`, тело inline на `:166`. Приземляется как `repo/.claude/settings.json` (есть в `golden/iva-role-qa/claude-code.json`; в `codex.json` его нет — Codex этот ингредиент не получает вовсе).

Область влияния — **дефолт модели сессии Claude Code**, отдельная сущность от модели субагентов. Модель субагентов задаётся независимо в лейне и `claude-settings` её не переопределяет:
- `iva-qa-autotest-base/manifest.yaml:248, 262, 276` — `model: gpt-5.4` в `agent_spec` каждого из трёх субагентов
- `ingredients/agents-codex/{codebase-analyst,dom-explorer,code-writer}.toml:11` — `model = "gpt-5.4"` в hand-authored codex-телах
- `iva-qa-autotest-base/manifest.yaml:237` и `CHANGELOG.md:149` — прежний хардкод `model: opus` у субагентов **снят** осознанно

Второе следствие того же тела, важнее модели: `tools.allowed` = `["Read","Edit","Bash"]`, без `Write`, `Glob`, `Grep` и **без `Task`**. Claude-путь всех трёх оркестраторов построен на `Task` (bare-имена субагентов), а самим субагентам объявлены `tools: [Read, Edit, Write, Glob, Grep, Bash]` (`iva-qa-autotest-base/manifest.yaml:277`). То есть на Claude Code этот allow-list блокирует и спавн субагентов, и запись новых файлов. Для Евгения не блокер (он на Codex — ингредиент `supports: [claude-code]`), но Claude-путь закрыт этой настройкой независимо от известного блокера «модель субагентов бейкается в `.claude/agents/*`».

## В4. `installation_id` — **грабля РЕАЛЬНАЯ, но в одной точке: первый pull, до установки скиллов**

### Механика подтверждена

`resolve_single_installation_for_membership` (`membership_installation_scope.py:22-64`) для `phk_*`-ключа считает установки **по организации**, а не по workspace: `.where(Organization.id == membership.organization_id)` (`:47`), с фильтрами non-archived + active profile (`:48-49`). Ровно одна → неявный резолв (`:53-54`). Ноль → `installation_id_required` (`:56-60`). **Два и больше → `installation_id_required`, «Membership keys cover multiple installations» (`:61-64`)** — та самая ошибка. При 13 живых установках неявный резолв не срабатывает никогда.

Вход в этот код — `resolve_caller_scope` (`scope_resolver.py:169-179`): пробует неявный резолв только когда `installation_id is None` И токен membership-овый.

Через `resolve_caller_scope` идут **все** `kb_*`, все `design_*`, `get_installation`, `check_updates`, `submit_artifact`/`submit_feedback`/`get_artifact` и — ключевое — **`pull_installation_content`** (`pull_installation_content.py:47` `installation_id: str | None = None`, `:66` `scope = await resolve_caller_scope(installation_id)`) и `pull_installation_content_manifest`.

`whoami()` через резолвер НЕ идёт — `whoami.py:32` без аргументов, читает `current_membership_scope` напрямую (`:45`) и перечисляет установки. Он не падает и служит escape hatch: из его вывода видно, какой id передать.

### Наши файлы: передают явно — но не в момент, когда это нужно

Что доставляется и учит правильно:
- `tacticum-core-base/ingredients/skills/kb-navigation/SKILL.md:16-24` — *«pass `installation_id` explicitly on EVERY call … a team `phk_*` token covering several installations fails with `installation_id_required`»*; `:33` — эта ошибка в таблице кодов с фиксом; `:54`, `:88`, `:116` — то же в процедурах
- `tacticum-core-base/ingredients/skills/tacticum-context/SKILL.md:101, 196` — вызовы `pull_installation_content_manifest(installation_id=…, target_cli=…)` с явным id; `:18-21` — резолв из `.tacticum/context.yaml`
- пак роли: `iva-role-qa/ingredients/repo-configs/claude-code/CLAUDE.md.fragment:27-28` и `codex/AGENTS.md.fragment:32-33` — «`installation_id` — из `.tacticum/context.yaml`, **передавай явно в каждый `kb_*`-вызов**»

Это не теория: прод уже ловил этот класс отказа — `tacticum-core-base/CHANGELOG.md:10` «incident 2026-07-22, **47× `installation_id_required` in prod**», и починка ушла именно в тело `kb-navigation` (версия 0.1.1).

**Где дыра.** Все перечисленные файлы приезжают ВМЕСТЕ С pull — то есть после того шага, на котором ошибка и происходит. Инструкция команде (`12-Features/iva-role-qa — установка и работа (для команды).md:120-129`) описывает первый шаг так: «Дайте Codex-агенту задачу *«Поставь профиль iva-role-qa»*. Агент сам: через `whoami` видит **вашу установку** профиля в каталоге; … вытягивает содержимое (`pull`)». Формулировка в единственном числе — допущение «установка одна». При 13 установках `whoami` вернёт 13, агент выбирает наугад или зовёт `pull` без id → `installation_id_required`. Указания «возьми id из `whoami` и передай в pull» в инструкции нет; во всём файле `installation_id` появляется только на `:239` — и то в curl регистрации токена, не в установке.

**Вердикт: факт.** Это грабля первого прогона для любого человека из workspace с >1 установкой, то есть для обоих QA-инженеров. Не фатальная и не про QA-профиль конкретно (дефект бутстрапа + умолчание инструкции), полностью обходится указанием передать id из `whoami`. Восстановление очевидное, ошибка самоописательная — но первый прогон Евгения она с высокой вероятностью прервёт, а «поставил и работает» не выполнится.

**Не проверено:** требует ли `installation_id` сервер `helm-analyst` (`https://helm.tacticum.ru/mcp/analyst`, лейн `iva-qa-mcp`) — это отдельный сервис, его кода в `tacticum-dev` нет. `iva-atlassian-write` — локальный `uvx mcp-atlassian`, к Tacticum-скоупу отношения не имеет. Скиллы автотест-лейна ни один Tacticum MCP не дергают вообще (`grep -rn 'kb_\|tacticum-mcp\|whoami' templates/iva-qa-autotest-base/ingredients/` пусто), так что для самой автотест-работы эта грабля не повторяется — только на установке и на `kb_*`-вызовах из core-скиллов.
