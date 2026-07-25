---
title: impl-my-todo
type: note
permalink: tacticum/00-board/impl-my-todo
tags:
- implementer
- my-todo
- helm
---

# impl-my-todo

Реализован MCP-тул `my_todo` (2C) в helm-analyst. Работа велась изолированно в git-worktree, не в основном дереве. Не мержил/не пушил/не деплоил.

Спека: [[Спека 2C my_todo — для implementer (2026-07-21)]] · разведка: [[explore-my-todo-helm-analyst]]

## Worktree / ветка
- worktree: `/Users/bubblemac/tacticum-worktrees/helm-my-todo`
- ветка: `feat/my-todo` (от `main` @ 1da2438)
- коммит: `9468d9d` feat(analyst-mcp): my_todo — персональный список дел актора

## Изменённые файлы:символы
- `src/helm/interface/api/routers/task_mgmt.py`
  - импорты: +ArchNode, ClientBundle, Person, PersonEmail, ProductArea, ProductAreaComponent, RequirementApproval, RequirementClient, RequirementComponent, RequirementJira
  - новые модели Pydantic: `MyTodoTaskItem`, `MyTodoApprovalItem`, `MyTodoOut`
  - хелпер `_min_due_for_reqs(...)`; константы `_PILOT_RANK`/`_RANK_TO_PILOT`/`_DUE_FAR`
  - чистый хендлер `async _my_todo(session, actor_email) -> MyTodoOut` (вся логика)
- `src/helm/interface/mcp/analyst_server.py`
  - импорт `task_mgmt as task_mgmt_router`
  - новый `@mcp.tool() async def my_todo(ctx, email=None)` — тонкая обёртка (`_require_principal` → `_with_session(task_mgmt_router._my_todo)` → `cast(dict, out.model_dump())`). 19-й тул.
- `tests/interface/test_analyst_mcp.py`
  - `test_all_tools_registered`: 18→19 (+"my_todo")
  - импорты моделей + фикстура `my_todo_app`
  - 4 кейса: `test_my_todo_tasks_filtered_and_enriched`, `test_my_todo_approvals_awaiting_me_via_arch_owner`, `test_my_todo_unknown_actor_matched_false`, `test_my_todo_approvals_via_product_po`

## Что реализовано (по спеке)
- **Шаблон-идиома** как у `who_to_involve`/`arch_map`: fail-closed `_require_principal(ctx)` первой строкой, актор = `email or principal.email`, логика в роутере, возврат структурный `dict` (`cast(dict, model_dump())`), без прозы.
- **Актор → display-name**: DB-мост `PersonEmail.email → Person.person_name` (учёт нескольких email/имён одного человека), фолбэк — локальная часть email. Матч против `EpicTask.assignee_name` нормализованный (`_norm`, регистр/пунктуация/ё). При промахе НЕ падаем — пустой результат + `matched=false` как сигнал разъезда имён.
- **tasks**: открытые задачи (status_category != done, max as_of), назначенные на актора. Оси `high_priority` (`_HIGH_PRIORITIES`) и `blocked` (`_is_incoming_block`, входящие Blocks) переиспользованы из task_attention. На каждой задаче: `criticality` (`RequirementClient.pilot_priority`, max по клиентам), `due_date` (ClientBundle, min), `waiting_approvals` (число pending-гейтов требования). Мост задача↔требование — `RequirementJira.jira_key ↔ EpicTask.key`.
- **Критичность** — именно `RequirementClient.pilot_priority` (critical|desired|later|none), НЕ Requirement.
- **approvals_awaiting_me**: pending-гейты (текущее состояние = max as_of на грани requirement/gate/area) по ОБЕИМ осям владения — `ArchNode.owner_person_id` (area гейта = узел C4) И `ProductArea.po_email` (по компонентам требования через RequirementComponent, removed=False). Плюс прямой сигнал `approver == актор`. Поле `via` = arch_owner|product_po|approver.
- **Сортировка задач**: (1) ждут:N ↓ → (2) критичность ↓ → (3) срок ↑ (None→в конец) → (4) застой (days_in_status) ↓.

## Результат тестов (реальные числа)
- `pytest tests/interface/test_analyst_mcp.py tests/interface/test_task_mgmt.py` → **86 passed, 1 warning**.
  - test_analyst_mcp: 74 passed (было 70; +4 кейса my_todo + обновлён registry).
  - test_task_mgmt: 12 passed (регрессий нет).
- Warning — предсуществующий шум teardown aiosqlite («Event loop is closed»), не связан с правкой (фикстуры не dispose'ят engine — так же во всех тестах файла).
- `ruff check` (3 файла) — clean.
- `mypy` изменённых файлов — clean по моему коду; остаётся 1 предсуществующая ошибка `no-any-return` на строке 591 (`arch_drift`, не мой код — та же идиома уже была не строгой).

## Открытые вопросы / риски
- **IdentityDirectory-фолбэк НЕ подключён** (сознательно). Спека упоминала доменный фолбэк `IdentityDirectory.resolve`, но он читает CSV `data/identity/identity_map.csv` (отдельное от БД хранилище, требует filesystem-связки и CSV-фикстуры в тестах). Оставил только DB-мост (PersonEmail→Person) + email-локалчасть, что покрывает канонический in-process путь и держит тул самодостаточным/тестируемым на чистом SQLite. Если нужна максимальная точность матча ФИО — можно добавить резолвер вторым фолбэком (потребует загрузчик identity_io + путь к данным в app.state). **Решение за тимлидом.**
- **email→name строковый матчинг** остаётся главным gap точности: если Jira display-name разошёлся с Person.person_name или человека нет в person_email → ложное «пусто». Смягчено `matched=false`-сигналом.
- **due из ClientBundle** беру как min `due_date` (легаси-алиас критичного срока) по всем бандлам клиентов требования; уровневые `due_critical/desired/other` не разбивал по фазировке — при желании можно сузить срок под `criticality` задачи (не заложено, чтобы не раздувать объём).
- **Мульти-требование на один jira_key**: агрегирую (criticality=max, due=min, waiting=sum), в `requirement_id` кладу первый — как диагностику.
- Данные снапшотные (as_of последнего среза), не realtime — консистентно с task_attention.

## Связано
- [[Спека 2C my_todo — для implementer (2026-07-21)]]
- [[explore-my-todo-helm-analyst]]
