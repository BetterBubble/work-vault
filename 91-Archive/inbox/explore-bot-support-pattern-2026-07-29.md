---
title: Разведка — конструкция бота «Поддержка» (helm) как образец для бота выдачи
  доступа
type: note
status: draft
role: explorer
tags:
- explorer
- iva-write
- bot
- helm
permalink: tacticum/00-board/explore-bot-support-pattern-2026-07-29-1-1
archived-at: 2026-08-05 15:19
---

Репо `~/tacticum/helm`, ветка `main`, HEAD `5c52811` (дерево чистое). Только чтение.
Каждое утверждение — файл:строка. Значения секретов не приводятся, только имена переменных.

## 1. Контракт вебхука

**Путь/метод:** `POST /api/bot/support/webhook` — префикс роутера `/api/bot/support`
(`src/helm/interface/api/routers/bot_support.py:44`), ручка `@router.post("/webhook")`
(там же:225-231). Тег OpenAPI `bot-support`.

**Тело запроса** — pydantic-модели, все с `ConfigDict(extra="ignore")` и дефолтами
(ни одно поле не обязательно, битый payload не даёт 422):

- `BotEvent` (bot_support.py:80-86): `type: str = ""`, `data: _BotEventData`
- `_BotEventData` (:69-77): `messageId: str`, `chatRoom: _BotChatRoom`, `author: _BotAuthor`, `message: str`
- `_BotChatRoom` (:59-66): `chatRoomId: str`, `chatRoomName: str`, `isGroupChat: bool = False`
- `_BotAuthor` (:49-55): `type: str` (`REGISTERED_USER | GUEST_USER | BOT`), `name: str`, `login: str`

**Ответ:** `dict[str, str]` — `{"status": "accepted"}` (:277) либо `{"status": "ignored"}`
(:252, :257, :263). Ошибка одна: `HTTPException(401, "неверный секрет webhook")` (:243).
Схемы ответа как pydantic-модели нет — возвращается голый dict.

**Проверка секрета** — `bot_support.py:240-243`:
`hmac.compare_digest(iva_secret, expected)`, constant-time. Заголовок принимается через
`Header(default=None, alias="Iva-Bot-Api-Secret-Token")` (:230). Fail-closed тройным
условием: секрет не задан в конфиге ИЛИ заголовка нет ИЛИ не совпал → 401. Импорт `hmac`
на :22.

**Почему вне `_AUTH`:** в `src/helm/main.py:174-175` роутер включается
`app.include_router(bot_support.router)` — без `dependencies=_AUTH` (`_AUTH = [Depends(require_user)]`,
main.py:112). Комментарий main.py:171-173: аутентификация своя (секрет-заголовок платформы,
не project-hub токен). Дополнительно роутер вообще не монтируется, если выключен флаг:
`if _boot_settings.iva_bot_support_enabled:` (main.py:174), где `_boot_settings = Settings()`
читается один раз на импорте модуля (main.py:97). Тот же приём «свой гейт → без `_AUTH`»
применён к `hrd.router` (main.py:168-169) и `docs.router` (main.py:141).

## 2. Обратный канал к боту

`src/helm/infrastructure/bot_support/client.py`, единственная функция `send_message` (:22-59),
async, keyword-only аргументы.

- URL: `f"{base_url.rstrip('/')}/api/bot/v1/chats/{chat_room_id}/send-message"` (:38);
  `base_url` — из `settings.iva_bot_base_url` (вызов в роутере bot_support.py:214).
- Метод POST, тело `{"message", "blocks", "replyOnMessageId"}` (:39-43).
- Заголовок авторизации `X-Iva-Bot-Api-Token: <token>` (:47), токен из
  `settings.iva_bot_support_token` (bot_support.py:215).
- **Адресат — только `chatRoomId` из входящего события** (bot_support.py:255 → :218).
  Ни почты, ни userId в исходящем вызове нет. Тред-ответ — `replyOnMessageId = data.messageId`.
- Таймаут: параметр `timeout: float = 8.0` (:31), передаётся в `httpx.AsyncClient` (:45).
- Ошибки: `httpx.HTTPError` (сеть/таймаут) → warning без токена, `return False` (:49-51);
  HTTP ≥400 → warning с кодом, `return False` (:53-58). Наружу не бросает.
- **Ретраев нет** — сознательно (docstring :6-8: платформа сама ограничивает переотправку).
- `transport: httpx.AsyncBaseTransport | None` (:30) — точка инъекции `httpx.MockTransport`
  в тестах; в проде None.

