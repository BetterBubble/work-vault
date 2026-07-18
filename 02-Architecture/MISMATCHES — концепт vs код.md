---
title: MISMATCHES — концепт vs код
type: report
permalink: tacticum/02-architecture/mismatches-kontsept-vs-kod
tags:
- architecture
- rag
- verification
- agents
---

# MISMATCHES — черновик концепта vs реальный код

**Источник на диске:** `~/tacticum/_analysis/MISMATCHES.md`
**Тип:** проверка по коду (read-only, 2026-06-23). Легенда: ✅ подтверждено · ⚠️ частично · ❌ опровергнуто · 🆕 новый факт.

## Ключевые расхождения
- **M1 ⚠️ «OpenAI = прокси».** Подтверждено, что LiteLLM-прокси (`agents/docker-compose.yml:21-22`), НО код явно просит `text-embedding-3-small`/1536 (`models.py:271-272`). Что прокси реально отдаёт — из репо не определить (внешний хост). Параметр `dimensions=1536` OpenAI-специфичен → при маршруте на bge-m3/1024 может игнорироваться. **Вывод:** план «реиндекс 1536→1024» нельзя строить без факта (Q-1).
- **M2 ⚠️ Реранкер Cohere вшит, но выключен** дефолтом (`reranker=None`) и падает без ключа; модель английская (`rerank-english-v3.0`).
- Прочее (из аудита): hybrid-слияния нет (путь по типу ресурса); eval нет нигде; целевой bge-m3/1024 через tei уже работает в tacticum-dev; движок Knowledge по ADR-0004 = adopt arch-mcp (напряжение с ADR-0001 D6.a).

## Relations
- part_of [[02-Architecture]]
- relates_to [[Концепт RAG/knowledge (refined)]]
- relates_to [[Аудит текущего RAG по флоту]]
- relates_to [[Карта зависимостей ядра retrieval]]
- relates_to [[glossary]]
