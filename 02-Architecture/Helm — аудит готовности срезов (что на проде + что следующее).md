---
title: Helm — аудит готовности срезов (что на проде + что следующее)
type: note
permalink: tacticum/02-architecture/helm-audit-gotovnosti-srezov-chto-na-prode-chto-sleduiushchee
tags:
- helm
- control-tower
- slices
- audit
- readiness
- roadmap
- ranking
---

# Helm — аудит готовности срезов (2026-07-08)

Задача 3 фонового прогона. Read-only снимок прода + сверка с [[Helm — Карта срезов (7 вью дашборда)]]. Ранжирование «какой срез следующий дешевле/ценнее».

## Что загружено на проде (факт, `ingest_run` + row-counts)
- [reference] Инженерный: commit 231358 · merge_request 61656 · repo_row 233 · commit_effort 102111 · assignment 203. Люди: person/person_email 1009. Коммерция: sales_initiative 316 · client_deal_money 267 · company/company_role 333 · product 22 · product_economics 8. Управление: initiative **601** · goal 15 · block 9 · dependency 37 · meeting_decision 30. Поддержка: sd_request 43059 · signal 21911 · sd_theme 12. Требования: requirement 1046 (1.0=519+1.5=527) · requirement_assessment 2254 · requirement_component 991 · conformance_row 170 · component 43 · generation 6.
- [gotcha] **Пусто/не загружено:** `need` 0 · `client` 0 · sensitive (CV/зарплаты 🔴) не залиты · bugs_ivaone (дефекты) не залиты.

## Готовность 7 срезов (данные × код)
- [done] **Conformance/Соответствие (CIO/CPO)** — Этап 1 ЖИВОЙ на проде (многомерная матрица + свод). Требования+оценки загружены.
- [done] **Портфель/Монитор ГД (CEO/COO)** — ЖИВОЙ; `initiative` 601 засеян, все слои ✓ (portfolio/gantt/brief/gaps + Portfolio.tsx). Остаток: подключить orphan `Brief.tsx` к маршруту, sales→delivery мост (опц.).
- [done] **Инциденты/SLA ServiceDesk (COO)** — Фаза 1 в проде (sd_request 43k · signal 21k · SignalDesk · company_kb). Остаток: SLA-возраст/контракт из support-выгрузки (⚙).
- [todo] **Бэклог/Потребности (CPO)** — данные спроса есть (FR в sd_request + сделки), `need` пуст. **Задача 2 фонового прогона построила механизм вычленения `Need`** (ветка `feature/need-ingest`). Остаток: прогнать Need на сервере + операторский отсмотр → роутер `/api/needs` + экран Backlog/Needs. **Самый разблокированный НОВЫЙ срез.**
- [todo] **Sales/Pipeline (CCO)** — данные+домен готовы (316 сделок, DealPipeline, экономика); ✗ read-роутер `/api/pipeline` + ✗ web (Stub). Средне.
- [todo] **Task Management/Гигиена (PMO)** — gap-машинерия готова (work_without_goal), ✗ orphan-rate агрегат + ✗ `/api/task-hygiene` + web WIP. Средне.
- [todo] **Ценность людей (COO/HR)** — `scoring.py` (квадранты/fit) готов, но ✗ роутер + ✗ web + **🔴 CV/зарплаты не загружены** + приватность-гейт. Дороже, data-blocked → отложить.

## Ранжирование «что следующее»
- [decision] 1) **Бэклог/Потребности (CPO)** — Задача 2 сняла самый тяжёлый кусок (вычленение Need); ценность×готовность максимальны, вяжется с клиентским спросом и Этапом 2.
- [decision] 2) **Sales/Pipeline (CCO)** — данные+домен готовы, писать роутер+web.
- [decision] 3) **Task Management (PMO)** — gap-машинерия готова, писать orphan-rate+роутер+web.
- [decision] 4) **Ценность людей** — отложить (🔴 sensitive не залиты + больше писать).
- [decision] Полировка: подключить `Brief.tsx`; для «Дефектов» CIO — залить bugs_ivaone.

## Relations
- relates_to [[Helm — Карта срезов (7 вью дашборда)]]
- relates_to [[Helm — Conformance v2- 3-этапный план (многомерная матрица требований 1.0+1.5)]]