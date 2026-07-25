---
title: Спека 2C my_todo — для implementer (2026-07-21)
type: note
permalink: tacticum/00-board/speka-2-c-my-todo-dlia-implementer-2026-07-21
status: ready-for-impl
role: lead (тимлид)
date: 2026-07-21
tags:
- spec
- my-todo
- helm
- implementer
---

# Спека: my_todo (2C) — тонкий MCP-тул в helm-analyst

Грунт выверен разведкой [[explore-my-todo-helm-analyst]]; поправки к плану и решения пользователя учтены ниже.

## Куда
`src/helm/interface/mcp/analyst_server.py` — новый `@mcp.tool()` `my_todo`, будет **19-м** (не 18-м). Шаблон — по образцу `who_to_involve`: `await _require_principal(ctx)` (fail-closed, `Principal.email`) → `_with_session(router_fn)` → `.model_dump()`. Проза не генерится — структурный выход.

## Актор
Из `Principal.email` + опциональный param `email` (override). Мост email→name: `PersonEmail.email → Person.person_name` (образец с фолбэком `cio.py:5775-5790`), доменный фолбэк `IdentityDirectory.resolve` (`domain/identity_directory.py:39`). Матч против `EpicTask.assignee_name` (⚠️ строковый — заложить фолбэк/нормализацию, не падать при промахе).

## Логика (переиспользовать task_attention)
`routers/task_mgmt.py:863-949` — оси `high_priority` (`_HIGH_PRIORITIES`+tier+days_in_status) и `blocked` (`_is_incoming_block`, входящие Blocks) + чистые хелперы (`_parse_json_list`, `_last_status_date`, `_days_since`, `_asof_ref_date`). Фильтр по актору добавить поверх (сам task_attention актора не принимает).

## Поправки к плану (ФАКТ по коду)
- Критичность — **`RequirementClient.pilot_priority`** (critical|desired|later|none), НЕ `Requirement.pilot_priority`.
- **Владение — ОБЕ оси** (решение пользователя): `ProductArea.po_email` (PO) И `ArchNode.owner_person_id` (гейтовый апрувер; `RequirementApproval.area` ссылается на ArchNode). `RequirementApproval`: approver=email, gate, verdict=pending.
- EpicTask↔Requirement сшивать через `RequirementJira.jira_key ↔ EpicTask.key`.

## Сортировка (3 оси)
(1) ждут:N → (2) критичность (RequirementClient.pilot_priority) → (3) **этап/срок из ClientBundle** (решение пользователя — дотянуть сроки из ClientBundle, в task-модели их нет).

## Тесты (обязательно, на данных)
- `tests/interface/test_analyst_mcp.py` — обновить `test_all_tools_registered` (18→19), добавить кейс my_todo (эфемерная SQLite `_sqlite_factory`).
- Образец фикстур — `tests/interface/test_task_mgmt.py::test_task_attention` (стр. 503, хелперы `_chlog/_links/_ago`).

## Границы
Работать ИЗОЛИРОВАННО в git-worktree, НЕ в основном дереве. Не мержить/не пушить/не деплоить. Результат — на доску.

## Связано
- [[План- три деливеравла ИВА (iva-write - 3 роли - my_todo) — привязка к системе]]
- [[Развилки плана iva-tri-deliveravla — решения пользователя (2026-07-21)]]