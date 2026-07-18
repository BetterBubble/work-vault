---
title: LightRAG implementer — план реализации (ждёт «ок» тимлида)
tags:
- lightrag
- codex
- migration
- plan
- implementer
worktree: /Users/bubblemac/tacticum-worktrees/rag_eval_service-lightrag
branch: migration/lightrag
date: 2026-07-01
permalink: tacticum/00-inbox/lightrag-implementer-plan
status: archived
updated: 2026-07-18
---

# LightRAG в codex — план реализации (implementer)

> ВАЖНО: SendMessage у меня отключён, текстовые ответы до тимлида не доходят.
> Кладу план сюда, в общий 00-Inbox, как durable-артефакт. Правок в коде НЕ начинал — жду «ок».

## Разведка (факты из кода worktree)
- `eval/backends.py` — порт `SearchBackend`(Protocol) + `EvalHit(doc_id, score, raw)` + фабрика `get_backend(EVAL_BACKEND)` (inprocess|knowledge).
- `eval/runner.py::evaluate` — метрики по `EvalHit.doc_id`; `_retrieved_doc_ids` дедупит по документу. **Не трогаю.**
- `rag/embedder.py` — порт `Embedder` + `get_embedder()` (bge-m3/1024 через gateway). Переиспользую как `embedding_func`.
- `bff/llm.py::GatewayLLM.complete(messages)` — готовый OpenAI-совместимый chat-клиент к тому же gateway (`CHAT_MODEL`). Переиспользую как `llm_model_func`. Новой модели не ввожу (ADR D8).
- `rag/config.py::Settings` — всё из env. `rag/requirements.txt` — основная сборка.
- lightrag-hku **не установлен** в worktree → импорт делаю опциональным (graceful).

## Файлы/символы (порядок = порядок коммитов)
1. **`requirements-lightrag.txt`** (новый, отдельно от основной сборки): `lightrag-hku`, `neo4j`.
2. **`rag/config.py`** — блок в `Settings` (только env): `lightrag_enabled`, `neo4j_uri/username/password/database`, `lightrag_working_dir`, `lightrag_workspace` (=`default_tenant`, изоляция), `lightrag_query_mode` (mix), `lightrag_vector_storage` (QdrantVectorDBStorage), `lightrag_qdrant_collection` (**отдельная**, дефолт `lightrag__zu`, НЕ боевая `knowledge__bge-m3_1024`), `lightrag_top_k`, `lightrag_chunk_top_k`. Секрет neo4j pwd — только env.
3. **`rag/lightrag_index.py`** (новый): опциональный импорт LightRAG **в теле функций**. `build_lightrag(tenant_id)` → `LightRAG(working_dir, llm_model_func=обёртка GatewayLLM.complete, embedding_func=обёртка Embedder+EmbeddingFunc(embedding_dim=1024), graph_storage="Neo4JStorage", vector_storage=..., workspace=tenant_id)` + `await initialize_storages()`. `ingest_text(rag, text, source_doc_id)` → `ainsert(text, file_paths=[source_doc_id])`.
4. **`eval/backends.py`** — `LightRAGBackend(name="lightrag")`: `search()` оборачивает async (`build_lightrag`→`initialize_storages`→`aquery(QueryParam(mode, only_need_context=True, top_k, chunk_top_k))`) → парс контекста → `EvalHit(doc_id=source_doc_id, raw=чанк)`. fail-closed: пустой tenant → `[]`. Ветка в `get_backend` (`EVAL_BACKEND=lightrag`), креды из env.
5. **`.env.example`** — блок LightRAG-переменных (без секретов).

## Маппинг retrieved→source_doc_id (главный риск)
`ainsert(text, file_paths=[source_doc_id])` проставляет `file_path` каждому чанку; при `only_need_context=True` секция «Document Chunks» контекста несёт это поле. Реализую **дефенсивный парсер** (LightRAG отдаёт контекст структурированной строкой — JSON/маркированные секции, зависит от версии), извлекаю `file_path`=`source_doc_id`, сырой контекст в `EvalHit.raw`. runner дедупит по документу → recall@k корректен, прямое сравнение с baseline 0.967. Точный формат `only_need_context` валидирую против живого пакета на сервере.

## Опциональный импорт
Только внутри функций `rag/lightrag_index.py` и метода `search` `LightRAGBackend` (не на уровне модуля) → отсутствие `lightrag-hku` не ломает существующий semantic-путь (дефолт inprocess).

## Коммиты
По одному на пункт 1..5 в `migration/lightrag`, **без push/merge**, символьными правками Serena.

## Серверное (локально НЕ делаю)
Живой Neo4j zu_demo, ingest 70 докам через gateway-LLM, полный замер vs 0.967, боевой Qdrant, валидация формата `only_need_context` под версией lightrag-hku.

## Вопросы к тимлиду
1. Ок ли дефолт `lightrag_vector_storage=QdrantVectorDBStorage` и имя отдельной коллекции `lightrag__zu`?
2. Даёшь «ок» на реализацию?