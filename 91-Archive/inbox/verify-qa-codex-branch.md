---
title: verify-qa-codex-branch
type: report
status: draft
tags:
- verification
- qa-codex
- iva-role-qa
permalink: tacticum/00-board/verify-qa-codex-branch-1
archived-at: 2026-07-31 17:27
---

# Независимая верификация ветки feat/qa-codex-rework

**Worktree:** `~/tacticum/tacticum-dev-qa-codex` · **HEAD:** `beee494` · **baseline origin/main:** `1061d10` (совпадает с заданием, #119 внутри).
**Прогон:** реальный, локально, `uv run --extra dev` (полный venv с conftest-зависимостями). Дата: 2026-07-24.
**Docker:** демон локально НЕДОСТУПЕН → Уровень 2 не запускался (см. ниже).

Ветка меняет ТОЛЬКО `templates/` — 21 файл, всё в `iva-qa-autotest-base` (codex-дивергентные тела + правки manifest). Код backend не тронут.

## Уровень 0 — матрица (проверено по факту, не по логам)

Все 4 точки подключения qa на месте:

| # | Точка | Факт |
|---|-------|------|
| 1 | `test_iva_role_presets.py` ROLE_LANES (стр.106) | `"iva-role-qa": ["tacticum-core-base","iva-qa-autotest-base","iva-qa-mcp"]` ✓ |
| 1 | ROLE_PERSONA (стр.126) | `"iva-role-qa": "qa"` ✓ |
| 2 | `test_role_install_smoke.py` ROLES (стр.43) | `"iva-role-qa"` ✓ |
| 3 | `test_role_replacement_parity.py` REPLACEMENTS | qa ОТСУТСТВУЕТ — корректно (у qa нет предшественника, как у iva-role-java) ✓ |
| 4 | `test_install_flow.py` _GENERIC_ROLES (стр.846) | present=[write-autotest, run-tests, codebase-analyst, dom-explorer, iva-atlassian-write, helm-analyst]; absent=[coder, run-implementation, fr-authoring, bug-fix] ✓ |

**Вердикт Уровня 0: да по всем 4 точкам.**

## Уровень 1 — статика (прогнан, зелёный)

| Файл | passed | failed | skipped |
|------|-------:|-------:|--------:|
| test_iva_role_presets.py | 91 | 0 | 0 |
| test_role_install_smoke.py | 77 | 0 | 0 |
| test_role_replacement_parity.py | 82 | 0 | 0 |
| test_manifest_schemas.py | 38 | 0 | 0 |
| **Итого** | **288** | **0** | **0** |

Скрипты:
- `scripts/check_profile_version_discipline.py --diff-against origin/main` → `OK — 46 profile(s) clean`, exit 0.
- `scripts/check_mirror_sync.py` → `OK — 62 зеркальных ингредиента в 6 парах синхронны`, exit 0.

### Состав qa (материализация — из smoke, зелёная)
- 3 субагента (`agent_spec`): **codebase-analyst, dom-explorer, code-writer** ← iva-qa-autotest-base — все присутствуют, body-файлы есть, целевые пути обоих CLI без коллизий (`test_all_body_files_exist`, `test_declared_target_paths_are_unique` — passed).
- Команды (`skill_spec`): write-autotest, playwright-cli, run-tests, fix-failed-test, batch-autotest, jira-issue-autotest, prepare-mr-branch, rebuild-autocore, retro, craft-stack.
- Входные точки: CLAUDE.md + AGENTS.md + .codex/config.toml — есть (entry-points оракул passed). Маркеры паков не коллизят.

### ⚠️ Нюанс smoke-оракула субагентов (честно)
Оракул `test_run_commands_have_their_agents` жёстко завязан на семейство
`{run-implementation, run-coder, run-tester, run-test-runner} → {coder, tester, test-runner}`.
Для iva-role-qa `present = orchestration ∩ composed` **пусто** (у qa команды write/fix/batch, а не run-*), поэтому тест делает **early-return** и НЕ проверяет пары qa-команд с её субагентами.
Итого: присутствие 3 субагентов qa проверено (через body/target-path smoke), но заявленная в задаче связка «write/fix/batch → 3 субагента» на Уровне 1 оракулом НЕ покрыта — она подтверждается только на Уровне 2 (_GENERIC_ROLES present-list: codebase-analyst, dom-explorer), а он Docker-gated. Это не падение и не регрессия, а предел покрытия оракула — флажок для приёмки.

**Вердикт Уровня 1: зелёный (288/0/0 + оба скрипта clean). Единственная оговорка — оракул не ассертит command→agent пары именно для qa.**

## Уровень 2 — e2e install (заблокирован Docker → В CI)

Docker-демон локально недоступен → оба параметра
(`test_install_flow_roles_generic[codex-iva-role-qa]` и `[claude-code-iva-role-qa]`) НЕ прогонялись. Уходит в CI.

**Статический прогноз дрейфа голденов (что CI покажет):**
- Ветка добавила codex-дивергентные тела в `iva-qa-autotest-base`: `agents-codex/{codebase-analyst,dom-explorer,code-writer}.toml` и `skills-codex/{write-autotest,fix-failed-test,batch-autotest,jira-issue-autotest}/SKILL.md` — через `codex_body_path` в manifest.
- Голдены `golden/iva-role-qa/{codex.json, claude-code.json}` **НЕ регенерированы** (не в diff vs origin/main).
- `seed_community._load_ingredient` читает `codex_body_path` для ЛЮБОГО kind (не ограничен agent_spec) → codex_body уедет в metadata и для skill_spec тоже.
- **Ожидаемо: `[codex-iva-role-qa]` в ASSERT-режиме УПАДЁТ голден-дрейфом** — added codex-native тела (3 агента `.codex/agents/*.toml` + codex-тела 4 скиллов). `[claude-code-iva-role-qa]` ожидается стабилен (codex_body_path — codex-only, claude-тела не менялись).
- **Действие:** перед ASSERT прогнать `E2E_INSTALL_REGEN_GOLDEN=1` только на `codex-iva-role-qa`, закоммитить обновлённый `golden/iva-role-qa/codex.json`. (Я голдены НЕ регенерил — задача read-only.)

**Открытый архитектурный флаг (закоммичен самими авторами в manifest, R7-FLAG):**
`codex_body_path` исторически применялся ТОЛЬКО к agent_spec; ветка применяет его к skill_spec (write/fix/batch/jira). Требует ратификации: благословить codex_body_path для skill_spec ИЛИ ввести отдельный механизм. Это решение, не тест-результат — на владельца/арх-ревью.

## ИТОГ

- **Локально достижимое (Уровень 0 + 1): ЗЕЛЁНОЕ.** 288/288 catalog-тестов, оба discipline/mirror скрипта clean, матрица подключена по всем 4 точкам.
- **Оговорка Уровня 1:** smoke-оракул НЕ проверяет command→agent пары для qa (early-return); присутствие субагентов проверено отдельными smoke-тестами.
- **Осталось на CI (Уровень 2, Docker):** e2e install обоих CLI. Ожидаемо codex-голден ДРЕЙФ (added codex-тела агентов+скиллов) → нужна регенерация `golden/iva-role-qa/codex.json`; claude-code ожидается стабилен.
- **На арх-ревью:** ратификация R7-FLAG (codex_body_path для skill_spec).
- Уровни 3-4 (живой агент / выкатка) — вне зоны этого прогона.