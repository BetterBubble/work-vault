---
title: explore-rag1-conversation-store-map
type: note
permalink: tacticum/00-board/explore-rag1-conversation-store-map-1
tags:
- rag1
- docs-bot
- conversation-store
- clarify
- explore
---

# explore-rag1-conversation-store-map

status: draft
Репо: /Users/bubblemac/tacticum/helm (main = ea295f8). Задача-контекст: RAG#1 docs-бот stateless per chatRoomId, не помнит предыдущий вопрос. Направление — по образцу `docs_clarify_pending` завести общий conversation-context store (последние N пар Q/A per chatRoomId, TTL) и инжектить недавние ходы в промпт ответа. Карта read-only, правок нет.

## 1. Стор docs_clarify_pending — прецедент обходки stateless

### Модель — `DocsClarifyPending`
`src/helm/infrastructure/db/models.py:2134-2167`, `__tablename__ = "docs_clarify_pending"`. Поля:
- `id` PK autoincrement
- `channel: str` — `bot | web`
- `conversation_key: str` — **chatRoomId (бот)** или email (веб)
- `original_question: Text` — накопленная формулировка
- `clarify_question: Text` (default "")
- `ask_count: int` (default 1, кап ≤2)
- `last_message_id: str` (default "") — идемпотентность webhook-дедупа
- `created_at` (server_default now), `expires_at: DateTime(tz)` **index=True** — TTL
- `UniqueConstraint("channel","conversation_key", name="uq_docs_clarify_conv")` — одна активная запись на диалог (upsert)

### Миграция
`alembic/versions/d2c3b4a5e6f7_docs_clarify_pending.py` — revision `d2c3b4a5e6f7`, down_revision `ab12cd34ef56`. Написана вручную (без autogenerate), по стилю `f1a2b3c4d5e7_docs_qa_feedback.py`. create_table + `create_index("ix_docs_clarify_pending_expires_at", ["expires_at"])`.

### Репо-функции — `src/helm/infrastructure/db/repository.py:2020-2149`
- `DOCS_CLARIFY_MAX_ASKS = 2` (:2023) — константа капа
- `get_active_clarify(session, *, channel, conversation_key, now) -> DocsClarifyPending|None` (:2026) — select по `(channel,key)`; TTL проверяется в Python через `_expired()`, НЕ в SQL (устойчиво к naive datetime sqlite)
- `upsert_clarify(...)` (:2056) — select-then-update/insert (без ON CONFLICT, переносимо sqlite↔pg); клампит ask_count 1..MAX
- `clear_clarify(session, *, channel, conversation_key)` (:2106) — DELETE по ключу (диалог завершён), `synchronize_session=False`
- `purge_expired_clarify(session, *, now)` (:2125) — DELETE `expires_at <= now` (гигиена)
- `_expired(expires_at, now)` (:2134) — корректное сравнение naive/aware
- `clarify_expires_at(now, ttl_seconds)` (:2147) — `now + timedelta(seconds=max(1,ttl))`

### Конфиг — `src/helm/config.py:172-186`
`docs_clarify_enabled: bool = False` (мастер-флаг), `docs_clarify_ttl_seconds: int = 900`, tau_answer/tau_floor.

## 2. Где стор читается/пишется в потоке (обе петли)

Общий алгоритм — `src/helm/interface/api/docs_clarify.py`, `resolve_with_clarify(session, ask_fn, *, channel, conversation_key, message_id, question, ttl_seconds, now)` (:34-89). Порядок:
1. `repo.purge_expired_clarify` (:51)
2. `repo.get_active_clarify` (:52)
3. дедуп по `last_message_id` → None (:58)
4. склейка `original_question + "\n" + question` если pending (:66)
5. `allow_clarify = prior_asks < DOCS_CLARIFY_MAX_ASKS` (:70), `answer = ask_fn(...)` (:71)
6. `answer.clarify` → `repo.upsert_clarify` (:74); иначе `repo.clear_clarify` (:86)

### Бот-webhook — `src/helm/interface/api/routers/bot_support.py`
- `POST /webhook` `webhook(...)` (:213-265): верификация секрета, фильтр CHAT_NEW_MESSAGE/не BOT/не group (:235), `chat_room_id = data.chatRoom.chatRoomId` (:243), `message_id = data.messageId`; планирует фон `_process_message` (:256). `session_factory = app.state.session_factory` (:255)
- `_process_message(...)` (:171-210) → `_resolve_answer(...)` (:127-168): если `docs_clarify_enabled` И есть session_factory → открывает КОРОТКУЮ сессию, зовёт `resolve_with_clarify(channel="bot", conversation_key=chat_room_id, message_id=message_id, now=datetime.now(UTC))` (:154), commit/rollback. Иначе — прямой `run_docs_ask` (:143, без БД).
- `_ask` адаптер (:147): `assistant.ask(q, plain=True, allow_clarify=...)` через `run_in_threadpool`.
- BotEvent-схемы тут же: `ChatRoom.chatRoomId` (:64), `isGroupChat` (:66), `messageId` (:74), `BotEvent` (:80).

### Веб /docs — `src/helm/interface/api/routers/docs.py`
- `ask` endpoint (:~239-286): `conversation_id: str|None` в body (:83). `clarify_on = docs_clarify_enabled AND conversation_id AND principal AND session_factory` (:254). Если on → `_ask_with_clarify(... conversation_key=principal.email ...)` (:262); иначе прямой `assistant.ask` (:271).
- `_ask_with_clarify(...)` (:289-327): `resolve_with_clarify(channel="web", conversation_key=email, message_id="")` (:311). Веб = запрос/ответ, дедупа нет (message_id пуст).

