---
title: TARGET — целевая архитектура knowledge v2.3
type: note
permalink: tacticum/02-architecture/target-tselevaia-arkhitektura-knowledge-v2.3
tags:
- architecture
- knowledge
- rag
- target
- platform
---

# TARGET — целевая архитектура RAG / сервис `knowledge` (v2.3)

**Источник на диске:** `~/tacticum/_analysis/TARGET_architecture_v2.3.md`
**Тип:** целевой дизайн-документ («как должно быть»).

## Суть
Один переиспользуемый платформенный сервис retrieval **`knowledge`** (движок+контракт+инфра); приложения — **потребители** контракта, RE-конвейер — **производитель** (ETL, кормит `knowledge.ingest`). Принцип водораздела (Лебедев): переиспользуемое между апками → Platform; уникальное апке → в апке. Всё, что апки делали по-своему, заменяется контрактом `knowledge.{search,ingest,list_collections}`.

## Ключевые решения версии
- Приведено в соответствие с **ADR-0005** (границы knowledge/docproc): `doc.extract` — модуль Document Processing (не в knowledge, порт `DocExtract`); контракт search/ingest/list_collections (без `/doc/extract`); passthrough = ref+metadata.
- **API Hub не вводим сейчас** — адресация через Platform SDK + project-hub AuthScope, порт `ServiceAddr`.
- Эмбеддинги **bge-m3/1024 везде**; `tenant_id` в payload обязателен; реиндекс+payload — одной миграцией Phase 1.
- Изоляция: логическая дефолт + физическая опция по ADR; `StoreRouter` — порт; утечка вектор-пути латентна.
- **v2.3:** добавлена 4-я операция контракта `knowledge.pattern_search` (governance-retrieval, most-specific-wins, ADR-0043); §1.1 sequence-потоки; RE-конвейер = generic producer; решения по 3 зависимостям при extract (settings→DI, Langfuse→порт, Pydantic→contracts).

## Relations
- part_of [[02-Architecture]]
- relates_to [[Концепт RAG/knowledge (refined)]]
- relates_to [[Аудит текущего RAG по флоту]]
- relates_to [[Карта зависимостей ядра retrieval]]
- relates_to [[План изменений по репозиториям (Phase 0-1-2)]]
- relates_to [[Водораздел: платформенный knowledge vs Codex]]
- relates_to [[ADR-0001 — Демо-стенд RAG ЗУ (Codex)]]
- relates_to [[glossary]]
