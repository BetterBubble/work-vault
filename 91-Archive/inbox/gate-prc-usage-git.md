---
title: gate-prc-usage-git
type: report
permalink: tacticum/00-board/gate-prc-usage-git-1
tags:
- controller
- gate
- pr-c
- ds-web
- usage-fix
archived-at: 2026-08-03 11:16
---

# Гейт controller — usage-фикс PR-C (G5-attribution), коммит 290f448

**Объект:** worktree `/Users/bubblemac/tacticum/tacticum-dev-web-axis1`, ветка `feat/ds-web-axis1`, HEAD `290f448` (usage-fix, append поверх PR-C). Дерево чистое, дельта origin/main..HEAD = 5 коммитов.

## ВЕРДИКТ: PASS

Все 5 пунктов пройдены. Падений/ошибок pytest — только DB-инфра (docker-демон недоступен), контент-тесты зелёные.

---

### 1. Usage-фикс корректен — ПРОШЛО
`git diff 290f448~1..290f448` тронул ровно 2 копии `angular-ds-component-usage/SKILL.md` (brownfield + development-base), в 3 местах:
- **depends-on (~стр.34):** «separate, not-yet-shipped mode — PR-B (gap G5)» → «its Figma numeric-compare mode — the acceptance path for Step 7».
- **Step 7 (~стр.155-161):** «not shipped yet (PR-B / gap G5)… Until it lands…» → «`ui-mockup-match` Figma numeric-compare mode — reference it as the acceptance path».
- **anti-patterns (~стр.172-176):** «Numeric Figma comparison is PR-B (gap G5)» → «is `ui-mockup-match`'s Figma numeric-compare mode».

Guardrail сохранён в обоих местах: «do not implement pixel matching in this skill» / «No pixel matcher here». Numeric Figma-mode назван доступным (acceptance-путь Сц.2 ш7).
grep по обеим usage-копиям на `not-yet-shipped|PR-B|gap G5|Until it lands` → **ПУСТО**.

### 2. Байт-идентичность — ПРОШЛО
- usage две копии: `cmp` → IDENTICAL
- authoring две копии: IDENTICAL
- iva-core-design-system две копии: IDENTICAL

### 3. Скоуп — ПРОШЛО
- usage-фикс `290f448` тронул ТОЛЬКО 2 usage-копии (14+/16-).
- Вся дельта origin/main..HEAD = 12 файлов в **2 пакетах** (iva-web-brownfield, iva-web-development-base) + **1 doc** (`docs/…/iva-web-figma-mapping-quickstart.md`). Ничего вне.
- `ui-mockup-match` НЕ тронут (unchanged vs origin/main), tacticum-ui-base/mirrors не в дельте.
- 0 секретов/.env/ключей, 0 AI-подписей (diff + commit msgs чисты), 0 мусора (`__pycache__`/`.DS_Store`/`.serena`).
- Ветка явная (не main).

### 4. Достоверность + acceptance — ПРОШЛО
- **Проверка подлинности «shipped»:** `git show origin/main:…/ui-mockup-match/SKILL.md` содержит реальную секцию «## Figma numeric-compare mode (Scenario 2, step 7 — gap G5)» (стр.164) + строки 8/31/47/62. Значит numeric-mode УЖЕ на origin/main (PR-B смержен ранее). Usage-фикс закрывает именно рассинхрон атрибуции — usage ошибочно называл существующий mode «не отгруженным». Claim достоверен, не self-cert.
- **pytest (полный catalog, прогнан контролёром):** `test_role_covers_replaced_profile[iva-role-web<-iva-web-brownfield]` — PASSED. Контент-тесты (role-parity, presets, manifest_schemas, ui-base, brownfield/dev-base profile) — зелёные (225 в целевом прогоне; 549 passed в полном).
- **Полный прогон:** `2 failed, 549 passed, 120 errors`. Все 120 errors = session-fixture `postgres_url` поднимает docker-контейнер `postgres:16-alpine` → `docker info` = DOWN (демон недоступен) → setup-error. Оба failed (`test_admin_catalog_authoring::test_patch_profile_404`, `test_create_draft_404_unknown_profile`) падают на `sqlalchemy engine.raw_connection()` — та же отсутствующая БД. **Падения = ТОЛЬКО DB-инфра, 0 контентных.**
- **mirror-sync:** OK — 64 зеркальных ингредиента в 6 парах синхронны.
- **version-discipline:** static → OK 48 profiles clean; `--diff-against origin/main` → OK 48 clean.

### 5. Версии — ПРОШЛО
- brownfield `0.5.0`, dev-base `0.1.2` (manifest.yaml) == заголовки CHANGELOG (обе датированы 2026-07-24).
- Usage-фикс — правка тела навыка, бамп не требуется; version-discipline зелёный в обоих режимах это подтверждает.

---

**Итог:** BLOCK-условий нет. Гейт PASS. Готово к финальному перегейту ГД. Пуш ветки был ранее (PR-C), usage-фикс — append без force после батареи проверок.