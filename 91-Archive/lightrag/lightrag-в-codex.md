---
title: lightrag-в-codex
type: note
permalink: tacticum/91-archive/lightrag/lightrag-v-codex-1
tags:
- codex
- lightrag
- graph-rag
- neo4j
- zu
- task
- home
status: archived
updated: 2026-07-18
---

# LightRAG в Codex — дом задачи

**Тип:** точка входа в задачу. Статус: активная.

## Цель
Добавить **LightRAG** (graph-RAG) в `codex` (`rag_eval_service`) и **протестировать на сервере** `zu_demo` — развернуть граф-слой для честного сравнения по данным с текущим semantic RAG (baseline recall@k=0.967), решение о проде — по метрикам, не авансом.

Решение зафиксировано в [[0002-zu-lightrag-graph-rag]] (Proposed, развёртывание одобрено руководителем 2026-07-01). Пересматривает прежнее [[lightrag-overkill-for-zus]] (условие «клиент явно попросит» сработало).

## Каркас плана (болванка, уточняется)
1. **Инфра на `zu_demo`:** поднять **Neo4j** отдельным сервисом рядом с демо (ADR-0002 D3/D5); добавить swap-файл (swap=0 → страховка от OOM при индексации); мониторить RAM на пике индексации.
2. **Интеграция LightRAG в codex:** подключить `lightrag-hku`; вектор-слой → **существующий Qdrant ЗУ** (D4), граф → Neo4j; эмбеддинги — тот же bge-m3/1024 (проверить совместимость с коллекцией `knowledge__bge-m3_1024`, O3).
3. **Построение графа:** индексация по **всем 70 документам** корпуса ЗУ (граф извлекается вызовами LLM через Gateway — следить за нагрузкой O1, ПДн-блокер O2 для прода).
4. **Замер (методика D6):** прогнать выверенный golden-set (35 кейсов) через LightRAG тем же eval-harness (см. [[Runbook: прогон eval на сервере]]), метрики recall@k/MRR/nDCG, срезы difficulty/source.
5. **Сравнение и решение:** таблица «где граф выигрывает vs semantic 0.967»; при подтверждённом приросте — путь в прод, иначе — откат Neo4j, система остаётся простой.

## Инварианты
Работающее демо не трогаем (изолированный сервис). Read-only к боевому индексу при замере. Секреты только из env. Тенант строго cifragen/zu (fail-closed). Клиентские данные не покидают стенд.

## Ключевые ссылки
- Решения: [[0002-zu-lightrag-graph-rag]], [[ADR-0001 — Демо-стенд RAG ЗУ (Codex)]]
- Прогон/метрики: [[Runbook: прогон eval на сервере]], [[Руководство: составление golden-set]]
- Данные: [[Корпус ЗУ (КЦ папка) — индекс]], [[Golden-set ЗУ — выверка по истине]]
- Термины: [[glossary]]

## Relations
- part_of [[20-Architecture]]
- relates_to [[0002-zu-lightrag-graph-rag]]
- relates_to [[ADR-0001 — Демо-стенд RAG ЗУ (Codex)]]
- relates_to [[Runbook: прогон eval на сервере]]
- relates_to [[Корпус ЗУ (КЦ папка) — индекс]]
- relates_to [[glossary]]

## Ход реализации (2026-07-01)

Команда Agent Team: тимлид + implementer2 (тип claude — у роли implementer нет SendMessage, переподняли) + verifier. Правки — в git-worktree `~/tacticum-worktrees/rag_eval_service-lightrag`, ветка **`migration/lightrag`** (без push/merge).

