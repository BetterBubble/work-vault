---
title: verify-iva-write-branch-2026-07-30
type: note
status: draft
created: 2026-07-30 12:00
updated: 2026-07-30 12:00
permalink: tacticum/00-board/verify-iva-write-branch-2026-07-30
tags:
- board
- iva-write
- verify
---

# Проверка ветки feat/iva-write-keystore против свежего origin/main (30.07.2026)

Проверял verifier, только чтение. Дерево: `/Users/bubblemac/tacticum-worktrees/helm-iva-write-keystore`,
ветка `feat/iva-write-keystore`, репозиторий helm. merge-base = `5c52811e`. git 2.50.1.
Состояние дерева на входе — чистое, ничего не менял.

## 1. Конфликты с origin/main — ОДИН, тривиальный

`git merge-tree --write-tree origin/main HEAD` → exit 1, конфликтный файл ровно один:

- **CONFLICT (content): `src/helm/infrastructure/db/models.py`**
- auto-merge прошёл: `docker-compose.prod.yml`, `src/helm/main.py`, `tests/infrastructure/test_models_metadata.py`

Пересечение по файлам: наших изменений 42 файла, у origin/main 53, пересекаются **4**:
`docker-compose.prod.yml`, `src/helm/infrastructure/db/models.py`, `src/helm/main.py`,
`tests/infrastructure/test_models_metadata.py`.

**Природа конфликта.** Трёхсторонний прогон (`git merge-file --diff3`) даёт **один хунк** в конце
файла: база пустая, обе стороны ДОПИСАЛИ новые классы в хвост `models.py`.
Наши: `ExternalCredential`, `CredentialUseLog`, oauth_state, bot-диалоги (+234 строки).
Их: `BotTask`, `SalesFile`, `IntakeCandidate` и др. (+160 строк). Пересечения по именам классов
и колонок нет. Импорт `LargeBinary` добавили обе стороны одинаково — смержился автоматически.
Разрешение: оставить оба блока подряд, порядок безразличен.

**Вердикт:** мерж/ребейз пройдёт с одним конфликтом «оба дописали в конец файла», механическое
разрешение, ~2 минуты, риска логики нет.

## 2. НАСТОЯЩАЯ проблема мержа — не конфликт git, а ДВЕ головы alembic

Обе стороны ветвятся от одной ревизии `hrd224_allure_activity`:

- наша цепочка: `key300_external_credential` → `oa310_iva_oauth_flow` → `bot330_bot_iva_write` → `idn340_identity_binding`
- origin/main: `pmb208_bot_task` → … → `pmb216_pregate_decision` (9 ревизий)

После мержа в дереве будет **две головы** (`idn340_identity_binding` и `pmb216_pregate_decision`).
git этого не покажет — файлы разные, конфликта нет. Чинится либо merge-ревизией alembic, либо
перевешиванием `down_revision = "pmb216_pregate_decision"` у `key300_external_credential`.
Это обязательный шаг перед деплоем, иначе `alembic upgrade head` упадёт на множественных головах.

## 3. Тесты на ветке — ПОДТВЕРЖДЕНО

`uv run pytest -q` (без Makefile; способ из README.md:96):

```
2371 passed, 32 skipped, 61 warnings in 73.12s
```

**0 падений.** Заявленное число 2371 совпадает точно. 32 скипа и 61 варнинг — фон
(в варнингах есть шум aiosqlite «Event loop is closed» на закрытии, тесты это не роняет).

`uv run alembic heads` → **`idn340_identity_binding (head)`**, голова одна. Заявление подтверждено —
но только ДЛЯ ВЕТКИ В ЕЁ ТЕКУЩЕМ ВИДЕ, см. п.2: после мержа их станет две.

## 4. Что физически нужно руками после прихода ключей

Префикс всех переменных `HELM_` (`src/helm/config.py:15`). Перечень из `.env.example` (54 строки,
коммит 6b32e2c) сверен с кодом.

### Обязательные, без них канал не работает

| Переменная | Что кладут | Откуда берётся | Нет значения → |
|---|---|---|---|
| `HELM_KEYSTORE_MASTER_KEY` | base64 от 32 байт (AES-256) | **генерим сами**: `openssl rand -base64 32` | сервис СТАРТУЕТ, но ручки `/api/iva/oauth/*` и `/mcp/iva-write` отдают 503 «нет мастер-ключа» |
| `HELM_IVA_OAUTH_JIRA_CLIENT_ID` | client_id incoming-приложения Jira DC | **админ Atlassian** | система выпадает из `configured_systems` → согласие по Jira 503 |
| `HELM_IVA_OAUTH_JIRA_CLIENT_SECRET` | client_secret того же приложения | **админ Atlassian** | то же |
| `HELM_IVA_OAUTH_CONFLUENCE_CLIENT_ID` | client_id приложения Confluence DC (регистрация отдельная!) | **админ Atlassian** | согласие по Confluence 503 |
| `HELM_IVA_OAUTH_CONFLUENCE_CLIENT_SECRET` | client_secret | **админ Atlassian** | то же |
| `HELM_IVA_OAUTH_PUBLIC_BASE_URL` | `https://helm.tacticum.ru` | **уже известно**, но обязано побуквенно совпасть с redirect_uri в регистрации | старт согласия и колбэк → 503 «не задан публичный адрес» |
| `HELM_IVA_OAUTH_JIRA_BASE_URL` / `_CONFLUENCE_BASE_URL` | `https://jira.iva.ru`, `https://wiki.iva.ru` | **уже известно** | система не сконфигурирована → 503 |

