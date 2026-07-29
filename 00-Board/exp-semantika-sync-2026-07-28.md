---
title: 'Разведка: семантика last_synced_version / profile_version_pinned / telemetry.profile_version'
type: note
status: current
created: 2026-07-28
updated: 2026-07-28
tags:
- board
- prod
- catalog
permalink: tacticum/00-board/exp-semantika-sync-2026-07-28
---

# Разведка: семантика полей синхронизации (инцидент `iva-role-go`, 28.07)

Репозиторий `~/tacticum/tacticum-dev`, ветка `main`, только чтение. Все ссылки — `файл:строка` от корня репо.

## Главный вердикт

**По имеющимся прод-данным различить нельзя — но перевес на стороне «на диске 0.1.1».**

`telemetry_events.profile_version` **не сообщается клиентом**: сервер подставляет туда
`installations.profile_version_pinned` в момент записи события
(`apps/backend/src/backend/platform/usage_telemetry.py:90`). Поэтому `profile_version=0.2.1`
в потоке `tool_call` за 27–28.07 доказывает ровно одно: **пин в БД равен 0.2.1**. О содержимом
на диске у человека это не говорит ничего.

Единственное поле каталога, которое хоть как-то отражает факт доставки, —
`last_synced_version`, и оно равно `0.1.1`, причём `last_synced_at` пишется **тем же UPDATE**
(`pull_installation_content.py:107–118`), то есть с 2026-07-23 06:56:55 **ни одного
`pull_installation_content*` не было**. Значит по «официальному» каналу доставки 0.2.1 не уезжало.

Оговорка, которая и делает вывод неполным: **есть второй канал доставки контента, который
`Installation` не трогает вообще** — `tacticum_init` / `tacticum_init_manifest` /
`tacticum_fetch_action` (см. Q1). Если человек ставил роль через них, у него на диске может
лежать 0.2.1 при `last_synced_version=0.1.1`. Данные каталога этот случай не различают.

---

## Q1. Кто и когда пишет `installations.last_synced_version`

**Вердикт: ровно два места записи, оба — `pull_installation_content*`. Устареть может, и это
штатное поведение, а не баг.**

Места записи (других нет во всём `apps/backend/src`):

- `apps/backend/src/backend/workspace/interface/mcp/pull_installation_content.py:114`
- `apps/backend/src/backend/workspace/interface/mcp/pull_installation_content_manifest.py:118`

Оба пишут одним `UPDATE` вместе с `last_synced_at`, `first_synced_at`, `sync_count`
(`pull_installation_content.py:107–123`). Отсюда инвариант: **`last_synced_version` и
`last_synced_at` всегда согласованы** — `last_synced_at=2026-07-23` означает, что последний
pull был 23.07 и отдал 0.1.1. Значение — `target.version`, то есть **фактически отданная**
версия, а не пин.

Не пишут это поле:

- `check_updates` — **только читает** (`check_updates.py:47`, `has_update = last_synced != pinned` — строка 51).
- Админский re-pin `PATCH /admin/installations/{id}` — трогает только пин
  (`workspace/interface/admin/installations.py:211–215`).
- Событий `applied` / `update_applied` **никто в коде не пишет**. В CHECK-констрейнте они есть
  (`platform/telemetry_models.py:53–56`), но живых writer'ов ровно три:
  `usage_telemetry.py:92` (`tool_call`), `workflow/interface/mcp/submit_feedback.py:35` (`feedback`),
  `platform/telemetry_ingest.py:56` (внешний ingest). То есть «apply» как серверное событие
  не существует — спрашивать про него нечего.

**Как остаётся устаревшим при реально полученном новом содержимом.** Каталожный путь
`tacticum_init` / `tacticum_init_manifest` / `tacticum_fetch_action` отдаёт полный контент и
**ни одной строкой не касается `Installation`** (в `catalog/interface/mcp/*.py` символ
`Installation` не встречается вовсе). Хуже: эти инструменты отдают **latest active**, игнорируя
пин (`tacticum_init_manifest.py:92–100`, `tacticum_init.py:120–128`). Человек получает свежий
контент — счётчики установки не двигаются. Это ровно тот workaround, который описан в
`docs/agents/codex-init.md:254`.

Бэкфилла не было: миграция 0020 оставляет NULL у существующих строк намеренно
(`apps/backend/alembic/versions/0020_installation_last_synced_version.py:13–20`). Значит
`0.1.1` в проде — след **настоящего** pull'а, а не заливки по умолчанию.

## Q2. Кто и когда пишет `installations.profile_version_pinned`

**Вердикт: re-pin меняет только указатель в БД. На диск у пользователя не влияет никак.**

- Админский PATCH: `workspace/interface/admin/installations.py:211–215` — присваивает поле после
  проверки, что версия засижена и `active` (`_validate_active_version`, строки 57–78). Больше
  ничего: ни `last_synced_version`, ни `last_synced_at` не двигаются, наружу ничего не шлётся.
- `pull_installation_content*` с `update=True` — двигает пин на отданную версию
  (`pull_installation_content.py:116–117`, `pull_installation_content_manifest.py:120–121`).
- Создание установки: админский POST (`admin/installations.py:115–121`), выдача токена
  (`platform/admin/router.py:308–313`, ставит latest active), провижининг
  (`workspace/application/provision_installation.py:228, 251`).
- Прод-практика re-pin — прямой SQL `UPDATE installations SET profile_version_pinned=...`
  (`docs/runbooks/prod-seed-2026-07-27-rollback.md:147–152`). Там же прямым текстом:
  «Установка получит прежнюю версию **при следующей синхронизации**» (строка 172) — то есть
  документ сам называет re-pin указателем, а не доставкой.

