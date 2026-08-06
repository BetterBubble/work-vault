---
title: recon-server-ds-2026-07-31 — разведка по прод-серверу и БД установок
type: note
status: current
created: 2026-07-31 12:15
updated: 2026-07-31 12:15
repo: tacticum-dev
project: дизайн-система / пилот KMP+Figma
tags:
- board
- ds
- recon
permalink: tacticum/00-board/recon-server-ds-2026-07-31
---

# Разведка по прод-серверу и БД установок (31.07)

**Главное сразу: на прод-сервер я не попал.** Ни одного факта из прод-БД в этой заметке нет.
Всё, что ниже — из git (`origin/main`, последний fetch 31.07 12:00 MSK) и из кода репозитория.
Раздел «Что НЕ удалось проверить» — не отговорка, а половина результата: без него заметку
можно прочитать как подтверждение, которого не было.

## 1. Доступ к серверу — заблокирован (пункты 1, 2, 3, 5 задания не выполнены)

| Канал | Что вышло |
|---|---|
| MCP `ssh-manager` (`ssh_list_servers`, `ssh_execute`, `ssh_db_query`, `ssh_db_list`) | **Инструменты не подключены к контексту воркера.** В `~/.claude/settings.json` они в allow-листе (строки 165–174), но в наборе тулов их нет — вызов возвращает `No such tool available` |
| `ssh` из Bash | Заблокирован классификатором прав. Плюс в `settings.json` есть deny `Bash(*/.ssh/*)` и `Read(**/.ssh/**)` |
| `curl` на `dev.tacticum.dev` / `mcp.tacticum.dev` | Заблокирован классификатором прав — сеть из Bash воркера недоступна |
| `~/.ssh/config` | `tacticum_dev` там нет вообще, только `zu_demo` |

Реестр серверов ssh-manager лежит в `~/.ssh-manager/.env`. Хост подтверждён, совпадает с runbook:

    SSH_SERVER_TACTICUM_PROD_HOST=159.194.224.59
    SSH_SERVER_TACTICUM_PROD_USER=root
    SSH_SERVER_TACTICUM_PROD_PORT=22
    SSH_SERVER_TACTICUM_PROD_DESCRIPTION=Прод tacticum-dev (catalog-mcp, каталог профилей)

Аутентификация — **пароль** (`SSH_SERVER_TACTICUM_PROD_PASSWORD`; значение не читал: ключа
`KEYPATH` у этой записи нет, в отличие от `zu_demo` и `helm`). Подставлять пароль в `sshpass`
из Bash — обход выставленной границы, а не работа, поэтому не делал.

**Вывод по доступу:** воркер без MCP `ssh-manager` серверную часть выполнить физически не может.
Гонять её должен тот, у кого тулы подключены — лид или ГД в основном окне.
## 2. Версии профилей: handover vs `origin/main`

Это НЕ проверка прода — это проверка репозитория. Прод может отставать от `origin/main`
(сид ручной, см. runbook), поэтому колонка «в проде» ниже — со слов handover, не моя.

| Шаблон | handover: «в проде» | `origin/main` (fetch 12:00) | Разница |
|---|---|---|---|
| лейн `iva-kmp-development-base` | 0.21.0 | **0.22.0** | +1 минор |
| роль `iva-role-kmp` | 0.4.4 | **0.4.5** | +1 патч |
| профиль `iva-kmp-brownfield` | 0.10.0 | **0.11.0** | +1 минор |
| ДС `iva-mobile` | 0.3.0 | **0.3.0** | совпадает |

Расхождение объяснимо и датируемо. Тройка **0.21.0 / 0.4.4 / 0.10.0 — ровно состояние коммита
`f4b883b`** («build resource discipline in kmp-build-verification», 31.07 11:16 +0400 = 10:16 MSK).
Следующий бамп — `d649661` («figma-access-setup — Codex wiring, open-file requirement,
premature-diagnosis anti-pattern», 31.07 12:45 +0400 = **11:45 MSK**) — поднял все три до
0.22.0 / 0.4.5 / 0.11.0 и уже влит в `origin/main` (проверено `git merge-base --is-ancestor`).

**Практический смысл:** handover писался до `d649661`, либо этот бамп ещё не засижен на проде.
Кто пойдёт на сервер — первым делом сверить засиженные версии с 0.22.0 / 0.4.5 / 0.11.0. Если там
0.21.0 / 0.4.4 / 0.10.0, то на проде НЕТ свежего `figma-access-setup` с Codex-проводкой — а пилот
держится именно за него.

