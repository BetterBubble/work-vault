---
title: explore-helm-инвентарь-слоёв-для-инцидент-среза
type: report
permalink: tacticum/00-inbox/explore-helm-inventar-sloiov-dlia-intsident-sreza
tags:
- helm
- explore
- inventory
- incident-slice
- wave-1a
---

# Инвентарь готового кода Helm по слоям (для вливания инцидент-среза)

Дерево: `/Users/bubblemac/tacticum/helm`, ветка `wave-1a-backend`. Разведка read-only (Serena `helm`). Только фактура.

## 1. domain / ORM (`src/helm/infrastructure/db/models.py`, 734 стр., SQLAlchemy 2.0)

Все ORM-таблицы (класс → `__tablename__` → ключ / важные FK):

**Рефдата:**
- `Product` → product (PK name)
- `Generation` → generation (PK name)
- `Client` → client (PK code, name) — back-compat, мигрирует в Company
- `Service` → service (code, name, product→product.name) — цель Dependency
- `Company` → company (company_id, inn) — поглощает Client (роль client) + employer
- `CompanyRole` → company_role (company_id→company, role) PK(company_id,role) — роли-метки
- `Team` → team (team_id, name, track)
- `Component` → component (component_id, name, product→product.name) — **ось атрибуции инцидентов (Jira-components)**
- `Alias` → alias (generic alias-слой, entity_kind=company|team|component|client|…, status candidate/confirmed/rejected) — очередь-гейт резолва
- `AdpAdoption` → adp_adoption (person_email PK, adoption_date)

**Продукт-воронка (Need→Requirement):**
- `Need` → need (id, title, product→product.name, client→client.code, status) — потребность/проблема
- `Requirement` → requirement (id, need_id→need.id, target_generation→generation.name, hypothesis_*, status) — решение под Need, порождает Initiative
- `Goal` → goal (goal_id, product→, generation→, owner_hint) — кураторская цель ГД

**Идентичность:**
- `Person` → person (person_id, person_name, team, role, grade, manager_email, repos JSON, jira_projects JSON)
- `PersonEmail` → person_email (email PK, person_id→person) — канонический ключ идентичности

**Ядро графа (Signal→Initiative):**
- `Initiative` → initiative (initiative_id, klass, genesis_source, genesis_ref, owner_email→person_email, block_id→block, product→, generation→, is_active_blocker)
- **`Signal` → signal (signal_id, source, external_id, type, ts, severity, entity_refs JSON, text, url, initiative_id→initiative)** — UniqueConstraint(source, external_id). `source` = "jira"|"service"|git/crm/mail/monitoring. **entity_refs — JSON-список строк (не FK).**
- `Dependency` → dependency (from_initiative→initiative, to_kind, to_ref, origin) — to_ref ПОЛИМОРФЕН (initiative/block/service), без единого FK
- `Assignment` → assignment (person_email→, initiative_id→) — человек↔инициатива

**Блоки/продажи:**
- `Block` → block (block_id, eol_email→person_email, product→, generation→) + `BlockTeam` (block_id→, team) — association-таблица блок↔команда
- `SalesInitiative` → sales_initiative (sales_id, title, client[строка], product→, deadline, stage, amount, probability, owner_email→) + `SalesLink` (sales_id→, initiative_id→, kind=decomposes|depends) — association с типом ребра

**Аннотации/снимки:**
- `Annotation` → annotation (override ЕОЛ, append-only)
- `Snapshot` → snapshot (snapshot_id, as_of, payload JSON)

**Витрины дэшей (read-only, идемпотентный ingest):**
- `IngestRun` → ingest_run (source PK, as_of, file_sha256, row_count) — провенанс/скип-по-хешу
- `ConformanceRow` → conformance_row (component, requirement, rn_snapshot, rn_verdict, kmp_verdict, evidence)
- `RepoRow` → repo_row (repo PK, product, generation, commit_count, mr_*, languages, adp_digitized)
- `Commit` → commit ((repo,sha), author_email, jira_keys, effort) — материализация RAG
- `MergeRequest` → merge_request ((repo,mr), state, author)
- `Meeting` → meeting (id="series:date") + `MeetingMinutes` + `MeetingDecision` (поручения ГД, depends_on JSON, override светофор) + `MeetingReport` (LLM-отчёт)

### Характер связей
Связи — **чисто реляционные FK + пара association-таблиц** (block_team, company_role, sales_link). Единственный «полиморфный» узел — `Dependency.to_ref` (initiative/block/service по to_kind) и `Signal.entity_refs` (JSON-список). **Настоящего node/edge графа НЕТ** — это реляционка, где «граф» = таблица `dependency` + FK initiative_id на signal/assignment.

