---
title: impl-tz2-addon
type: note
permalink: tacticum/00-board/impl-tz2-addon-1
tags:
- draft
archived-at: 2026-08-03 11:16
---

# impl-tz2-addon — ДОБОР полноты ТЗ#2 (lead-modes)

status: draft
worktree: /Users/bubblemac/tacticum-worktrees/modes-addon
ветка: feat/workflow-modes-addon (от origin/main)
коммит: **7674d53** (один, amend; локально, без пуша, без AI-подписей)
эталон: workflow-modes-proposal.md (§1.2/§3/§6) + research-task-flow-scenario.md

Закрыты 3 реальных пробела полноты (отчёт [[fidelity-tz2-completeness]]) + 2 опц-мелочи + 3 правки по вердикту fidelity (MAJOR + 2 MINOR). Всё СТРОГО аддитивно, по proposal Солонко.

## ПРОБЕЛ 1 — ADR-first вход как параметр /start-task
- `templates/iva-analysis-base/ingredients/commands/start-task.md`: frontmatter argument-hint `[adr-draft-path]`; Arguments — новый `$3` ADR-first (опц.); handoff-блок `**ADR-first вход:** ${3:-none}`.
- `.../agents/tacticum-workflow.md` + `.../agents-codex/tacticum-workflow.toml` Phase 1: врезка «ADR-first» синхронно в оба тела — работать ОТ поданного ADR, проверить полноту, сверить API/модули по KB+helm, расхождения до планирования, не досочинять. По scenario Шаг 2.
- Закрывает контракт research→build.

## ПРОБЕЛ 2 — 2-й слой гейта пересмотра режима (Фаза-1)
- `.md` + `.toml`, перед Gate summary: PIN→1 стадия без UI/классов → `/lite-task`; KB не отвечает/механизм не ясен → `/start-research`; независимые работы → split. «Предложить, не молча» + handoff.md. Синхронно оба тела. Существующий Gate summary не тронут.

## ПРОБЕЛ 3 — lite → research(мини)
- `.../tacticum-lite-base/.../skills/lite-task-workflow/SKILL.md`, «Escalation mid-flight»: абзац «Down/sideways to research» — корень/механизм не подтверждён (правки вне ордера / корень не подтвердился / verify падает 2-й раз) → stop + `/start-research`(мини) с handoff.md. Эскалация вверх сохранена.

## ОПЦ. мелочи
- (а) `.../tacticum-bugfix-base/.../skills/bug-fix/SKILL.md` «Scope tripwire»: дефект не воспроизводится / механизм неизвестен → `/start-research`.
- (б) `start-task.md` 1-й гейт, ветка РЕФАКТОРИНГ: «поведение МЕНЯЕТСЯ → полный».

## Правки по вердикту fidelity (round 2)
- **MAJOR** — `.../tacticum-lite-base/ingredients/commands/lite-task.md` блок «Escalate mid-flight»: research-ветка отражена и в КОМАНДНОМ теле (была только в SKILL) — «root/mechanism not confirmed → STOP, propose /start-research (mini) with same handoff.md». Формулировка согласована со SKILL.
- **MINOR-1 phantom report.md** — в mode-review-врезке (Фаза-1) `.md` + `.toml`: убрано «зафиксируй смену в report.md» (design-агент report.md НЕ производит) → причина смены фиксируется прямо в `handoff.md`. Синхронно оба тела.
- **MINOR-2 $3 × гейт** — `start-task.md` Шаг 1: «если задан $3 (готовый ADR — решение принято) → режим РИСЕРЧ исключается; классифицируй между ПОЛНЫМ и ЛАЙТ».
- CHANGELOG iva-analysis-base/lite-base уточнены под round-2 (report.md убран из текста, добавлены поправка гейта + синк командного тела). Повторного bump не было — версии 0.1.6/0.1.2/0.1.3 актуальны.

## Синк 2 CLI-тел
tacticum-workflow: оба тела (`agents/*.md` claude + `agents-codex/*.toml` codex) — врезки ADR-first + mode-review + правка report.md внесены синхронно.

## Version bumps + CHANGELOG (§4a)
- iva-analysis-base **0.1.6** · tacticum-lite-base **0.1.2** · tacticum-bugfix-base **0.1.3**. Дата 2026-07-24. Повторно НЕ бампались (round-2 — уточнения в тех же версиях).

## Проверки (после round-2)
- Дисциплина версий: `check_profile_version_discipline.py --diff-against origin/main` → **OK — 48 profile(s) clean** (0 нарушений).
- Тесты: `test_manifest_schemas + test_iva_role_presets + test_role_replacement_parity` → **211 passed**.
- Запуск тестов: `uv run --extra dev python -m pytest` — pytest в extra `dev`, не в базовом venv; прямой `uv run pytest` тянет системный python без pydantic.

## Аддитивность
Затронуто 7 файлов в 3 целевых пакетах. 1-й гейт классификации, Phase 1-2, grounding-gate, существующие эскалации/tripwire — на месте, новые блоки рядом. НЕ тронуты: role-манифесты/ROLE_LANES/тесты, _mirrors.yaml, зеркалируемые ингредиенты, другие лейны, ось-2, NIT-дупликация bug-fix (предсущ., не наш PR).

## Осталось / риски
- start-task/tacticum-workflow НЕ зеркалируются → правки только в iva-analysis-base.
- Не пушено (autonomy off) — ждёт ревью тимлида/пользователя.
- Тексты промптов — возможна финальная редактура owner'ом перед выкаткой.