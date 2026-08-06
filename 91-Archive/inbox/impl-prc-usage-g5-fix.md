---
status: draft
role: implementer
task: ТЗ#1 figma-ds — PR-C финальный фикс пробела (устаревшая атрибуция G5 в angular-ds-component-usage)
branch: feat/ds-web-axis1
worktree: /Users/bubblemac/tacticum/tacticum-dev-web-axis1
commit: 290f448
date: 2026-07-25
permalink: tacticum/00-board/impl-prc-usage-g5-fix-1-1
archived-at: 2026-08-03 11:16
---

# PR-C: usage — ui-mockup-match Figma mode shipped (не future)

## Суть
Навык `angular-ds-component-usage` (главный навык Сц.2) в 3 местах называл ui-mockup-match
Figma numeric-compare режим «not-yet-shipped (PR-B / gap G5)». Но G5 УЖЕ в main
(PR-B #154, iva-web-brownfield 0.4.0). Агент, читая §7, не вызвал бы доступный matcher →
приёмка Сц.2 ш7 не проходит. Убрал устаревшую атрибуцию, переформулировал matcher как
ДОСТУПНЫЙ — консистентно с уже-исправленным sibling `angular-ds-component-authoring`
(коммит ab4a693). Доктрина/токен-якоря/installation_id/прочие секции не тронуты —
только устаревшая G5-атрибуция в 3 местах.

Правки внесены **байт-идентично в ОБЕ копии**:
- `templates/iva-web-brownfield/ingredients/skills/angular-ds-component-usage/SKILL.md`
- `templates/iva-web-development-base/ingredients/skills/angular-ds-component-usage/SKILL.md` (coverage-копия, CI-fix e1f8394)

## 3 правки (до → после)

### 1. Таблица depends-on (стр.37)
**До:** `…matches against **HTML** mockups… The **numeric Figma comparison** …is a **separate, not-yet-shipped mode** — tracked as PR-B (gap G5). Reference it…`
**После:** `…It matches against **HTML** mockups… in its HTML mode. The **numeric Figma comparison** …is its **Figma numeric-compare mode** — the acceptance path for Step 7. Reference it…`

### 2. Step 7 — Acceptance (стр.155-161)
**До:** `…is the **Figma mode of ui-mockup-match**, which is **not shipped yet (PR-B / gap G5)** — reference it as the intended acceptance path… Until it lands, the current ui-mockup-match (HTML…) covers the HTML path only.`
**После:** `…is the **ui-mockup-match Figma numeric-compare mode** — reference it as the acceptance path… Its HTML mode (semantic + DOM) covers the HTML mockup path.`

### 3. Anti-patterns / guardrails (стр.176)
**До:** `**No pixel matcher here.** Numeric Figma comparison is PR-B (gap G5) — this skill only references it (step 7).`
**После:** `**No pixel matcher here.** Numeric Figma comparison is ui-mockup-match's Figma numeric-compare mode — this skill only references it (step 7).`

Во всех трёх сохранён guardrail «не реализуй pixel-matcher здесь, только ссылайся» — убрана
ТОЛЬКО ложная временная атрибуция.

## Верификация (полная)

- **cmp двух копий usage** → `IDENTICAL` (до и после коммита).
- **grep чистоты** (обе копии): `not yet shipped|not-yet|PR-B|gap G5|future` → COPY1 CLEAN, COPY2 CLEAN; остаточных `G5` → NO G5.
- **grep новой формулировки**: `Figma numeric-compare mode` присутствует в обеих (стр.37, 176 + Step7).
- **mirror-sync** (`scripts/check_mirror_sync.py`) → `OK — 64 зеркальных ингредиентов в 6 парах синхронны`.
- **version-discipline** (`scripts/check_profile_version_discipline.py`):
  - static → `OK — 48 profile(s) clean`
  - `--diff-against origin/main` → `OK — 48 profile(s) clean` (бамп НЕ требуется — правка в теле навыка, не в контракте).
- **coverage-тест (file-level, без БД)**: `pytest test_role_install_smoke.py test_iva_role_presets.py test_role_replacement_parity.py` → **252 passed** (это те самые тесты, что покрывают angular-ds-component навыки для iva-role-web, фикс e1f8394).
- **catalog-suite**: DB-тесты ошибаются (нет Postgres 5432 в worktree — окружение). 2 FAILED (`test_admin_catalog_authoring::test_patch_profile_404`, `test_create_draft_404_unknown_profile`) — **преэкзистинг**: проверено `git stash` → падают ИДЕНТИЧНО без моих правок (та же ошибка коннекта 5432), к правке SKILL.md отношения не имеют (HTTP admin endpoints).

## Commit
`290f448 fix(ds-web): usage — ui-mockup-match Figma mode shipped (not future), unblocks Сц.2 step 7 acceptance`
Без AI-подписей. НЕ запушен (лид пушит после батареи). Append на ветку feat/ds-web-axis1 (не мержена).

## Самопроверка
- [x] 3 места исправлены в ОБЕИХ копиях
- [x] копии байт-идентичны (cmp)
- [x] устарелость (not-yet/PR-B/G5/future) убрана — grep пусто в обеих
- [x] coverage-тест зелёный (252 passed)
- [x] mirror-sync + version-discipline зелёные
- [x] доктрина/якоря/прочие секции не тронуты