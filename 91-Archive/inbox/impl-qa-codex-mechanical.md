---
status: draft
role: implementer
topic: qa-codex-mechanical
date: 2026-07-24
worktree: ~/tacticum/tacticum-dev-qa-codex
branch: feat/qa-codex-rework
base: tacticum-dev main@20ff9b8
permalink: tacticum/00-board/impl-qa-codex-mechanical-1
archived-at: 2026-07-31 17:27
---

# Передел автотест-лейна QA под Codex — механический чанк 1 (реализация)

Worktree: `~/tacticum/tacticum-dev-qa-codex` · ветка `feat/qa-codex-rework` (от `main@20ff9b8`).
3 локальных коммита, НЕ запушено. 12 файлов, +83/−39. Все в `templates/iva-qa-autotest-base/`.

## Что переделано по ingredient

### 1. secrets-example + gitignore-secrets → кросс
- **manifest только:** `supports: [claude-code]` → `[claude-code, codex]` для обоих.
- `codex_target_path` НЕ добавлял: целятся в корень репо (`secrets.yaml.example`,
  `.gitignore`-append), путь не `.claude/`-скоупнутый — одинаков на обоих провайдерах.
- Контент файлов не менялся (уже провайдер-нейтральны, `.claude/` в них нет).

### 2. run-tests → кросс + 2 примитива заменены
- **manifest:** `supports` → `[claude-code, codex]` + `codex_target_path: ".agents/skills/{ingredient_id}/SKILL.md"`.
- **Phase 5, ветка B (было L162-166):** `Skill(skill="fix-failed-test", args="./allure-raw")`
  → провайдер-нейтральный вызов скилла `fix-failed-test` по имени: Claude Code — тул `Skill`
  (тот же вызов, но в явной Claude-ветке); **Codex** — загрузка `.agents/skills/fix-failed-test/SKILL.md`
  инлайн со входом `./allure-raw`.
- **Phase 0 (было L85):** `AskUserQuestion` (2 пункта) → один свободный текстовый вопрос
  «какие тесты и в каком браузере?» (у Codex структурного тула-вопроса нет).
- **Bash-механика (rm/pytest/cp/jq) — 1:1**, не тронута.
- Ссылка на craft-stack (`.claude/skills/craft-stack/shared/pipeline-gate.md`, L156)
  нейтрализована до `общий дом стека craft-stack → shared/pipeline-gate.md`.

### 3. craft-stack → дом `$CRAFT` вместо хардкода `.claude/`
- **manifest:** `supports` → `[claude-code, codex]` + `codex_target_path`.
- Введена переменная **`$CRAFT`** (определена в SKILL.md): Claude Code `.claude/skills/craft-stack/`,
  Codex `.agents/skills/craft-stack/`. `${CLAUDE_PLUGIN_ROOT}` из активных пояснений убран
  (осталось лишь одно **историческое** упоминание «раньше в kit было `${CLAUDE_PLUGIN_ROOT}/stacks/...`» —
  как и в самом kit).
- **13 ссылок-дома** `.claude/skills/craft-stack/` → `$CRAFT/` (SKILL.md, failure-taxonomy,
  allure-raw-parser, fix-playbooks, rules/tests, shared/{pipeline-gate,coverage-ledger,ledger-and-deviations}).
  4 шапочных строки-определения получили провайдер-маппинг.
- **3 sibling-ссылки** (`.claude/skills/fix-failed-test/...`, `.claude/skills/write-autotest/...`)
  → skill-относительные (`fix-failed-test/references/...`), провайдер-нейтральные.
- **`rules/invariants.md` НЕ тронут** — он уже байт-в-байт совпадает с kit `craft/rules/invariants.md`
  (нейтраль); оставшиеся там `.claude/rules/` (L6, пример проектных rules) и `.claude/`/`.tasks/`
  (L58, item 14 — список агент-маркеров, что НЕ должно попадать в продуктовый код) — намеренные,
  как в kit.

### 4. retro → нейтраль по образцу kit `keep:retro`
- `supports` уже был `[claude-code, codex]` + `codex_target_path` в manifest — не трогал.
- **Тело:** добавлена платформенная нота Phase 0 (дом инфры `.claude/{rules,skills,agents}` у
  Claude Code ↔ `.agents/{skills,agents}` у Codex; память `$HOME/.claude/projects/<slug>/memory`
  ↔ `.codex/memory.md`) — дословно по модели kit keep:retro. Интро (L9) провайдер-нейтрально
  (`CLAUDE.md`/`AGENTS.md`).
