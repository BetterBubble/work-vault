---
title: verify-qa-autotest-base
type: note
permalink: tacticum/00-board/verify-qa-autotest-base
status: draft
role: verifier
track: B
tags:
- verifier
- qa
archived-at: 2026-07-29 18:12
---

# verify-qa-autotest-base

Независимая проверка (read-only, код не правил) автотест-лейна `iva-qa-autotest-base` и пересобранной роли `iva-role-qa`. Worktree `/Users/bubblemac/tacticum/tacticum-dev-iva-write`, ветка `feat/iva-write-base`, коммит `5df061c` (подтверждён `git rev-parse` = `5df061c96e78...`).

Отчёт implementer: [[impl-qa-autotest-base]]. Эталон разведки: [[explore-qa-autotest-skills]].

## Вердикт: PASS (с одним задокументированным флагом низкого риска — codex)

Все 6 проверяемых пунктов зелёные на реальном прогоне. Единственное расхождение — заявленный флаг codex (п.6), оценка ниже.

## 1. Композиционные тесты — PASS (реальный прогон `uv run pytest`)
- `tests/catalog/test_iva_role_presets.py` → **35 passed / 0 failed in 0.40s**.
- `tests/catalog/test_manifest_schemas.py` → **38 passed / 0 failed in 0.04s**.
- Для `iva-role-qa` присутствуют и зелёные все 7 релевантных тестов (проверил по `--collect-only`):
  - `test_manifest_validates_against_v2_schema[iva-role-qa]` — schema-v2 ✓
  - `test_role_is_pure_composition_leaf[iva-role-qa]` — ingredients==[] ✓
  - `test_depends_on_is_the_declared_lanes_in_order[iva-role-qa-lanes3]` — depends_on ✓
  - `test_lanes_are_depth1_bases[iva-role-qa-lanes3]` — depth-1 ✓
  - `test_single_owner_lanes_are_pairwise_disjoint[iva-role-qa-lanes3]` — disjointness ✓
  - `test_golden_parity_union_equals_sum_of_lanes[iva-role-qa-lanes3]` — golden-parity ✓
  - `test_ide_targets_claude_and_codex_full[iva-role-qa]` — ide_targets ✓
- Seed/docker-тесты не гонял (Postgres в песочнице недоступен — не наш регресс, per инструкции).

## 2. Целостность лейна `iva-qa-autotest-base` — PASS
- manifest `kind: skill_spec` = **9**; `kind: mcp_server_spec` = **0**; `body_path` = 9. ✓
- 9 ingredient_id == **9 директорий** `ingredients/skills/<id>/` == **9 SKILL.md**, ни один не пустой (`find -empty` пусто). ✓
- Frontmatter каждого SKILL.md валиден: только `name` + `description` (allowed-tools снят, как и планировалось). ✓
- references: **35 файлов** (точное совпадение с эталоном) — write-autotest 7, batch-autotest 3, playwright-cli 7, fix-failed-test 15, prepare-mr-branch 3; остальные 4 — 0. ✓

## 3. Композиция роли `iva-role-qa` — PASS
- `depends_on` == `[tacticum-core-base, iva-qa-autotest-base, iva-write-base]` — analysis отсутствует. ✓
- `persona.role: qa` ✓, `ingredients: []` ✓, version 0.2.0.

## 4. Дизъюнктность вручную — PASS (независимо от теста)
- core-base (6): `context7, conventional-git, getting-started, kb-navigation, tacticum-context, tacticum-mcp`
- qa-autotest (9): `batch-autotest, fix-failed-test, jira-issue-autotest, playwright-cli, prepare-mr-branch, rebuild-autocore, retro, run-tests, write-autotest`
- write-base (1): `iva-write`
- `sort | uniq -d` по объединению трёх лейнов → **пусто** (0 коллизий). 6+9+1 = 16 уникальных, union==sum. Попарное непересечение подтверждено. ✓

## 5. Guardrail — PASS
- `git show 5df061c` : 51 файл. Grep запретных путей `iva-analysis-base|iva-role-analyst` → **NONE**. ✓
- Все изменённые файлы попадают строго в `templates/iva-qa-autotest-base/*` (новые) + `templates/iva-role-qa/{manifest,README,CHANGELOG}` + `apps/backend/tests/catalog/test_iva_role_presets.py`. Ничего вне этого. ✓
- Дифф теста — ровно **1 строка**: `iva-analysis-base` → `iva-qa-autotest-base` в ROLE_LANES. Больше в тесте ничего. ✓

## 6. Флаг codex — расхождение манифест↔реальность, риск НИЗКИЙ, приемлемо как задокументированный флаг
Факты:
- Роль `iva-role-qa` объявляет `ide_targets.codex: full` (manifest стр. 71).
- Лейн `iva-qa-autotest-base` объявляет `ide_targets.codex: best-effort` (manifest стр. 69), честно: половина скиллов (write/batch/fix/run/jira) завязана на Claude-специфику (Task-субагенты, Skill(), PostToolUse-хуки), их `supports: [claude-code]`.
- Тест `test_ide_targets_claude_and_codex_full` читает **только** поле роли (`_manifest(role_id)["ide_targets"]`) и ассертит `codex == "full"`. Он **НЕ вычисляет** "min over lanes", хотя его же docstring декларирует именно эту семантику ("Effective CLI support = min over lanes"). Поэтому тест не ловит несоответствие роль(full)↔лейн(best-effort) и проходит только за счёт буквального чтения поля роли.

Оценка: это **реальное расхождение** — роль завышает codex-поддержку относительно собственного скомпонованного лейна. По декларированной семантике "min over lanes" честное эффективное значение было бы `best-effort`, а не `full`. Риск низкий (роль установочна, а не рантайм-контракт; implementer явно вынес расхождение в manifest/README/CHANGELOG роли). Приемлемо на текущем шаге как осознанный флаг, НО оставлять молча нельзя.
- Follow-up (для лида): либо понизить `iva-role-qa.codex` → `best-effort` (честно, но сломает конвенцinput-тест в текущем виде), либо доработать тест, чтобы он реально считал min-over-lanes и был согласован с фактом. Сейчас тест не является гарантией согласованности codex-таргетов.

## 7. Прочее
- Инфра-зависимые seed/init-тесты не запускались осознанно (нет Postgres/Docker в песочнице) — не относится к правкам.
- Открытый флаг implementer про 3 отсутствующих субагента (codebase-analyst/dom-explorer/code-writer) не относится к acceptance файловой/композиционной целостности лейна; это функциональная неполнота 3 из 9 скиллов, честно отражена в manifest.post_install_notes/README. На PASS текущих проверок не влияет, но остаётся follow-up к QA-команде.

## Acceptance
Пункты 1–5 — PASS с реальными числами. Пункт 6 — приемлемое, задокументированное расхождение низкого риска (не блокер), с рекомендацией follow-up. Блокеров для сдачи лейна+роли по проверенным критериям нет.

## Связано
- [[impl-qa-autotest-base]]
- [[explore-qa-autotest-skills]]