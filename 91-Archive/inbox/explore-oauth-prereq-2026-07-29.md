---
title: Предпосылки OAuth 2.0 к Jira/Confluence DC заказчика — разведка helm
type: note
status: draft
role: explorer
tags:
- explorer
- iva-write
- oauth
- helm
permalink: tacticum/00-board/explore-oauth-prereq-2026-07-29-1
archived-at: 2026-08-05 15:19
---

# Предпосылки OAuth 2.0 (Jira/Confluence DC) — что проверено в helm

Репозиторий `/Users/bubblemac/tacticum/helm`, HEAD `5c52811`. Прод-сервер `helm`
(159.194.233.33) — только чтение (`ssh-manager exec helm …`). Значения секретов не выводились.

## Вердикт коротко

- **П.1 — ДА, безусловно.** Публичный HTTPS-эндпоинт вида `/api/.../callback` добавляется
  БЕЗ правок прокси: Traefik роутит по `Host(...)`, без path-фильтра; сертификат Let's Encrypt
  валиден. Ограничение одно: путь `/callback` УЖЕ занят SPA (логин в project-hub) — новому
  колбэку нужен другой путь.
- **П.2 — ДА, проверено живым запросом.** helm ходит в Jira/Confluence через SSH-туннель;
  и GET, и POST доходят до приложения. У обоих подняты рабочие OAuth2-эндпоинты
  (`/rest/oauth2/latest/authorize` и `/token`), версии DC достаточные.
- **П.3 — НЕТ однозначного ответа. Это вопрос к заказчику.** Прямого подтверждения,
  что браузер сотрудника ИВА достаёт `helm.tacticum.ru`, нет. Есть сильные косвенные
  свидетельства ЗА (сотрудники ИВА пользуются публичным сервисом Tacticum с рабочих машин)
  и одно зафиксированное «не подтверждено» по соседнему направлению.

---

## 1. Публичная доступность helm

**Конфиг.** `/Users/bubblemac/tacticum/helm/docker-compose.prod.yml`:

- Traefik `v2.11`, порты `80:80` и `443:443` (:29-31), редирект `web → websecure` (:16-18),
  ACME httpchallenge, резолвер `le` (:20-23).
- Роутер приложения — `docker-compose.prod.yml:110-113`:
  `traefik.http.routers.helm.rule=Host(`helm.tacticum.ru`)`, `entrypoints=websecure`,
  `tls.certresolver=le`, backend `helm:8000`.
  **Правило только по Host — path-allowlist'а НЕТ.** Любой новый путь приложения
  автоматически публичен, правка прокси не нужна.
- Сам helm наружу не публикуется (`expose: "8000"`, :106-107), фронт `web/dist` отдаётся
  тем же процессом (`src/helm/main.py:263`) → same-origin, CORS не нужен.

**Живая проверка (с локальной машины):**

```
dig +short helm.tacticum.ru A            → 159.194.233.33
curl https://helm.tacticum.ru/api/auth/config → 200, tls verify ok
openssl s_client …                        → issuer Let's Encrypt CN=YR2,
                                            subject CN=helm.tacticum.ru,
                                            notAfter Oct 2 15:07:20 2026 GMT
curl -o/dev/null -w%{http_code} https://helm.tacticum.ru/api/oauth/does-not-exist → 404
```

Последняя строка важна: несуществующий путь отдаёт 404 ПРИЛОЖЕНИЯ, то есть весь префикс
уже проксируется — значит достаточно объявить роут в FastAPI.

**Контейнеры на сервере** (`docker ps`): `helm-traefik-1` (0.0.0.0:80,443), `helm-helm-1`
(8000/tcp, наружу не проброшен), `helm-postgres-1`.

**Что потребуется для нового колбэка:** только роут в FastAPI. Путь `/callback` занят —
`src/helm/main.py:238-243` отдаёт там `index.html` SPA (колбэк логина project-hub PKCE).
Брать надо свободный, напр. `/api/iva/oauth/callback` (под `/api/*` статик-маунт не перехватывает,
роутеры имеют приоритет — `main.py:231-234`).

## 2. Сетевой путь helm → Jira/Confluence

