---
title: port-spec-eva-block3
type: report
permalink: tacticum/00-board/port-spec-eva-block3
tags:
- helm
- wave1b
- eva
- port-spec
- block3
- implementer
---

# Порт-спек EVA (Блок 3) — для implementer, дословно

База сравнения: `origin/main` для `real_source.py`/`calc.py`; текущая `feature/task-manager` для `loader.py`. Источник EVA-кода: `migration/wave1b-eva` (real_source/loader), `feature/wave1b-data-ingest` (calc). Метод read-only (`git show`/`git diff`, без checkout).

---
## 1. `eva_source.py` — CLEAN ADD (копировать as-is)
`git show migration/wave1b-eva:src/helm/ingest/eva_source.py > src/helm/ingest/eva_source.py`
Импорты все на main, сигнатуры совпадают 1:1 (проверено):
- `read_csv_rows` ← `helm.infrastructure.csv_io` ✅
- `RawEpic, RawJiraIssue` ← `helm.ingest.contract` ✅ — `RawJiraIssue(key,status,assignee_email,epic_key,parent,labels,issue_type,story_points,created,due,priority,project,summary)` — eva конструирует ровно эти поля (story_points=None) ✅
- `RawIssueLink` ← `helm.ingest.links_source` ✅ (source_key/target_key/link_type/source_project/target_project)
Правок НЕ требует. ⚠️ Кавеат: `load_eva_links` инвертирует ребро (source depends_on target → кладёт source_key=target). Проверить на реальных `eva_task_links.csv`.

---
## 2. `real_source.py` — 6 вставок; 5 ВЕРБАТИМ + 1 РУЧНАЯ
Регионы main совпадают с merge-base, кроме блока `if with_git:`. `replace` уже импортирован на main (стр.24 `from dataclasses import dataclass, replace`).

**(2.1) Импорт** — после стр.47 (`from helm.ingest.csv_source import CsvSourceError, load_reference`):
```python
from helm.ingest.eva_source import load_eva_epics, load_eva_issues, load_eva_links
```
**(2.2) Поля RealDataset** — после стр.85 (`service_signals: tuple[Signal, ...] = ()  # §6.1 …`):
```python
    eva_issues: tuple[RawJiraIssue, ...] = ()  # EVA — второй трекер (source="eva")
    eva_epics: tuple[RawEpic, ...] = ()  # EVA genesis-эпики
    eva_links: tuple[RawIssueLink, ...] = ()  # EVA зависимости §5.4
```
**(2.3) Хелперы** — после `_canonicalize_merges` (перед `def load_real_dataset`, ~стр.200):
```python
def _canonicalize_issues(
    issues: Sequence[RawJiraIssue], identity: IdentityMap
) -> tuple[RawJiraIssue, ...]:
    """Сшивает EVA-логин исполнителя → канонический `person_email` (§7)."""
    return tuple(
        replace(i, assignee_email=identity.canonicalize(i.assignee_email))
        if i.assignee_email is not None
        else i
        for i in issues
    )


def _canonicalize_epics(
    epics: Sequence[RawEpic], identity: IdentityMap
) -> tuple[RawEpic, ...]:
    return tuple(
        replace(e, assignee_email=identity.canonicalize(e.assignee_email))
        if e.assignee_email is not None
        else e
        for e in epics
    )
```
**(2.4) eva_present + init** — после стр.251 (`service_signals = tuple(load_service_signals(sd_path)) …`):
```python
    # EVA — второй трекер задач (§3/§5.4), рядом с Jira (если выгрузка есть).
    eva_tasks_path = real_dir / "eva" / "eva_tasks.csv"
    eva_present = eva_tasks_path.exists()
```
И в блок инициализации переменных (там, где `raw_commits = ()`, `raw_merges = ()`, перед `repo_product = None`) добавить:
```python
    eva_issues: tuple[RawJiraIssue, ...] = ()
    eva_epics: tuple[RawEpic, ...] = ()
    eva_links: tuple[RawIssueLink, ...] = ()
```
**(2.5) 🔴 РУЧНАЯ — блок `if with_git:` (main стр.259-272).** Main богаче ветки (multi-line `load_identity_map` + `git_identity_map_path`), поэтому НЕ применять diff ветки, а сделать так:

