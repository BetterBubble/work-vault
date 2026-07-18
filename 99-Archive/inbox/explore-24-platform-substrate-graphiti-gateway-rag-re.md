---
title: explore-24-platform-substrate-graphiti-gateway-rag-re
type: report
permalink: tacticum/00-inbox/explore-24-platform-substrate-graphiti-gateway-rag-re
tags:
- helm
- explore
- platform
- graphiti
- rag
- re
- tenant-isolation
status: archived
updated: 2026-07-18
---

# Разведка #24 — субстрат platform (Graphiti/Gateway/knowledge_rag/RE)

Черновик explorer для ADR поворота Helm. Каноническую запись делает лид. Факты — из кода `/Users/bubblemac/tacticum/{platform,rag_eval_service,KB-Brownfield-Bootstrap}`.

## Таблица: компонент → connect|build → контракт → изоляция → готовность

| Компонент | connect/build | Точка входа / контракт | Тенант-изоляция | Готовность |
|---|---|---|---|---|
| **Graphiti** | vendored lib + connect | lib `platform/lib/graphiti` (`tacticum_graphiti`: GraphitiClient over `graphiti_core`); сервис `services/memory` MCP `memory.{write,read,list_by_type}` | **group_id namespace** `mem:{tenant}:u/p/shared` на ОБЩЕМ FalkorDB; fail-closed by construction (tenant_id обязателен, sanitize) | lib+scope готовы; write/read L0 wired; L1/L2+list — заглушки |
| **Backend графа** | connect | **FalkorDB** (redis://falkordb:6379, RediSearch) — НЕ Neo4j/AGE/Postgres | — | работает |
| **LLM Gateway** | adopt/connect | `services/ai/llm_gateway` = **LiteLLM** (OpenAI-compat, «единственный шов»); base_url+key (project-hub-issued LLM service key) | ключ/бюджет по тенанту | running (adopt) |
| **knowledge_rag** | build (extract из agents) | `services/data/knowledge_rag` MCP `knowledge.{search,ingest,list_collections}`; **Qdrant + Meilisearch** hybrid | **логическая**: общие коллекции + обязательный `tenant_id` payload-фильтр, fail-closed (ScopedRepository); физич. — опция под ADR | Слайсы 1-5 ✅, 35 тестов, **deploy-ready** (нужен Qdrant+Meili+Gateway+hub) |
| **Эмбеддинги/реранк** | connect | через Gateway: `tacticum/embed` = **bge-m3/1024**, `tacticum/rerank` = bge-reranker-v2-m3 (TEI) | — | канон D6; rerank-форма LiteLLM — подтвердить curl |
| **project-hub (IdP)** | connect | `POST /api/internal/resolve` (service key + user token `phk_…`) → ResolvedIdentity(tenant_id…); ИЛИ trusted headers X-Tenant-Id/X-Auth-User-Id за MCP Runtime gateway (ADR-0007) | источник tenant_id (не от клиента) | используется memory+knowledge |
| **RE (KB-Brownfield)** | connect (producer) | `tacticum_re` DDD; CLI typer (`run`, topology/proposal), phases p1..p11, phase-cache; artifacts | по проекту/снапшоту | зрелый конвейер, много артефактов |
| **LightRAG** | — | **НЕ найден** как библиотека (grep пуст в platform+rag_eval). Есть `rag_eval_service/rag` — кастомный hybrid RAG (store/ingest/chunker/fusion/search/embedder/fulltext/routing) — eval-харнес, прототип под knowledge_rag | — | прототип/eval |

## Детали по пунктам

### 1. Graphiti
- **Библиотека** `tacticum_graphiti` (тонкая обёртка `graphiti_core.Graphiti`, factory build_graphiti). Узлы = entity nodes (L0 summary), рёбра = facts (L1/L2, bi-temporal valid_at/invalid_at, episode provenance). API: `add_episode`, `search_nodes` (L0), `search_facts` (L1/L2). Degraded: NullGraphitiClient.
- **Backend = FalkorDB** (Redis graph). НЕ Neo4j, НЕ Postgres/AGE.
- **Изоляция = group_id**: `scope.py` — `mem:{tenant_id}:u:{user}` / `:p:{project}` / `:shared`. tenant-префикс = жёсткая граница, «cross-tenant impossible by construction, fail-closed» (ADR-0005 D4). Read fan-out user→project→shared, всё под tenant-префиксом. group_id санитайзится (FalkorDB RediSearch токенизирует по `-`/`:`).
- **Подключение Helm**: MCP-тулы `memory.*` через MCP Runtime; identity — из project-hub resolve (token) ИЛИ trusted-headers за gateway. Общий FalkorDB, свой group_id-namespace (`mem:`).

### 2. LLM Gateway
LiteLLM (adopt, ADR-0001 D3), OpenAI-совместимый, backend сменный по профилю (суверенитет). Поддержка: chat (llm_model, дефолт gpt-4o-mini), embeddings (bge-m3), rerank (`/rerank`). Всё — с project-hub-issued ключом, без прямых провайдер-ключей. Для Helm: lead_time/скоринг/dedup-judge → chat; эмбеддинги для dedup → embed-тир.

### 3. knowledge_rag / векторка
`services/data/knowledge_rag` (owned, extract из agents). Qdrant (vector) + Meilisearch (fulltext), hybrid (union по умолчанию, RRF за флагом), + rerank. Контракт `knowledge.search/ingest/list_collections` в `platform/contracts` + SDK. Изоляция логическая (tenant_id payload, fail-closed), физическая — опция под отдельный ADR (КИИ/банк on-prem). Коллекции по embedding-space (`knowledge__bge-m3_1024`). В проде ≥3 вектор-стора (долг на консолидацию). Deploy-ready.

### 4. LightRAG
Как отдельной библиотеки LightRAG в репо **не нашёл** (grep `lightrag` пуст). `rag_eval_service` = eval-сервис (bff/frontend/rag/eval); `rag/` — рукописный hybrid (fusion.py/routing.py/search.py) — прототип, легший в дизайн ADR-0005/knowledge_rag. GraphRAG упоминается только как Phase 2 концепт в ADR (adopt arch-mcp как GraphStore). → Термин канона «LightRAG» скорее собирательный к этому прототипу, не к OSS-либе. (Требует уточнения у автора канона.)

### 5. RE-компоненты (KB-Brownfield-Bootstrap `tacticum_re`)
DDD: `phases/` (p1_ingest, p7_code_scenario_match, p9_business_context, p10/p10a/p10b_cjm, p11_review_enrich), `artifacts/` (use_case, topology+topology_compact, sbom_compact, behaviour, business_scenarios, business_context, adr, components, cjm, api_index, code_signatures), `adapters/` (snapshot/repomix, code_index/{serena,scip,composite}, sbom, vcs, vector_db/**chroma_store**, llm, docs, contracts, authorization), `services/{topology_profiles, code_block_classifier}`, `app/{pipeline_runner,onboard,doctor}`, `cli/main.py` (typer). Запуск: CLI `run` + раннеры `run_p7/p10b/p11_only.py`, phase-cache (`docs/phase-cache-plan.md`, событийный прогресс по фазам). Вектор RE = **Chroma** (ADR-0005 Phase 2: Chroma→Qdrant, RE как producer через `knowledge.ingest`).
- Подтверждено наличие: topology (+serena-адаптер code_index), use_case, sbom (+классификаторы code_block_classifier), profile (services/topology_profiles), pipeline/phases.
- **semantic_dedup** как отдельный именованный модуль по имени НЕ подтверждён (есть dedup-логика в compact_packer/classifier). Требует уточнения по §8.2.

## Рекомендация по Graphiti (для ADR)

**Дефолт — connect к platform-memory** (group_id), НЕ строить свой Postgres+AGE:
- group_id-изоляция здесь НЕ «слабая»: структурная, fail-closed by construction (tenant_id обязателен, санитайз, префикс `mem:{tenant}` — кросс-тенант невозможен по построению), симметрична knowledge_rag (tenant_id payload). Та же строгость, что канон принял платформенно (ADR-0005 D4).
- Плюсы connect: общий субстрат, Gateway (суверенные модели), project-hub auth, RE-интеграция, bi-temporal KG из коробки.
- **Оговорка (важно для ИВА)**: это ЛОГИЧЕСКАЯ изоляция на ОБЩЕМ FalkorDB, не physical DB-per-tenant. Если суверенитет данных ИВА требует физической изоляции (КИИ/on-prem) — это предусмотренная ADR-0005 D4 опция «выделенный инстанс под тенант» через тот же `memory`-контракт (изменение ops, не кода Helm).
- Альтернатива Postgres+AGE у себя оправдана ТОЛЬКО если Helm обязан не зависеть от platform. Иначе — против (теряем субстрат/Gateway/RE/temporal, дублируем изоляцию).

**Архитектурная оговорка для декомпозиции**: ядро control-tower (детерминированный граф Signal→Initiative, разрывы §6.1, calc) — реляционное, у Helm уже Postgres. Graphiti/memory уместен для НЕЧЁТКОГО слоя (semantic dedup инициатив, «работа=фигня»-суждения, нарративная память), knowledge_rag — для документов/RE-артефактов. НЕ загонять детерминированный граф в Graphiti — субстраты дополняют, не заменяют Postgres-ядро.
