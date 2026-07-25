---
title: Карта зависимостей ядра retrieval
type: note
permalink: tacticum/20-architecture/karta-zavisimostei-iadra-retrieval-1
tags:
- architecture
- rag
- dependencies
- agents
- knowledge
---

# RAG_deps — карта зависимостей ядра retrieval (для выноса в `knowledge`)

**Источник на диске:** `~/tacticum/_analysis/RAG_deps.md`
**Тип:** анализ зависимостей (read-only, 2026-06-23). Что составляет ядро retrieval `agents` и что блокирует вынос в самостоятельный сервис.

## Состав ядра (agents)
Embedder (`langchain_openai`), Stores Qdrant/pgvector (`qdrant_client`/`psycopg`/`pgvector`), Reranker (`langchain_cohere`), Query expansion (`langchain_openai` + Langfuse prompts), Pipeline (`langchain_core`), Fulltext Meili (чистый), Chunker (`pdfplumber`/`docx`/`openpyxl`, чистый).
**Вывод:** ядро НЕ импортирует LangGraph/auth/ORM/UoW → выносимость подтверждена.

## Три зависимости, блокирующие чистый вынос
- **D-1. Settings-синглтон** (`src.config.settings`) — reranker читает cohere-ключи внутри ядра. Развязка: передавать конфиг параметрами (у embedder/query_gen уже так; у reranker — чинить).
- **D-2. Langfuse** для query-expansion (prompt provider). → за порт, v1 NAIVE.
- **D-3. Pydantic-контракты** — модели → в `contracts`.

## Relations
- part_of [[20-Architecture]]
- relates_to [[Аудит текущего RAG по флоту]]
- relates_to [[TARGET — целевая архитектура knowledge v2.3]]
- relates_to [[MISMATCHES — концепт vs код]]
- relates_to [[glossary]]