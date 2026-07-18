---
title: explore-wave1b-port-manifest
type: report
permalink: tacticum/00-inbox/explore-wave1b-port-manifest
tags:
- helm
- wave1b
- migration
- port-manifest
- explore
status: archived
updated: 2026-07-18
---

# Манифест портирования wave1b → main (read-only разведка)

Метод: `git merge-tree --write-tree main <branch>` (реальные конфликты без правки дерева) + `git diff main...<branch>`. Цель порта — **origin/main** (e21a29e). Текущая рабочая ветка `feature/task-manager` от main отличается только `models.py`+4 / `loader.py`+1 (колонка status, Фаза 2).

## Отношения веток (важно!)
- **`feature/wave1b-data-ingest` = ЗОНТИК** = `wave1b-eva` ∪ `wave1b-gitapi` ∪ доп. `calc.py`. Superset.
- `migration/wave1b-eva` ⊂ data-ingest (EVA-трекер).
- `migration/wave1b-gitapi` ⊂ data-ingest (git repo→product, MR).
- `impl-identity-1b` — **СУПЕРСЕД�ед**: main's `identity.py` БОГАЧЕ (231 строк только в main: alias_map/load_identity_map/IdentityMap/ФИО-норм; в ветке уникальны лишь 2 док-строки). Порт НЕ нужен.
- `wave1b-people`, `wave1b-teams` — независимые, чистые.

## Merge-tree на main (факт)
| Ветка | Вердикт | Конфликт |
|---|---|---|
| wave1b-gitapi | ✅ CLEAN | — |
| wave1b-people | ✅ CLEAN | — |
| wave1b-teams | ✅ CLEAN | — (identity.py авто-мерж) |
| wave1b-eva | ⚠️ CONFLICT | `real_source.py` (loader.py авто-мерж) |
| wave1b-data-ingest | ⚠️ CONFLICT | `real_source.py` (loader.py+re_enrich.py авто) |
| impl-identity-1b | ⚠️ add/add | `identity.py` → **SKIP** (superseded) |

## Манифест: файл → блок → clean/adapt/skip

### Блок 3 — EVA (второй трекер) [задача #11] — источник `wave1b-eva`
- `src/helm/ingest/eva_source.py` — НЕТ на main → **CLEAN ADD** (154 стр: `load_eva_epics/eva_issues/eva_links`, source="eva").
- `src/helm/ingest/real_source.py` — **ADAPT (единственный настоящий конфликт)**: ветка добавляет на `RealDataset` поля `eva_issues/eva_epics/eva_links`, хелперы `_canonicalize_issues/_canonicalize_epics` (сшивка EVA-логина→person_email через IdentityMap), и блок загрузки `data/real/eva/eva_tasks.csv`. Конфликт — потому что main отрефакторил блок `if with_git:`; ветка меняет на `if with_git or eva_present:`. Мердж ручной, но АДДИТИВНЫЙ, ~30 строк, низкий риск.
- `src/helm/ingest/loader.py` — **CLEAN (авто-мерж)**: `build_graph` += `source="jira"` параметр в `_build_signals_and_assignments`, `eva_issues/epics/links`, EVA-эпики в общий генезис, eva-signals/deps. ⚠️ КАВЕАТ: `feature/task-manager` тоже трогает loader (status) — при порте на task-manager-базу, а не чистый main, глянуть тот же `_build_signals_and_assignments` (status vs source в одной функции).
- `src/helm/domain/calc.py` — **CLEAN** (только в data-ingest, не в eva-ветке!): вводит `_WORK_GAP_SOURCES={jira,eva}` и фильтрует `work_without_goal` по source → orphan считает по ТРЕКЕРАМ задач, ИСКЛЮЧАЯ сырые git-коммиты (216k без jira-ключа) и service-desk как шум. **Ключевой для достоверности orphan-rate + мультитрекерности.** Брать из data-ingest.
- `scripts/seed_db.py` (+14) — CLEAN; `tests/ingest/test_eva_source.py` — CLEAN ADD.

### Gitapi — repo→product / MR [часть #10-#11] — источник `wave1b-gitapi` (ВСЯ ветка CLEAN)
- `src/helm/ingest/git_source.py` (+51) — CLEAN (repo→product map).
- `src/helm/ingest/mr_source.py` — НЕТ на main → CLEAN ADD (+97, merge-request как PR-прокси).
- `src/helm/ingest/re_enrich.py` (+93) — CLEAN (авто-мерж; main уже имеет файл).
- tests: `test_all_commits`, `test_mr_source`, `test_repo_product_full` — CLEAN ADD.
- Рекомендация: **портировать gitapi целиком одним мержем** (конфликтов нет).

### Блок 2.1 — identity/teams/people
- `impl-identity-1b` — **SKIP** (superseded, см. выше).
- `wave1b-teams` (CLEAN): `identity.py` +110 (`normalize_name`/`name_variants`/`match_name_to_email` + `resolve_git_clusters`/`GitClusterResolution` — дотягивание несшитых git-кластеров, Блок2.1 риск №1); `operator_inputs.py` +20; `scripts/derive_real_manual.py` +171; тесты. Порт чистый.
- `wave1b-people` (CLEAN): `application/scoring.py` +38, `governance.py`, `ingest/economics.py` (FOT), `scripts/score_value.py`, тесты. Порт чистый.

## Порядок портирования (рекоменд.)
1. `wave1b-gitapi` целиком (CLEAN) → git repo→product + MR.
2. EVA: `eva_source.py` (clean add) + `calc.py` `_WORK_GAP_SOURCES` (из data-ingest) + `loader.py` (авто) + **`real_source.py` руками** (~30 строк аддитивно).
3. `wave1b-teams` + `wave1b-people` (CLEAN, независимо).
4. `impl-identity-1b` — не портировать.
Альтернатива для шагов 1-2: взять зонтик `wave1b-data-ingest` (eva+gitapi+calc разом), разрулив один конфликт в `real_source.py`.

