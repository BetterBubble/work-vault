---
title: explore-my-todo-helm-analyst
type: note
permalink: tacticum/00-board/explore-my-todo-helm-analyst-1
status: draft
role: explorer
repo: /Users/bubblemac/tacticum/helm
tags:
- explorer
- my-todo
- helm
archived-at: 2026-08-03 11:16
---

# explore-my-todo-helm-analyst

Разведка точек вшивки нового MCP-тула `my_todo` в helm-analyst. Read-only, правок нет.
План: [[План- три деливеравла ИВА (iva-write - 3 роли - my_todo) — привязка к системе]]

## 0. Поправки к формулировке задачи (важно перед вшивкой)
- **Тулов сейчас 18, не 17.** `grep '@mcp.tool()'` = 18; `test_all_tools_registered` перечисляет ровно 18 имён. `my_todo` будет **19-м**, а не 18-м. При добавлении обязательно расширить множество в тесте (иначе `test_all_tools_registered` падает — см. §5).
- **`Requirement.pilot_priority` не существует.** `pilot_priority` живёт на `RequirementClient` (см. §3). У `Requirement` есть только `priority: str|None` и `core: bool`.
- **`ProductArea.owner_email` не существует.** Поле владельца области — `ProductArea.po_email` (FK `person_email.email`). Гейтовое владение (`RequirementApproval.area`) вообще НЕ про ProductArea — `area` = id узла C4 (`arch_node`), владелец узла — `ArchNode.owner_person_id` (FK person). Это ДВЕ разные модели владения — см. Риски.

---

## 1. MCP-сервер — шаблон обёртки
Файл: `src/helm/interface/mcp/analyst_server.py`

**Актор / fail-closed.** `_require_principal(ctx: Context) -> Principal` (стр. 261-328). Поток: Bearer из `ctx.request_context.request.headers` → project-hub `/resolve` (или dev-MCP `phk_*`) → `principal_for_tenant(...)`, fail-closed (нет членства в тенанте `iva` → `raise ValueError`). При `settings.auth_required=False` — открытый dev-принципал `Principal(email="dev@local", roles=("platform_admin",), ...)` БЕЗ чтения `ctx`. **`principal.email`** — то, что нужно `my_todo` как актор.
- Роли из БД: `_db_roles_for(app, email) -> tuple[str,...]` (стр. 251) — читает `user_role`.

**Доступ к БД.** `_with_session(fn, **kwargs)` (стр. 135-144) — открывает сессию из `app.state.session_factory` и зовёт REST-хендлер напрямую с `session=` (в обход `Depends`). Именно так тулы переиспользуют роутеры. `_require_app()` — достаёт FastAPI из модульного `_app`; `configure(app)` — инъекция.

**Канонический шаблон тула (все 18 одинаковы):**
```python
@mcp.tool()
async def <name>(ctx: Context[Any, Any, Any], <params>) -> dict[str, Any]:
    """<docstring — попадает в описание тула для LLM>"""
    app = _require_app()               # если нужен app напрямую
    await _require_principal(ctx)      # ПЕРВЫМ делом, fail-closed
    out = await _with_session(<router_fn>, **kwargs)  # переиспользуем роутер
    return out.model_dump()            # структурные данные, dict[str, Any]
```
Возврат всегда `dict[str, Any]` (структура без прозы). Примеры для образца:
- `requirement_tests` (стр. 446-488) — ветвистый возврат вручную собранным dict.
- `who_to_involve` (стр. 1061-1116) — **единственный, кто идейно близок my_todo**: собирает людей (`owners`/`recent_contributors`), несколько источников, `_unique(...)`, финальный dict.
- `arch_map` (стр. 494-541) — `_with_session(cio_router.arch_map, ...)` + `.model_dump()`.

Все 20 вызовов `_require_principal(ctx)` — строки перечислены в теле (347,364,378,402,422,439,464,512,557,587,695,751,1029,1075,1143,1260,1399,1434). Ни один тул сейчас НЕ фильтрует по `principal.email` — my_todo будет первым.

---

## 2. task_attention — что переиспользуемо
Файл: `src/helm/interface/api/routers/task_mgmt.py`, хендлер `task_attention` (стр. 863-949), `@router.get("/task-attention", response_model=TaskAttentionOut)`.

