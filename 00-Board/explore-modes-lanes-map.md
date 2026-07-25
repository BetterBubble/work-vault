---
title: explore-modes-lanes-map
type: note
permalink: tacticum/00-board/explore-modes-lanes-map
tags:
- draft
- explore
- lead-modes
- adr-0057
- adr-0059
---

# explore-modes-lanes-map

status: draft
Разведка explorer'а под ТЗ#2 Солонко (режимы workflow + двухслойный гейт).
Репо: /Users/bubblemac/tacticum/tacticum-dev, ветка main, HEAD 20412ff.
Все пути ниже — относительно корня репо.

## Резюме (одной строкой)
Ось разбиения профилей — **процессы-лейны** (ADR-0057/0059): `Роль = core + процесс-лейны + стек-лейн + способность-лейны`, композиция через `depends_on` (depth-1, single-owner, ADR-0056). «Режим» ТЗ#2 ложится на существующую ось лейнов. Отдельного mode/gate-механизма НЕТ — гейты сейчас живут внутри команд/агентов как текст промпта.

---

## 1. Как устроен лейн (схема манифеста + композиция роли)

**Схемы:** `templates/_schema/manifest.v2.schema.json`, `templates/_schema/ingredient.v1.schema.json`.

Манифест v2 (`schema_version: "2"`), ключевые поля:
- `depends_on` — массив id базовых лейнов, **порядок = приоритет** при коллизии ingredient_id (later base wins, profile wins all). minItems 1, unique.
- `ide_targets` — per-CLI: `claude-code|codex|opencode|gemini|copilot` → `full|best-effort|unsupported`.
- `ingredients[]` — список кирпичей, дискриминатор `kind`.

**9 kind'ов ингредиентов** (ingredient.v1): `instruction_pack, rule_set, agent_spec, skill_spec, mcp_server_spec, command_spec, hook_spec, permission_policy, repo_config`. У каждого своя `metadata`-схема:
- `command_spec` → metadata.{name, description, args_schema?}; тело в `body_path`.
- `skill_spec` → metadata.description_trigger; тело `body_path` (SKILL.md).
- `agent_spec` → metadata.{model, tools, description, permissions_ref?, delegation?}; тело `body_path` (claude .md) + `codex_body_path`/`codex_target_path` (codex .toml).
- Поля размещения на диске: `install_scope` (user|repo), `target_path_template`, `codex_target_path`, `body`/`body_path`, `tier` (trial|full).

**Поле `codex_body_path`** есть только у agent_spec — раздвоение тела агента: claude-версия (`body_path` → `ingredients/agents/<id>.md`) и codex-версия (`codex_body_path` → `ingredients/agents-codex/<id>.toml`). Пример: iva-analysis-base ingredient `tacticum-workflow`.

**Пример роли `templates/iva-role-go/manifest.yaml`** — implement-only пресет (ADR-0059 Решение 6). `depends_on`:
```
- tacticum-core-base        # общий минимум
- tacticum-development-core # процесс реализации (/run-*)
- iva-go-development-base   # стек Go
- tacticum-bugfix-base      # починка
```
Роль несёт **РОВНО пак-ингредиенты** (маркер `tacticum:iva-role-go`): claude-md-fragment (instruction_pack), codex-agents-md, codex-config-toml, claude-settings (repo_config). Весь контент — из лейнов.
`templates/iva-role-analyst/manifest.yaml` depends_on = `[tacticum-core-base, iva-analysis-base]`.

Вывод: новый лейн `tacticum-research-base` создаётся как base-манифест (NO depends_on), добавляется в `depends_on` нужных ролей. Новые типы задач в bugfix-лейне — это правка command_spec/skill_spec внутри tacticum-bugfix-base.

---

## 2. tacticum-bugfix-base — /fix-bug + scope-tripwire (готовая половина 2-го слоя)

**Манифест:** `templates/tacticum-bugfix-base/manifest.yaml` — base (NO depends_on), 2 ингредиента: skill `bug-fix` + command `fix-bug`, single-tier, stack-agnostic.
**Команда:** `templates/tacticum-bugfix-base/ingredients/commands/fix-bug.md`.
**Скилл:** `templates/tacticum-bugfix-base/ingredients/skills/bug-fix/SKILL.md`.

