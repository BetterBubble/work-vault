---
title: gate-bundle1-critic-fixes
type: note
permalink: tacticum/00-board/gate-bundle1-critic-fixes
tags:
- ds
- web-to-kmp
- critic
- bundle1
- controller
- gate
- PASS
---

# Гейт controller — critic-фиксы бандла #1 `feat/ds-web-to-kmp` (перед PR-ready)

**ВЕРДИКТ: PASS (PR-ready) — с 2 наблюдениями (не блокеры).**
Кто/когда: controller для lead-ds, 2026-07-24. Read-only. Объект: worktree `tacticum-dev-ds-web-to-kmp`, ветка `feat/ds-web-to-kmp`, коммит `39ae642` (поверх `65e4fbf`). НЕ пушено/не мержено.

## 1. Обязательные critic-правки — внесены корректно ✓

**§7 «static gate + three-way parity» — разнобой «три vs четыре» СНЯТ ✓**
- Заголовок: `## 7. Verification — static gate + three-way parity (acceptance criteria)`.
- Структура: ОДИН **Static gate — component level** (Iva*, 0 raw Color/dp/material3, hoisting, keyed; под-буллеты LazyColumn-vs-Column и «read the widgets») — ОТДЕЛЬНО, не лег.
- **Three-way parity — three distinct legs** ровно из 3 нумерованных: (1) web-sample `ui-mockup-match`, (2) token `design-token-usage`, (3) Compose-render `kmp-ui-testing`/Roborazzi. «three distinct legs» указывает ровно на 3. Итоговая строка «full three-way parity» охватывает те же 3.

**Behavioural/logic parity — отдельный явный блок ✓**
- Болд-блок «Behavioural / logic parity — the main acceptance of the port»: ручная code-сверка source↔target (iva-one ↔ ported Decompose component/state holder), view-state enum + list/select/detail pipeline + action set. Явно «**NOT** an instrumented `ui-mockup-match`» (тот лочит веб-сторону, лег 1). Читается самостоятельно как главный acceptance.

**TODO-якорь словаря синхронизирован ✓**
- §TODO: `[Iva*↔web-component dictionary — RESOLVED on board, awaiting repo-native delivery]`, помечен RESOLVED на `00-Board/phase2-provisional-iva-web-dictionary`, **32 figma_key + 17 justified null**, reverse-keyed (Iva*→web, порт делает обратный lookup).
- §1.6 (шаг 6) и §5 — добавлены crosswalk-ссылки на словарь с пометкой reverse-keyed.

**Кросс-ссылки §7.x — целостны, битых нет ✓**
- Числовых `§7.1–§7.4` в файле НЕ осталось (`grep §7.[0-9]` = 0). Переведены на именные якоря: `§7 static gate` / `§7 web-sample leg` / `§7 token leg` / `§7 Compose-render leg` (в §8-таблице ×4 строки) + `§7 static gate + behavioural + render legs` (в §TODO). Все 5 якорей резолвятся в реальные заголовки/леги §7. Битых ссылок нет.

## 2. Гит / скоуп ✓
- Diff `39ae642~1..39ae642` = ровно 3 файла: `iva-kmp-development-base/{CHANGELOG.md, ingredients/skills/web-to-kmp-screen-port/SKILL.md, manifest.yaml}`.
- Второй навык `web-to-kmp-source-reference`, шаренное, роль, зеркала — этим коммитом НЕ тронуты. mirror-sync подтверждает (навык не в зеркалах).
- 0 секретов / 0 AI-подписей (скан diff по co-authored/claude/generated/ключам — clean).

## 3. Сверх-ТЗ (принцип президента) — нет ✓
- §0 rewrite-port vs move-port — цела (правка = только добавлена строка «record the scope estimate … before committing», critic nice-to-have #7, не новое ограничение).
- §8-таблица: строки НЕ добавлялись/удалялись — изменены ТОЛЬКО метки якорей §7.x→именные (4 строки). Доктрина цела.
- Нет новых фич/запретов сверх critic-замечаний.

## 4. Версия / валидаторы ✓
- manifest `0.6.0 → 0.7.0` (diff подтверждён) == CHANGELOG `[0.7.0] — 2026-07-24`. ✓
- `check_profile_version_discipline.py` → **OK — 46 profile(s) clean**.
- `--diff-against HEAD` → **OK — 46 profile(s) clean**.
- `check_mirror_sync.py` → **OK — 62 зеркальных ингредиента в 6 парах синхронны**.
- `pytest apps/backend/tests/catalog/test_manifest_schemas.py` → **38 passed** (implementer указал неверный путь `tests/...` без `apps/backend/` — прогнал из корректного места, зелёно).

## 5. Достоверность чисел (аудит подлинности, не self-cert) ✓
- Числа словаря в SKILL.md (**32 figma_key + 17 justified null = 49**) сверены с resolved-доской `phase2-provisional-iva-web-dictionary` (источник `design-systems/iva-web/tokens.json`, code-bindings). Совпадают, не выдуманы. reverse-keyed консистентен.

## 6. Готовность к PR ✓
- Бандл от merge-base (`git merge-base origin/main HEAD` = `20412ff`): ровно **4 файла** — `web-to-kmp-screen-port/SKILL.md`, `web-to-kmp-source-reference/SKILL.md`, `manifest.yaml`, `CHANGELOG.md`. ✓
- `git status --porcelain` = пусто (working tree clean). Ветка `feat/ds-web-to-kmp`, не main.

## Наблюдения (не блокеры)
1. **Вход `00-Board/critic-bundle1` на доске НЕ найден** (прямой permalink + семантика). Критик-ревью лид не выложил или под другим именем. Фиксы проверены против критериев ТЗ гейта + отчёта `impl-ds-critic-fixes` + фактического файла — все независимо верифицируемы, вердикт не зависит от отсутствующей заметки. Лиду: приложить critic-ревью к доске для полноты аудит-цепочки.
2. **§8 = 12 строк, ТЗ гейта ожидало «11 ссылок».** 12-я строка (`web-to-kmp-source-reference`) добавлена легитимно РАНЕЕ в бандле (коммит 0.5.0, `impl-ds-source-reference-skill`), НЕ в critic-fix коммите. Доктрина не нарушена, регресса нет — расхождение только в формулировке ТЗ.

## Итог
critic-правки корректны и полны; скоуп чист (3 файла в коммите, 4 в бандле); версия/CHANGELOG синхронны; валидаторы зелёные; числа подлинны; working tree clean, не main. **Бандл готов к PR.** Push — команда ГД после апрува президента; словарь Фазы-2 — board-note вне PR; рантайм-пилот отложен.

## Relations
- verifies [[impl-ds-critic-fixes]]
- part_of [[gate-bundle1-git-final]]
- audits [[phase2-provisional-iva-web-dictionary]]
