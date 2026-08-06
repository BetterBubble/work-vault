---
title: gate-tz2-addon
type: note
permalink: tacticum/00-board/gate-tz2-addon-1
tags:
- draft
archived-at: 2026-08-03 11:16
---

# Gate: lead-modes добор полноты ТЗ#2 — вердикт

**Роль:** controller-гейт (read-only). **Worktree:** `/Users/bubblemac/tacticum-worktrees/modes-addon`, ветка `feat/workflow-modes-addon`, коммит `f66be51` от `origin/main @928fe37`.

## ВЕРДИКТ: GO

Все 6 пунктов батареи — PASS. Правки строго аддитивны, скоуп ровно по плану, CI и тесты зелёные, секретов/AI-подписей нет.

---

## Пункт 1 — ГИТ-ЧИСТОТА: PASS
- `git status` — `working tree clean`, ничего не застейджено/не отслеживается лишнего.
- Ровно 1 коммит впереди origin/main: `f66be51`. Не в main, ветка явная `feat/workflow-modes-addon`.
- Сообщение по сути: описывает 3 пробела (ADR-first вход, 2-й слой mode-review, lite→research), version bumps + CHANGELOG, «Все правки строго аддитивны». Без мусора.

## Пункт 2 — SCOPE (origin/main...HEAD): PASS
Ровно 11 файлов, все в 3 заявленных пакетах, ничего лишнего:
- `templates/iva-analysis-base/`: `manifest.yaml`, `CHANGELOG.md`, `ingredients/commands/start-task.md`, `ingredients/agents/tacticum-workflow.md`, `ingredients/agents-codex/tacticum-workflow.toml` (5).
- `templates/tacticum-lite-base/`: `manifest.yaml`, `CHANGELOG.md`, `ingredients/skills/lite-task-workflow/SKILL.md` (3).
- `templates/tacticum-bugfix-base/`: `manifest.yaml`, `CHANGELOG.md`, `ingredients/skills/bug-fix/SKILL.md` (3).
- ⚠️ НЕ тронуты: role-манифесты, ROLE_LANES, `test_iva_role_presets.py`, `_mirrors.yaml`, ось-2 (iva-kmp-brownfield) — подтверждено name-status (в диффе их нет).

## Пункт 3 — АДДИТИВНОСТЬ (прод-safe): PASS
- `tacticum-workflow.md` / `.toml` — чистые вставки (ADR-first врезка перед п.1 Phase 1 + «Гейт пересмотра режима» 2-й слой перед Gate summary). Существующие Phase 1 (заземление-GATE), Phase 2, grounding-gate, Gate summary НЕ удалены и не переписаны — только дополнены.
- 1-й гейт классификации в `start-task.md` сохранён; добавлены `$3` ADR-first и caveat «рефакторинг с изменением поведения → полный конвейер».
- **7 удалений — все не-содержательные замены на месте**, содержимое не терялось:
  - 3× version bump (0.1.5→0.1.6, 0.1.1→0.1.2, 0.1.2→0.1.3);
  - `description:` и `argument-hint:` в start-task — расширены (добавлен ADR-first / `[adr-draft-path]`);
  - refactor-строка start-task — та же + продолжение про смену поведения;
  - bugfix-строка «many modules → /start-task.» — та же, точка заменена на запятую + добавлен bullet про `/start-research` при невоспроизводимом дефекте.
- Вывод: только добавления, существующий живой конвейер iva-analysis-base не разрушен.

## Пункт 4 — §4a ДИСЦИПЛИНА ВЕРСИЙ: PASS
- Bump всех 3 в том же коммите: iva-analysis-base `0.1.5→0.1.6`, tacticum-lite-base `0.1.1→0.1.2`, tacticum-bugfix-base `0.1.2→0.1.3`.
- CHANGELOG в том же коммите, датированные записи 2026-07-24 под каждую новую версию (`## [0.1.6]`, `## [0.1.2]`, `## [0.1.3]` с секциями Added/Changed).
- CI: `cd apps/backend && uv run python ../../scripts/check_profile_version_discipline.py --diff-against origin/main` → `OK — 48 profile(s) clean.` (exit 0, **0 violations**).

## Пункт 5 — ТЕСТЫ: PASS
`cd apps/backend && uv run --extra dev python -m pytest tests/catalog/test_manifest_schemas.py tests/catalog/test_iva_role_presets.py tests/catalog/test_role_replacement_parity.py` → **211 passed in 3.72s**. Все зелёные (флаг `--extra dev` использован).

## Пункт 6 — СЕКРЕТЫ + AI-ПОДПИСИ: PASS
- Grep добавленных строк по password/secret/api_key/token/private-key/AKIA/ghp_/xox/.env → `NO SECRET MATCHES`.
- Grep диффа + тела коммита по claude/co-authored/generated with/anthropic/claude.ai → `NO AI SIGNATURES`.

## Observations
- [gate] Гит-чистота PASS — 1 коммит f66be51, working tree clean, ветка вне main #verification
- [gate] Scope PASS — ровно 11 файлов в 3 пакетах, role/тесты/ось-2 не тронуты #verification
- [gate] Аддитивность PASS — вставки в .md/.toml, 7 удалений = замены version/строк на месте #verification
- [gate] Версии PASS — bump 3 профилей + CHANGELOG, CI 0 violations #verification
- [gate] Тесты PASS — 211 passed catalog-suite #verification
- [gate] Секреты/AI-подписи PASS — 0/0 #verification
- [verdict] GO #decision

## Relations
- part_of [[Lead-modes ТЗ#2]]