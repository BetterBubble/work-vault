---
title: 'explore-eval-harness (вынос #86)'
type: note
permalink: tacticum/00-board/explore-eval-harness-vynos-86
tags:
- eval
- refactor
- task-86
- explore
status: archived
updated: 2026-07-18
---

# explore-eval-harness — разведка для выноса в платформу (#86)

Репо: `rag_eval_service` (serena-проект `codex`). Только чтение. Дата: 2026-06-30.
ВАЖНО: класса `EvalHarness` нет — harness реализован функционально (`eval/runner.py::evaluate` + CLI `main`).

## (а) Публичная поверхность (символ → сигнатура → модуль)

### eval/metrics.py — чистые функции (только `math`)
- `recall_at_k(retrieved: list[str], relevant: list[str], k: int) -> float`
- `mrr(retrieved: list[str], relevant: list[str]) -> float`
- `ndcg_at_k(retrieved: list[str], relevant: list[str], k: int) -> float`

### eval/backends.py — порт + адаптеры
- `EvalHit` (@dataclass): `doc_id: str`, `score: float|None=None`, `raw: dict|None=None`
- `SearchBackend(Protocol)`: атрибут `name: str`; `search(query, *, tenant_id, mode, top_k, rerank=False) -> list[EvalHit]`; `describe() -> dict`
- `InProcessBackend` (name="inprocess"): адаптер на `rag.search.search()` — Codex-специфичен; rerank → NotImplementedError
- `KnowledgeBackend` (name="knowledge"): HTTP-адаптер, env-driven (KNOWLEDGE_*)
- `get_backend(name: str|None=None) -> SearchBackend`: фабрика по `EVAL_BACKEND`/имени (inprocess|knowledge)

### eval/runner.py — ядро harness + CLI
- `evaluate(backend: SearchBackend, golden: list[dict], k: int, mode: str, tenant_override: str|None, rerank: bool=False) -> list[dict]` — чистое ядро (зависит только от порта + metrics)
- `main()` — argparse CLI; приватные хелперы агрегации/срезов: `_aggregate`, `_slice_by`, `_slices`, `_print_slices`, `_warn_empty`, `_list_docs`, `_print_run`, `_retrieved_doc_ids`, `_METRICS`

### eval/validate.py — валидатор golden-схемы
- `check_schema(...)`, `_index_doc_ids`, `main`; константы `REQUIRED/OPTIONAL/DIFFICULTY/SOURCE`

`eval/__init__.py` — пустой (нет ре-экспортов).

## (б) Call-sites — ВСЕ внутри пакета eval/ (внешних потребителей в исходниках НЕТ; тестов на eval нет)
- `evaluate` ← `runner.main` (runner.py:199 мультимод, 213 одиночный)
- `get_backend` ← `runner.main` (runner.py:181)
- `SearchBackend` ← сигнатура `evaluate` (runner.py:39); база Protocol в backends
- `EvalHit` ← `_retrieved_doc_ids` (runner.py:29); конструируется в `InProcessBackend.search`/`KnowledgeBackend.search`
- `recall_at_k`/`mrr`/`ndcg_at_k` ← `evaluate` (runner.py:60-62), импорт внутри функции (runner.py:46)
- Внешний импорт `from eval...` вне пакета: НЕ найдено → весь пакет переносится как целое, «адаптировать вызов снаружи» пока некого.

## (в) Локальные «провода» (привязки к rag_eval_service) + вердикт
1. `from rag.config import settings` (runner.py:24, backends.py:15). Использует: `settings.search_mode` (дефолт --mode, runner.py:165), `settings.default_tenant` (runner.py:178), `settings.embed_backend`+`settings.collection` (InProcessBackend.describe). → **ПАРАМЕТРИЗОВАТЬ**: пробросить через config-объект/CLI-аргументы платформы, не тащить `rag.config`.
2. `from rag import search as rag_search` (InProcessBackend.search, backends.py:44). → **АДАПТЕР, специфичен Codex**. Порт `SearchBackend`+`EvalHit`+`KnowledgeBackend` переносятся; `InProcessBackend` остаётся в потребителе (rag_eval_service) либо в платформе как опц. адаптер за портом.
3. `from rag.store import get_store` (runner.py:25) — только в `_list_docs` (diag-команда `--list-docs`, scroll Qdrant через `qdrant_client.models`). → **вынести опционально/оставить в потребителе**; жёсткая привязка к Qdrant и store rag_eval_service. Не часть ядра evaluate.
4. `qdrant_client` (import внутри `_list_docs`, runner.py:125) — внешняя либа, только для diag.
5. golden default path: `pathlib.Path(__file__).parent / "golden_sets" / "example.json"` (runner.py:163). → **ПАРАМЕТРИЗОВАТЬ дефолт** (сам аргумент `--golden` уже есть; путь-зависимость на golden_sets — содержательно разбирает verifier).
6. env-vars в `get_backend`: `EVAL_BACKEND`; `KNOWLEDGE_URL/TOKEN/DOC_ID_FIELD/PROJECT_ID/USE_GATEWAY_HEADERS`. → env-driven ок для платформы; KNOWLEDGE_* специфичны knowledge-адаптеру (TOKEN — секрет, только env).

## Резюме для плана выноса
- Переносится «чисто»: `metrics.py` (0 локальных зависимостей), порт `SearchBackend`+`EvalHit`, ядро `evaluate`, `KnowledgeBackend` (env-driven).
- Требует параметризации: `settings.*` (mode/tenant/embed_backend/collection), дефолтный путь golden.
- Остаётся/адаптируется в потребителе: `InProcessBackend` (rag.search), `_list_docs`/get_store (Qdrant diag).
- Граница «переносим vs адаптируем вызов»: внешних call-sites нет → мигрирует весь пакет; адаптировать нужно только внутренние Codex-привязки (#2,#3), а не чужие вызовы.
