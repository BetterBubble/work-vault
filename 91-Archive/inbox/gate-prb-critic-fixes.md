---
title: gate-prb-critic-fixes
type: note
permalink: tacticum/00-board/gate-prb-critic-fixes-1
status: resolved
role: controller
topic: Финальный re-гейт PR-B после критик-правок (перед вехой ГД)
repo: /Users/bubblemac/tacticum/tacticum-dev-web-mockup
worktree: /Users/bubblemac/tacticum/tacticum-dev-web-mockup
branch: feat/ds-web-mockup-figma @ 09e523a
date: 2026-07-24
tags:
- board
- controller
- gate
- design-system
- tz1
- sc12
archived-at: 2026-08-03 11:16
---

# Гейт PR-B — критик-правки (re-проверка перед вехой)

**Вердикт: PASS — PR-B ready к вехе.** Read-only проверка коммита `09e523a` (критик-фиксы поверх `00c6edb`), merge-base `origin/main` = `5552118`. Все 5 пунктов пройдены, валидаторы прогнаны самостоятельно — зелёные.

## 1. Критик-правки корректны — ПРОШЛО (все 4)

- **#1 (обяз.) Colour ΔE → computed style.** §Metrics п.1 переписан: рантайм-цвет читается из **computed style** через `browser_evaluate` (`color` для text/icon, `background-color` для fill, `border-color` для border — тот же `getComputedStyle`-путь, что метрика размера §2 и capture §1), затем ΔE (CIEDE2000) против токен-значения. bbox-семпл скриншота **явно удалён** с обоснованием: «`get_metadata` bboxes = Figma-frame coords, not runtime layout; bbox-семпл text/icon/border-ноды вернёт фон, не искомый цвет». Метрика цвета больше не в неверных координатах. Усилена формулировка «not a blind pixel-diff» (computed-style-цвет ≠ пиксель).
- **#2 Tolerance без висячей «uiMatch».** Удалены обе отсылки в SKILL («per the uiMatch example» → просто «required:»; «These defaults follow the uiMatch reference» → «These are defaults, not hard limits»). В CHANGELOG строка про Tolerance тоже без «uiMatch», оперта на анти-алиасинг. ПРОШЛО.
- **#3 Детект темы.** В «Honest dependencies → ΔE needs the active theme mode» добавлено: определять light/dark по рантайму через `browser_evaluate` (`.dark` класс / `data-theme` на `<html>`/root / theme CSS-переменная), не предполагать light. ПРОШЛО.
- **#4 Опечатка veb→web.** CHANGELOG исправлен («This is the web (`iva-web-brownfield`) copy»). `grep -rn "veb"` по `iva-web-brownfield/` — пусто (exit 1). ПРОШЛО.

## 2. Защищённое НЕ тронуто — ПРОШЛО

Токен-якоря на месте и не менялись: `radius.control-m = 10`, `padding.content-area-sidebar = 16`, `gap.mail-chips = 6`, `padding.*`/`gap.*`/`fontSize.*`, color-токены (`bg.*`/`text.*`/`border.*`/`icon.*`). `installation_id` mandatory на каждом `design_*` вызове — выдержано. Разведение HTML/Figma («two modes do not mix») цело. Запрет слепого pixel не выброшен, а усилен. Источник биндинга — Figma `get_variable_defs` (не code-bindings). HTML-режим не тронут.

## 3. Гит / скоуп — ПРОШЛО

- `git diff --name-only <merge-base> HEAD` = ровно **3 файла**, все в `iva-web-brownfield`: `CHANGELOG.md`, `ingredients/skills/ui-mockup-match/SKILL.md`, `manifest.yaml`. Manifest = бамп версии 0.3.0→0.4.0 + расширение `description_trigger` (Two modes) — часть бандла PR-B, ок.
- Другие **4 копии** `ui-mockup-match` (mail / kmp / rn / tacticum-ui-base) — НЕ в диффе (0 совпадений). Owner/зеркала не тронуты.
- 0 секретов / `.env` / ключей / мусора. AI-подписей — **0** (grep -ic = 0).
- Ветка явная `feat/ds-web-mockup-figma`, ahead of origin/main на 2 коммита, working tree clean, **не main**. Не запушено.

## 4. Версия / валидаторы — ЗЕЛЁНЫЕ (прогнал сам, venv `apps/backend/.venv`)

- version-discipline (static) → `OK — 48 profile(s) clean.`
- version-discipline `--diff-against origin/main` → `OK — 48 profile(s) clean.`
- mirror-sync → `OK — 64 зеркальных ингредиентов в 6 парах синхронны.` (`ui-mockup-match` не в `templates/_mirrors.yaml` — grep exit 1 — CI-пара не сработает, копии независимы).
- `pytest -k manifest_schemas` → `38 passed, 1399 deselected`.
- Версия 0.4.0: критик-фиксы покрыты подсекцией `### Fixed` под `[0.4.0]` («critic fix-round (PR-B)»), все 3 содержательных фикса описаны. Бамп не требуется (правки внутри уже-добавленного на 0.4.0 навыка).

## 5. Сверх-ТЗ — НЕ обнаружено

Правки строго = 4 пункта критик-ревью. Метрики ΔE/size/token-conformance + допуск — ровно по шагу 7 (Сц.2, gap G5). Разрастания нет.

## Достоверность доказательств

Числа отчёта implementer'а (`impl-prb-critic-fixes`) перепроверены на реальном: дифф, grep защищённого, прогон всех 4 валидаторов — совпадают с заявленным. Не self-cert.

## Итог

**PASS — PR-B готов к вехе ГД.** Контекст: не пушено (push — после вехи + синхронизации + апрув Президента, окно ведёт ГД); PR-C (ось-1) вне бандла — не проверялся.

## Связано
[[impl-prb-critic-fixes]] · [[critic-prb]] · `00-Board/gate-prb-git` · `00-Board/gate-prb-fidelity`