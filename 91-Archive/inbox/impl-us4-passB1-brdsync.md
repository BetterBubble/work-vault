---
status: draft
type: impl-report
task: US#4 Проход B1 — синк brd-authoring в монолиты mail+rn
tz: ТЗ#3
branch: feat/us4-passB1-brd-sync
author: implementer
date: 2026-07-24
permalink: tacticum/00-board/impl-us4-pass-b1-brdsync-1
archived-at: 2026-08-03 11:16
---

# US#4 Проход B1 — синк brd-authoring в mail+rn (impl-отчёт)

## Итог
Канонические изменения Прохода A (US#4) из `tacticum-dev-base/brd-authoring`
синкнуты в 2 безопасных монолита — `iva-brownfield-mail` и `iva-rn-brownfield`.
Оба brd теперь читают FR v2 так же, как канон; v1 без маркера читается по-старому
(backward-safe). Один коммит, 6 файлов. НЕ push/PR/merge.

## Worktree и ветка
- Worktree: `/Users/bubblemac/tacticum-wt/us4-passB1-brd-sync`
- Ветка: `feat/us4-passB1-brd-sync` от свежего `origin/main` (5552118 — включает
  Проход A, PR#148 `feat/us4-conveyor-brd`)
- Коммит: `ca1fdb2` (ahead 1)

## Что синкнуто (человеческим языком)
В обе копии `brd-authoring/SKILL.md` (mail и rn) перенесена дельта Прохода A:
- Переписанное описание Purpose: BRD генерится из **FR** (постановка аналитика,
  выход `fr-authoring`, входной контракт dev-конвейера).
- Новый раздел **«Input: reading the FR (входной контракт)»** с детекцией версии
  по маркеру `fr_skeleton`:
  - **К-1** — в v2 `FT-n` из §1.4, `UC-n` из §1.5 Часть 1 (не из Приложения);
    в v1 — из П.A/П.B. ID наследуются verbatim, без перенумерации.
  - **К-2-brd** — в v2 регистрируются проектные серии `CT-n` (§1.6) / `DM-n`
    (§1.7) / `EV-n` (§1.8) для передачи вниз (pin-authoring, TESTS).
  - **К-5** — версия определяется по маркеру `fr_skeleton: 2`; маркер отсутствует
    → v1 (legacy layout, backward-compatible); malformed → трактуем как v1.
- Соответствующие пункты в **rules** и **anti-patterns** (не читать FT/UC из
  Приложения на v2; не перенумеровывать FT/UC/CT/DM/EV).

## Стек-специфика
**Не было.** Проверено диффом: до синка mail-brd и rn-brd отличались от канона
РОВНО на дельту Прохода A (диффы mail и rn byte-identical друг другу; единственная
строка со стороны монолитов — старое описание Purpose, всё остальное — только
добавления канона). Стек-специфичного контента в brd монолитов нет →
**чистый синк копированием канона**, ничего durably-сохранять не потребовалось.
Подтверждает разведку explorer (brd был byte-identical у mail/rn).

## Версии и CHANGELOG
- `iva-brownfield-mail/manifest.yaml`: 0.7.1 → **0.7.2** + запись в CHANGELOG.
- `iva-rn-brownfield/manifest.yaml`: 0.5.1 → **0.5.2** + запись в CHANGELOG.
- Обе CHANGELOG-записи (Changed): «brd-authoring синкнут с каноном — читает FR v2
  по маркеру fr_skeleton (§1.4/§1.5, серии CT/DM/EV §1.6-1.8); v1 backward-safe».

## Проверки (все зелёные)
- `diff -q` mail-brd vs канон tacticum-dev-base brd → **identical (mail==canon OK)**
- `diff -q` rn-brd vs канон → **identical (rn==canon OK)**
- `check_profile_version_discipline.py --diff-against origin/main` →
  **OK — 48 profile(s) clean**
- `check_mirror_sync.py` → **OK — 64 зеркальных ингредиента в 6 парах синхронны**
  (brd не в mirror — не затронут, как и ожидалось)
- pytest целевые (schemas + role_presets + install_smoke, `--noconftest`,
  `PYTHONPATH=apps/backend`) → **206 passed** (test_manifest_schemas.py +
  test_iva_role_presets.py + test_role_install_smoke.py)

## git diff --stat origin/main..HEAD (6 файлов, как и планировалось)
```
 templates/iva-brownfield-mail/CHANGELOG.md         | 15 +++++++
 .../iva-brownfield-mail/.../brd-authoring/SKILL.md | 47 +++++++++++++++++-
 templates/iva-brownfield-mail/manifest.yaml        |  2 +-
 templates/iva-rn-brownfield/CHANGELOG.md           | 15 +++++++
 .../iva-rn-brownfield/.../brd-authoring/SKILL.md   | 47 +++++++++++++++++-
 templates/iva-rn-brownfield/manifest.yaml          |  2 +-
 6 files changed, 124 insertions(+), 4 deletions(-)
```

## Развилки / примечания (durably)
1. **Локальный `main` отставал на 15 коммитов** от origin после fetch — worktree
   создан явно от `origin/main` (5552118), не от локального main. Проход A в базе
   присутствует.
2. **pytest без PYTHONPATH падал на сборке** (34 ошибки `ModuleNotFoundError:
   backend`) — тесты каталога импортят пакет `backend`. Решено `PYTHONPATH=apps/backend`.
   Это инфраструктурная деталь окружения, не связана с синком. Целевые
   schemas/role_presets/install_smoke на этом PYTHONPATH зелёные (206 passed).

## НЕ трогалось (по границам задачи)
iva-analysis-base, web/kmp-brownfield, композиты ios/firebird, go-backend/
task-workflow, pin/tests/start-task, сам канон tacticum-dev-base. Только
mail+rn brd SKILL + их manifest + CHANGELOG.