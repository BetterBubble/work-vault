---
status: draft
role: implementer
task: ТЗ#1 figma-ds · PR-C финал · навык angular-ds-component-authoring
branch: feat/ds-web-axis1
worktree: /Users/bubblemac/tacticum/tacticum-dev-web-axis1
commit: 5622f48707cd8e9d231e6738d961072b62d029c0
permalink: tacticum/00-board/impl-prc-authoring-fixes
---

# impl · PR-C authoring fixes (battery: fidelity + critic)

Правки батареи внесены в навык `angular-ds-component-authoring`, обе байт-идентичные копии (brownfield + development-base). НЕ пушил.

## Правки по пунктам (до → после)

### 1. (критично) ui-mockup-match — убрана устаревшая атрибуция «future / not yet shipped (PR-B / gap G5)»
PR-B смержен, режим уже в main. Переформулирован как доступный.

**Acceptance-секция (было):**
> This numeric Figma comparison is the **future `ui-mockup-match` mode, not yet shipped (PR-B / gap G5)** — reference it as the intended acceptance path, do **not** implement pixel/ΔE matching here.

**Стало:**
> This numeric Figma comparison is the **`ui-mockup-match` Figma numeric-compare mode** (showcase ↔ Figma) — reference it as the acceptance path, do **not** implement pixel/ΔE matching here.

**Related skills (было):**
> `ui-mockup-match` — showcase↔Figma numeric compare via its future mode, not yet shipped (PR-B / gap G5).

**Стало:**
> `ui-mockup-match` — showcase↔Figma numeric compare via its Figma numeric-compare mode.

Теперь не противоречит секции Migration, где режим уже назван доступным.

### 2. (fidelity, ТЗ Сц.3) единица батча = экран/поверхность, не компонент
**Было:** `Do not mix the two design systems in one **component** — a batch migrates it whole.`
**Стало:** `Do not mix the two design systems in one **screen/surface** — a batch migrates the screen/surface whole.`

### 3. (provenance) источник transition table
**Было:** `tokens first (legacy values → new tokens by the transition table), then components`
**Стало:** `tokens first (legacy values → new tokens by the transition table — the legacy→new token map maintained by the RE / dictionary pipeline; use `design-token-usage` to resolve the new token paths), then components`

### 4. (nice-to-have) migration как ещё один повод вызвать навык — §When to call
Добавлена одна фраза, без изменения формулировки «two entry points» (Сц.3 переиспользует путь Сц.1):
> Migration (Scenario 3) also lands here — it reuses the Scenario 1 path when a legacy component has no new-DS analog.

Токен-якоря / installation_id / доктрина / другие секции — не тронуты. Правки строго по батарее.

## Верификация
- **cmp обеих копий:** IDENTICAL (перед правками и после `cp`-синхронизации).
- **grep самопроверка:** `not yet shipped` / `future …mode` / `PR-B` / `gap G5` — пусто в обеих копиях. `screen/surface` присутствует (batch=screen).
- **mirror-sync:** `OK — 64 зеркальных ингредиентов в 6 парах синхронны.`
- **version-discipline (static):** `OK — 48 profile(s) clean.`
- **version-discipline (--diff-against origin/main):** `OK — 48 profile(s) clean.` → bump/CHANGELOG не требуется (правки в теле навыка).
- **pytest apps/backend/tests/catalog/ -q:** `549 passed, 2 failed, 120 errors` — все fail/error = отказ соединения с Postgres (`5432` недоступен), т.е. DB-тесты; не-DB и coverage зелёные.

## Commit
`5622f48` — `fix(ds-web): authoring — ui-mockup-match shipped (not future) + Сц.3 batch=screen + transition-table source (battery)` (без AI-подписей). НЕ запушено.