## 3. Слой application и что приходит в payload

`src/helm/application/bot_support.py` — **чистая функция без I/O**: `build_bot_reply(answer: DocsAnswer) -> BotReply`
(:163-183). Она не разбирает входящее сообщение — она превращает ответ RAG#1 в payload
платформы: чистка markdown (`_process_markdown`, :91-117), сбор источников с дедупом
(`_collect_sources`, :129-160), allowlist публичных url `("https://iva.ru/",)` (:43).
`BotReply` — frozen dataclass `message: str` + `blocks: list[dict[str,str]]` (:76-81).

Разбор входящего живёт в **роутере**, не в application (bot_support.py:246-257).

**Состояние диалога.** Без флагов — каждый запрос независим (bot_support.py:143-146:
если не (`docs_clarify_enabled` или `docs_conversation_context_enabled`) или нет
`session_factory` → просто `run_docs_ask`). С флагами — состояние в БД через
`resolve_with_clarify` (`src/helm/interface/api/docs_clarify.py:35-136`), канал `"bot"`,
`conversation_key = chatRoomId` (bot_support.py:163-166).

**КТО написал — ключевой ответ.** Почты пользователя в разобранном payload НЕТ.
- Модель автора парсит только `type`, `name`, `login` (bot_support.py:54-56), и **ни одно
  из этих полей нигде не используется**, кроме проверки `data.author.type == "BOT"` (:249).
- Идентификатор адресата во всём потоке один — `chatRoomId` (:255), он же ключ диалога
  и адрес отправки.
- По спеке платформы (сверено с заметкой `20-Architecture/Бот «Поддержка» — webhook RAG-1
  в чат IVA (задеплоен).md:22`, там контракт подтверждён по openapi) платформа шлёт
  `author: {profileId, type, name, login}` — то есть **есть `profileId` и `login`**, но
  `profileId` наша модель даже не объявляет (`extra="ignore"` его молча роняет).
- **Является ли `login` почтой — не проверял**, в коде это просто строка, ни одного примера
  реального значения в репозитории нет. В тестовом фикстур-событии автор задан как
  `{"type": ..., "name": "Иван"}`, без `login` (`tests/interface/test_bot_support.py:41`).
- Локальный срез openapi платформы `tests/data/api/real-bot-subset.json` содержит только
  три пути (`/chats/{chatRoomId}/files/create`, `/chats/{chatRoomId}/send-message`,
  `/botEventsChannel/chats`) и **пустой `components.schemas`** — схемы `BotEvent`/автора там нет.
  Файл ни на что в коде не ссылается (grep по `tests/`, `src/` — ноль вхождений).
- Замечание: в этом срезе `servers[0].url = "/api/rest/bot"` (stage), а наш клиент ходит в
  `/api/bot/v1/...`. Расхождение не проверял; заметка в vault утверждает, что боевая база — `/api/bot/v1`.

**Вывод для бота выдачи доступа:** из текущего контракта персональная почта НЕ выводится.
Варианты (все требуют внешней проверки, не кода): расширить модель автора на `profileId`/`login`
и подтвердить у платформы, что `login` = корпоративная почта; либо резолвить профиль по
`profileId` через API платформы (такого эндпоинта в локальном срезе openapi нет); либо
спрашивать почту у пользователя в диалоге.

## 4. Настройки (`src/helm/config.py`)

`Settings(BaseSettings)`, `env_prefix="HELM_"`, `env_file=".env"`, `extra="ignore"` (config.py:12-15).
Блок бота — config.py:496-508, все четыре поля с дефолтами (обязательных нет):

| Поле | Дефолт | Env | Смысл |
|---|---|---|---|
| `iva_bot_support_enabled` | `False` | `HELM_IVA_BOT_SUPPORT_ENABLED` | Флаг монтирования роута (config.py:499) |
| `iva_bot_base_url` | `"https://iva-uc.ru"` | `HELM_IVA_BOT_BASE_URL` | База API платформы (config.py:501) |
| `iva_bot_support_token` | `None` | `HELM_IVA_BOT_SUPPORT_TOKEN` | Bot-токен исходящих (config.py:504) |
| `iva_bot_support_webhook_secret` | `None` | `HELM_IVA_BOT_SUPPORT_WEBHOOK_SECRET` | Секрет входящего вебхука (config.py:508) |

