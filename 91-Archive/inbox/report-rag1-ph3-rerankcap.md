---
title: report-rag1-ph3-rerankcap
type: report
permalink: tacticum/00-board/report-rag1-ph3-rerankcap
status: draft
tags:
- rag1
- latency
- rerank
- implementer
archived-at: 2026-07-29 18:12
---

# RAG#1 Ф3 — кап кандидатов на входе реранка (вариант A, латентность)

## Изоляция
- Worktree: `/Users/bubblemac/tacticum/helm-wt-rag1-ph3`
- Ветка: `feat/rag1-ph3-rerankcap` (стек поверх `feat/rag1-ph2-clarify`)
- Коммит: `119ff00`. НЕ мержено/пушено/деплоено.

## Что сделано
Режем ВХОД реранка до топ-N кандидатов ПОСЛЕ ретрива/RRF/dedup и ДО реранка. Порядок стадий сохранён (retrieve → cap → rerank → floor → cap-per-doc → context). Путь без реранка (`reranker is None`) не затронут.

### Изменённые файлы/символы
- `src/helm/config.py` — новое поле `Settings.docs_rerank_candidate_cap: int = 20`, env `HELM_DOCS_RERANK_CANDIDATE_CAP`. 0/None → без капа.
- `src/helm/application/docs_assistant.py` — `DocsAssistant.__init__` новый параметр `rerank_candidate_cap: int = 20` (поле `self._rerank_candidate_cap`); в `ask` перед `rerank`: `candidates = chunks[:cap] if cap and cap > 0 else chunks`, реранкуем `candidates`, `top_n=len(candidates)`.
- `src/helm/infrastructure/docs_assistant/service.py` — поле `DocsAssistantContext.rerank_candidate_cap` + проброс `settings.docs_rerank_candidate_cap` в `build_docs_assistant_context`.
- `src/helm/interface/api/routers/docs.py` и `.../bot_support.py` — `_build_assistant` пробрасывает `ctx.rerank_candidate_cap`.
- `tests/interface/test_analyst_mcp.py` — фикстура ctx (SimpleNamespace) дополнена полем (иначе AttributeError).

### Как прокинут cap
`Settings.docs_rerank_candidate_cap` → `build_docs_assistant_context` → `DocsAssistantContext.rerank_candidate_cap` → `_build_assistant` (оба роутера) → `DocsAssistant(rerank_candidate_cap=...)` → применяется в `ask` строго перед вызовом реранка.

## Тесты
- Юнит-тесты на cap (`tests/application/test_docs_assistant.py`, новый `_RecordingReranker` запоминает поданный список):
  - `test_rerank_candidate_cap_limits_input_to_reranker` — корпус 60, cap=20 → реранкер получает РОВНО топ-20 в порядке ретрива, `top_n=20`.
  - `test_rerank_candidate_cap_no_truncation_when_corpus_below_cap` — корпус 5, cap=20 → 5 (без усечения).
  - `test_rerank_candidate_cap_zero_disables_cap` — cap=0 → весь набор 60.
  - `test_rerank_candidate_cap_ignored_without_reranker` — без реранка cap не режет (в контексте все 20, `[20]` присутствует).
- Регресс: путь без реранка, floor, context_limit/top_n — без изменений.
- Результаты: `pytest tests/application tests/infrastructure/test_reranker.py tests/interface` → **539 passed**. `ruff check` (изменённые файлы) → clean. `mypy` (5 src-файлов) → Success, no issues. Baseline-ошибок репо не обнаружено.

## Подтверждения
- Env `HELM_DOCS_RERANK_CANDIDATE_CAP=7` → `Settings().docs_rerank_candidate_cap == 7`; без env дефолт `20`. Проверено.
- Замер эффекта (recall/латентность cap20 vs cap60) снимает ГД на проде отдельно.

## Diff stat (vs Ф2)
7 файлов, +97/-2. Только cap — clarify/промпт/параллелизация не тронуты.
