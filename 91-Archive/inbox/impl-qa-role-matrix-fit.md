---
title: impl-qa-role-matrix-fit
type: report
permalink: tacticum/00-board/impl-qa-role-matrix-fit
tags:
- lead-qa
- iva-role-qa
- ADR-0060
- e2e-golden
- pr-119
archived-at: 2026-07-31 04:54
---

# impl: iva-role-qa вписана в матрицу ролей (пак + e2e-golden)

status: draft
worktree: ~/tacticum/tacticum-dev-qa-kit (ветка feat/qa-kit-subagents) — НЕ смержено, НЕ запушено
доделка: 5 (lead-qa)

## Что сделано

Роль `iva-role-qa` переведена с pure-composition leaf (`ingredients: []`) на **пак уровня роли** (канон ADR-0060, зеркало PR #119). 3 коммита в ветке:

- `3010307` — пак роли (manifest + body-файлы + README + CHANGELOG)
- `1044e50` — реконсиляция test_iva_role_presets.py
- `dc7456f` — e2e_install golden

### 1. Пак роли
Манифест `iva-role-qa`: `ingredients: []` → 4 пак-ингредиента (маркер `tacticum:iva-role-qa`), зеркало `pr-119:templates/iva-role-go`:
- `claude-md-fragment` (instruction_pack → CLAUDE.md, append_section)
- `codex-agents-md` (instruction_pack → AGENTS.md, append_section)
- `codex-config-toml` (instruction_pack → .codex/config.toml, create_if_missing)
- `claude-settings` (repo_config → .claude/settings.json, deep_merge)

Body-файлы **авторены под QA** (не копия Go) в `templates/iva-role-qa/ingredients/repo-configs/{claude-code,codex}/`: идентичность QA-автоматизатора (execute-only автотесты, дефолт-процесс write/batch/run/fix + jira-issue-autotest + prepare-mr-branch, write-канал Jira под личным PAT, READ факт-базы helm-analyst). `config.toml.template` несёт MCP-ориентир QA (core tacticum-mcp активен; qa-mcp `iva-atlassian-write`/`helm-analyst` + serena — закомментированный ориентир, активные записи рендерятся из лейна).

**Вариант B сохранён:** пак ≠ mcp_server_spec. MCP-серверы остаются в `iva-qa-mcp`, скиллы/агенты — в `iva-qa-autotest-base`. `depends_on` не менялся. version 0.4.0 → 0.5.0.

### 2. e2e_install golden
`apps/backend/tests/e2e_install/golden/iva-role-qa/{claude-code.json,codex.json}` (22 файла claude-code / 11 codex). Docker недоступен → полный e2e-тест не прогнать; golden сгенерирован **оффлайн через РЕАЛЬНЫЙ render-путь** (`compose` → `with_builtins` → рендеры → `apply_actions` → `snapshot_tree`, те же функции что в e2e). Генератор **провалидирован**: воспроизводит 4 существующих golden (iva-web/ios/firebird/dev-base) **байт-в-байт** по обоим CLI. На qa-дереве зелёные e2e-оракулы: unique-ingredients, runnable-MCP, markers-once, no-stale-id, entrypoint materialized.

Примечание: codex golden тоньше (агенты + write/run/fix/batch/jira скиллы autotest-лейна помечены `supports: [claude-code]`) — это **верно** отражает рендер, не баг.

### 3. Реконсиляция теста
`test_role_is_pure_composition_leaf` → `test_role_carries_only_role_packs` (+ `ROLE_PACK_KINDS`), зеркало #119: пак, ЕСЛИ есть, обязан быть только pack-kind'ами с маркером `tacticum:<role>`. **Ослаблено vs #119** (там пак обязателен для всех): пустой пак допустим, т.к. прочие роли (go/analyst/architect/techwriter) в нашей ветке пока pure-composition. Правка минимальна, чтобы уменьшить merge-конфликт.

## Проверка (вывод показан)
- Валидатор manifest.v2 + ingredient.v1: `iva-role-qa` + все затронутые (qa-autotest, qa-mcp, core) — **VALID**.
- `test_iva_role_presets.py` + `test_manifest_schemas.py` (backend venv, uv) — **зелёные**.
- e2e golden — консистентен с рендером (генератор совпал с 4 эталонами байт-в-байт); полный e2e не прогнан (нет docker).

## ⚠️ Конфликт-риски с #119 (флаг)
- **test_iva_role_presets.py — конфликт гарантирован.** #119 полностью переписывает `ROLE_LANES` (выкидывает qa/analyst/architect/techwriter, добавляет dev-матрицу go/kmp/web/mail/ios/java/internal/platform/firebird) и переименовывает ту же функцию с `assert own` (пак обязателен). Наша правка держит qa в `ROLE_LANES` и снимает `assert own`. При merge #119 конфликт на этом файле неизбежен — qa придётся вернуть в ROLE_LANES/`_GENERIC_ROLES` #119 вручную. Это следствие того, что снапшот #119 qa-роль вовсе не содержит.
- **Golden — БЕЗ конфликта** (новая папка `golden/iva-role-qa/`, чисто аддитивно).
- **e2e-тест для golden НЕ добавлен намеренно.** Роль-e2e инфра (`seed_role_*` фикстуры в conftest.py, `test_install_flow_role_*` + `_GENERIC_ROLES` в test_install_flow.py) — это добавления #119, в нашей ветке их НЕТ. #119 тяжело правит оба файла и qa в них не включает → добавление qa-теста туда дало бы гарантированный конфликт. Решение: golden готов, а wiring — тривиальный шаг после merge #119 (добавить `"iva-role-qa"` в `_GENERIC_ROLES` с present/absent + при необходимости `seed_role_qa_fixture`). Разумный дефолт под «минимизируй merge-конфликт».
- **ADR-0060 отсутствует в docs/adr нашей ветки** (есть в основном дереве `~/tacticum/tacticum-dev`); ссылки — на канон, приедет с merge.

## Границы соблюдены
Не тронуты: субстанция лейнов (скиллы/субагенты/MCP), гейт-работа доделки 4, другие роли. Только пак qa + golden + минимальная правка общего теста.

## git diff --stat (HEAD~3..HEAD)
```
 test_iva_role_presets.py                 | 42 +++++---
 golden/iva-role-qa/claude-code.json      | 24 ++++
 golden/iva-role-qa/codex.json            | 13 ++
 iva-role-qa/CHANGELOG.md                 | 36 ++++++
 iva-role-qa/README.md                    | 38 +++++--
 .../claude-code/CLAUDE.md.fragment       | 57 ++++++++
 .../codex/AGENTS.md.fragment             | 49 ++++++
 .../codex/config.toml.template           | 33 +++++
 iva-role-qa/manifest.yaml                | 78 ++++++++++---
 9 files changed, 332 insertions(+), 38 deletions(-)
```
