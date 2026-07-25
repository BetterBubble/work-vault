---
title: WP-B git-API — hand-off implementer-2 → implementer-1 (точки вшивки в real_source)
type: note
tags:
- helm
- wave-1b
- wp-b
- git-api
- handoff
permalink: tacticum/00-board/wp-b-gitapi-handoff-to-implementer-1-1
---

# WP-B git-API — как вшить в `real_source.py` (владелец — implementer-1)

Ветка `migration/wave1b-gitapi` (implementer-2). Я НЕ трогал `real_source.py`.
Готовые хуки ниже — подключаются в блоке `if with_git:` (`load_real_dataset`).

## 1. Коммиты: старый `commits.csv` (24k) → `all_commits.csv` (216k)
```python
from helm.ingest.git_source import RealGitAdapter
git = RealGitAdapter(all_commits_path=real_dir / "git" / "all_commits.csv")
raw_commits = _canonicalize_commits(git.fetch_commits(), identity)
```
`RealGitAdapter.all_commits_path` — новый приоритетный источник (non-breaking:
`commits_path` стал опционален). `repo` в коммитах теперь ПОЛНЫЙ gitlab-путь
(`iva-m/android/kmp`), а не короткое имя — это важно для джойна repo→product (см. §3).

## 2. Merge-requests: `merge_requests_all.csv` (1115, со `state`)
```python
from helm.ingest.mr_source import load_merge_requests_as_raw
mr_path = real_dir / "git" / "merge_requests_all.csv"
raw_merges = (
    _canonicalize_merges(load_merge_requests_as_raw(mr_path), identity)
    if mr_path.exists() else ()
)
```
`load_merge_requests_as_raw` → `list[RawGitMerge]` (контракт, готов для `_canonicalize_merges`).
Если нужен `state` (merged/opened/closed) — есть богатый `load_merge_requests(path)` →
`list[RawMergeRequest]` (repo·mr·state·author_name·date·target_branch·jira_keys·title).

## 3. repo→product на полном корпусе (233 репо) для M5-скоринга
Старый `load_repo_product_map` ключевал по коротким именам — НЕ джойнится с полными
путями `all_commits.repo`. Замена:
```python
from helm.ingest.re_enrich import build_full_repo_product_map
repo_product = {
    repo: rp.product
    for repo, rp in build_full_repo_product_map(real_dir / "git").items()
    if rp.product != "unknown"
}
```
Ключи = полные пути (совпадают с `RawGitCommit.repo`). Провенанс на реальных:
curated 10 / heuristic 41 / unknown 182 (unknown не кладём в карту — «не выдумываем»).

## Проверено на реальных
- commits 216161 · с jira-ключом 75828 · distinct-ключей 30316;
- MR 1115 · с ключом 755 · states merged 860 / opened 169 / closed 86;
- repo→product 233 репо · curated 10 / heuristic 41 / unknown 182.

Тесты: `tests/ingest/test_all_commits.py`, `test_mr_source.py`, `test_repo_product_full.py`
(425 passed, mypy+ruff чисто). Ядро/loader не трогал.