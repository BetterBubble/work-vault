---
title: SUMMARY — webhook бота «Поддержка» (RAG#1 → чат IVA)
type: handoff
tenant: iva
worktree: ~/tacticum-worktrees/helm-bot-webhook (feat/bot-support-webhook)
date: 2026-07-16
permalink: tacticum/00-inbox/bot-webhook-summary-1
status: archived
updated: 2026-07-18
---

# SUMMARY: webhook бота «Поддержка» — готово в worktree

Реализовано по одобренному плану + дополнения тимлида (групповые чаты, отказоустойчивость
фона, allowlist fail-closed). **Не мержено, не пушено.**

## Что сделано

### Новые файлы
- `src/helm/application/bot_support.py` — чистая `build_bot_reply(DocsAnswer) -> BotReply`:
  `message` = текст ответа (с [n]); `blocks` = link-блоки ТОЛЬКО из публичных doc-цитат,
  прошедших allowlist `https://iva.ru/` (url вне allowlist/пусто/None → блок НЕ добавляем,
  fail-closed). `not_found` → честный текст отказа, `blocks=[]`. Нет источников после
  фильтра → текст всё равно уходит без блоков.
- `src/helm/infrastructure/bot_support/client.py` — `async send_message(...)`:
  POST `{base}/api/bot/v1/chats/{chatRoomId}/send-message`, заголовок `X-Iva-Bot-Api-Token`,
  body `{message, blocks, replyOnMessageId}`. Отказоустойчиво: сеть/401/5xx → лог БЕЗ
  секрета + `return False`, без ретрая. `transport` инъектируется в тестах.
- `src/helm/interface/api/routers/bot_support.py` — `POST /api/bot/support/webhook`:
  1. Верификация `Iva-Bot-Api-Secret-Token` через `hmac.compare_digest`; секрет не задан
     в конфиге → **401 fail-closed**; несовпал/пусто → 401.
  2. Фильтр: обрабатываем только `CHAT_NEW_MESSAGE` + `author.type != BOT` +
     **личный чат** (`isGroupChat == false`); прочее (edit / бот / группа / пустой текст)
     → 200 `{"status":"ignored"}`.
  3. Быстрый 200 `{"status":"accepted"}`, а docs_ask + send-message — в `BackgroundTasks`.
     RAG#1 берётся из общего кэша `app.state.docs_assistant_context` (как `/api/docs/ask`),
     логика не дублирована. Фон отказоустойчив (лог без секрета, без крашей/ретраев).
     ctx=None (не сконфигурирован) → 200 ignored + лог.
  - Сиды `run_docs_ask` / `send_message` — модульные, для монки-патча в тестах.

### Правки
- `src/helm/config.py` — +4 поля Settings (env `HELM_*`): `iva_bot_support_enabled=False`,
  `iva_bot_base_url="https://iva-uc.ru"`, `iva_bot_support_token`,
  `iva_bot_support_webhook_secret` (последние два — `| None = None`, только из env).
- `src/helm/main.py` — импорт `bot_support` + монтаж за флагом БЕЗ `_AUTH`:
  `if _boot_settings.iva_bot_support_enabled: app.include_router(bot_support.router)`.

### Тесты (новые)
- `tests/application/test_bot_support.py` (5): blocks из публичных цитат, allowlist
  отбрасывает внутренний/чужой/пустой url, not_found без блоков, fallback-текст.
- `tests/infrastructure/test_bot_support_client.py` (3): URL/заголовок токена/body;
  401/5xx и сетевая ошибка → `False` без исключения.
- `tests/interface/test_bot_support.py` (7): happy-path (200 accepted + фон send с
  ожидаемым body), неверный/отсутствующий секрет → 401, author=BOT / группа / edit /
  пустой текст → ignored, send НЕ зван. Платформа не вызывается (монки-патч).
  Роут за флагом — поднят отдельный `FastAPI` с `include_router` (как включил бы прод).

## Проверки (все зелёные)
- `uv run pytest` — **1447 passed, 12 skipped** (регресса нет; новых тестов 15).
- `ruff check` по всем изменённым/новым файлам — **All checks passed**.
- `mypy` по новому коду — **Success, no issues**.
- Хардкод-секретов нет (grep чист); роут off по дефолту (проверено: route не смонтирован
  при флаге off, `token`/`secret` = None без env).

## Definition of Done — выполнено
Роутер+логика+конфиг+тесты в worktree; тесты зелёные; ruff+mypy чисто; секреты только из
env; роут за флагом (дефолт off); НЕ мержено/не пушено.

## На заметку (для интеграции/следующего шага)
- Прод-включение: `HELM_IVA_BOT_SUPPORT_ENABLED=true` + `HELM_IVA_BOT_SUPPORT_TOKEN` +
  `HELM_IVA_BOT_SUPPORT_WEBHOOK_SECRET` в env. Ассистент доков должен быть сконфигурирован
  (иначе webhook отвечает 200 ignored и молчит).
- MVP отвечает только в личных чатах. Ответ на @упоминание в группе — отдельный follow-up
  (контракт mention пока не подтверждён).