Процедура (редуцированный doc-light loop, БЕЗ BRD/MOCKUPS/ADR/full-PIN): Phase0 context (jira_get_issue, jira_get_issue_development_info, requirement_tests) → Phase1 feedback loop → Phase2 reproduce → **Phase3 hypothesise (единственный human checkpoint, 3–5 falsifiable гипотез)** → Phase4 instrument (`[BUGFIX-xxxx]` префикс) → Phase5 fix (Tester red → Coder smallest fix → Test Runner green) → Phase6 cleanup + `fix.md` (~10–15 строк). Маркер завершения `BUGFIX_COMPLETE`.

**Scope-tripwire / эскалация в /start-task — «готовая половина 2-го слоя».** Две реализации, одинаковые триггеры:

fix-bug.md:51-52:
> **Scope tripwire:** if the fix would *change* intended behaviour, needs an ADR, or touches a fragile zone / many modules — STOP, tell me, and offer `/start-task`. Do not silently turn a bug-fix into a feature.

SKILL.md «Scope tripwire — hand back to /start-task» (строки 142-152):
> Stop and **pause** the moment the fix reveals it is not a fix:
> - it would **change intended/public behaviour** (feature, not restoration),
> - it needs an **ADR-level** decision or a new pattern,
> - it touches a **fragile-zone guard** or spreads across **many modules**.
> On trip: do **not** silently continue. Tell the developer... offer to switch to `/start-task`.

Плюс rule-of-thumb (SKILL.md:33): **restore** behaviour → /fix-bug, **change** → /start-task. И раздел «When this lane — and when NOT» (SKILL.md:21-36) — фактически критерии классификации.

Вывод: 2-й слой (пересмотр режима + handoff) уже частично живёт здесь как текст. ТЗ#2 расширяет паттерн на development-core (/run-*) и делает его двусторонним (bugfix→start-task уже есть; нужен start-task→bugfix/research и mid-flight пересмотр).

---

## 3. iva-analysis-base — /start-task (куда встроить 1-й гейт классификации)

**Команда:** `templates/iva-analysis-base/ingredients/commands/start-task.md` — ОЧЕНЬ тонкая (22 строки): просто хэндофф в sub-agent `tacticum-workflow` с текстом «Start Phase 1... generate BRD, MOCKUPS, ADR, PIN, TESTS. Present all documents... before proceeding.» **Никакого режим-выбора/условности на входе НЕТ** — команда безусловно идёт в полный дизайн-цикл.

Реальная логика фаз — в агенте `templates/iva-analysis-base/ingredients/agents/tacticum-workflow.md` (claude) и `...agents-codex/tacticum-workflow.toml` (codex):
- **Phase 1: Analysis & Design** — шаг 0 «Grounding (GATE — verifiable)»: affected_systems → список контейнеров (пустой = gate FAIL), arch_map, constraints, kb_get_task_context. Затем генерация 4 артефактов (brd/adr/pin/tests) + pin-api-verification + Jira-декомпозиция (вариант A). «Gate summary before Phase 2».
- **Phase 2: Validation (GATE — requires explicit user approval)** — печать сводки, ожидание "approved/ок/go/утверждаю".
- **Phase 3: Handoff** — link Jira US + sub-tasks, хэндофф разработчикам development-<стек>.

Вывод: **1-й гейт классификации встраивается ЛИБО в start-task.md (перед хэндоффом), ЛИБО как новый Phase 0 в tacticum-workflow (перед grounding-gate Phase 1)**. Место точное: в start-task.md — между строкой args и блоком «Hand off to the tacticum-workflow sub-agent»; в агенте — перед «## Phase 1: Analysis & Design». Существующий grounding-GATE Phase 1 — НЕ классификация (он про полноту фактов), классификация «bug/feature/refactor/research?» — новый слой ПЕРЕД BRD.

---

## 4. tacticum-development-core — /run-* (куда встроить промпт 2-го слоя)

**Манифест:** `templates/tacticum-development-core/manifest.yaml` — base (NO depends_on, ADR-0059 Решение 2). Ингредиенты: команды `run-implementation, run-coder, run-tester, run-test-runner, setup-code-intelligence` + mcp `serena` + rule `local-code-intelligence`. Стек-агностичен (агенты coder/tester/test-runner — в стек-лейнах).

