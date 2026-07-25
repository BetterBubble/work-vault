---
title: lightrag-codex-checkpoint-2026-07-01
type: note
permalink: tacticum/01-sessions/lightrag-codex-checkpoint-2026-07-01-1
tags:
- session
- checkpoint
- lightrag
- codex
- migration
- server-todo
- postgres
- age
status: archived
updated: 2026-07-18
---

# Чекпойнт: LightRAG в codex — граф-бэкенд переведён на Postgres+AGE, готово к серверному тесту (2026-07-01)

Работа Agent Team по внедрению LightRAG в codex. Доведено до «код готов + переписан под новый бэкенд», НЕ дальше (по границе задачи). Релиз (merge/push) — за руководителем.

**Обновление (миграция бэкенда, 2026-07-01):** по обновлённой ADR-0002 (D3) граф переведён с **Neo4j на PostgreSQL + Apache AGE**. Причины: мультитенантность штатным Postgres (наследуется AGE), без нового сервиса (граф в уже работающем Postgres), on-prem (`CREATE EXTENSION age`). Векторы остались в Qdrant (D4), KV+doc-status — файлами в working_dir.

## Где всё лежит
- Ветка **`migration/lightrag`** в worktree `~/tacticum-worktrees/rag_eval_service-lightrag`. **Не запушена, не смёржена.**
- **7 коммитов**: 5 Neo4j-эпохи (до `f7dfabe`) + 2 сверху под AGE (`7334cb2` refactor Neo4j→Postgres+AGE, `bd38ab2` verifier-тесты). Историю НЕ переписывали (решение пользователя). HEAD `bd38ab2`, working tree чист.
- Push-превью (origin/main..HEAD): 12 файлов, +745, без .env/секретов/.venv312/.serena.
- Дом задачи: [[lightrag-в-codex]]. Решение: [[0002-zu-lightrag-graph-rag]]. План сервера: [[plan-lightrag-server-eval]].

## Что в ветке (после миграции)
- `rag/config.py` — env-блок LightRAG + **POSTGRES_*** (host/port/user/password/database) + `LIGHTRAG_GRAPH_STORAGE` (дефолт PGGraphStorage); отдельная Qdrant-коллекция, граф-LLM=chat_model, эмбеддинги=bge-m3.
- `rag/lightrag_index.py` — build_lightrag/ingest/extract_doc_ids, fail-closed, graceful-импорт; `_bridge_env_from_settings()` пробрасывает POSTGRES_*; `graph_storage=settings.lightrag_graph_storage`.
- `eval/backends.py` — LightRAGBackend + регистрация в get_backend (EVAL_BACKEND=lightrag); event-loop rationale = asyncpg-пул. Ядро runner.py не тронуто.
- `eval/tests/` + `pytest.ini` + smoke; `requirements-lightrag.txt` (**asyncpg>=0.29** вместо neo4j); `.env.example` (POSTGRES_*); `.gitignore`; dev `docker-compose.lightrag.yml` (**gzdaniel/postgres-for-rag:pg18-age-pgvector**, порт 5432).

## Локально пройдено ✅
- **22 юнита passed** (после миграции), изоляция fail-closed держит, регресс semantic без изменений.
- grep-чисто от `neo4j/bolt://Neo4JStorage` в нашем коде.
- **Живой AGE-smoke уровня A — PASS** (локально: docker Postgres+AGE `gzdaniel/postgres-for-rag:pg18-age-pgvector` + asyncpg): `build_lightrag(PGGraphStorage)` → `initialize_storages` коннектится к AGE, граф с namespace по tenant (`graph_name=<tenant>_chunk_entity_relation`, «AGE extension enabled»), openCypher `get_all_labels()`→`[]`, fail-closed пустой tenant, `finalize` чисто.
- Smoke поймал **2 бага, оба исправлены**: (1) docker-compose — том PG18 монтируется в `/var/lib/postgresql` (не `/data`), иначе контейнер падает; (2) `build_lightrag` не пинил KV/doc-status → закреплены **файлами явно** (`LIGHTRAG_KV_STORAGE=JsonKVStorage`, `LIGHTRAG_DOC_STATUS_STORAGE=JsonDocStatusStorage`).
- ⚠️ Уровень B (ingest 70 доков + aquery + golden-set) — только на сервере (нужен gateway + данные).
- Следствие для сервера: PGGraphStorage при init бутстрапит пустые `LIGHTRAG_*`-таблицы общим PG-клиентом → держать граф в **ОТДЕЛЬНОЙ БД `lightrag`**, чтобы не смешивать с БД истории чатов.

## ОСТАЛОСЬ НА СЕРВЕРЕ (zu_demo, следующий этап)
1. **PostgreSQL + Apache AGE** — граф в уже работающем Postgres стенда (`CREATE EXTENSION age`, отдельная БД `lightrag` или graph-структура через workspace), + swap-файл (swap=0 → страховка OOM при индексации, ADR-0002 D5). Новый сервис не поднимаем.
2. **Установить `lightrag-hku` + `asyncpg`** (requirements-lightrag.txt) в изолированное окружение; сверить фактический формат `aquery(only_need_context=True)` с extract_doc_ids **на живом AGE**.
3. **Построить граф по 70 докам** корпуса ЗУ через боевой gateway-LLM — следить за нагрузкой (O1), ПДн-блокер для прода (O2, локальный LLM).
4. **Отдельная Qdrant-коллекция** для LightRAG (не трогать боевую `knowledge__bge-m3_1024`); совместимость (O3, `eval.validate --check-index`).
5. **Полный замер** `python -m eval.runner --backend lightrag` (recall@k/MRR/nDCG) + Δ vs baseline semantic **0.967**, срезы по difficulty/source. Решение о проде граф-слоя — по этим числам (ADR-0002 D1/D6).
6. Мониторинг RAM/swap при индексации.

## Relations
- part_of [[01-Sessions]]
- relates_to [[lightrag-в-codex]]
- relates_to [[0002-zu-lightrag-graph-rag]]
- relates_to [[plan-lightrag-server-eval]]
- relates_to [[Runbook: прогон eval на сервере]]
- relates_to [[Корпус ЗУ (КЦ папка) — индекс]]