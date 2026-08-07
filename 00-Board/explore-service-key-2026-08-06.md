---
title: "Разведка: сервисный ключ для Taiga проекта 12 (TAIGA_TOKEN)"
type: note
permalink: tacticum/00-board/explore-service-key-2026-08-06
status: draft
date: 2026-08-06
---

# Разведка: сервисный ключ для Taiga проекта 12

## ВЕРДИКТ-0: существует ли уже пригодный ключ

**НЕТ.** Пригодного сервисного ключа не существует. Ближайший по факту работоспособности —
`phk_535c`, но он **личный**: принадлежит `a.shulga@tacticum.ru` и открывает семь Taiga-проектов,
а не один.

Проверил все пять мест: таблицу ключей на проде (242 ключа), три префикса на руках, окружение
контейнеров, конфигурации на машине, кейчейн и переменные окружения.

### Три ключа на руках — ни один не сервисный

| Префикс | Владелец | Скоуп | Имя | Создан | Состояние | Что видит в Taiga |
|---|---|---|---|---|---|---|
| `phk_54ae` | a.shulga@**cifragen**.ru | mcp | `codex-oidc` | 2026-06-26 | активен | **только проект 28** (zus-codex) |
| `phk_abb7` | a.shulga@**tacticum**.ru | mcp | `taiga+wiki mcp_2` | 2026-06-29 | **ОТОЗВАН** | ничего, ключ мёртв |
| `phk_535c` | a.shulga@**tacticum**.ru | mcp | `iva-write bot for a.shulga@tacticum.ru` | 2026-08-04 | активен | **7 проектов, включая 12** |

Все три принадлежат живым людям (двум разным учёткам Александра Шульги). Сервисным по критерию
«отдельная личность» не является ни один.

Членства владельцев (определяют, что ключ видит через taiga-mcp):

- `a.shulga@cifragen.ru` — editor на `cifragen/zus-codex` → Taiga-проект **28**. Один проект.
- `a.shulga@tacticum.ru` — editor на `tacticum/tacticum-dev` (**12**), `tacticum/agents` (18),
  `tacticum/platform` (20), `tacticum/codex` (22), `tacticum/helm` (32), `cifragen/zus-codex` (28),
  `iva/control-tower` (31, роли editor + CEO), плюс тенант-уровневые CPO и COO в `iva`.

**Итог по трём:** `phk_535c` — тот самый, что сейчас работает на проекте 12. Именем он
`iva-write bot`, но по сути это личный ключ человека: отозвать его — сломать работу Александра,
а не только бота, и он даёт доступ ещё к шести проектам сверх нужного.

**Поправка к постановке:** «один рабочий на проект 12 — личный админский `mr.diaret@ya.ru`» —
неверно вдвойне. Рабочий на проект 12 это `phk_535c`, владелец `a.shulga@tacticum.ru`, и это не
админская учётка (platform-админы: `d@cifragen.ru`, `d.lebedev@tacticum.ru`).
`mr.diaret@ya.ru` пользователем хаба не является вовсе — это учётка Taiga (`users_user.id=5`).
«Второй видит только проект 28» — это `phk_54ae`. «Третий мёртв» — это `phk_abb7`, отозван.

### Сервисные учётки: есть две, обе не годятся

| Учётка | Ключ | Скоуп | Членство | Почему не годится |
|---|---|---|---|---|
| `arch-sync@tacticum.ru` «Dev Arch Sync (service)» | `phk_5927`, активен | mcp | editor, тенант `tacticum`, **тенант-уровень** | `taiga_project_id` берётся только из проектного членства (`api/internal.py:266`) → множество разрешённых проектов **пустое** → taiga-mcp откажет во всём, включая проект 12 |
| `helm-service@tacticum.ru` «helm (служебная учётка канала записи iva-write)» | `phk_3696`, активен, использован 2026-08-06 | write:users | tenant_admin, тенант **`iva`**, тенант-уровень | чужой тенант, Taiga-проектов нет; это ключ, которым helm выпускает iva-write-ключи |

Обе — правильные по форме сервисные учётки, и это доказывает, что схема рабочая. Но ни одна не
имеет **проектного** членства на `tacticum-dev`, а без него ACL taiga-mcp не пропустит.

### Сервисные ключи без владельца — структурно непригодны

Восемь ключей с `user_id IS NULL`: `phk_036b` codex-doctranslate, `phk_f85a` helm-resolve,
`phk_1429` mcp-runtime, `phk_b1cc` knowledge-rag, `phk_09ed` codex, `phk_3f73` memory-service,
`phk_575a` basic (все скоуп `resolve`), `phk_9b33` wiki-digest service
(скоуп `wiki_digest_recipients`).

Использовать любой из них как токен для taiga-mcp **нельзя в принципе**: `/api/internal/resolve`
отвергает ключ без владельца — `if api_key is None or api_key.user_id is None: 401`
(`api/internal.py:286-287`). Это ключи «для входа в служебный API», а не «для работы от лица».

### Ключи с говорящими именами — все личные

Прогнал имена по шаблону `ci|bot|service|github|action|runner|pipeline|automation|agent|token|sync`
— 34 совпадения. Ни одного, выданного не человеку, кроме двух сервисных учёток выше. Наиболее
«ботовидные» — серия `iva-write bot for <почта>` (9 штук на `a.shulga@tacticum.ru`, из них 8
отозваны; 1 на `d.ostryakov@iva.ru`) — это ключи, выдаваемые **людям** для канала записи, а не
ботам. Имя `bot` в них означает назначение, а не отдельную личность.

### Конфигурации на машине

Всего на машине встречается **11 различных ключей** формата `phk_*`. Но в **действующих
конфигурациях** — только два:

