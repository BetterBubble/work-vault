---
title: Карта переноса kit → ингредиенты QA-лейна (две версии CLI)
type: note
status: current
created: 2026-07-28
tags:
- board
- qa
- kit
- transfer
permalink: tacticum/00-board/explore-kit-transfer-map-2026-07-28-1
archived-at: 2026-08-05 15:19
---

# Карта переноса kit в наши ингредиенты

Заказ lead-qa после GO Президента на пересборку QA-профиля от цельного среза `kit`.
Только чтение; в `~/tacticum/tacticum-dev` и в vault (кроме этой заметки) ничего не менялось.

**Источники и обозначения**

| Обозначение | Путь | Дата файлов |
|---|---|---|
| `KIT` | `…/scratchpad/kit/kit-main` (распаковка `90-Materials/kit-main.zip`) | 2026-07-22 |
| `TK` | `…/scratchpad/repos/aqa-agent-toolkit-main` | 2026-07-15 |
| `KMP` | `…/scratchpad/repos/one-web-kmp-main` | 2026-07-27 |
| `LANE` | `/Users/bubblemac/tacticum/tacticum-dev/templates/iva-qa-autotest-base` | 2026-07-24 |
| `DEV` | `/Users/bubblemac/tacticum/tacticum-dev` | — |

Полный префикс scratchpad: `/private/tmp/claude-501/-Users-bubblemac-tacticum-vault/598cba87-f04d-46da-9ae8-a23ed99f697e/scratchpad`.

Предшественники: `explore-toolkit-kmp-2026-07-28`, `qa-plan-svedeniya-steka-2026-07-28`.

---

## 0. Четыре поправки к постановке — читать до карты

Три допущения заказа проверены и не подтвердились (§0.1–0.3); четвёртая поправка (§0.4) — ревизия
карты под рамку «поверхностей десять», пришедшую после выдачи заказа. Всё это меняет объём
работы, поэтому идёт первым.

### 0.1 `model: gpt-5.4` в живой доставке Claude вообще НЕ попадает в файл

Живой путь установки — MCP-тулы `tacticum_init` / `tacticum_fetch_action` /
`pull_installation_content`, все они зовут `render_for_claude_code`
(`DEV/apps/backend/src/backend/catalog/interface/mcp/tacticum_init.py:35`,
`tacticum_fetch_action.py:34`, `tacticum_init_manifest.py:34`,
`DEV/apps/backend/src/backend/workspace/interface/mcp/pull_installation_content.py:29`).

`render_for_claude_code` для `agent_spec` идёт в ветку `elif kind in _WRITE_FILE_KINDS`
(`DEV/apps/backend/src/backend/catalog/domain/renderer.py:207-212`) и пишет **сырое body без
всякого frontmatter**. `metadata.model` и `metadata.tools` на этом пути не читаются вовсе.
Frontmatter в доставленном `.claude/agents/*.md` — тот, что уже лежит в файле-теле:
`LANE/ingredients/agents/codebase-analyst.md:1-4` несёт `name` + `description`, и **не несёт**
ни `model`, ни `tools`.

Итог: сегодня субагенты у потребителя Claude приезжают без модели (наследуют модель сессии)
и без ограничения тулов. `gpt-5.4` их не ломает — он туда не доезжает.

**Но `gpt-5.4` реально печётся на втором пути.** Канонический
`ClaudeCodeRenderer._render_agent`
(`DEV/apps/backend/src/backend/catalog/infrastructure/renderers/claude_code.py:58-71`) собирает
`---\nname:…\ndescription:…\nmodel: {a.metadata.model}\n{tools}\n---\n\n` + body. Этот путь
живёт в админском превью (`DEV/…/catalog/interface/admin/render_preview.py:50`) и в
`_render_via_canonical` для codex/copilot. Там `model: gpt-5.4` действительно уедет во
frontmatter Claude — и вдобавок **вторым блоком `---` поверх собственного frontmatter тела**
(тела наших субагентов свой блок несут). Это латентный дефект рендера, не только наш.

**Развести модель per-CLI механизма нет.** `AgentMetadata` — плоская модель с
`model: str | None` и `extra="forbid"`
(`DEV/apps/backend/src/backend/catalog/domain/ingredients/agent_spec.py:11-18`); полей вида
`claude_model` / `codex_model` нет. Per-CLI разводится только **тело целиком**, ключами
`codex_body_path` / `copilot_body_path` (`DEV/apps/backend/scripts/seed_community.py:75-91`,
потребитель — `renderer._cli_body_passthrough`, `DEV/…/domain/renderer.py:281-304`);
`claude_body_path` не существует и не нужен — общее body И ЕСТЬ Claude-тело.

**Отсюда лечение:** `metadata.model` — это поле Claude-слоя, и в нём обязан стоять Claude-id.
Codex свою модель берёт из hand-authored toml (`LANE/ingredients/agents-codex/codebase-analyst.toml:12`
→ `model = "gpt-5.4"`), который доставляется verbatim мимо `AgentMetadata`. Так сделано во всех
остальных лейнах: по всем 20 манифестам `templates/*/manifest.yaml` в `metadata.model` стоит
`sonnet` (46 раз) или `opus` (13 раз), а `gpt-5.4` — ровно **3 раза, и все три наши**.
Прецедент к копированию — `templates/iva-kmp-brownfield/manifest.yaml:362-366`
(`codex_body_path` + `model: opus`). Комментарий `LANE/manifest.yaml:236-237` («Ранее был
хардкод model: opus — снят») описывает ровно ту правку, которую надо откатить.

### 0.2 `claude-settings` роли — не QA-проблема и, скорее всего, мёртвый ключ

`{"defaultModel":"opus","tools":{"allowed":["Read","Edit","Bash"]}}` — это
`templates/iva-role-qa/manifest.yaml:166`. **Байт-в-байт та же строка стоит в 19 других ролях
и лейнах** (`iva-role-analyst:114`, `iva-role-go:116`, `iva-role-kmp:116`, `iva-role-web:109`,
`tacticum-dev-base:352`, `brownfield-task-workflow:280` и т.д. — 20 файлов всего). Это
общерепозиторная конвенция пака роли (ADR-0059, `docs/adr/0059-single-axis-process-lanes-and-role-packs.md:49`),
а не локальная настройка QA.

Форма ключей при этом **не совпадает с тем, что тот же репозиторий пишет в тот же файл**:
`ClaudeCodeRenderer._render_permission`
(`DEV/…/infrastructure/renderers/claude_code.py:145-157`) кладёт в `.claude/settings.json`
структуру `{"permissions": {"allow": [...], "deny": [...]}}`. Два разных контракта на один
файл — верным может быть только один. Свойство живого Claude Code я по коду репозитория
доказать не могу; **что могу утверждать: это внутреннее противоречие репозитория, и опираться
на `tools.allowed` как на работающий радиус нельзя.**

Практический вывод один и тот же при обеих гипотезах: если ключ мёртв — он не мешает, но и не
даёт ничего; если жив — он режет `Task`/`Write`/`Glob`/`Grep` и ломает спавн субагентов.
Поэтому блок надо либо убрать, либо переписать в `permissions.allow` с полным набором.
**Это правка на 20 файлов и решение lead-arch, а не QA** — в объём пересборки лейна её брать
нельзя, надо вынести отдельным вопросом.

### 0.3 `.kit/` у потребителя быть МОЖЕТ — и это самый дешёвый носитель параметров

