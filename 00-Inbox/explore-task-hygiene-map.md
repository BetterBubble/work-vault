---
title: explore-task-hygiene-map
type: report
permalink: tacticum/00-inbox/explore-task-hygiene-map
tags:
- helm
- task-manager
- explore
- srez-7
---

# Explore: Task Management / PMO (срез №7, TaskHygiene) — карта по вертикали

Разведка репо `/Users/bubblemac/tacticum/helm`. Только чтение. Serena = Python-only (LSP), фронт читан Read.

## 1. Фронт
- `web/src/screens/TaskHygiene.tsx:1-124` — статичный explainer, 6 `<section className="panel">` (Проблема/Состояние источников/Что готово/Предложение/Этапы/Метрики). НЕТ `api`, `useAsync`, KPI, таблиц. Голая заглушка.
- Слот: `web/src/screens/roles.tsx:141-146` — роль **coo**, key `"hygiene"`, `status:"wip"`, `el:<TaskHygiene/>`. Импорт стр.8.
- Живой образец `web/src/screens/Pipeline.tsx` (253 стр): `useAsync<PipelineOut>(()=>api.pipeline(filters),[deps])` (стр.65); паттерн «opts из нефильтрованного среза» (67-74); `<Loader state>{(data)=>...}` (102); `<div className="kpis"><Kpi n= label= to=/>` (108-117); `<Section id title right>` + `<table className="init">` + `overflowX:auto`; `<MoreToggle>` для Top-N (191-201). Серверные фильтры через query.
- `web/src/screens/Portfolio.tsx` — фильтры-объект `useState<PortfolioFilters>`, `upd(patch)` чистит пустые, `qs()`.
- UI из `web/src/components.tsx`: `Kpi`(91, tone red|yellow|green, to=якорь), `Section`(132), `Loader`(73), `MoreToggle`(155), `StatusChip`/`Dot`(47-58), `Spinner`/`ErrorBox`. `lib.ts`: `useAsync`, `fmtDate`, `fmtMoney`, `daysUntil`, `klassLabel`, `STATUS_ORDER`.
- `web/src/api.ts` — тонкий клиент `api.*`, `request<T>(path)`, `qs(filters)` (113). Типы в `types.ts`. НЕТ `api.taskHygiene`/`tasks` — писать.

## 2. Роутеры-образцы
- Регистрация: **`src/helm/main.py:22` (импорт роутеров), `:73-96` (`app.include_router(x.router, dependencies=_AUTH)`)**. НЕ отдельный роутер-файл. Новый роутер добавить в импорт+include.
- `routers/gaps.py` (26 стр) — эталон read-витрины: `router=APIRouter(prefix="/api", tags=["gaps"])`; `@router.get("/gaps", response_model=GapReportOut)`; `Depends(get_session)`, `today: date = Depends(current_today)`; `portfolio = await build_current_portfolio(session, today)`; `return gap_report_out(portfolio.gaps)`.
- `routers/portfolio.py` — `filter_: PortfolioFilter = Depends(portfolio_filter)` (серверные фильтры), `build_current_portfolio(session, today, filter_)`.
- Хелперы: `helm.interface.api._common.build_current_portfolio`, `deps.py`: `get_session`, `current_today`, `portfolio_filter`.
- Out-схемы: `interface/api/schemas.py` — `GapReportOut`(115) + `gap_report_out`(123); `PortfolioOut`(as_of + blocks). Портфель несёт `as_of`.

## 3. Домен/application
- `domain/gap_detector.py` — dataclasses `Task(key, epic_link="")`, `Epic(key)`; `GapDetector(tasks, epics)`: `.work_without_goal()->list[Task]` (epic_link пуст), `.goal_coverage()->float`, `.goals_without_work()->list[Epic]`. Это IN-MEMORY, поверх `CompanyKnowledgeBase.from_data_dir` (data-dir CSV), НЕ БД.
- `domain/calc.py` — БД-путь: `work_without_goal(signals)->list[str]` (282; `return [s.external_id for s in signals if s.initiative_id is None]`); `urgent_important_without_owner(initiatives, sales, today)->list[str]` (324); `goals_without_work`, `promises_without_work`, `work_without_money`, `traffic_light`, `is_urgent`, `importance`.
- `application/portfolio.py:70 GapReport` (dataclass, 5 списков) — то, что отдаёт `/api/gaps`.
- `application/company_kb.py CompanyKnowledgeBase` — фасад: методы `work_without_goal/goals_without_work/goal_coverage`, `top_blockers`, `open_signals_by_category`, `.from_data_dir()`. Работает по data-dir, не по live-БД.
- `domain/blocker_graph.py BlockerGraph(blockers).top_blockers()/for_project()`; `Blocker` dataclass.
- `domain/genesis.py`, `domain/initiative.py` — генезис Initiative из эпиков/целей/сделок.

