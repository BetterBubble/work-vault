---
title: gate-axis2-kmp
type: note
permalink: tacticum/00-board/gate-axis2-kmp-1
tags:
- draft
- controller
- gate
- kmp
- axis2
archived-at: 2026-08-03 11:16
---

# Гейт-вердикт: lead-modes ось-2 (KMP multi-repo)

- Роль: controller (read-only гейт)
- Объект: ветка `feat/kmp-multirepo-axis2`, коммит `ac30efc` (единственный)
- Worktree: `/Users/bubblemac/tacticum-worktrees/kmp-axis2`
- База оценки: 3-точечный diff `origin/main...HEAD` (истинный scope). Ветка 16 позади origin/main — ОЖИДАЕМО (нарезана до мержа #142), НЕ дефект.

## ВЕРДИКТ: **GO** ✅

Все 6 пунктов батареи — PASS.

---

## 1. ГИТ-ЧИСТОТА — PASS
- `git status` → `working tree clean`, ничего не застейджено/не забыто.
- Ровно 1 коммит впереди: `ac30efc iva-kmp-brownfield: cross-repo source/reference support (axis-2)`.
- Сообщение по сути: описывает additive/conditional характер, перечисляет затронутые файлы (start-task $3, 3 CLI-тела, bump+CHANGELOG). Разошлось 1 вперёд / 16 назад — как и предупреждено.

## 2. SCOPE — PASS
`git diff --name-only origin/main...HEAD` → ровно 6 файлов, все под `templates/iva-kmp-brownfield/`:
- `manifest.yaml`
- `CHANGELOG.md`
- `ingredients/commands/start-task.md`
- `ingredients/agents/tacticum-workflow.md` (claude)
- `ingredients/agents-copilot/tacticum-workflow.md`
- `ingredients/agents-codex/tacticum-workflow.toml`

Подтверждено: `iva-web-brownfield` НЕ тронут (R1=kmp-only). Зеркалируемые coder/tester/test-runner, `_mirrors.yaml`, `iva-kmp-development-base` — НЕ в диффе. Лишнего ничего.

## 3. АДДИТИВНОСТЬ (R5, прод-safe) — PASS
Дифф-факты доказывают «аддитивно/условно»:
- **start-task.md**: `$1`/`$2` и их дефолты (`${1:-…}`, `${2:-Tasks/}`) сохранены дословно; добавлен только опциональный `$3` (`[source-repo]`) с явной формулировкой *«Omit for the normal single-tree brownfield task — behaviour is unchanged»* и рендер `${3:-none — single-tree task}`.
- **3 CLI-тела**: copilot и codex — **100% вставки, 0 удалений** (numstat 36/0 и 18/0). Claude-тело — только вставки (существующие шаги discovery 1-4 и §3.0-проверки 1-6 не переписаны; кросс-репо добавлено как отдельный абзац discovery и как пункт 7 гейта).
- Каждый добавленный блок несёт явный guard: *«skip entirely when no source repo is given — the normal single-tree flow is unchanged»*, *«single-tree flow in steps 1-4 above is unchanged»*, *«checks 1-6 are the whole gate then»*.
- Единственные 2 удаления во всём диффе — замена-на-месте строки `argument-hint` и строки `version`. Регрессии одно-древесного режима нет.

## 4. §4a ДИСЦИПЛИНА ВЕРСИЙ — PASS
- `manifest.yaml`: `version 0.4.5 → 0.4.6`.
- `CHANGELOG.md`: запись `## [0.4.6] — 2026-07-24 / ### Added` в ТОМ ЖЕ коммите.
- CI-скрипт: `uv run python scripts/check_profile_version_discipline.py --diff-against origin/main` →
  `OK — 46 profile(s) clean.` / `EXIT=0` — **0 violations**.

## 5. ТЕСТЫ (без docker) — PASS
`cd apps/backend && uv run pytest` (schemas + presets + parity):
- `tests/catalog/test_manifest_schemas.py` — зелёные (38 dots).
- `+ test_iva_role_presets.py + test_role_replacement_parity.py` — совместный прогон всё зелёное, 0 failed (211 тестов суммарно, 100%).

## 6. 0 СЕКРЕТОВ + 0 AI-ПОДПИСЕЙ — PASS
- grep по диффу на `.env`/ключи/`secret`/`password`/`token=`/PRIVATE KEY — совпадений нет.
- grep на `claude|co-authored|generated with|claude.ai/.com`: единственные хиты — имена flavor-директорий агентов **`claude / copilot / codex`** (в CHANGELOG и коммит-сообщении). Это НЕ AI-подписи, а перечисление трёх целевых CLI. Реальных футеров `Co-Authored-By` / `Generated with` / ссылок claude.ai нет ни в диффе, ни в коммит-сообщении.

---

### Итог для тимлида
Коммит `ac30efc` чист, в скоупе, строго аддитивен и прод-safe для живого пакета `iva-kmp-brownfield` (0.4.5→0.4.6). Версионная дисциплина соблюдена, тесты зелёные, секретов и AI-подписей нет. **GO** на доставку. Напоминание: 16-позади-origin/main сводит тимлид на доставке (не дефект гейта).