### Сопоставление с 30 сущностями — покрыто ORM
Продукт ✅, Поколение ✅, Клиент ✅ (Client+Company), Сервис ✅, Компонент ✅, Команда ✅, Человек ✅, Signal ✅, Need ✅, Requirement ✅, Initiative ✅, Блок/ЕОЛ ✅, Сделка ✅ (SalesInitiative), Зависимость ✅, Коммит ✅, Репо ✅ (RepoRow), MergeRequest ✅, Meeting ✅, Задача → как Signal (source jira/service).

**НЕ покрыто отдельной таблицей (важно для инцидент-среза):**
- **Обращение/ServiceDesk — НЕТ своей таблицы.** Моделируется как `Signal(source="service", type="incident"|"fr")`. Нет полей SLA/статус-закрытия/время-решения/исполнитель на ORM Signal — только severity + entity_refs(JSON: категория/подкатегория/overdue).
- **Связь обращение→продукт→клиент** реляционно НЕ выражена: у `Signal` нет FK на product/client/component/team. Есть только `initiative_id` (и он у service-сигналов = None по §6.1). Продукт/клиент обращения сейчас лежат в `entity_refs` JSON как строки, не резолвлены.
- Эпик, Репо-как-сущность (есть RepoRow-витрина), Bug (есть domain-датакласс, не ORM).

## 2. ingest/ (`src/helm/ingest/`)

Адаптеры: commits, contract, crm_source, csv_source, economics, eva_source, git_identity, git_source, identity, jira_adapter, links_source, loader, meetings, mr_source, re_enrich, **real_source (оркестратор реальных данных)**, sensitive, **service_source**, synthetic, vitrines.

- **`service_source.py` (ServiceDesk) — ЕСТЬ, рабочий.** Читает `data/real/service/sd_tasks.csv` (~42k), фильтр категорий {IVA, 1Ф}, → `Signal(source="service", type="fr"|"incident", severity=приоритет, entity_refs=(категория,подкатегория,overdue), initiative_id=None)`. Есть тест `tests/ingest/test_service_source.py`. Возвращает **domain** `Signal` (`helm.domain.initiative.Signal`), не ORM напрямую.
- **`crm_source.py` (CRM-сделки) — ЕСТЬ, рабочий.** Читает `data/real/crm/crm_deals.csv` → `SalesInitiative` (stage-маппинг, probability, amount из `сумма_iva_с_ндс`, только открытый пайплайн, продукт нормализуется). Тест есть. owner_email пуст (в выгрузке ФИО, не email).
- `real_source.py` — оркестратор: вызывает `load_crm_sales` (sales) и `load_service_signals` (service_signals) при наличии файлов, собирает `RealDataset`.
- `loader.py` — сборка графа Signal→Initiative (генезис), persist через repository.
- Персист: `repository.py::_upsert_signals` (стр. 452, вызывается `persist_graph` стр.605) — Signal ЛОЖИТСЯ в ORM. Sales → `_upsert_sales`.

**Для инцидент-среза:** загрузчик ServiceDesk и CRM ГОТОВЫ. Нет: резолва product/client/component/team для обращения (сейчас строки в entity_refs), нет привязки severity→SLA, нет времени-решения/статуса на модели.

## 3. application/ и domain/ (грани/расчёты)

`application/`: brief, company_kb, dependency_report, governance, meeting_pack, meeting_report, monitor_gd, portfolio, scoring, snapshot.
`domain/` (много граней тут): access, adp_impact, blocker_graph, calc, conformance, deal_pipeline, effort, gap_detector, genesis, identity_directory, **incident_attribution**, initiative, jira_lite, meetings, monitor, operator_inputs, product_catalog, refdata, repo_registry, salary_queries, sensitive_inputs, **signal_desk**, snapshot_gate, stack, status, tabular.

Релевантно инциденту/сервису/качеству:
- **`domain/signal_desk.py`** — `SignalDesk`: поток обращений, `open_by_category()`, `overdue_by_category()`. Домен-датакласс `Signal(category, status, closed, overdue)` (свой, отдельный от ORM/ingest).
- **`domain/incident_attribution.py`** — `IncidentAttribution` над `Bug`: `by_component()`, `by_release()`, `by_month()`. Источник — Jira bugs_ivaone.csv.
- `domain/deal_pipeline.py` — воронка сделок (клиент×деньги).
- `domain/blocker_graph.py`, `gap_detector.py` — разрывы/блокеры.
- `application/portfolio.py`, `scoring.py`, `monitor_gd.py` — портфель/скоринг.

**Для инцидент-среза:** доменные расчёты `SignalDesk` (open/overdue по категории) и `IncidentAttribution` (по компоненту/релизу/месяцу) ГОТОВЫ как чистые функции. НЕТ: расчётов «обращение→продукт→клиент→команда» (агрегация по этим осям), т.к. осей нет на модели.

## 4. interface/api/routers/

Роутеры: brief, cio, gaps, initiatives, inputs, meetings, portfolio, snapshots. Монтируются в `main.py` (все под `Depends(require_user)`).