**`ingredients/commands/run-implementation.md`** — оркестратор ПОЛНОГО цикла реализации (фазы 3–5), работает в top-level сессии (НЕ спавнит tacticum-workflow). Mandatory ack-gate sequence:
| # | Phase | Sub-agent | ack marker |
|1|Implement per PIN|coder|`PHASE_3_CODER_COMPLETE`|
|2|Author tests|tester|`PHASE_3_TESTER_COMPLETE`|
|3|Build+run+fix (≤3 итер)|test-runner|`PHASE_4_RUNNER_COMPLETE`|
|4|report.md + summary|orchestrator|`PHASE_5_REPORT_COMPLETE`|

Failure handling: пропущенный маркер = FAILURE → report.md status INCOMPLETE, не фабриковать маркер. PARTIAL от test-runner → всё равно к Phase 4.

Вывод: **промпт 2-го слоя (пересмотр режима при блокере/после фазы) встраивается в run-implementation.md в раздел «Failure handling»** — там, где test-runner упирается (после итерационного лимита / при обнаружении, что задача не тянется как реализация → эскалация обратно в analysis/start-task или в bugfix). Точки: после каждого ack-маркера и в блоке эскалации. Атомарные команды run-coder/run-tester/run-test-runner — те же точки для одиночных фаз.

---

## 5. Есть ли lite / research / refactor лейн или команда? — НЕТ

Полнотекстовый grep по templates/ + docs/:
- `lite-task` / `lite_task` / «lite task» — **0 совпадений**.
- `start-research` / `research-base` / `tacticum-research` — **0 совпадений**.
- `refactoring-S` / `refactor-lane` / «feature-S» — **0 совпадений**.

Подтверждаю: `tacticum-research-base` в репо НЕ существует (создаётся с нуля). lite-лейна/команды НЕТ (по ТЗ должен стать расширением bugfix — сейчас bugfix знает только тип «defect»). Существующие типы задач разведены только текстом: defect (bugfix), feature/change/new-UI/ADR-worthy (start-task). Слова «режим» в репо — ложные хиты (`existing-target-mode`, `run-launch`, RU «режим» в QA-скиллах), к mode-механизму отношения не имеют.

---

## 6. _mirrors.yaml — зеркалирование на brownfield-профили

**Файл:** `templates/_mirrors.yaml` (US #714, ADR-0059 Решение 7). Механизм: пока старый профиль жив, задекларированные ингредиенты обязаны быть **БАЙТ-В-БАЙТ** одинаковы у владельца (лейн) и в зеркале (старый профиль). **Правки — только у владельца; зеркало обновляется тем же PR.** Сверка CI: `scripts/check_mirror_sync.py` (workflow `profile-version-discipline.yml`) + `tests/catalog/test_role_replacement_parity.py`.

6 пар (61 ингредиент):
| owner (лейн) | mirror (старый профиль) |
|iva-analysis-base|iva-fr-analyst (api-contracts-discovery, design-system-discovery, fr-authoring, mockup-authoring, start-feature, update-feature)|
|iva-kmp-development-base|iva-kmp-brownfield|
|iva-web-development-base|iva-web-brownfield|
|iva-mail-development-base|iva-brownfield-mail|
|iva-ios-development-base|iva-ios-brownfield|
|firebird-web-development-base|firebird-web-brownfield|

**ВАЖНО для ТЗ#2:** `iva-analysis-base` зеркалит в `iva-fr-analyst` (только 6 перечисленных ингредиентов; start-task в список НЕ входит → его правка зеркала не требует, но проверить, что зеркалируемые start-feature/update-feature/fr-authoring не задеты). `tacticum-bugfix-base` в _mirrors.yaml **НЕ фигурирует** (нет старого профиля-зеркала) — правка bugfix зеркал не требует. Депрекированные профили зеркал не имеют (frozen): iva-go-backend-brownfield уже deprecated 0.2.2 (ADR-0059 §8).

Вывод: правка start-task.md (analysis) — свериться, что не входит в зеркалируемый набор iva-fr-analyst (start-task там нет — ОК). Правка fix-bug/bug-fix (bugfix) — зеркал нет, свободно.

---

## 7. kmp-прототип lite-task-workflow (skill) — НЕТ в нашем репо

grep `lite-task-workflow` / `lite_task_workflow` по всему репо (не только templates) — **0 совпадений**. Подтверждаю: скилл `lite-task-workflow` в tacticum-dev ОТСУТСТВУЕТ (он в kmp-репо потребителя, как и ожидалось). У нас нет ни kmp-lite, ни его следов.

---

## 8. Токены / GPT-5.6 — где задаётся модель

Модель роли/агента задаётся в ДВУХ местах в зависимости от CLI:

**a) agent_spec.metadata.model** (в манифесте лейна) — для claude-code. Пример iva-analysis-base: system-analyst → `model: opus`, tacticum-workflow → `model: opus`. Схема ingredient.v1 требует `model` (minLength 1) + `tools[]`.