БЫЛО (main 259-272):
```python
    if with_git:
        identity = load_identity_map(
            real_dir / "identity" / "identity_map.csv",
            git_identity_map_path=real_dir / "identity" / "git_identity_map.csv",
        )
        team_members = _merge_roster(members, identity)
        git = RealGitAdapter(
            commits_path=real_dir / "git" / "commits.csv",
            merges_path=real_dir / "git" / "merges.csv",
        )
        raw_commits = _canonicalize_commits(git.fetch_commits(), identity)
        raw_merges = _canonicalize_merges(git.fetch_merges(), identity)
        manifest = real_dir / "git" / "repos_manifest.csv"
        repo_product = load_repo_product_map(manifest) if manifest.exists() else {}
```
СТАЛО (2 правки: guard + расщепление на второй guard):
```python
    # Карта идентичности нужна и git-канонизации, и EVA-логинам → грузим при любой.
    if with_git or eva_present:
        identity = load_identity_map(
            real_dir / "identity" / "identity_map.csv",
            git_identity_map_path=real_dir / "identity" / "git_identity_map.csv",
        )
        team_members = _merge_roster(members, identity)

    if with_git and identity is not None:
        git = RealGitAdapter(
            commits_path=real_dir / "git" / "commits.csv",
            merges_path=real_dir / "git" / "merges.csv",
        )
        raw_commits = _canonicalize_commits(git.fetch_commits(), identity)
        raw_merges = _canonicalize_merges(git.fetch_merges(), identity)
        manifest = real_dir / "git" / "repos_manifest.csv"
        repo_product = load_repo_product_map(manifest) if manifest.exists() else {}
```
(Ровно: строка guard → `if with_git or eva_present:`; пустая строка + новый `if with_git and identity is not None:` перед `git = RealGitAdapter(`. Тело git не переотступать.)

**(2.6) Блок загрузки EVA** — сразу после git-блока, перед `return RealDataset(`:
```python
    if eva_present and identity is not None:
        eva_epics = _canonicalize_epics(load_eva_epics(eva_tasks_path), identity)
        eva_issues = _canonicalize_issues(load_eva_issues(eva_tasks_path), identity)
        eva_links_path = real_dir / "eva" / "eva_task_links.csv"
        eva_links = tuple(load_eva_links(eva_links_path)) if eva_links_path.exists() else ()
```
**(2.7) Поля return** — в `return RealDataset(...)` после `service_signals=service_signals,`:
```python
        eva_issues=eva_issues,
        eva_epics=eva_epics,
        eva_links=eva_links,
```

---
## 3. `loader.py` — дельта поверх ТЕКУЩЕЙ версии (не поверх старой)
Текущие якоря (feature/task-manager == working tree): `_build_signals_and_assignments` def@310, `source="jira"`@325, `build_graph` def@535.

