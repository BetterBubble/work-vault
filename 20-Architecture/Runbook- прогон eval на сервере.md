---
title: 'Runbook: прогон eval на сервере'
type: note
permalink: tacticum/20-architecture/runbook-progon-eval-na-servere-1
tags:
- eval
- runbook
- codex
- golden-set
- server
---

# Runbook: прогон eval на сервере → готовность к переносу на платформу

**Источник на диске:** `~/tacticum/fast_task/Проверка_eval_на_сервере.md`
**Тип:** руководство/runbook. Для того, кто запускает eval-harness на демо-стенде cifragen на маленьком golden-set — убедиться, что harness рабочий и переносим на платформу как общий движок.

## Принцип
Проверка через бэкенд `inprocess` (наш `rag.search` против нашего индекса) — «движок как таковой»: метрики, валидатор, режимы, срезы. Перенос на платформу = тот же движок, бэкенд `knowledge`; если `inprocess`-прогон зелёный — движок к переносу готов.

## Жёсткие правила
- Если на демо-сервере работают клиенты — **не запускать** (никакого rsync/rebuild/переиндексации/прогона под нагрузкой). Только в окно без клиентов.
- **Read-only к индексу** (поиск + scroll по `source_doc_id`), ничего не пишет/не удаляет.
- Секреты только из env (`GATEWAY_API_KEY`, `MEILI_MASTER_KEY`), `.env` не коммитить/не печатать.
- Чужие тенанты не трогать — прогон строго под `tenant_id=cifragen`; golden-set других клиентов не подмешивать (fail-closed).

## Предусловия
На сервере подняты: Qdrant (индекс cifragen, коллекция `knowledge__bge-m3_1024`), Gateway (bge-m3, EMBED_BACKEND=gateway + секреты), опц. Meili для hybrid.

> Этот runbook — прямая основа для замера LightRAG vs baseline (ADR-0002 D6): тот же инструмент, те же метрики.

## Relations
- part_of [[20-Architecture]]
- relates_to [[Руководство: составление golden-set]]
- relates_to [[ADR-0002 — LightRAG (graph-RAG) для ЗУ]]
- relates_to [[lightrag-в-codex]]
- relates_to [[glossary]]