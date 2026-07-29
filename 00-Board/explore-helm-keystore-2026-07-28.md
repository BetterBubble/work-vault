---
title: explore-helm-keystore-2026-07-28
type: note
permalink: tacticum/00-board/explore-helm-keystore-2026-07-28
status: draft
role: explorer
tags:
- explorer
- iva-write
- helm
- keys
created: 2026-07-28 11:38
updated: 2026-07-28 11:38
repo: tacticum-dev
project: iva-write
---

Read-only разведка helm под концепт «хранилище персональных ключей доступа по e-mail, отдельно по системам». Канон не пишу.

**Репо:** `/Users/bubblemac/tacticum/helm`, ветка `main`, HEAD `eaafbf6` «Add PO internal criticality cell (US #205)», дерево чистое.
**Связь:** [[explore-iva-write-multikey]] · [[explore-iva-write-base]] · [[explore-0058-gap-2026-07-27]]

> ЗНАЧЕНИЯ секретов не выводились. Только имена переменных/полей/файлов. На серверы не ходил.

## ГЛАВНЫЙ ВЫВОД (сразу)

- **Идентичность по e-mail в helm есть и она каноническая** — `person` + `person_email` (email = PK), плюс живой `Principal.email` из project-hub. Ключ `email → человек` строить не надо, он уже есть.
- **Инфраструктуры хранения ЧУЖИХ секретов в helm нет вообще.** Ни одной секрет-колонки в БД, ни одного шифрования на покое, ни одного секрет-менеджера. Все 11 секретов helm живут в env/`.env` и все они — СВОИ, сервисные.
- **Каркас «чей ключ и что им можно» есть** — `ownership_assignment` (ADR-0008) + `user_role`, оба с готовым RBAC и готовым паттерном «чтение всем, правка под `require_full_access`/`require_cpo`» + append-only аудит.
- **helm сегодня НИЧЕГО не пишет в Jira/Confluence и не имеет к ним даже API-клиента на запись.** Jira-адаптер — заглушка `NotImplementedError`; живые REST-клиенты только read (`extract`), и оба ходят под ОДНОЙ сервисной учёткой из env. Выпускать токены за других людей этими кредами по коду нечем — эндпоинтов управления токенами не вызывается нигде.

---

## 1. Идентичность: есть, канон — e-mail

### Модели БД
| Сущность | Файл:строка | Ключ |
|---|---|---|
| `Person` | `src/helm/infrastructure/db/models.py:630-651` | сурр. `person_id`; поля `person_name/team/role/grade/manager_email/repos/jira_projects` |
| `PersonEmail` | `models.py:654-669` | **`email` — PK** (`:665`), FK `person_id`, `is_primary`; 1:N задел |
| `UserRole` | `models.py:1328-1345` | `(email, role)` uniq; роли CEO/COO/CPO/CIO/CCO/HR/viewer/full_access/sales |
| `OwnershipAssignment` | `models.py:1348-1409` | см. §3 |

Комментарий в шапке `models.py:7-11` объявляет `person_email.email` **каноническим ключом идентичности домена** — на него FK-ссылаются `block.eol_email`, `initiative.owner_email`, `assignment.person_email`, `product_area.po_email` (`models.py:202-203`, `706`, `748`, `795`, `882`).

**Чего нет:** ни одного поля с Jira-логином / accountId / username. Единственный «мостик» в Jira — `assignee_name` строкой (`models.py:1795`), матч по display-name. Для «писать ЗА человека» это не идентификатор.

### Как helm узнаёт, кто пришёл
**REST** — `src/helm/interface/api/auth.py:54-122` `require_user`:
- `auth_required=False` → открытый dev-принципал `dev@local`, роль `platform_admin` (`:60-69`);
- иначе Bearer → project-hub `/resolve` (`resolve_url` + `resolve_service_key`, `:71-97`) → `ResolvedUser{email, memberships}`;
- роли из БД `user_role` дочитываются **live** каждым запросом (`:43-51`, `:102`);
- `principal_for_tenant` (`domain/access.py:140+`) → `Principal{email, tenant, roles, data_classes, contour}`; нет членства в `iva` → 403 (`:110-114`).
- `optional_user` (`:125-135`) — тот же поток, аноним → `None`.
- Заголовков вида `X-Auth-User-Id` от gateway helm **не читает** — grep по репо пустой. Идентичность только из `/resolve` или dev-MCP `whoami`.

**MCP аналитика** — `src/helm/interface/mcp/analyst_server.py:297-364` `_require_principal`, сознательная копия REST-потока (докстринг `:300-309`):
- заголовки берёт из `ctx.request_context.request` (`:275-283`);
- Bearer → project-hub `/resolve` (`:333-336`);
- **второй источник identity — `phk_*` токен dev-MCP** (`:338-352`): при заданном `dev_mcp_url` зовёт `resolve_dev_token` → тул `whoami` на `mcp.tacticum.dev`, берёт `caller.user_email`, принимает **только** `auth_method == "phk"` (`infrastructure/auth/dev_token.py:71-81`; `tac`-сервисный отвергается как обезличенный). Дополнительный гейт по домену почты `dev_token_email_domain_set` (`analyst_server.py:345-347`, дефолт `iva.ru,tacticum.ru` — `config.py:552`). Кэш `sha256(token) → email`, TTL 300с (`dev_token.py:31-39`), сам токен не логируется (`dev_token.py:12-16`).
- Синтезируется минимальное членство `viewer` (`:351-352`) — dev-ветка доступ НЕ расширяет.

**`my_todo` — по ADR-0058 Реш. 8 проверено, реализовано именно так** (`analyst_server.py:1558-1600`):
```python
principal = await _require_principal(ctx)
actor_email = (email or principal.email or "").strip()
```
Сигнатура `my_todo(ctx, email: str | None = None)` — параметр `email` **безусловно переопределяет** актора из Bearer (`:1591`). Никакой проверки прав на «посмотреть чужой список» нет; докстринг это признаёт открыто («параметр `email` его переопределяет (посмотреть чужой список)»). Для read-витрины это осознанно, но **как прототип «под кем действуем» — это дыра**: если тем же приёмом выбирать ключ для записи, любой аутентифицированный сможет писать под чужим ключом. См. риск R1.

**Монтирование:** `main.py:173-174` — MCP-приложения смонтированы как ASGI-mount, **вне** `_AUTH` (`main.py:105`), auth у каждого тула свой (комментарий `main.py:168-170`).

## 2. Секреты сегодня: только env, только свои, без шифрования

`config.py:15` — `SettingsConfigDict(env_prefix="HELM_", env_file=".env", extra="ignore")`. Все секреты — поля `Settings`, читаются из окружения/`.env`:

| Поле | Строка | Назначение |
|---|---|---|
| `gateway_api_key` | `config.py:27` | LLM Gateway |
| `projecthub_token` | `:49` | project-hub, коннекторы субстрата |
| `resolve_service_key` | `:66` | service-key для `/resolve` |
| `iva_llm_api_key` | `:92` | внутренний LLM |
| `meili_api_key` | `:119` | Meilisearch |
| `rag2_live_mcp_token` | `:259` | live-MCP RAG#2 |
| `rag2_synth_llm_api_key` / `..._fallback_...` | `:385`, `:402` | синтез |
| `allure_token` | `:484` | Allure (`Api-Token`) |
| `iva_bot_support_token` | `:504` | исходящие вызовы бота |
| `iva_bot_support_webhook_secret` | `:508` | верификация webhook |

Плюс мимо `Settings`, прямо из `os.environ` в ингесте (`ingest/rag2_extract.py:1223-1226`, `1252-1254`; `ingest/rag2_harness.py:731-735`): `HELM_RAG2_JIRA_USER`, `HELM_RAG2_JIRA_PASSWORD`, `HELM_RAG2_CONFLUENCE_TOKEN`, `HELM_RAG2_JIRA_BASE_URL`, `HELM_RAG2_CONFLUENCE_BASE_URL`.

**Ответ однозначный:**
- секретов, хранящихся В БАЗЕ, — **ноль**. Grep по `models.py` на `token|secret|password|credential|api_key` не даёт ни одной колонки;
- шифрования на покое — **нет**. Grep `fernet|cryptography|encrypt|decrypt|kms|vault|keyring|nacl|secretbox` по `src/`, `scripts/`, `alembic/`, `docs/`, `pyproject.toml` даёт **единственное** попадание — `scripts/seed_devops_track.py:45`, и то это ТЕКСТ задачи бэклога («Внедрить sealed-secrets/Vault; снять флаг риска "живые glpat в git"»), а не код;
- зависимости `cryptography` в `pyproject.toml` нет (транзитивно через httpx/TLS — не проверял);
- правила репо про секреты: `.gitignore:34-37` (`.env`, `.env.*`, кроме `.env.example`), `README.md:134-139` («секреты кладёт оператор в `.env`»).

Косвенно рядом: `rag2_redact_secrets` (`config.py:331`) — редакция секретов в ВЫДАЧЕ RAG, к хранению отношения не имеет.

**Вывод: инфраструктуры для хранения чужих секретов в helm нет вовсе. Её нужно строить с нуля — модель, миграция, крипто-слой, ротация, аудит.**

## 3. Модель владения и RBAC — годный каркас

**ADR-0008** `docs/adr/0008-helm-ownership-model.md` (принят 2026-07-26, эпик Taiga #188): 4 измерения × 5 ролей.

| Измерение (`scope_kind`) | Роль |
|---|---|
| `client` | sales, presale |
| `product` | po |
| `area` | tpo |
| `container` (C4 `arch_node`) | competence_lead |
| портфель | CPO — арбитр, full-доступ |

Реализация:
- таблица `ownership_assignment` — `models.py:1348-1409`: `(scope_kind, scope_ref, role, person_id, is_primary, active, tenant, granted_at, granted_by_person_id)`; три индекса, в т.ч. partial-unique `WHERE active` (`:1383-1393`) → идемпотентный add, soft-remove не мешает пере-назначению;
- домен (чистый RBAC) — `src/helm/domain/ownership.py`: `has_role` (`:113`), `primary_owner` (`:134`), `scopes_owned_by` (`:159`), `effective_owners` (`:180`), валидаторы `validate_scope_kind/role/scope_role` (`:61/71/79`), составной ref «область×продукт» (`:46-52`);
- репозиторий — `infrastructure/db/ownership_repo.py`: `list_assignments` (`:25`), `assignment_rows` (`:53`), **`person_id_for_email`** (`:77` — готовый мост e-mail→Person), `add/remove/set_primary` (`:158/232/246`), всё tenant-скопнуто;
- API — `interface/api/routers/ownership.py`: чтение под `require_user` (`:92`), правки под `require_cpo` = `is_full_access` (`:29-40`, `:162/186/198`);
- миграции — `alembic/versions/own210_ownership_assignment.py`, `own220_backfill_area_tpo_ownership.py`, `own230/240/250`; TPO-реестр — `ingest/tpo_registry.py` (коммит `8718aa7` «Add TPO registry ingest (Phase A+B)»), матрица область×продукт — `dd9ebf5`.

**Второй слой прав** — `domain/access.py`: классы данных `green/amber/red_identity/red_money` (`:22-41`), `HRD_VIEW_ROLES` (`:43`), `is_full_access` (`:121`), роль-ограничитель `sales` со скоуп-гейтом путей (`:96`, `:116`).

Итог: связка **человек ↔ область ↔ права** уже есть и работает. Для keystore она даёт готовый ответ на «чей ключ и что им можно» — но НЕ даёт ответ на «в какой системе» (`scope_kind` про продукты/области/контейнеры, не про Jira/Confluence/Allure как системы). Ось «система» — новая.

## 4. Интеграции: кто под кем ходит сейчас

| Система | Клиент | Аутентификация | Учётка | Направление |
|---|---|---|---|---|
| **Jira (API)** | `ingest/rag2_extract.py:867` `httpx.BasicAuth(user, password)`, поиск `/rest/api/2/search` (`:886`) | HTTP Basic | ОДНА сервисная из env `HELM_RAG2_JIRA_USER`/`_PASSWORD` (`:1223-1226`) | только чтение |
| **Jira (домен-адаптер)** | `ingest/jira_adapter.py` | — | — | **заглушка**: `JiraAdapter.fetch_issues/fetch_epics` → `NotImplementedError` (`:57-62`); реально данные идут CSV-выгрузкой (`RealJiraAdapter`, `data/jira_issues.csv`) |
| **Confluence** | `rag2_extract.py:950-968`, `:1029-1044` | `Authorization: Bearer <PAT>` | ОДИН токен из env `HELM_RAG2_CONFLUENCE_TOKEN` (`:1252-1254`) | только чтение (`/rest/api/content`, `/rest/api/content/search`, вложения) |
| **Allure TestOps** | `infrastructure/allure/client.py:29-40` | `Authorization: Api-Token <token>` (`:35`) | ОДИН токен `HELM_ALLURE_TOKEN` (`config.py:484`), база `HELM_ALLURE_BASE_URL` через SSH-туннель `helm → adp-jump → allure.iva.ru:8791` (`config.py:479-486`), `verify_tls=False` (`:486`) | **явно read-only**: докстринг `client.py:10-11` «Только чтение — ничего в TestOps не пишем (внешний контур заказчика)»; в классе только `_get`/`_paged` |

Базовые URL для цитат: `rag2_jira_base_url = https://jira.iva.ru/browse` (`config.py:244`), `rag2_confluence_base_url = https://confluence.iva.ru` (`config.py:326`).

**Пишет ли helm куда-то наружу:** нет. Grep на `create_issue|add_comment|create_page|update_page` по `src/` даёт только ВНУТРЕННИЕ вещи (`product.py:1489` — комментарий в своей БД, `meetings.py:426` — публикация своего отчёта). Единственный `.post()` в интеграционном слое — Qdrant scroll (`rag2_harness.py:338`).

**Вывод по п.4:** сейчас helm ходит в Atlassian под **одной технической учёткой** (Jira — логин/пароль Basic, Confluence — сервисный PAT), в Allure — под одним Api-Token. По коду **не видно ничего**, что позволяло бы выпускать токены за других людей: ни вызовов Jira/Confluence admin-API управления PAT, ни impersonation-заголовков, ни OAuth-флоу к Atlassian. Прав техучётки на сервере не проверял (на серверы не ходил) — но даже при наличии прав механики в коде нет вообще, её писать с нуля.

## 5. API-поверхность и миграции — есть куда встраивать

- **~30 роутеров** в `src/helm/interface/api/routers/`, монтируются в `main.py:101-165`. Общий гейт `_AUTH = [Depends(require_user)]` (`main.py:105`); исключения со СВОИМ гейтом: `docs` (публичный, `:130`), `vectors` (per-handler, `:135`), `hrd` (`require_hrd`, `:158`), `bot_support` (секрет-заголовок, `:164`), MCP-mount'ы (`:173-174`).
- **Готовый паттерн админки** — `routers/settings.py` («Настройки → Справочник»): чтение под `require_user`, мутации под `require_full_access` (`:96`), каждая правка → append-only `refdata_audit` с email актора (`models.py:1285-1297`). Там же **уже есть CRUD ролей**: `GET/POST /api/user-roles` (`settings.py:366-375`). Это ближайшая по смыслу точка встраивания раздела «Ключи доступа».
- **Миграции — alembic**, ~60 ревизий в `alembic/versions/`, `alembic.ini` в корне. Стиль — **ручные** миграции, не autogenerate (прямо сказано: `own210_ownership_assignment.py:18` «Ручная миграция (не autogenerate, §fastapi)»), линейная цепочка `revision`/`down_revision`, докстринг с обоснованием индексов. Добавление таблицы — механически дёшево.
- Все таблицы несут `tenant` с row-level фильтрацией (`domain/tenancy.scoped`, ADR-0005/mt195) — новая таблица обязана его унаследовать.

## 6. Границы: что репозиторий говорит о своей роли

`docs/adr/` — **всего 8 ADR** (0001-0008; номер 0002 занят дважды: `0002-adp-adoption-dashboard-model.md` и `0002-operator-review-console.md`). ADR-0051 и ADR-0058 в helm **отсутствуют** — они живут в `tacticum-dev` (`/Users/bubblemac/tacticum/tacticum-dev/docs/adr/0051-...`, см. [[explore-iva-write-multikey]]).

Прямого документа «helm — система записи» или «helm — не система записи» **нет**. Что есть косвенно:
- `infrastructure/allure/client.py:10-11` — явный самозапрет на запись во внешний контур заказчика («Только чтение — ничего в TestOps не пишем»);
- ADR-0003 `0003-rag-assistant-llm-contour-and-knowledge-scope.md` — контур LLM и границы корпуса знаний (детально не читал);
- `README.md:134-139` — секреты только в `.env` у оператора;
- `.gitignore:34-37` — `.env` не коммитим.

Иными словами, **действующая норма helm сегодня — «читаем чужие системы, пишем только в свою БД»**. Хранилище персональных ключей + запись под человеком это первое, что её нарушает. Отдельного ADR под это в helm пока нет — «не проверял» тут не годится, документа просто не существует.

## Риски

- **R1 — переопределение актора параметром.** `my_todo(email=...)` (`analyst_server.py:1591`) даёт сменить актора без всякой проверки прав. Если тем же приёмом выбирать ключ для ЗАПИСИ — любой аутентифицированный пишет под чужой личностью. Прототип копировать нельзя: выбор ключа обязан идти от `Principal.email` и только от него.
- **R2 — крипто-слой отсутствует полностью.** Ни библиотеки, ни ключа шифрования, ни места под него. Чужие персональные токены в открытых колонках Postgres = компрометация всех людей разом при одном дампе БД. Нужно решить, где живёт мастер-ключ (env? KMS? sealed-secrets — уже висит в бэклоге `scripts/seed_devops_track.py:45`), и это решение крупнее самой таблицы.
- **R3 — dev-принципал в открытую.** При `auth_required=False` (дефолт! `config.py:59`) и REST, и MCP отдают `dev@local` с ролью `platform_admin` и ВСЕМИ классами данных (`auth.py:60-69`, `analyst_server.py:313-320`). Для БД с чужими ключами дефолт-открыто недопустим: keystore обязан требовать auth независимо от флага, fail-closed.
- **R4 — механики выпуска токенов нет ни в каком виде.** Ни Jira/Confluence admin-API, ни OAuth, ни impersonation. «Выпускаем токен ЗА человека» упирается либо в ручной ввод человеком своего PAT (тогда это хранилище, а не выпуск), либо в админ-права техучётки в Atlassian, которых по коду не видно и на сервере я не проверял. Это разница между двумя разными продуктами — развести до дизайна.
- **R5 — нет Jira-идентификатора человека.** Есть только `email` и `assignee_name` строкой (`models.py:1795`); `my_todo` матчит по display-name и сам признаёт разъезд (`matched=false`, `diagnostic`). Ключ надо привязывать к чему-то, что Jira подтверждает, — сейчас такого поля нет.
- **R6 — ось «система» отсутствует в модели владения.** `ownership_assignment.scope_kind ∈ {client, product, area, container}` (`models.py:1398`) — про предметные измерения, не про системы. «Ключ Иванова к Atlassian» в эту модель не ложится без новой оси; натягивать `scope_kind='system'` — ломать смысл ADR-0008.
- **R7 — ротация/отзыв не предусмотрены нигде.** Единственный прецедент TTL — кэш dev-токенов 300с (`dev_token.py:31`). Ни срока жизни, ни отзыва, ни уведомления о протухании в helm нет — а для чужих ключей это обязательная часть, не «потом».
- **R8 — аудит есть, но не под это.** `refdata_audit` (`models.py:1285-1297`) пишет `value` НОВЫМ ЗНАЧЕНИЕМ текстом. Переиспользовать его для ключей нельзя — утечёт секрет в аудит-лог. Нужен свой аудит, пишущий только факт и метаданные.

## Уточняющий вопрос

Что именно заказывает руководитель — **хранилище** (человек сам приносит свой PAT, helm его шифрует и использует) или **выпуск** (helm админ-правами техучётки создаёт токен за человека)? Это два разных проекта: первый упирается в крипто-слой и UI, второй — ещё и в админ-права в Atlassian, которых по коду не видно. Дизайн начинать до ответа не стоит.

## Что НЕ проверял

- ADR-0003 и остальные ADR helm целиком (читал заголовки + ADR-0008 подробно).
- Права техучёток Jira/Confluence/Allure на серверах — на серверы не ходил by design.
- Транзитивные зависимости (`uv.lock`) на предмет наличия `cryptography` — смотрел только `pyproject.toml`.
- `web/` (SPA) — на предмет существующих экранов настроек.
- Тесты `tests/` — покрытие auth-путей.