**(3.1)** В сигнатуру `_build_signals_and_assignments` (после `candidates: Sequence[GoalCandidate] = (),`) добавить:
```python
    source: str = "jira",
```
**(3.2)** В конструкторе `Signal(...)` заменить `source="jira",` (стр.325) на:
```python
                source=source,  # "jira" | "eva" (второй трекер) — различимы в графе
```
⚠️ Если Блок 2a (#9) добавил в этот `Signal(...)` поля status/assignee/project — СОХРАНИТЬ их, менять ТОЛЬКО строку source. Дельта EVA ортогональна Block 2a.

**(3.3)** `build_graph`: применить хунк ветки (регион не тронут Block 2a):
- в сигнатуру после `service_signals: Sequence[Signal] = (),` добавить `eva_issues`, `eva_epics`, `eva_links: Sequence[...] = ()`;
- `initiatives_by_id = _build_initiatives(goals, sales, raw_epics)` → `all_epics = (*raw_epics, *eva_epics)` + `_build_initiatives(goals, sales, all_epics)`;
- после `issue_index = _issue_initiative_index(raw_issues, …)` добавить `eva_index = _issue_initiative_index(eva_issues, ref_to_id, task_goal_judge, candidates)`;
- после основного `_build_signals_and_assignments(raw_issues, …)` добавить `eva_signals, eva_assignments = _build_signals_and_assignments(eva_issues, ref_to_id, task_goal_judge, candidates, source="eva")`;
- после `dependencies += _build_link_dependencies(raw_links, …)` добавить `dependencies += _build_link_dependencies(eva_links, ref_to_id, eva_index)`;
- в `LoadedGraph(...)`: `signals=signals + eva_signals + git_signals + merge_signals + tuple(service_signals)`, `assignments=assignments + eva_assignments + git_assignments + merge_assignments`.

⚠️ Проверить, что вызов `build_graph` из real_source-конвейера прокидывает `eva_issues=…, eva_epics=…, eva_links=…` из `RealDataset` (иначе EVA грузится, но в граф не попадает). Найти call-site build_graph и добавить 3 kwargs.

---
## 4. `calc.py` — `_WORK_GAP_SOURCES` (из feature/wave1b-data-ingest), регион не тронут
Перед `def work_without_goal` вставить:
```python
#: Источники-сигналы, которые считаются «работой» в разрыве §6.1 «работа без цели».
#: Только ТРЕКЕРЫ ЗАДАЧ (jira/eva): несопоставленная задача с выраженным намерением =
#: governance-разрыв. Сырые git-коммиты и service-desk НЕ считаем (шум для ГД).
_WORK_GAP_SOURCES: frozenset[str] = frozenset({"jira", "eva"})
```
И тело `work_without_goal` (`return [s.external_id for s in signals if s.initiative_id is None]`) → 
```python
    return [
        s.external_id
        for s in signals
        if s.initiative_id is None and s.source in _WORK_GAP_SOURCES
    ]
```
⚠️ ДУБЛЬ смысла: в `application/tasks.py` уже есть `TASK_SOURCES={jira,eva}` (для витрины headline). Здесь `_WORK_GAP_SOURCES` — для §6.1 gaps. Оба корректны, но чтобы не разошлись — рассмотреть единый источник (импортировать один из другого). Не блокер.

---
## Тесты и данные
- `git show migration/wave1b-eva:tests/ingest/test_eva_source.py > tests/ingest/test_eva_source.py` — тесты eva_source (clean).
- Данные: `data/real/eva/eva_tasks.csv`, `data/real/eva/eva_task_links.csv`. Без них `eva_present=False` → всё деградирует gracefully (прод не ломается).
- Порядок применения: eva_source(add) → calc → loader → real_source(руч.) → прокинуть kwargs в call-site build_graph → тест.


---
## 5. `scripts/seed_db.py` — call-site build_graph (ГЛАВНЫЙ прод-путь ingest→DB)
Call-sites `build_graph` (не тесты): `scripts/seed_db.py:_load_graph@74` (реальный DB-сид, load_real_dataset→build_graph — **это путь, который питает витрину/API**), плюс вспомогательные `scripts/export.py:_build_portfolio@37`, `scripts/project_to_memory.py@32`, `scripts/seed.py@27` (синтетика).

**wave1b-eva УЖЕ содержит проводку в `seed_db.py` — портировать её diff ВЕРБАТИМ** (защитный `getattr`, безопасно):
```python
    eva_issues = getattr(dataset, "eva_issues", ())  # EVA второй трекер
    eva_epics = getattr(dataset, "eva_epics", ())
    eva_links = getattr(dataset, "eva_links", ())
    graph = build_graph(
        ...
        service_signals=service_signals,
        eva_issues=eva_issues,
        eva_epics=eva_epics,
        eva_links=eva_links,
        task_goal_judge=judge,
        manual_merges=merges,
    )
```
+ диагностический print (EVA: сколько сигналов, привязано/висит, epic-инициатив jira+eva) — тоже в diff, полезен для sanity-чека после сида.

⚠️ `export.py`/`project_to_memory.py` ветка НЕ трогает → там EVA пойдёт по дефолту (пусто). Если EVA нужна и в portfolio-export/memory-sync — добавить те же 3 `getattr`+kwarg (низкий приоритет; деградирует gracefully). Для витрины orphan-rate достаточно `seed_db.py`.

**Итог call-site: пробел закрыт — порт `seed_db.py` diff из wave1b-eva завершает проводку.**
