---
title: 'Руководство: составление golden-set'
type: note
permalink: tacticum/20-architecture/rukovodstvo-sostavlenie-golden-set-1
tags:
- eval
- golden-set
- guide
- platform
- metrics
---

# Как составить golden-set для eval — руководство для разработчиков платформы

**Источник на диске:** `~/tacticum/fast_task/Руководство_golden-set_для_платформы.md`
**Тип:** руководство/спека. Принцип: **eval-движок общий (платформенный), golden-set — данные клиента (свой у каждого).**

## Суть
Golden-set — список пар «вопрос → правильный документ(ы)» из базы знаний клиента. Eval прогоняет вопрос через поиск, смотрит, нашёлся ли правильный документ, считает метрику (recall@k / MRR / nDCG). Меряет **retrieval** («находит ли поиск нужный документ»), а не качество формулировки ответа.

## Формат записи (JSON-массив)
`{ id, tenant_id, query, relevant_doc_ids[], tags[], difficulty (typical/hard/edge), source (synthetic/client), ideal_answer (опц.) }`.

## Relations
- part_of [[20-Architecture]]
- relates_to [[Runbook: прогон eval на сервере]]
- relates_to [[Golden-set ЗУ — покрытие (30 вопросов)]]
- relates_to [[Golden-set ЗУ — выверка по истине]]
- relates_to [[glossary]]