**Логика (обе оси на одном срезе `max(EpicTask.as_of)`, только `status_category != "done"`):**
- **high_priority:** фильтр `r.priority in _HIGH_PRIORITIES` (стр. 297-299: Критический/Высокий/Blocker/Highest/Critical/High). `tier = 0 if priority in _TOP_PRIORITIES else 1` (298-301). Сортировка `(tier ASC, days_in_status DESC)`. `days_in_status = _days_since(_last_status_date(r.changelog), ref)`, `ref = _asof_ref_date(max_as_of)` (от среза, не сегодня). Топ-30.
- **blocked:** для каждой задачи `blocked_by = [link.k for link in _parse_json_list(r.links) if _is_incoming_block(link)]`. `_is_incoming_block` (стр. 304-310): `("blocks"|"blocked" in t.lower()) and d=="in"`. Сортировка по числу блокеров DESC. Топ-30.

**Переиспользуемые хелперы (модульные, чистые):**
- `_parse_json_list(raw) -> list[dict]` (73-86) — безопасный парс JSON-строк links/changelog/comments.
- `_last_status_date(changelog_raw) -> str|None` (89-97).
- `_days_since(iso, ref) -> int|None` (100-111), `_asof_ref_date(tasks_as_of) -> date` (114-121).
- `_is_incoming_block(link) -> bool` (304-310).
- Константы `_HIGH_PRIORITIES` / `_TOP_PRIORITIES`.

**Возвращаемые модели** (все `BaseModel` в этом же файле): `TaskAttentionOut{as_of, high_priority: TaskAttentionHighGroup, blocked: TaskAttentionBlockedGroup}` (290-293); `TaskAttentionHighItem{key,summary,priority,assignee_name,status,epic}` (260-266); `TaskAttentionBlockedItem{key,summary,assignee_name,status,blocked_by:list[str]}` (269-274); группы (277-287).

**Что переиспользуемо для my_todo:** вся ось blocked (входящие Blocks) и вся ось high_priority — 1:1. `EpicTask`-снапшот, срез, хелперы дат/статусов. НЕДОСТАЁТ (нет в task_attention): фильтр по актору, гейты (`RequirementApproval`), критичность из `RequirementClient.pilot_priority`, срок/этап. `task_attention` НЕ принимает актора — фильтр по `principal.email` придётся добавить поверх (см. §4 мост).

---

## 3. Модели данных
Все в `src/helm/infrastructure/db/models.py`.

**`EpicTask`** (стр. 1358-1403, `__tablename__="epic_task"`), PK `key`. Ключевые: `epic: str|None (index)`, `status`, `status_category` (new|indeterminate|done), **`assignee_name: str`** (Jira display name, НЕ email!), `components: str` (CSV), `summary: Text`, `priority: str`, `reporter_name`, `updated`, `created`, `sprint`, `parent`, `estimate`, `time_spent`, `links: Text` (JSON-массив {t,k,d}), `changelog: Text` (JSON), `comments: Text` (JSON), `as_of: str (index)`. Идемпотентность — replace по as_of.

**`RequirementApproval`** (стр. 1666-1696, `requirement_approval`). PK id, uq `(requirement_id, gate, area, as_of)`. Поля: `requirement_id (FK requirement.id)`, `gate` (sales|cto|cpo), **`area`** (= `arch_node.id` для cto, "" = финал/без зоны), `verdict` (approved|rejected|pending|na), **`approver: str`** (email актора), `comment`, `as_of` (текущее состояние = последняя as_of), `source` (manual|jira). Для my_todo «ждут согласования»: verdict=pending на последней as_of.

**`ProductArea`** (стр. 163-179, `product_area`). PK `area_id`, `name`, **`po_email: str|None`** (FK `person_email.email`) — владелец области (НЕ `owner_email`!), `active`. M:N с компонентами — `ProductAreaComponent` (182+, `area_id`+`component`).

**`Requirement`** (стр. 314-381). PK `id: str`. `title`, `description`, `status` (candidate|committed|done), **`priority: str|None`**, `core: bool`, `manager: str|None`, `target_quarter: str|None`, `req_class/req_type/req_subtype`, `scope`. **Нет `pilot_priority`.**

