---
title: impl-rag2-tests-match-fix
type: note
permalink: tacticum/00-board/impl-rag2-tests-match-fix
tags:
- rag2
- allure
- tests-coverage
- impl-report
---

# impl-rag2-tests-match-fix

status: draft
worktree: /Users/bubblemac/tacticum/helm-wt-rag2-testsfix
branch: feat/rag2-tests-fix
commit: bbdb1e4

## Вердикт
**Код-фикс** (в джобе снапшота Allure). НО эффект наступит только после **ре-ингеста снапшота на проде** (`python -m helm.ingest.allure_snapshot`) — артефакт `allure_snapshot.json` перегенерится с расширенным набором ключей. Есть остаточная **зависимость от данных** (см. ниже) — если после ре-ингеста звонки всё ещё null, корень в данных/конфиге, не в коде.

## Причина (точная, по коду)
Матч требование→тесты идёт так:
1. `build_tests_summary` (topology.py:130) → берёт rid'ы, `_requirement_test_bridge` (analyst_server.py:202) отдаёт **все** `jira_keys`+`components` требования (тут фильтра НЕТ).
2. `AllureCoverage.for_requirement` (domain/allure.py:154) матчит `jira_keys` требования против `snapshot.by_jira`, `components` против `by_feature`. Ключа нет в снапшоте → `matched_by=none` → строка выпадает → `tests=null`.
3. Снапшот `by_jira` строит джоба `ingest/allure_snapshot.py::_main`. **Корень:** выборка ключей была захардкожена
   `select(RequirementJira.jira_key).where(jira_key.like("IVAONE-%"))`.

`by_jira` — это покрытие ОДНОГО TestOps-проекта `project_one`, где `testcases_by_issue(project_one, key)` ищет по RQL `issue = "<key>"`. В этом проекте лежат TC, привязанные к jira-задачам из **разных** jira-проектов (IVAONE, IVAONEHALF, IVACS/IVASWQA…). Фильтр `IVAONE-%`:
- терял ключи требований из других проектов (звонки IVA CS/SIP на IVACS-/IVASWQA-/IVAONEHALF-),
- даже IVAONEHALF- (который сам код в `tracks.PROJECT_TRACK_MAP` считает track "one") — выпадал.
Мы просто **никогда не запрашивали** эти ключи в Allure, хотя TC под них в `project_one` есть → покрытие ложно нулевое.

`RequirementJira.track` для скоупа НЕ годится — это эвристика по компонентам задачи (`component_track`, дефолт "kmp"), ненадёжна. Префикс проекта надёжнее, но неполон. Поэтому берём **все** ключи, а отсев делает сам Allure.

## Что починил
`src/helm/ingest/allure_snapshot.py`:
- Вынес выборку в тестируемые хелперы `select_jira_keys(session)` / `select_components(session)`.
- `select_jira_keys` берёт **ВСЕ** `RequirementJira.jira_key` (любой проект), без `LIKE 'IVAONE-%'`.
- `_main` использует хелперы.
- Регрессии для kmp-актов (Почта/ВКС/Календарь) НЕТ: их покрытие идёт через `by_feature` (Feature=компонент в `project_kmp`); их jira-ключи в `project_one` TC не имеют → `_entry` вернёт None → в `by_jira` не попадут (существующее IVAONE-поведение неизменно, `for_requirement` предпочитает jira только при реальном совпадении).

`build_snapshot` (чистая функция) уже был prefix-agnostic — правка не в нём.

## Файлы
- src/helm/ingest/allure_snapshot.py — фикс + хелперы
- tests/ingest/test_allure_snapshot.py — 2 новых теста

## Тесты (числа)
- `pytest tests/ingest/test_allure_snapshot.py tests/interface/test_req_tests_block.py tests/interface/test_analyst_mcp.py` → **85 passed** (warnings — косметика aiosqlite при dispose).
- Новые: `test_build_snapshot_includes_non_ivaone_key` (ключ IVACS-7 с TC → в by_jira, `for_requirement(jira_keys=["IVACS-7"])` → matched_by="jira", 2 TC), `test_select_jira_keys_all_projects` (ключи IVACS-7/IVAONE-11087/IVAONEHALF-2 — все три возвращаются, раньше 2 из 3 терялись).
- ruff: clean. mypy (src): Success, no issues.

## Остаточная зависимость от данных (проверить на проде после ре-ингеста)
Код-фикс сработает ТОЛЬКО если для требования по звонкам выполнено ОДНО из:
1. В `requirement_jira` у требования есть jira_key (любой проект), И в TestOps `project_one` есть TC с `issue = "<этот key>"`. ← наиболее вероятно (эталон: 96 ТК существуют).

Если после ре-ингеста звонки всё ещё `tests=null` — корень в ДАННЫХ/КОНФИГЕ, кодом не чинится:
- звонковые TC лежат в ДРУГОМ Allure-проекте (не `project_one` и не `project_kmp`) → нужен маппинг доп. проекта в `build_snapshot`/настройках (`allure_project_*`);
- у требования нет jira-привязки в `requirement_jira`, а имя компонента "Звонки" не совпадает буквально с Feature в Allure `project_kmp` → by_feature промах (наименование/привязка в БД);
- TC в Allure не привязаны к jira-issue звонков (пустой RQL `issue=`).

Как проверить быстро на проде: (а) прогнать джобу, глянуть `manifest.json` `by_jira` (должно вырасти); (б) в snapshot `by_jira` поискать ключ требования звонков; (в) если нет — руками `testcases_by_issue(project_one, "<key>")` вернёт ли TC.

## Действие для лида/пользователя
1. Ревью ветки `feat/rag2-tests-fix` (commit bbdb1e4).
2. После мержа — **ре-ингест снапшота на проде**, иначе `/answer` для звонков останется null (артефакт старый).
3. Проверить звонки в `/api/rag2/answer` → поле `tests` должно подтянуться (26/63 зелёных ≈41%).
