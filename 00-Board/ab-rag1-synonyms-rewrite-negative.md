---
title: ab-rag1-synonyms-rewrite-negative
type: note
permalink: tacticum/00-board/ab-rag1-synonyms-rewrite-negative-1
status: current
role: rag1-lead
project: helm / RAG#1
tags:
- rag1
- ab-test
- retrieval
- negative-result
- real-data
- do-not-revisit
---

# RAG#1 проблема 1 — A/B синонимов/rewrite: ОТРИЦАТЕЛЬНО (не возвращаться)

Замер на прод-контейнере helm-helm-1 (retrieval-only + rerank, без мутации живого сервиса; probe флипал query_rewrite/synonyms в одном процессе). Golden: ambiguous-10 + сэмпл базы-184 (40 кейсов).

## Числа
| вариант | amb mrr | amb hit | base mrr | base hit |
|---|---|---|---|---|
| **baseline (rewrite OFF — прод)** | **0.633** | **0.800** | **0.854** | 0.900 |
| rewrite + текущие синонимы | 0.467 | 0.600 | 0.825 | 0.900 |
| rewrite + v2 (ВКС расщеплён) | 0.467 | 0.600 | 0.825 | 0.900 |

## Вывод
- **Включение query_rewrite РЕГРЕССИТ всё:** ambiguous 0.633→0.467, база 0.854→0.825 (совпадает с ранее задокументированным −0.019 на golden-1306).
- **Расщепление ВКС↔мероприятие (v2) = no-op:** результат идентичен текущим синонимам. Причина: ambiguous-запросы содержат «звонок/вызов/групповой звонок», а НЕ токен «ВКС» → синонимная склейка не была механизмом. Event-страницы перебивают звонок семантикой bge-m3 на «добавить участника» ≈ «участник мероприятия», не лексикой.
- Прод `docs_query_rewrite_enabled=False` — синонимы в проде и так не применялись. «Смоки-ган синонимов» из первичного диагноза опровергнут данными.

## Решение
**НЕ включать query_rewrite. НЕ трогать synonyms.json. Проблема 1 не имеет безопасного retrieval-рычага** (глобальный режим менять нельзя — прод-оптимум hybrid+rerank recall@5 0.949). Проблема 1 закрывается ПОВЕДЕНЧЕСКИ через **clarify** (переспрос на двусмысленном запросе) — единственный робастный фикс.

Остаточный follow-up (не блокер): 1 честный recall-miss `one-ios-ug-calls` не в топ-8 для «активный вызов на iPhone» — отдельная retrieval/корпус-задача, 1 кейс.

Связи: [[diag-rag1-issue1-correctness-realdata]] · [[plan-rag1-quality-3issues — корректность - clarify - контекст чата (новому лиду)]]. Ср. session-state «НЕ включать — доказано замером».