## 4. Модель данных (`infrastructure/db/models.py`)
- **Initiative (423)**: `initiative_id`(pk), `title`, `klass`, `genesis_source`, `genesis_ref`(=goal_id/sales_id/epic-ключ), `owner_email?`, `block_id?`, `product?`, `generation?`, `lead_time_days?`, `plan_finish_override?`, `status_override?`, `origin`(structural/inferred), `is_active_blocker`. Дедлайн/plan_finish НЕ хранятся — считаются в calc.py.
- **Signal (455)** = задача (Jira): `signal_id`(pk), `source`("jira"), `external_id`(ключ задачи), `type`(=issue_type), `ts`, `severity`(=priority), `entity_refs: JSON` (список), `text`(=summary), `url?`, `initiative_id?` (FK, **None = orphan/работа без цели**). **НЕТ колонок status / assignee / epic_link / parent** — они схлопнуты в `entity_refs` (см. ниже).
- **Dependency (475)**: `from_initiative`→`to_kind`(initiative/block/service)+`to_ref` (полиморф). Это граф зависимостей **на уровне Initiative**, НЕ задача↔задача.
- **Assignment (494)**: `person_email`→`initiative_id` (+`allocation` пусто в 1a, `origin`). Уровень Initiative, НЕ задача.
- **Epic как ORM-таблицы НЕТ** — эпик становится Initiative (`genesis_source=jira_epic`, `genesis_ref=epic-ключ`).

## 5. Ingest задач (crux)
- Контракт `ingest/contract.py`: `RawJiraIssue(key, status, assignee_email, epic_key, parent, labels, issue_type, story_points, created, due, priority, project, summary)` — богатое сырьё. `RawEpic(key,title,project,product,generation,assignee_email)`. `initiative_id_for(gsource,gref)="gsource:gref"`.
- `ingest/loader.py`:
  - `_structural_target(issue, ref_to_id)` (246): эпик → иначе parent → иначе первый label, из `ref_to_id`. Нет → None.
  - `_resolve_issue_target` (258): структурно, иначе LLM-судья (`TaskGoalJudge`) → origin="inferred", иначе `(None,"structural")` = разрыв.
  - `_build_signals_and_assignments` (297): **`entity_refs = filter(not-None, (assignee_email, epic_key, parent))`** — сохранены, но позиционно (если assignee=None, epic сдвигается на индекс 0 → парсить нельзя надёжно). Assignment создаётся ТОЛЬКО если `target is not None and assignee`. → orphan-задачи БЕЗ Assignment.
  - `OPEN_JIRA_STATUSES={To Do,In Progress,In Review}` (54) **определён, но нигде не используется** (grep по src = только определение) → фильтра по статусу нет, Signal.status не хранится.
- `ingest/jira_adapter.py` (212 стр), `ingest/links_source.py` (141 стр — грузит Dependency-рёбра). **`eva_source.py` в git ОТСУТСТВУЕТ** (подтверждено; untracked на сервере). Есть `crm_source`, `git_source`, `real_source`, `synthetic`.

## Выводы для метрик
- **orphan-rate** ✅ считается сейчас: `Signal.source='jira' & initiative_id IS NULL` / все jira-Signal. Уже отдаётся как `gaps.work_without_goal` (external_id).
- **покрытие epic_link** ⚠️ косвенно: структурная привязка (initiative_id≠None по эпику/parent/label). Отдельного «есть epic vs нет» из БД чисто нет — epic сидит в `entity_refs` позиционно. Дораб: либо парсить entity_refs, либо добавить колонки.
- **блокеры** ⚠️ только Initiative-уровень: `Initiative.is_active_blocker` + `Dependency`. Задача↔задача блокеров нет.
- **разрез по assignee** ⚠️ только по Initiative (`Assignment`, `owner_email`). По задаче — только для привязанных (entity_refs[assignee]), для orphan ненадёжно.
- **Jira↔EVA расхождение** ❌ EVA не грузится (нет eva_source), общего ключа нет.

## Риски
- Задача=Signal без status/assignee/epic-колонок → иерархия epic/parent и статус теряются (в entity_refs позиционно, парсинг хрупкий).
- Два трекера без общего ключа; EVA отсутствует в коде.
- `CompanyKnowledgeBase`/`gap_detector` работают по data-dir CSV, live-путь — `calc.py`+`build_current_portfolio`. Не перепутать.
- Signal хранит ВСЕ статусы (нет фильтра OPEN) → знаменатель orphan-rate включает закрытые задачи, если так грузили.


---

## Акцент 1 — ЗАДЕЛ под будущую полную выгрузку Jira

Принцип: метрики считать динамически из данных, чтобы приход более полной Jira улучшил цифры/связи сам, без переделки. Задача = первокласс (не только плоский Signal), но прод-версия работает уже на текущих данных (поля nullable, тонко заполнены сейчас, богаче потом).

