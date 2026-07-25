---
title: plan-iva-tri-deliveravla (my_todo · iva-write · 3 роли) — APPROVED, передача
  лиду
type: note
permalink: tacticum/00-board/plan-iva-tri-deliveravla-my-todo-iva-write-3-roli-approved-peredacha-lidu
status: APPROVED
created: 2026-07-21 08:25
updated: 2026-07-21 08:25
role: director→lead
autonomy: 'off'
scope: три деливерабла, старт с my_todo
project: helm / tacticum-dev
tags:
- plan
- approved
- handoff
- helm
- iva-write
- my-todo
- role-presets
---

# plan-iva-tri-deliveravla — APPROVED (ГД → лид)

Владелец: @aleksandr_shulga0507. Полный план с грунтом по коду: [[План- три деливеравла ИВА (iva-write - 3 роли - my_todo) — привязка к системе]]. Здесь — апрувленный хендофф лиду.

## Режим
- **autonomy: off** — пользователь на смене (present). Исполнять до **controller-гейта**; перед любым мержем/пушем — OK пользователя (гит-гигиена: только feature-ветка явно, без AI-подписей, PR не создаём).
- Ждём более полный ответ Diaret, **но старт разрешён**: всё в скоупе первого этапа от его ответа не зависит.

## Зафиксированные решения (развилки закрыты)
1. **iva-write = продакшн существующего write-канала `fr-authoring`** (тот же iva-mcp, сохранить имена тулов `confluence_create_page`/`jira_create_issue`/…), НЕ новый параллельный сервер. ✅
2. **iva-write оформляется отдельным лейном `iva-write-base`** (вынужденно — single-owner). ✅
3. **Аутентификация — личный PAT сотрудника** (финал; переопределяет техучётку из ADR-0058 §5). ✅
4. **Архитектор — Вариант A:** реюз полного `iva-analysis-base` (arch-тулы приходят с монолитным helm-analyst; отдельный arch-лейн с переносом adr-authoring НЕ делаем). ✅

## Скоуп и порядок исполнения
**Этап 1 (старт сейчас) — `my_todo` до результата (фича «до результата»):**
- Файл: `helm/src/helm/interface/mcp/analyst_server.py` — 18-й `@mcp.tool() async def my_todo(ctx, email: str | None = None)`, тонкая обёртка (application/domain не дублировать).
- Актор — `principal.email` из `_require_principal(ctx)` (ADR-0051). Переиспользовать логику `routers/task_mgmt.py::task_attention` (оси `blocked` + `high_priority`), отфильтровать по актору + добавить гейты (`RequirementApproval`) + владение (`ProductArea.owner_email`) + критичность (`Requirement.pilot_priority`).
- Выход — структурные данные (без прозы), сортировка: (1) ждут:N → (2) критичность → (3) этап/срок/свежесть.
- ⚠️ **Gap точности — решить в дизайне до кода:** `EpicTask.assignee_name` = display-name, актор = email → мост `email→name` через `Person`/`PersonEmail`. Explorer: подтвердить, есть ли надёжный мост / email ассайни в ингесте Jira.
- Acceptance: юнит по образцу `tests/interface/test_analyst_mcp.py` + **сверка с реальным источником** (правило: зелёные тесты ≠ верные данные) — выдача для конкретного человека сверяется с его Jira/task-attention; `waiting_for:N` == числу реальных входящих blocks.

**Этап 2 (параллельно) — `iva-write-base` дизайн-спека + PoC + скелет лейна:**
- `templates/iva-write-base/manifest.yaml` (1 ингредиент `mcp_server_spec` iva-write, `url: https://mcp.tacticum.ru/iva-write/mcp`, bearer, личный PAT в env). Скоуп тулов = superset того, что зовёт `fr-authoring` + `jira transition_issue` (статусы ЖЦ) + Allure-статусы.
- PoC на песочном Jira/Confluence (личный PAT): create page + create issue + transition — проверить нативную атрибуцию.
- ⛔ Прод (боевой IVAREQ/space/PAT-права) — ждёт утверждения ADR-0058 + админа Jira (Монахов). В этот этап НЕ входит.

**Этап 3 (после появления лейна `iva-write-base`) — три роли:**
- `iva-role-architect` = `[tacticum-core-base, iva-analysis-base, iva-write-base]`.
- `iva-role-qa` = `[tacticum-core-base, iva-analysis-base, iva-write-base]` (tests-authoring + requirement_tests; прогоны — родной Allure; статусы через iva-write).
- `tacticum-role-techwriter` = `[tacticum-core-base, tacticum-documentation-base, iva-write-base]`.
- Каждая: тонкий leaf (`ingredients: []`) + manifest по образцу `iva-role-analyst` + README/quickstart + строка в `ROLE_LANES` теста `apps/backend/tests/catalog/test_iva_role_presets.py` + провижн-проверка.
- Проверить enum персон (architect/qa/techwriter) в `templates/_schema/manifest.v2.schema.json` (при необходимости расширить; прецедент — helm-миграция `0026_membership_role_architect.py`).
- Acceptance: `test_iva_role_presets.py` зелёный (single-owner-disjoint / golden-parity / depth1) = доказательство «не дублирует»; провижн на CLI + дымовой прогон (architect → arch_map/contradiction_check на реальном требовании; QA → requirement_tests).

## Воркеры (предложение лиду)
explorer (подтвердить мост email→name + место встройки my_todo; структуру лейнов/ролей) → implementer (в git-worktree: my_todo; затем iva-write-base + PoC; затем 3 роли) → verifier (тесты + сверка на реальных данных) → controller-гейт перед мержем.

## Развилки на потом (в ответе Diaret, не блокируют старт)
Мост email→имя для ассайни; финальный список тулов iva-write; когда IVAREQ/space заводит Монахов.

## ⛔ ОБНОВЛЕНИЕ (2026-07-21, после утреннего созвона) — читать лиду

**GUARDRAIL (критично):** сегодня Diaret учит аналитиков (Лавров + Тарасова + их ребята, чат ~10:30) **на текущем профиле аналитика**. Поэтому:
- **НЕ трогать и НЕ ломать** ничего связанного с аналитиками: лейн `iva-analysis-base`, роль `iva-role-analyst`, их скиллы/манифесты. Сегодня они в проде обучения — заморожены.
- Работа лида ведётся ТОЛЬКО в: `helm` (my_todo — новый тул), новый лейн `iva-write-base` (новые файлы), новые роли-пресеты (новые файлы). Существующие analysis-сущности не редактировать.

**QA-роль — уточнение:** собирать **из того, что уже есть в tacticum-dev** (там уже много описано под QA) — не ждать. Скиллы QA от Diaret (Allure-интеграция) придут позже → доинтегрируем. Explorer: найти существующий QA-контент в tacticum-dev, лид соберёт `iva-role-qa` на нём, оповестить о находках.

**iva-write — уточнение по ключам (в дизайн-спеку):** тот же сервер, что iva-mcp/EVA; пишут под своими учётками, в т.ч. в Allure. Ключей потенциально 3-4: hub-ключ (ПХК/phk_*) + PAT Jira/Confluence + Allure-PAT + общий tactic. Продумать мульти-ключевую схему (несколько ключей на адрес).

**Приоритет сместился:** тест профиля аналитика + обучение — P0 у ГД/пользователя (не у лида). Лид ведёт P1 (my_todo → iva-write-base → роли architect/qa/techwriter) параллельно, соблюдая guardrail.
