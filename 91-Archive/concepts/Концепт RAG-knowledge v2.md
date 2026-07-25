---
title: Концепт RAG/knowledge v2
type: note
permalink: tacticum/91-archive/concepts/kontsept-rag-knowledge-v2-1
tags:
- architecture
- knowledge
- rag
- concept
- superseded
status: archived
superseded-by: '[[Концепт- три RAG для ИВА на общем движке]]'
updated: 2026-07-18
---

# Концепт архитектуры RAG / knowledge (v2)

**Источник на диске:** `~/tacticum/_analysis/CONCEPT_rag_architecture-2.md`
**Тип:** архитектурный концепт v2 (для согласования). ⚠️ **Заменён** доработанной версией → [[Концепт RAG-knowledge (refined)]].

## Суть
Сейчас RAG трижды переизобретён (в `agents`, `tacticum-dev`, `TactFlow/форум`), везде по-своему и почти не обкатан. Консолидируем retrieval в один платформенный сервис `knowledge`; апки — потребители, RE-конвейер — производитель.

## Вводные из созвонов (что меняло картину)
- agents RAG — конфигурируемая инфра, почти НЕ используется (один Excel-ресурс fulltext); reranker Cohere в коде, не подключён; hybrid не делали; eval/golden нет → baseline почти greenfield.
- «OpenAI» в коде — прокси с OpenAI-совместимым интерфейсом (может быть GigaChat) → проверить, что реально отдаёт.
- «Ресурс» = текущая сущность документа; изоляция держится на «одна коллекция на ресурс»; тенант = организация; требование — строгая изоляция.
- LLM Gateway уже внедряют (dev заменил свой LiteLLM); задержка не мерена.
- eval надо СОЗДАТЬ под продуктовую необходимость.
- Codex/ADR — срочно для продажной демо.

## Relations
- part_of [[20-Architecture]]
- relates_to [[Концепт RAG-knowledge (refined)]]
- relates_to [[TARGET — целевая архитектура knowledge v2.3]]
- relates_to [[glossary]]