| Файл | Ключи |
|---|---|
| `~/.claude.json` | `phk_535c…b093`, `phk_abb7…6ec5` |
| `~/tacticum-vault/.mcp.json` | `phk_535c…b093`, `phk_abb7…6ec5` |
| `~/tacticum/.mcp.json` | `phk_abb7…6ec5` — **только отозванный** |
| `~/tacticum/_analyst-e2e/.mcp.json` | `phk_abb7…6ec5` — **только отозванный** |
| `~/tacticum-vault/.mcp.json.bak`, `.mcp.json.bak-2026-08-06` | `phk_abb7…6ec5` |
| бэкапы `~/.claude/backups/.claude.json.*` | `phk_535c…b093`, `phk_abb7…6ec5` |

**Побочная находка, к делу не относится, но скажу:** в `~/tacticum/.mcp.json` и
`~/tacticum/_analyst-e2e/.mcp.json` прописан **только отозванный** `phk_abb7`. Эти конфигурации
нерабочие — если ими кто-то пользуется, он получает 401 и, вероятно, считает это поломкой сервера.

Остальные девять ключей найдены **только в логах сессий** (`~/.claude/projects/**.jsonl`,
`~/.claude/history.jsonl`, `90-Materials/rollout-*.jsonl`) — это транскрипты прошлой работы, а не
конфигурации. Из них: `phk_1a5f`, `phk_27e3`, `phk_3cc6`, `phk_4166` — отозванные iva-write;
`phk_ad91` — активный, но чужой (`d.ostryakov@iva.ru`); `phk_0123…cdef` — очевидная заглушка из
примера; `phk_94c0…25c1` и `phk_b5f2…1b68` — **в таблице ключей отсутствуют вовсе**
(либо из другого контура, либо выдуманы в тексте транскрипта).

### Кейчейн и переменные окружения

Кейчейн: записей, относящихся к Taiga / tacticum / cifragen / project-hub, **ноль**
(`security dump-keychain`, поиск по этим подстрокам — 0 совпадений).
Переменные окружения оболочки с `taiga|phk|mcp|project_hub`: **нет ни одной**.

### Переменные с ключами на сервере (только имена)