**Важно:** ключ диалога у бота = chatRoomId, у веба = email. conversation-store должен использовать ту же схему `(channel, conversation_key)`.

## 3. Точка инжекта в промпт ответа

`src/helm/application/docs_assistant.py`, `DocsAssistant.ask(question, *, filters, plain, allow_clarify)` (:240-340). Промпт собирается в два куска:
- system = `SYSTEM_PROMPT + (PLAIN_SUFFIX if plain) + (CLARIFY_INSTRUCTION if allow_clarify)` (внутри ask, ~строки после known-gaps/ретрива)
- user = `_build_user_prompt(question, context)` (:203-204) → `f"Вопрос: {question}\n\nФрагменты документации:\n{context}"`, где `context = render_docs_context(chunks)`.
- вызов `self._llm.generate(system=..., user=_build_user_prompt(question, context))`.

**Логичное место инжекта «последних N ходов»:** между ретривом и вызовом LLM — либо расширить `_build_user_prompt(question, context, history=...)` (добавить блок «Предыдущие ходы диалога»), либо добавить историю в system. `ask()` сейчас не знает про conversation_key — историю нужно прокинуть параметром в `ask()` (напр. `history: Sequence[tuple[q,a]] | None`) из петли/адаптеров `_ask`, ЛИБО читать стор внутри resolve-обёртки и передавать в ask_fn. Точка вызова known-gap `classify_known_gap(question)` (:начало ask) — история туда не нужна.

Символы-константы промпта: `SYSTEM_PROMPT`, `PLAIN_SUFFIX`, `CLARIFY_INSTRUCTION`, `CLARIFY_MARKER` (overview файла).

## 4. Контракт бота (почему stateless)

- Бот получает изолированные webhook-события; НЕТ серверной сессии диалога. Идентификатор нити: **chatRoomId** приходит в `BotEvent.data.chatRoom.chatRoomId` (bot_support.py:64,243). Веб — `conversation_id` в body + `principal.email` (docs.py:83,266).
- Стейт держится ТОЛЬКО в БД helm (таблица `docs_clarify_pending`), не в платформе ИВА. Тот же принцип для conversation-store — своя таблица в helm, ключ `(channel, conversation_key)`, TTL. Платформу не трогать.
- Сессия открывается КОРОТКО в фоне (webhook уже отдал 200), фабрика `app.state.session_factory` (`src/helm/main.py:67`, `create_session_factory`).

## 5. Миграции / alembic

- Директория `alembic/versions/` (68 файлов). В истории был multi-head merge: `93ce6c200263_merge_heads_velocity_stat_arch_c4_nodes.py` (down_revision = tuple `('0a1c4e2b9d77','f9c0d1e2a3b4')`) — прецедент слияния голов.
- **Текущая единственная head: `e3d4c5b6a7f8`** (`e3d4c5b6a7f8_topology_placement_adr0004.py`, down_revision `d2c3b4a5e6f7`). Цепочка near-clarify: `ab12cd34ef56` → `d2c3b4a5e6f7` (clarify) → `e3d4c5b6a7f8` (head).
- Новую миграцию conversation-context ставить как ребёнка текущей головы: `down_revision = "e3d4c5b6a7f8"`. Писать вручную (без autogenerate), по образцу `d2c3b4a5e6f7_docs_clarify_pending.py`: create_table + index по `expires_at`, UniqueConstraint если нужна.

## Что переиспользуемо для conversation-store
- Модель-паттерн: `(channel, conversation_key, ..., expires_at index)`. Но conversation-store — это N последних пар, значит либо **строки-события** (по одной на ход, без UniqueConstraint, выборка top-N по created_at desc), либо одна строка с JSON-массивом ходов (upsert как clarify). Первый вариант чище для «последних N + TTL».
- Репо-функции по образцу: `get_recent_turns(channel, key, now, limit=N)`, `add_turn(...)`, `purge_expired_turns(now)`, TTL через `_expired`/`*_expires_at`. Переиспользовать `_expired` (:2134) как есть.
- Интеграция в петлю: расширить `resolve_with_clarify` или добавить параллельную обёртку — читать историю ДО `ask_fn`, писать пару (question, answer.text) ПОСЛЕ. Прокинуть историю в `ask_fn` → `assistant.ask(history=...)`.
- Миграция: копия `d2c3b4a5e6f7_...` с новой таблицей, `down_revision="e3d4c5b6a7f8"`.

## Риски / открытые вопросы
- `ask()` сейчас не принимает историю — сигнатуру придётся расширить (или инжектить историю в `_build_user_prompt`). Это касается ВСЕХ вызовов ask (бот plain, веб, а также прямые пути без clarify).
- Для веба ключ = email, для бота = chatRoomId; при N-turns store учесть, что у веба conversation_id тоже есть (может быть лучшим ключом, чем email).
- Взаимодействие с clarify: склейка вопросов уже частично «помнит» в рамках одной clarify-петли; conversation-store — отдельный слой (завершённые пары Q/A). Не смешивать `original_question` (clarify) и историю ходов.
- TTL conversation-store, вероятно, длиннее clarify (900s) — новый конфиг-параметр.