- **Командные литералы `.claude/`** в Phase 0 (`wc`, python-glob, `git diff`) сохранены — **kit
  keep:retro их тоже держит** (Phase 0.1), нота маппит на Codex. Это осознанная интерпретация
  «привести к kit keep:retro» = выровнять провайдер-нейтральность, а НЕ заменить тело целиком
  (полная kit-версия тянет worktrees/plugin-cache/feedback-channel/`$KEEP`/`$BASE`/marketplace —
  чужеродно для non-kit-deploy модели лейна, это семантический скоуп за пределами механики).

### manifest + версия
- `version` **0.1.0 → 0.2.0** (обязательно: `check_profile_version_discipline.py` требует bump
  при любом изменении контента лейна vs origin/main, иначе seed отвергнет `version_already_exists`).
- CHANGELOG: новая секция `## [0.2.0] — 2026-07-24`.
- Комментарий про codex best-effort уточнён (run-tests снят из списка Claude-специфичных).
- **`ide_targets` НЕ менял** — codex остаётся `best-effort` (write/batch/fix/jira + 3 субагента
  ещё Claude-only). Правильно для текущего состояния.

## Пути/примитивы — сводка замен
| было (Claude) | стало (кросс) |
|---|---|
| `Skill(skill="fix-failed-test", …)` | вызов скилла по имени (Claude: `Skill`; Codex: `.agents/skills/…` инлайн) |
| `AskUserQuestion` (run-tests) | свободный текстовый вопрос |
| `.claude/skills/craft-stack/` (дом, 13×) | `$CRAFT/` (per-провайдер) |
| `${CLAUDE_PLUGIN_ROOT}` (craft-stack, активн.) | убрано (осталось истор. упоминание, как в kit) |
| `$HOME/.claude/projects/<slug>/memory` (retro, жёстко) | + Codex `.codex/memory.md` (нота) |

## Что взято из kit
- Модель `$CRAFT` и провайдер-резолв дома — из kit `craft/stacks/*` (там `$CRAFT/...`).
- Платформенная нота памяти retro — дословно по kit `keep/skills/retro/SKILL.md` Phase 0.
- Подтверждение, что `invariants.md` и командные `.claude/`-литералы retro — kit-нейтраль (не менять).

## Статус тестов
Backend-venv в репо/дереве отсутствует; системный python3.14 сломан (pyexpat). Поднял отдельный
python3.12-venv в scratchpad (pytest/jsonschema/pyyaml/pytest-asyncio).
- `test_manifest_schemas.py` + `test_iva_role_presets.py`: **73 passed** (baseline до правок — тоже 73;
  запуск `--noconftest`, т.к. общий `apps/backend/tests/conftest.py` тянет alembic/sqlalchemy,
  которых нет в лёгком venv; целевые тесты — file-level, фикстуры conftest им не нужны).
- `check_profile_version_discipline.py` (static + `--diff-against main`): **OK, 29 profile(s) clean**.
- Манифест парсится (yaml.safe_load), 15 ingredients, version 0.2.0. Схема допускает
  `codex_target_path` как свободное доп-поле (ingredient.v1 не рестриктит additionalProperties).

## За скоупом этого чанка (не трогал)
- write-autotest / batch-autotest / fix-failed-test (Task→Codex-делегирование — ждёт R1/прототип).
- 3 субагента codebase-analyst/dom-explorer/code-writer (agent_spec, Codex-упаковка — ждёт R7-схему).
- jira-issue-autotest (кросс-вызовы `Skill()` + `AskUserQuestion` — отдельный чанк).
- rebuild-autocore авто-триггер (PostToolUse, R3).
- `manifest.post_install_notes` оставил как есть (описывает Claude-Code install; субагенты/write/batch/fix
  ещё Claude-only, так что заметки актуальны для текущего кросс-набора) — флажу как минорную
  косметику на будущий чанк.

## Коммиты (ветка feat/qa-codex-rework)
- `f6f1e8e` craft-stack: провайдер-нейтральный дом $CRAFT
- `2579aec` run-tests + retro: замена Claude-примитивов на провайдер-нейтраль
- `09b9cae` manifest: supports кросс + version 0.2.0 + CHANGELOG

НЕ запушено, НЕ мержено — по протоколу (решение президента).