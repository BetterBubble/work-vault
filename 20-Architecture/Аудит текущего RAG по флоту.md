---
title: Аудит текущего RAG по флоту
type: report
permalink: tacticum/20-architecture/audit-tekushchego-rag-po-flotu-1
tags:
- architecture
- rag
- audit
- agents
- knowledge
---

# Аудит текущего RAG/Knowledge для консолидации в `knowledge`

**Источник на диске:** `~/tacticum/_analysis/CURRENT_rag_audit.md`
**Тип:** аудит (2026-06-22). Фактическое состояние retrieval по репозиториям vs целевая спека Platform.

## TL;DR
- Рабочий зрелый RAG — в `agents` (Qdrant/pgvector + Meili + Cohere-reranker + chunker + ingest), назначен «спиной» модуля `knowledge` (#32).
- Но это **НЕ hybrid**: семантика и полнотекст не сливаются (нет RRF); выбор пути по типу ресурса. Ключевой gap против #32.
- Векторный путь в `agents` **не фильтрует по tenant** → риск утечки; изоляция только на «коллекция на ресурс». Gap против #32.
- Эмбеддинги `agents` — OpenAI text-embedding-3-small/1536 через LiteLLM, НЕ tei/bge-m3. Расхождение модели/dim.
- Параллельный dense-RAG в `tacticum-dev/knowledge` (Qdrant + bge-m3 через tei, без полнотекста/reranker) — RE-KB.
- `tei_service` — готовый сервис эмбеддингов (bge-m3 через DeepInfra + локальный TEI gte), reranker'а в нём нет.
- `graph-builder` — НЕ GraphRAG, а конструктор диалоговых сценариев; канон GraphRAG — `arch-mcp` (Graphiti+FalkorDB).
- Самодостаточность: ядро retrieval в `agents` слабо связано; вынос реалистичен; разорвать 3 вещи (settings-синглтон, Langfuse, Pydantic-контракты).

## Relations
- part_of [[20-Architecture]]
- relates_to [[TARGET — целевая архитектура knowledge v2.3]]
- relates_to [[Карта зависимостей ядра retrieval]]
- relates_to [[MISMATCHES — концепт vs код]]
- relates_to [[glossary]]