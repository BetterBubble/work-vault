---
title: RAG#2 кросс-корпусный реранк — отчёт воркера
tags:
- helm
- rag2
- rerank
- worker-report
date: 2026-07-15
permalink: tacticum/00-inbox/cross-rerank-summary
status: archived
updated: 2026-07-18
---

# RAG#2 кросс-корпусный реранк — отчёт воркера

## Итог
Добавлен ОПЦИОНАЛЬНЫЙ (за флагом, дефолт OFF) кросс-корпусный реранк финального
слитого списка в RAG#2. Единый кросс-энкодер реранкает ВЕСЬ `federate`-список
(глобальная сортировка по релевантности) — A/B к per-corpus реранку + RRF.
Строго fail-soft: сбой реранка не роняет ответ, сохраняется RRF-порядок.

## Ветка / коммит
- Worktree: `~/tacticum-worktrees/helm-cross-rerank`
- Ветка: `feat/rag2-cross-rerank` (от `origin/main` = `3a23974`)
- Финальный SHA: **`2209f98`**
- НЕ запушено (по протоколу).

## Изменённые файлы + символы
1. **`src/helm/config.py`** — новое поле `Settings.rag2_cross_rerank_enabled: bool = False`
   (рядом с `rag2_rerank_enabled`, с комментарием).
2. **`src/helm/application/rag2.py`**:
   - Новый `class DocsReranker(Protocol)` — structural-порт `rerank(query, docs, *, top_n)`,
     без импорта infrastructure (слои соблюдены).
   - `Rag2Orchestrator`: поле `cross_reranker: DocsReranker | None = None` (после `policy`).
   - `Rag2Orchestrator.answer`: сразу после `index_hits = federate(...)` — опц. блок
     кросс-реранка с `try/except` (fail-soft, `log.warning` → RRF-порядок).
3. **`src/helm/infrastructure/rag2/service.py`**:
   - `Rag2Config`: поле `cross_rerank_enabled: bool`; в `from_settings` —
     `cross_rerank_enabled=bool(g("rag2_cross_rerank_enabled", False))`.
   - `build_rag2_context`: условие постройки reranker →
     `if cfg.rerank_enabled or cfg.cross_rerank_enabled:`; введена локальная
     `corpus_reranker = reranker if cfg.rerank_enabled else None` и передана во ВСЕ три
     `JiraIndexSearch` (search/confluence/helm) — per-corpus реранк только при
     `rerank_enabled`; в `Rag2Orchestrator(...)` добавлен
     `cross_reranker=(reranker if cfg.cross_rerank_enabled else None)`.
4. **`tests/application/test_rag2_orchestrator.py`** — 3 новых теста:
   - `test_cross_reranker_reorders_merged_list` — реверс-фейк меняет порядок vs RRF.
   - `test_no_cross_reranker_keeps_rrf_order` — None → прежний RRF (регресс).
   - `test_cross_reranker_failure_is_fail_soft` — Exception → ответ не падает, порядок RRF.

## Проверки (числами)
- `pytest tests/application/test_rag2_orchestrator.py` — **14 passed** (0.03s).
- `pytest -k rag2` (весь набор) — **195 passed, 1189 deselected** (8.92s).
- `ruff check` (4 затронутых файла) — **All checks passed**.
- `mypy` (config.py, application/rag2.py, service.py) — **Success: no issues**.
- `mypy` (весь проект, config `files=["src","tests"]`) — 31 error в 7 файлах, ВСЕ
  **пред-существующие и НЕ в моих файлах** (`interface/api/routers/cio.py`,
  `tests/interface/test_req_matrix_api.py` и др.). Мои 4 файла — чисто.

## Решение по неоднозначности (важно для ревью)
Исходно все 3 `JiraIndexSearch` получали `reranker=reranker` безусловно (при выключенном
флаге переменная была None). Теперь reranker строится и в cross-only режиме, поэтому
я ЯВНО ввёл `corpus_reranker = reranker if cfg.rerank_enabled else None` и передаю её
в корпуса — чтобы в режиме «только кросс-реранк» per-corpus реранк НЕ включился
(реранк один раз — по слитому списку). Это соответствует ТЗ («Per-corpus reranker
передавай ... ТОЛЬКО при cfg.rerank_enabled»). Отклонений от ТЗ нет.

## Вопросы
Нет. ТЗ было исчерпывающим.