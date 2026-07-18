---
title: Концепт RAG/knowledge (refined)
type: note
permalink: tacticum/02-architecture/kontsept-rag-knowledge-refined
tags:
- architecture
- knowledge
- rag
- concept
---

# Концепт RAG / `knowledge` — доработанный (fact-checked)

**Источник на диске:** `~/tacticum/_analysis/CONCEPT_refined.md`
**Тип:** архитектурный концепт (refined, 2026-06-23). База: концепт v2 + проверка по коду/MCP. Заменяет [[Концепт RAG/knowledge v2]].

## Суть
RAG в флоте реализован фактически **дважды** (`agents` — бизнес-агенты; `tacticum-dev/knowledge` — RE-KB) + producer (RE-конвейер, ChromaDB). Консолидируем retrieval в один платформенный сервис `knowledge`; апки — потребители контракта, RE — производитель.

## Что уточнено проверкой кода (главное)
1. Прокси подтверждён (LiteLLM), но реальная размерность эмбеддингов открыта (код просит text-embedding-3-small/1536, что отдаёт прокси — неизвестно). → Q-1.
2. У `agents` нет ingest в Qdrant вообще — вектор наполняется извне; «фикс утечки tenant» = новый ingest + владение payload, не правка фильтра.
3. TactFlow/форум — НЕ третья RAG-реализация (нет вектора/эмбеддингов/реранка); будущий потребитель.
4. Движок Knowledge по ADR-0004 — adopt `arch-mcp` (Graphiti/FalkorDB); напряжение с ADR-0001 D6.a; развести «extract vs adopt».
Подтверждено: reranker Cohere выключен; hybrid-слияния нет; eval нет нигде; целевой стек bge-m3/1024 через tei уже работает в `tacticum-dev`.

## Relations
- part_of [[02-Architecture]]
- relates_to [[TARGET — целевая архитектура knowledge v2.3]]
- relates_to [[Концепт RAG/knowledge v2]]
- relates_to [[MISMATCHES — концепт vs код]]
- relates_to [[Аудит текущего RAG по флоту]]
- relates_to [[glossary]]