Отдельная ловушка для всех, кто будет смотреть версии локально: **чекаут
`/Users/bubblemac/tacticum/tacticum-dev` отстаёт от `origin/main` на 45 коммитов**
(`main...origin/main [behind 45]`, HEAD `b612fd9`). Локальные `templates/*/manifest.yaml`
показывают 0.19.0 / 0.4.2 / 0.8.0 — **это мусорные числа**. Все версии выше сняты через
`git show origin/main:templates/<profile>/manifest.yaml`.

Версия ДС: `git show origin/main:design-systems/iva-mobile/design-system.yaml` → `version: 0.3.0`,
`status: published`, `platform: mobile`, `framework_hint: compose-multiplatform`,
`source_type: tokens_studio_export`. Совпадает с заявленным в handover.

## 3. Схема `installations` — чего в ней НЕТ (это меняет постановку задачи)

Модель: `apps/backend/src/backend/workspace/infrastructure/models.py:74`, класс `Installation`
(читал по `origin/main`).

Реальные колонки: `id`, `workspace_id`, `profile_id`, `profile_version_pinned`, `label`,
`first_synced_at`, `last_synced_at`, `last_synced_version`, `sync_count`, `created_at`,
`archived_at`.

| Что просили вытащить | Есть ли в таблице |
|---|---|
| UUID, пин, профиль | **Есть**: `id`, `profile_version_pinned`, `profile_id` |
| workspace | **Косвенно**: `workspace_id` → `workspaces.slug` / `.name` (`models.py:30`) |
| **репозиторий** (`D:/iva/kmp`) | **Такой колонки нет вообще.** Server-side привязки установки к репо не существует — это открытый энейблер `installations.kb_repo_id`, US #759, прямо зафиксирован в ADR-0062 п.5: «у Installation нет server-side привязки к репо, `own_repo` невычислим». Значит **«репо D:/iva/kmp» из прод-БД не подтверждается в принципе** — оно приходит из клиентского `.tacticum/context.yaml` |
| **primary-пользователь** | Прямой колонки владельца нет. Ближайшее — таблица `installation_seen_users` (`installation_id`, `user_id`, `last_seen_at`, `models.py:116`), и это **детектор шаринга установки, а не владелец**. Владелец выводится только через `workspaces.account_id` → `accounts` |
| дата обновления | **`updated_at` у установки НЕТ.** Есть `created_at`, `last_synced_at`, `first_synced_at`, `archived_at`. `updated_at` есть только у `workspaces` |

Следствие для вопроса Codex: расхождения по «профиль / workspace / пин / владелец» запросом
снимаются, а «репо» — не снимается ничем, кроме слов человека и клиентского конфига.

## 4. Готовые SELECT для того, у кого есть доступ

Чтобы работа не пропала — запросы под `ssh_db_query` (БД `tacticum_catalog`, только чтение):

```sql
-- (а) три установки целиком + workspace + владелец
SELECT i.id, i.profile_id, i.profile_version_pinned, i.last_synced_version,
       i.sync_count, i.first_synced_at, i.last_synced_at, i.created_at, i.archived_at,
       i.label, w.slug AS workspace_slug, w.name AS workspace_name, w.account_id
FROM installations i JOIN workspaces w ON w.id = i.workspace_id
WHERE i.id::text LIKE '687db0e8%' OR i.id::text LIKE 'cbb20b79%' OR i.id::text LIKE '0a4d393e%';

-- (б) все установки, связанные с KMP — нет ли ещё когорт
SELECT i.id, i.profile_id, i.profile_version_pinned, i.last_synced_version,
       i.sync_count, i.last_synced_at, w.slug
FROM installations i JOIN workspaces w ON w.id = i.workspace_id
WHERE i.profile_id IN ('iva-role-kmp','iva-kmp-brownfield','iva-kmp-development-base')
ORDER BY i.profile_id, i.last_synced_at DESC NULLS LAST;

-- (в) кто реально ходит на эти установки
SELECT installation_id, user_id, last_seen_at FROM installation_seen_users
WHERE installation_id::text LIKE '687db0e8%' OR installation_id::text LIKE 'cbb20b79%';
```

Плюс — какие версии реально засижены (сравнить с 0.22.0 / 0.4.5 / 0.11.0). Имя таблицы версий
каталога я в этой сессии **не проверял** и наугад не пишу: сначала `\dt`, потом выбрать по
`profile_id IN ('iva-role-kmp','iva-kmp-brownfield','iva-kmp-development-base')`.

