---
status: draft
role: explorer
topic: qa-codex-rework
source_main: tacticum-dev @20ff9b8
date: 2026-07-24
permalink: tacticum/00-board/explore-qa-codex-rework-map-1
archived-at: 2026-07-31 17:27
---

# Карта передела автотест-лейна QA: Claude-специфика -> Codex

Разведка read-only. Источники: A = `tacticum-dev` main@20ff9b8 `templates/iva-qa-autotest-base/` · B = kit Жени (`kit-main.zip`, распакован в scratchpad) · C = Codex-first лейны репо (`iva-go-development-base`, `iva-role-qa`).

## TL;DR (главное)

1. **Claude-примитивы, которых нет на Codex 1:1**: `Task` (спавн именованного субагента в сессии), `Skill(skill=...)` (кросс-вызов скилла), `AskUserQuestion` (структурный вопрос), **эвристика «report file» Claude Code** (блокирует Write субагента в `analysis.md`/`*-report.md` — на ней построен контракт «субагент возвращает текстом, файл пишет оркестратор»), **PostToolUse-hook** (авто-триггер `rebuild-autocore`), `${CLAUDE_PLUGIN_ROOT}` (в нашем лейне уже захардкожен в `.claude/skills/craft-stack/`), пути `.claude/{skills,agents,rules}` и `~/.claude/plans/`, frontmatter `model:`/`tools:` формата Claude-агента.
2. **Ключевой актив kit для Codex — модуль `dispatch`**: провайдер-нейтральная обёртка `invoke_role.py`, собирает и запускает `codex exec <prompt> -C <root> -m <tier>` (или `claude -p ... --agent <bare> --model`). Готовый механизм спавна субагента на Codex. НО спроектирован под **встречную роль** (reviewer, чистая сессия, `depth=1`), не под внутрипайплайновую тройку write/fix. См. R1.
3. **Тела субагентов и скиллов craft в kit уже провайдер-нейтральны** (`$CRAFT`/`$DISPATCH`, стек-нейтральный канон). Но: (а) craft-скиллы всё ещё делегируют через `Task`; (б) `render_codex.py` конвертирует в Codex-native **только skills**, агентов (`agents/*.md`) НЕ рендерит — `.codex-plugin/plugin.json` несёт лишь `"skills": "./skills/"`. Тройку субагентов на Codex надо выражать заново.
4. **3 кросс-провайдерных «профиля» = профили ёмкости базы** (`base/templates/answers.template.toml [capacity].profile`): **0** Codex-only (все роли codex) · **1** Claude только встречный reviewer · **2** Claude-полный (fog_writer+scout->claude, pipeline_writer->codex, reviewer->counter). Деградация: базовый носитель каждой роли — Codex; недоступен Claude -> тихий откат на Codex + событие `degraded`. **База полноценна на Codex-only без Claude** — то, что нужно QA-команде.
5. Эталон формата Codex в репо (C): скиллы дублируются `codex_target_path: ".agents/skills/{id}/SKILL.md"`, конфиг `.codex/config.toml` c `[agents] max_threads=4, max_depth=1` (нативная многопоточность/субагенты Codex) + `.agents/skills/` как дом скиллов. Готового Codex **agent_spec** в репо пока НЕТ (go-лейн — только skill_spec).

---

## Таблица по ingredient (15)

Усилия: S <= полдня механики · M = 1–2 дня + проверка · L = проектирование + прогон.

