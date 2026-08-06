---
title: Разведка — можем ли мы выпускать ключи Tacticum сами (30.07)
type: note
status: draft
created: 2026-07-30
updated: 2026-07-30
repo: tacticum-dev
project: iva-write
tags:
- board
- iva-write
- project-hub
- explorer
permalink: tacticum/00-board/explore-key-issuing-2026-07-30-1
---

# Короткий ответ

**Ключи `phk_` мы выпускаем сами — полностью, своим кодом, на своём проде, своим админ-токеном.
Ни одного чужого шага в цепочке нет.** Выпуск делается тремя HTTP-вызовами под общим bearer'ом,
то есть **программно, ботом**, а не только руками через UI.

**Но гипотеза лида неверна: `project-hub` — это НЕ наш `tacticum-dev`.** Это отдельный сервис
(`master.cifragen.ru` / `10.16.0.17:8770`), исходников которого в наших рабочих каталогах нет.
`tacticum-dev` — его **клиент**, а не он сам.

Путаницу породило совпадение: **оба сервиса выпускают ключи с префиксом `phk_`**, и оба имеют
ручку резолва. Это разные ключи из разных баз.

---

# 1. Тот ли это сервис — НЕТ

Ручки `POST /api/internal/resolve` в `tacticum-dev` **не существует**. Проверено по всем
роутерам, зарегистрированным в приложении
(`apps/backend/src/backend/platform/app_factory.py:355-372`) — префиксы там `/admin/*`,
`/api` (identity customer-api), `/api/architecture`. Ни одного `/api/internal`.

Зато `tacticum-dev` **сам ходит** на `/api/internal/resolve` наружу — то есть является клиентом
project-hub:

- `apps/backend/src/backend/architecture/infrastructure/project_hub_adapter.py:16,49,57` —
  HTTP-адаптер, вызывает `/api/internal/resolve-by-email` и `/api/internal/resolve`;
- `apps/backend/src/backend/platform/config.py:86-88` — `project_hub_url`,
  `project_hub_service_key`;
- `docker-compose.prod.yml:125-134` — `PROJECT_HUB_URL: ${PROJECT_HUB_URL:-}`,
  `PROJECT_HUB_SERVICE_KEY`, в комментарии прямо: «reached over the provider private network
  (`PROJECT_HUB_URL=http://10.16.0.17:8770`)».

Что такое project-hub — сказано в нашем же ADR:
`docs/adr/0051-project-hub-as-single-idp-and-rbac-authority.md:16` — «**project-hub**
(`master.cifragen.ru`) — самостоятельный IdP (свой логин `auth/passwords.py` + session-cookie +
`phk_` API-ключи)». Строка 95 того же ADR: работа по нему — «**кросс-репо** (project-hub + Dev
backend + Legends UI + admin_web)». То есть у project-hub свой репозиторий, и его у нас нет.

Что делает наш **похожий** эндпоинт: `apps/backend/src/backend/identity/interface/admin/resolve_key.py:41`
— `POST /admin/identity/resolve-key`, тело `{"key": "phk_…"}`, ответ
`{email, org_id (slug), active, key_id}`. Он резолвит **наши** ключи для MCP-гейтвея (iva-read),
и в его же докстроке (строка 4) написано «двухслойная auth, **как resolve project-hub**» — то есть
это зеркало чужого контракта, а не он сам.

**Адреса, которые лид назвал, тоже надо развести:**

| Адрес | Что это | Чьё |
|---|---|---|
| `master.cifragen.ru` | OIDC-издатель project-hub (`docker-compose.prod.yml:384-385`) | **чужое** |
| `10.16.0.17:8770` | приватный адрес project-hub (`arch.cifragen.ru` там же, compose:96-99) | **чужое** |
| `project.cifragen.ru` | Taiga-трекер (ссылки на US/эпики в CONTEXT.md, доках) | чужой хостинг, наш проект |
| `master.tacticum.dev` | админ-REST **нашего** backend'а | **наше** |
| `dev.tacticum.dev` (10.16.0.15) | admin_web + `/admin/*` нашего backend'а | **наше** |
| `mcp.tacticum.dev/mcp` | публичный MCP нашего backend'а | **наше** |

`master.tacticum.ru` из постановки лида **в репозиториях не встречается ни разу** — такого хоста
нет. Скорее всего имелся в виду либо `master.tacticum.dev` (наш админ-REST, вход по вставленному
bearer-токену, никакого OIDC), либо `master.cifragen.ru` (чужой OIDC project-hub).

