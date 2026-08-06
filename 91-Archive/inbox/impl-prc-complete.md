---
title: impl-prc-complete
type: report
permalink: tacticum/00-board/impl-prc-complete-1
status: draft
role: implementer
for: lead-ds
worktree: ~/tacticum/tacticum-dev-web-axis1 (feat/ds-web-axis1)
commit: cc016e6
autonomy: false
tags:
- ds-web
- figma-ds
- pr-c
- axis1
- iva-core
- implement
- done
archived-at: 2026-08-03 11:16
---

# impl-prc-complete — PR-C доведён, coverage зелёный, всё закоммичено (не запушено)

TZ: figma-ds ТЗ#1, PR-C ось-1 — довести (coverage-врезка + коммит всего PR-C).

Блокер из `impl-prc-consolidation` закрыт по прецеденту `e1f8394` (angular-ds-component-*): `iva-core-design-system` врезан в лейн `iva-web-development-base`. Роль `iva-role-web` теперь покрывает замещаемый `iva-web-brownfield`. Весь PR-C собран одним коммитом. **НЕ пушил** (autonomy off).

## Врезка покрытия (iva-web-development-base)

1. **Копия body** — `templates/iva-web-development-base/ingredients/skills/iva-core-design-system/SKILL.md` скопирован из brownfield **байт-идентично** (md5 совпал: `027233a078ea04e17e309f02277c5232`).
2. **skill_spec в manifest** — добавлена запись `iva-core-design-system` сразу после `angular-ds-component-usage`, в **base-формате** по образцу тамошних angular-ds записей: `tier trial`, `supports: [claude-code, codex]` (БЕЗ copilot, блочный стиль), `install_scope user`, `target_path_template .claude/skills/...`, `codex_target_path .agents/skills/...`, `body_path`, `metadata.description_trigger` (тот же текст, что в brownfield). БЕЗ `copilot_target_path` — как у соседних записей base.
3. **Версия + CHANGELOG** — `iva-web-development-base` 0.1.1 → **0.1.2**; секция CHANGELOG `[0.1.2] — 2026-07-24` (Fixed, ссылка на прецедент 0.1.1/e1f8394).

⚠️ **Mirror НЕ трогал:** iva-core-design-system в паре `iva-web-development-base` / `iva-web-brownfield` (`_mirrors.yaml` строки 42-59) НЕТ — как и angular-ds-component-* там нет. DS-навыки не зеркалятся; mirror-sync зелёный без правки пары.

## Коммит всего PR-C — `cc016e6`

`feat(ds-web): iva-core skill + design-system-discovery ось-1 fix + web quickstart + role coverage (G7/G8)`

`git diff --stat cc016e6~1 cc016e6` (8 файлов, +394/-11):
```
 docs/user_manuals/iva-web-figma-mapping-quickstart.md            | 133 +++  (G8, new)
 templates/iva-web-brownfield/CHANGELOG.md                        |  37 +++
 .../iva-web-brownfield/.../design-system-discovery/SKILL.md      |  27 +-   (G7)
 .../iva-web-brownfield/.../iva-core-design-system/SKILL.md        |  83 +++  (new)
 templates/iva-web-brownfield/manifest.yaml                       |  18 +-   (0.5.0)
 templates/iva-web-development-base/CHANGELOG.md                  |  10 +++
 .../iva-web-development-base/.../iva-core-design-system/SKILL.md  |  83 +++  (new, coverage-копия)
 templates/iva-web-development-base/manifest.yaml                 |  14 +-   (0.1.2)
```
Часть iva-web-brownfield = работа консолидатора (iva-core skill + G7 discovery-fix + G8 quickstart + 0.5.0 + CHANGELOG). Часть iva-web-development-base = coverage-врезка (эта задача). Без AI-подписей. Рабочее дерево чистое.

## Верификация (venv `apps/backend/.venv/bin/python`)

**Целевой тест — PASSED:**
```
pytest test_role_replacement_parity.py::test_role_covers_replaced_profile[iva-role-web<-iva-web-brownfield]
1 passed
```
Весь набор `test_role_covers_replaced_profile` — 10 passed.

**Полный `pytest apps/backend/tests/catalog/ -q`:** `2 failed, 549 passed, 120 error`.
- **120 error** = все требуют Postgres/Docker (`Connect call failed 127.0.0.1:5432`, `docker run ... exit 125`) — инфра, ожидаемо (нет БД/докера).
- **2 failed** = `test_admin_catalog_authoring.py::{test_patch_profile_404, test_create_draft_404_unknown_profile}` — тоже DB-инфра (endpoint запрашивает БД → SQLAlchemy connection refused → 500 вместо 404). **Pre-existing:** проверено через `git stash -u` — те же 2 FAILED на baseline без моих правок. Контента не касаются (мои изменения — только manifest-YAML; эти тесты — admin-API против БД).
- **Все контент-тесты зелёные:** явный прогон `test_role_replacement_parity + test_iva_role_presets + test_manifest_schemas` = **213 passed**; `test_role_install_smoke` = passed.

**Валидаторы (пост-коммит):**
| Проверка | Итог |
|---|---|
| `./scripts/check_mirror_sync.py` | OK — 64 зеркальных ингредиентов в 6 парах синхронны |
| `./scripts/check_profile_version_discipline.py` (static) | OK — 48 profile(s) clean |
| `... --diff-against origin/main` | OK — 48 profile(s) clean |

## Границы соблюдены
- НЕ трогал `tacticum-ui-base` / 6 копий discovery / другие роли / PR-B-файлы (ui-mockup-match).
- НЕ трогал `_mirrors.yaml` (iva-core вне пары, как и angular-ds).
- НЕ пушил (autonomy off; push — после мержа PR-B, решение пользователя).

## Самопроверка
- coverage-тест `[iva-role-web<-iva-web-brownfield]` зелёный.
- iva-core-design-system в ОБОИХ пакетах (brownfield + development-base), body байт-идентичен.
- mirror-пара не задета, check_mirror_sync зелёный.
- версии == CHANGELOG обоих пакетов: brownfield 0.5.0 == `[0.5.0]`; development-base 0.1.2 == `[0.1.2]`.
- 2 FAILED — pre-existing DB-инфра (доказано stash-baseline), не регресс.