| Контейнер | Переменные |
|---|---|
| `infra-taiga-mcp-1` | `MCP_API_KEY`, `PROJECT_HUB_KEY`, `TAIGA_ADMIN_PASSWORD`, `TAIGA_MCP_AUTH_SOURCE` |
| `infra-wiki-mcp-1` | `PROJECT_HUB_KEY`, `WIKIJS_API_KEY` (без `MCP_API_KEY`) |
| `infra-project-hub-mcp-1` | `MCP_API_KEY`, `PROJECT_HUB_SERVICE_KEY` |
| `infra-arch-mcp-1` | `PROJECT_HUB_KEY`, `PROJECT_HUB_SERVICE_KEY`, `ARCH_MCP_SERVICE_KEY`, `ARCH_MCP_AUTH_SOURCE`, `ARCH_LITELLM_API_KEY`, `ARCH_TEI_API_KEY` |
| `infra-excel-mcp-1`, `infra-word-mcp-1` | `MCP_API_KEY`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY` |
| `infra-transcription-mcp-1` | `PROJECT_HUB_KEY`, `TRANSCRIPTION_MCP_AUTH_SOURCE`, `MINIO_*` |

Режимы: `taiga-mcp` → `auth`, `arch-mcp` → `auth`; у `wiki-mcp` и `project-hub-mcp` переменной
`AUTH_SOURCE` нет вовсе.

**Что такое `MCP_API_KEY`.** Это **не** phk-ключ: длина 64 символа, префикса `phk_` нет.
Значит, он не привязан ни к какой личности и через `/resolve` не проходит — это общий статический
секрет «пустить в транспорт». Сравнил значения по sha256 (сами значения не выводил): у
`taiga-mcp`, `excel-mcp` и `word-mcp` он **один и тот же** (хеш `9c9abd42fe7a…`), у `wiki-mcp` пуст.
`PROJECT_HUB_KEY` у taiga-mcp — формата `phk_`, это служебный ключ скоупа `resolve`.

Практический вывод: **как сервисный ключ для Taiga `MCP_API_KEY` не годится и не даёт ничего** —
в режиме `auth` taiga-mcp его вообще не читает (`__main__.py:66-73`). Для `excel-mcp` и `word-mcp`
он остаётся единственным гейтом, и он у них общий: кто знает его для одного, входит во все три.

**Риск, который стоит увидеть отдельно.** Весь ACL taiga-mcp держится на одной переменной. Если
`TAIGA_MCP_AUTH_SOURCE` вернуть в `arg`, middleware сменится на `StaticKeyAuthMiddleware`,
личности в scope не будет, `get_identity()` вернёт `None`, и **все проверки станут пропускать**
(`_check_project`/`_require_project`/`_require_admin` возвращают `None` при `identity is None` —
`server.py:160-161, 169-170, 178-179`). Тогда любой держатель общего `MCP_API_KEY` получает
админские права на все Taiga-проекты. Сейчас режим `auth`, всё в порядке — но переключатель
односимвольный и без предупреждения.

---

## ВЕРДИКТ-1: если выпускать новый — что он даст

**1. Выпустить ключ можем сами, без браузера.** Постановка «выпуск только через веб» неверна:
в `api/admin.py` есть Bearer-доступный маршрут выпуска, добавленный именно для этого случая.
Вся цепочка (пользователь → членство → ключ) делается по HTTP тем ключом, что у нас уже есть.

**2. Главный вопрос — даёт ли сервисный ключ реальное ограничение прав в Taiga.**
Ответ: **частично, и не то ограничение, которого вы ждёте.**

- Постановка «любой валидный Bearer исполняет запросы правами админа Taiga, включая
  `delete_issue`/`delete_project`» — **неверна**. ACL есть, он живой, и на проде включён.
- Но ACL ограничивает **проект, а не операцию**. Ключ, привязанный к личности с членством
  только в проекте 12, **не увидит другие проекты — и при этом сможет удалять задачи в
  проекте 12**. `delete_issue`, `delete_user_story`, `delete_task` внутри разрешённого
  проекта доступны полностью.
- Роль в хабе (`viewer` / `editor`) на набор доступных инструментов taiga-mcp **не влияет
  вообще**. Единственное, что она меняет, — `create_project`/`delete_project`.
- Скоуп ключа (`mcp` / `read`) на ACL taiga-mcp **не влияет вообще**.

**Формулировка для решения:** через taiga-mcp сервисный ключ даёт **изоляцию по проекту,
отзываемость и атрибуцию в аудите** — но НЕ даёт защиты «только чтение и комментарии».
Ограничение «нельзя удалять» через MCP недостижимо без правки кода.

**3. Настоящие права даёт только обход MCP** — прямой REST Taiga под обычным пользователем
Taiga. Там работает родная модель прав Taiga. Оговорка: из шести штатных ролей проекта 12
**ни одна не годится как есть** — даже `Stakeholder` имеет `delete_issue`. Нужна кастомная
роль в проекте 12.

---

## Проверено

### Данные: ACL в taiga-mcp существует и включён на проде

Постановка опиралась на строку описания сервера «All tools operate on a single Taiga instance
authenticated via admin credentials» (`taiga-mcp/src/taiga_mcp/server.py:55`). Строка верна
буквально — клиент действительно один и админский (`server.py:61-73`), — но она устарела
относительно кода: поверх клиента добавлен слой ACL.

| Что | Где |
|---|---|
| Резолвнутая личность достаётся из ASGI-scope | `server.py:131-140` (`get_identity`) |
| Проверка проекта | `server.py:148-173` (`_project_allowed`, `_check_project`, `_require_project`) |
| Проверка админства | `server.py:176-182` (`_require_admin`) |
| Документация поведения | `server.py:15-24` (docstring «ACL (auth mode)») |
| Выбор middleware по режиму | `taiga-mcp/src/taiga_mcp/__main__.py:66-73` |
| Резолв Bearer через хаб | `taiga-mcp/src/taiga_mcp/auth_middleware.py:160-199` |

Идентичность **используется**, а не просто пропускает запрос. Проверки стоят в каждом
инструменте: `list_projects` фильтрует выдачу (`server.py:193-195`), `list_issues`
(`server.py:266`), `get_issue` (`server.py:287-291`), `create_issue` (`server.py:315`),
`update_issue` (`server.py:348-355`), `delete_issue` (`server.py:377-383`), `add_comment`
(`server.py:895-909`), справочники (`server.py:937,949,961,973,985,997`).

**Подтверждение живого режима** (прод, `project_hub` 45.141.79.157):

```
TAIGA_MCP_AUTH_SOURCE=auth
MCP_TRANSPORT=streamable-http
PROJECT_HUB_URL=http://project-hub:8770
TAIGA_API_URL=http://taiga-back:8000/api/v1
TAIGA_ADMIN_USERNAME=admin
```

Контейнер `infra-taiga-mcp-1`, up 3 weeks (healthy). Значение `auth` → `_run_http_with_hub_auth`
(`__main__.py:68-69`) → `HubAuthMiddleware`. Заданные переменные (только имена):
`TAIGA_API_URL`, `TAIGA_ADMIN_USERNAME`, `TAIGA_ADMIN_PASSWORD`, `MCP_API_KEY`, `MCP_HOST`,
`MCP_PORT`, `MCP_TRANSPORT`, `LOG_LEVEL`, `TAIGA_MCP_AUTH_SOURCE`, `PROJECT_HUB_URL`,
`PROJECT_HUB_KEY`. `MCP_API_KEY` задан, но в режиме `auth` не читается.

### Данные: ACL ограничивает проект, а не операцию

`_project_allowed` (`server.py:148-152`) сравнивает только `project_id` со множеством
`allowed_taiga_project_ids()`. Ни `_check_project`, ни `_require_project` не смотрят, что за
операция. Единственная проверка уровня операции — `_require_admin`, и она стоит ровно на
двух инструментах: `create_project` (`server.py:235`) и `delete_project` (`server.py:247`).

Следствие для нашего сценария: личность с членством в проекте 12 проходит `_require_project(12)`
в `delete_issue` (`server.py:377-383`) точно так же, как в `add_comment`. **Удаление задач
проекта 12 остаётся доступным.**

Дополнительно: `identity.capabilities` в `server.py` не используется нигде, кроме
`is_platform_admin()`/`is_tenant_admin()` — единственное вхождение `server.py:180`. Поэтому
хабовая роль (`editor` vs `viewer`) на инструменты не влияет.

### Данные: скоуп ключа не влияет на ACL taiga-mcp

taiga-mcp читает `memberships[].capabilities` из ответа `/resolve`
(`auth_middleware.py:101-112`). Хаб кладёт туда **сырые capabilities роли**, без пересечения
со скоупом ключа (`api/internal.py:261,269`). Пересечение со скоупом попадает в поле верхнего
уровня `capabilities` (`api/internal.py:300-308,319`), а `parse_resolve_response` это поле
**не читает вовсе** (`auth_middleware.py:79-118`).

Кроме того, `/resolve` принимает любой ключ, привязанный к пользователю, независимо от скоупа —
проверяется только `api_key.user_id is not None` (`api/internal.py:285-287`). Ключ со скоупом
`read` пройдёт через taiga-mcp ровно так же, как `mcp`.

Отдельная находка про сам хаб: `Actor.is_platform_admin()` (`auth/dependencies.py:136-141`)
считает права **только по членствам, без пересечения со скоупом ключа**, а
`require_capability_in_tenant` для platform_admin выходит раньше проверки capabilities
(`auth/dependencies.py:189-190`). Значит, для владельца-platform_admin скоуп ключа не
ограничивает ни один маршрут `/api/admin`. Скоуп реально сужает только ключ **не**-platform-админа.
Ещё: нераспознанный скоуп (`iva-write`, `helm-vectors` — такие в базе есть) даёт пустое
множество в `_expand_scope_to_caps` (`auth/dependencies.py:288`), а пустое трактуется как
«без ограничений» (`api/internal.py:304-308`). Придуманное имя скоупа прав не сужает.

### Данные: выпуск без браузера есть

Символьный поиск call-sites `api_keys.issue` (не грепом) даёт пять точек:

| Файл:строка | Маршрут | Гейт |
|---|---|---|
| `api/admin.py:805` | `POST /api/admin/tenants/{tenant_slug}/members/{user_id}/api-keys` | **Bearer подходит**: `require_capability_in_tenant("manage:users")` |
| `web/routes.py:626` | `POST /oauth/token` | OAuth-код, до него браузерный логин |
| `web/routes.py:873` | `POST /me/api-keys` | сессия + CSRF |
| `web/routes.py:1845` | `POST /tenants/{slug}/members/{user_id}/api-keys` | сессия + CSRF |
| `web/routes.py:2360` | `POST /users/{user_id}/api-keys` | сессия + CSRF, platform_admin |

Маршрут `api/admin.py:748-837` добавлен намеренно для программного выпуска — в его докстроке
(`api/admin.py:760-777`) прямо сказано: «Выпустить PHK-ключ члену тенанта — программно, без
интерфейса… Позвать такое из другого сервиса нельзя, и поэтому helm не мог выдать ключ».

Ограничения маршрута: скоуп только из `_TENANT_PHK_SCOPES = ("mcp", "read")`
(`api/admin.py:745,784-788`); цель обязана быть членом тенанта (`api/admin.py:790-800`);
значение ключа отдаётся один раз (`api/schemas.py:287-297`).

Прочие пути выпуска — **отсутствуют**: в CLI (`cli/__main__.py:20-85`, семь подкоманд:
`import`, `reset-wiki`, `resync-memberships`, `reconcile-wiki`, `reconcile-llm`,
`alert-budgets`, `seed-spec`) выпуска ключей нет; в `scripts/` только `backup.sh` и
`deploy.sh`; ENV-бутстрап (`app.py:207-251`, `project_hub_bootstrap_admin_email/password`)
создаёт platform-админа с паролем, но ключей не выпускает; Makefile в репозитории нет.

### Данные: сервисный аккаунт — это соглашение, а не флаг

В таблице `users` полей `is_service`/`is_bot` нет. Состав: `id`, `email`, `full_name`,
`status`, `password_hash`, `created_at`, `wants_wiki_digest`. Аккаунт без пароля создать можно
(`password_hash` nullable), **но для Taiga это тупик**: адаптер отказывается —
«create_user requires a password (Taiga has no password-less users)»
(`adapters/taiga.py:129-131`).

**Прецедент уже есть.** `arch-sync@tacticum.ru` / «Dev Arch Sync (service)», active, с паролем,
заведён в Taiga (id 39, username `arch-sync-tacticum-ru-81fd0bcd`), Wiki.js (id 37), arch,
litellm. Его ключ: `phk_5927`, скоуп `mcp`, имя `dev-arch-sync`, создан 2026-06-24, не отозван.

Важная деталь про этот прецедент: его членство **тенант-уровня** (`tacticum`, роль `editor`),
без привязки к проекту. `taiga_project_id` в `/resolve` берётся только из проектного членства
(`api/internal.py:266`: `m.project.taiga_project_id if m.project else None`), поэтому у
`arch-sync` множество разрешённых Taiga-проектов **пустое** — через taiga-mcp он получил бы
отказ на всё. Для нашей задачи членство обязано быть **проектным**.

### Данные: роли и минимальная комбинация

Роли хаба (таблица `roles`, прод):

| Роль | Тенант | Capabilities |
|---|---|---|
| `platform_admin` | — | `*` |
| `tenant_admin` | — | read:tenants, read:projects, read:users, read:roles, read:audit, manage:projects, manage:users, manage:roles, manage:credentials |
| `editor` | — | read:tenants, read:projects, read:users, edit:wiki, edit:taiga |
| `viewer` | — | read:tenants, read:projects, read:users |
| `dev` | — | `{}` (пусто) |
| `architect` | tacticum | read:tenants, read:projects, read:users, arch:author, arch:review |
| `CEO`/`CIO`/`COO`/`CPO` | iva | read:tenants, read:projects, read:users |
| `iva-atlassian-writer` | iva | iva:atlassian:write |
| `llm-manager` | kaz | manage:llm |

Совпадает с сидом `app.py:51-65`.

**Минимальная комбинация для «читать задачи проекта 12 + комментировать» через taiga-mcp:**
проектное членство на хабовый проект `tacticum-dev` с ролью **`viewer`**. Роль `viewer` не
даёт `manage:*`, значит `is_tenant_admin()` = false и `create_project`/`delete_project`
закрыты. Всё остальное — как описано в вердикте: удаление задач проекта 12 остаётся.
Роль `dev` не подходит: она в `NON_PROVISIONING_ROLES` (`adapters/role_map/taiga.py:26`),
в Taiga пользователя не заведут вовсе.

### Данные: живое состояние

Прод `project_hub` (45.141.79.157), БД `project_hub` на контейнере `infra-taiga-db-1`.

Ключей всего **242**. По скоупам:

| Скоуп | Всего | Отозвано | Сервисных (`user_id IS NULL`) |
|---|---|---|---|
| `mcp` | 228 | 26 | 0 |
| `resolve` | 7 | 0 | 7 |
| `iva-write` | 2 | 1 | 0 |
| `read` | 1 | 0 | 0 |
| `wiki_digest_recipients` | 1 | 0 | 1 |
| `all` | 1 | 1 | 0 |
| `helm-vectors` | 1 | 0 | 0 |
| `write:users` | 1 | 0 | 0 |

Сервисные ключи (`user_id IS NULL`, доступ только к `/api/internal/*`): `phk_036b`
codex-doctranslate, `phk_f85a` helm-resolve, `phk_1429` mcp-runtime, `phk_b1cc` knowledge-rag,
`phk_09ed` codex, `phk_3f73` memory-service, `phk_575a` basic (все `resolve`), `phk_9b33`
wiki-digest service.

Проект 12 = хабовый `tacticum-dev` «Tacticum AI Dev Platform»,
uuid `b3ecd718-fb05-4cd9-a1bc-6c0ecfe14dda`, тенант **`tacticum`**. В Taiga —
`tacticum-tacticum_dev`, приватный, owner_id 5.

Platform-админы хаба: `d@cifragen.ru`, `d.lebedev@tacticum.ru`. Тенант-админ `tacticum`:
`d@cifragen.ru`.

Проектные членства на `tacticum-dev` (все `editor`): a.berezin, a.shulga, d.evstigneev,
d.parshakov, d.solonko, g.felyust, i.monakhov (все @tacticum.ru), i.popovkin@cifragen.ru.

Публичные адреса: хаб `https://master.cifragen.ru`, Taiga `project.cifragen.ru`
(→ REST `https://project.cifragen.ru/api/v1`).

### Данные: роли Taiga в проекте 12 — ни одна не годится как «только чтение + комментарии»

Таблица `users_role`, `project_id=12`:

| Роль | Права |
|---|---|
| UX, Design, Front, Back, Product Owner | идентичный полный набор: add/modify/**delete** для issue, milestone, task, us, wiki_page, epic + все `comment_*` |
| **Stakeholder** | add_issue, modify_issue, **delete_issue**, view_issues, view_milestones, view_project, view_tasks, view_us, modify_wiki_page, view_wiki_pages, add_wiki_link, delete_wiki_link, view_wiki_links, view_epics, comment_epic, comment_us, comment_task, comment_issue, comment_wiki_page |

То есть `Stakeholder` — ближайшая, но и она даёт `delete_issue` и `modify_issue`.
Нужного набора (view_* + comment_*) в готовых ролях **нет**.

Маппинг хаб → Taiga (`adapters/role_map/taiga.py:17-35`): `tenant_admin` → Product Owner,
`editor` → Back, `viewer` → Stakeholder, неизвестная роль → Stakeholder,
`dev` → None (не заводить).

Члены Taiga-проекта 12: admin (`mr.diaret@ya.ru`, **is_superuser**, Stakeholder),
`d@cifragen.ru` (**is_superuser**, Product Owner), плюс 10 обычных.
Заметьте: у суперпользователей роль в проекте роли не играет — они всё равно всё могут.

---

## Поправки к постановке

1. **«`api_keys.issue` импортируется только в `web/routes.py`»** — неверно. Пять call-sites,
   один из них Bearer-доступный (`api/admin.py:805`). Именно он решает задачу.
2. **«Любой валидный Bearer исполняет запросы правами админа Taiga, включая `delete_issue`/
   `delete_project`»** — неверно. `delete_project` закрыт `_require_admin` (`server.py:247`),
   чужие проекты закрыты ACL. Верна ослабленная версия: **внутри разрешённого проекта**
   доступны все операции, включая удаление.
3. **«Личный админский ключ `mr.diaret@ya.ru`»** — такого пользователя в хабе нет
   (есть `d@cifragen.ru`, `d.lebedev@tacticum.ru`, `d.lebedev@iva.ru`).
   `mr.diaret@ya.ru` — это учётка **Taiga** (`users_user.id=5`, username `admin`,
   is_superuser). PHK-ключ на неё оформлен быть не может. Чей именно ключ используется
   сейчас — по значению не определял (см. «НЕ проверено»).

---

## РЕЦЕПТ: как выпустить сервисный ключ самим

Всё делается по HTTP. Человек нужен **только там, где отмечено**.

### Вариант А — через MCP (быстро, но защиты «нельзя удалять» не даёт)

Исполнитель: ключ, владелец которого **platform_admin** (шаг 1 требует platform_admin;
шаги 2-3 хватило бы `manage:users` в тенанте `tacticum`).

**Шаг 1. Завести сервисного пользователя** — *требует platform_admin*
(`api/admin.py:280-304`, `dependencies=[require_platform_admin()]`):

```
POST https://master.cifragen.ru/api/admin/users
Authorization: Bearer <ключ platform-админа>
Content-Type: application/json

{"email": "taiga-bot@tacticum.ru",
 "full_name": "Taiga CI Bot (service)",
 "password": "<пароль, задать явно>"}
```

Пароль задавайте **явно**: он же станет паролем учётки в Taiga
(`services/users.py:78-84` → `adapters/taiga.py:129-155`), и он понадобится в варианте Б.
Без пароля Taiga-адаптер откажет. Учтите побочный эффект: на адрес уйдёт welcome-письмо
(`services/users.py:94-99`) — заводите почтовый ящик, который существует, либо примите, что
письмо отвалится.

**Шаг 2. Проектное членство** (`api/admin.py:358-410`, нужен `manage:users` в тенанте):

```
POST https://master.cifragen.ru/api/admin/tenants/tacticum/memberships
Authorization: Bearer <тот же ключ>

{"email": "taiga-bot@tacticum.ru",
 "project_id": "b3ecd718-fb05-4cd9-a1bc-6c0ecfe14dda",
 "role_name": "viewer"}
```

Именно **проектное** (`project_id` заполнен), иначе `taiga_project_id` не попадёт в ACL и
ключ не увидит вообще ничего — ровно как у `arch-sync`. `viewer` → в Taiga роль Stakeholder.

**Шаг 3. Выпустить ключ** (`api/admin.py:748-837`):

```
POST https://master.cifragen.ru/api/admin/tenants/tacticum/members/<user_id>/api-keys
Authorization: Bearer <тот же ключ>

{"scope": "read", "name": "github-actions taiga comments", "expires_in_days": 365}
```

`user_id` — uuid из ответа шага 1. Скоуп: `mcp` или `read`, других маршрут не примет.
На поведение taiga-mcp скоуп не влияет (доказано выше), поэтому берите `read` — он сузит
доступ к самому хабу, если владелец не platform_admin (а он не platform_admin).
Значение ключа приходит в поле `key` **один раз** — сразу класть в секрет
GitHub Actions `TAIGA_TOKEN`.

**Отзыв** — независимый: `revoked_at` у этого ключа, остальные не трогаются.

### Вариант Б — прямой REST Taiga (права настоящие)

Шаги 1-2 те же (учётка в Taiga создаётся автоматически на шаге 1, членство в проекте 12 —
на шаге 2). PHK-ключ не нужен вовсе. Дальше скрипт ходит в Taiga напрямую:

```
POST https://project.cifragen.ru/api/v1/auth
{"type": "normal", "username": "<username из user_system_links>", "password": "<пароль шага 1>"}
→ {"auth_token": "...", "id": ...}

GET  https://project.cifragen.ru/api/v1/issues?project=12&status=<id>
     Authorization: Bearer <auth_token>

PATCH https://project.cifragen.ru/api/v1/issues/<id>
     Authorization: Bearer <auth_token>
     {"version": <текущий version>, "comment": "текст"}
```

Всё это уже реализовано и работает — можно списывать один в один с
`taiga-mcp/src/taiga_mcp/client.py`: логин `client.py:69-85`, заголовки `client.py:87-95`,
листинг с фильтрами `client.py:209-211`, комментарий `client.py:453-468`
(комментарий — это действительно PATCH сущности с полями `version` + `comment`),
справочник статусов `client.py:474-475`.

**Объём переписывания: малый** — 4 вызова, ~80-120 строк на Python+httpx, без новых
зависимостей. Логика «прочитать задачи по статусу → добавить комментарий» переносится
дословно.

**Чем платим:** `username` в Taiga генерируется из почты с хвостом-хешем
(`adapters/taiga.py:42`, вида `taiga-bot-tacticum-ru-XXXXXXXX`) — его надо прочитать из
`user_system_links.external_username` после шага 1. И в секрет уедет **пароль** учётки Taiga,
а не отзываемый токен: отзыв = смена пароля/деактивация пользователя, а не `revoked_at`.

**Что даёт:** права проверяет сам Taiga по роли в проекте. `delete_project` недоступен
физически. Но с ролью Stakeholder `delete_issue` **всё ещё доступен** (см. таблицу прав выше).

**Чтобы получить настоящее «только чтение + комментарии» — нужен человек.** Создать в
проекте 12 кастомную роль (в Taiga: настройки проекта → Permissions → добавить роль) с
набором ровно `view_project, view_issues, view_us, view_tasks, view_milestones, view_epics,
comment_issue, comment_us, comment_task, comment_epic` и переставить на неё членство бота.
Через хаб этого не сделать: маппинг ролей жёстко зашит в
`adapters/role_map/taiga.py:17-21`, а неизвестные роли падают в Stakeholder
(`role_map/taiga.py:35`). Править существующие роли проекта нельзя — они общие с людьми.

### Что требует человека — сводка

| Действие | Кто |
|---|---|
| Шаги 1-3 варианта А | мы, по HTTP |
| Шаг 1-2 варианта Б | мы, по HTTP |
| Запись в секрет GitHub Actions `TAIGA_TOKEN` | человек |
| Кастомная роль в Taiga (для настоящей узости) | человек, в UI Taiga |
| Мерж/деплой скрипта | человек |

---

## Что нужно изменить, чтобы права стали настоящими

**Путь 1 — правка `taiga-mcp`: фильтр инструментов по личности.** Не «клиент на пользователя»
(это дороже и ломает совместимость с `arg`-режимом), а простой список: в `auth` режиме
сверять имя инструмента с набором, разрешённым для роли из членства. Точка врезки одна —
рядом с существующими `_check_project`/`_require_project` (`server.py:148-182`), плюс по
одной строке в `delete_issue` (`server.py:373`), `delete_user_story` (`server.py:498`),
`delete_task` (`server.py:642`). **Объём малый** (~40-60 строк + тесты; в
`taiga-mcp/tests/test_acl.py` уже 426 строк каркаса). **Риск средний**: меняется поведение
живого прод-сервера, которым пользуются все агенты; ошибка в списке ломает рабочие сценарии
не только бота.

**Путь 2 — вариант Б выше (обход MCP).** **Объём малый** (~80-120 строк нового скрипта).
**Риск низкий**: ничего существующего не трогаем, taiga-mcp остаётся как есть, отказ скрипта
никого больше не задевает. Права проверяет Taiga.

Замечу как факт, не как рекомендацию: пути не исключают друг друга и решают разное — путь 2
закрывает нашу задачу, путь 1 закрывает дыру для всех остальных ключей, которых 228.

---

## НЕ проверено

- **Не проверял, что Taiga действительно отказывает по `permissions` роли** — прочитал состав
  прав из `users_role`, но запроса на удаление под непривилегированным токеном не делал
  (это было бы изменение на проде). Модель прав Taiga штатная, но фактом это назвать не могу.
- **Не определял, какой именно PHK-ключ используется сейчас** для проекта 12 — для этого
  нужно значение токена из `~/.claude.json`, а по хешу владельца не найти. Поэтому
  утверждение «ключ принадлежит `mr.diaret@ya.ru`» опроверг только частично: такого
  пользователя в хабе нет, но чей ключ работает — не установил.
- **Не проверял, влияет ли hub-auth плагин Taiga (US #443)** на обычный логин
  `POST /api/v1/auth`. Косвенно работает: taiga-mcp логинится именно так и держится healthy
  3 недели.
- **Не проверял поведение почты** при заведении сервисного аккаунта на несуществующий адрес.
- **Не проверял** ветку `arg`-режима на живом сервере — он в `auth`, `MCP_API_KEY` не читается.

## Инцидент при разведке

При снятии имён переменных окружения хаба использовал `grep -iE 'db|database|postgres'` — под
шаблон попали переменные с секретами (пароли БД, session secret, Wiki.js API key), и их
значения оказались в выводе команды. В отчёт и никуда дальше они не попали, но **значения
светились в транскрипте сессии**. Ничего не менял, ротацию не делал — это решение не моё.
Отмечаю, чтобы вы знали и решили сами.

---

## ВЕРДИКТ-2: что мы можем ключом `phk_535c` (проверено 2026-08-06)

**Ничего из трёх шагов. Все упрутся в 403.** Без человека или без чужого ключа мы не можем
ни создать пользователя, ни добавить членство, ни выпустить ключ.

### По шагам

| Шаг | Маршрут | Гейт | `phk_535c` |
|---|---|---|---|
| 1 | `POST /api/admin/users` | `require_platform_admin()` (`api/admin.py:284`) | **403** |
| 2 | `POST /api/admin/tenants/tacticum/memberships` | `require_capability_in_tenant("manage:users")` (`api/admin.py:367`) | **403** |
| 3 | `POST /api/admin/tenants/tacticum/members/{user_id}/api-keys` | `require_capability_in_tenant("manage:users")` (`api/admin.py:757`) | **403** |

### Почему — механика, а не догадка

Ваше наблюдение верно: `a.shulga@tacticum.ru` не platform_admin. Данные с прода — у него
**десять** членств, и в тенанте `tacticum` все до одного **проектного** уровня:

| Уровень | Тенант | Проекты | Роль |
|---|---|---|---|
| PROJECT-LEVEL | tacticum | codex, helm, tacticum-dev, platform, agents | editor |
| PROJECT-LEVEL | cifragen | zus-codex | editor |
| PROJECT-LEVEL | iva | control-tower | editor + CEO |
| TENANT-LEVEL | **iva** | — | COO, CPO |
| PLATFORM | — | — | **нет ни одного** |

Тенант-уровневые членства у него есть только в `iva`, и роли там читающие
(`read:tenants, read:projects, read:users`).

Решает это `Actor.capabilities_in` (`auth/dependencies.py:107-134`). При тенант-проверке
`require_capability_in_tenant` зовёт его **без `project_id`** (`dependencies.py:198`:
`caps = actor.capabilities_in(tenant_id=tenant.id)`). Внутри засчитываются только две ветки:
платформенные членства (`dependencies.py:121-123`) и тенант-уровневые
(`dependencies.py:124-126`). Третья ветка, проектная, требует `project_id is not None`
(`dependencies.py:127-129`) — а он `None`. **Проектные членства при тенант-проверке не
засчитываются вовсе.**

Скоуп ключа тут ни при чём: `mcp` → `{"*"}` → пересечение не применяется
(`dependencies.py:132-133`). Но сужать и нечего — множество уже пустое.

### Чем подтверждено

Прогнал **настоящий класс `Actor` из репозитория** на точной форме прод-членств (без сети,
без секретов, ничего не создавая):

```
scope 'mcp' разворачивается в: {'*'}
is_platform_admin(): False

capabilities_in(tenant=tacticum) = ПУСТО
capabilities_in(tenant=iva)      = ['read:projects', 'read:tenants', 'read:users']

  tacticum: manage:users   есть? False
  tacticum: read:users     есть? False
  tacticum: read:projects  есть? False

capabilities_in(tenant=tacticum, project=tacticum-dev) =
  ['edit:taiga', 'edit:wiki', 'read:projects', 'read:tenants', 'read:users']
```

Последняя строка показывает суть: права у него в `tacticum` есть, но **только когда проверка
идёт с указанием проекта**. Все три наших маршрута проверяют на уровне тенанта, поэтому видят
пустоту.

**Эмпирическую проверку сделать не смог:** GET-запросы к `/api/admin/*` этим ключом
заблокировала система разрешений (нужно было прочитать значение ключа из `~/.claude.json`).
Обходить не стал. Доказательство выше — по коду и данным, и оно однозначно; но помечаю честно,
что живым запросом это не подтверждено.

### Кто может

Только двое, и оба — люди:

| Кто | Основание |
|---|---|
| `d@cifragen.ru` | platform_admin (платформенный) + tenant_admin в `tacticum` (тенант-уровень) |
| `d.lebedev@tacticum.ru` | platform_admin (платформенный) |

Ключи у них есть (например, `phk_265d` «bootstrap MCP key», активен), но значений мы не имеем.

---

## Шаг 1 не нужен вовсе — маршрут членства сам создаёт пользователя

Это упрощает мой прежний рецепт и снижает требование к человеку.

`POST /api/admin/tenants/{slug}/memberships` умеет создавать пользователя на лету: если передать
`email` и пользователя с таким адресом нет, он создаётся тут же через `user_service.create`
(`api/admin.py:376-388`), а `MembershipAddIn` принимает `email`, `full_name`, `password`,
`project_id`, `role_name` (`api/schemas.py:127-133`).

**Значит вся задача — два вызова, и `platform_admin` не требуется, достаточно `manage:users`
в тенанте `tacticum`:**

```
POST https://master.cifragen.ru/api/admin/tenants/tacticum/memberships
{"email": "taiga-bot@tacticum.ru",
 "full_name": "Taiga CI Bot (service)",
 "password": "<задать явно>",
 "project_id": "b3ecd718-fb05-4cd9-a1bc-6c0ecfe14dda",
 "role_name": "viewer"}

POST https://master.cifragen.ru/api/admin/tenants/tacticum/members/<user_id>/api-keys
{"scope": "read", "name": "github-actions taiga comments", "expires_in_days": 365}
```

Оговорка: роль `viewer` не входит в `PRIVILEGED_ROLES`, поэтому guard на эскалацию
(`services/memberships.py:66-70`) не сработает и `actor_is_platform_admin` не понадобится.

---

## Про `arch-sync`: задачу целиком не решает

**Короткий ответ: не решает и выигрыша не даёт.** Шаг 1 и так не нужен (см. выше), а шаги 2-3
одинаково требуют `manage:users` — то есть человека. Экономии не возникает.

### Переиспользовать её ключ нельзя

`phk_5927` придётся выпускать заново: значение невосстановимо (в базе argon2id-хеш) и нигде
не развёрнуто. Проверил — не найден в окружении ни одного контейнера на `project_hub`, ни в
`/opt/project/infra/.env`, ни на сервере `helm` (`grep -rl` по `/opt /root /etc` — 0 файлов),
ни в конфигурациях на машине (в найденных 11 ключах его нет).

Значит шаг 3 остаётся обязательным при любом раскладе.

### Требование к человеку не понижается

Реюз `arch-sync` убрал бы только требование platform_admin — но оно и так снято маршрутом
членства. Остаётся `manage:users` в `tacticum`, а он есть лишь у `d@cifragen.ru`
(он же platform_admin) и `d.lebedev@tacticum.ru`. **Тот же человек в обоих сценариях.**

### Побочные эффекты, если человек всё же пойдёт этим путём

1. **Молчаливое понижение роли в Taiga.** Сейчас `arch-sync` (Taiga user 39) имеет в проекте 12
   роль **Back** — с полными правами, включая удаления. Добавление хабового членства `viewer`
   вызовет `grant_role`, а тот на проде идёт через прямую запись в БД Taiga
   (`app.py:77-82` передаёт `database_url=settings.taiga_db_url`; он не `None`, потому что
   `TAIGA_DB_PASSWORD` задан — `config.py:186-192`). Запрос —
   `on conflict (user_id, project_id) do update set role_id = excluded.role_id`
   (`adapters/taiga.py:298-308`), то есть роль **перезапишется** Back → Stakeholder.
   Для нашей цели это сужение и скорее плюс, но это изменение существующей учётки, и происходит
   оно без предупреждения.
2. **Остальные 11 проектов никуда не денутся.** `arch-sync` состоит в **12** Taiga-проектах с
   ролью Back (3, 7, 11, 12, 13, 18, 19, 20, 21, 22, 29, 32) — следствие симметричного бэкфилла
   от её тенант-уровневого членства (`services/memberships.py:114-126`). Хабовое проектное
   членство сузит только то, что пропускает taiga-mcp (станет ровно `{12}`); прямой вход в Taiga
   под её паролем по-прежнему даст Back на одиннадцати других проектах.
3. **Смешение назначений.** Учётка заведена под arch-канал. Второй ключ на ней означает, что
   отзыв по инциденту GitHub Actions заденет и arch-сценарий.

### Чего я про `arch-sync` НЕ знаю — и не буду делать вид

**Работает она сейчас или простаивает — не установил.** Соблазнительный вывод «ключ ни разу не
использовался, значит учётка мёртвая» **неверен**, и вот почему: `last_used_at` проставляется в
`verify_token` (`auth/api_keys.py:74`), но `get_db` намеренно не коммитит
(«Caller is responsible for commit» — `db/session.py:50-52`), а `/resolve` коммита не делает.
Поэтому ключ, который ходит только через MCP, навсегда остаётся с пустым `last_used_at`.

Подтверждение цифрой: из **228** ключей скоупа `mcp` непустой `last_used_at` лишь у **5**.
Столбец не годится как признак использования.

`last_login` у Taiga-учётки 39 тоже пуст, но и это не доказательство: taiga-mcp ходит в Taiga
под админским клиентом, а не под ней, так что её собственный вход и не должен фиксироваться.

Что я знаю точно: её ключ не развёрнут ни на одном из двух проверенных серверов и не лежит в
конфигурациях на машине. Где-то ещё он быть может.

### Вывод

Если человека всё равно звать — **дешевле и чище завести отдельную учётку**, а не приспосабливать
`arch-sync`: у человека одинаковое число действий (два вызова против двух), зато нет смешения
назначений и нет молчаливого изменения чужой учётки.