Отдельная находка мимо вопроса: админская сериализация установки **не отдаёт
`last_synced_version`** (`admin/installations.py:42–54` — есть `last_synced_at` и `sync_count`,
поля версии нет). Через админку/UI это значение не видно вообще; смотреть можно только в БД.

## Q3. Откуда берётся `telemetry_events.profile_version` у `tool_call`

**Вердикт: подставляет СЕРВЕР из пина установки. Клиент свою версию не сообщает нигде.**

`apps/backend/src/backend/platform/usage_telemetry.py:86–96`:

```python
inst = await session.get(Installation, installation_id)
if inst is not None:
    profile_id = inst.profile_id
    profile_version = inst.profile_version_pinned
```

То же самое у `feedback`: `submit_feedback.py:38` пишет `scope.profile_version_pinned`.
Внешний ingest (`telemetry_ingest.py:56–60`) версию не заполняет вовсе.

Следствие: **ни одна строка `telemetry_events` ни при каком `event_type` не несёт версию,
сообщённую клиентом.** Поток `tool_call` с `profile_version=0.2.1` — это просто отражение
текущего пина, переписанное на каждое событие. Как доказательство состояния диска телеметрия
непригодна by design.

Обратной связи по содержимому тоже нет: `body_hash` отдаётся клиенту наружу
(`pull_installation_content.py:97`), но **приёмника хешей от клиента в бэкенде не существует** —
ни одного эндпоинта, принимающего хеши локальных файлов, в `apps/backend/src` нет.

## Q4. Почему `target_cli` пуст у `tool_call`

**Вердикт: middleware телеметрии его просто не заполняет. Это не «только на applied/pull» —
это дыра ровно в основном писателе.**

`UsageTelemetryMiddleware._record` конструирует `TelemetryEvent` **без `target_cli`**
(`usage_telemetry.py:91–102`); из аргументов вызова он достаёт только `installation_id` и
`kb_run_id` (строки 69–76). Аргумент `target_cli`, который клиент реально передаёт в
`pull_installation_content(target_cli=...)` / `tacticum_init_manifest(target_cli=...)`,
на этом пути **выбрасывается**.

Кто вообще пишет `target_cli` в события — только двое:
`submit_feedback.py:40` (из аргумента инструмента) и `telemetry_ingest.py:60` (внешний ingest,
дефолт `iva-read`). Поэтому `target_cli IS NULL` у всех `tool_call` — ожидаемо и о клиенте
не говорит ничего.

## Q5. Можно ли по данным каталога узнать содержимое на диске и CLI

**Вердикт: содержимое — нет, никак. CLI — не из БД, но из логов приложения можно.**

- **Содержимое диска.** Обратного канала нет: клиент серверу ничего не подтверждает,
  хешей не возвращает, событий `applied` никто не пишет (Q1). Максимум, что есть, —
  `last_synced_version` + `last_synced_at` + `sync_count`, и они покрывают **только**
  путь `pull_installation_content*`.
- **CLI.** В БД — нет (Q4). Но `target_cli` уходит в structlog на каждом вызове:
  `pull_installation_content.py:133`, `tacticum_init.py:152`, `tacticum_init_manifest.py:131`,
  `tacticum_fetch_action.py:100`. То есть по логам сервиса за 27–28.07 (события
  `tacticum_init_manifest_called` / `tacticum_fetch_action_called` /
  `installation_content_pulled`) CLI и отданная версия восстанавливаются точно.
- **Дешёвый косвенный признак из БД** — разрез телеметрии по `tool_name`
  (поле пишется, `usage_telemetry.py:98`):

  ```sql
  SELECT tool_name, count(*), min(occurred_at), max(occurred_at)
  FROM telemetry_events
  WHERE installation_id = '<id>' AND occurred_at >= '2026-07-23'
  GROUP BY tool_name ORDER BY 2 DESC;
  ```

  Появление `tacticum_init_manifest` / `tacticum_fetch_action` после 23.07 = человек забирал
  контент мимо пина и на диске у него **latest active**, а не 0.1.1. Отсутствие этих вызовов
  при `last_synced_at=2026-07-23` = на диске 0.1.1.

## Грабля вокруг: «latest» — это последний засиженный, а не старший semver

`latest_active` сортирует по `seeded_at DESC`, не по версии
(`workspace/interface/mcp/common.py:30–40`); так же `tacticum_init.py:120–128`,
`tacticum_init_manifest.py:92–100`, `platform/admin/router.py:293–302`.

У `iva-role-go` история версий немонотонна: 0.2.0 — 22.07, **0.1.1 — 23.07**, 0.2.1 — 24.07
(`templates/iva-role-go/CHANGELOG.md:5, 12, 36`). Поэтому 23.07 «latest active» был именно
0.1.1, и pull того дня штатно отдал его. Состав тоже расходится по существу: 0.2.0 убрал
`iva-analysis-base` из `depends_on`, а 0.2.1 добавил `tacticum-lite-base` +
`tacticum-research-base` (`templates/iva-role-go/manifest.yaml:18–24`, CHANGELOG строки 5–24).
Ветка 0.1.x `iva-analysis-base` всё ещё несёт.

Замечание к матчасти: в `templates/` на `main` каталога `tacticum-lite-base` **нет** — ребро в
манифесте 0.2.1 есть, шаблона рядом нет. Проверить, чем это ребро закрывается при сиде, стоит
отдельно (это вне моей разведки).

## Что осталось непроверенным

- Прод-БД я не трогал: все выводы — из кода на `main`. Совпадает ли задеплоенный прод с этим
  `main`, не проверял.
- Логи сервиса за 27–28.07 не читал — доступа нет; запрос по `tool_name` выше выполнять тоже
  не мне.