**Туннели на сервере** (`systemctl list-units | grep tunnel` — все active/running):

| Юнит | Что |
|---|---|
| `iva-sources-tunnel` | helm → adp-jump → `10.22.0.10:443` (jira/wiki) |
| `iva-contour-sources-tunnel` | beta `10.0.200.58:443` / distrohost `10.3.7.199:80` / eva `10.6.10.9:443` / eva-wiki `10.3.0.245:443` |
| `iva-allure-tunnel` | allure.iva.ru:443 |
| `iva-triva-tunnel` | vLLM `10.0.196.14:8004` |

`iva-sources-tunnel` в файле юнита (`/etc/systemd/system/iva-sources-tunnel.service`) биндит
только loopback, но drop-in `…service.d/gateway-bind.conf` переопределяет ExecStart и добавляет
bind на docker-gateway. Реальный cmdline процесса:

```
ssh -N -T … -L 127.0.0.1:8443:10.22.0.10:443 -L 172.18.0.1:8443:10.22.0.10:443 adp-jump
```

`ss -lntp` подтверждает LISTEN на `172.18.0.1:8443` и `127.0.0.1:8443`.

**Как это использует код.** `_tunnel_client_kwargs` —
`/Users/bubblemac/tacticum/helm/src/helm/ingest/rag2_extract.py:720-738`: физический адрес
подключения берётся из `connect_url`, а `Host`-заголовок и TLS-SNI — из hostname `base_url`
(jira.iva.ru и wiki.iva.ru — один IP за туннелем, различаются по Host/SNI). Сертификат
проверяется всегда, опции отключения нет (докстринг модуля, :34-38).

Env контейнера (`docker inspect helm-helm-1`, секреты отфильтрованы):
`HELM_RAG2_JIRA_CONNECT_URL=https://172.18.0.1:8443`,
`HELM_RAG2_CONFLUENCE_CONNECT_URL=https://172.18.0.1:8443`,
`HELM_RAG2_JIRA_BASE_URL=https://jira.iva.ru`, `HELM_RAG2_CONFLUENCE_BASE_URL=https://wiki.iva.ru`.

**Живая проверка ИЗ КОНТЕЙНЕРА** (`docker exec helm-helm-1 python -c …`, httpx на
`https://172.18.0.1:8443` с `Host`/`sni_hostname`):

```
jira GET  /rest/api/2/serverInfo        → 401 application/json   (дошло до приложения)
jira POST /rest/auth/latest/session     → 403                    (POST не режется на сети)
```

**Версии и наличие OAuth2-провайдера** (`/rest/applinks/1.0/manifest`, 200 без auth):

```
jira.iva.ru   typeId=jira        version=10.3.15  buildNumber=10030015
wiki.iva.ru   typeId=confluence  version=9.2.13   buildNumber=21755
```

```
jira.iva.ru  GET  /rest/oauth2/latest/authorize → 303 → /plugins/servlet/oauth2/error
                  ?errorName=invalid_request  «Указанное значение для вводимого параметра
                  «client_id» недопустимо»
jira.iva.ru  POST /rest/oauth2/latest/token    → 400 {"error":"invalid_request",
                  "error_description":"Отсутствует обязательный параметр «redirect_uri»"}
wiki.iva.ru  GET  /rest/oauth2/latest/authorize → 303 → …error?errorName=Unable+to+find+client
                  +with+that+Id
wiki.iva.ru  POST /rest/oauth2/latest/token    → 400 {"error":"invalid_request",
                  "error_description":"The required parameter 'redirect_uri' is missing"}
```

Ответы — корректная OAuth-семантика: сервер авторизации живой и жалуется ровно на то, чего
не хватает (нет зарегистрированного клиента, нет `redirect_uri`). Вывод: **обмен кода на токен
со стороны helm выполним**; недостающее — зарегистрированный на стороне ИВА клиент
(client_id/secret + разрешённый redirect_uri).

Не проверял: работает ли обмен под реальным client_id (клиента нет), и держит ли туннель
нагрузку интерактивного (не батч) трафика.

## 3. Обратный путь: браузер сотрудника ИВА → helm.tacticum.ru

**Однозначного подтверждения нет.** Развожу два разных вопроса, которые легко перепутать.

