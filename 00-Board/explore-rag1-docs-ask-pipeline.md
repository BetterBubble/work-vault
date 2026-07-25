---
title: explore-rag1-docs-ask-pipeline
type: explore
permalink: tacticum/00-board/explore-rag1-docs-ask-pipeline
status: draft
repo: /Users/bubblemac/tacticum/helm (main, eaf10f8)
role: explorer (read-only)
tags:
- rag1
- docs_ask
- explore
- draft
- helm
---

# RAG#1 `docs_ask` — разведка пайплайна ответа

Заметка-разведка для плана улучшений чат-бота RAG#1 (ассистент публичной доки ИВА). Канон не пишу.

## 1. Схема пайплайна (этапы + path:line + модель)

Точка входа оркестрации: `DocsAssistant.ask` — `src/helm/application/docs_assistant.py:176-226`.

1. **Гейт known-gaps** — `docs_assistant.py:187-190` → `classify_known_gap` (`domain/known_gaps.py`, антипримеры `domain/known_gaps.json`). Вопрос вне охвата (цены/роадмап/конкуренты) → честный отказ СРАЗУ, без ретрива и LLM.
2. **Ретрив** — `DocsSearch.search` `infrastructure/docs_assistant/search.py:106-132`. Запрос: `SEARCH_LIMIT=30` (`docs_assistant.py:33`), но из каждого стора тянется `CANDIDATE_K=60` (`search.py:29`).
   - (опц.) query-rewrite синонимов `search.py:112-117` (флаг `docs_query_rewrite_enabled=False` дефолт).
   - dense Qdrant `DocsVectorStore` (коллекция `iva_docs__bge_m3_1024`, `config.py:102`); эмбеддинг запроса — bge-m3 через основной Gateway (`DEFAULT_EMBED_MODEL`).
   - fulltext Meili + RRF-слияние `rrf_fuse` (`_hybrid` `search.py:157-187`); dense и Meili зовутся ПОСЛЕДОВАТЕЛЬНО. Сбой Meili → semantic-only (не роняем).
   - (опц.) near-dup дедуп `search.py:130-131` (флаг `docs_near_dup_dedup_enabled=False`).
3. **Реранк (опц.)** — `docs_assistant.py:193-204`, `DocReranker` `infrastructure/docs_assistant/reranker.py` через `GatewayRerankClient` (`tacticum/rerank`, bge-reranker-v2-m3). Флаг `docs_rerank_enabled=False` дефолт. Реранкует ВЕСЬ набор кандидатов (`top_n=len(chunks)`). Порог `rerank_floor=None` дефолт.
4. **Cap + усечение** — `_cap_per_doc` (max 4 чанка/страница) затем `[:context_limit=10]` — `docs_assistant.py:205`.
5. **Сборка контекста** — `render_docs_context` (`domain/docs.py`) `docs_assistant.py:206`.
6. **LLM-генерация** — `docs_assistant.py:208-213`. Порт `IvaLlm` → `TrivaLlm.generate` (`infrastructure/assistant/service.py:26-53`) → `GatewayClient.chat` (`src/helm/llm/gateway.py:168-195`) → OpenAI `chat.completions.create`.
   - **Модель: `triva`** (`config.iva_llm_model="triva"`, `config.py:89`), внутренний **vLLM** в контуре ИВА (`iva_llm_base_url`), отдельный Gateway-клиент. temperature=0.2, `max_tokens=docs_answer_max_tokens=1536`.
7. **sanitize_markdown** (анти-LaTeX) `docs_assistant.py:213`.
8. **Guardrail** — `decide_guardrail(len(chunks), text)` `docs_assistant.py:215` (пусто/«не нахожу» → `not_found`).
9. **select_cited_chunks** — только реально процитированные [n] `docs_assistant.py:218`.

Каналы поверх: REST `/api/docs/ask` (`interface/api/routers/docs.py:192-235`, через `run_in_threadpool`); MCP-тул `docs_ask` (`interface/mcp/analyst_server.py:395-405`, зовёт тот же `docs_router.ask`); бот «Поддержка» (`routers/bot_support.py`, `application/bot_support.py`).

## 2. Латентность — где время

