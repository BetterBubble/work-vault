---
title: plan-lightrag-server-eval
type: note
permalink: tacticum/20-architecture/plan-lightrag-server-eval-1
tags:
- lightrag
- zu_demo
- eval
- plan
- deploy
- postgres
- age
- server
status: archived
updated: 2026-07-18
---

# План: серверная проверка LightRAG на zu_demo

**Статус:** утверждён пользователем 2026-07-01, к исполнению в НОВОЙ сессии (нужны разрешения push/PR, которых текущая сессия не видит). Доставка кода — через чистый push feature-ветки на GitHub → fetch на сервере → при успехе PR.

**Обновление 2026-07-01 (миграция бэкенда):** по обновлённой ADR-0002 (D3) графовый бэкенд переведён с **Neo4j на PostgreSQL + Apache AGE**. Код в ветке `migration/lightrag` переписан и **закоммичен** (2 AGE-коммита поверх f7dfabe: `7334cb2`+`bd38ab2`, историю не переписывали), 22 unit passed. **Живой AGE-smoke уровня A пройден ЛОКАЛЬНО** (docker Postgres+AGE + asyncpg: коннект, граф namespace по tenant, fail-closed) — попутно исправлены 2 бага (том PG18 в /var/lib/postgresql; пин KV/doc-status = файлы). Осталось: уровень B (ingest 70 доков + golden-set) на сервере (Ш4/Ш5) и чистый push.

## Контекст
LightRAG (graph-RAG) интегрирован в codex (rag_eval_service) в ветке **migration/lightrag** (worktree `~/tacticum-worktrees/rag_eval_service-lightrag`). Локально зелёное: 22 unit passed, изоляция fail-closed держит, маппинг extract_doc_ids толерантен под lightrag-hku 1.5.4 (баг пойман/починен в Neo4j-эпоху, f7dfabe). По ADR-0002 (D1/D6) последний этап — честный замер на боевом стенде: поднять граф-слой (Postgres+AGE), построить граф по 70 докам ЗУ, прогнать golden-set (35 кейсов) тем же eval-harness, сравнить с baseline semantic **recall@k=0.967 / MRR=0.860 / nDCG=0.875**. Решение о проде — по числам. Релиз (merge в main) — за руководителем.

Код готов (переписан под AGE). Задача сессии: аккуратная серверная операция + живой smoke AGE, не ломая боевое демо.

## Режим взаимодействия с сервером
Пользователь собирает узкий whitelist (ALLOW_PATTERNS) ровно под команды ниже (включая `sudo -u tacticum-deploy …` и конкретные `docker compose …`), правит `.env` ssh-manager (+бэкап) и переключает zu_demo в **restricted**. Дальше серверную работу выполняет Claude: разрешённое проходит, опасное режется. Если команда отказана политикой — не обходить, сказать пользователю.

## Состояние zu_demo (снято read-only 2026-07-01)
- codex-git на ветке **main** (52525c4) — LightRAG-кода НЕТ (нет lightrag_index.py, requirements-lightrag.txt, docker-compose.lightrag.yml; backends.py = версия main).
- **Postgres на стенде УЖЕ работает** (история чатов) — граф AGE ложится в него (ADR D2/D3), новый сервис не поднимаем. AGE-расширения (`CREATE EXTENSION age`) и lightrag-hku в боевом codex-1 пока нет — ставятся в Ш3/Ш4.
- Qdrant: только боевая коллекция `knowledge__bge-m3_1024`.
- RAM 3.8Gi (2.8 avail), **swap=0** → нужен swap-файл (ADR-0002 D5).
- Golden-set УЖЕ на сервере: `codex-git/eval/golden_sets/zu_golden.json` (+known_gaps) — довозить не надо.
- Ветка migration/lightrag НЕ в origin (только локально+worktree).

