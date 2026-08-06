---
title: impl-qa-gates-soft
type: report
permalink: tacticum/00-board/impl-qa-gates-soft
status: draft
role: implementer
lane: lead-qa
worktree: ~/tacticum/tacticum-dev-qa-kit
branch: feat/qa-kit-subagents
commit: 12be10e
tags:
- lead-qa
- adr-0060
- qa-autotest
- gates
- implementer
archived-at: 2026-07-31 04:54
---

# impl-qa-gates-soft — SOFT-гейт шага 4 (ADR-0060 §6, доделка 4)

Реализовано изолированно в worktree `tacticum-dev-qa-kit` (ветка `feat/qa-kit-subagents`), НЕ смёржено/НЕ запушено. Тимлид подхватывает.

## Важное расхождение (флаг)
Бриф ссылался на **ADR-0059**, но по указанному слагу (`profile-interaction-model-mcp-scoping-pipeline-gates`) канон — это файл **`docs/adr/0060-...`** (в основном дереве `~/tacticum/tacticum-dev`). 0059 — про IVAREQ/iva-write (ADR-0058-родственный). Решения 5 (пайплайн+словарь статусов) и 6 (гейты) — именно в 0060. Работал по 0060. Если у тимлида другой источник — сверить.

## Что сделано (все 4 пункта брифа)
1. **Регресс от PIN Impact — обязательная часть гейта шага 4** (не опция). Impact-зона (из `pin-authoring`, вход от dev) требуется на входе; нет → гейт помечает пробел/`Impact-зона: n/a`, но **SOFT — не блокирует**.
2. **`verified: yes/no` формализован** — приёмка не по зелёным логам, а сверкой результата с реальным источником/UI («зелёные тесты ≠ верные данные»). Ось `verified` рядом с `tests: green/red`.
3. **Эмиссия в ЛОКАЛЬНЫЙ coverage-ledger** по словарю ADR-0060 §5: `tc: reviewed→covered` (только при green И verified:yes И покрытой Impact-зоне), `tests: green/red(+cov%)`, `verified: yes/no`. Запись в US IVAREQ НЕ делается — задокументирована как TODO.
4. **SOFT-характер явно задокументирован**: гейт advisory (помечает, не блокирует). HARD стоп-кран (`requirement: done` невозможен без `tc: covered` + `verified: yes`) + запись статусов в IVAREQ — будущая фаза (нужны `iva-write` + проект IVAREQ + ёмкость; их ещё нет).

## Изменённые файлы (`templates/iva-qa-autotest-base/`)
- **NEW** `ingredients/craft-stack/shared/pipeline-gate.md` — канон SOFT-гейта (single source of truth, паттерн как у `ledger-and-deviations.md`; в манифесте не регистрируется — bundled-ресурс craft-stack, как и другие shared/).
- `ingredients/craft-stack/shared/coverage-ledger.template.md` — машиночитаемый оверлей «Гейт §4» (колонки `tc`/`tests`/`verified`/Impact-зона) поверх stack-нейтрального `disposition` (ядро не переопределено).
- `ingredients/craft-stack/shared/ledger-and-deviations.md` — указатель на pipeline-gate.md.
- `ingredients/craft-stack/SKILL.md` — pipeline-gate.md в карте файлов.
- `ingredients/skills/batch-autotest/SKILL.md` — шаг §2.11 + Сквозные правила (гейт SOFT).
- `ingredients/skills/batch-autotest/references/phases.md` — §2.2 (Impact-зона как вход) + §2.5 (эмиссия гейта тем же закрытием TC).
- `ingredients/skills/run-tests/SKILL.md` — Phase 5 ветка A (гейт достоверности: green ≠ приёмка).
- `CHANGELOG.md` — запись «Gates (2026-07-23)».

`git diff --stat`: 7 файлов +64/-1, +1 новый файл.

## Проверка
- Тесты `test_iva_role_presets.py` + `test_manifest_schemas.py` (backend worktree, `uv run --extra dev pytest`): **73 passed**.
- Манифест `iva-qa-autotest-base` VALID: `manifest.yaml` не менялся (правки — в bundled markdown craft-stack/skills, не в зарегистрированных body_path); валидатор manifest.v2/ingredient.v1 зелёный.
- Состав/композиция ролей, MCP-лейны, субагенты — **не тронуты** (правки только в гейт-логике автотест-лейна).

## Развилки/решения (дефолты без блокировки)
- Гейт вынесен единым канон-файлом `shared/pipeline-gate.md` (DRY-паттерн репо: reference, не триггеримый skill), из скиллов — тонкие указатели. Не дублировал прозой.
- Оверлей §5-словаря сделан отдельной осью поверх `disposition`, а не переопределением ядра (template явно запрещает переопределять ядро).
- `run-tests` — только advisory-примечание достоверности (он низкоуровневый раннер, леджер не ведёт; эмиссию статусов держит `batch-autotest` на закрытии TC).