Поведение без токена:
- **Нет `iva_bot_support_webhook_secret`** → любой запрос 401 (bot_support.py:242), fail-closed.
- **Нет `iva_bot_support_token`** → вебхук отвечает 200 `accepted`, RAG#1 отрабатывает,
  а на отправке фон логирует `logger.warning("bot-токен не задан — ответ в chat=%s не отправлен")`
  и выходит (bot_support.py:210-212). Тихая деградация, не ошибка.
- **Нет `iva_bot_support_enabled`** → роут не смонтирован вовсе, 404 (main.py:174).

Смежные флаги диалога (config.py:177-205): `docs_clarify_enabled=False`,
`docs_clarify_tau_answer=0.55`, `docs_clarify_tau_floor=0.30`, `docs_clarify_ttl_seconds=900`,
`docs_conversation_context_enabled=False`, `docs_conversation_context_turns=3`,
`docs_conversation_context_ttl=1800`.

## 5. Как писать такой роутер по местным правилам

**Слои** (по факту этого бота):
- `interface/api/routers/<name>.py` — роутер: pydantic-схемы запроса живут **прямо в файле
  роутера** (bot_support.py:49-86, приватные с `_`, публичная только `BotEvent`), проверка
  аутентификации, фильтры, планирование фона.
- `interface/api/<shared>.py` — разделяемая между каналами логика с БД (`docs_clarify.py`),
  лежит в interface, не в application.
- `application/<name>.py` — чистые преобразования без I/O (`build_bot_reply`).
- `infrastructure/<name>/client.py` — исходящий HTTP, `__init__.py` с однострочным docstring
  (`src/helm/infrastructure/bot_support/__init__.py`).
- Регистрация — одна строка в `main.py` рядом с соседями, с комментарием, почему гейт такой.

**Ошибки:** `raise HTTPException(status.HTTP_401_UNAUTHORIZED, "...")` прямо в ручке.
Глобальный обработчик в приложении один — на доменный `InputValidationError` → 422
(main.py:217-229); к боту он не применяется.

**Фон:** `BackgroundTasks.add_task` (bot_support.py:268-276) — быстрый 200, т.к. платформа
блокирует вебхук на 5 минут при задержке/ошибке (docstring :13-14). Синхронный CPU/LLM-код
внутри фона — через `starlette.concurrency.run_in_threadpool` (:146, :156).
Фоновая функция ловит `Exception` целиком и логирует (`:221-222`) — фон падать не должен.

**Сессия БД в фоне:** request-scope там нет, поэтому берут фабрику из
`request.app.state.session_factory` (bot_support.py:267; ставится в lifespan, main.py:80) и
открывают короткую сессию сами с явным `commit`/`rollback` (bot_support.py:160-179).

**Кэш тяжёлого контекста:** ленивая сборка в `request.app.state` через сентинел `_UNSET`
(bot_support.py:46, :89-101) — чтобы `None` (не сконфигурировано) отличался от «ещё не строили».

**Логирование:** `logger = logging.getLogger(__name__)` в каждом модуле (роутер :42, клиент :19),
формат %-args, по-русски, `logger.warning` для ожидаемых сбоев, `logger.exception` для
неожиданных. Жёсткое правило: секрет/токен в лог не попадает никогда — логируют только
`chat_room_id` и HTTP-код (client.py:37, :54).

**Метрик нет** — prometheus/OTel в проекте не используется (grep по `src/` — ноль).

**Стиль:** `from __future__ import annotations` в каждом файле, ruff `line-length = 100`,
mypy `strict = true` (pyproject.toml:45, :61), развёрнутые русские docstring'и модуля с
описанием потока и мотивации решений.

## 6. Тесты как образец

Три файла, по слоям:

- `tests/interface/test_bot_support.py` — **своё приложение, не общий `app`**: фикстура
  `client` (:46-75) собирает `FastAPI()`, делает `app.include_router(bot_support.router)`,
  кладёт `app.state.settings = Settings(...)` с тестовыми секретами (:65-69) и
  `app.state.docs_assistant_context = object()` (:71). Причина в docstring (:3-6): в основном
  `app` роут за флагом выключен.
  Клиент — `httpx.AsyncClient(transport=ASGITransport(app=app))`, не `TestClient` (:73-75);
  тесты `@pytest.mark.asyncio`. **Важно: фоновая задача успевает отработать в рамках
  ASGI-цикла**, поэтому проверка `send_message` идёт сразу после ответа (:88-89).
  Моки — `monkeypatch.setattr(bot_support, "run_docs_ask", fake_ask)` и
  `monkeypatch.setattr(bot_support, "send_message", fake_send)` (:60-61); `fake_send`
  складывает kwargs в список `sent`, который фикстура отдаёт вторым элементом кортежа.
  Хелпер `_event(...)` (:26-43) строит payload с параметрами.
  Негативные пути покрыты все: неверный секрет 401 (:105), нет заголовка 401 (:117),
  автор BOT (:125), групповой чат (:138), `CHAT_EDIT_MESSAGE` (:151), пустое сообщение (:164) —
  и во всех `assert sent == []`.
