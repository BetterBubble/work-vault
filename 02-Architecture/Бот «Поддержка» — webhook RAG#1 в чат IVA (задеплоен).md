---
title: Бот «Поддержка» — webhook RAG#1 в чат IVA (задеплоен)
type: note
permalink: tacticum/02-architecture/bot-podderzhka-webhook-rag-1-v-chat-iva-zadeploen
tags:
- bot
- support
- webhook
- rag1
- helm
- deploy
- iva
- integration
---

## Что это
Первый исходящий чат-бот IVA — **«Поддержка»**: клиент пишет боту → мы отвечаем по **публичной документации (RAG#1, `docs_ask`)** с цитатами. Реализован как webhook-сервис в helm, **задеплоен на прод** (2026-07-16), ждёт регистрации URL на платформе + боевого теста.

## Контракт (платформа IVA, подтверждён по openapi)
- Публичная спека бота: `https://iva-uc.ru/api/docs/bot/openapi.yaml` (сама страница `/api/docs/bot` — Swagger SPA; `/api/openapi.json` за 401).
- Base API бота: `https://iva-uc.ru/api/bot/v1`.
- **Входящее (платформа → наш webhook), POST BotEvent:** `{type:"CHAT_NEW_MESSAGE"|"CHAT_EDIT_MESSAGE", data:{messageId, chatRoom:{chatRoomId,chatRoomName,isGroupChat}, author:{profileId,type:"REGISTERED_USER|GUEST_USER|BOT",name,login}, message, createdAt}}`. Заголовок `Iva-Bot-Api-Secret-Token` — секрет верификации (задаём мы).
- **Исходящее (ответ):** `POST {base}/api/bot/v1/chats/{chatRoomId}/send-message`, заголовок `X-Iva-Bot-Api-Token: <botToken>`, body `{message, blocks:[{type:"text"|"link",text,url}], replyOnMessageId}`.
- У бота **нет read-only эндпоинта** для валидации токена → токен подтверждается только реальной отправкой в чат.
- Надёжность: ошибка/задержка нашего webhook → платформа блокирует его на 5 мин; 502/503/504 → лимит переотправок. Поэтому отвечаем **200 сразу**, обработку — в фоне.

## Реализация (helm, PR #61, main `7fd134d`)
Ветка `feat/bot-support-webhook` → merged. Файлы:
- `interface/api/routers/bot_support.py` — `POST /api/bot/support/webhook`, БЕЗ общего `require_user`; верификация секрета `hmac.compare_digest` (нет секрета в конфиге → **401 fail-closed**); фильтр **`CHAT_NEW_MESSAGE` + `author.type!="BOT"` + `isGroupChat==false`** (в группах не спамит, себе не отвечает); быстрый 200 + `BackgroundTasks`; фон отказоустойчив (лог без секрета, без ретраев/дублей).
- `application/bot_support.py` — `build_bot_reply`: `message`=текст ответа; `blocks`=link-цитаты **только через allowlist `https://iva.ru/`** (url вне allowlist/None → блок не добавляем, fail-closed); `not_found` → честный текст без блоков.
- `infrastructure/bot_support/client.py` — `send_message` (httpx, `X-Iva-Bot-Api-Token`, таймауты).
- `config.py` — 4 поля (env `HELM_IVA_BOT_*`), `main.py` — роут за флагом `iva_bot_support_enabled` (дефолт off).
- Тесты: 15 новых, весь прогон 1447 passed, ruff/mypy clean.

**Ключевой принцип — двухконтурность allowlist по источнику:** наружу клиенту физически может уйти только контент из публичной доки (RAG#1). Никакого RAG#2/внутреннего. Это база для [[support-mcp-concept]] (концепт support-MCP, `helm/docs/iva-support-mcp-concept.md`).

## Деплой (прод helm)
- Сервер `helm` (159.194.233.33), `/opt/helm` — git-репо, но **без git-доступа к GitHub** (deploy-key нет) → код доставлен **git-bundle с локали** (ff до main), т.к. `git pull` на сервере не работает.
- Env в `/opt/helm/.env`: `HELM_IVA_BOT_SUPPORT_TOKEN` (токен «Поддержка»), `HELM_IVA_BOT_SUPPORT_WEBHOOK_SECRET`, `HELM_IVA_BOT_SUPPORT_ENABLED=1`. **Значения секретов в память НЕ кладём** — токены 4 ботов лежат в `~/tacticum/.secrets/bots.env`; webhook-секрет в серверном `.env` и передан пользователю для платформы.
- Пересборка: `SEED=0 bash scripts/deploy.sh` (граф сохранён). helm-helm-1 пересоздан, postgres healthy, миграции head.
- **Боевой URL:** `https://helm.tacticum.ru/api/bot/support/webhook` (публичный, проверено извне).

## Проверки после деплоя (всё зелёное)
- `POST` без секрета → **401** (роут смонтирован, fail-closed).
- `POST` с верным секретом + автор BOT → **200 `{"status":"ignored"}`** (секрет ок, фильтр работает, клиенту не шлём).
- `/mcp/analyst` → **406** (MCP аналитиков жив после рестарта).

## Что осталось
1. **Отдать платформе** WebHook URL + `Iva-Bot-Api-Secret-Token` (у пользователя) → они впишут в настройки бота «Поддержка».
2. **Боевой end-to-end тест:** написать боту в личном чате → проверяет разом входящую достижимость (платформа `iva-uc.ru` → наш helm, ещё НЕ подтверждена), RAG#1-ответ, валидность токена доступа. Если платформа не достанет helm снаружи (контур ИВА) → нужен эндпоинт в контуре/туннель (вопрос к Медведеву — бэкендер бота).
3. **Остальные 3 бота** (Техподдержка, Sales/Продажка, Аналитики — токены в `.secrets`): своя логика/контур, дадим URL по мере готовности. Аналитики, вероятно, внутренний контур (не RAG#1).

## Связано
- [[support-mcp-concept]] — концепт support-MCP (тот же двухконтурный принцип).
- Достижимость: helm.tacticum.ru публичен из интернета; helm → iva-uc.ru работает (проверено); iva-uc.ru → helm не подтверждено.
