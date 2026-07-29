---
title: report-rag1-convctx
type: report
permalink: tacticum/00-board/report-rag1-convctx-1
tags:
- rag1
- conversation-context
- docs-bot
- helm
- implementer
archived-at: 2026-07-29 18:12
---

# report-rag1-convctx

status: draft
Ветка: `feat/rag1-conversation-context`
Worktree: `/Users/bubblemac/tacticum/helm-wt-rag1-convctx` (база main ea295f8)

## Задача
Проблема 3 плана: бот RAG#1 stateless per chatRoomId, не помнит предыдущий вопрос.
Добавлен conversation-context store — последние N пар вопрос/ответ per диалог,
инжект недавних ходов в промпт ответа. По образцу легитимной обходки stateless
`docs_clarify_pending`. Отдельный слой ПОВЕРХ clarify (его логику не трогает).

## Всё за фиче-флагом — ships OFF
`HELM_DOCS_CONVERSATION_CONTEXT_ENABLED` (config `docs_conversation_context_enabled`),
**default False**. При OFF: каналы историю НЕ собирают и НЕ пишут, промпт и поведение
1:1 как сейчас. Также: `..._TURNS` (default 3), `..._TTL` (default 1800с).

## Изменённые файлы + символы
- `src/helm/infrastructure/db/models.py` — новый `DocsConversationTurn` (append-only,
  БЕЗ UniqueConstraint; индексы `ix_docs_conversation_turn_conv` (channel,key,created_at)
  и `..._expires_at`).
- `alembic/versions/f4e5d6c7b8a9_docs_conversation_turn.py` — ручная миграция,
  down_revision `e3d4c5b6a7f8`.
- `src/helm/infrastructure/db/repository.py` — `add_turn`, `get_recent_turns`
  (top-N created_at desc + тай-брейк по id → хронология, Python-фильтр `_expired`),
  `purge_expired_turns`. Переиспользован `_expired`/`clarify_expires_at`(now+ttl).
- `src/helm/config.py` — три конфига рядом с `docs_clarify_*`.
- `src/helm/application/docs_assistant.py` — `DocsAssistant.ask(history=...)`;
  `_render_history` + `_build_user_prompt(history)` — блок «Недавний контекст диалога»
  МЕЖДУ ретривом и `llm.generate`, помечен как прошлые ходы (не источник фактов);
  пусто/None → блока нет.
- `src/helm/interface/api/docs_clarify.py` — `resolve_with_clarify` расширен НЕЗАВИСИМЫМИ
  слоями: `clarify_enabled` (дефолт True — старое поведение) и `context_enabled`
  (+`context_turns`,`context_ttl_seconds`). При context ON: `get_recent_turns` перед ask
  (history в ask_fn только если непусто), `add_turn` после завершённого (НЕ clarify)
  ответа по сырому вопросу; `purge_expired_turns` рядом с purge clarify.
- `src/helm/interface/api/routers/bot_support.py::_resolve_answer` — гейт
  `(clarify OR context) AND session_factory`; адаптер `_ask(history=...)`; проброс флагов/TTL.
- `src/helm/interface/api/routers/docs.py::ask/_ask_with_clarify` — то же для веба
  (ключ=email), переменная гейта переименована clarify_on→stateful_on.
- Тесты: `tests/interface/test_docs_conversation_context.py` (новый),
  `tests/application/test_docs_assistant.py` (+2), `tests/infrastructure/test_models_metadata.py` (+таблица).

## Тесты
Команда: `uv run pytest tests/interface/test_docs_conversation_context.py
tests/application/test_docs_assistant.py tests/interface/test_docs_clarify.py
tests/domain/test_docs_clarify_gate.py tests/interface/test_docs.py
tests/interface/test_bot_support.py tests/application/test_bot_support.py
tests/infrastructure/test_models_metadata.py -q`
Результат: **111 passed, 0 failed**. Ruff/mypy по изменённым файлам чисто
(единственная ruff-ошибка E501 в models.py:1425 — ПРЕДсуществующая, не моя).

## Alembic
Head после миграции: **f4e5d6c7b8a9** (единственный head, chain от e3d4c5b6a7f8).
На прод НЕ применял (по заданию). Проверка `alembic heads` — один head; upgrade на
sqlite не гоняется (env.py смотрит в postgres), ORM-создание таблицы валидирует
`test_models_metadata` + фикстура session_factory (create_all).

## Осталось / риски
- get_recent_turns полагается на purge перед выборкой (top-N desc + Python-фильтр
  expired): если протухших в top-N много, вернёт меньше N валидных даже при наличии
  старых валидных. На практике purge вычищает их той же транзакцией — ок.
- В историю пишутся и честные отказы (not_found) как терминальный ход — осознанно
  (бот «помнит», что сказал «не нахожу»). Если нужно писать только содержательные —
  тривиально сузить в `resolve_with_clarify`.
- Формат блока истории/промпта — на калибровку продуктом (объём, метка).
- Не мержил / не пушил / не деплоил / миграцию на прод не применял.