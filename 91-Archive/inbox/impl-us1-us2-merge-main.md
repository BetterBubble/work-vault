---
status: draft
role: implementer
topic: ТЗ#3 US#1+US#2 — merge свежего origin/main в ветку + резолюция конфликта
repo: /Users/bubblemac/tacticum/tacticum-dev-us1-owner
branch: feat/us1-fr-authoring
date: 2026-07-24
permalink: tacticum/00-board/impl-us1-us2-merge-main-1
archived-at: 2026-08-03 11:16
---

# Merge origin/main в feat/us1-fr-authoring + резолюция конфликта CHANGELOG

Свёл ушедший вперёд `origin/main` (PR #142 workflow-modes) в нашу ветку US#1+US#2.
`git merge origin/main` (НЕ rebase). НЕ push — push сделает лид.

## Merge-коммит
**SHA: `e53597d`** — «Merge remote-tracking branch 'origin/main' into
feat/us1-fr-authoring» (без AI-подписей, проверено grep).

`git log origin/main..HEAD --oneline`:
```
e53597d Merge remote-tracking branch 'origin/main' into feat/us1-fr-authoring
43aaf68 feat(api-contracts): контрактный формат 3.1 + серия CT-n (US#2-А, owner+mirror)
c028882 feat(fr-authoring): FR v2 двухзонная модель + валидатор границы (US#1-А, owner+mirror)
```
origin/main перед merge = `fb87459`. US#1 `c028882` и US#2 `43aaf68` не тронуты.

## Что менял #142 (осмотр до резолюции)
`origin/main` относительно merge-base (`abed800`) в `iva-analysis-base` тронул:
`CHANGELOG.md`, `ingredients/commands/start-task.md`, `manifest.yaml` (0.1.3→**0.1.4**).
- **Зеркалируемые скиллы `fr-authoring` / `api-contracts-discovery` в
  iva-analysis-base #142 НЕ трогал** (workflow-modes — про start-task-гейт, не про
  аналитика). Проверено `git diff` — 0 изменений в этих путях. Наши правки целы.
- **`iva-fr-analyst` #142 НЕ трогал вообще** (0 файлов). Зеркало и его версия
  (0.1.11) не задеты.

## Конфликт и как разрешил
Единственный конфликт — `templates/iva-analysis-base/CHANGELOG.md` (обе стороны
писали в раздел 0.1.4). `manifest.yaml` авто-смержился (обе стороны 0.1.3→0.1.4,
git свёл в 0.1.4 без конфликт-маркеров) — потребовал ручной доводки версии (ниже).

**CHANGELOG — сохранил ОБЕ записи, ни одной не потеряно, порядок хронологический
(новейшее сверху):**
- `## [0.1.5]` — наша US#1-2 (FR v2 двухзонная модель + контрактный формат 3.1 / CT-n);
- `## [0.1.4]` — запись #142 (гейт классификации режима в /start-task, §4a bump);
- `## [0.1.3]` … далее без изменений.

Заголовки разделов CHANGELOG после резолюции:
`0.1.5 → 0.1.4 → 0.1.3 → 0.1.2 → 0.1.1 → 0.1.0`. Маркеров конфликта не осталось
(grep `<<<<<<<`/`=======`/`>>>>>>>` → пусто).

## Примирение версии manifest iva-analysis-base — развилка (durably)
**#142 поднял iva-analysis-base 0.1.3 → 0.1.4** (проверено
`git show origin/main:…/manifest.yaml`). Наша ветка тоже метила 0.1.3 → 0.1.4 —
**прямая коллизия номера версии** (два разных набора изменений под одним 0.1.4).

**Решение:** наша версия стала **СЛЕДУЮЩЕЙ поверх #142 → `0.1.5`**. Так
version-discipline монотонна: 0.1.4 (#142, уже на main) < 0.1.5 (наши US#1-2).
Заголовок нашей CHANGELOG-записи выровнен под 0.1.5, manifest поднят до 0.1.5.
Запись #142 сохранена под своим родным 0.1.4. Ни один набор изменений не «украл»
чужой номер.

`iva-fr-analyst` — версия **0.1.11 без изменений** (#142 профиль не трогал, конфликта
нет; его версия/CHANGELOG независимы от iva-analysis-base — они и до merge были
разведены 0.1.11 vs 0.1.4).

## Проверки (после merge)
- `check_mirror_sync.py` → **OK — 62** зеркальных ингредиента в 6 парах синхронны.
- `diff -q` fr-authoring обоих профилей → **identical** (наши US#1-правки целы).
- `diff -q` api-contracts-discovery обоих профилей → **identical** (наши US#2-правки
  + полировки critic целы).
- `check_profile_version_discipline.py --diff-against origin/main` → **OK — 48
  profile(s) clean** (48, а не 46: main добавил новые профили tacticum-lite-base /
  tacticum-research-base; наш 0.1.5 монотонен).
- `pytest apps/backend/tests/catalog/` целевые (test_manifest_schemas +
  test_role_replacement_parity + test_iva_role_presets + test_role_install_smoke,
  `--noconftest`) → **288 passed, 1 warning** (env `uv run --with pyyaml --with
  pytest --with jsonschema`).
- merge-коммит: grep `claude|co-authored|generated with|anthropic|claude-session`
  → **чисто** (нет AI-подписей). Секретов в diff нет.

## git diff --stat origin/main..HEAD — ровно наши 8 файлов
Merge поглотил всё остальное (2-точечный diff чистый — только наша работа):
```
 templates/iva-analysis-base/CHANGELOG.md                        |  42 +
 templates/iva-analysis-base/.../api-contracts-discovery/SKILL.md| 131 +
 templates/iva-analysis-base/.../fr-authoring/SKILL.md           | 291
 templates/iva-analysis-base/manifest.yaml                       |   2
 templates/iva-fr-analyst/CHANGELOG.md                           |  46 +
 templates/iva-fr-analyst/.../api-contracts-discovery/SKILL.md   | 131 +
 templates/iva-fr-analyst/.../fr-authoring/SKILL.md              | 291
 templates/iva-fr-analyst/manifest.yaml                          |   2
 8 files changed, 742 insertions(+), 194 deletions(-)
```

## Итог
Merge чистый, обе CHANGELOG-записи сохранены, версия iva-analysis-base примирена
до 0.1.5 (монотонно поверх #142=0.1.4), зеркало identical, наши US#1+US#2 правки
целы, все проверки зелёные. НЕ push (лид). Развилка по версии (коллизия 0.1.4 →
наша стала 0.1.5) описана выше.

## Связано
[[impl-us2-api-contracts]] · [[spec-us1-fr-authoring-v2]] ·
[[spec-us2-api-contracts-format-31]]