- **Стриминга НЕТ.** `GatewayClient.chat` зовёт `chat.completions.create` без `stream=True` — ответ возвращается ЦЕЛИКОМ (`gateway.py:189-195`). Пользователь ждёт генерацию до 1536 токенов, прежде чем увидит хоть что-то. Это главный источник «долгих ответов», если речь про скорость.
- Этапы с сетью: embed запроса (1 вызов Gateway) → Qdrant + Meili (последовательно) → RRF/дедуп (локально) → опц. реранк (ещё 1 вызов Gateway по всем ~60 кандидатам) → **LLM-генерация (доминирует по времени)**.
- top-k: запрос 30 / кандидатов 60 на стор; контекст 10 чанков; при включённом реранке ранжируется весь набор.
- **Кэша ответов нет** — только `lru_cache` на словарь синонимов (`query_rewrite.py:36`).
- Бот: `_process_message` в `BackgroundTasks`, webhook отвечает 200 сразу; но сам ответ клиенту уходит ОДНИМ `send-message` после полной генерации — в чате стриминга тоже нет.

## 3. Длина/многословность ответа

- Управляется прежде всего **системным промптом** `SYSTEM_PROMPT` (`docs_assistant.py:61-79`): явно требует «РАЗВЁРНУТЫЙ пошаговый ответ», «не сокращай инструкцию» → тянет в многословие.
- Жёсткий потолок `max_tokens=docs_answer_max_tokens=1536` (`config.py:144`), прокинут в `TrivaLlm`.
- Инструкции краткости НЕТ. Рычаги сокращения: правка `SYSTEM_PROMPT` / снижение `max_tokens`. Если «долгие» = многословные — менять здесь.

## 4. Уточняющие вопросы (clarify / ask-back)

- **Механизма НЕТ.** Греп `clarify|уточня|follow-up|multi-turn|диалог` по docs/bot — пусто. Ответ всегда одноходовый.
- Где встроить логически: в `DocsAssistant.ask` (после ретрива — если контекст слабый/неоднозначный, вернуть уточняющий вопрос) либо в оркестраторе бота.
- **Контракт бота многоходовость НЕ поддерживает:** входящее событие `_BotEventData` (`bot_support.py:66-74`) несёт ОДНО `message: str`, без истории/треда. `chatRoomId` есть (можно коррелировать), но состояние нигде не хранится. Для clarify нужен внешний стор состояния беседы по `chatRoomId`.

## 5. История/контекст диалога

- **/docs (веб):** `DocsQa` персистится на пользователя (`save_docs_qa`/`list_docs_qa`, `routers/docs.py:244-277`) — но это ЛОГ-снимок «недавних вопросов» для повторного открытия, в промпт генерации НЕ подаётся. Не разговорный контекст.
- **Бот:** состояния НЕТ вообще — stateless на каждое сообщение.
- Опереться на историю для «продолжения»/уточнений сейчас нельзя без доработки.

## 6. Точки улучшения (эффект/сложность)

- **Стриминг ответа** — высокий эффект на воспринимаемую скорость / средняя сложность (нужен `stream=True` в Gateway/vLLM + SSE-эндпоинт + рендер; в чат-канале платформа шлёт одно сообщение — там стриминг может быть неприменим).
- **Контроль многословности** (правка `SYSTEM_PROMPT` / `max_tokens`) — высокий эффект / низкая сложность.
- **Кэш ответов** (по вопросу+фильтрам) — средний эффект / низко-средняя.
- **Clarify / follow-up-шаг** — высокий эффект / средне-высокая (нужен многоходовый контракт + стор состояния по chatRoomId).
- **Включить реранк + floor** (уже написаны, только флаги `docs_rerank_enabled`/`docs_rerank_floor`) — качество↑, но латентность↑ (+1 сетевой вызов).
- **Разговорная история в промпт** (веб уже пишет DocsQa) — средний / средняя.
- **Параллелить Qdrant+Meili** в `_hybrid` (сейчас последовательно) — малый-средний эффект на латентность ретрива / низкая.

## Не подтверждено / не найдено
- Реальные значения `iva_llm_*` в проде (только дефолты конфига).
- Точность формулировки «долгие ответы» (скорость vs многословность) — покрыл оба.
- End-to-end работа бота (по коду задеплоено, фактического прогона в коде не видно).