---
title: 'План: три деливеравла ИВА (iva-write / 3 роли / my_todo) — привязка к системе'
type: report
permalink: tacticum/20-architecture/plan-tri-deliveravla-iva-iva-write-3-roli-my-todo-priviazka-k-sisteme
status: current
created: 2026-07-21 08:10
updated: 2026-07-21 08:10
project: helm / ИИ-контур ИВА
tags:
- plan
- helm
- iva-write
- role-presets
- my-todo
- adr-0058
- requirements
---

# План: три деливеравла ИВА — привязка к текущей системе

Апрувный план (грунт проверен по коду). Собран из ADR-0058 + реального устройства helm-analyst и платформы профилей tacticum-dev. Владелец — @aleksandr_shulga0507. Личный PAT (переопределяет техучётку из ADR-0058 §5).

## Грунт (проверено по коду)
- **helm-analyst** = `helm/src/helm/interface/mcp/analyst_server.py` (17 тулов `@mcp.tool()` на одном FastMCP, путь `/mcp/analyst`, прод `helm.tacticum.ru/mcp/analyst`). Каждый тул — тонкая обёртка над REST-хендлерами; application/domain не дублируется. Актор — `_require_principal(ctx).email` через project-hub `/resolve` (ADR-0051), fail-closed. `my_todo` в коде НЕТ.
- **Данные под my_todo уже есть:** `EpicTask` (снапшот Jira: assignee_name, priority, status, links, changelog), готовый `GET /task-attention` (routers/task_mgmt.py) уже считает оси `blocked` (входящие Blocks) + `high_priority`. Гейты — `RequirementApproval(gate: sales|cto|cpo, area)`. Владение — `ProductArea.owner_email`. Маппинг req↔US — `RequirementJira`.
- **tacticum-dev:** роли-пресеты = тонкие leaf (`ingredients: []`, всё через `depends_on` depth-1). Тесты `apps/backend/tests/catalog/test_iva_role_presets.py`: single-owner (лейны попарно не пересекаются по ingredient_id), golden-parity, depth1. `fr-authoring` уже зовёт iva-mcp (`confluence_create_page`, `jira_create_issue`), но analysis-base его НЕ провиженит (только helm-analyst + iva-read); write = follow-up Taiga #712.

## Ключевые находки (влияют на дизайн)
1. **helm-analyst монолитен** — bearer открывает все 17 тулов, срез на уровне манифеста невозможен. Arch-тулы архитектору приходят вместе с ингредиентом `helm-analyst` (живёт в analysis). Субсетить нечего — вопрос лишь какие *скиллы* дать.
2. **iva-write обязан быть отдельным лейном `iva-write-base`** (single-owner: нужен 5 ролям с разными наборами; техпис без analysis) — не в core (не все пишут), не в analysis.
3. **iva-write = продакшн того же write-канала, что уже зовёт fr-authoring** (iva-mcp), НЕ новый параллельный сервер. Сохранить имена тулов, чтобы fr-authoring не переписывать.

## 2A — iva-write (личный PAT)
Продакшн-оформление write-канала fr-authoring: инстанс mcp-atlassian за `mcp.tacticum.ru` (рядом с iva-read), личный PAT сотрудника (нативная атрибуция в Jira/Confluence — автор = сам сотрудник). Оформить как лейн `templates/iva-write-base/manifest.yaml` (1 ингредиент mcp_server_spec). Скоуп: confluence create/update, jira create/comment/transition (статусы ЖЦ), Allure-статусы. Права по ролям через scope hub-ключа `iva-req-write`.
- **Сегодня:** дизайн-спека + PoC на песочном Jira/Confluence + скелет лейна.
- **Блокер (прод):** IVAREQ + space + PAT-права — после утверждения ADR-0058 + админ Jira (Монахов).

## 2B — три роли-пресета (композиция ADR-0057)
- `iva-role-architect` = core + analysis + iva-write-base (arch-тулы бесплатно с helm-analyst).
- `iva-role-qa` = core + analysis (tests-authoring + requirement_tests) + iva-write-base; прогоны — родной Allure; статусы через iva-write.
- `tacticum-role-techwriter` = core + documentation-base + iva-write-base.
Для каждой: manifest (по образцу iva-role-analyst) + README/quickstart + строка в ROLE_LANES теста + провижн. Проверить enum персон (architect/qa/techwriter) в `_schema/manifest.v2.schema.json`.
- Зависит от 2A (лейн iva-write-base). Сборка+тесты — не ждут прод-контура.

## 2C — my_todo в helm-analyst
18-й `@mcp.tool()` в analyst_server.py, тонкая обёртка: актор из `principal.email` (+ опц. param email). Переиспользует логику `task_attention` (blocked + high_priority), фильтр по актору + гейты (RequirementApproval) + владение (ProductArea.owner_email) + критичность (Requirement.pilot_priority). Выход — структурные данные, сортировка: (1) ждут:N → (2) критичность → (3) этап/срок. Без прозы.
- **Блокеров нет — старт первым.**
- ⚠️ Gap точности: `EpicTask.assignee_name` = display name, а principal = email → нужен мост email→name через `Person`/`PersonEmail`. Заложить в дизайн.

## Порядок + матрица
1. my_todo (2C) первым — блокеров нет. 2. iva-write (2A) дизайн+PoC+скелет лейна параллельно. 3. Три роли (2B) — после появления лейна iva-write-base.
| Деливеравл | Зависит от | Внешний блокер |
|---|---|---|
| 2C my_todo | — | нет |
| 2A iva-write (дизайн+PoC) | — | нет |
| 2A iva-write (прод) | ADR-0058 | IVAREQ/space/PAT — Монахов |
| 2B роли | 2A (лейн), enum персон | боевая запись ждёт 2A-прод |

## Три развилки на апрув (архитектурные — решает пользователь)
1. iva-write как отдельный лейн `iva-write-base` (вынужденно из-за single-owner) — подтвердить.
2. Архитектор: **Вариант A** (реюз полного iva-analysis-base, MVP, чисто) vs B (новый iva-architecture-base с переносом adr-authoring — дороже, трогает существующие роли). Рекомендация — A.
3. iva-write = продакшн существующего write-канала fr-authoring (те же имена тулов), НЕ новый сервер — подтвердить.

## Связь с e2e-санити (отдельно)
Профиль аналитика (собран ночью Diaret): ядро (TC + контракты) работает на живых данных; BRD/ADR/FR — не подтвердить без kb_*/iva-mcp в среде роли. 2 бага в скиллах: gap_questions/who_to_involve зовут `query` вместо `requirement_text`; fr-authoring publish-дрейф vs manifest-MVP. См. [[Созвон 3 (Diaret) — разбор- конвейер артефактов, профиль аналитика, mcp-iva-write, привязка к helm]].