## Ключевые файлы ветки
- `rag/lightrag_index.py` — build_lightrag(tenant) (graph=**PGGraphStorage** (Postgres+AGE), vector=Qdrant/отдельная коллекция, workspace=tenant, fail-closed), ingest_documents, extract_doc_ids (толерантный маппинг под 1.5.4 «Reference Document List» — от графового движка НЕ зависит). `_bridge_env_from_settings()` пробрасывает `POSTGRES_*` в os.environ.
- `eval/backends.py` — LightRAGBackend(SearchBackend) (only_need_context=True, персистентный event-loop — **asyncpg-пул привязан к loop**, fail-closed tenant→[]) + регистрация в get_backend("lightrag"). Ядро runner.py/metrics.py не тронуто.
- `rag/config.py` — env-блок LightRAG + **POSTGRES_HOST/PORT/USER/PASSWORD/DATABASE** + `LIGHTRAG_GRAPH_STORAGE` (дефолт PGGraphStorage, де-хардкод).
- `requirements-lightrag.txt` — lightrag-hku>=1.4, **asyncpg>=0.29** (вместо neo4j), openai>=1, numpy>=1.24.
- `docker-compose.lightrag.yml` — dev-only **gzdaniel/postgres-for-rag:pg18-age-pgvector** (AGE+pgvector предустановлены; используем только AGE, векторы в Qdrant), порт 5432, volume lightrag_pg_data, креды rag/testpassword → на сервере переопределить секретом.

### Env LightRAG
LIGHTRAG_ENABLED(false), **POSTGRES_HOST(localhost), POSTGRES_PORT(5432), POSTGRES_USER(rag), POSTGRES_PASSWORD(секрет, обязателен), POSTGRES_DATABASE(lightrag)**, LIGHTRAG_WORKING_DIR(./.lightrag — kv/doc-status файлами), LIGHTRAG_WORKSPACE(=tenant), **LIGHTRAG_GRAPH_STORAGE(PGGraphStorage)**, LIGHTRAG_QUERY_MODE(mix), LIGHTRAG_VECTOR_STORAGE(QdrantVectorDBStorage), LIGHTRAG_QDRANT_COLLECTION(отдельный префикс, НЕ knowledge__bge-m3_1024), LIGHTRAG_TOP_K(60), LIGHTRAG_CHUNK_TOP_K(20).

