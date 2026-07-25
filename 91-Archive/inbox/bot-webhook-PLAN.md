---
title: PLAN — webhook-сервис бота «Поддержка» (RAG#1 → чат IVA)
type: plan
tenant: iva
worktree: ~/tacticum-worktrees/helm-bot-webhook (feat/bot-support-webhook)
date: 2026-07-16
permalink: tacticum/91-archive/inbox/bot-webhook-plan
status: archived
updated: 2026-07-18
---

# PLAN: webhook бота «Поддержка» (RAG#1 → чат-бот IVA)

Первый исходящий бот. Строго ДВУХКОНТУРНО наружу: **только RAG#1 (публичная дока
iva.ru/docs)**, никакого RAG#2/внутреннего. Allowlist по источнику цитат.

## Что изучено (как звать RAG#1)
- `routers/docs.py::ask` — образец: `build_docs_assistant_context(settings)` →
  `DocsAssistantContext` (кэш на `app.state.docs_assistant_context`), затем
  `DocsAssistant(ctx.search, ctx.llm, reranker=..., ...)` и
  `await run_in_threadpool(assistant.ask, question, filters=...)` → `DocsAnswer`.
- `DocsAnswer(text, evidence: tuple[DocChunk], not_found, reason)`.
  Цитаты для UI: `build_citations(evidence)` → `DocCitation(n,title,slug,url,...)`.
  У doc-цитат `url` = публичная ссылка вида `https://iva.ru/docs/...`.
- `main.py`: data-роутеры идут под `_AUTH=[Depends(require_user)]`; `hrd.router`
  монтируется БЕЗ `_AUTH` (свой гейт). Наш webhook — так же БЕЗ `_AUTH`
  (аутентификация платформы = секрет-заголовок, не project-hub токен).
- httpx уже в deps; образец async-клиента с инъекцией `transport` для тестов —
  `infrastructure/auth/resolve_client.py` (`httpx.MockTransport` в тестах).

## Архитектура (переиспользуем RAG#1, не дублируем)

### 1. Config (`config.py`, +4 поля Settings; env-префикс `HELM_`)
- `iva_bot_base_url: str = "https://iva-uc.ru"`
- `iva_bot_support_token: str | None = None`  (env `HELM_IVA_BOT_SUPPORT_TOKEN`)
- `iva_bot_support_webhook_secret: str | None = None` (env `HELM_IVA_BOT_SUPPORT_WEBHOOK_SECRET`)
- `iva_bot_support_enabled: bool = False`
Секреты — только из env, в код не пишем.

### 2. Application (чистая логика) — `application/bot_support.py`
- `BotReply(message: str, blocks: list[dict])` (dataclass).
- `build_bot_reply(answer: DocsAnswer) -> BotReply`:
  - `message = answer.text` (или вежливый дефолт при пустом тексте).
  - `not_found=True` → message = вежливый отказ «не нашёл в документации ИВА»
    (если у answer нет своего текста), `blocks=[]`.
  - иначе blocks = link-блоки из `build_citations(answer.evidence)` —
    **только цитаты с url, прошедшие allowlist** публичной доки
    (`_ALLOWED_DOC_URL_PREFIXES = ("https://iva.ru/",)`). Формат блока:
    `{"type":"link","text":<title>,"url":<url>}`; перед ними — 1 text-блок «Источники:».
  - Без citations → blocks=[] (message без источников).
- Чистые функции, без I/O — легко тестируются.

### 3. Infrastructure — `infrastructure/bot_support/client.py`
- `async def send_message(*, base_url, token, chat_room_id, message, blocks,
  reply_on_message_id, transport=None, timeout=8.0)`:
  POST `{base_url}/api/bot/v1/chats/{chat_room_id}/send-message`,
  заголовок `X-Iva-Bot-Api-Token: <token>`,
  body `{"message":..., "blocks":..., "replyOnMessageId":...}`.
  Таймауты; лог ошибок 401/5xx **без секрета**; при 5xx — не ретраим агрессивно
  (платформа сама ограничивает). `transport` инъектируется в тестах.

### 4. Router — `interface/api/routers/bot_support.py`
- Pydantic-модели `BotEvent`/вложенные (`data.chatRoom`, `data.author`), `extra=ignore`.
- `POST /api/bot/support/webhook` (prefix `/api/bot/support`), БЕЗ `_AUTH`:
  1. **Верификация** заголовка `Iva-Bot-Api-Secret-Token` против
     `settings.iva_bot_support_webhook_secret` через `hmac.compare_digest`.
     Секрет в конфиге не задан → **401 fail-closed**. Несовпал → 401.
  2. **Фильтр**: обрабатываем только `type=="CHAT_NEW_MESSAGE"` И
     `data.author.type != "BOT"`. Прочее (edit, сообщения бота) → **200 без действия**.
  3. **Быстрый ответ**: сразу `200`, обработка — в `BackgroundTasks`
     (docs_ask + send-message фоном; платформа блокирует webhook на 5 мин при задержке).
- Сиды для тестов (монки-патч): модульные `run_docs_ask(ctx, question) -> DocsAnswer`
  (строит `DocsAssistant` из ctx, зовёт `ask` через `run_in_threadpool`) и `send_message`.
- Контекст RAG#1 берём тем же кэшем `app.state.docs_assistant_context`
  (через `build_docs_assistant_context`) — RAG-логику НЕ дублируем.
- ctx=None (enabled, но ассистент не сконфигурирован) → 200 + лог, отправку пропускаем.

### 5. Монтаж (`main.py`)
- `if _boot_settings.iva_bot_support_enabled: app.include_router(bot_support.router)`
  — БЕЗ `_AUTH`, за флагом (дефолт off → без токенов не активен).

### 6. Тесты
- `tests/application/test_bot_support.py`: build_bot_reply — blocks из citations,
  allowlist (нефильтрованный/чужой url отбрасывается), not_found → вежливый ответ без blocks.
- `tests/interface/test_bot_support.py`: парсинг BotEvent; фильтр author=BOT (200, send
  НЕ зван); неверный/пустой секрет → 401; happy-path → 200 и `send_message` вызван с
  ожидаемым body (монки-патч `run_docs_ask` + `send_message`, реальную платформу НЕ зовём).
  Роут за флагом — включаем через отдельный app/фикстуру или проверку include.
- `tests/infrastructure/test_bot_support_client.py`: send_message с `httpx.MockTransport`
  — проверяем URL, заголовок токена, body; 401/5xx не роняют (лог без секрета).

## Definition of Done
Роутер+логика+конфиг+тесты в worktree; тесты зелёные; ruff+mypy чисто; без хардкод-
секретов; роут за флагом (дефолт off); НЕ мержено/не пушено. Хендофф SUMMARY + пинг.

## Открытые вопросы (нужно решение тимлида)
1. **blocks-формат**: подтверждён `{"type":"link","text":...,"url":...}` +
   ведущий `{"type":"text","text":"Источники:"}`. Класть ли сам `answer.text` ещё и
   text-блоком, или достаточно поля `message`? → предлагаю только `message`, blocks —
   лишь источники.
2. **Allowlist url**: жёстко `https://iva.ru/` как единственный публичный префикс
   доки. Ок? (RAG#1 корпус и так только публичная дока — это belt-and-suspenders.)
3. Флаг включения в тестах interface: поднять отдельный `app` с
   `iva_bot_support_enabled=True` через monkeypatch Settings, либо всегда монтировать
   роут, а гейтить только логику. → предлагаю монтаж за флагом + тест-фикстура,
   форсящая флаг (переопределение через отдельный include в тесте).