`.kit/` в выгрузке KMP отсутствует не потому, что его нельзя завести, а потому, что никто не
ставил тощую базу kit. **Ничто не мешает нам самим доставить `.kit/answers.toml` обычным
`repo_config`-ингредиентом.** Механика подтверждена: `repo_config` пишется по
`metadata.target_file` без ограничений на путь
(`DEV/…/domain/renderer.py:178-188`, `claude_code.py:159-167`, `codex.py:190-197`).

Цена альтернативы огромна: `.kit/answers.toml` захардкожен в `KIT/tms/scripts/testops/params.py:19`
(`_ANSWERS_FILE = Path(".kit") / "answers.toml"`), а прозой на него ссылаются **58 мест** в
телах kit, плюс `.kit/schema.md` — 31 место, `.kit/local.md` — 21, `.kit/craft/feature-mapping.md` — 8.
Свой носитель = патч `params.py` + переписывание ~118 ссылок. Доставка по родному пути = ноль
правок и байт-идентичный `params.py`. Подробности — §4.

### 0.4 Поправка рамки: поверхностей десять, поэтому репо-зависимое — параметр

Уточнение рамки пришло после выдачи заказа: сегодня поверхность одна (`one-web-kmp` +
Playwright), но всего их **десять**. Значит всё репо-зависимое — env, пути, стек, маркеры,
имя vendored-библиотеки — обязано ложиться параметром, а не константой. Ниже — ревизия
карты под это требование.

**Хорошая новость: kit уже параметричен, и ровно в тех местах.** Ни одного хардкода стенда,
браузера или пути в телах скоупа нет — везде именованный параметр с дефолтом-референсом
(`KIT/craft/README.md:26-34`, `track/README.md:25-32`, `ship/README.md:28-40`,
`tms/skills/read/SKILL.md:70-75`). Хардкоды — это болезнь **нашего** лейна, не kit: 25
вхождений `at-test-1-one`/`at-test-3-one`, 20 вхождений `browser_{win,lin,mac}`, 30 вхождений
`$HOME/Projects/AT` (§5). Перенос от цельного среза их лечит сам по себе — при условии, что мы
не зашьём значения обратно в поставку.

**Что я в первой редакции предлагал зашить — и как это исправлено (правки внесены в §1.2, §4.1, §5):**

| Место | Было | Стало |
|---|---|---|
| `[craft].stack` | «фиксируем, для лейна константа» | параметр; ось выбора `stacks/<stack>/` |
| `[craft].default_env` | «`= kmp-stage`» | параметр, значение заполняет консьюмер |
| `[craft].vendored_dep` | «пусто, что нам и нужно» | параметр-выключатель оси; пусто только для текущей поверхности |
| `[track].raw_results` / `test_paths` / `parallel_unsafe_marker` | «совпадает с KMP» | дефолт kit остаётся дефолтом, значение — за консьюмером |
| `craft/stacks/pytest-selenium` | «не берём» | ось `stack`, выключена для текущей поверхности; цена включения посчитана |
| `vendor/**` | «не берём» | ось `vendored_dep`, то же |
| `rebuild-autocore` | «удалить без замены» | удалить наш (хардкоды пути); замена — `vendor:rebuild` по оси |
| `secrets.yaml.example` | «взять KMP-версию целиком» | секция `testops:` + вторая секция **плейсхолдером `<имя env>:`**, без кред `kmp-stage` |

**Плохая новость: механизма условной поставки у нас НЕТ.** Единственный фильтр установки —
`tier`: `allowed_tiers = ("trial",) if scope.tier == "trial" else ("trial","full")`
(`DEV/…/workspace/interface/mcp/pull_installation_content.py:86-90`). Он про объём (пробный
срез против полного), а не про ось. Поле `when` («Boolean expression… uses questionnaire
context») в схеме есть — но **только в v1 и без парсера**:
`templates/_schema/manifest.v1.schema.json:150-154` прямо пишет «v0 parser: always true»,
а в действующей v2 (`manifest.v2.schema.json`, 29 строк) его нет вовсе. Это тот же класс, что
`assets`/`scripts` (§2.2): поле объявлено, механики нет. **На `when` опираться нельзя.**

Отсюда три честных варианта включения оси — выбор за lead-arch, я фиксирую цену:

| Вариант | Как | Цена |
|---|---|---|
| **(1) Ось = отдельный лейн + `depends_on`** | `iva-qa-stack-selenium`, `iva-qa-vendored-dep` — свои профили, роль подключает нужные (ADR-0056, depth-1) | архитектурно чисто, ложится на существующую композицию; +1 профиль на ось, порядок в `depends_on` решает коллизии |
| **(2) Везти всегда, гасить параметром в теле** | так делает сам kit: `vendored_dep` пусто → блоки `[vendored-dep]` не активны (`KIT/craft/README.md:32`) | ноль механики, но +11 ингредиентов и +200.9 KB мёртвого груза за второй стек; для осей-текстов приемлемо, для второго стека — нет |
| **(3) Разные версии/профили лейна на поверхность** | `iva-qa-autotest-canvas`, `iva-qa-autotest-selenium` | дублирование 60+ общих ингредиентов; **не рекомендую** — это ровно та болезнь зеркал, от которой репо лечится `_mirrors.yaml` |

**Рекомендация: (1) для стека и `vendor`, (2) для всего остального.** Тела craft/track/ship/keep
стек-нейтральны by design (`KIT/craft/stacks/README.md`) — они читают `stacks/<stack>/`, и
переключение поверхности это смена значения параметра плюс наличие нужного каталога.

**Что от этого меняется в числах §2.4: ничего.** 86 ингредиентов — поставка под текущую
поверхность. Дельты осей поверх: `stack=pytest-selenium` → +11 `repo_config` (+200.9 KB);
`vendored_dep` непусто → +1 `skill_spec` +2 `repo_config` (+22.2 KB). Обе дельты изолированы в
свои каталоги и ни одного из 86 ингредиентов не трогают.

**Ещё две привязки к поверхности, которые надо снять в манифесте лейна** (обе — не kit, наши):

- `LANE/manifest.yaml:61-63` — `stack.required: [one-web-kmp]`. При десяти поверхностях
  обязательная привязка неверна: должно быть `required: []` + `optional`, иначе лейн
  формально неустановим на девять из десяти репо.
- `LANE/manifest.yaml:10-14, 42-43, 59-60, 326` — прозаические утверждения «лейн НЕ
  стэк-агностичен», «осмыслен только внутри one-web-kmp», `non_goals` про
  стек-агностичность. После переноса от цельного среза это перестаёт быть правдой: лейн
  становится агностичным, а привязку несёт `.kit/answers.toml` консьюмера. Переписать.

**Оговорка про `.kit/answers.toml` при десяти поверхностях.** Родной путь (§0.3) при этом
становится ещё правильнее: у каждой поверхности свой файл в своём репо, поставка одна и та же.
Стратегия `create_if_missing` (§4.2) — обязательна, а не предпочтительна: затирать заполненный
файл консьюмера при обновлении лейна нельзя.

---

## 1. Инвентарь kit под наш скоуп

`KIT` — 149 файлов. В скоуп «автотест-работа на стеке `pytest-playwright-canvas`» попадают
**89 файлов, 908 KB**.

### 1.1 Что берём (пофайлово)

**craft/skills — 27 файлов, 297.6 KB**