**(а) Сервер ИВА → helm (входящая достижимость из инфраструктуры ИВА) — ЯВНО НЕ ПОДТВЕРЖДЕНО.**
`20-Architecture/Бот «Поддержка» — webhook RAG-1 в чат IVA (задеплоен).md:55`:
«Достижимость: helm.tacticum.ru публичен из интернета; helm → iva-uc.ru работает (проверено);
**iva-uc.ru → helm не подтверждено**». Там же (:50) записано, что если платформа не достанет
helm снаружи — нужен эндпоинт в контуре/туннель, вопрос к Медведеву. Косвенно подтверждается
логами: `docker logs helm-helm-1 --since 2160h | grep -c 'bot/support/webhook'` → **0**, при
`HELM_IVA_BOT_SUPPORT_ENABLED=1`. Ни одного входящего вебхука от платформы ИВА за 90 дней.

**(б) Рабочая машина сотрудника ИВА → публичный сервис Tacticum — ЕСТЬ ФАКТ, но по соседнему
хосту.** `00-Board/qa-kartina-2026-07-28.md:74-78`: 27.07 18:29 **Брейкин Никита**
(`n.breykin@iva.ru`) успешно отработал `get_installation` → `pull_installation_content` →
`…manifest`; 28.07 10:32–11:36 **Байрамбеков Евгений** (`e.bayrambekov@iva.ru`) —
`list_profiles`, `tacticum_init`, `provision_installation`, `tacticum_fetch_action` ×111.
Это каталог-MCP: `dig +short mcp.tacticum.dev A → 159.194.224.59` (сервер `tacticum_prod`),
то есть **другой публичный хост**, не helm (159.194.233.33). Сам факт — сотрудники ИВА с
рабочих машин ходят в публичный HTTPS Tacticum и это работает.

**(в) Профили ролей ИВА уже сконфигурированы на helm.** `00-Board/explore-fr-analyst-baseline.md:40`
и `00-Board/impl-qa-mcp-thin-lanes.md:25`: в профилях аналитика и QA прописан сервер
`helm-analyst` = `https://helm.tacticum.ru/mcp/analyst` РЯДОМ с `iva-atlassian-write`
(stdio, личный PAT на jira.iva.ru). То есть модель эксплуатации уже предполагает, что одна и
та же машина достаёт и jira.iva.ru, и helm.tacticum.ru. Подтверждения, что этот конкретный
сервер кто-то из ИВА реально дёрнул, я не нашёл: в логах helm клиентский IP замаскирован
Traefik (`INFO: 172.18.0.2:… - "POST /mcp/analyst"` — это адрес контейнера прокси),
access-log Traefik не включён, X-Forwarded-For не логируется. Запрос в прод-Postgres на
принадлежность диалогов был заблокирован политикой — **не проверял**.

**Итог п.3:** доказательства «браузер сотрудника ИВА откроет helm.tacticum.ru» у нас НЕТ.
Есть доказательство, что рабочие машины ИВА ходят в публичный интернет к сервису Tacticum на
соседнем хосте. Это вопрос к заказчику, и формулировать его надо точно: не «есть ли интернет»,
а «открывается ли `https://helm.tacticum.ru` из браузера на рабочем месте» — при этом
известно, что серверная сторона ИВА (платформа ботов) до нас пока не достучалась ни разу.

## 4. Существующие внешние эндпоинты как образец

`src/helm/main.py`. Общий принцип: `_AUTH = [Depends(require_user)]` / `_FULL =
[Depends(require_full_access)]` навешиваются на роутер (`main.py:112-116`), а «публичный путь
со своей проверкой» делается БЕЗ этих зависимостей.

