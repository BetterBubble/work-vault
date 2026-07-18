---
title: glossary
type: note
permalink: tacticum/02-architecture/glossary
tags:
- glossary
- transcription
- after-call
- architecture
---

# Глоссарий для разбора транскриптов

Назначение: канонические написания терминов, имён и проектов. При разборе транскрипта созвона (/after-call) исправлять искажения распознавания на эти формы.

## Имена коллег (Whisper их особенно коверкает)
- Дмитрий Лебедев
- Александр Шульга (это я, автор)
- Дмитрий Евстигнеев
- Дмитрий Солонко
- Иван Монахов
- Дмитрий (Дима) Паршаков — принимает Залог Успеха на кик-оффе
- Василенко (тех-лид клиента «Залог Успеха») — предложил LightRAG — уточнить имя
- Евгений Воронцов
- Ганцев (фамилия) — уточнить имя
- Бахтир (админ, сброс доступов) — уточнить написание (возможно Бахтиёр)

## Сервисы и технологии
- Qdrant (векторная БД) — варианты искажений: "кудрант", "кьюдрант"
- Meilisearch / Meili (полнотекстовый поиск)
- bge-m3 (модель эмбеддингов, 1024-dim) — искажения: "бэ-же-эм-три", "беджи эм три"
- reranker / RU-реранкер bge-reranker-v2-m3
- RRF (Reciprocal Rank Fusion) — искажения: "эр-эр-эф"
- knowledge_rag / knowledge (платформенный модуль hybrid search)
- LightRAG (graph-based RAG) — искажения: «лайтрак», «лайтрага», «LightRack», «Lightrack», «лайтага»
- Neo4j (графовая БД, нативный бэкенд LightRAG для графа сущностей/связей) — искажения: «неофорджей», «нео фор джей», «ниао4джей»
- document_processing (извлечение/OCR)
- LLM Gateway (единый шлюз к моделям)
- Langfuse (observability/трейсинг LLM) — искажения: «кланк фьюз», «клик фьюз», «лангфьюз»
- tei / tei_service (on-prem эмбеддинги)
- Eval Harness (golden-set, метрики recall@k/MRR/nDCG)
- fail-closed scope_filter, tenant_id, AuthScope (Tenant→Project→User)
- Project Hub, MCP, ADR
- Portainer (управление контейнерами) — искажение: «партейнер»
- Komodo (управление серверами/контейнерами, кандидат на замену Portainer) — искажения: «комода», «комодом»
- AEZA (провайдер bare metal/хостинга) — искажения: «аэза», «аэро», «аэво»
- TacticumFlow (платформа) — искажение: «тактикумфлоу»
- 1С (интеграция) — искажение: «11-ка», «одиннадцатка»

## Проекты, эпики, кодовые имена
- Залог Успеха (ЗУ) — проект
- Codex (бывш. rag_eval_service)
- platform, agents, tacticum-dev, graph-builder, KB-Brownfield-Bootstrap
- cifragen/zus-codex (демо-доска ЗУ)
- Эпики Taiga: #1 Фундамент, #11 LLM Gateway, #13 Knowledge/RAG, #10 MCP Runtime, #14 Memory, #15 Identity/RBAC, #20 Tech Debt
- Ключевые истории: #32 (acceptance hybrid-search), #86 (Eval Harness), #33 (Document Processing)

## Жаргон
- on-prem, brownfield, water-shed / водораздел (platform vs app), cutover, multi-tenant, fail-closed

## Relations
- relates_to [[Architecture]]
- part_of [[02-Architecture]]