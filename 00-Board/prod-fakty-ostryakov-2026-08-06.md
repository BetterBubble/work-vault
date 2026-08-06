---
title: Прод-факты по Острякову (iva-write, 06.08.2026)
type: note
status: draft
created: 2026-08-06 11:40
updated: 2026-08-06 12:40
repo: helm
project: iva-write
tags:
- board
- iva-write
- prod
- verify
permalink: tacticum/00-board/prod-fakty-ostryakov-2026-08-06
---

# Прод-факты: d.ostryakov@iva.ru, канал записи iva-write

Роль: verifier (read-only). Сервер `helm` = 159.194.233.33, только чтение.
Время на сервере в момент съёма: **2026-08-06 08:24:20 UTC** (`date -u`), это 11:24 МСК.

**Границы съёма.** SQL по прод-БД я снять НЕ смог: гейт воркера режет `docker exec`
(единственный путь к psql на этом хосте), а `ssh_db_query` в этой сессии не подключён.
Поэтому пункты 1 и 2 идут как блокер с готовыми запросами, а не как факт. Всё остальное
снято логами, `docker inspect`, чтением задеплоенного кода и публичными пробами.

**Окно логов.** Везде, где ниже стоит счётчик по логам, он снят командой `docker logs
--since 30h -t helm-helm-1` в 08:24–08:36 UTC. Интервал подписан у каждого числа намеренно:
в разборе лида те же величины сняты за 36 часов, и без подписи 91263 против 94749 читались бы
как противоречие, а это одна и та же картина в разных окнах.

**Правки 12:40** (по разбору лида, [[razbor-zhaloby-ostryakov-2026-08-06]]): уточнён адрес БД
каталога в разделе 2 · четыре из шести 401 опознаны как наши собственные пробы · находка про
автопродление вынесена из раздела 1 в отдельный раздел 5, потому что она про систему, а не
про Острякова.

## 1. Токены/доступы Острякова — НЕ СНЯЛ (блокер)

Таблицы в схеме helm (`/opt/helm/src/helm/infrastructure/db/models.py`):

- `external_credential` — PK (`tenant`, `subject`, `system`), поля `kind`, `expires_at`,
  `refresh_expires_at`, `revoked_at`, `refresh_ciphertext`, `external_login`,
  `created_at`, `updated_at` (строка 3435);
- `credential_use_log` — журнал подстановок, `outcome ∈ {ok, denied, missing, granted,
  key_status}`, `reason` (строка 3514);
- `key_handout` — выдача ключа Tacticum, `key_prefix`, `created_at`, `expires_at`,
  `used_at` (строка 3555).

Запросы к прогону (БД `helm`, контейнер `helm-postgres-1`, порт хоста 127.0.0.1:15432):

```sql
SELECT tenant, subject, system, kind, external_login,
       created_at, updated_at, expires_at, refresh_expires_at, revoked_at,
       (refresh_ciphertext IS NOT NULL) AS has_refresh,
       (expires_at > now())              AS alive_now
  FROM external_credential
 WHERE subject = 'd.ostryakov@iva.ru';

SELECT id, subject, system, outcome, reason, created_at
  FROM credential_use_log
 WHERE subject = 'd.ostryakov@iva.ru'
 ORDER BY id DESC LIMIT 50;

SELECT subject, key_prefix, created_at, expires_at, used_at
  FROM key_handout
 WHERE subject = 'd.ostryakov@iva.ru';

SELECT now() AS db_now;
```

**Что уже известно про сроки без БД, из ответа бота в 10:06 МСК (07:06 UTC):** Confluence
до 06.08 08:00 UTC, Jira до 06.08 07:59 UTC. На момент съёма (08:24 UTC) оба срока
**прошли** — 24 и 25 минут назад соответственно.

Продление в коде **ленивое, на использовании канала** (`application/iva_write.py:668-700`:
«просроченный access с живым refresh продлеваем НА МЕСТЕ и отдаём свежий»), то есть
«продлится сам» верно только для того, кто каналом пользуется. Фонового процесса, который
делал бы это без него, на проде нет вовсе — см. раздел 5. Осталось ли чем продлевать
(`refresh_expires_at`) — из БД, не снято.

## 2. Установка в каталоге — эти таблицы не в БД helm, а в БД каталога

Запрошенные `installations` / `installation_seen_users` / `profile_version_dependencies`
в схеме helm отсутствуют вовсе: `grep "__tablename__"
/opt/helm/src/helm/infrastructure/db/models.py` их не содержит, `grep -rn
"installation\|pinned\|profile_version_dependencies"` по тому же файлу — пусто. Строки
`iva-write-base` в задеплоенном коде helm тоже нет: `grep -rn "iva-write-base"
/opt/helm/src --include=*.py` → пусто.

