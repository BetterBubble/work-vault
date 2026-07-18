---
title: План интеграции RAG в helm (helm-стиль)
type: note
permalink: tacticum/02-architecture/plan-integratsii-rag-v-helm-helm-stil
tags:
- iva
- rag
- helm
- интеграция
- план
- roadmap
---

## Решение (2026-07-13)
Всё живёт на **helm** (репа + сервер). Интеграция наших RAG в **helm-стиле** (порты+DDD как у ассистента требований), переиспользуя готовую helm-инфру. Ветка `feat/iva-rag-knowledge` от origin/main. Handoff: `helm/docs/superpowers/specs/2026-07-13-requirements-assistant-handoff.md`.

## Эталон — ассистент требований (RAG#3a, задеплоен)
Поток `POST /api/assistant/ask`: async-роутер (БД async, порты sync) → retrieve_rids (embed bge-m3/Gateway→Qdrant) → load_requirement_records (async БД: Requirement+вердикты) → RequirementsAssistant.ask (render_evidence→TrivaLlm→guardrail). **async↔sync мост:** retrieval+evidence в основном loop, sync ask() через PreloadedSearch+run_in_threadpool. Порты: RequirementSearch/Reranker(объявлен, #32)/IvaLlm. Инфра: Qdrant `10.16.0.19:6333` (коллекция helm_requirements__bge_m3_1024, 1465 точек), triva `172.18.0.1:8790` (bus-factor=1 Сакевич→таймауты), Gateway tacticum/embed batch≤4. Тесты uv/ruff/mypy strict (971+).

## RAG#1 (доки) в helm
- Коллекция `iva_docs__bge_m3_1024`. Корпус `iva-rag1-docs/md/` (2151 .md, frontmatter, docproc не нужен).
- Перенос из `iva-rag/rag/` (готовый, лучший по метрикам): md_chunker (структурный H1-H6+heading-path), md_ingest, search (hybrid+RRF+rerank+cap+fail-closed), reranker, near_dup, query_rewrite, store/fusion/fulltext.
- helm-паттерн: `domain/docs.py` (DocChunk+render_docs_context), `application/docs_assistant.py` (retrieve→rerank→context→generate→guardrail; evidence из чанков, БД-мост НЕ нужен), `infrastructure/docs_assistant/` (DocsVectorStore/DocsSearch/DocsReranker), `ingest/docs_index.py`, router `POST /api/docs/ask`.
- **Meili для доков — ДА** (в отличие от требований): наш baseline golden 1306 hybrid R@5=0.948 vs semantic 0.924, R@1 0.710 vs 0.679 — доки длинные с кодами/API/продуктами, полнотекст помогает. Режим ретрива **per-корпус** (доки=hybrid, требования=semantic).

## RAG#2 (wiki/jira) — отдельный on-prem сервис
Наш `iva-rag2/`: ingest_jira (8119 задач), граф 2b (build_graph+retrieval, networkx, инфра не нужна), router (index/live/hybrid+structural), live_mcp (порт MCPTransport→iva-mcp), mcp_tool (морда rag2_search/context). В helm тянуть не обязательно — helm=реестр+пресейл-морда; RAG#2 держать отдельным on-prem, но за ЕДИНЫМ портом search(query,scope) для будущей платформы. Пробелы: тела Confluence (заглушка), live-MCP боевой транспорт, golden RAG#2 мал.

## Наши задачи из handoff
- **#32 reranker:** порт Reranker объявлен в helm; наш `iva-rag/rag/reranker.py::GatewayReranker` (tacticum/rerank, Cohere-совм.) переносится 1:1, fallback Noop→records[:8]. Для требований И доков. Самый дешёвый прирост.
- **#31 приём:** домен готов (parse_requirement_draft/build_coverage_map); эндпоинты /intake/{structure,coverage,register}; register через create_requirement (cio.py). Риск: Gateway прокидывает ли guided_json во vLLM.
- **#34 ADR-0003:** контур (RAG#1 публичный / RAG#2#3 on-prem) — из нашего концепта §1.2 → docs/adr/0003-*.

## Три морды (голосовое рук-ва)
1. helm чат-бот — RAG#1(доки) `/api/docs/ask` + RAG#3(требования) `/api/assistant/ask`. Фронт Assistant.tsx (Лебедев #30).
2. MCP-инструмент аналитикам (phk-ключ) — RAG#2 rag2_search/context, on-prem.
3. Встроенный в Evo One — RAG#1(доки) виджет за той же search-мордой.

## Развилки/риски
- Форк-код↔helm-стиль: переносим ПАТТЕРН+порты, тело алгоритмов — наш код адаптированный (pydantic Settings, mypy strict, русские комменты; httpx-REST как helm или разрешить qdrant-client). Дисциплина: морды за портом `search(query,scope)→hits`.
- async-мост: для доков НЕ нужен (чанки самодостаточны); для требований остаётся.
- triva bus-factor=1 → таймауты+graceful degradation везде.

## Порядок реализации
1. ✅ Синхрон репы (feat-ветка). 2. RAG#1(доки) в helm + #32 reranker (первый, показуемый). 3. #31 приём. 4. #34 ADR. 5. RAG#2 (on-prem, отдельно). 6. Тесты→коммит→пуш(с подтверждением)→проверка на сервере.

## Отношения
- part_of [[Концепт: три RAG для ИВА на общем движке]]
- relates_to [[RAG#1 морда собрана по коду (iva-rag)]]
- relates_to [[RAG#2 — дизайн: оффлайн-индекс + live-MCP подгрузка]]
