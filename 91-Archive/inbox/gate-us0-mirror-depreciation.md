---
title: gate-us0-mirror-depreciation
type: report
permalink: tacticum/00-board/gate-us0-mirror-depreciation-1
tags:
- gate
- controller
- fr-analyst
- mirror
- tz3
archived-at: 2026-07-31 17:27
---

# Гейт US#0 — разъединение mirror-пары (ТЗ#3, lead-fr)

status: draft
роль: controller (read-only, только вердикт)
worktree: `/Users/bubblemac/tacticum/tacticum-dev-fr-contracts`
ветка: `feat/fr-analyst-contracts` · HEAD: `483dc0e` (ровно 1 коммит поверх main) ✔

## ИТОГ: PASS — готово к передаче на малый PR президенту

Все 6 пунктов чеклиста пройдены. Зелёность перепроверена независимо (не по отчёту implementer, а своим прогоном). Owner-профиль `iva-analysis-base` не тронут ни одним файлом.

---

## Пункт 1 — Гит-чистота и скоуп: PASS
- `git log main..HEAD --oneline`: единственный коммит `483dc0e feat(fr-analyst): decouple 3 ingredients from mirror pair (ТЗ#3 US#0)`.
- `git diff --stat main..HEAD`: ровно 4 ожидаемых файла, +24/−5:
  - `templates/_mirrors.yaml` (−3)
  - `docs/adr/0059-single-axis-process-lanes-and-role-packs.md` (+3/−1)
  - `templates/iva-fr-analyst/CHANGELOG.md` (+20)
  - `templates/iva-fr-analyst/manifest.yaml` (+1/−1)
- `git status`: рабочее дерево чистое, нет незакоммиченного.
- Мусора нет: ни `.DS_Store`, ни `__pycache__`, ни `.serena`, ни worktree-артефактов, ни случайных файлов.

## Пункт 2 — Owner НЕ тронут (критично): PASS
- `git diff --name-only main..HEAD | grep -c iva-analysis-base` → **0**. Ни одного файла под `templates/iva-analysis-base/`. Заморожённая база Diaret не затронута.

## Пункт 3 — Содержание правок корректно: PASS
- **(а) `_mirrors.yaml`.** Из первой пары `iva-analysis-base ↔ iva-fr-analyst` удалены РОВНО `api-contracts-discovery`, `fr-authoring`, `update-feature`. Остались `design-system-discovery`, `mockup-authoring`, `start-feature`. Остальные 5 пар в диффе не тронуты (следующая пара `iva-kmp-development-base` идёт как контекст без изменений). На HEAD: 6 пар, 59 ингредиентов.
- **(б) ADR-0059 §7.** Счётчик обновлён `61 → 59` и **равен фактическому выводу** `check_mirror_sync.py` (59). Добавлен абзац про осознанное расхождение аудиторий (`iva-analysis-base` — честная постановка as-is, заморожен, прод Diaret; `iva-fr-analyst` — design-capable направление, релаксация §2 под предохранителями). Зафиксировано: owner не трогается, из зеркала уходит только сторона `iva-fr-analyst`; остальные 3 ингредиента пары остаются зеркалом.
- **(в) CHANGELOG + manifest.** Запись `[0.1.11] — 2026-07-24` с причиной (разъединение зеркала = enabling; контент трёх ингредиентов НЕ менялся, фактическое расхождение вводят US#1–#2). Явно указано frozen-owner исключение: отступление от шапки `_mirrors.yaml` (L9–11 «объясни в CHANGELOG владельца») осознанное — владелец заморожен, поэтому запись в CHANGELOG зеркала. `manifest.yaml version: "0.1.11"` согласован с CHANGELOG.

## Пункт 4 — Секреты/мусор: PASS
- В диффе только текст (yaml/markdown). Нет `.env`, ключей, PAT, токенов, бинарников.

## Пункт 5 — Независимая перепроверка зелёности: PASS
Окружение репо не настроено (нет venv/yaml/pytest) — поднял эфемерный venv (uv) с pyyaml+pytest, прогнал сам:
- `python scripts/check_mirror_sync.py` → **exit 0**: `OK — 59 зеркальных ингредиентов в 6 парах синхронны.`
- `pytest apps/backend/tests/catalog/test_role_replacement_parity.py` → **exit 0**, 79 тестов pass (корневой conftest тянет alembic/бэкенд-зависимости — прогнал с `--noconftest`; сам тест использует только pathlib+yaml).
- `scripts/check_profile_version_discipline.py --diff-against main` (как в profile-version-discipline.yml на PR) → **exit 0**: `OK — 46 profile(s) clean.` Статический прогон — тоже exit 0.

## Пункт 6 — Достоверность счётчика: PASS (с примечанием)
- Самосогласованность: ADR §7 = **59** == вывод `check_mirror_sync.py` (**59**) == `grep -c '^  - ' _mirrors.yaml` на HEAD (**59**). ✔
- Baseline: на main было **62** ингредиента (проверено `git show main:templates/_mirrors.yaml`), удалено 3 → 59, что арифметически верно.
- Примечание: старый текст ADR §7 говорил «61 ингредиент», хотя фактически на main было 62 — это **преждевременная off-by-one ошибка в доке** (не в данных). Новая правка приземляет корректное 59 и заодно молча чинит этот латентный −1. Конечное состояние верное и согласовано с тулингом.
  - Моё мнение: отдельная задача НЕ требуется — итоговое число фактически правильное и совпадает с CI-проверкой. Достаточно этой пометки в истории; блокером не является.

---

## Наблюдения (не блокеры)
- Стоит отметить в PR-описании, что 62→59 включает исправление прежней доко-ошибки (ADR раньше писал 61 при фактических 62), чтобы ревьюер не спотыкался о «−2 вместо −3».
- Доставка — отдельным малым PR, autonomy off, мерж — решение президента (в скоупе, не нарушено).

## relations
- refines [[impl-us0-mirror-depreciation]]