Здоровье (тоже только чтение, runbook шаг 7):

    curl -sk https://dev.tacticum.dev/healthz
    curl -sk -o /dev/null -w "%{http_code}" -X POST https://mcp.tacticum.dev/mcp \
      -H 'Accept: application/json, text/event-stream' -H 'content-type: application/json' \
      -d '{"jsonrpc":"2.0","method":"initialize","id":1}'      # ждём 401, НЕ 421

## 5. Три расхождения 687db0e8 vs cbb20b79

**Фактами из прод-БД — ноль. Я до БД не дошёл.** Ниже честно: что предъявляю и откуда.

| # | Признак | 687db0e8 | cbb20b79 | Мой источник |
|---|---|---|---|---|
| 1 | Профиль | `iva-kmp-brownfield` | `iva-role-kmp` | handover + разведка по БД от 28.07 на доске — **сам не видел** |
| 2 | Workspace / когорта | org-shared, репо `D:/iva/kmp` | `iva-codex-smoke` | только handover; **репо в БД не хранится вообще**, см. §3 |
| 3 | Пин версии | 0.10.0 (brownfield) | 0.4.4 (роль) | только handover — **сам не видел** |

Единственное независимое подтверждение по `687db0e8` — **не из БД, а с нашей же доски**,
разведка 28.07: `00-Board/ds-vyvody-izuchenie-2026-07-28.md:31` фиксирует
`iva-kmp-brownfield` / `687db0e8` / пин 0.5.4 / **136 синхронизаций** / последняя 28.07 08:08,
и там же полный UUID `687db0e8-a359-48d3-957c-d9b6ad63af99` назван рабочей установкой. Рядом
`00-Board/ds-prod-check-2026-07-28.md:84` — событие `feedback` от неё 27.07 14:34 UTC.
В той же таблице `0a4d393e` — смоук-контур (4 синка, пин 0.5.4, содержимое 0.4.5 от 23.07).

Осторожный вывод: 136 синхронизаций и живой feedback — профиль поведения боевой установки,
а не фикстуры, и профиль там `iva-kmp-brownfield`, не роль. Это согласуется с handover.
Но данные трёхдневной давности, пины с тех пор двигались (0.5.4 → 0.10.0), и
**подтверждением сегодняшнего состояния они не являются**.

### Вердикт: ту ли установку просят проверить

**Вердикт фактами БД я дать не могу — доступа не было.** Честная формулировка: версия
«`687db0e8` — боевая установка чужой когорты, а не пилотная» **не опровергнута и косвенно
поддержана** записями доски от 28.07 (другой профиль, 136 синков, живой feedback против
смоук-профиля у `0a4d393e`). Но это косвенные и устаревшие данные.

Прежде чем что-либо делать с `687db0e8` по просьбе Codex — прогнать запрос (а) из §4. Одна
строка ответа закрывает вопрос окончательно. Пока её нет, любой вердикт — это пересказ
handover с моей подписью, а не проверка.

## 6. ADR-0061 и ADR-0062 — прочитаны (пункт 4 задания, выполнен)

Файлы: `/Users/bubblemac/tacticum/tacticum-dev/docs/adr/0061-kmp-design-system-web-base-and-mockup-trust.md`
(72 строки) и `/Users/bubblemac/tacticum/tacticum-dev/docs/adr/0062-skill-feedback-loop-labels-reader-helm.md`
(76 строк). **Локальные копии совпадают с `origin/main` байт-в-байт** (`git diff --stat origin/main --`
по обоим файлам пуст) — несмотря на отставание чекаута на 45 коммитов, эти два файла не менялись.
Копию `docs/adr` НА СЕРВЕРЕ сверить не смог; формально вопрос «не устарел ли прод по докам»
открыт, хотя доки в образ и не едут — на прод едут `templates/`.

Оба ADR: статус **Accepted**, дата 2026-07-30, владелец mr.diaret@ya.ru.

**ADR-0061, по трём запрошенным темам:**

- *Источники макетов* (решение 3, «гард источников», в лейне с 0.13.0): доверенные входы —
  веб/десктоп фрейм UI KIT (`AG11paSthGC7zSoovfjip0`), живой iva-one, записи DQA-реестра из
  любого раздела. Мобильный фрейм вне DQA → **стоп с переспросом**; фрейм iva-core → **стоп**
  (конференц-поверхность — чужая ДС). Эталонная поверхность KMP — iva-one `@iva/design-system`,
  не iva-core (решение 2).