- `tests/application/test_bot_support.py` — 20 чистых юнит-тестов на `_clean_text` /
  `_process_markdown` / `build_bot_reply`, без фикстур и без сети.
- `tests/infrastructure/test_bot_support_client.py` — `httpx.MockTransport` с handler'ом,
  который снимает url/заголовок/тело (:17-21); проверка точного URL, `X-Iva-Bot-Api-Token`
  и body (:33-39); отдельно 401/500 (:43-54) и `httpx.ConnectError` (:58-71) → `ok is False`.

**Общие фикстуры:** единственный conftest — `tests/interface/conftest.py` (155 строк, корневого
нет). Даёт `session_factory` (async sqlite in-memory, `Base.metadata.create_all`, :109-118),
`seeded_session_factory` (:121-129) и `client` (общий `helm.main.app` + `dependency_overrides`
для `get_session`/`current_today`, :132-155). Бот-тест этот общий `client` **перекрывает своей
одноимённой фикстурой** — важно не спутать. Пример теста с реальным sqlite-стором —
`tests/interface/test_docs_clarify.py` (использует `session_factory`, канал `"bot"`, скриптованный
`_ScriptedAsk` вместо ассистента).

## 7. Миграции и БД

Сам бот поддержки ничего не хранит: в базовом режиме (оба флага диалога выключены) путь
`webhook → docs_ask → send_message` не касается БД вовсе (bot_support.py:145-146).

Хранение появляется только при `docs_clarify_enabled` / `docs_conversation_context_enabled` —
и это не таблицы бота, а общий стор RAG#1, разделяемый с вебом через колонку `channel`:
- `docs_clarify_pending` — модель `DocsClarifyPending` (`src/helm/infrastructure/db/models.py:2768-2803`),
  UNIQUE `(channel, conversation_key)`, поля `original_question`, `clarify_question`,
  `ask_count`, `last_message_id`, `expires_at`. Миграция `alembic/versions/d2c3b4a5e6f7_docs_clarify_pending.py`
  (revision `d2c3b4a5e6f7`, down `ab12cd34ef56`), написана вручную, без autogenerate.
- `docs_conversation_turn` — `DocsConversationTurn` (models.py:2806-2844), append-only, два
  индекса; миграция `alembic/versions/f4e5d6c7b8a9_docs_conversation_turn.py`.
Для бота `channel="bot"`, `conversation_key=chatRoomId` (bot_support.py:163-166).
Каталог миграций — `alembic/versions/` (97 файлов), стиль: ручной upgrade/downgrade,
docstring с мотивацией, `revision`/`down_revision` строками.

## Риски и открытые вопросы для нового бота

1. **Почты в payload нет.** Без ответа платформы про `login`/`profileId` персональную ссылку
   генерировать не из чего. Это блокирующий вопрос, а не деталь реализации.
2. `chatRoomId` — идентификатор комнаты, а не человека. В личном чате он де-факто соответствует
   собеседнику, но это допущение, а не гарантия контракта.
3. Бот-токен на каждого бота свой (`HELM_IVA_BOT_SUPPORT_TOKEN` — для «Поддержки»); новому боту
   понадобятся свои `*_TOKEN` и `*_WEBHOOK_SECRET`, флаг и base_url можно переиспользовать.
4. Ретраев нет by design: единичный сбой отправки = ответ потерян молча. Для выдачи доступа
   (действие, а не справка) молчаливая потеря — совсем другая цена ошибки, чем у справочного бота.
5. Аутентификация вебхука подтверждает платформу, но **не пользователя**: секрет общий на
   вебхук. Любой, кто пишет боту, для нас одинаково анонимен.
6. `extra="ignore"` на всех моделях означает: платформа может прислать поле, а мы его молча
   потеряем — при добавлении полей их надо явно объявлять.