| id | сейчас supports | Claude-примитивы, что мешает Codex | Codex-замена | готово в kit? | усилия |
|---|---|---|---|---|---|
| **write-autotest** | claude-code | `Task`x3 (Шаги 3/4/5 -> codebase-analyst/dom-explorer/code-writer); контракт «report file» (Шаг 3 персистит `analysis.md`); пути `.claude/skills/craft-stack/*` | делегирование -> `dispatch:invoke`/`invoke_role.py` (codex exec) ИЛИ native subagent; «report file» на Codex не нужен (субагент пишет сам); пути -> `$CRAFT`/`.agents/` | тело нейтрально (kit `craft/skills/write`), делегирование в kit всё ещё `Task` | M |
| **fix-failed-test** | claude-code | `Task`x3 (Phase 4); `Edit` у code-writer; «report file» персист; порог «оркестратор вносит сам (Edit)» | делегирование как write; `Edit`->Codex apply_patch; фазы нейтральны | тело нейтрально (kit `craft/skills/fix`), делегирование `Task` | M |
| **batch-autotest** | claude-code | `Task`x3 + веер «дефолт 3 агента: 1 general-purpose + 2 Explore»; `~/.claude/plans/`; сигналы компактации CC | делегирование -> dispatch; веер -> `[agents] max_threads` (config уже=4); планы -> `.tasks/`; компактация -> Codex-аналог/снять | тело нейтрально (kit `craft/skills/batch`), веер+планы Claude-специфичны | L |
| **jira-issue-autotest** | claude-code | `Skill(skill="fix-failed-test")`; делегирование в `/batch-autotest`, `/prepare-mr-branch` как скиллы; `AskUserQuestion`x2 | кросс-вызов -> Codex-скилл из `.agents/skills/` по имени/инлайн; вопрос -> свободный текст; `glab`/`allurectl` нейтральны | тело нейтрально (kit `craft/skills/issue`), кросс-вызов+вопросы переписать | M |
| **run-tests** | claude-code | `Skill(skill="fix-failed-test", args="./allure-raw")`; `AskUserQuestion` | кросс-вызов -> Codex-скилл/инлайн; вопрос -> текст; механика (rm/pytest/cp/jq) — чистый Bash 1:1 | нет прямого аналога (ближе `track:run`); механика тривиальна | S–M |
| **playwright-cli** | **[claude-code, codex]** | нет (уже кросс, есть `codex_target_path`) | уже есть | да (обёртка CLI-бинаря) | готово |
| **prepare-mr-branch** | **[claude-code, codex]** | нет (git cherry-pick, Bash); есть `codex_target_path` | уже есть; kit `ship:mr` | да | готово |
| **rebuild-autocore** | **[claude-code, codex]** (оговорка) | шаги — чистый Bash (портируются), НО авто-триггер = **PostToolUse-hook** (`detect-autocore-commit.py`+`.claude/settings.json`) CC-only; на Codex авто-триггера НЕТ | шаги -> Codex как есть; триггер -> ручной/git-hook. kit `vendor` несёт тот же hook, тоже CC-only (`${CLAUDE_PLUGIN_ROOT}`/`CLAUDE_PROJECT_DIR`/CC-схема) | тело — да (`vendor:rebuild`); авто-триггер — НЕТ на Codex | S (тело)/L (триггер, R3) |
| **retro** | **[claude-code, codex]** | housekeeping-текст, ссылки `.claude/`/`.tasks/` | пути -> нейтральные; kit `keep:retro` | да (`keep:retro`) | S |
| **craft-stack** | claude-code | не примитивы, а **пути**: дом захардкожен `.claude/skills/craft-stack/*` (в kit был `$CRAFT`/`stacks/{stack}/`) | путь-рерайт на `.agents/` + `codex_target_path`; контент нейтрален | да (kit `craft/stacks/*`+`rules/invariants.md`+`shared/*`) | S–M |
| **codebase-analyst** | claude-code (базово Codex gpt-5.4) | целится в `.claude/agents/`, спавн через `Task`; frontmatter `model: gpt-5.4`,`tools:[Read,Glob,Grep]` (формат Claude-агента); «report file» | Codex-агент: `.agents/`-раскладка ИЛИ роль `scout`(`scout-recon`); спавн dispatch/native; `render_codex` агентов НЕ покрывает -> вручную | тело — да (kit, нейтрально, `$CRAFT`); Codex-упаковка — НЕТ | M |
| **dom-explorer** | claude-code (базово Codex gpt-5.4) | то же; `tools:[Bash,Read,Write,Glob,Grep]`; **уже** имеет блок «Preflight носителя Codex (macOS): браузер в песочнице не стартует» | Codex-агент (`scout`); эскалация sandbox описана в теле | тело — да (kit); Codex-упаковка — НЕТ | M |
| **code-writer** | claude-code (базово Codex gpt-5.4) | то же; `tools:[Read,Edit,Write,Glob,Grep,Bash]`; `Edit`; `py_compile` через Bash (нейтр.) | Codex-агент (`pipeline_writer`/`fog_writer`); `Edit`->apply_patch | тело — да (kit); Codex-упаковка — НЕТ | M |
| **secrets-example** | claude-code | нет (файл-пример) | тривиально `[claude-code, codex]` | да | S |
| **gitignore-secrets** | claude-code | нет (append в `.gitignore`) | тривиально кросс | да | S |

Итог: **3 скилла уже кросс** (playwright-cli, prepare-mr-branch, retro) + rebuild-autocore (кросс, но триггер CC-only). **6 Claude-only с примитивами**: write, fix, batch, jira-issue, run-tests, craft-stack. **3 субагента** — claude-only-упаковка при Codex-first теле (gpt-5.4). **2 config** — тривиально кросс.

---

