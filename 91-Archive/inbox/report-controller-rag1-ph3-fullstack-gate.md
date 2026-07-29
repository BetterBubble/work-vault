---
title: report-controller-rag1-ph3-fullstack-gate
type: report
permalink: tacticum/00-board/report-controller-rag1-ph3-fullstack-gate
tags:
- rag1
- controller
- gate
- rerank
- latency
archived-at: 2026-07-29 18:12
---

# Контролёр — финальный гейт стека RAG#1 (Ф1+Ф2+Ф3)

Ветка `feat/rag1-ph3-rerankcap`, worktree `/Users/bubblemac/tacticum/helm-wt-rag1-ph3`, коммит `119ff00` (единственный поверх `feat/rag1-ph2-clarify`). Ф1+Ф2 не переигрываю (см. `report-controller-rag1-ph1-ph2-gate`) — фокус на дельте Ф3 (вариант A) и интеграции стека.

## Вердикт по пунктам

1. **Дельта Ф3 (коммит 119ff00) — OK.** Порядок стадий цел: `search → cap(chunks[:cap]) → rerank → floor → cap_per_doc/context_limit → clarify → generation` (`application/docs_assistant.py:220-242`). Cap применяется ПОСЛЕ ретрива/RRF/dedup и ДО реранка, ВНУТРИ `if self._reranker is not None` → путь без реранка не затронут. `top_n=len(candidates)` корректен для урезанного списка. Проброс полный: `config.py:169 docs_rerank_candidate_cap=20` (env `HELM_DOCS_RERANK_CANDIDATE_CAP`) → `service.py:43,117` (context) → `docs.py:217` + `bot_support.py:115` (оба роутера) → `DocsAssistant.__init__`.

2. **Гит-чистота дельты — OK.** 7 файлов строго по задаче A (+97/-2), без миграций. Нет секретов/`.env`/бинарников/мусора/AI-подписей. Автор коммита — человек (Александр Шульга). Ветка явная, не main. Правка тест-фикстуры `test_analyst_mcp.py:434` (новое поле ctx) — обязательна, иначе AttributeError.

3. **Cap=0/None — OK.** `candidates = chunks[:cap] if cap and cap > 0 else chunks`: `0` и `None` falsy → без урезания (short-circuit до `cap > 0`, TypeError нет). Покрыто `test_rerank_candidate_cap_zero_disables_cap`.

4. **Полный сьют (запущен, не на доверии) — OK.** `uv run pytest -q` → **1707 passed, 31 skipped, 32 warnings, 33.89s** (ожидали ~1707 ✓). ruff на файлах Ф3 → чисто. mypy src/ → 23 ошибки в 5 файлах ВНЕ Ф3 (`req_matrix.py`, `contract_index.py`, `gateway.py`, `cio.py`) — baseline. ruff 5 ошибок ВНЕ Ф3: заявленный `models.py:1425` (E501) + `_confluence.py:197`, `requirements.py:487/488`, `test_component_dedup.py:90` — baseline. thread-warning «Event loop is closed» (ResourceWarning) — baseline, не падение. Все новые файлы Ф3 чисты по ruff/mypy. Уточнение: воркер гонял mypy только на 5 изменённых src-файлах («Success») — верно для них; полный mypy даёт baseline-ошибки в нетронутых файлах, противоречия нет.

5. **Интеграция стека — OK.** Ф3 не трогает clarify (Ф2) и краткость (Ф1) — только cap. Confidence-гейт судит по реранкованным chunks, полученным из капнутого набора → целостность гейта сохранена. Замечание (НЕ блокер): при `allow_clarify=True` реранк теперь видит топ-20 вместо всех ≤30 (SEARCH_LIMIT=30), т.е. гейт может видеть чуть иной набор — это принятый tradeoff варианта A; при деплое clarify OFF рантайм-влияния нет.

6. **Готовность к мержу стека — OK.** ph3 линейно поверх ph2 (`merge-base --is-ancestor` = true) → сквозной ff-merge ph3 в main приносит ph1+ph2+ph3 разом (либо последовательно ph1→ph2→ph3). Alembic: Ф3 БЕЗ миграций; multi-head baseline → на деплое `alembic upgrade heads`. Флаги: `docs_clarify_enabled=False` (clarify OFF ✓).
   **Эксплуатационное напоминание (важно):** `docs_rerank_enabled` по дефолту False. Cap Ф3 влияет ТОЛЬКО на путь с активным реранкером — чтобы оптимизация латентности сработала, реранк должен быть ВКЛючён на деплое (`HELM_DOCS_RERANK_ENABLED=true`). Иначе cap — no-op.

## Итог
**Весь стек RAG#1 (Ф1+Ф2+Ф3) готов к мержу + деплою.** Блокеров нет. Единственное операционное условие для эффекта Ф3: rerank ON на деплое.