Они живут в ДРУГОМ продукте — backend tacticum-dev:
`apps/backend/src/backend/workspace/infrastructure/models.py:89` (`installations`) и `:130`
(`installation_seen_users`), `apps/backend/src/backend/catalog/infrastructure/models.py` +
`apps/backend/alembic/versions/0037_profile_version_dependencies.py`.

**Точный адрес, куда идти (прошёл лид 06.08, я туда не ходил):** сервер `tacticum_prod`
= 159.194.224.59, контейнер `tacticum-postgres-1`, база `tacticum_catalog`, пользователь
`catalog`. Моя догадка про project-hub `10.16.0.17:8770` была неверной — верным было только
«искать надо не в helm».

**Чем это кончилось (снял лид, не я).** Наводка про монолитный профиль **опровергнута**:
в момент жалобы Остряков работал в `iva-role-kmp` 0.5.2, где канал ЕСТЬ — `iva-write-base`
0.1.0, позиция 8 в композиции. Монолиты (`iva-web-brownfield` 0.6.3, `iva-kmp-brownfield`
0.12.0, у обоих 0 рёбер) у него тоже есть, но причина не в них. Подробности —
[[razbor-zhaloby-ostryakov-2026-08-06]].

## 3. События по нему за сутки — снято логами

Всё по UTC, `docker logs --since 30h -t helm-helm-1`.

| Время UTC | Что | Источник строки |
|---|---|---|
| 06:58:42 | бот подтвердил личность «Остряков Данил» → d.ostryakov@iva.ru | `bot_iva_write.py:276` |
| 06:59:18, 07:01:29, 07:02:05, 07:06:17 | то же подтверждение личности, ещё 4 раза | `bot_iva_write.py:276` |
| 07:02:06 | project-hub выпустил ключ `scope=mcp prefix=phk_ad91` для члена `1d9d7dc0-7132-4a9b-862c-cde0c8fe504d`, ответ 201 Created | `project_hub_keys.py:251` |
| 07:02:07 | «выпущен ключ Tacticum для d.ostryakov@iva.ru (на адрес d.ostryakov@iva.ru), префикс phk_ad91, **погашено прежних: 0**» | `key_issue.py:291` |
| 07:02:44 | «выдача ключа: показан ключ phk_ad91…» — страница выдачи открыта | `iva_write.py:557` |

Больше `phk_ad91` в логах за 30 часов не встречается (всего 3 вхождения, все выше).

**Канал записи его ни разу не обслужил.** Строку `канал записи: <email> системы=…`
(`iva_write_surface.py:1141`) за 30 часов получили только двое:

```
docker logs --since 30h helm-helm-1 | grep -A2 "канал записи:" | grep -oE "[a-z._-]+@iva\.ru" | sort | uniq -c
     15 n.tarasova@iva.ru
      2 a.bespalov@iva.ru
```

Острякова там нет ни разу.

**Коды ответов на `/mcp/iva-write`, окно `--since 30h` (в разборе лида те же коды за 36ч,
числа больше — это интервал, не расхождение):**

```
  91263 "GET /mcp/iva-write HTTP/1.1" 200
     54 "POST /mcp/iva-write HTTP/1.1" 200
     25 "POST /mcp/iva-write HTTP/1.1" 202
     11 "DELETE /mcp/iva-write HTTP/1.1" 200
      2 "POST /mcp/iva-write HTTP/1.1" 403
      2 "POST /mcp/iva-write HTTP/1.1" 401
      2 "GET /mcp/iva-write HTTP/1.1" 401
```

- оба **403** — 07:38:57, и оба это `канал записи: отказ, нет живых доступов
  (a.bespalov@iva.ru)`, то есть НЕ Остряков;
- **401 — 07:57:58, 08:23:02, 08:23:03, 08:26:12. Из этих четырёх три — НАШИ СОБСТВЕННЫЕ
  ПРОБЫ:** 08:23 — проба лида, 08:26 — проба разведчика (опознано лидом по времени, 12:40).
  К делу относится ровно один — **07:57:58 UTC**, через пять минут после жалобы. Строго
  привязать и его нельзя: 401 выдаётся ДО опознания личности (`require_write_identity`,
  `routers/iva_write.py:113-141` — «нужен bearer-токен» либо «токен не признан»), почты в
  строке нет, а за traefik у всех один IP.

  🔴 Первая редакция этого раздела перечисляла все четыре как неопознанные, и список
  читался как «клиент четырежды стучался и получал отказ». Это был бы неверный довод,
  собранный из нашего же трафика. Своё присутствие в логах прода при разборе надо вычитать
  раньше, чем считать чужое.