---

# 2. Полный контракт выпуска ключа

`apps/backend/src/backend/identity/interface/admin/memberships.py:311-373`

```
POST /admin/memberships/{org_id}/{user_id}/api-keys
Authorization: Bearer <admin-token>
тело: НЕТ (оба параметра — в пути)
→ 201 {"id", "key": "phk_<64 hex>", "key_prefix": "phk_"+8, "created_at"}
→ 404 если Membership(org_id,user_id) не найден
```

Ключ генерится `secrets.token_hex(32)` с префиксом `phk_` (строки 302-308), в БД лежит только
`sha256` (`hash_token`, строка 345), плейнтекст отдаётся **один раз**. Пишется `AdminAudit`
(строка 349). Повторный вызов создаёт **ещё один** ключ — уникальности на membership нет
(докстрока, строка 323).

**Права — `require_admin_either`** (`interface/admin/_deps.py:25-70`). Принимает **два** пути,
оба через `Authorization: Bearer`:

1. **Общий админ-токен** — `settings.catalog_admin_token`, сравнение `secrets.compare_digest`
   (строки 42-45), actor в аудите = `"admin"`. В проде это env `CATALOG_ADMIN_TOKEN`
   (`docker-compose.prod.yml:123`).
2. **Authentik staff JWT** — если сконфигурены `authentik_issuer_url/audience/jwks_url`;
   требуется `claims.is_staff`, иначе 403 (строки 47-65).

**Ответ на ключевой вопрос лида: да, дёргается программно.** Ветка №1 — это обычный статический
bearer из env, никакой сессии/куки/интерактива. Бот с этим токеном выпускает ключи сам.
Сессионного пути (cookie) в этой зависимости нет вообще.

Рядом — весь остальной CRUD под тем же `require_admin_either`: `GET .../api-keys` (список,
строка 145), `GET /admin/memberships/api-keys` (все ключи с email владельца — для HELM,
строка 181), `DELETE .../api-keys/{key_id}` (отзыв, строка 376).

---

# 3. Что нужно, чтобы выпустить ключ живому человеку

Цепочка обязательна и вся программная:

| Шаг | Эндпоинт | Файл:строка | Что нужно |
|---|---|---|---|
| 1. учётка | `POST /admin/users` `{email, full_name, project_hub_user_id?}` → `{user_id}` | `identity/interface/admin/users.py:39` | только email; **идемпотентно** по email (никогда не перетирает существующее) |
| 2. членство | `POST /admin/memberships` `{organization_id, user_id, role}` | `memberships.py:58` | org и user должны существовать; **упирается в seats** |
| 3. ключ | `POST /admin/memberships/{org}/{user}/api-keys` | `memberships.py:311` | нужен существующий Membership |

**Да, `user_id` и `org_id` должны существовать заранее — но завести их мы можем сами.**
`POST /admin/users` (роутер зарегистрирован, `app_factory.py:359`) создаёт локального юзера
**по одной только почте**, ни Authentik, ни project-hub не требуются. Serena-память
`.serena/memories/gotchas/identity_provisioning_and_no_email.md:10` («Нет admin-API create local
user») **устарела** — этот эндпоинт есть; заведён он как примитив провижининга для
project-hub CatalogAdapter, но админ-токеном вызывается кем угодно.

**Единственная нетривиальная преграда — seats.** `_enforce_seats_cap`
(`identity/application/sync_handlers.py:229-255`): у организации должна быть Subscription
в статусе `active`/`trial`, иначе **409 «has no active subscription»**; и `count(Membership) >= seats`
тоже даёт 409. Это наша же БД — лечится нашими же руками, но это реальный шаг, а не формальность.

**Письма никуда не уходят.** В `apps/backend` нет ни SMTP, ни какой-либо отправки почты
(та же память, строки 3-7). Заведение юзера — молчаливое; ключ отдаём человеку сами, в чат.
Эндпоинта «пригласить по почте» нет.

**Для работы ключа на рантайме** (не для выпуска): `middleware.py:57-70` резолвит `phk_` в
`MembershipScope`; а `build_authscope_from_membership(installation_id)`
(`identity/application/membership_installation_scope.py:67`) дополнительно требует цепочку
Installation → Workspace → Org-Account → активная Subscription. То есть для инструментов,
привязанных к installation, org'у нужны подписка и installation. Для `whoami` — не нужны
(`identity/interface/mcp/whoami.py:32-96`, пустой список installations там штатный, не ошибка).