Полуконфигурация (адрес есть, секрета нет) в набор НЕ попадает намеренно —
`IvaOAuthClient.from_settings`, фильтр `if base_url and client_id and client_secret`.

### Есть дефолт, трогать по ситуации

| Переменная | Дефолт | Комментарий |
|---|---|---|
| `HELM_IVA_OAUTH_JIRA_CONNECT_URL` / `_CONFLUENCE_CONNECT_URL` | None | локальный конец SSH-туннеля `https://172.18.0.1:8443`; для прямого доступа не нужен |
| `HELM_IVA_OAUTH_PKCE_ENABLED` | `true` | выключать, только если админ ответил, что сервер PKCE не держит |
| `HELM_IVA_OAUTH_SCOPE` | `WRITE` | грубые уровни Atlassian DC |
| `HELM_IVA_OAUTH_VERIFY_TLS` | `True` | False — только для своего trial с самоподписанным сертификатом |
| `HELM_IVA_OAUTH_STATE_TTL_SECONDS` / `_TIMEOUT_S` / `_REFRESH_HORIZON_SECONDS` | 600 / 15.0 / 3600 | |

### Бот выдачи доступа — ключи от команды ботов, всё выключено по умолчанию

| Переменная | Дефолт | Нет значения → |
|---|---|---|
| `HELM_IVA_BOT_IVA_WRITE_ENABLED` | `false` | роут webhook НЕ монтируется (`main.py:203`) |
| `HELM_IVA_BOT_IVA_WRITE_TOKEN` | None | ответ в чат отправить нечем |
| `HELM_IVA_BOT_IVA_WRITE_WEBHOOK_SECRET` | None | webhook отвечает 401 на всё, fail-closed (`bot_iva_write.py:350-353`) |
| `HELM_IVA_BOT_BASE_URL` | `https://iva-uc.ru` | общий с ботом «Поддержка»; токены у ботов РАЗНЫЕ |
| TTL диалога / попытки / пауза | 900 / 3 / 2.0 | |

Без бота канал работает: ссылку на согласие можно отдавать руками.

### Внутренняя выдача доступа шлюзу — нужна, только если канал вынесут из helm

`HELM_IVA_WRITE_INTERNAL_ENABLED` (false → роут не монтируется), `HELM_IVA_WRITE_SERVICE_KEY`
(None → 401 fail-closed). При топологии «всё в helm» остаются выключенными.

### Адреса канала записи — уже в docker-compose.prod.yml, в .env не задаются

`HELM_IVA_WRITE_MCP_URL=http://mcp-atlassian:9000/mcp`, `HELM_IVA_WRITE_JIRA_URL=https://jira.iva.ru:8443`,
`HELM_IVA_WRITE_CONFLUENCE_URL=https://wiki.iva.ru:8443` (коммит c02e63a). Пусто → поверхность 503,
а не 404. Контейнер `mcp-atlassian:0.23.0` поднимается с ПУСТЫМ окружением по учётным данным —
это условие безопасности, туда ничего не дописывать.

### Неточность в .env.example

Строка 11: «Обязательно. Без него сервис НЕ СТАРТУЕТ». По коду это неверно: `keystore_crypto()`
вызывается лениво из роутеров (`iva_write.py:161`, `iva_internal.py:107`, `iva_write_surface.py:144`),
в `main.py` при старте не дёргается. Без ключа сервис поднимется, а канал iva-write будет отдавать
503. Формулировку в примере стоит поправить — оператор может решить, что отсутствие ключа заметит
по упавшему контейнеру, а он не упадёт.

## Итог

1. Конфликт один и механический (`models.py`, оба дописали в хвост) — не блокер.
2. Реальный блокер мержа — две головы alembic после слияния, чинится merge-ревизией или
   перевешиванием `down_revision`.
3. Тесты: 2371 passed / 0 failed / 32 skipped — заявление подтверждено. Голова alembic на ветке одна.
4. Руками после ключей: 4 значения от админа Atlassian + 1 генерим сами (мастер-ключ) +
   регистрация redirect_uri `https://helm.tacticum.ru/api/iva/oauth/callback` побуквенно.
   Бот и внутренняя ручка — опциональны, оба fail-closed по умолчанию.
