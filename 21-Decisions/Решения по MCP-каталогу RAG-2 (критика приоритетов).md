---
title: Решения по MCP-каталогу RAG#2 (критика приоритетов)
type: note
permalink: tacticum/21-decisions/resheniia-po-mcp-katalogu-rag-2-kritika-prioritetov
tags:
- iva
- rag2
- mcp
- decisions
- eval
- glossary
- design
---

# Решения по MCP-каталогу RAG#2

Приняты после критики пользователя (пересмотр приоритетов из [[format-postanovki-analitika-iva-fr-feature-request-kanon]] / слои / стратегия). Заземлено фактами через iva-mcp и репо.

## Р1. Eval-гейт ИДЁТ ДО изменений ретрива (в т.ч. glossary)
- «Glossary системно чинит качество» — было на вере. Query-expansion: recall↑, но precision↓ (синонимы=шум); в мультипродукте один термин значит разное (MCU vs Mail) — реальный риск.
- Eval операционален: `src/helm/eval/` (rag2_eval, rag2_runner, sweep, metrics recall/precision/ndcg/mrr, judge) + golden `tests/data/rag2_golden.json`, RAG#1 `golden_iva_rag1.json` (1306 кейсов).
- **Решение:** baseline на golden ПЕРЕД включением; glossary/expansion — вариант в `sweep`; гейт = **precision не просел**. Правило распространяется на ЛЮБОЕ изменение ретрива. Expansion — product-scoped + только редкие/OOV-термины (снизить precision-риск). Glossary → из «приоритет №1» в «гейтируемый эксперимент после baseline».

## Р2. effort_hint — считать активное время, честные подписи
- created→resolved = lead time (ехало), НЕ трудоёмкость (тикет мог лежать в бэклоге).
- **Решение:** из changelog считать время в активных статусах отдельно; поля `active_days` + `lead_time_days`; подпись «справка, как обычно едет — не оценка работы». Слово «оценка» не использовать.

## Р3. contradiction_check — только кандидаты, не вердикт
- Ложноотрицательный опаснее отсутствия тула («конфликтов нет» → аналитик идёт дальше).
- **Решение:** выход = «потенциальные пересечения для проверки» + обоснование; НИКОГДА не «чисто/нет конфликтов»; отсутствие ≠ отсутствие конфликтов. Recall-ориентир, верифицирует человек.

## Р4. Тулы различаются по ВОПРОСУ, не по источнику (борьба с routing-хаосом)
- Риск: `analyst_context`/`docs_ask`/`related_tasks`/`nearest_spec` = 4 «найди похожий текст» → агент путает роутинг (главная смерть MCP).
- **Решение:** описания взаимоисключающие по вопросу: docs_ask=«как СЕЙЧАС по офиц.доке», analyst_context=«контекст понять требование», related_tasks=«похожие ЗАДАЧИ», nearest_spec=«готовая ПОСТАНОВКА взять за основу». `analyst_context`+`nearest_spec` проверить на routing-eval — если путает, СЛИТЬ (nearest_spec = режим context с FR-фильтром). Цель ~6-8 различимых тулов, не 14.

## Р5. IVASUP — data-ingest, не отдельный тул (пока)
- **Решение:** добавить IVASUP (+IVA1C/IVACRM) в jira-scope; surface «сигнал боли» (N обращений по фиче) внутри analyst_context/related_tasks. Отдельный тул `support_signal` — только если eval покажет спрос. Зафиксировано осознанно (не забыто).

## Р6. Produce-половины в MCP НЕТ — это решение
- **Решение:** MCP = провайдер контекста/ретрива; генерацию (заполнение FR-скелета) делает КЛИЕНТ (Claude Code) из данных MCP. Это «не пихать тяжёлую генерацию» (подтверждение прошлого решения). Никаких «дай черновик ФТТ» тулов в MCP.

## Пересмотренный порядок
0. **Baseline eval** на rag2_golden (сейчас) — точка отсчёта для всего.
1. **Триада действий** (nearest_spec · who_to_involve · effort_hint[исправленный]) — на готовых данных, макс ROI, меряем golden'ом.
2. **affected_systems✎** (+релиз/клиенты/риски).
3. **Glossary** — только как гейтируемый эксперимент (precision-guard).
4. constraints/contradiction_check[кандидаты] (после индекса ADR+IS), gap_questions, contracts, surface changelog/комментов.

Связано: [[RAG-2 для аналитика — слои данных за пределами топологии + глоссарий (дизайн)]], [[RAG-2 — внешние корпуса, упущенное внутри, и «действия важнее данных» (стратегия)]].