- *DQA-реестр* (решение 4, Сценарий 5): инструкция реестра версионируется и читается агентом
  живьём при каждом заходе, не вшивается в навык. Маршрутизация записей — навык
  `dqa-registry-intake` (лейн 0.14.0): Баг → `/fix-bug`, задача с эталоном → трек M,
  компонентный уровень → `ds-component-adoption`.
- *Токены* (решение 1): `iva-mobile` — **не самостоятельная ДС, а лист переопределений поверх
  веб-основы**. Типографика и числовые шкалы из веб-источников, join со стилями кода **по роли,
  а не по индексу**; мобильные оверрайды `mobile.*` (push 15, description 13, titleLarge 22),
  iOS `bg (Grouped)`, context — сверху; цвета общие с вебом по значению. Реализовано в 0.3.0
  (PR #176), рецепт сборки — `apps/backend/scripts/design-profiles.yaml`.
- Решение 5, прямо бьющее по пилоту: **геометрия KMP ручная by design** (шкал отступов и
  радиусов в ДС нет и не будет) — сырой `dp` идёт инвентаризацией в отчёт, **гейт не валит**;
  сырой цвет — нарушение всегда; `letterSpacing` исключён из допусков сверки **постоянно**.
- Хвост из «Последствий»: пересборка `iva-mobile` из настоящих Tokens Studio-экспортов сейчас
  невозможна — `docs/concept/design/tokens/` gitignored и на машинах отсутствует.

**ADR-0062, коротко:** универсальное правило `maintainer-feedback` в `tacticum-core-base` 0.2.0
(достаётся всем ролям бесплатно); пять меток `skill-drift: / skill-gap: / skill-conflict: /
alt-approach: / component-gap:` плюс машинная часть в `extra`; читалка
`GET /api-backend/stats/feedback` (scope `stats:read`; **префикс `/api-backend` обязателен —
Traefik голый `/stats/*` не маршрутизирует**); командные сигналы уходят сводкой в helm-реестр,
а не в телеметрию; аудит `kb_cross_read` отложен до энейблера US #759. Отдельно: фидбэк ≠
разрешение отклониться от навыка — отклонение только решением пользователя (`decided_by: "user"`).

## 7. Что НЕ удалось проверить и почему

| Пункт задания | Статус | Причина |
|---|---|---|
| Зайти на прод-сервер | **не выполнено** | MCP `ssh-manager` не подключён к контексту воркера; `ssh` и `curl` из Bash блокирует классификатор прав |
| Факты по трём установкам из `installations` | **не выполнено** | нет доступа к БД: полные UUID, пины, workspace, даты — не сняты |
| Поиск других KMP-установок и пилотных когорт | **не выполнено** | то же |
| Реально засиженные версии профилей в проде | **не выполнено** | сверил только handover vs `origin/main` (§2) — это репозиторий, не прод |
| Владельцы 687db0e8 / cbb20b79 | **не выполнено** | доступа нет; вдобавок прямой колонки владельца в схеме не существует (§3) |
| Три расхождения фактами БД | **не выполнено** | §5: есть только косвенные данные с доски от 28.07 |
| `/healthz` | **не проверен** | сеть из Bash воркера недоступна |
| `POST /mcp` → 401 vs 421 | **не проверен** | то же. Гоча runbook (шаг 7) остаётся непроверенной — **это дыра, её надо закрыть тем, у кого есть доступ** |
| ADR-0061 / ADR-0062 прочитаны | **выполнено** | §6 |
| Сверка `docs/adr` на сервере с локальной | **не выполнено** | нет доступа к серверу; локальная копия = `origin/main`, это проверено |

Правок не делал ни одной, SQL не выполнял, на сервер не заходил, ничего не деплоил.
Граница «только чтение» соблюдена — в том числе там, где чтения не получилось.

---

# Дополнение 13:30 — разбор дословной формулировки Codex

Запрос дословно: «Проверить, активна ли installation `687db0e8-a359-48d3-957c-d9b6ad63af99`
в workspace `iva/base`, и привязан ли к ней профиль `iva-kmp-brownfield` с правами KMP/design read.»

Доступа к БД у меня по-прежнему нет — ДА/НЕТ по живым строкам я не даю. Но по каждому из трёх
утверждений код отвечает на вопрос «а есть ли вообще такое поле и что оно значит», и два из трёх
утверждения оказываются сформулированы в терминах, которых в схеме не существует. Это тоже ответ.

## Д1. «Активна» — поля `is_active` у установки НЕТ

В `installations` нет ни `is_active`, ни `deleted_at`, ни `status`. Удаление — **мягкое, через
`archived_at`**: `apps/backend/src/backend/workspace/interface/admin/installations.py:241,282`
(«Soft-delete: set `archived_at=now()`. Per Q4-A grilling 2026-05-10»).

Но «активна» на практике — **составное условие из четырёх частей**, и проверять надо все, иначе
ответ будет ложно-положительным. Собрано из трёх независимых мест кода:

| Условие | Где проверяется |
|---|---|
| `installations.archived_at IS NULL` | `identity/infrastructure/token_db.py:44`, `whoami.py:60`, `membership_installation_scope.py:121` (ошибка `installation_archived`) |
| `workspaces.archived_at IS NULL` | `token_db.py:45` — архивированный воркспейс убивает установку молча |
| `profiles.is_active = TRUE` | `catalog/infrastructure/models.py:71`; фильтр в `whoami.py` и `membership_installation_scope.py`. **`is_active` в схеме есть — но у ПРОФИЛЯ, не у установки** |
| активная/trial подписка аккаунта | `token_db.py` (`Subscription.status IN ('active','trial')`) — иначе `resolve_scope_from_db` вернёт `None` и авторизация упадёт целиком |

Практический смысл: установка может быть не-архивирована и при этом невидима в `whoami` —
если профиль помечен `is_active=false` или архивирован воркспейс. Ответ «да, активна» без этих
трёх дополнительных проверок неполон.

## Д2. Workspace `iva/base` — такой строки в БД быть не может

Регулярка слага воркспейса: `_SLUG_RE = ^[a-z0-9](?:[a-z0-9-]{0,126}[a-z0-9])?$`
(`workspace/interface/admin/workspaces.py:27`). **Слэш в слаг не проходит** — ни при создании,
ни при PATCH (обе точки валидируют одной и той же регуляркой, строки 65 и 169).

`iva/base` — это склейка **двух разных полей**, которые `whoami` отдаёт раздельно
(`identity/interface/mcp/whoami.py`, функция `_entry`): `organization_slug` (из `organizations.slug`
через `Account.organization_id`) и `workspace_slug` (из `workspaces.slug`).

Значит утверждение №2 надо читать как «организация `iva` **и** воркспейс `base`», и проверять
двумя сравнениями, а не одним. Искать в БД строку `iva/base` — искать то, чего там нет by design.

Заодно: `workspaces` — единственная из трёх таблиц, у которой есть `updated_at`; у установки его
нет (§3).

## Д3. «Права KMP/design read» — такого поля нет ни у установки, ни у токена

**Скоупов `design:*` в системе не существует.** Греп по всему бэкенду даёт ровно четыре скоупа:
`catalog:read`, `kb:read`, `kb:write`, `stats:read`. Ничего про design. Поле `ApiToken.scopes`
есть, но `design`-значений в нём не бывает.

У самой установки поля прав нет вообще — ни колонки, ни связанной таблицы разрешений.

Доступ к инструментам `design_*` определяется **двумя условиями, и ни одно не живёт на установке**:

**(1) Премиум-гейт по tier.** `design/interface/mcp/common.py`, `require_premium_tier(scope)`:
падает с `AuthError("seat_required")`, если `scope.tier not in ("full","trial")`. Вызывается в
`design_list_systems`, `design_get_tokens`, `design_get_theme_tokens`, `design_resolve_token`.

**Важное:** `tier` — свойство **подписки аккаунта**, а не установки. Для legacy `tac_*`-токенов
`tier = "trial" if sub.status == "trial" else "full"` (`token_db.py`). А для `phk_*`-ключей —
это ключи Codex и Claude — **tier захардкожен `tier="full"`** в
`build_authscope_from_membership` (`identity/application/membership_installation_scope.py`).

Отсюда вывод, который снимает половину гипотезы: **премиум-гейт не может быть причиной, по
которой у них «design read не работает»**. Для phk-ключа он проходится всегда, а если активной
или trial-подписки у организации нет, то падает не design, а вся авторизация целиком
(`no_active_subscription`) — сломались бы и `catalog:read`, и `whoami`, а не только токены.

**(2) Привязка дизайн-системы к воркспейсу — вот настоящий кандидат.** Таблица
`workspace_design_systems` (`design/infrastructure/models.py:168`, класс `WorkspaceDesignSystem`):
композитный PK `(workspace_id, design_system_id)`, плюс `design_system_version_pinned`,
`attached_at`, `attached_by_user_id`. Привязка делается админ-ручкой
`design/interface/admin/workspace_attach.py`.

Поведение без привязки — **разное и одно из них тихое**:

- `design_list_systems` возвращает **пустой список без всякой ошибки** (docstring прямо:
  «empty list if no attachments»). Агент это читает как «дизайн-систем нет» — ровно тот же
  сорт грабель, что «пустой `kb_discover` принимали за KB недоступна» (гоча §3.3 handover);
- `design_get_tokens` бросает `DesignSystemNotFoundError`: «DesignSystem {slug} not attached to
  current Workspace; call `design_list_systems` first».

**(3) Третий, отдельный режим отказа для phk-ключей.** `phk_*` не пинует установку. Если у
организации доступных установок **не ровно одна**, резолвер бросает
`AuthError("installation_id_required")` — «Membership keys cover multiple installations; pass
installation_id explicitly». У KMP-когорты установок заведомо больше одной, значит Codex обязан
передавать `installation_id` явно; без него `design_*` упадёт независимо от прав и привязок.

### Что из этого проверяется SELECT-ом

```sql
-- Д3: привязаны ли ДС к воркспейсу установки (главная проверка «design read»)
SELECT ds.design_system_id, ds.name, ds.platform,
       wds.design_system_version_pinned, wds.attached_at
FROM installations i
JOIN workspace_design_systems wds ON wds.workspace_id = i.workspace_id
JOIN design_systems ds ON ds.id = wds.design_system_id
WHERE i.id = '687db0e8-a359-48d3-957c-d9b6ad63af99';
-- пусто = design_list_systems молча вернёт [] и design_get_tokens упадёт

-- Д1: активность целиком, все четыре условия сразу
SELECT i.id, i.archived_at AS inst_archived, w.archived_at AS ws_archived,
       p.is_active AS profile_active, s.status AS sub_status, s.plan,
       o.slug AS org_slug, w.slug AS ws_slug,
       i.profile_id, i.profile_version_pinned, i.last_synced_version, i.sync_count
FROM installations i
JOIN workspaces w ON w.id = i.workspace_id
JOIN accounts a ON a.id = w.account_id
LEFT JOIN organizations o ON o.id = a.organization_id
LEFT JOIN profiles p ON p.id = i.profile_id
LEFT JOIN subscriptions s ON s.account_id = a.id AND s.status IN ('active','trial')
WHERE i.id IN ('687db0e8-a359-48d3-957c-d9b6ad63af99')
   OR i.id::text LIKE 'cbb20b79%';
```

Второй запрос закрывает Д1 и Д2 одной строкой на установку и сразу даёт сравнение с пилотной
`cbb20b79`. Имена `profiles` / `subscriptions` / `organizations` взяты из импортов моделей;
если реальные `__tablename__` отличаются — сверить по `\dt`.

## Д4. Сводка ответов по трём утверждениям Codex

| # | Утверждение Codex | Мой ответ |
|---|---|---|
| 1 | установка активна | **Фактом БД — не проверил.** Поля `is_active` у установки НЕТ; «активна» = `archived_at IS NULL` у установки И у воркспейса, плюс `profiles.is_active` и живая подписка |
| 2 | workspace `iva/base` | **Так записать нельзя.** Слаг воркспейса не допускает `/`; `iva/base` = `organization_slug=iva` + `workspace_slug=base`, два отдельных поля из `whoami` |
| 3 | профиль `iva-kmp-brownfield` с правами «KMP/design read» | Привязка профиля — **есть поле** (`installations.profile_id`), проверяется. **Прав — нет такого поля вообще**: скоупов `design:*` в системе не существует (есть только `catalog:read`, `kb:read`, `kb:write`, `stats:read`), у установки колонки прав нет. Доступ к `design_*` = премиум-tier аккаунта + строка в `workspace_design_systems` |

**Главное, что стоит передать Codex:** формулировка «профиль с правами KMP/design read» описывает
модель разрешений, которой в системе нет. Если у них `design_*` не отдаёт данные — смотреть надо
не на права установки, а на три вещи по порядку: привязка ДС к воркспейсу (пустой список приходит
**молча**, без ошибки), передан ли `installation_id` явно (для phk-ключа при нескольких установках
он обязателен), и только потом на подписку аккаунта. Премиум-гейт для phk-ключей проходится
всегда — `tier="full"` там захардкожен, так что версия «не хватает премиума» отпадает.