**`RequirementClient`** (стр. 1262-1294, `requirement_client`). PK `(requirement_id, client_code, provenance)`. **`pilot_priority: str` (default "none")** — КАНОН фазировки: critical|desired|later|none (стр. 1282-1287). `pilot_priority_source` (manual|None), `pilot_critical: bool` (legacy-производный), `value_money`. **Критичность требования = здесь.**

**`RequirementJira`** (стр. 1585-1611, `requirement_jira`). PK `(requirement_id, jira_key)`. `jira_status`, `jira_status_category` (new|indeterminate|done), `assignee: str`, `track` (kmp|one|rn), `as_of`, `summary`, `components`, `issuetype`. **Мост требование↔Jira-задача** — понадобится, чтобы связать `EpicTask.key` ↔ `Requirement` (через jira_key) для критичности/гейтов.

**`ArchNode`** (стр. 1837+): `owner_person_id: int|None` (FK person) — владелец узла C4, на который ссылается `RequirementApproval.area`.

---

## 4. Мост email→name (⚠️ gap точности)
`EpicTask.assignee_name` = Jira display name. `principal.email` = email. Прямой связи нет.

**Канонический DB-путь (используется в cio.py):**
```python
select(PersonEmail.email, Person.person_name).join(
    Person, Person.person_id == PersonEmail.person_id)
# name_by_email[email.lower()] = person_name; фолбэк email.split("@")[0]
```
Образец с фолбэком — `src/helm/interface/api/routers/cio.py:5775-5790` (комментарий «email → каноническое имя … фолбэк — локальная часть email»). Тот же join повторяется в cio.py: 2178, 3246, 3539-3546, 4785, 5025, 5178, 5779.
- `Person` (models.py 387-406): `person_id` PK, **`person_name`**, `team`, `role`, `emails` (rel).
- `PersonEmail` (409-422): `email` PK, `person_id` (FK), `is_primary`.

Для my_todo: `principal.email` → PersonEmail→Person.person_name → сравнить с `EpicTask.assignee_name`. **Обратное направление того же join.**

**Альтернатива (домен, не БД):** `src/helm/domain/identity_directory.py` — `IdentityDirectory.resolve(query) -> Person|None` (стр. 39-52), `Person{person_email, fio, git_emails, sources}`. Резолвит и email, и ФИО (token-overlap ≥2). Источник — `data/identity/identity_map.csv` (отдельно от БД Person/PersonEmail). Полезно как фолбэк-матчинг ФИО↔email, но это НЕ то же хранилище, что DB Person.

**Gap:** нет гарантии, что `EpicTask.assignee_name` (строка из Jira-выгрузки) строково совпадёт с `Person.person_name`. Если человека нет в `person_email` или display-name разошёлся — актор не смэтчится → пустой my_todo (fail-closed по данным, но риск ложного «пусто»).

---

## 5. Тесты
- **`tests/interface/test_analyst_mcp.py`** — тесты MCP-обёрток. Паттерн: фейки портов (`_FakeRag2Search`, `_FakeRetriever`, `_FakeLlm`); dev-app `_dev_app(**state)` (стр. 106-114, `Settings(auth_required=False, tenant="iva")` + `analyst_server.configure(app)`); `_DevCtx`/`_ctx_with_headers`. БД — эфемерная async SQLite: `_sqlite_factory()` (стр. 128-133, `create_async_engine("sqlite+aiosqlite://")` + `Base.metadata.create_all`), кладётся в `app.state.session_factory`, сидируется `EpicTask/Requirement/...` напрямую.
  - `test_all_tools_registered` (стр. 137+) — **фиксирует множество из 18 имён; добавление my_todo ТРЕБУЕТ правки этого set** (иначе падает).
  - Auth fail-closed: `test_auth_missing_bearer_fails_closed`, `test_auth_invalid_token_fails_closed`, `test_auth_dev_bypass_without_bearer`.
  - Импортит `EpicTask, Person, Requirement, RequirementClient, RequirementJira` — фикстуры под них уже заводятся.
