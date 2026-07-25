---
title: explore-analyst-web-vs-mcp
type: report
permalink: tacticum/00-board/explore-analyst-web-vs-mcp
status: draft
role: explorer
repo: helm (main, eaf10f8)
tags:
- explore
- rag2
- mcp
- analyst
- helm
---

# Разведка: веб `/analyst` vs MCP аналитиков (helm)

## Итог одной строкой
Веб `/analyst` — это ОДИН чат RAG-контекста (Jira/Confluence/helm), дергает единственный эндпоинт `POST /api/rag2/context`. Он покрывает функционал ровно одного MCP-тула — `analyst_context`. Остальные 16 тулов MCP (в т.ч. `api_registry_check` Р-1 и `contract_check` Р-4) через браузерную страницу НЕДОСТУПНЫ.

## 1. Что такое веб `/analyst`
- Standalone-страница: `web/analyst.html` → `web/src/analyst-main.tsx` → `web/src/screens/AnalystChat.tsx`. Vite multi-page entry `analyst` (`web/vite.config.ts:26-29`).
- Это чат/поисковая форма: свободный ввод вопроса + чипы-фильтры источника (Все / Jira / Confluence / helm). Лента Q&A, ответ = КОНТЕКСТ-БЛОК с inline-цитатами [n], карточки источников, бейджи свежести. Генерации прозы-ответа НЕТ — показывается контекст (`AnalystChat.tsx:234-352`).
- Пользователь: вводит запрос → жмёт «Найти контекст».

## 2. Какой бэкенд дёргает страница
- Единственный вызов: `api.rag2Context(text, {filters})` → `POST /api/rag2/context` (`AnalystChat.tsx:256`, `web/src/api.ts:192-201`).
- `rag2Search` (`/api/rag2/search`) определён в api.ts (`api.ts:202`), но в вебе НЕ используется нигде (мёртвый метод). Проверено грепом.
- Бэкенд: `src/helm/interface/api/routers/rag2.py:197` `context()` → `ctx.orchestrator.answer(...)`, где `ctx` = `Rag2Context` из `helm.infrastructure.rag2.service.build_rag2_context`, кэш на `app.state.rag2_context` (`rag2.py:116-124`).

## 3. Общий ли бэкенд/корпус с MCP — ДА, доказательно
- MCP `analyst_search`/`analyst_context` вызывают ТЕ ЖЕ функции роутера in-process: `rag2_router.search/context(body, _fake_request())` (`analyst_server.py:348-350, 365-367`).
- `_fake_request()` = `SimpleNamespace(app=_require_app())` — отдаёт тот же `app.state` того же FastAPI-процесса (`analyst_server.py:128-133`), значит тот же кэш `rag2_context`, тот же `orchestrator`, тот же Qdrant/корпус.
- Вывод: что видит MCP `analyst_context`, то же увидит веб `/analyst` — один ретрив-бэкенд, один корпус.

## 4. Покрытие тулов через веб `/analyst`
Веб не тул-центричен: страница = 1 чат = 1 эндпоинт. Соответствие:

| MCP-тул | Доступен через браузер `/analyst` |
|---|---|
| analyst_context | ДА (это и есть страница, `POST /api/rag2/context`) |
| analyst_search | Частично/косвенно — контекст-блок строится по тем же хитам, но чистого search-списка на `/analyst` нет (rag2Search в UI не вызывается) |
| related_tasks | НЕТ (обёртка над search с фильтром source=jira; в вебе нет) |
| docs_ask (RAG#1) | НЕТ на `/analyst` (есть отдельная страница `/docs` → DocsChat, другой корпус) |
| **api_registry_check (Р-1)** | **НЕТ** — MCP-only, REST-роута нет |
| **contract_check (Р-4)** | **НЕТ** — MCP-only, REST-роута нет |
| requirement_tests | НЕТ |
| arch_map | НЕТ на `/analyst` (данные C4 есть в control-tower SPA, но это другая страница/эндпоинты) |
| arch_container | НЕТ |
| affected_systems | НЕТ |
| requirement_coverage | НЕТ |
| nearest_spec | НЕТ |
| who_to_involve | НЕТ |
| effort_hint | НЕТ |
| gap_questions | НЕТ |
| constraints | НЕТ |
| contradiction_check | НЕТ |

Всего в MCP 17 тулов (14 структурных + api_registry_check + contract_check + requirement_tests). Через `/analyst`-браузер доступен функционал только `analyst_context`.

## 5. Р-1 / Р-4 — почему только MCP
- `api_registry_check` → `_api_registry(app).check(...)`, реестр грузится из `settings.api_registry_dir`, кэш `app.state.api_registry` (`analyst_server.py:162-170, 411-425`).
- `contract_check` → `_contract_registry(app).check(...)`, из `settings.contract_registry_dir` (`analyst_server.py:173-181, 428-442`).
- Греп по `src/helm/interface/api/` на `ApiRegistry|ContractRegistry|registry.check` — ПУСТО. HTTP-эндпоинта нет ни в одном роутере. → эти тулы вызываются ТОЛЬКО через MCP-транспорт `POST /mcp/analyst`, через браузер не проверить.

## 6. Как проверять через веб
- На `/analyst` есть свободный ввод (`AnalystChat.tsx:316`), отправляется `{query, filters?}` в `/api/rag2/context`. Так можно проверять только RAG-контекст (Jira/Confluence/helm).
- Р-1/Р-4 и прочие структурные тулы — проверяемы ТОЛЬКО через MCP-клиент (streamable-http `POST /mcp/analyst`), не через браузерную страницу.

## Риски/оговорки
- `rag2Search` в api.ts — мёртвый метод в вебе; если понадобится search-режим на `/analyst`, его сейчас нет.
- Control-tower SPA (`index.html`) имеет отдельные дашборды (arch-map, gaps, req-matrix) со своими `/api/cio/*` эндпоинтами — но это НЕ страница `/analyst` и не гарантированно тот же вывод, что MCP-тулы. В рамках вопроса про `/analyst` — не относится.