| Файл | Байт |
|---|---|
| `craft/skills/write/SKILL.md` | 19797 |
| `craft/skills/write/references/csv-parsing.md` | 3589 |
| `craft/skills/write/references/description-to-input.md` | 6942 |
| `craft/skills/write/references/feature-mapping.template.md` | 3103 |
| `craft/skills/write/references/input-sources.md` | 4724 |
| `craft/skills/write/references/sanitization.md` | 6367 |
| `craft/skills/write/references/tc-review.md` | 11755 |
| `craft/skills/fix/SKILL.md` | 29615 |
| `craft/skills/fix/references/batch-discovery.md` | 10110 |
| `craft/skills/fix/references/batch-report.md` | 14098 |
| `craft/skills/fix/references/deferred-handling.md` | 13836 |
| `craft/skills/fix/references/failure-localization.md` | 8031 |
| `craft/skills/fix/references/input-sources.md` | 12379 |
| `craft/skills/fix/references/known-issues.md` | 5279 |
| `craft/skills/fix/references/launch-input.md` | 9499 |
| `craft/skills/fix/references/reproduce-phases.md` | 10555 |
| `craft/skills/fix/references/verification-loop.md` | 10405 |
| `craft/skills/fix/references/xfail-verification.md` | 7995 |
| `craft/skills/batch/SKILL.md` | 18296 |
| `craft/skills/batch/references/conventions.md` | 12198 |
| `craft/skills/batch/references/phases.md` | 28013 |
| `craft/skills/batch/references/progress.md` | 6238 |
| `craft/skills/update/SKILL.md` | 17880 |
| `craft/skills/update/references/delivery.md` | 3033 |
| `craft/skills/update/references/detection-and-mapping.md` | 5754 |
| `craft/skills/update/references/diff-routing-guards.md` | 11251 |
| `craft/skills/issue/SKILL.md` | 13990 |

**craft/agents — 3 файла, 40.1 KB:** `code-writer.md` (13951), `codebase-analyst.md` (13015),
`dom-explorer.md` (14050).

**craft — rules/shared/references/evals — 6 файлов, 51.0 KB:** `rules/invariants.md` (5778),
`shared/coverage-ledger.template.md` (2654), `shared/jira-candidates.md` (6351),
`shared/ledger-and-deviations.md` (8476), `references/product-api/research_user.py` (12918),
`evals/EVALS.template.md` (16005).

**craft/stacks/pytest-playwright-canvas — 9 файлов, 102.1 KB:** `allure-raw-parser.md` (10832),
`batch-conventions.md` (4070), `failure-taxonomy.md` (17282), `fix-playbooks.md` (12327),
`pytest-runner.md` (6890), `recon.md` (11100), `rules/page-objects.md` (15457),
`rules/tests.md` (15192), `test-templates.md` (11420).
*(Предшественник называл 10 — десятым был `craft/stacks/README.md`, он общий на оба стека.)*

**track — 4 файла в скоупе, 63.8 KB:** `skills/run/SKILL.md` (26992), `skills/parse/SKILL.md`
(11998), `skills/retest/SKILL.md` (14054), `shared/pipeline-run.md` (10772).

**tms — 11 файлов, 68.0 KB.** Скилл: `skills/read/SKILL.md` (6887),
`skills/read/references/aql-notes.md` (2984), `launch-json.md` (4507), `tc-plain.md` (2462).
Клиент: `scripts/testops_cli.py` (994), `scripts/testops/__init__.py` (2242), `__main__.py`
(12233), `client.py` (7145), `launch.py` (12642), `params.py` (3369), `testcase.py` (14177).

**ship — 10 файлов в скоупе, 115.5 KB:** `skills/mr/SKILL.md` (29367),
`skills/mr/references/exclusion-patterns.md` (8521), `existing-target-mode.md` (7729),
`output-blocks.md` (15708), `skills/land/SKILL.md` (24316), `shared/commit-hygiene.md` (2942),
`shared/delivery-protocol.md` (13754), `references/worktree-model.md` (5241),
`templates/PROGRESS-template.md` (1407), `scripts/worktree_bootstrap.py` (9422).

**keep — 6 файлов в скоупе, 74.6 KB:** `skills/retro/SKILL.md` (32533),
`skills/refactor/SKILL.md` (14986), `skills/dashboard/SKILL.md` (2905),
`scripts/backlog_dashboard.py` (23855), `templates/TODO-INDEX.template.md` (1443),
`templates/TODO-task.template.md` (895).

**atlas — 2 файла в скоупе, 21.0 KB:** `skills/bump/SKILL.md` (12962),
`references/living-aut-map.md` (8044).

**audit — 2 файла в скоупе, 24.1 KB:** `skills/review/SKILL.md` (9166),
`agents/work-reviewer.md` (14898).

**base — 2 файла, 20.3 KB (не весь модуль):** `scripts/task_meta.py` (6490), `CONTRACT.md` (14305).
Это **жёсткая зависимость**, а не опция: `task_meta.py` вызывают `craft/skills/write/SKILL.md:90`,
`craft/skills/update/SKILL.md:51`, `craft/skills/batch/SKILL.md:37,96`,
`craft/skills/fix/SKILL.md:144`, `keep/skills/refactor/SKILL.md:88`,
`keep/skills/retro/SKILL.md:226,336`; `CONTRACT.md` назван контрактом меты в
`craft/skills/update/SKILL.md:51`.

**7 модульных `README.md`** (`craft`, `craft/stacks`, `track`, `ship`, `keep`, `atlas`, `audit`)
физически в скоупе, но **как ингредиенты не едут** — это документация kit-репо про свой модуль
и свою секцию answers. Их содержание уходит в наш `README.md` лейна.

### 1.2 Что НЕ берём и почему

| Группа | Файлов / KB | Почему |
|---|---|---|
| `craft/stacks/pytest-selenium/**` | 11 / 200.9 KB | **ОСЬ `stack`, не исключение.** Для текущей поверхности (`one-web-kmp`: один headless chromium) не активен. Включается значением параметра `stack` — цена включения 11 `repo_config`, +200.9 KB. См. §0.4 |
| `vendor/**` | 4 / 22.2 KB | **ОСЬ `vendored_dep`, не исключение.** Для `one-web-kmp` не активен (`autocore` ставится с внутреннего PyPI — `KMP/requirements.txt:17`, локальной пересборки нет). Поверхность с правимой библиотекой-зависимостью включает его параметром: цена — 1 `skill_spec` + 2 `repo_config` (`hooks/`), +22.2 KB. См. §0.4 |
| `dispatch/**` | 6 / 58.7 KB | кросс-вызов ролей Codex↔Claude — способность самого kit, не автотест-работа; тянет `.kit/presets/**` и `[roles.*]` |
| `base/**` кроме двух файлов | 13 / 95.6 KB | это машинерия установки `.kit/` (`install`/`update`/`check`/`contribute`, `render_base.py`, `update_kit.py`, `inventory_local.py`, `deliver_feedback.py`) — её роль у нас играет наш манифест |
| корневые `README.md`, `CLAUDE.md`, `scripts/render_codex.py`, `.claude-plugin/`, `.agents/`, `*/.{claude,codex}-plugin/plugin.json` | 22 | публикация marketplace-репо kit; в поставку потребителя не входит по определению |

---

## 2. Карта в наши ингредиенты

### 2.1 Проверенные ограничения платформы

