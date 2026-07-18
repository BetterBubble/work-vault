---
title: Граф Signal→Initiative vs реляционные витрины — read-path на SQL
type: decision
permalink: tacticum/03-decisions/graf-signal-initiative-vs-reliatsionnye-vitriny-read-path-na-sql
tags:
- helm
- architecture
- graph
- data-model
- read-path
- decision
- task-manager
---

В Helm намеренно сосуществуют ДВА подхода к данным. Понимание этого — ключ к тому, как строить/масштабировать срезы. Зафиксировано 2026-07-09 (при проектировании «идеального» таск-менеджера).

## Два подхода к данным

### A. Реляционный, запросно-ориентированный
Дэши: **Соответствие требованиям** (`routers/cio.py`), ServiceDesk, Бэклог/Потребности, Pipeline, HRD.
- **Модель = матрица-снежинка**: `Requirement` в центре + **event-sourced** факт `RequirementAssessment` (датированные строки на грань `требование×компонент×трек×источник×as_of`, `models.py:1141`) + оси-справочники (`Component`/`Client`/`ClientBundle`) через FK/M:N.
- **Ингест = идемпотентный UPSERT по строкам** (`on_conflict_do_update`) + scoped-replace по грани/provenance, sha-skip (`ingest/requirements.py`). Влил модель — она лежит.
- **Витрина = прямые SQL select+join НА КАЖДЫЙ запрос** (`cio.py:337 conformance_v2`); «текущий вердикт» = `max(as_of)` в SQL (`_state_at cio.py:1024`). Граф НЕ грузится.
- Масштабируется (БД считает group by/фильтры), даёт серверные фильтры, event-sourced историю.

### B. Графовый, вычисляемый в памяти
Дэши: **Монитор ГД / Портфель / Гантт** (`portfolio.py`), **Разрывы §6.1** (`gaps.py`), **Таск-менеджер** (`tasks.py`), брифы/снимки. Ядро Волны 1a (канон `control-tower-v02.md §3`).
- **Модель = граф `Signal → Initiative`** (`domain/initiative.py`, frozen dataclasses): задача=Signal агрегируется на Initiative; Initiative рождается из генезиса (цель ∪ продажа ∪ Jira-эпик, `domain/genesis.py`); зависимости, назначения.
- **Ингест = `build_graph` собирает ВЕСЬ граф** из jira/eva/git/crm/sd (`ingest/loader.py:536`) → `clear_graph`+`persist_graph` (полная перезаливка граф-таблиц, `repository.py:1122 _CLEAR_ORDER`, `:1138 clear_graph`).
- **Витрина = `load_graph` грузит ВЕСЬ граф в память** (`repository.py:1173`, select без фильтров) → чистые Python-функции (`build_portfolio`, `build_task_hygiene`) считают на КАЖДЫЙ запрос, без кэша (`_common.py:13 build_current_portfolio`).

## Зачем нужен граф
- [insight] Граф — это модель «работа ↔ цель ↔ клиент ↔ деньги». Даёт то, чего плоские таблицы не дают: генезис инициатив (откуда работа), агрегацию задач→инициатива (**orphan = разрыв**), зависимости/блокеры, дерево Блок→ЕОЛ, сшивку jira+git+crm+sd+eva в одну модель.
- [insight] Главная ценность — **разрывы §6.1** (работа без цели / цель без работы / продажа без разработки): видеть ПОРВАННЫЕ СВЯЗИ между таблицами, а не таблицы по отдельности. Реляционная витрина отвечает «что сделано» (факт); граф — «связана ли работа с целью» (связь).

## Решение (read-path) — 2026-07-09
- [decision] **Граф как МОДЕЛЬ остаётся** (генезис/связи незаменимы). Проблема была только в РЕАЛИЗАЦИИ чтения: `load_graph` грузит весь граф в память на каждый запрос — не тянется на росте (уже 46k+ сигналов после EVA/git; полная Jira ещё увеличит).
- [decision] Граф **ХРАНИТСЯ реляционно** (FK: `signal.initiative_id`, `initiative.block_id`, `dependency.from_initiative/to_ref`; колонки задачи `signal.status/assignee_email/epic_key/project` — уже добавлены). → **Read-path перевести на SQL** (как `cio.py`): метрики = SQL-агрегации по `signal`/`initiative`/`dependency`, цепочка/блокеры = FK-джойны, серверные фильтры, снапшоты для тренда (как `hrd_people_snapshot`). `build_graph` (генезис) остаётся ingest-моделью.
- **Итог одной фразой: граф не исчезает — меняется способ его чтения** (весь-в-память → таргетный SQL). Консистентно с двумя зрелыми срезами (conformance + HRD), а не третий паттерн.

## Реализация (план «идеального таск-менеджера», спроектирован 2026-07-09)
5 фаз, переиспользуют существующее: Ф0 read-path на SQL (образец `cio.py`); Ф1 копилот-гейт (orphan→`TaskGoalJudge`→оператор через HITL `review.py`+`operator_review`); Ф2 квадранты «объём×покрытие» + разрез по команде (identity)/продукту (`links_source.load_jira_project_product`+`_EVA_PRODUCT`); Ф3 тренд (snapshot read-model как `refresh_hrd_snapshot.py`); Ф4 Jira↔EVA расхождение. Инвариант: единый источник расчёта для live-эндпоинта и снапшота; ingest не затирает операторские привязки.

## Отношения
- relates_to [[control-tower-v02]]
- relates_to [[Helm — Карта срезов (7 вью дашборда)]]
- relates_to [[Helm — сущностная ER-модель (30 сущностей)]]
- relates_to [[Helm — Conformance v2- 3-этапный план (многомерная матрица требований 1.0+1.5)]]