## (1) Забираем из kit ГОТОВЫМ

- **Модуль `dispatch`**: `invoke_role.py` (`codex exec`/`claude -p`, резолв роль->носитель из `answers.toml`, пресеты `presets.toml`, события `events.jsonl`, деградация CC->Codex, depth=1), фасад `dispatch:invoke`, `render_presets.py`, `references/role-rule.md`. Фундамент замены `Task` на Codex.
- **Тела 3 субагентов** `craft/agents/{code-writer,codebase-analyst,dom-explorer}.md` — нейтральны, `$CRAFT`-пути. dom-explorer с Codex-preflight браузера.
- **Тела craft-скиллов** write/fix/batch/issue/update — нейтральный канон, `$CRAFT`/`$DISPATCH`.
- **craft-stack контент**: `craft/stacks/{pytest-playwright-canvas,pytest-selenium}/*`, `craft/rules/invariants.md`, `craft/shared/*`.
- **Профили ёмкости**: `base/templates/answers.template.toml` (`[capacity].profile` 0/1/2, `[roles.*].carrier`, `[strictness].auto_counter_review`), `base/scripts/render_base.py` (валидация carrier), `base/scripts/task_meta.py` (stdlib, нейтрален).
- **Упаковка Codex**: `scripts/render_codex.py` + `.codex-plugin/plugin.json` + `.agents/plugins/marketplace.json` (генерат из CC-канона, но только skills).
- **Codex-аналоги инфра-скиллов**: `vendor:rebuild` (=rebuild-autocore тело), `ship:mr`/`ship:land` (=prepare-mr-branch), `keep:retro` (=retro), `track:run`/`track:parse` (=run-tests/разбор ланча).
- **Эталон Codex-конфига** репо: `iva-role-qa/.../codex/config.toml.template` (`[agents] max_threads=4 max_depth=1`, mcp_servers, `.agents/skills/`).

## (2) АДАПТИРУЕМ (тело есть, нужен переход примитива/путей)

- **Делегирование `Task`->Codex** во write/fix/batch: решить механизм (dispatch vs native `[agents]`) и переписать блоки вызова. Тела помощников готовы.
- **Кросс-вызовы `Skill()`** в run-tests и jira-issue -> вызов Codex-скилла по имени из `.agents/skills/` либо инлайн.
- **`AskUserQuestion`** (run-tests, jira-issue) -> свободный текстовый вопрос (у Codex нет структурного тула — проверить, R4).
- **Пути** `.claude/skills/craft-stack/*` -> `$CRAFT`/`.agents/`; добавить `codex_target_path` 6 Claude-only скиллам + craft-stack.
- **Frontmatter субагентов** `model:`/`tools:` -> Codex-форма (`.codex/config.toml [agents]`, роли из answers). Расширить `render_codex.py` на агентов.
- **Профили ролей**: смапить тройку (codebase-analyst~`scout`, dom-explorer~`scout`, code-writer~`pipeline_writer`/`fog_writer`) на роли answers + пресеты (`scout-recon`,`writer-full`).

## (3) ПИШЕМ ЗАНОВО

- **Codex-упаковка 3 субагентов**: `render_codex.py` рендерит только skills; агентов выразить заново (native Codex-агенты либо `dispatch`-роли). Нет прецедента в репо.
- **Механизм внутрипайплайнового спавна тройки на Codex** (не «встречная роль»): dispatch = counter/clean-session depth=1; оркестратор write->code-writer тот же провайдер, depth может конфликтовать. Спроектировать (R1).
- **run-tests как Codex-скилл**: ближайший `track:run`, но точная семантика (allure-raw backup, ветка A/B, вызов fix) — переписать.
- **Замена авто-триггера rebuild-autocore**: PostToolUse отсутствует -> явная дисциплина в оркестраторе или git-hook репо (R3).

## (4) Риски / открытые вопросы