---

# 4. Самообслуживание — НЕТ

Человек **не может** выпустить ключ себе сам. Никакого OIDC-входа для этого нет.

- Страница ключей одна: `apps/admin_web/src/app/(authed)/orgs/[id]/members/[userId]/api-keys/page.tsx`
  — это **superadmin-UI**, называется «Tacticum admin / Staff control plane»
  (`apps/admin_web/src/app/login/page.tsx:42-46`).
- Вход в него — **форма, куда вставляют админ-bearer**, который кладётся в `localStorage`
  (`login/page.tsx:15-22`, `src/lib/api.ts:8,22-27`). Ни логина/пароля, ни OIDC, ни сессии.
  Гейт `(authed)/layout.tsx:10-18` — просто «есть токен в localStorage → пускаем».
- Фронт бьёт в те же `/admin/*` ручки через `/api-backend` (`src/lib/api.ts:9,38-42`).
  То есть UI не даёт **ничего**, чего не даёт curl; кто держит админ-токен — тот и админ.
- Второй фронт `apps/web` (Legends) — только `(authed)/architecture`, страниц ключей там нет.
- Self-service заявлен как **будущее** (`docs/agents/installing-new-profile.md:79-93` — MCP-тул
  `provision_installation`, Taiga US #434, не реализован).

Владелец членства свою страницу ключей **не видит**: у него нет доступа в admin_web в принципе.

---

# 5. Где задеплоено и чей прод — НАШ

`.serena/memories/deployment.md:1-30` (verified 2026-06-30):

- SSH-сервер `tacticum_dev`, host `159.194.224.59`, доступ через ssh-manager MCP;
- стек в `/opt/tacticum/`, git-checkout `main`, `docker compose -f docker-compose.prod.yml`;
- сервисы: `catalog-mcp`, `postgres:16-alpine` (БД `tacticum_catalog`), `qdrant`, `minio`,
  `traefik`, `authentik-server/worker`, `redis`, `admin_web`, `legends`;
- админ-токен — env `CATALOG_ADMIN_TOKEN` в серверном `.env` (`docker-compose.prod.yml:123`).

Prod-рецепт выпуска ключей прямо в контейнере (обход HTTP, тот же результат) описан в
`.serena/memories/gotchas/identity_provisioning_and_no_email.md:21-35`, там же боевые id:
IVA org `4034524f-fa46-4e1c-a4ae-28ee202d1c7d` (slug `iva`), контейнеры
`tacticum-catalog-mcp-1` / `tacticum-postgres-1`.

project-hub на `10.16.0.17` — **соседний хост того же провайдера**, приватная сеть
`10.16.0.0/16`, но чужой деплой и чужой репозиторий.

---

# 6. Развилка «нужен код» vs «нужен доступ»

## Можем полностью сами (ни кода, ни чужих людей)

| Что | Чем |
|---|---|
| Завести учётку по почте | `POST /admin/users` — работает сегодня |
| Дать членство в org | `POST /admin/memberships` — работает, упирается только в наши же seats |
| Выпустить `phk_` | `POST /admin/memberships/{org}/{user}/api-keys` — работает |
| Отозвать / посмотреть / выгрузить все ключи с email | `DELETE`/`GET` там же — работают |
| Сделать это ботом | админ-токен статический, `require_admin_either` ветка bearer — **интерактив не нужен** |
| Доступ к боевому инстансу | **уже есть**: `tacticum_dev`, `/opt/tacticum/.env` |

## Нужен код (наш, небольшой)

- Обёртка «выдать ключ по почте» одним вызовом (сейчас три HTTP-вызова + разбор 409 по seats).
- Обработка seats: подписка/лимит мест — сейчас это ручное вмешательство в нашу же БД.

## Нужен чужой доступ — **только для project-hub-ключей**

Если нужен ключ, который признаёт **project-hub** `/api/internal/resolve` (то есть личность в
`master.cifragen.ru`) — это чужая система, туда мы не ходим. Нужны: заведение юзера в hub,
роль/capability, и, если по API — их service-ключ.

---

# Отдельно: почему это, возможно, вообще не нужно просить

**HELM умеет принимать НАШИ `phk_`-ключи вместо project-hub'овских.** В `helm` во всех трёх
MCP-серверах есть второй источник identity:

- `src/helm/interface/mcp/analyst_server.py:338-350` (аналогично `process_server.py:190-200`,
  `hrd_server.py:158-168`): если project-hub `/resolve` токен не признал — идёт
  `resolve_dev_token(token, mcp_url=settings.dev_mcp_url)`;
- `src/helm/infrastructure/auth/dev_token.py:84` — зовёт тул `whoami` нашего MCP
  (`https://mcp.tacticum.dev/mcp`) под этим ключом и берёт `caller.user_email`;
  принимает только `auth_method == "phk"` (строки 71-81), `tac_` обезличен и отвергается;
- два гейта: фича включается **только** если задан `HELM_DEV_MCP_URL`
  (`src/helm/config.py:546`, дефолт `None` → выключено), и почта должна быть в
  `dev_token_email_domains` (дефолт `iva.ru,tacticum.ru`, `config.py:552`);
- права при этом не расширяются: синтезируется членство `viewer`, роли решает
  `principal_for_tenant` + таблица `user_role`.

То есть цепочка «мы выпускаем `phk_` у себя → человек ходит им в HELM» **уже написана и
покрыта тестами** (`tests/interface/test_analyst_mcp.py:252-364`, пять сценариев).

**Чего я НЕ проверил и проверять не могу (read-only, локально):** задан ли `HELM_DEV_MCP_URL`
в боевом `.env` на сервере helm. Локальный `/Users/bubblemac/tacticum/helm/.env` содержит всего
четыре переменные (`HELM_GATEWAY_BASE_URL`, `HELM_GATEWAY_API_KEY`, `HELM_TENANT`,
`HELM_PROJECTHUB_TOKEN`) — `HELM_DEV_MCP_URL` там нет, но это dev-файл, не прод.
**Это первое, что стоит посмотреть на сервере** — если переменная не выставлена, фича выключена,
и включение = одна строка в `.env` + рестарт, без правки кода.

---

# Риски и оговорки

- **Совпадение префиксов `phk_` у двух разных систем** — главный источник путаницы. Ключ
  `tacticum-dev` не резолвится в project-hub и наоборот. При разборе инцидентов всегда уточнять,
  чей ключ.
- **Общий админ-токен = полный контроль** над орг/членствами/ключами/подписками. Он один на
  весь `/admin/*`; ролевого разграничения внутри админки нет. Бот, которому его дадут, сможет
  всё, что может админ.
- **Стухшая документация:** `docs/agents/installing-new-profile.md:65` описывает
  `POST /admin/api-keys` как выдающий `phk_` — **такого эндпоинта нет**. В
  `platform/admin/router.py` есть только `/admin/tokens` (строка 207), и он выпускает
  `tac_*`-токены installation-уровня (`router.py:420`), а не `phk_`. Кто пойдёт по этой доке —
  упрётся в 404.
- **Serena-память `identity_provisioning_and_no_email` частично неверна** (утверждает отсутствие
  admin-API создания юзера) — актуализировать при случае; правку не делал, зона не моя.

# Открытый вопрос к лиду

Формулировка задачи «выпускать ключи Tacticum» допускает два прочтения, и ответ разный:
**наши `phk_` — можем сами хоть сейчас**; **ключи project-hub — не можем, чужая система**.
Из контекста `iva-write` похоже, что нужны первые. Если нужны вторые — скажи, соберу отдельно,
что именно просить у владельца hub.
---
---

# Цена прямого пути (дополнение 30.07, по запросу лида)

Код iva-write живёт в ветке `feat/iva-write-keystore`, worktree
`/Users/bubblemac/tacticum-worktrees/helm-iva-write-keystore` (HEAD `baa7895`). Все ссылки
`файл:строка` ниже — по нему.

## Сразу: главное препятствие не в коде и не в правах

**Через бота личность человека НЕ подтверждена.** Почта приходит текстом в чате и помечается
`SOURCE_DECLARED` (`src/helm/interface/api/routers/bot_iva_write.py:219`). Код это знает и говорит
прямо — докстрока `src/helm/application/bot_iva_write.py:18-21`: «**Почта из сообщения — адрес
доставки, а не личность.** Человек может назвать чужой адрес, и проверка домена этого не ловит…
Поэтому здесь нет ни одной функции, которая "подтверждает" личность».

Единственный доверенный источник — поле `login` из события платформы ботов
(`routers/bot_iva_write.py:261-263`: `login_email = extract_email(login)`, отсев по домену, иначе
пусто). Именно поэтому статус отвечает **только** по нему, а на названный в чате адрес — отказ
`status_identity_unknown_text` (`bot_iva_write.py:32-35, 218-231`).

**А содержит ли `login` корпоративную почту — неизвестно.** Это дословно вопрос №3 из запроса
Васе Медведеву ([[zaprosy-iva-write-2026-07-30]]): «`login` — это корпоративная почта? Ни одного
реального примера значения у меня нет».

Отсюда:

- **выпуск ключа на названный адрес = выдача доступа Tacticum тому, кто угадал чужую почту.**
  Делать этого нельзя ни за какие деньги;
- **даже простой ответ «у вас ключ есть/нет» на названный адрес — разглашение о другом
  человеке**, ровно того сорта, который в этом коде уже запрещён для статуса.

**Вывод: и «выпускать», и «проверять» упираются в ответ Васи, а не в наш код.** Без него
допустим только вариант 0 ниже.

## 1. Что именно вызывает бот

Все три ручки — `tacticum-dev`, все под `Authorization: Bearer <CATALOG_ADMIN_TOKEN>`.

| # | Вызов | Обязательное | Ответ | Ошибки, которые надо разобрать |
|---|---|---|---|---|
| 1 | `POST /admin/users` (`identity/interface/admin/users.py:39-81`) | `email`; опц. `full_name`, `project_hub_user_id` | `{user_id, email, created: bool}` | практически нет — идемпотентно по email (регистронезависимо, `func.lower`), существующее не перетирает |
| 2 | `POST /admin/memberships` (`memberships.py:58-123`) | `organization_id` (UUID), `user_id` (UUID), `role` ∈ `po\|techlead\|member` | 201 + `{organization_id, user_id, role, created_at}` | **404** org не найдена · **404** user не найден · **409** членство уже есть · **409** мест нет |
| 3 | `POST /admin/memberships/{org_id}/{user_id}/api-keys` (`memberships.py:311-373`) | оба в пути, **тела нет** | 201 + `{id, key: "phk_"+64hex, key_prefix, created_at}` | **404** членства нет |

**409 по местам — два разных текста**, оба из `_enforce_seats_cap`
(`identity/application/sync_handlers.py:229-255`):
`"Organization {id} has no active subscription; cannot add Membership."` (нет Subscription в
`active`/`trial`) и `"Organization {id} reached seat cap (N/M)."`. Различать их придётся по
подстроке — отдельных кодов нет.

**Полезный побочный эффект шага 1:** `created: false` означает «человек уже был в системе».
То есть `POST /admin/users` работает как **найти-или-создать** и заодно отдаёт `user_id` — а
отдельного «найти юзера по email» в API **нет** (в `admin/users.py` только POST). Если юзера не
было, он создастся; запись без членства ничего не открывает, так что побочный эффект безобидный.

**`org_id` — он один на всех.** Это UUID организации IVA
`4034524f-fa46-4e1c-a4ae-28ee202d1c7d` (slug `iva`), зафиксирован в
`.serena/memories/gotchas/identity_provisioning_and_no_email.md:33`. От человека не зависит —
все сотрудники ИВА идут в одну org.

**Но взять его по slug программно нечем.** `GET /admin/orgs`
(`identity/interface/admin/orgs.py:112-133`) — это листинг с `limit`/`offset`, **фильтра по slug
нет**; `GET /admin/orgs/{org_id}` (строка 136) требует уже известный UUID. Значит либо UUID
кладётся константой в настройки helm, либо бот листает орги и фильтрует у себя. Настройкой —
честнее и дешевле.

## 2. Где это писать в helm

Клиента к нашему backend в helm нет: в `infrastructure/auth/` только `resolve_client.py`
(64 строки, project-hub) и `dev_token.py` (114 строк, ходит MCP-протоколом на
`mcp.tacticum.dev`, не REST). Нужен новый — обычный httpx-клиент по образцу `resolve_client.py`.

**Шов готов и вшит.** `application/tacticum_key.py` (74 строки) — единственное место решения о
ключе, и в его докстроке прямо написано, что ветка выпуска добавляется СЮДА. Три точки чтения:
`application/iva_write.py:354-374`, `interface/api/routers/iva_write.py:399-408`,
`interface/api/bot_iva_write_chat.py:55-73` (плюс `application/bot_iva_write.py:191-207` берёт
формулировку). Ни одна не ветвится сама — все читают код статуса. Это реально экономит работу.

**Одна архитектурная деталь.** `tacticum_key_status()` сейчас **чистая синхронная функция без
I/O**, и в этом смысл слоя (`application/bot_iva_write.py:1` — «Чистый слой, без I/O»). Тащить
туда HTTP нельзя. Значит факт «ключ есть / ключа нет» добывается в вызывающем и передаётся
параметром — правка сигнатуры в трёх точках, отсюда строки в таблицах ниже.

**Мелочь на исправление при реализации:** докстрока `application/tacticum_key.py:15-18`
утверждает — «ключи Tacticum выпускает **project-hub**, не helm, и локальной операцией это не
станет никогда». Разведка это опровергла. Комментарий станет прямой ложью в коде, править
обязательно.

**Ещё:** `USE_OUTCOMES` — закрытый словарь исходов журнала
(`infrastructure/db/credential_repo.py:38`, сейчас `{ok, denied, missing, granted, key_status}`,
`log_use` бросает `ValueError` на чужое значение). Новый исход выпуска = правка этой строки.

### Коэффициент тестов — измеренный, не заявленный

| Аналог | Код | Тесты | ×|
|---|---|---|---|
| `infrastructure/auth/resolve_client.py` | 64 | 87 (`tests/infrastructure/test_resolve_client.py`) | **1.36** |
| `infrastructure/bot_iva_write/client.py` | 131 | 147 (`tests/infrastructure/test_bot_iva_write_client.py`) | **1.12** |
| `application/bot_iva_write.py` | 234 | 178 (`tests/application/test_bot_iva_write.py`) | **0.76** |
| весь репозиторий | 77 728 | 50 603 | **0.65** |

`dev_token.py` (114 строк) своего тест-файла **не имеет** — покрыт косвенно через
`tests/interface/test_analyst_mcp.py:252-364`.

Заявленные лидом ×1.17 — близко к среднему по двум HTTP-клиентам (1.24). Ниже считаю
**×1.3 для инфраструктурного клиента** и **×1.0 для прикладного слоя** (там тесты исторически
легче кода). Репозиторный ×0.65 к новому коду с сетью неприменим: он размазан по большим
модулям данных.

## 3. Админ-права: чей токен и где живёт

**(а) Где лежит сейчас.** Имя переменной — `CATALOG_ADMIN_TOKEN`. Живёт на сервере
`tacticum_dev` (`159.194.224.59`) в `/opt/tacticum/.env`, пробрасывается в контейнер
`catalog-mcp` (`tacticum-dev/docker-compose.prod.yml:123`), читается как
`settings.catalog_admin_token` (`platform/config.py:62`). В окружении helm его **нет**: в
`/Users/bubblemac/tacticum/helm/.env` четыре переменные (`HELM_GATEWAY_BASE_URL`,
`HELM_GATEWAY_API_KEY`, `HELM_TENANT`, `HELM_PROJECTHUB_TOKEN`), админских среди них ни одной.

**(б) Придётся ли класть в helm.** Да — новая переменная в `/opt/helm/.env` на сервере helm.
Это прод-изменение на двух серверах сразу (значение берётся с одного, кладётся на другой).

**(в) Чем это плохо — честно, риск больше, чем звучит.** Токен даёт **не «выпуск ключей», а весь
`/admin/*`**. Под тем же `require_admin_either` висят:

- `admin/orgs.py` — создание/переименование любых организаций;
- `admin/subscriptions.py` — подписки и места;
- `admin/memberships.py` — членства и ключи **всех** организаций, не только `iva`;
- `admin/users.py` — заведение пользователей;
- admin workspaces / installations / design-systems / catalog / render-preview
  (`platform/app_factory.py:355-368`);
- `platform/admin/router.py` — `/admin/seed` (111), `/admin/retract` (157), `/admin/tokens` (207,
  выпуск `tac_*`), `/admin/audit` (363), `/admin/usage` (387);
- `admin/identity/resolve_key.py:41` — резолв **любого** `phk_` в email + org.

Ролевого разграничения внутри админки нет вообще: `require_admin_either` возвращает строку
`actor` для аудита, но **ни один эндпоинт её не проверяет**. Значит компрометация helm =
компрометация control-plane всего каталога Tacticum, а не «утечка ключей org iva». Helm —
сервис с публичным HTTP-входом (`helm.tacticum.ru` через Traefik), то есть поверхность
у него шире, чем у backend'а.

**(г) Узкого способа сейчас НЕТ.** Проверено по коду: `catalog_admin_token` — **одно** значение
типа `SecretStr` (`platform/config.py:62`), сверка `secrets.compare_digest`
(`admin/_deps.py:42-45`). Ни списка токенов, ни скоупов, ни привязки к org. Второй путь
(Authentik staff JWT) шире, а не уже: проверяется только `claims.is_staff`
(`_deps.py:60-64`) — staff-джвт может всё то же самое.

Сделать узкий токен **можно, но это своя работа в `tacticum-dev`** (репозиторий наш — значит
выполнимо без чужих людей): таблица сервисных токенов со `scope` и `organization_id`, миграция,
новая зависимость вместо `require_admin_either` на трёх нужных ручках. Оценка на аналоге
`admin/_deps.py` (74 строки) + модель + миграция: **150–250 строк кода, 120–200 тестов,
4–6 файлов, плюс деплой backend'а**. Вынесен вариантом C в итоговой таблице.

## 4. Что видит человек

**Ключ показывается ровно один раз** (`memberships.py:368-373` возвращает плейнтекст;
в БД только `sha256` через `hash_token`, строка 345). Повторно его не достать никак — только
выпустить новый.

**Значит, если бот пришлёт ключ текстом, секрет навсегда останется в переписке платформы ботов
ИВА** — в чужой системе, в истории, доступной их администраторам. И это **прямо ломает правило,
уже принятое в этом коде**: `application/bot_iva_write.py:22-24` — «**Ни одного значения секрета
в текстах.** Статус говорит, что подключено и до какого срока, и молчит про токены».

Обычное решение — одноразовая ссылка на страницу выдачи вместо секрета в чате. **И механика для
неё в репозитории уже есть, писать её заново не нужно:** одноразовый `state` с TTL
(`iva_oauth_state_ttl_seconds`, дефолт 600 сек, `config.py:590`), выдача ссылки в чат
(`bot_iva_write.py:offers_text` — «Ссылка одноразовая и живёт несколько минут»), HTML-страница
результата (`routers/iva_write.py:_page`). Переиспользуется, а не пишется.

**Если выпуск не удался.** Бот получит 409 с текстом уровня инфраструктуры («no active
subscription» / «reached seat cap (12/10)») — показывать это человеку нельзя. Нужен свой текст
(«места под вас пока нет, я сообщил администратору») плюс сигнал нам. Таких формулировок в
`bot_iva_write.py` сейчас нет — пишутся с нуля; там же лежат образцы тона
(`not_configured_text`, `bad_email_text`).

## 5. А нужно ли вообще выпускать — проверка наличия есть, и она дешёвая

**Да, проверить можно одним вызовом, ничего не создавая.**
`GET /admin/memberships/api-keys?org_slug=iva` (`memberships.py:181-219`) отдаёт по каждому
ключу: `email, full_name, org, role, key_prefix, issued_at, last_used_at, status`
(`active`/`revoked`).

**Эта ручка написана ПОД нас.** Её докстрока (строки 186-190) дословно: «Потребитель — **HELM**
(pull по приватной сети): стадия `assigned` в воронке внедрения ADP = у разработчика есть
активный `phk_*`-ключ. Одна строка на ключ; у разработчика их может быть несколько (HELM
агрегирует по email)». То есть контракт под этот сценарий уже существует.

Бонус: `last_used_at` различает «ключ есть и им ходят» и «ключ есть, но ни разу не
использован» — второе и есть «потерял, не знает где».

**Оговорка:** ручка отдаёт ключи **всей** организации — helm получит список всех разработчиков
IVA с почтами и именами. Узкий вариант `GET /admin/memberships/{org}/{user}/api-keys`
(`memberships.py:145`) требует `user_id`, а его по почте узнать можно только через
`POST /admin/users` (см. §1) — то есть узкий путь стоит одного лишнего вызова с побочным
созданием юзера. Оба варианта рабочие; узкий — аккуратнее по данным.

**Сценарий «сначала посмотреть, потом выпускать» действительно дешевле и безопаснее** — он
вынесен вариантом A, и в большинстве случаев выпуск не понадобится вовсе.

## 6. Итоговая таблица

Строки — по измеренным аналогам (§2), не на глаз. Диапазон отражает неопределённость,
а не запас.

| Вариант | Файлов | Код | Тесты | Всего | Что нужно от людей | Риск |
|---|---|---|---|---|---|---|
| **0. Бот просто называет, к кому идти** | 1 (`application/tacticum_key.py`, функция `key_note`) | 5–10 | 5–10 | **~10–20** | ничего | нет. Ни секретов, ни админ-токена, ни разглашения |
| **A. Бот проверяет наличие и отвечает** | 5–6 (2 новых) | 150–215 | 175–245 | **~325–460** | ответ Васи Медведева про `login`; ОК Президента на `CATALOG_ADMIN_TOKEN` в `/opt/helm/.env` | **высокий по правам** (весь `/admin/*` в helm), средний по данным (видит ключи всей org) |
| **B. Бот выпускает сам** | 7–8 (2–3 новых) | 290–415 | 340–485 | **~630–900** | то же + решение, как отдавать ключ (ссылка, не чат) + тексты отказа по местам | **высокий по правам** + новый: выдача чужого доступа при неподтверждённой личности |
| **C. B + узкий токен в `tacticum-dev`** | B + 4–6 в `tacticum-dev` | 440–665 | 460–685 | **~900–1350** | то же + деплой backend'а (миграция) | **низкий по правам** — helm получает только «выпустить ключ в своей org» |

### Расшифровка A

| Что | Файл | Код | Тесты |
|---|---|---|---|
| Клиент к нашему admin-API (один GET + датакласс + разбор ошибок), образец `resolve_client.py` (64) | `infrastructure/auth/tacticum_admin.py` (новый) | 70–90 | 90–120 (×1.3) |
| Настройки: базовый URL, токен, org-UUID | `config.py` | 12–15 | — |
| Третий код состояния + факт параметром вместо вычисления | `application/tacticum_key.py` | 25–40 | 30–45 |
| Намерение «нет ключа / потерял» + тексты (`parse_intent` знает только `status`/`access`, `bot_iva_write.py:80-86`) | `application/bot_iva_write.py` | 30–45 | 35–50 |
| Правка трёх точек шва под новую сигнатуру | `iva_write.py:354`, `routers/iva_write.py:399`, `routers/bot_iva_write.py` | 15–25 | 20–30 |

### Расшифровка B (сверх A)

| Что | Файл | Код | Тесты |
|---|---|---|---|
| +3 метода: upsert user / add membership / create key + разбор двух 409 по подстроке | `infrastructure/auth/tacticum_admin.py` | 80–110 | 100–140 |
| Отдача ключа одноразовой ссылкой (переиспользует `state` + `_page`), не текстом в чат | `routers/iva_write.py`, `application/iva_write.py` | 40–60 | 45–70 |
| Тексты отказа (нет мест / нет подписки) + сигнал нам + новый исход в `USE_OUTCOMES` | `application/bot_iva_write.py`, `credential_repo.py:38` | 20–30 | 20–30 |

## Что я бы подсветил Президенту

1. **Прежние 1800–2400 строк были не про это.** Честная цена выпуска — **630–900 строк**,
   а если сначала проверять наличие (и в большинстве случаев не выпускать) — **325–460**.
2. **Вариант 0 стоит ~15 строк и не требует вообще ничего.** Если жалоба человека — «не знаю,
   что мне вставлять и где это взять», она закрывается одной фразой в `key_note`. Всё остальное
   имеет смысл, только если мы хотим убрать человека-администратора из цикла.
3. **Главная цена — не строки, а `CATALOG_ADMIN_TOKEN` в окружении helm.** Это не «право
   выпускать ключи», это весь control-plane каталога Tacticum в публично доступном сервисе.
   Узкого токена сегодня не существует; сделать его можно, но это ещё ~300–450 строк в
   `tacticum-dev` с миграцией и деплоем.
4. **Выпуск блокирован не нами.** Пока неизвестно, лежит ли в `login` события корпоративная
   почта (вопрос №3 Васе Медведеву), доверенной личности у бота нет, а выпускать ключ Tacticum
   на названный в чате адрес нельзя.

## Чего я не проверял

- не запускал ни одного вызова к боевым ручкам — разведка по коду, read-only;
- не смотрел серверные `.env` (ни `/opt/tacticum/.env`, ни `/opt/helm/.env`) — вывод «в helm
  админ-токена нет» сделан по локальному `helm/.env` и по отсутствию соответствующего поля в
  `helm/src/helm/config.py`; на сервере файл может отличаться;
- оценки в строках — по аналогам того же репозитория, но это оценки, а не замер.