## Данные, которых требует EVA-порт (проверить наличие)
`data/real/eva/eva_tasks.csv`, `data/real/eva/eva_task_links.csv`, `data/real/identity/identity_map.csv` (нужен и для git, и для EVA-канонизации). Без них EVA-путь просто не активируется (граждан. деградация: `eva_present=False`).


---

## АДДЕНДУМ (по уточнённому запросу лида: read full eva_source, делетабельность, gitapi-вердикт)

Метод: только `git show`/`git log`/`git diff --stat <merge-base>..<branch>`/`git cherry` (без checkout — рабочее дерево занято другим воркером).

### Уник-коммиты vs origin/main (git cherry, `+`=невлитой)
| Ветка | ahead | unmerged(+) | Вывод |
|---|---|---|---|
| migration/wave1b-eva | 1 | **1** | уник — портировать |
| feature/wave1b-data-ingest | 6 | **5** | superset (1 уже в main) |
| migration/wave1b-gitapi | 3 | **3** | уник, но git-скоуп |
| migration/wave1b-people | 1 | **1** | уник — проверить точечно |
| migration/wave1b-substrate-wire | 1 | 1 | уник, НО в `src/` изменений нет (док/тест) → для кода делетабельно |
| migration/wave1b-m9 | 1 | 1 | уник: `application/governance.py` — вне таск-скоупа, проверить |
| **impl-identity-1b** | 1 | **0** | **УЖЕ В MAIN** → skip+delete |
| **migration/wave1b-teams** | 2 | **0** | **УЖЕ В MAIN** → skip+delete |
| wave1b-m8 / re-m3 / substrate / stream-ingest | 0 | 0 | уже в main → delete |
| wave1a-polish, wave1a-real-seed, wave1b-c3c5, competency, deps, git-identity, links, m5, sales-quality, seed-clean, t2 | 0 | 0 | уже в main → delete |
| wave-1a-backend | 8 | 6 | старая, вне скоупа |

**Под удаление сразу (0 невлитых): impl-identity-1b, wave1b-teams, wave1b-m8, wave1b-re-m3, wave1b-substrate, wave1b-stream-ingest + все ahead=0** (wave1a-polish, wave1a-real-seed, wave1b-c3c5, competency, deps, git-identity, links, m5, sales-quality, seed-clean, t2). Перед удалением — safety-тег (задача #13).

### eva_source.py — полный разбор (154 стр, migration/wave1b-eva)
Импорты и совпадение контрактов на main (ПРОВЕРЕНО):
- `from helm.infrastructure.csv_io import read_csv_rows` — ✅ есть на main.
- `from helm.ingest.contract import RawEpic, RawJiraIssue` — ✅ есть; сигнатуры СОВПАДАЮТ (RawJiraIssue: key/status/assignee_email/epic_key/parent/labels/issue_type/story_points/created/due/priority/project/summary — eva конструирует ровно эти; RawEpic: key/title/project/product/generation/assignee_email — совпадает).
- `from helm.ingest.links_source import RawIssueLink` — ✅ есть; поля source_key/target_key/link_type/source_project/target_project совпадают.
- **Вердикт: eva_source.py портируется ЧИСТО (pure NEW-файл, 0 конфликтов), НО инертен без проводки** (loader source= + real_source loader-блок).

Что делает (3 публ. функции):
- `load_eva_epics(tasks_path)` → genesis-эпики: `type=Epic` ИЛИ проект `operations-directorate` (управленческие инициативы, аналог Jira-CEO).
- `load_eva_issues(tasks_path)` → `RawJiraIssue` со `status=status_type`, `source` проставит loader = "eva".
- `load_eva_links(links_path)` → `RawIssueLink` из `depends_on`/`affects`, с ИНВЕРСИЕЙ рёбер (source зависит от target → кладёт source_key=target). ⚠️ проверить направление на реальных данных.
- Встроенный `_EVA_PRODUCT` dict (project_code→продукт) — EVA-задача→product УЖЕ внутри (для orphan-by-product EVA отдельный map не нужен).

### gitapi — вердикт: СКИП для таск-менеджера
- `git_source.load_repo_product_map(manifest_path)->dict[str,str]` = git **REPO→продукт** (из `repos_manifest.csv`, 233 репо). Это для атрибуции git-КОММИТОВ на продукт (Т1/M3), НЕ для задач.
- Для orphan-**by-product** таск-менеджера нужен **ЗАДАЧА→продукт**, а он уже есть: `links_source.load_jira_project_product` (Jira project→product, ✅ на main) + `_EVA_PRODUCT` внутри eva_source. **gitapi не нужен** для таск-скоупа.
- `mr_source.py` (merge-requests) и re_enrich — тоже git-контур, не задачи. Скип для #9-13; ветку `wave1b-gitapi` держать для будущего git/CI-среза (там 3 реальных уник-коммита — НЕ удалять).

### Итог по 3 целевым
1. **wave1b-eva** → БРАТЬ. eva_source.py (clean add) + проводка: loader (авто) + **real_source руками ~30 стр** (конфликт из-за рефактора `if with_git:`→`if with_git or eva_present:`) + calc `_WORK_GAP_SOURCES` (взять из data-ingest).
2. **wave1b-data-ingest** → superset; либо цельным мержем (1 конфликт real_source), либо cherry-pick eva+calc.
3. **wave1b-gitapi** → СКИП для таск-менеджера (git repo→product, не задача→product); ветку сохранить.