**Одна ошибка приложения за 100 минут** — 07:38:19, `starlette.requests.ClientDisconnect`
в `iva_write_surface.py:996` (`call = _jsonrpc(await request.body())`): клиент оборвал
POST на приёме тела. Не 401/403/404, к доступам отношения не имеет.

**Фон, который стоит знать:** `POST http://10.16.0.17:8770/api/internal/resolve` за 30
часов ответил 401 — **95588** раз, 200 — **179** раз. Опознание при этом работает через
второй источник (`HELM_DEV_MCP_URL=https://mcp.tacticum.dev/mcp`,
`HELM_DEV_TOKEN_EMAIL_DOMAINS=iva.ru,tacticum.ru,iva-tech.ru`) — так устроен фолбэк в
`require_write_identity`. Причину такой пропорции я не разбирал.

## 4. Канал жив — да

**Контейнеры** (`docker ps`, 08:24 UTC):

```
helm-helm-1            Up 9 hours         8000/tcp
helm-traefik-1         Up 2 days          0.0.0.0:80->80, 0.0.0.0:443->443
helm-mcp-atlassian-1   Up 6 days          9000/tcp
helm-postgres-1        Up 7 days (healthy) 127.0.0.1:15432->5432
```

Healthcheck объявлен только у postgres (`healthy`); у helm его нет — Traefik проверяет
живость сам, `traefik.http.services.helm.loadbalancer.healthcheck.path=/api/auth/config`,
интервал 5s (`docker inspect helm-helm-1`). В логах эта проверка отвечает 200 непрерывно.

**Маршрут наружу** (`docker inspect helm-helm-1 --format '{{json .Config.Labels}}'`):
`traefik.http.routers.helm.rule=Host('helm.tacticum.ru')`, entrypoint `websecure`,
certresolver `le`, порт сервиса 8000. Отдельного контейнера у канала записи нет:
`mcp-atlassian` traefik-меток не имеет вовсе и наружу не смотрит — helm ходит к нему
внутри compose-сети (`HELM_IVA_WRITE_MCP_URL=http://mcp-atlassian:9000/mcp`).

**Точка монтирования канала** — в коде: `MOUNT_PATH = "/mcp/iva-write"`
(`/opt/helm/src/helm/interface/mcp/iva_write_surface.py:78`), монтируется в
`main.py:233`. Отдельных `/sse` нет, транспорт streamable-http на этом же пути.

### ТОЧНЫЙ URL

```
https://helm.tacticum.ru/mcp/iva-write
```

Проверено пробами с рабочей машины (08:35 МСК):

```
GET  https://helm.tacticum.ru/mcp/iva-write   → 401
GET  https://helm.tacticum.ru/mcp/iva-write/  → 401   (слэш обрабатывается, main.py:238)
GET  https://helm.tacticum.ru/api/auth/config → 200
POST https://helm.tacticum.ru/mcp/iva-write   → 401  {"error":"unauthorized","detail":"нужен bearer-токен"}
```

401 с телом `нужен bearer-токен` — это и есть «поверхность жива, нужен ключ», а не «нет
такого адреса». TLS валиден (`ssl_verify_result=0`). `/healthz`, `/readyz`, `/health`,
`/api/health` на этом хосте дают 404 — их тут просто нет, живость смотрится по
`/api/auth/config`.

Ключ уходит заголовком `Authorization: Bearer <phk_…>`; сама страница выдачи говорит
пользователю положить его в переменную `TACTICUM_TOKEN` и что у Codex это
`bearer_token_env_var = "TACTICUM_TOKEN"` (`routers/iva_write.py:563-570`).

## 5. Автопродления как ПРОЦЕССА на проде нет — подтверждение с сервера

Вынесено из раздела 1 отдельно: это факт о системе, а не об Острякове, и на его случай он
не влиял (там причина другая — ключ не доведён до среды). До этого «таймер не запускается
ничем» было известно **по коду**; ниже — то же самое, снятое **с работающего прода**.

- в коде ручки продления прямо написано: *«Ручной вход в ту же функцию, которую зовёт
  планировщик: своего таймера сервис не заводит»*
  (`/opt/helm/src/helm/interface/api/routers/iva_write.py:648-660`);
- `POST /api/iva/oauth/refresh` за 30 часов **не вызывался ни разу**:
  `docker logs --since 30h helm-helm-1 | grep -c "oauth/refresh"` → **0**;
- **ни одного таймера под это:** `systemctl list-timers --all` — 17 таймеров, все
  системные плюс `iva-rag-refresh`, к продлению доступов отношения не имеет ни один;