| Поверхность | Где | Гейт |
|---|---|---|
| `GET /api/auth/config` | `main.py:108`, `api/auth.py:163-175` | нет (публичный по замыслу) |
| `POST /api/auth/token` | `api/auth.py:188-207` | нет; сам обменивает code→token на idP |
| `GET /callback` | `main.py:238-243` | нет (отдаёт SPA) |
| `docs` (RAG#1) | `main.py:141` | роутер без auth; персональные ручки держат свой `Depends(require_user)` внутри |
| `POST /api/bot/support/webhook` | `main.py:174-175`, `routers/bot_support.py` | свой: заголовок `Iva-Bot-Api-Secret-Token`, `hmac.compare_digest`, **fail-closed 401 если секрет не задан**; монтируется только при флаге |
| `hrd` REST | `main.py:169` | свой `require_hrd` внутри роутера |
| `/mcp/hrd` | `main.py:185`, `mcp/hrd_server.py:200-231` | ASGI-обёртка `gated_app`: 403 на ЛЮБОЙ запрос без роли, включая handshake |
| `/mcp/analyst`, `/mcp/process` | `main.py:184-186` | Bearer через project-hub `/resolve` внутри каждого тула |

**Лучший образец для OAuth-колбэка — связка `/callback` + `POST /api/auth/token`**: это уже
работающий OAuth2 PKCE-поток того же вида. Браузер редиректится на `helm.tacticum.ru/callback`,
фронт (`web/src/auth.ts:112-131`) сверяет `state` из `sessionStorage` (:123, «OAuth state
mismatch»), затем зовёт same-origin `POST /api/auth/token`, а helm ходит на idP server-side —
`infrastructure/auth/oauth.py::exchange_code` (мотив в докстринге: обход CORS).

**Важное отличие для нашей задачи:** в существующем потоке `state` и `code_verifier` живут в
браузере (`sessionStorage`), сервер состояния не держит. Для потока «сотрудник разрешает нам
доступ к своей Jira» токен нужен СЕРВЕРУ, значит `state` обязан быть серверным и одноразовым —
переиспользовать `web/src/auth.ts` нельзя, это другой поток.

Второй образец, ближе по духу (внешний вход со своей проверкой, fail-closed, за флагом) —
`bot_support.py`: 401 при отсутствии секрета в конфиге, фильтр событий, быстрый 200 + фон.

## 5. Хранение состояния OAuth (`state`)

**Redis/memcached в helm НЕТ.** Грепом по `src/`, `docker-compose*.yml`, `pyproject.toml` —
ни одного упоминания. В `docker-compose.prod.yml` три сервиса: traefik, helm, postgres.

Значит **`state` кладём в Postgres** — и это не костыль, а уже принятый в helm паттерн
«таблица с `expires_at`»:

- `DocsClarifyPending` — `src/helm/infrastructure/db/models.py:2768-2803`:
  `expires_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), index=True)` (:2797),
  `UniqueConstraint("channel","conversation_key")` (:2799-2803), TTL описан в докстринге (:2779).
- `DocsConversationTurn` — `models.py:2806-2844`, отдельный
  `Index("ix_docs_conversation_turn_expires_at","expires_at")` «гигиена TTL — быстрый purge» (:2842-2843).
- `Rag2ConversationTurn` — `models.py:2847-2885`, тот же приём.
- Хелпер срока годности уже есть: `clarify_expires_at(now, ttl_seconds)` —
  `src/helm/infrastructure/db/repository.py:2222-2224`; проверка TTL делается в Python,
  а не в SQL (`repository.py:2111` — «устойчиво к отсутствию tz на sqlite»).

In-memory кэши в helm тоже есть (`infrastructure/auth/dev_token.py:29-31`, TTL 300 с;
`functools.lru_cache` в `docs_assistant/query_rewrite.py:36`), но для одноразового `state`
они не годятся: процесс перезапускается, а колбэк придёт позже.

**Таблицы для хранения самих токенов/учёток в helm нет** — грепом по `models.py` на
`class .*Token|Credential|Secret|Key` находится только `PersonIdentityKey` (:672), это про
идентичность людей, не про секреты. Хранилище токенов придётся заводить с нуля.

## 6. Как в helm заводится новая таблица (alembic)

- `alembic.ini` — в корне репозитория. `script_location = alembic`, `prepend_sys_path = src`,
  `sqlalchemy.url` пустой: «is set dynamically in env.py from Settings (HELM_DATABASE_URL)».
- Ревизии — `alembic/versions/`, **97 файлов**, имя файла = имя ревизии
  (`hrd224_allure_activity.py`), никаких хешей.
- **Миграции ручные, не autogenerate** — это написано прямо в докстрингах:
  `hrd224_allure_activity.py:10` «Ручная миграция (не autogenerate). Row-level tenant в PK по ADR-0005»,
  то же в `hrd220_person_identity.py:8`.
- `revision` / `down_revision` — строки-имена, цепочка ручная.
  ⚠️ **В репозитории исторически несколько голов.** Прямое предупреждение в
  `hrd220_person_identity.py:21-23`: «Конец живой линии (… → own250_requirement_product_state),
  а НЕ tpo172: в репе исторически несколько голов, и ветка от tpo172 добавила бы ещё одну».
  Текущий head на проде: `alembic current` → **`hrd224_allure_activity (head)`**.
- **Столбец `tenant` обязателен** (row-level multitenancy, ADR-0005) и входит в PK:
  `sa.Column("tenant", sa.String(), nullable=False, server_default="iva")`.
- Накат — `scripts/deploy.sh:52-53`: `docker compose exec -T helm uv run --no-sync alembic upgrade head`.

**Образец целиком** — `/Users/bubblemac/tacticum/helm/alembic/versions/hrd224_allure_activity.py`:

```python
"""HRD: авторство тест-кейсов Allure — `allure_activity`.
…
Ручная миграция (не autogenerate). Row-level tenant в PK по ADR-0005.

Revision ID: hrd224_allure_activity
Revises: hrd222_confluence_activity
Create Date: 2026-07-29
"""

from __future__ import annotations

import sqlalchemy as sa
from alembic import op

revision = "hrd224_allure_activity"
down_revision = "hrd222_confluence_activity"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_table(
        "allure_activity",
        sa.Column("test_case_id", sa.Integer(), nullable=False),
        sa.Column("action", sa.String(), nullable=False),
        sa.Column("tenant", sa.String(), nullable=False, server_default="iva"),
        sa.Column("project_id", sa.Integer(), nullable=False),
        sa.Column("author_key", sa.String(), nullable=False),
        sa.Column("ts", sa.String(), nullable=False),
        sa.Column("name", sa.String(), nullable=False, server_default=""),
        sa.Column("automated", sa.Boolean(), nullable=False, server_default=sa.text("false")),
        sa.PrimaryKeyConstraint("test_case_id", "action", "tenant"),
    )
    op.create_index("ix_allure_activity_author", "allure_activity", ["author_key"])
    op.create_index("ix_allure_activity_ts", "allure_activity", ["ts"])
    op.create_index("ix_allure_activity_project", "allure_activity", ["project_id"])


def downgrade() -> None:
    op.drop_index("ix_allure_activity_project", table_name="allure_activity")
    op.drop_index("ix_allure_activity_ts", table_name="allure_activity")
    op.drop_index("ix_allure_activity_author", table_name="allure_activity")
    op.drop_table("allure_activity")
```

## Риски и что осталось непроверенным

1. **П.3 — единственная реальная неизвестная.** Пока не подтверждено, что браузер сотрудника
   ИВА открывает `helm.tacticum.ru`, весь поток остаётся гипотезой. Соседний факт (платформа
   ботов ИВА до нас ни разу не достучалась) — повод спрашивать, а не предполагать.
2. **Клиента OAuth на стороне ИВА не существует.** Регистрация incoming-приложения
   (client_id/secret + точный `redirect_uri`) — действие администратора Jira/Confluence ИВА.
   Проверить работоспособность обмена до этого нельзя.
3. **Путь `/callback` занят** SPA-логином (`main.py:238-243`) — нельзя переиспользовать.
4. **Хранилища токенов нет** — ни таблицы, ни шифрования. Появится новая поверхность хранения
   секретов сотрудников; это отдельное решение (пересекается с работой по keystore).
5. **Не проверял:** реальный HTTP-обмен под валидным client_id; профиль нагрузки туннеля на
   интерактивных запросах; принадлежность существующих обращений к `/mcp/analyst` конкретным
   людям (запрос в прод-БД заблокирован политикой).
6. **Наблюдаемость колбэка слабая:** access-log Traefik выключен, X-Forwarded-For не пишется —
   отладить «пришёл ли редирект» по логам сегодня нельзя, это надо будет закрывать.