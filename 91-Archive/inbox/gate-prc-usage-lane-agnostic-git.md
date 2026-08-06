---
title: gate-prc-usage-lane-agnostic-git
type: note
permalink: tacticum/00-board/gate-prc-usage-lane-agnostic-git-1
archived-at: 2026-08-03 11:16
---

# Гейт: usage lane-agnostic (Figma numeric-compare mode) — коммит 789dbde

**Контролёр:** controller-гейт (read-only). **Объект:** дерево `tacticum-dev-web-axis1`, ветка `feat/ds-web-axis1`, коммит `789dbde` поверх `290f448`.
**ИТОГ: PASS (публикация разрешена).** 8/8 пунктов PASS. Единственная оговорка — DB-инфра-падения тестов (docker/postgres не подняты), НЕ блокер, разобраны ниже.

## Вердикт по пунктам

1. **Дельта узкая — PASS.** `git show --stat 789dbde`: тронуто РОВНО 2 файла — `templates/iva-web-brownfield/.../angular-ds-component-usage/SKILL.md` и `templates/iva-web-development-base/.../angular-ds-component-usage/SKILL.md`, 18 ins / 14 del. Ровно 3 хунка в каждой копии: depends-on-таблица (стр.37, `ui-mockup-match`), Step 7 «Visual match» (стр.154-161), anti-patterns «No pixel matcher» (стр.173-178). Доктрина Step 1-6, резолв словаря, токены — не тронуты.

2. **Lane-agnostic, не over-claim — PASS.** Во всех 3 местах формулировка стала УСЛОВНОЙ:
   - depends-on: «Use its **Figma numeric-compare mode** … **when the attached `ui-mockup-match` provides it**; otherwise fall back to its **HTML mode / design review**».
   - Step 7: «use its Figma numeric-compare mode … **when the attached profile provides it**; otherwise fall back to its **HTML mode / design review**».
   - anti-patterns: «Figma numeric-compare mode **when the attached profile provides it (else its HTML mode / design review)**».
   Безусловных «is the acceptance path / shipped / available» больше нет. Верно и для brownfield, и для dev-base.

3. **Guardrail цел — PASS.** Во всех 3 местах сохранён запрет реализации матчера в этом навыке: «do **not** implement pixel matching in this skill», «do **not** build a numeric matcher in this skill», «this skill only references it, **never implements it** (step 7)». В «реализуй тут» не превратилось.

4. **Устарелость не вернулась — PASS.** `grep -niE "not.yet|not-yet-shipped|PR-B|gap G5|until it lands|shipped|future"` по обеим копиям → единственное совпадение стр.23 «instance is **not yet** in the dictionary» (легитимно). Маркеров устаревшей атрибуции / безусловной доступности (shipped/future/PR-B/gap G5) — нет.

5. **Байт-идентичность — PASS.** `cmp brownfield … development-base …` → IDENTICAL (тихий выход, различий нет).

6. **Скоуп / чистота — PASS.** Ветка `feat/ds-web-axis1` (НЕ main), рабочее дерево чистое. AI-подписей в телах коммитов `origin/main..HEAD` (`claude/co-authored/generated with/anthropic/session/эмодзи-робот`) — 0. Секретов/`.env`/ключей/`__pycache__`/`.serena`/`.DS_Store`/worktree в дельте — 0. `tacticum-ui-base` и `_mirrors` — вне дельты (не тронуты).
   - _Наблюдение (не блокер):_ `git diff --stat origin/main..HEAD` = 12 файлов (не «2 пакета + 1 quickstart»), т.к. диапазон включает 6 коммитов PR-C (eb70dcb…789dbde): помимо usage-пары это quickstart-doc, authoring×2, design-system-discovery, iva-core×2, manifests×2, CHANGELOG×2. Все — легитимные doc/manifest артефакты предыдущих коммитов PR-C, не относятся к 789dbde. Мусора/секретов среди них нет.

7. **Валидаторы — PASS (инфра-падения отдельно).**
   - `check_mirror_sync.py` → EXIT 0 («64 зеркальных ингредиентов в 6 парах синхронны»).
   - `check_profile_version_discipline.py` → EXIT 0 («48 profile(s) clean»); `--diff-against origin/main` → EXIT 0. Бамп версии не требуется (правка тела навыка) — подтверждено.
   - `pytest apps/backend/tests/catalog/ -q`: **549 passed, 2 failed, 120 errors**. Целевой `test_role_covers_replaced_profile` (файл `test_role_replacement_parity.py`, включая параметр `iva-role-web<-iva-web-brownfield`) прогнан изолированно → **10 passed**.
   - **Все 2 failed + 120 errors — DB-инфра, НЕ блокер:** failed = `test_admin_catalog_authoring::test_patch_profile_404` и `::test_create_draft_404_unknown_profile` — падают на `engine.raw_connection()` → `localhost:5432` недоступен. Errors = postgres-fixture в conftest не смог поднять docker (`docker run postgres:16-alpine` exit 125; `Errno 61 Connect call failed 127.0.0.1:5432`). Ровно та инфра-категория (postgres/docker down, `raw_connection()`), что оговорена в ТЗ как не-блокер. Контентных/ассертных провалов, связанных с правкой, нет.

8. **Достоверность claim — PASS.** Проверено по origin/main:
   - brownfield co-located `ui-mockup-match/SKILL.md` РЕАЛЬНО содержит секцию «Figma numeric-compare mode» (стр.8-9, 31, 164+: ΔE / px size-deltas / token-conformance, «Scenario 2, step 7 — gap G5») → условная ссылка правдива для brownfield-лейна.
   - `tacticum-ui-base/.../ui-mockup-match/SKILL.md` секции Figma numeric-compare mode НЕ содержит: стр.27 «**HTML mockups only** for this version. Figma URLs and PNG exports are out of…», стр.135 «No pixel SSIM / pixel-diff» → HTML-only, Figma-mode отсутствует. Условность формулировки оправдана.

## Итог для тимлида
Коммит 789dbde корректно снимает over-claim: формулировка стала lane-agnostic, guardrail сохранён, устарелость не вернулась, копии байт-идентичны, claim про Figma-mode достоверен (есть в brownfield, нет в ui-base). Валидаторы mirror/version — EXIT 0; role-coverage-тест — passed. Единственное — тесты, требующие postgres/docker, падают из-за не поднятой БД (инфра, вне скоупа правки). **Публикация ветки разрешена.** Ничего не правил.