- **R1 (критично).** Механизм субагентов Codex для тройки write/fix **принципиально иной**. `dispatch` = встречная роль другого провайдера/чистой сессии, depth=1, преимущественно reviewer. Codex-native субагенты (`config.toml [agents] max_threads/max_depth`) — отдельный механизм, в kit для craft-тройки не выражен. **Вопрос тимлиду:** тройку на Codex гоним через (а) native `[agents]`-треды, (б) `dispatch:invoke`/`codex exec`, (в) инлайн (оркестратор сам, без субагентов)? Определяет объём (M vs L).
- **R2.** Контракт «report file» — обход эвристики Claude Code. На Codex эвристики нет -> можно упростить (субагент пишет сам), но контракт заодно страхует от отсутствия write-прав; оставить ли ради портируемости CC<->Codex — решение.
- **R3.** Авто-триггер `rebuild-autocore` = PostToolUse-hook (CC-only). На Codex недоступен. kit `vendor` несёт тот же hook, тоже CC-only. На Codex — ручной вызов либо git-hook репо one-web (вне лейна).
- **R4.** `AskUserQuestion` — есть ли у Codex структурный аналог? Нет -> везде деградировать в текст (меняет UX 2 скиллов).
- **R5.** `render_codex.py` покрывает только skills, агентов — нет. Расширять генерат либо паковать вручную/через роли. Влияет на дрейф CC<->Codex.
- **R6.** Наш лейн уже разошёлся с kit: `$CRAFT`/`$DISPATCH` захардкожены в `.claude/skills/craft-stack/`, `Task` приведены к bare-именам, craft-stack встроен ingredient'ом. Передел = частичный откат к kit-нейтральности ИЛИ второй хардкод под `.agents/`. Решить: тянуть из kit vs форкнуть в репо.
- **R7.** `codex_target_path` в схеме манифеста есть, но только для skill_spec. Для agent_spec Codex-таргета в манифестах нет (субагентов go-лейн не имеет). Возможно нужен `codex_agent_path`/новый kind — сверить с ADR-0023 schema v2.

## (5) Черновой порядок работ

1. **Решить R1** (механизм субагентов Codex) — блокер. Прототип: codebase-analyst/`scout` через выбранный механизм, прогнать write-autotest на 1 TC.
2. **Тривиальное** (S): секреты/gitignore -> кросс; retro->`keep:retro`; craft-stack — путь-рерайт + `codex_target_path`.
3. **run-tests** (S–M): Bash-механика 1:1, заменить `Skill()`+`AskUserQuestion`.
4. **3 субагента** (M): упаковка под Codex по R1 + `render_codex`/ручная, мапинг на роли answers.
5. **write / fix** (M каждый): `Task`->Codex-делегирование, «report file» упростить/сохранить (R2).
6. **jira-issue** (M): кросс-вызовы скиллов + вопросы.
7. **batch** (L): веер параллельных (`max_threads`), планы `~/.claude/plans/`->`.tasks/`, компактация.
8. **rebuild-autocore триггер** (R3): ручная дисциплина/git-hook.
9. Прогнать профиль **0 (Codex-only)** end-to-end на one-web как приёмку.

---

## Ссылки на факты (файл:строка)

- Манифест: `templates/iva-qa-autotest-base/manifest.yaml` — supports (write L93, playwright L103+L106 codex_target_path, агенты L201/214/226 model gpt-5.4); ide_targets L68-73 (codex: best-effort); Task/report-file коммент L86-87.
- `Task`-спавн: write `skills/write-autotest/SKILL.md` L134,151,164; fix `skills/fix-failed-test/SKILL.md` L57-60,209-218; batch `skills/batch-autotest/SKILL.md` L36 (веер),55-59.
- «report file»: write L142-143; codebase-analyst `agents/codebase-analyst.md` L14-18.
- `Skill()`: run-tests `skills/run-tests/SKILL.md` L162-166; jira-issue `skills/jira-issue-autotest/SKILL.md` L82.
- `AskUserQuestion`: run-tests L85; jira-issue L27,81.
- PostToolUse-hook: rebuild-autocore `skills/rebuild-autocore/SKILL.md` L153; kit `vendor/hooks/hooks.json` L3-9, `vendor/hooks/detect-dep-commit.py` L20-24,58.
- Codex-preflight в теле: `agents/dom-explorer.md` L60-67.
- kit dispatch: `dispatch/scripts/invoke_role.py` L226 (claude -p), L238 (codex exec); `dispatch/skills/invoke/SKILL.md` (depth=1); `dispatch/references/role-rule.md` (роли/carrier/деградация).
- Профили 0/1/2: `base/templates/answers.template.toml` L22-23,43-61; `base/skills/install/SKILL.md` L51-65; `base/templates/schema.template.md` L38-46.
- render_codex только skills: `scripts/render_codex.py` L94-96; kit `CLAUDE.md` (канон CC, генерат Codex).
- Эталон Codex репо: `templates/iva-go-development-base/manifest.yaml` L60-63 (`.agents/skills/`); `templates/iva-role-qa/ingredients/repo-configs/codex/config.toml.template` (`[agents] max_threads=4 max_depth=1`).
- kit снапшот: `/private/tmp/claude-501/-Users-bubblemac-tacticum/47359696-4a55-470f-bf12-3b80ea797f36/scratchpad/kit/kit-main/`.