**Паттерн роутера (эталон — `cio.py`):**
```python
router = APIRouter(prefix="/api/cio", tags=["cio"])
class XxxOut(BaseModel): ...          # pydantic-схема ответа
@router.get("/repos", response_model=ReposOut)
async def repos(session: AsyncSession = Depends(get_session)) -> ReposOut:
    rows = (await session.execute(select(RepoRow)...)).scalars().all()
    return ReposOut(as_of=await _as_of(session,"repos"), rows=[...])
```
Каждый ответ несёт `as_of` из `ingest_run` (бейдж свежести). Session через `Depends(get_session)` (`interface/api/deps.py`). Чистый SQLAlchemy select. cio.py — самый богатый (conformance, repos, track-activity, contributors, effort, adp-impact).

**ServiceDesk/инцидент-эндпоинтов НЕТ ни в одном роутере** (grep: prefix'ы только cio/gaps/brief/initiatives/meetings/portfolio/inputs/snapshots; ни один файл не содержит signal/incident/service/deal-эндпоинт).

## 5. web/src/screens/

Экраны (живые): Portfolio, Gaps, Gantt, Conformance, Repos, Meetings, TaskHygiene, Brief. + `roles.tsx`, `RoleView.tsx`.

**Регистрация экрана:**
- `roles.tsx` — `ROLES: Record<string, RoleConfig>`, `ROLE_ORDER=[ceo,cpo,coo,cio,cco,hrd]`. Каждая роль → `dashboards: DashDef[]`. `DashDef = {key, label, purpose, status?: "ready"|"wip", el: ReactNode}`. Готовый экран = `el: <Repos/>`; не готовый = `<Stub text/>` + `status:"wip"`.
- `RoleView.tsx` — рендерит subnav по dashboards, строку purpose, бейдж «в разработке» для wip.
- `App.tsx` — топбар с `ROLE_ORDER.map` → `<NavLink to={/key}>`, `<Outlet/>`. Роутинг по роли.
- Паттерн экрана (Repos.tsx) — вызывает `api.*` → таблица.

**Инцидент/сервис-экраны — УЖЕ ЕСТЬ КАК ЗАГЛУШКИ (wip Stub), не живые:**
- COO → `servicedesk` («ServiceDesk & Инциденты», SLA/операционка) — Stub
- CIO → `servicedesk` («Дефекты» по компоненту/релизу) — Stub
- CCO → `pipeline` (Sales pipeline), `commitments` (обязательства/sales-blocked-by-delivery) — Stub
Т.е. **место в IA под инцидент-срез уже зарезервировано**, нужен живой `el` + api + router.

## 6. Точки интеграции
- **Роутер → API:** `main.py` стр.65-72 — `app.include_router(<name>.router, dependencies=_AUTH)` + импорт в блоке стр.22-31. Добавить срез = новый файл роутера + 1 импорт + 1 include_router.
- **Экран → web:** `roles.tsx` — заменить `status:"wip"`/`<Stub/>` у нужного dashboard (напр. COO.servicedesk) на `el:<ServiceDesk/>` + импорт; добавить метод в `api.ts`. RoleView/App трогать не нужно.

## ВЫВОД: минимум «взять готовое / дописать» для инцидент-среза
**Взять готовым:**
- ORM: Signal(source=service), Product, Client/Company, Service, Component, Team, Person — таблицы есть.
- ingest: `service_source.py` (ServiceDesk→Signal) + `crm_source.py` (сделки→SalesInitiative) — рабочие, с тестами; persist через `_upsert_signals`.
- domain: `SignalDesk` (open/overdue by category) + `IncidentAttribution` (by component/release/month) + `deal_pipeline` — чистые расчёты.
- Паттерн роутера (cio.py) и экрана (roles.tsx DashDef + Stub-слоты COO.servicedesk / CIO.servicedesk / CCO.pipeline).

**Дописать:**
1. domain/ORM: оси атрибуции обращения — резолв product/client/component/team вместо строк в `Signal.entity_refs` (FK или association), + поля жизненного цикла обращения (статус закрытия, время-решения/SLA, исполнитель) — их на ORM Signal нет.
2. ingest: обогащение service-сигналов резолвом в канон (Alias-гейт уже есть механизмом) — сейчас категория/подкатегория сырьём.
3. application: агрегаты «обращения × продукт × клиент × команда» (сейчас только by-category и by-component) + связка обращение→инициатива (§6.1 разрыв, initiative_id=None).
4. interface: новый роутер `incidents`/`servicedesk` по образцу cio.py + монтаж в main.py.
5. web: живой экран вместо Stub в COO.servicedesk (и/или CIO.servicedesk) + метод api.

**Развилка для лида:** ORM Signal — «тонкий» (событие + JSON entity_refs). Инцидент-срез (обращение→продукт→клиент→команда реляционно) требует ЛИБО расширить Signal осями/FK, ЛИБО отдельную таблицу Обращение/Ticket. Прототип среза жил в отдельной БД `incidents` (граф-lite, задачи #17-19) — не влит в ORM Helm.