- **`tests/interface/test_task_mgmt.py`** — тесты роутера. Фикстура `tm_client` (стр. 92+) сидирует ~10 `EpicTask` с разными assignee/priority/status. **`test_task_attention` (стр. 503+)** — прямой сид `EpicTask` с `priority`, `changelog=_chlog(...)`, `links=_links((rel,key,dir),...)`; проверяет тир high_priority и входящие Blocks. Хелперы `_chlog`, `_links`, `_ago`, `TODAY` — готовый инструментарий для фикстур my_todo (high_priority/blocked). Фикстур под RequirementApproval/pilot_priority в связке с задачами — НЕТ, их надо будет добавить.

---

## Рекомендации по вшивке my_todo
1. Разместить `@mcp.tool() async def my_todo(ctx, email: str|None=None)` в `analyst_server.py` рядом с task-тематикой; первой строкой `await _require_principal(ctx)`, актор = `email or principal.email`.
2. Логику держать в `task_mgmt.py` как новый чистый хелпер/хендлер (напр. `def _my_todo(session, actor_email, actor_name)`), а тул зовёт его через `_with_session(...)` и `.model_dump()` — так же, как arch_map→cio_router. Переиспользовать `_parse_json_list`, `_is_incoming_block`, `_last_status_date`, `_days_since`, `_asof_ref_date`, `_HIGH_PRIORITIES`.
3. Мост актора: построить `name_by_email` join (cio.py:5775 как образец), развернуть в `email→person_name`, фильтровать `EpicTask.assignee_name == person_name` (+ нормализация регистра/пробелов, как в task-people).
4. Ось «ждут:N»: `RequirementApproval` verdict=pending на последней as_of, где актор — назначенный согласующий. Связать актора с зоной: gate=cto→`area`=arch_node.id→`ArchNode.owner_person_id`→Person; либо владение областью через `ProductArea.po_email`. Требуется решить, какая модель владения канонична (см. Риски).
5. Критичность: через `RequirementJira` (EpicTask.key→requirement_id) → `RequirementClient.pilot_priority` (critical>desired>later>none).
6. Сортировка (по плану): (1) ждут:N DESC → (2) критичность → (3) этап/срок. Срок — `RequirementClient` не имеет due; сроки живут в `ClientBundle.due_*` (models.py ~1418). Уточнить у тимлида источник «этап/срок».
7. Возврат — новые `BaseModel` (структурные группы, как TaskAttention*), без прозы.
8. Обновить `test_all_tools_registered` (+"my_todo") и добавить фикстуры (актор с person_email + assignee_name-матч; pending-approval; pilot_priority).

## Риски / gap
- **Off-by-one:** тулов 18, не 17 → my_todo 19-й; забыть обновить `test_all_tools_registered` = красный тест.
- **email→name строковый матчинг** ненадёжен: `EpicTask.assignee_name` (Jira) может не совпасть с `Person.person_name`; человека может не быть в `person_email`. Ложное «пусто». Нужна нормализация + фолбэк (IdentityDirectory / email-локалчасть). ⚠️ главный gap точности.
- **Две несовместимые модели владения областью:** `RequirementApproval.area` = C4 `arch_node.id` (владелец `ArchNode.owner_person_id`), а `ProductArea.po_email` — иная ось (области бэклога). План говорит «ProductArea.owner_email» — такого поля нет; надо явно выбрать, через что определять «моя зона согласования».
- **`pilot_priority` не на Requirement, а на `RequirementClient`** (грань клиента): у одного требования разные клиенты → разная критичность. Нужен агрегат (max по клиентам?) — решить правило.
- **Нет прямого EpicTask↔Requirement:** связь только через `RequirementJira.jira_key` ↔ `EpicTask.key` (форматы ключей: IVAONE-*, IVAONEHALF-*). Матчинг по ключу, не гарантирован для всех задач.
- **«Срок/этап»** в task-модели отсутствует у RequirementClient; сроки — в `ClientBundle`. Ось сортировки (3) требует уточнения источника.
- Все данные — снапшотные (`as_of`); my_todo считает от последнего среза, не «сегодня» — консистентно с task_attention, но не realtime.