**Лимитов на количество ингредиентов, длину body и глубину `target_file` — нет.**

- Схема манифеста `templates/_schema/manifest.v2.schema.json` — 29 строк, валидирует ТОЛЬКО
  `schema_version` (`:9`), `depends_on` (`:10-16`) и `ide_targets` (`:17-27`). Ключа
  `ingredients` в ней нет вообще, `additionalProperties: false` — только внутри `ide_targets`.
  Массив ингредиентов схемой не ограничен ничем.
- `ingredient.v1.schema.json` — ни `maxLength`, ни `maxItems`, ни паттерна на `target_file`
  (`:170-178`: только `"type":"string","minLength":1`).
- БД: `body` — `Text`, `metadata_` — `JSONB`
  (`DEV/…/catalog/infrastructure/models.py:204-206`). Ограничение только на `ingredient_id` —
  `String(128)`.
- Транспорт: манифестный путь отдаёт `(action_id, path, size_bytes)` без тел и добирает их
  поштучно `tacticum_fetch_action`; счётчик `expected_action_count`
  (`tacticum_init_manifest.py:132-138`) существует ровно затем, чтобы ловить потерю чанков
  MCP (issue #347) на больших поставках. Числового потолка в коде нет.
- Прецеденты в репо: максимум ингредиентов — **51** (`templates/iva-web-brownfield`), затем 49
  (`iva-rn-brownfield`, `iva-kmp-brownfield`). Максимальное тело — **70524 байт**
  (`templates/iva-fr-analyst/ingredients/skills/fr-authoring/SKILL.md`). Наш максимум —
  `craft/skills/fix/SKILL.md` 29615 байт, вдвое меньше существующего рекорда.

Итог: ~86 ингредиентов — это в 1.7 раза выше текущего рекорда репо, но **упирается не в код, а
в отсутствие прецедента**. Технического запрета нет; риск — операционный (время установки,
число round-trip'ов `fetch_action`).

**`merge_strategy: replace` для файла в ещё не существующем подкаталоге — работает, спец-обработки нет.**
Клиентский контракт применения описан в
`DEV/…/catalog/domain/builtins/tacticum_context_skill.md:115-136`: особо обрабатываются ровно
две стратегии — `create_if_missing` (пропустить, если файл есть) и `append_section`
(правка блока по `marker_id`). **Всё остальное, включая `replace` и `deep_merge`, падает в
`else: write_file(path, content)`** с комментарием «profile-owned path (.claude/skills/*,
.codex/agents/*, …) — overwrite freely». Каталоги создаёт Write-инструмент применяющего агента.
Дефолт `repo_config` — `deep_merge` (`DEV/…/domain/renderer.py:184`,
`domain/ingredients/repo_config.py:21`), для `.md`/`.py` он семантически неверен, хотя ведёт
себя так же — **ставить `replace` явно**.

### 2.2 Найденная дыра: вспомогательные файлы сегодня не доставляются вовсе

Это ключевой факт для оценки объёма. `seed_community._load_ingredient` читает **один** файл на
ингредиент — `body_path` (`DEV/apps/backend/scripts/seed_community.py:56-72`), рендер эмитит
**одно** `write_file` на ингредиент. Поля `assets`/`scripts` в схеме есть
(`ingredient.v1.schema.json:81-82`), но модель прямым текстом их дезавуирует:

> «Поля объявительные… **Доставки этих файлов сейчас нет** — рендер ставит только `body`.
> Не выдавать наличие поля за наличие механики.»
> — `DEV/apps/backend/src/backend/catalog/domain/ingredients/skill_spec.py:24-26`

Следствие для действующего лейна: `LANE/manifest.yaml` объявляет `craft-stack` одним
`skill_spec` с `body_path: ingredients/craft-stack/SKILL.md` (`:230`), а **15 файлов рядом с
ним** (`recon.md`, `rules/*`, `shared/*`, `test-templates.md`, `failure-taxonomy.md`, …)
ингредиентами не объявлены → у потребителя их нет. То же с `skills/fix-failed-test/references/*`
(10 файлов) и `skills/batch-autotest/references/*` (3). Карта файлов в
`LANE/ingredients/craft-stack/SKILL.md:18-36` ссылается на файлы, которые не приезжают.
`scripts/check_mirror_sync.py:54-58` обходит папку скилла целиком — но это **CI-сверка зеркал,
не доставка**.

Отсюда: `repo_config` с `target_file` — не «один из вариантов», а **единственный способ**
довезти 62 вспомогательных файла. Именно поэтому число ингредиентов вырастает с 15 до ~86.

### 2.3 Развилка домов: `.claude/` vs нейтральный дом

`_render_repo_config` **одинаков** у Claude и Codex: оба берут `metadata.target_file` дословно
(`claude_code.py:159-167` и `codex.py:190-197`). Значит `repo_config` с
`target_file: .claude/skills/craft-stack/recon.md` приедет по пути `.claude/…` и на Codex.

Отсюда два пути:

**(A, рекомендую) Нейтральный дом `/.agent-kit/<модуль>/…`, раскладка kit сохраняется.**
Один `repo_config` на файл, один путь на оба CLI: 62 ингредиента.
Главная выгода — **ссылки kit не переписываются вообще**. В телах скоупа: `$CRAFT` — 131
вхождение, `$TMS` — 18, `$SHIP` — 12, `$KEEP` — 8, `$BASE` — 7, `$ATLAS` — 2 (итого 178).
Это переменные, и определяются они по одной строке на файл: в 24 файлах скоупа есть ровно одна
строка вида «Путь к модулю ниже обозначен `$CRAFT`: в Claude Code это `${CLAUDE_PLUGIN_ROOT}`»
(`craft/skills/write/SKILL.md:12`, `tms/skills/read/SKILL.md:8` и т.д.). **24 однострочные
правки** переопределяют дом — и все 178 ссылок остаются валидны, включая
`$CRAFT/stacks/{stack}/…` (слой `stacks/` сохраняем, не уплощаем).

**(B) Дом в `.claude/skills/craft-stack/` (как сейчас).** Требует либо дублирования 62
ингредиентов под `supports: [codex]` с путём `.agents/…` (124 ингредиента), либо признания,
что Codex читает из `.claude/`. Плюс уплощение `stacks/{stack}/` ломает часть из 131 ссылки.
Это и есть источник сегодняшнего расхождения.

### 2.4 Итоговое число ингредиентов по типам

| `kind` | Штук | Что именно | Целевой путь |
|---|---:|---|---|
| `skill_spec` | **16** | craft: write/fix/batch/update/issue (5); track: run/parse/retest (3); ship: mr/land (2); keep: retro/refactor/dashboard (3); tms: read (1); atlas: bump (1); audit: review (1) | `.claude/skills/{id}/SKILL.md`; Codex — `.agents/skills/{id}/SKILL.md` |
| `agent_spec` | **4** | craft: codebase-analyst, dom-explorer, code-writer; audit: work-reviewer | `.claude/agents/{id}.md` + `.codex/agents/{id}.toml` через `codex_body_path` |
| `repo_config` | **65** | 62 файла-нагрузки kit (§1.1) + `.kit/answers.toml` + `.kit/schema.md` + `.gitignore` | `.agent-kit/<модуль>/…`; носители параметров — `.kit/…` |
| `instruction_pack` | **1** | `secrets.yaml.example` (`create_if_missing`) | `secrets.yaml.example` |
| `mcp_server_spec` | **0** | TestOps — не MCP, а CLI-пакет `testops`; остальное — внешние бинари | — |
| **Итого** | **86** | | |

Разбивка `repo_config` (65): craft references 22 · craft rules/shared/refs/evals 6 ·
craft/stacks/canvas 9 · track/shared 1 · tms references 3 · tms scripts 7 · ship (references,
shared, templates, scripts) 8 · keep (scripts, templates) 3 · atlas/references 1 ·
base (`task_meta.py`, `CONTRACT.md`) 2 · наши носители (`.kit/answers.toml`, `.kit/schema.md`,
`.gitignore`) 3.

Опционально +1 `skill_spec` — индекс-карта дома (наследник нынешнего `craft-stack`): описывает
`$CRAFT`/`$TMS`/… и карту файлов. Не обязателен, если определения переменных остаются в телах.

**Про `codex_target_path` без `codex_body_path` — это no-op.** `seed_community.py:79-81`:
`if cli_body_path is None: continue` — `codex_target_path` кладётся в metadata только вместе с
телом. У шести наших скиллов (`playwright-cli`, `run-tests`, `retro`, `prepare-mr-branch`,
`rebuild-autocore`, `craft-stack` — `LANE/manifest.yaml:127,138,195,206,217,229`) он объявлен
без тела и молча игнорируется. Совпадения он не ломает: `CodexRenderer._render_skill` жёстко
пишет `.agents/skills/{id}/SKILL.md` (`codex.py:78`) и `target_path_template` не смотрит —
результат тот же. Но декларация мёртвая, и в новой сборке её не надо повторять.

### 2.5 Имена ингредиентов

Kit зовёт скиллы `craft:write`, `track:run`. `ingredient_id` подставляется в путь
(`.claude/skills/{ingredient_id}/SKILL.md`), двоеточие в имени каталога — плохая идея.
Предложение: `craft-write`, `craft-fix`, `craft-batch`, `craft-update`, `craft-issue`,
`track-run`, `track-parse`, `track-retest`, `ship-mr`, `ship-land`, `keep-retro`,
`keep-refactor`, `keep-dashboard`, `tms-read`, `atlas-bump`, `audit-review`.
Все нынешние имена (`write-autotest`, `run-tests`, …) при этом меняются — **решение lead-qa**,
я фиксирую только техническое ограничение.

---

## 3. Два CLI: как kit и что чинит нашу claude-версию

### 3.1 Как kit разводит слои — **никак на уровне тел**

Разведка предшественника («Codex-слой стал проекцией живого CC») подтверждается, но с важным
уточнением: **проекцией стали МАНИФЕСТЫ, а не тела**.

`KIT/scripts/render_codex.py` (148 строк) рендерит ровно два вида файлов:
`<модуль>/.codex-plugin/plugin.json` и `.agents/plugins/marketplace.json` — из CC-манифестов
`<модуль>/.claude-plugin/plugin.json` и `.claude-plugin/marketplace.json`
(`render_codex.py:30-33, 64-106`). На тела скиллов он **указывает тем же путём**:
`native_plugin["skills"] = "./skills/"` (`render_codex.py:94-95`). Обе платформы читают
физически один и тот же `<модуль>/skills/<имя>/SKILL.md`.

Отдельных codex-тел в kit **нет**: `grep -r spawn_agent` по всему дереву — **0 совпадений**.
`KIT/craft/skills/write/SKILL.md:139` по-прежнему пишет «Вызови субагент `craft:codebase-analyst`
через `Task`». Расхождение провайдеров kit закрывает не текстом, а двумя механизмами:
контрактом fallback обязательного делегирования (`KIT/base/templates/schema.template.md:54-63`
— «платформа отказала в запуске дочерней роли до старта → основной агент исполняет шаг
локально… помечает `subagent-fallback: <роль>`») и модулем `dispatch` (кросс-вызов ролей).

**Вывод:** по разводке тел наши `LANE/ingredients/skills-codex/*` (4 дивергентных тела) и
`agents-codex/*` (3 toml) — **строго богаче kit, а не отставание от него**. Kit тут брать
нечего; забирать надо содержание, а механизм дивергенции оставить свой.

### 3.2 Что чинит claude-путь — конкретно

| № | Что | Где | Правка |
|---|---|---|---|
| 1 | `model: gpt-5.4` в трёх `agent_spec` | `LANE/manifest.yaml:248, 262, 276` | → Claude-id (`sonnet`/`opus`). Codex свою модель уже несёт в `agents-codex/*.toml:12`. На живом пути установки модель сейчас теряется вовсе (§0.1) — правка чинит превью/канон-рендер и снимает единственную аномалию репо (3 из 3 `gpt-5.4` — наши) |
| 2 | `tools:` не доезжает до `.claude/agents/*` | `DEV/…/domain/renderer.py:207-212` пишет сырое body; frontmatter тела (`LANE/ingredients/agents/*.md:1-4`) несёт только `name`+`description` | Вписать `tools:` **в сам frontmatter тела** — kit так и делает: `KIT/craft/agents/codebase-analyst.md:4` → `tools: Read, Glob, Grep`. Тогда ограничение работает на живом пути, а не только в каноне |
| 3 | Спавн субагентов через `Task` | тела `LANE/ingredients/skills/{write-autotest,batch-autotest,fix-failed-test}/SKILL.md:134,151,164` уже зовут `Task` корректно | Само по себе **не сломано**. Сломать может только радиус разрешений — см. п.4 |
| 4 | Радиус тулов роли: `{"tools":{"allowed":["Read","Edit","Bash"]}}` — без `Task`/`Write`/`Glob`/`Grep` | `templates/iva-role-qa/manifest.yaml:166` **и ещё 19 файлов** | Либо снять блок, либо переписать в `permissions.allow` (форма, которую пишет `claude_code.py:145-157`) с полным набором `Read, Write, Edit, Glob, Grep, Bash, Task`. **Правка общерепозиторная — вне объёма QA-лейна**, вынести в решение lead-arch (§0.2) |
| 5 | Двойной frontmatter на каноническом пути | `claude_code.py:58-71` клеит свой блок `---` поверх блока, который тело уже несёт | Дефект рендера, не лейна. Зафиксировать как отдельный баг платформы |
| 6 | 62 вспомогательных файла не доставляются | §2.2 | Объявить `repo_config`-ами. Это и есть основной объём пересборки |

Заметить отдельно: пункты 1, 2, 5 — про **frontmatter**, и их корень один — два рендер-пути
Claude, которые не согласованы между собой (`domain/renderer.py` для живой установки против
`infrastructure/renderers/claude_code.py` для превью/канона). ADR-0021
(`docs/adr/0021-hybrid-render-architecture.md:12`) описывает как правильный именно второй
(«render должен обернуть его в `---\nname:…\nmodel:…\n---` frontmatter»), а работает первый.

---

## 4. Носитель параметров вместо `.kit/answers.toml`

### 4.1 Полный список параметров

Собран по `KIT/base/templates/answers.template.toml` и разделам «Параметры проекта» в README
модулей. **Столбец «куда у нас»** — предложение; обоснование после таблиц.

**`[craft]`** (`KIT/craft/README.md:26-34`) — ядро генерации:

| Ключ | Дефолт kit | Кто читает | Куда у нас |
|---|---|---|---|
| `stack` | `pytest-playwright-canvas` | `craft/skills/write:14`, `fix:16`, `update:10`, `batch:10`, `issue:10`, все 3 субагента (`code-writer:11`, `codebase-analyst:12`, `dom-explorer:13`), `audit/agents/work-reviewer:26` | **`.kit/answers.toml`, параметр — НЕ константа лейна.** Ось выбора каталога `stacks/<stack>/`; сегодня активна одна реализация, поставка второй — вопрос объёма, не архитектуры (§0.4) |
| `default_env` | `""` (референс `kmp-stage`) | `track/skills/run/SKILL.md:20`, стек | `.kit/answers.toml`, **значение заполняет консьюмер**. Имя стенда — свойство репо, в поставку не зашивается |
| `shared_users` | `[]` | `craft/agents/dom-explorer.md:78`, `write/references/sanitization.md:19`, `description-to-input.md:47-48` | `.kit/answers.toml`, **значения заполняет консьюмер** — это логины стенда, не секреты |
| `research_user_cmd` | `""` | `craft/agents/dom-explorer.md:71-74`, `ship/skills/land/SKILL.md:230` | `.kit/answers.toml`; для KMP пусто (фабрики нет) |
| `vendored_dep` | `""` (референс `autocore`) | `ship/shared/delivery-protocol.md:25`, `ship/skills/mr/references/output-blocks.md:11`, стек | `.kit/answers.toml`, заполняет консьюмер. **Это выключатель оси:** пусто → блоки `[vendored-dep]` и модуль `vendor` не нужны; непусто → ось включается (§0.4). Для `one-web-kmp` пусто |
| `product_clone` | `""` | `craft/stacks/pytest-playwright-canvas/recon.md:116`, `rules/tests.md:19` | `.kit/answers.toml`, заполняет консьюмер |

**`[tms]`** (`KIT/tms/skills/read/SKILL.md:70-75`, дефолты — `params.py:23-33`) — единственная
секция, читаемая **кодом**:

| Ключ | Дефолт | Куда у нас |
|---|---|---|
| `wont_automate_tag` | `wont-automate` | `.kit/answers.toml`, дефолт kit; перекрывает консьюмер |
| `skip_attachments` | `["Видео"]` | то же |
| `browser_marker_pattern` | `browser_(?:win\|lin\|mac\|android\|ios)_\w+` | то же. Для поверхности без browser-маркеров (`one-web-kmp`) `browser_marker()` вернёт `None` — штатная деградация, не поломка; поверхности с гридом дефолт подходит как есть |

**`[track]`** (`KIT/track/README.md:25-32`):

| Ключ | Дефолт | Куда у нас |
|---|---|---|
| `raw_results` | `./allure-raw` | `.kit/answers.toml`, дефолт kit оставляем как дефолт; заполняет консьюмер (для `one-web-kmp` дефолт совпадает с `KMP/pytest.ini:9`) |
| `test_paths` | `["tests/","tools/"]` | `.kit/answers.toml`, дефолт kit; заполняет консьюмер (для `one-web-kmp` дефолт совпадает) |
| `parallel_unsafe_marker` | `parallel_unsafe` | `.kit/answers.toml`, дефолт kit; заполняет консьюмер (в `KMP/pytest.ini` маркер объявлен) |
| `parallel_default` | `4` | `.kit/answers.toml` |
| `launch_name_template` | `""` | **оставить пустым**: CI-контура в выгрузке KMP нет; при пустом параметре `track/shared/pipeline-run.md:68` предписывает спросить пользователя, а не угадывать |

**`[ship]`** (`KIT/ship/README.md:28-40`):

| Ключ | Дефолт | Куда у нас |
|---|---|---|
| `vcs_cli` | `glab` | `.kit/answers.toml` |
| `extra_exclusions` | `[]` | `.kit/answers.toml` |
| `dep_aliases` | `[]` | пусто (нет vendored-dep) |
| `link_dirs` | `["venv"]` | `.kit/answers.toml` |
| `link_files` | `["secrets.yaml"]` | `.kit/answers.toml` |
| `[ship.branches."<ветка>"]` → `line`, `env` | — | `.kit/answers.toml`, заполняет консьюмер |

**`[project]`** (`KIT/base/templates/answers.template.toml:11-19`) — общая база:
`name`, `main_branch`, `agent_branch`, `lines` (`single`/`dual`), `python`, `runner`,
`worktrees_root`, `kit_channel`. Все → `.kit/answers.toml`; `lines = "single"`,
`kit_channel` — нерелевантен (канал marketplace kit), можно опустить.

**`[kit]`**: `schema_version`, `base_version` — служебные для `render_base.py`. Мы этот скрипт
не везём → **опустить или поставить фиксированные значения** как метку среза (см. §6).

**Секции вне скоупа:** `[capacity].profile`, `[strictness].auto_counter_review`,
`[roles.pipeline_writer|fog_writer|scout|reviewer]` (`carrier`/`tier`/`effort`/`permissions`),
`[dispatch].timeout_sec`, `[contribute]` (`core_repo`, `feedback_dir`, `inventory_roots`,
`generated_globs`), `[vendor]` (`dep_repo`, `dep_name`) — обслуживают модули `dispatch`,
`base:contribute` и `vendor`, которых мы не берём.
**Оговорка:** `[roles.*]` косвенно нужны — на «секцию Роли» `.kit/schema.md` ссылается контракт
fallback делегирования (`craft/skills/write/SKILL.md:21-22`). Если `.kit/schema.md` мы
доставляем статикой (см. ниже), сами `[roles.*]` в answers не нужны.

### 4.2 Куда это селить — рекомендация

**Носитель: `.kit/answers.toml` как обычный `repo_config`-ингредиент нашего лейна.**
Одна декларация:

```yaml
- ingredient_id: kit-answers
  kind: repo_config
  metadata:
    target_file: ".kit/answers.toml"
    merge_strategy: create_if_missing   # см. оговорку ниже
```

Основания:
1. `params.py:19` читает путь константой — при родном пути пакет `testops` едет **байт-в-байт**,
   без нашего форка. Это ровно тот класс правок, за который `TK/…/reference-allure-testops/README.md`
   ругает форки апстрима.
2. Прозаические ссылки в телах остаются истинными: `.kit/answers.toml` — 58 вхождений,
   `.kit/schema.md` — 31, `.kit/local.md` — 21, `.kit/craft/feature-mapping.md` — 8.
   Свой носитель = ~118 правок в телах + патч `params.py`, и это в чистом виде форк.
3. Файл — свойство проекта, не секрет: `params.py:10-12` прямо разделяет — «параметр живёт в
   git, секрет — вне git».

**Оговорка по стратегии.** `create_if_missing` защищает уже заполненный файл потребителя от
затирания при обновлении лейна (клиентский контракт — `tacticum_context_skill.md:117-120`), но
тогда новые ключи в новых версиях лейна к нему не доедут. Альтернатива — `replace` + завести
рядом `answers.example.toml`. **Развилка на lead-qa.** Я бы взял `create_if_missing`: файл
заполняется реальными логинами стенда, затирать его нельзя.

**Спутники, которые придётся доставить вместе:**

| Файл | Зачем | Как |
|---|---|---|
| `.kit/schema.md` | 31 ссылка; главное — контракт fallback делегирования (`KIT/base/templates/schema.template.md:54-63`) и гейты (`:70-76`) | **`repo_config` со статикой**: отрендерить шаблон один раз под наши ответы (плейсхолдеры `{{capacity.profile}}`, `{{roles_table}}`, `{{strictness_table}}`) и везти готовым файлом. `render_base.py` не везём |
| `.kit/craft/feature-mapping.md` | 8 ссылок (роутинг теста по фиче) | **ингредиент не нужен**: сеется первым прогоном `craft:write` из `references/feature-mapping.template.md` — а этот шаблон мы везём как `repo_config` |
| `.kit/local.md` | 21 ссылка — местные конвенции проекта | **не везём**: файл проекта, авторы — люди/агенты проекта (`KIT/base/templates/schema.template.md:18`). Упомянуть в `post_install_notes` |
| `.kit/presets/**` | 5 ссылок | вне скоупа (модуль `dispatch`) |
| `secrets.yaml` | значения TestOps | остаётся как есть: env `TESTOPS_*` → gitignored `secrets.yaml`, форма — `secrets.yaml.example`. **Ключи `[tms]` и секреты не смешивать** |

`.kit/answers.toml` читают, кроме `params.py`, ещё два скрипта из скоупа —
`ship/scripts/worktree_bootstrap.py:69-76` и (через `[contribute]`) `base/scripts/*`.
Первый с родным путём тоже поедет без правок.

---

## 5. Что из текущего лейна умирает

`LANE` — 15 ингредиентов, 67 файлов в `ingredients/`. Замеры по доставляемым телам
(`grep -rn … ingredients/`): `tools.testops` — **26** вхождений, `tms:read` — **24**,
`.kit/*` — **31** (17 `answers.toml`, 10 `feature-mapping.md`, 6 `schema.md`),
`at-test-1-one|at-test-3-one` — **25**, `browser_{win,lin,mac}` — **20**,
`$HOME/Projects/AT` — **30**, `AUT_OVERVIEW` — **4**.

### 5.1 Удаляем / заменяем

| Ингредиент | Судьба | Основание |
|---|---|---|
| `run-tests` (193 стр.) | **заменить** на `track:run` | матрица браузеров, `--browser-tag`/`--local`, `at-test-1-one`, selenoid-параллель — ничего этого в KMP нет (`KMP/conftest.py:366-370` — ровно три опции `--env`/`--lang`/`--headed`; `KMP/config.yaml:7-13` — один стенд `kmp-stage`). `KIT/track/skills/run/SKILL.md:22` уже несёт ветку вырождения одно-браузерного стека |
| `rebuild-autocore` (162 стр.) | **удалить**; замена — `vendor:rebuild` по оси, не в этой поставке | 30 хардкодов `$HOME/Projects/AT/…` — путь к клону библиотеки обязан быть параметром `[vendor].dep_repo`, а не константой. Для `one-web-kmp` ось выключена (`autocore` — pip-пакет, `KMP/requirements.txt:17`); поверхность с правимой зависимостью включает модуль `vendor` (§0.4) |
| `playwright-cli` (335 стр.) | **удалить без замены** | вендорённый снапшот апстрима с вписанными стендами one-web и требованием `AUT_OVERVIEW.md`; `TK/adapters/browser-driver/reference-playwright-cli/README.md:5-14` называет ровно эту практику антипаттерном. Роль инструмента в canvas-стеке описана узко и на месте: `KIT/craft/stacks/pytest-playwright-canvas/recon.md:130-134` («только для DOM-поверхностей… на canvas-узлах `click <ref>` мёртв») |
| `jira-issue-autotest` (98 стр.) | **заменить** на `craft:issue` | резолв стенда по ветке, `TEST_BROWSER:browser_lin_chrome`, пайплайн `glab` при отсутствующем CI-контуре |
| `write-autotest` / `batch-autotest` / `fix-failed-test` | **заменить** на `craft:write` / `craft:batch` / `craft:fix` + их references | тела уже из kit, но **без 13 файлов references** и с разрешёнными вручную `$CRAFT`/`$TMS`; после переноса ссылки становятся родными |
| `prepare-mr-branch` (283 стр.) | **заменить** на `ship:mr` + 3 references + `shared/delivery-protocol.md` + `commit-hygiene.md` | |
| `retro` (107 стр.) | **заменить** на `keep:retro` (32533 байт — сильно полнее) | |
| `craft-stack` (skill_spec) | **распустить** | его 15 файлов становятся отдельными `repo_config` в нейтральном доме; сам SKILL.md — либо индекс-скилл, либо удаляется |
| `craft-answers.example.toml` | **заменить** на реальный `.kit/answers.toml` (§4) | плейсхолдеры вместо носителя |
| ссылки `tools/testops` (26) | **умирают вместе с телами** | путь отменён 2026-07-05 (`TK/docs/DESIGN.md:17`); клиент едет как `repo_config` в `.agent-kit/tms/scripts/testops/` |
| `codex_target_path` без `codex_body_path` (6 шт.) | **не воспроизводить** | молча игнорируется (`seed_community.py:79-81`), §2.4 |

### 5.2 Оставляем

| Ингредиент | Почему |
|---|---|
| 3 `agent_spec` (`codebase-analyst`, `dom-explorer`, `code-writer`) | те же роли, тела освежаются из `KIT/craft/agents/*` (там есть `tools:` во frontmatter — `codebase-analyst.md:4`); правится `metadata.model` (§3.2 п.1) |
| `agents-codex/*.toml` (3) | механизм дивергенции Codex — наш и он богаче kit (§3.1); тела перегенерировать под новые `.md` |
| `skills-codex/*` (4 дивергентных тела) | то же; перегенерировать под новые тела craft/track |
| `secrets-example` (`instruction_pack`, `create_if_missing`) | env-модель верна и совпадает с апстримом. Наша версия несёт только секцию `testops:` (`LANE/ingredients/config/secrets.yaml.example`, 17 строк) — **добавить вторую секцию, но ПЛЕЙСХОЛДЕРОМ `<имя env>:`**, а не кредами `kmp-stage`. Форма секции — `KMP/secrets.yaml.example:15-23`; значения и имя env — дело консьюмера |
| `gitignore-secrets` (`repo_config`, `append_lines`, marker) | безвреден и идемпотентен; в `.gitignore` KMP запись уже есть (`KMP/.gitignore:34`). Добавить туда же `.agent-kit/`? — **нет**: дом должен быть в git, иначе не переживёт clone |

---

## 6. Провенанс

**Готового поля в схеме нет.** `templates/_schema/manifest.v2.schema.json` не описывает ни
`provenance`, ни `source`, ни `upstream` (весь файл — 29 строк, §2.1). Грep по
`templates/*/manifest.yaml` на `provenance|upstream_ref|source_commit` — ноль совпадений.
`seeded_from_commit` (`DEV/…/catalog/infrastructure/models.py:125`) заполняется из
`os.environ["GITHUB_SHA"]` (`seed_community.py:121`) — это коммит **нашего** репо, про kit он
ничего не знает.

**Соглашение вводить можно и дёшево.** Верхний уровень схемы не закрыт
(`additionalProperties` на корне не задан, `required: ["schema_version"]`), а весь YAML
целиком уезжает в БД: `_build_payload` кладёт `manifest=manifest` (`seed_community.py:119`) в
JSONB-колонку `ProfileVersion.manifest` (`models.py:119`). Значит **любой новый верхнеуровневый
ключ переживёт seed и будет доступен следующей синхронизации**.

Предложение — блок в `LANE/manifest.yaml`:

```yaml
provenance:
  upstream: "git.hi-tech.org/ivaqa/kit"
  snapshot_date: "2026-07-22"
  snapshot_source: "90-Materials/kit-main.zip"
  snapshot_sha256: "<sha256 архива>"
  modules: [craft, track, tms, ship, keep, atlas, audit, base(partial)]
  excluded: [vendor, dispatch, "craft/stacks/pytest-selenium", "base(installer)"]
  local_deltas: "см. README.md §Provenance"
```

Точной версии/коммита kit в архиве нет: `<модуль>/.claude-plugin/plugin.json` несёт `version`
модуля, но не git-ревизию. Поэтому якорь — **sha256 самого архива + дата файлов (2026-07-22)**.

Хвост: `LANE/README.md:58-71` уже держит раздел «Provenance» прозой, но он описывает **прежний**
перенос (byte-copy от QA-команды + частичный kit) и после пересборки станет ложью — переписать
тем же PR. Формальный блок в манифесте — машиночитаемый дубликат, README — человеческий.

---

## 7. Порядок кусков

### 7.1 Жёсткая точка сериализации

`scripts/check_profile_version_discipline.py` — pre-merge гейт: **любое** изменение контента в
каталоге профиля обязано в том же диффе поднять `manifest.yaml` `version:` и добавить
`## [<version>]` в `CHANGELOG.md` (докстринг `:7-16`). Значит `manifest.yaml` и `CHANGELOG.md` —
**общий ресурс всех воркеров**. Параллелить надо по файлам-телам, а манифест сводить одному.

Хорошая новость: `iva-qa-autotest-base` **не участвует в зеркалах** — в
`templates/_mirrors.yaml` его нет, обязательства байтовой сверки с чужим профилем отсутствуют.
Пересборка свободна.

### 7.2 Порядок

**Куски 1 и 2 — параллельно, первыми (разблокируют инженера).**

| # | Кусок | Файлы (непересекающиеся) | Зависимости |
|---|---|---|---|
| **1** | **Клиент TestOps** — 7 `repo_config`: `tms/scripts/testops_cli.py` + пакет `testops/` (6 файлов) → `.agent-kit/tms/scripts/…`; плюс `tms:read` (`skill_spec`) и 3 его references | `ingredients/tms/**` | нет |
| **2** | **`track:run`** — `skill_spec` + `track/shared/pipeline-run.md` (`repo_config`); заодно `track:parse`, `track:retest` | `ingredients/track/**` | мягкая на #1 (тела зовут `$TMS/scripts/testops_cli.py`) — **не блокирует**, путь фиксируется соглашением дома |

**Кусок 0 (полчаса, но строго до 1 и 2):** зафиксировать письменно **соглашение о доме** —
`$CRAFT`/`$TMS`/`$SHIP`/`$KEEP`/`$BASE`/`$ATLAS` → `.agent-kit/<модуль>/`, и решение по
`.kit/answers.toml`. Без этого куски 1-2 разъедутся по путям.

**Дальше — параллельно, каталоги не пересекаются:**

| # | Кусок | Файлы | Зависимости |
|---|---|---|---|
| **3** | craft: 5 `skill_spec` + 22 references | `ingredients/craft/skills/**` | #0 |
| **4** | craft: 3 `agent_spec` + `agents-codex` + rules/shared/refs/evals (6) | `ingredients/craft/agents/**`, `ingredients/craft/shared/**` | #0 |
| **5** | стек canvas: 9 `repo_config` | `ingredients/craft/stacks/**` | #0 |
| **6** | ship: 2 `skill_spec` + 8 `repo_config` | `ingredients/ship/**` | #0 |
| **7** | keep: 3 `skill_spec` + 3 `repo_config` | `ingredients/keep/**` | #0 |
| **8** | atlas + audit: 2 `skill_spec`, 1 `agent_spec`, 1 `repo_config` | `ingredients/atlas/**`, `ingredients/audit/**` | #0 |
| **9** | носители параметров: `.kit/answers.toml`, `.kit/schema.md` (статика), `base/task_meta.py`, `base/CONTRACT.md`, `secrets.yaml.example` (KMP-версия) | `ingredients/config/**`, `ingredients/base/**` | #0 |

**Строго последовательно, в конце и одним исполнителем:**

| # | Кусок | Почему нельзя параллелить |
|---|---|---|
| **10** | `manifest.yaml` целиком: ~86 ингредиентов, `provenance`, `version` bump, `ide_targets`, `post_install_notes`, `non_goals` | один файл, гейт версий |
| **11** | `CHANGELOG.md` + `README.md` (переписать §Provenance) | тот же гейт |
| **12** | `codex_body_path`-тела: перегенерировать `skills-codex/*` и `agents-codex/*` под НОВЫЕ тела | зависит от 3, 4, 6, 7 — раньше делать нечего |
| **13** | Прогон `scripts/check_profile_version_discipline.py`, `scripts/check_mirror_sync.py`, `pytest apps/backend/tests/catalog/` | финальная сверка |

**Вне объёма лейна — отдельным решением lead-arch:** `claude-settings` в 20 файлах (§0.2) и
двойной frontmatter канонического рендера (§3.2 п.5).

**Куски, добавленные поправкой §0.4** (в первую волну не входят, но и не откладываются на потом):

| # | Кусок | Когда | Зависимости |
|---|---|---|---|
| **10a** | Снять привязку к поверхности в манифесте: `stack.required: [one-web-kmp]` → `required: []` + `optional`; переписать прозу «не стэк-агностичен» (`LANE/manifest.yaml:10-14, 42-43, 59-60, 61-63, 326`) | внутри куска 10 | тот же файл, тем же исполнителем |
| **9a** | `secrets.yaml.example`: добавить вторую секцию **плейсхолдером** `<имя env>:` по форме `KMP/secrets.yaml.example:15-23`, без кред `kmp-stage` | внутри куска 9 | — |
| **14** | Решение по механизму осей: отдельные лейны `iva-qa-stack-*` / `iva-qa-vendored-dep` против «везти и гасить» (§0.4) | **после** 1–13, отдельным заходом | решение lead-arch; `when` в схеме мёртв — опираться нельзя |

Оси в первую волну **не берём сознательно**: пока поверхность одна, дельты (+11 `repo_config`
за второй стек, +3 за `vendor`) не проверить в бою, а решение о механизме — архитектурное.
Важно другое: первая волна не должна создать препятствий для осей — этого достигает нейтральный
дом `.agent-kit/<модуль>/` с сохранённым слоем `stacks/<stack>/` (§2.3, вариант A).

---

## 8. Чего не нашёл

- Точной **версии/коммита kit** — в архиве только версии модулей в `plugin.json`, git-ревизии нет.
- **Подтверждения по коду**, что живой Claude Code читает `tools.allowed` из `.claude/settings.json`.
  Зафиксировано только внутреннее противоречие репозитория (§0.2).
- **Лимитов** на число ингредиентов / длину body / глубину `target_file` — их нет ни в схемах,
  ни в моделях, ни в MCP-слое (§2.1). Это «не найдено» = «отсутствует», а не «не проверял».
- `.tasks/README.md` — карта раскладки, на которую ссылаются 8 мест kit: это **ручной файл
  проекта** (`KIT/keep/README.md:40`), в поставке kit его нет и у нас его не будет.