## План работ
- **Ш0. Read-only baseline** — зафиксировать состояние; пересчитать baseline semantic на сервере (`eval.runner --backend inprocess`) под тем же tenant для честной дельты.
- **Ш1. Доставка кода** — сперва решить судьбу 5 Neo4j-коммитов + незакоммиченных AGE-правок (переработать/сквошнуть, чтобы ветка была под AGE, а не Neo4j+патч). Затем чистый push migration/lightrag на GitHub (правила push из CLAUDE.md: только текущую ветку явно, НЕ main, показать git status + log origin/main..HEAD + diff --stat ДО пуша, проверить что не уходят .env/секреты/**.venv312**/.serena/__pycache__/worktree-мусор) → на сервере `sudo -u tacticum-deploy git -C …/codex-git fetch origin` + `checkout migration/lightrag`.
- **Ш2. Swap-файл** (страховка OOM при индексации, ADR-0002 D5) — под root: fallocate 2G→chmod 600→mkswap→swapon.
- **Ш3. PostgreSQL + Apache AGE** — граф в **уже работающем Postgres** стенда (ADR D2/D3, НОВЫЙ сервис не поднимаем): `CREATE EXTENSION age` в целевой БД (или отдельная БД lightrag), search_path с ag_catalog. Секрет POSTGRES_PASSWORD — через env/override, НЕ боевой .env. Для dev-смоука есть `docker compose -f …/codex-git/docker-compose.lightrag.yml up -d` (образ postgres-for-rag с AGE). Проверить подключение asyncpg.
- **Ш4. Изолированное eval-окружение** (НЕ трогать боевой codex-1) — эфемерный контейнер с примонтированным codex-git, pip install requirements-lightrag.txt (**asyncpg вместо neo4j**), env (Postgres+AGE/Qdrant/Gateway/tenant, LIGHTRAG_ENABLED=1, LIGHTRAG_GRAPH_STORAGE=PGGraphStorage). Внутри `pytest eval/tests -q` (санити на боевой версии lightrag-hku) + **живой smoke на AGE**: подтвердить, что LightRAG принимает graph_storage=PGGraphStorage, initialize_storages() коннектится к AGE, workspace=tenant изолирует, и формат aquery(only_need_context=True) сходится с extract_doc_ids. Боевой образ НЕ пересобираем.
- **Ш5. Построить граф по 70 докам** — LightRAG ingest (LLM-извлечение через боевой Gateway, O1/O2). Мониторить RAM/swap на пике. Граф → Postgres+AGE + отдельная Qdrant-коллекция lightrag_*; боевую knowledge__bge-m3_1024 не трогать (read-only).
- **Ш6. Валидация+прогон** — `eval.validate --check-index`, затем `python -m eval.runner --golden eval/golden_sets/zu_golden.json -k 5 --mode mix --backend lightrag --out runs/lightrag.json`. Срезы difficulty/source.
- **Ш7. Сравнение** — таблица LightRAG vs baseline 0.967 (recall@k/MRR/nDCG + «где граф выигрывает»). Итог — в чекпойнт.
- **Ш8. Teardown при нулевой дельте** — граф-данные AGE снести (DROP graph в ag_catalog / отдельную БД lightrag), вернуть codex-git на main. Merge/PR — осознанный шаг пользователя.

## Открытые пункты
- **tenant для замера**: golden помечен `cifragen`, дефолт в коде `zu` — сверить, под каким tenant проиндексированы 70 доков, гонять eval и LightRAG-workspace под ним же.
- **Postgres-пароль (POSTGRES_PASSWORD)**: сгенерировать секрет, только в env/override, не в боевой .env. **AGE в боевой БД**: решить — целевая БД истории чатов + отдельная graph-структура через workspace, ИЛИ отдельная БД `lightrag` в том же инстансе (чище для teardown).
- **PR**: gh не установлен намеренно — Claude готовит заголовок+описание+ссылку, PR создаёт пользователь через веб. Push ≠ merge.

## Подводные камни сервера (из [[server-zu-demo]])
Два дерева кода (боевое /home/tacticum-deploy/codex-git, /root/codex мёртвое); git через sudo -u tacticum-deploy; docker compose из /root/zu-deploy или с явным -f; код собирается в образ (COPY, не монтируется) → eval в изолированном окружении; .env не трогать/не печатать, секреты только env; системное (swap) под root.

## Верификация
1. pytest eval/tests -q в изолированном окружении на боевой версии lightrag-hku — зелёное.
2. Живой AGE-smoke: LightRAG строится с PGGraphStorage, коннект к Postgres/AGE есть, fail-closed чужой/пустой tenant → [] на живом графе.
3. eval.validate --check-index — golden сходится с индексом.
4. eval.runner --backend lightrag даёт непустые ранжированные doc_id + таблицу метрик со срезами.
5. Дельта vs baseline 0.967 посчитана → вывод «граф выигрывает / нулевая дельта».
6. Боевое демо не задето: knowledge__bge-m3_1024 неизменна, история чатов в Postgres не задета, zu-deploy-codex-1 не пересобирался, /ask работает.

## Relations
- part_of [[20-Architecture]]
- relates_to [[lightrag-в-codex]]
- relates_to [[ADR-0002 — LightRAG (graph-RAG) для ЗУ]]
- relates_to [[lightrag-codex-checkpoint-2026-07-01]]
- relates_to [[server-zu-demo]]
- relates_to [[Runbook: прогон eval на сервере]]