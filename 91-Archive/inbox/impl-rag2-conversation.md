---
title: impl-rag2-conversation
type: report
permalink: tacticum/00-board/impl-rag2-conversation
tags:
- rag2
- conversation
- implementer
- analyst
archived-at: 2026-07-29 18:12
---

# impl-rag2-conversation

status: draft
worktree: /Users/bubblemac/tacticum/helm-wt-rag2-conv
branch: feat/rag2-conversation
commit: 809426e83847e73f3e69c411c4ff1bd9bed21f48
base: 62e4e0e

## Что сделано
Память диалога для RAG#2 `/analyst` (`/api/rag2/answer`) — аналог проверенного conversation-context RAG#1 (docs) 1:1. Follow-up («а что по срокам?», «а кто отвечает?») теперь понимается в контексте прошлых вопросов/ответов той же беседы. Одиночный вопрос без `conversation_id` — байт-в-байт как раньше (демо-путь и тёплый синтез-кэш не тронуты).

## Файлы
- `alembic/versions/a5b6c7d8e9fa_rag2_conversation_turn.py` — НОВАЯ миграция (стор). down_revision=`f4e5d6c7b8a9` (RAG#1 turn), single head.
- `src/helm/infrastructure/db/models.py` — модель `Rag2ConversationTurn` (id, conversation_id, question, answer, created_at, expires_at; 2 индекса).
- `src/helm/infrastructure/db/repository.py` — `add_rag2_turn` / `get_recent_rag2_turns` / `purge_expired_rag2_turns` + `RAG2_TURN_ANSWER_CAP=800`.
- `src/helm/config.py` — флаги `rag2_conversation_context_enabled` (True), `_turns` (3), `_ttl` (1800).
- `src/helm/interface/api/routers/rag2.py` — `conversation_id` в `Rag2SearchIn`/`Rag2AnswerOut`; `_render_rag2_history`, `_history_fingerprint`, `_conversation_on`, `_load_rag2_history`, `_store_rag2_turn`; перепись `answer`, инжект истории в `_build_synth_user_prompt`/`_synthesize`.
- `tests/interface/test_rag2_conversation.py` — НОВЫЙ, 13 тестов.
- `tests/infrastructure/test_models_metadata.py` — +`rag2_conversation_turn` в EXPECTED_TABLES.

## Миграция
ДА, новая: `a5b6c7d8e9fa_rag2_conversation_turn` (таблица `rag2_conversation_turn` + 2 индекса). Проверена upgrade/downgrade против sqlite. Single head. **Лид накатит на проде.**

## Контракт (как фронт шлёт conversation_id)
- Вход POST `/api/rag2/answer`: тело `Rag2SearchIn` + опц. поле `conversation_id: str` (нить диалога, генерит фронт).
- Выход `Rag2AnswerOut`: добавлено эхо-поле `conversation_id` (фронт шлёт его обратно в следующем запросе — так `/answer` помнит контекст). Без conversation_id → `conversation_id=null`, память не задействована.
- Первый вопрос беседы: фронт генерит новый id (напр. uuid), шлёт его; последующие — тот же id.
- Стор: последние 3 хода, TTL 30 мин; синтез-ответ хранится усечённым до 800 симв.

## env-флаг
`HELM_RAG2_CONVERSATION_CONTEXT_ENABLED` (дефолт true). Также `HELM_RAG2_CONVERSATION_CONTEXT_TURNS` (3), `HELM_RAG2_CONVERSATION_CONTEXT_TTL` (1800с). Без conversation_id поведение = как сейчас независимо от флага.

## Механика
1. Задан conversation_id + флаг ON + есть session_factory → подтягиваем последние ходы ДО кэша.
2. Контекстный ретрив: follow-up-запрос обогащается прошлыми вопросами (`prior + query`) — как RAG#1. Топология/тесты — по сырому query.
3. Инжект истории отдельным блоком «НЕДАВНИЙ КОНТЕКСТ ДИАЛОГА» в начало промпта синтеза; наружу (context/цитаты/эхо query) — по исходному вопросу.
4. Кэш: без истории ключ = как раньше (тёплый прогрев 1:1); при непустой истории — суффикс `|hist=<sha1[:16]>` → follow-up НЕ читает/не перетирает тёплые записи. Записи в кэше хранятся БЕЗ conversation_id (нить проставляется на выдачу) — тёплый кэш одиночных вопросов не загрязняется.
5. Ход пишется после успешного синтеза (в т.ч. на кэш-хите — цепочка follow-up продолжается). Всё fail-soft: сбой БД не роняет ответ.

## Проверка (числа)
- Новые: `tests/interface/test_rag2_conversation.py` — **13 passed**.
- Регресс: `test_rag2` + `test_rag2_conversation` + `test_docs_conversation_context` + `test_docs_clarify` + `tests/infrastructure/` — **297 passed**.
- docs-роутер + eval rag2 — **22 passed**.
- ruff: clean на всех затронутых (единственная E501 в models.py:1425 — предсуществующая, чужой класс).
- mypy: 0 новых ошибок (база 24 = после 24, ни одной в моих файлах).

## Границы соблюдены
- RAG#1 (`docs_assistant`/`docs_conversation_turn`), `/context` — не тронуты.
- tests-матч функции (`topology.py`/`analyst_server.py`/`service.py::build_tests_summary`) — не касался (их правит другой implementer).
- Работал только в worktree `helm-wt-rag2-conv`, ветка `feat/rag2-conversation`. Не пушил/не мержил.

## Замечание
При правке `rag2.py` символьными операциями Serena правки не записывались на диск (десинк LSP, при этом возвращала OK) — переделал через Edit, всё сохранено и проверено. models/repository/config через Serena записались нормально.