Готовые слоты, куда новый экспорт «втечёт» без изменения витрины:
- **Иерархия задача→эпик→parent**: контракт `RawJiraIssue.epic_key/parent` УЖЕ есть (`ingest/contract.py`). `loader._structural_target` (246) резолвит epic→parent→label→`ref_to_id`. Сейчас epic_key заполнен тонко → мало structural-связок. Полная выгрузка → больше epic_key → больше `Signal.initiative_id≠None` → goal_coverage/orphan-rate улучшаются САМИ. Слот готов. Дораб: чтобы отдать «есть эпик да/нет» и иерархию явно — вынести epic_key/parent из позиционного `entity_refs` в колонки (или Task-таблицу).
- **Граф зависимостей (блокеры)**: `ingest/links_source.py` — `RawIssueLink`(Blocks) + `ActiveBlocker` из `jira_issue_links.csv`/`active_blockers.csv` → резолв в `Dependency` + `Initiative.is_active_blocker` в loader. Полная выгрузка issue-links → больше рёбер Dependency → метрика блокеров богаче сама. Слот готов, но уровень Initiative; задача↔задача блокеры требуют Task-first-class.
- **assignee**: `RawJiraIssue.assignee_email` есть; сейчас → `entity_refs` (позиционно) + `Assignment`(только если задача привязана). Полная выгрузка — то же поле. Для разреза по assignee НА УРОВНЕ ЗАДАЧИ (вкл. orphan) нужна колонка assignee на Signal/Task.
- **fixVersion / release**: ❌ В `RawJiraIssue` поля НЕТ — это НОВЫЙ слот, добавить в контракт заранее (optional), чтобы релизный разрез включился с полной выгрузкой без переделки витрины.
- **status**: `RawJiraIssue.status` есть, но НЕ доходит до Signal (нет колонки; `OPEN_JIRA_STATUSES` определён и не используется). Задел: колонка status на Signal/Task + фильтр открытых → знаменатель orphan-rate станет честным.

Рекомендация дизайна: ввести Task как первокласс (колонки status/assignee/epic_key/parent/fix_version, source=jira|eva), заполнять из текущего сырья тонко, метрики строить SQL-агрегатами по этим колонкам. Тогда полная Jira = просто больше заполненных строк, витрина и формулы неизменны.

## Акцент 2 — ДОСТОВЕРНОСТЬ (сверка на БД)

СТАТУС: прямой доступ к dev-prod БД (сервер `helm`, postgres в docker, helm/helm/helm) ЗАБЛОКИРОВАН политикой Production-Reads — `ssh_execute` и `ssh_db_query` отклонены авто-классификатором (цель названа тиммейтом, не пользователем). Нужна авторизация пользователя ИЛИ лид гоняет SQL и присылает вывод — интерпретирую.

Готовый verification-SQL (SELECT-only, безопасно):
```sql
SELECT
  (SELECT count(*) FROM signal WHERE source='jira') AS jira_tasks,
  (SELECT count(*) FROM signal WHERE source='jira' AND initiative_id IS NOT NULL) AS linked,
  (SELECT count(*) FROM signal WHERE source='jira' AND initiative_id IS NULL) AS orphans,
  (SELECT count(*) FROM initiative) AS initiatives,
  (SELECT count(*) FROM assignment) AS assignments,
  (SELECT count(*) FROM dependency) AS deps,
  (SELECT count(*) FROM initiative WHERE is_active_blocker) AS active_blockers;
-- orphan_rate = orphans / jira_tasks
-- epic-покрытие (косвенно) = linked / jira_tasks
-- разрез по источнику: GROUP BY source; по типу: GROUP BY type
-- по «владельцу» задачи: entity_refs позиционно — надёжно только через Assignment/Initiative.owner_email
```

НЕПРОВЕРЕННЫЕ оценки (из кода + данных лида signal=21911/initiative=601/assignment=203 + фронта «~92% без epic_link», «покрытие 7–29%») — пометить как гипотезы до прогона:
- orphan-rate, вероятно, ВЫСОКИЙ (значит linked мало): фронт заявляет ~92% задач без epic_link. Но 601 инициатива и 203 assignment на 21911 сигналов → грубо ≤~1–3% сигналов имеют assignment; сколько имеют initiative_id — надо считать (эпик/parent/label + LLM-inferred). Это и есть главная цифра — БЕЗ прогона не утверждаю.
- epic-покрытие сейчас ЗАНИЖЕНО дырявым epic_link (7–29%) → orphan-rate ЗАВЫШЕН. С полной Jira epic_link вырастет → orphan-rate упадёт до честного. Метрику подавать как «нижняя граница качества привязки на текущих данных», с бейджем свежести/полноты источника.
- блокеры: `dependency` и `is_active_blocker` считаются из links-выгрузки; фактический count — только прогоном (реальность блокеров = сколько active_blockers резолвятся в инициативы).

Правило достоверности: на витрине показывать ЗНАМЕНАТЕЛЬ и «полноту источника», не голый %, т.к. на дырявом epic_link % вводит в заблуждение.
