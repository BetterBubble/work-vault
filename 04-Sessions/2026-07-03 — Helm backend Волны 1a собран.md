---
title: 2026-07-03 — Helm backend Волны 1a собран
type: report
permalink: tacticum/04-sessions/2026-07-03-helm-backend-volny-1a-sobran
tags:
- helm
- control-tower
- wave-1a
- backend
- session
---

Собран **backend Волны 1a (Монитор ГД)** продукта Helm/Control Tower. Только backend (фронт-дэш пишет руководитель отдельно), по канону [[control-tower-v02]] §3–5. Репозиторий `github.com/TacticumApps/helm`, работа локально в `~/tacticum/helm`, ветка `wave-1a-backend`, 10 локальных коммитов, **НЕ запушено** (по договорённости — копим локально, пуш/PR отдельной командой оператора).

## Что готово

- [layer] Домен (чистый stdlib, frozen dataclasses, mypy strict): Signal→Initiative, Block, SalesInitiative, Dependency, Assignment; генезис из трёх источников (цель ∪ продажа ∪ Jira-эпик) + ручной merge #domain
- [layer] Расчёты §4/§5.2/§6.1: дедлайн=min по обязательствам, план-финиш=дедлайн−lead_time, светофор+override ЕОЛ, срочно×важно, разрывы графа #calc
- [layer] Слой забора изолирован: единый входной контракт, генератор синтетики, заглушка-адаптер Jira, идемпотентный loader #ingest
- [layer] application: сборка портфеля (роллап в блоки), снимок+diff (§5.3), недельный бриф (детерминированный, без LLM-прозы) + markdown-экспорт #application
- [layer] Персистентность: async SQLAlchemy 2.0 + Alembic + Postgres (docker-compose), репозитории, CRUD, снимки #persistence
- [layer] API: 15 JSON-эндпоинтов FastAPI (/api/portfolio, /gantt, /brief, /gaps, /snapshot(+diff), CRUD sales/goals/blocks/teams, /annotations) #api
- [health] 134 теста зелёные, mypy strict + ruff чисто, end-to-end проверено на живом Postgres #verified

## Ключевые решения

- [decision] Стек снят с эталонного репо `agents`: FastAPI · Pydantic v2 · async SQLAlchemy 2.0 · Alembic · asyncpg · uv #stack
- [decision] Оператор осознанно выбрал async SQLAlchemy 2.0 + Postgres в docker-compose (после разбора, что async для батча избыточен) #stack
- [decision] Эволюционировал существовавший `Commitment` в `SalesInitiative` (§3), а не завёл две сущности #model
- [decision] LLM замокан (интерфейс LeadTimeEstimator + детерминированный мок), реальный Gateway — в готовую точку, вызовы не жгутся #llm
- [decision] input-schemas.md/apps-helm.md недоступны (нет на remote/диске/вики) — новые контракты реконструированы из канона §3/§7/§13 #assumption
- [decision] Разрывы графа §6.1 = сигнал-семантика (работа = наличие Jira-сигнала, не факт Initiative) — иначе всегда пусты, т.к. loader мнёт Initiative на каждую цель/продажу #calc

## Как работать с реальными данными (когда придут)

- [howto] Меняется ТОЛЬКО адаптер `ingest/jira_adapter.py` (тот же выходной контракт, что генератор) — остальной код не трогается #ingest
- [howto] Ручные входы оператор ведёт через CRUD-API или CSV-импорт (шаблоны в `templates/`) #api
- [data] Ждём для 1a: Jira (ключ/статус/ассайни/эпик/лейблы/тип/SP/даты/приоритет/проект) + ручные входы (цели, блоки+ЕОЛ, состав команд, продажные инициативы, справочники product/generation) #data
- [data] Для 1a НЕ нужно: git/код, CRM-детали, штатка/зарплаты/CV (красный контур), карта идентичности, RE-обогащение — это Волна 1b (поля-заделы уже заложены) #data
- [blocker] Блоки+ЕОЛ — кураторская нарезка CEO/CPO; без неё дерево ответственности пустое (§12.0) #open

## Заделы 1b (заложены полями, не заполняются)

- [note] allocation на Assignment, провенанс origin (structural/inferred), слой идентичности person↔person_email (1:N) #wave-1b

## Отношения

- implements [[control-tower-v02]]
- part_of [[Tacticum Helm]]
