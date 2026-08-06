---
status: draft
task: ТЗ#3 US#0 (enabling) — разъединить mirror-пару iva-analysis-base ↔ iva-fr-analyst
  для 3 ингредиентов
author: implementer (lead-fr)
date: 2026-07-24
permalink: tacticum/00-board/impl-us0-mirror-depreciation-1
archived-at: 2026-07-31 17:27
---

# US#0 — вывод 3 ингредиентов из mirror-зеркала (готово, всё зелёное)

## Где сделано
- **Worktree:** `/Users/bubblemac/tacticum/tacticum-dev-fr-contracts`
- **Ветка:** `feat/fr-analyst-contracts` (от `main` @ `20412ff`)
- **Коммит:** `483dc0e` — `feat(fr-analyst): decouple 3 ingredients from mirror pair (ТЗ#3 US#0)`
- **НЕ push, НЕ merge, НЕ PR** (autonomy off).

## Что именно изменено (человеческим языком)
Четыре файла, все — в общих/зеркальных файлах. **owner `iva-analysis-base` НЕ тронут ни одним файлом.**

1. **`templates/_mirrors.yaml`** — в первой паре (owner `iva-analysis-base` / mirror `iva-fr-analyst`) из списка `ingredients:` удалены ровно три строки: `api-contracts-discovery`, `fr-authoring`, `update-feature`. Остались: `design-system-discovery`, `mockup-authoring`, `start-feature`. Остальные 5 пар не тронуты.

2. **`docs/adr/0059-…-role-packs.md`, §7 «Переходный период — правило зеркал»**:
   - Счётчик исправлен. **Важно про число:** в ADR стояло «61 ингредиент», но фактический baseline по `check_mirror_sync.py` = **62** (текст ADR уже был устаревшим на 1). После удаления 3 → **59**. Поставил «6 пар, 59 ингредиентов» (по факту, не по формуле 61−3=58 из ТЗ). То есть в ADR теперь корректное актуальное число 59, а не 58.
   - Добавлен абзац: первое осознанное расхождение ингредиентов внутри пары; причина — расхождение аудиторий (база = обучение честной постановке as-is, §2 без ослаблений, прод Diaret, заморожена; fr-analyst = design-capable эволюция, релаксация §2 в Части 1 под предохранителями); owner не трогается; остальные 3 ингредиента пары остаются зеркалом.

3. **`templates/iva-fr-analyst/CHANGELOG.md`** — добавлена запись сверху `## [0.1.11] — 2026-07-24` (Changed): вывод 3 ингредиентов из зеркала, причина как выше, явно «skill-контент не менялся — enabling, расхождение вводят US#1-2», «owner НЕ тронут». Отмечено осознанное отступление от шапки `_mirrors.yaml` (L9-11 велит писать в CHANGELOG владельца, но владелец заморожен → пишем в CHANGELOG зеркала).

4. **`templates/iva-fr-analyst/manifest.yaml`** — `version: "0.1.10"` → `"0.1.11"` (patch bump).

### Про bump — почему подняли (проверил, не вслепую)
Энфорсер `check_profile_version_discipline.py` (check 1 «bump-needed») **явно исключает `CHANGELOG.md`** из «substantive», поэтому CHANGELOG-only правка bump строго НЕ требует. НО: конвенция репо — каждая запись CHANGELOG = версионный заголовок `## [X.Y.Z]`; если добавить новый заголовок без bump manifest → падает check 3 (manifest-lags-changelog: highest CHANGELOG > manifest.version). Чтобы запись была по конвенции и все три проверки остались зелёными, нужен согласованный patch-bump — ровно тот, что ТЗ предавторизовало (0.1.10→0.1.11). Bump безвреден по ADR-0009 (новая версия всегда принимается сидом; skill-контент идентичен 0.1.10).

## Результаты проверок (все зелёные)

**1. `scripts/check_mirror_sync.py`** → exit **0**:
```
OK — 59 зеркальных ингредиентов в 6 парах синхронны.
```
(baseline до правки был `62`; 62−3=59, сходится.)

**2. `scripts/check_profile_version_discipline.py`** (static и `--diff-against main`) → exit **0**:
```
OK — 46 profile(s) clean.
```
(это и есть проверка version-discipline из `.github/workflows/profile-version-discipline.yml`; workflow триггерится на `templates/**` — наши правки под ним, поэтому важно, что зелёный. Прогнал в diff-режиме против main — как на PR/push.)

**3. `pytest apps/backend/tests/catalog/test_role_replacement_parity.py`** → **79 passed** (0 failed):
   - **Блок A (mirror byte-check, `test_mirror_content_is_byte_identical`)** параметризуется из `_mirrors.yaml`: теперь **59** кейсов (было 62). Явно проверил: в коллекции для пары `iva-analysis-base->iva-fr-analyst` остался только `design-system-discovery`; `api-contracts-discovery`, `fr-authoring`, `update-feature` — исчезли. ✓
   - **Блок B (replacement-parity, `test_role_covers_replaced_profile` + `test_allowlist_entries_are_real_gaps`)** — зелёный, не затронут (новых ингредиентов не добавляли; пара `iva-role-analyst<-iva-fr-analyst` с пустым allowlist проходит). ✓
   - _Прим.:_ тест гонял через `uv run --with pytest --with pyyaml --with pytest-asyncio` с `--confcutdir=apps/backend/tests/catalog` — верхний `tests/conftest.py` тянет Postgres-в-Docker/alembic (нерелевантно этому файловому тесту), confcutdir его отсекает. Сам parity-тест читает только файлы.
   - _Прим. про окружение:_ системный `python`/`python3` без PyYAML — использовал `uv` для изоляции зависимостей. На PyYAML-энфорсерах результат тот же.

**`git diff --stat`:**
```
 docs/adr/0059-single-axis-process-lanes-and-role-packs.md |  4 +++-
 templates/_mirrors.yaml                                   |  3 ---
 templates/iva-fr-analyst/CHANGELOG.md                     | 20 ++++++++++++++++++++
 templates/iva-fr-analyst/manifest.yaml                    |  2 +-
 4 files changed, 24 insertions(+), 5 deletions(-)
```
**`git status`:** clean (всё закоммичено в `483dc0e`, рабочее дерево чистое).

## owner НЕ тронут
Ни один файл под `templates/iva-analysis-base/` не изменён — в diff только `_mirrors.yaml` (общий), ADR (общий) и два файла `iva-fr-analyst` (сторона зеркала). skill-контент `fr-authoring`/`api-contracts-discovery`/`update-feature` не трогал (это US#1-2).

## Отклонения от буквы ТЗ (обозначил, не расширял объём)
- **Число в ADR: 59, а не 58.** ТЗ предполагало 61−3=58, но фактический baseline = 62 (текст ADR был устаревшим на 1). Следовал инструкции «пересчитай фактически, не верь на слово» → поставил 59. Если тимлид хочет отдельно поправить/зафиксировать, что 62 — это правильный baseline (расхождение с прежним «61»), скажите — уточнять причину «62 vs 61» в рамках US#0 не стал, чтобы не расширять объём.

Ничего блокирующего. Готово к ревью тимлида.