**b) codex .toml (codex_body_path)** — для codex тело агента само несёт модель. `templates/iva-analysis-base/ingredients/agents-codex/tacticum-workflow.toml`:
```
model = "gpt-5.5"
model_reasoning_effort = "high"
sandbox_mode = "workspace-write"
```
То есть **под GPT-5.6 «оптимизировать» = править `model`/`model_reasoning_effort` в codex-.toml агентов** (agents-codex/*.toml) — сейчас там `gpt-5.5`. Grep по templates: маркеров `gpt-5.6`/`5.6` НЕТ нигде — апгрейд 5.5→5.6 ещё не сделан.

**c) repo-config уровня роли:**
- `templates/iva-role-go/ingredients/repo-configs/codex/config.toml.template` — `.codex/config.toml` целевого репо: `[agents] max_threads=4, max_depth=1` + `[mcp_servers.*]`. **Модель здесь НЕ задаётся** (только threads/depth/MCP).
- `claude-settings` (repo_config, body inline): `{"defaultModel":"opus","tools":{"allowed":[...]}}` → `.claude/settings.json` — дефолтная модель для claude-code на уровне репо.

Вывод: точки правки под GPT-5.6 — (1) codex-.toml агентов в лейнах (`model = "gpt-5.6"`, `model_reasoning_effort`), (2) при необходимости `metadata.model` в манифестах для claude. `config.toml.template` роли модель не трогает.

---

## Открытые вопросы / риски для плана

1. **Где физически 1-й гейт: команда или агент?** start-task.md (тонкая, 22 строки) vs Phase 0 в tacticum-workflow.md/.toml. Правка агента = править claude .md И codex .toml синхронно (дублирование текста). Правка команды проще, но команда только хэндофит — гейт до спавна агента может не иметь доступа к KB-фактам для классификации. Решить на плане.
2. **2-й слой в /run-implementation работает в top-level сессии** (не sub-agent) — эскалация «пересмотр режима» должна уметь позвать /start-task или /fix-bug из середины цикла. Механизма межкомандного вызова в репо нет — это будет текстовая инструкция оркестратору, не автоматика.
3. **research-base лейн — с нуля**: нет образца-предшественника в каталоге (в отличие от bugfix/analysis, у которых были brownfield-профили). Аналог по структуре — tacticum-bugfix-base (base, single-tier, skill+command). Нужен owner-маркер, README с provenance, CHANGELOG.
4. **Расширение bugfix типами refactoring-S/feature-S** ломает текущий инвариант скилла bug-fix: «restore behaviour → fix-bug, change → start-task». feature-S = change intended behaviour, что сейчас ЯВНО отправляется в start-task (SKILL.md:26-33, tripwire). Нужно переопределить границу лейна — риск смысловой коллизии с scope-tripwire.
5. **_mirrors.yaml:** правки analysis — свериться с CI `check_mirror_sync.py`; хотя start-task не в зеркалируемом наборе iva-fr-analyst, любой сдвиг shared-ингредиента уронит `test_role_replacement_parity.py`. bugfix зеркал не имеет — свободно.
6. **eval:** в ТЗ упомянут eval, но специализированного lane-eval в репо не разведано детально. Есть E2E harness `apps/backend/dev/e2e/` (real TEI + Codex CLI, ADR-0036) — как площадка для eval режимов. Требует отдельной разведки, если lead попросит.
7. **GPT-5.6:** апгрейда 5.5→5.6 нет нигде; если ТЗ требует — это сквозная правка всех agents-codex/*.toml по всем лейнам (не только modes-затронутых), объём надо оценить отдельно.
