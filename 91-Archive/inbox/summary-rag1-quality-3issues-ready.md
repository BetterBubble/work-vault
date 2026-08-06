---
title: summary-rag1-quality-3issues-ready
type: note
permalink: tacticum/00-board/summary-rag1-quality-3issues-ready-1-1
status: awaiting-user-ok
role: rag1-lead
project: helm / RAG#1
tags:
- rag1
- summary
- ready-for-deploy
- handoff
- quality
archived-at: 2026-08-03 11:16
---

# RAG#1 3 проблемы — сводка, готово к деплою (ждёт OK пользователя)

Исполнение плана [[plan-rag1-quality-3issues — корректность - clarify - контекст чата (новому лиду)]]. Autonomy off — стоп на OK перед мержем/деплоем.

## Проблема 1 — корректность: диагноз + измеренный вывод
Диагностика на реальных данных (прод helm, ambiguous-golden). Воспроизведён баг Лебедева: «добавить участника в идущий звонок» → бот отвечает про мероприятие.
- Корень: event-interface-participant страницы перебивают `*-ug-calls` семантикой bge-m3 («добавить участника» ≈ «участник мероприятия»); правильная страница недоранжирована (ранги 2-3), 1 recall-miss (iOS). [[diag-rag1-issue1-correctness-realdata]]
- **Retrieval-фикс — измеренный ТУПИК:** включение query_rewrite регрессит и ambiguous (0.633→0.467), и базу (0.854→0.825); расщепление синонимов ВКС↔мероприятие = no-op. [[ab-rag1-synonyms-rewrite-negative]]
- **Вывод: проблема 1 не имеет безопасного retrieval-рычага → закрывается ПОВЕДЕНЧЕСКИ через clarify** (переспрос на двусмысленном). Никаких правок retrieval/synonyms.

## Проблема 2 — clarify: готово
Ветка **feat/rag1-correctness-clarify** (PASS гейт, 910 passed). CLARIFY_INSTRUCTION откалибрована на темпоральную (идущий звонок vs мероприятие) + продуктовую (One vs MCU) двусмысленность + анти-ложный блок. Мёртвый score-код убран. Ambiguous-golden (10 кейсов) + clarify_cases.json — в iva-rag1-docs. Флаг `docs_clarify_enabled=False` (ships OFF).

## Проблема 3 — контекст чата: готово
Ветка **feat/rag1-conversation-context** (PASS гейт, 111 passed). Стор `docs_conversation_turn` (append-only, TTL), инжект последних N ходов в промпт. Миграция f4e5d6c7b8a9. Флаг `HELM_DOCS_CONVERSATION_CONTEXT_ENABLED=False` (ships OFF).

## Гейт (controller [[gate-rag1-corrclar-convctx]])
Обе PASS: гит чист, без секретов/мусора/AI-подписей, скоуп ровно по задачам, retrieval не тронут, флаги OFF. Конфликт мержа НИЗКИЙ (merge-tree exit 0). Порядок: corrclar → convctx.

## Ключевое: дифф ИНЕРТЕН в проде
Оба флага OFF, retrieval-путь не меняется → деплой не может уронить correctness. Полный judge-замер golden-184 до/после сейчас измерял бы тот же прод → нецелесообразен. Реальная валидация — post-deploy при включении clarify на тесте.

## Осталось (ждёт OK)
1. Push обеих feature-веток (с OK) → материалы PR пользователю (сам создаёт+мержит; PR не создаём, без AI-подписей).
2. Деплой по runbook: ff→main (corrclar, затем convctx) → `SEED=0 bash scripts/deploy.sh` → `alembic upgrade heads` → verify (webhook 401 fail-closed, живой docs_ask). Флаги OFF.
3. **Post-deploy валидация:** включить clarify на тесте (`docs_clarify_enabled=1`), проверить по clarify_cases.json + ambiguous-кейсу, что переспрашивает по делу (не ложно) → оставить ON если ок. Аналогично conversation-context на тесте.

Follow-up (не блокер): recall-miss one-ios-ug-calls; конфиг-цепочка clarify_tau_answer.