- **cron пуст:** `crontab -l` — пусто, в `/etc/cron.d/` только `e2scrub_all` и `sysstat`.

То есть «дальше продлится сам», которое обещает бот, работает **только для того, кто
каналом пользуется**: продление ленивое, на обращении. Для человека, который получил
доступ и ушёл, обещание неверно.

Это прод-подтверждение к дефекту «механизм написан, вызывающего нет» —
третьему случаю той же формы в направлении iva-write.

## Вердикт

```
ВЕРДИКТ: частично
Проверено: живость канала записи iva-write и его публичный URL; события по
  d.ostryakov@iva.ru в логах helm за 30 часов; наличие/отсутствие механизма
  автопродления доступов; наличие таблиц каталога установок в схеме helm.
Данные: время сервера 2026-08-06 08:24:20 UTC. Контейнеров 4, все Up, postgres healthy.
  Окно всех счётчиков ниже — `docker logs --since 30h`. На /mcp/iva-write: 91263 GET 200,
  54 POST 200, 25 POST 202, 11 DELETE 200, 2 POST 403, 2 POST 401, 2 GET 401.
  Обслуженных канал записи: n.tarasova 15, a.bespalov 2, d.ostryakov 0. Оба 403 —
  a.bespalov. Из шести 401 три — наши собственные пробы (08:23 лид, 08:26 разведчик),
  к делу относится один: 07:57:58 UTC. Ключ Tacticum ему выпущен 07:02:07 UTC, префикс
  phk_ad91, погашено прежних 0; после выдачи в логах не появлялся ни разу. Вызовов
  /api/iva/oauth/refresh за 30ч: 0. Таймеров под продление: 0 из 17 systemd-таймеров,
  crontab пуст. Сроки из ответа бота (07:06 UTC): Jira 07:59 UTC, Confluence 08:00 UTC —
  на момент съёма прошли 25 и 24 минуты назад.
Подтверждение: ssh helm 'date -u; docker ps' · docker logs --since 30h -t helm-helm-1
  (grep по ostryakov, phk_ad91, "канал записи", кодам ответов) · docker inspect
  helm-helm-1 (labels, Config.Env) · systemctl list-timers --all · crontab -l ·
  grep по /opt/helm/src (MOUNT_PATH:78, main.py:233, routers/iva_write.py:648-660,
  models.py:3435/3514/3555) · curl с рабочей машины по 4 адресам, вывод выше.
  Дата прогона: 2026-08-06, 08:24–08:36 UTC.
НЕ проверено: (1) состояние external_credential / credential_use_log / key_handout по
  d.ostryakov@iva.ru — SQL не прогонял, гейт воркера режет docker exec, а ssh_db_query в
  сессии не подключён; поэтому expires_at, refresh_expires_at, revoked_at, наличие
  refresh-токена и число продлений — не снятые числа, а не «нули». Позже это снял лид
  сам на tacticum_prod, в моём вердикте они так и остаются неснятыми. (2) Установка в
  каталоге: я установил только, что таблиц нет в БД helm; куда идти (tacticum_prod /
  tacticum_catalog) и что там оказалось — снял лид, я в ту БД не ходил. (3) Чей клиент
  дал 401 в 07:57:58 UTC — 401 отдаётся до опознания личности, почты в логе нет, за
  traefik один IP; остальные три опознаны как наши пробы, но не мной. (4) Причина
  95588 ответов 401 от project-hub /resolve — не разбирал (лид проверил отдельно: не
  поломка, работает фолбэк). (5) Клиентскую сторону (конфиг codex у человека, значение
  его TACTICUM_TOKEN) не смотрел вовсе — со стороны сервера она неразличима.
```

**Что из этого перевесило.** Раздел 3 оказался решающим для всего разбора: выдача
`phk_ad91` в 07:02:06 UTC и отсутствие ключа в логах после неё — это и есть корень
(ключ выдан и не доведён до среды). До этого вывод строился на отметке `key_status:
key_unknown` в журнале, которая была правдой ровно в свою секунду: ключ пришёл двумя
минутами позже согласий. Отметка состояния датирована — сверять её с временем событий
обязательно, иначе состояние на момент читается как постоянное свойство.

## Что нужно было на проде, чего я сделать не мог — ЗАКРЫТО

Прогнать SELECT-ы из разделов 1 и 2. **Сделал лид 06.08 сам** (`tacticum_prod` для каталога,
helm для доступов); мой блокер снят не мной. Изменений на проде не делал никто, только чтение.

## Связано

[[razbor-zhaloby-ostryakov-2026-08-06]] — итоговый разбор с выводом и текстом для человека ·
[[Направление- Write-канал в контур ИВА (iva-write)]]