**Сделано (закоммичено, 4 коммита, +448 строк / 11 файлов):**
- `rag/config.py` — env-блок LightRAG (LIGHTRAG_ENABLED, NEO4J_URI/USERNAME/PASSWORD/DATABASE, LIGHTRAG_WORKING_DIR, LIGHTRAG_WORKSPACE=tenant, LIGHTRAG_QUERY_MODE=mix, LIGHTRAG_VECTOR_STORAGE, LIGHTRAG_QDRANT_COLLECTION=отдельная, TOP_K/CHUNK_TOP_K). Граф-LLM = `chat_model` через gateway, эмбеддинги = codex Embedder (bge-m3/1024) — новой модели нет.
- `requirements-lightrag.txt` (lightrag-hku+neo4j+openai+numpy) — отдельно от основной сборки. `.env.example`, `.gitignore` (.lightrag/), dev `docker-compose.lightrag.yml` (neo4j:5-community).
- `rag/lightrag_index.py` (новый) — `build_lightrag(tenant)` (graph=Neo4JStorage, vector=тот же Qdrant/отдельная коллекция, workspace=tenant, fail-closed пустой tenant→ValueError, graceful-импорт), `ingest_documents` (ainsert file_paths=source_doc_id), `extract_doc_ids()` — толерантный маппинг retrieved→doc_id.
- `eval/backends.py` — `LightRAGBackend(SearchBackend)` name="lightrag" (персистентный event-loop под async Neo4j, only_need_context=True, fail-closed tenant→[], rerank→NotImplementedError), регистрация в `get_backend` (EVAL_BACKEND=lightrag). **Ядро `runner.py` не тронуто.**
- `eval/tests/` (conftest+__init__) + корневой `pytest.ini` (testpaths=eval/tests, изолирован от document_processing/pytest.ini) + smoke.

**Коммиты:** 284a62f (deps+env+compose), 87737ef (lightrag_index), c801a21 (LightRAGBackend), 867b2a0 (pytest-скэффолд).

**Локально проверено (smoke-импортом, у implementer2 нет pytest/lightrag-hku в system python):** extract_doc_ids корректно ранжирует+дедупит; fail-closed (пустой tenant→ValueError/[]); get_backend роутинг цел; semantic-дефолт не сломан. Полный pytest+изоляция+docker-Neo4j — на verifier.

**Инварианты соблюдены:** боевая Qdrant-коллекция `knowledge__bge-m3_1024` не затронута (LightRAG ведёт свои коллекции по workspace-префиксу), semantic-дефолт цел, секреты только env.

Статус: интеграция в ветке, идёт локальная проверка verifier'ом. Конфликтов с ADR-0002 нет.

## Статус: ГОТОВО К СЕРВЕРНОМУ ТЕСТУ (2026-07-01)

Интеграция завершена и локально проверена. Ветка **`migration/lightrag`** (без push/merge): 5 коммитов, 11 файлов, +546 строк.
Коммиты: 284a62f (deps+env+compose), 87737ef (lightrag_index), c801a21 (LightRAGBackend), 867b2a0 (pytest-скэффолд), **f7dfabe (fix маппинга под lightrag-hku 1.5.4 + закрытие event-loop)**.

**Локальная проверка (verifier, venv+pytest, живой docker-Neo4j):**
- Юниты `pytest eval/tests` — **22 passed** (10 smoke + 12 verifier), event-loop warning ушёл.
- Изоляция fail-closed — **держит** (пустой tenant→[], build("")→ValueError, per-tenant графы, живой workspace B→пусто).
- `eval.validate --check-schema` — PASS. Регресс semantic — без изменений (ядро runner/metrics/search/store байт-в-байт, backends +additive).
- docker-Neo4j смоук — **PASS**: ingest→`aquery(only_need_context=True)`→`extract_doc_ids` даёт непустой ранжированный doc_id под своим tenant, пусто под чужим.

**Пойманный и починенный баг (ценность локального смоука):** первый `extract_doc_ids` грепал `file_path`, а lightrag-hku 1.5.4 отдаёт чанки с `reference_id` + секцию «Reference Document List» (`[N] <doc_id>`) → recall@k был бы 0 на сервере (выглядело бы как провал vs 0.967, хотя это баг маппинга). Фикс f7dfabe: парсинг ref-списка, ранг по релевантности чанков (reference_id), file_path — fallback.

Чекпойнт и список серверных шагов: [[lightrag-codex-checkpoint-2026-07-01]]. Конфликтов с ADR-0002 нет. Релиз (merge/push) — за руководителем, командой не делаем.