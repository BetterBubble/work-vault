---
title: 'Helm — внедрение срезов в код: вертикаль по слоям, Ticket отдельной таблицей'
type: decision
permalink: tacticum/21-decisions/helm-vnedrenie-srezov-v-kod-vertikal-po-sloiam-ticket-otdelnoi-tablitsei-1
tags:
- helm
- control-tower
- slice
- architecture
- ingest
- ticket
- decision
- phase-1
---

# Helm — внедрение срезов в код: вертикаль по слоям, Ticket отдельной таблицей

Зафиксировано 2026-07-07. Как вливать вертикальные срезы в готовый код Helm (после merge origin/main). Основано на разведке code-scout (`00-Board/explore-helm-инвентарь-слоёв-для-инцидент-среза`).

## Принцип
- [decision] **Срез = вертикаль через существующие слои Helm** (ingest→domain→application→interface→web), переиспользуя готовое; писать с нуля только пробелы. Прототип Фазы 0 (граф-lite Postgres + отдельный контейнер) = чертёж, раскладывается по слоям; отдельная БД `incidents` НЕ вливается — данные едут в основной ORM.
- [reference] **Рецепт для любого среза:** (1) domain — сущность есть в ORM→берём, нет→таблица+миграция; (2) ingest — адаптер есть→берём, резолв строк в FK; (3) application — грани переиспустить+дополнить; (4) interface — роутер по образцу `cio.py`; (5) web — заменить готовый WIP-слот на живой экран; (6) сверка с baseline прототипа.

## Что готово в коде (переиспустить)
- [reference] **ingest:** `service_source.py` (ServiceDesk sd_tasks→Signal, с тестом), `crm_source.py` (сделки→SalesInitiative, с тестом), `real_source` оркестрирует. **domain-грани:** `SignalDesk` (open/overdue), `IncidentAttribution` (by_component/release), `deal_pipeline` (клиент×деньги). **domain-модели:** Client/Company/Service/Component/Team/Product/SalesInitiative/Signal (~19/30 сущностей). **router-эталон** `cio.py`. **web:** WIP-слоты COO→ServiceDesk, CIO→Дефекты, CCO→Pipeline (roles.tsx).
- [gotcha] Модель Helm **чисто реляционная** (FK+3 association: block_team/company_role/sales_link), настоящего node/edge графа нет. Signal «тонкий»: связи обращения лежат строками в `entity_refs`, НЕ резолвлены в FK.

## Решение по обращению
- [decision] **Обращение ServiceDesk = отдельная таблица `Ticket`** (решение оператора 2026-07-07), НЕ расширение Signal. FK product/client/component/assignee + поля type/status/priority/tariff/created/age. Signal остаётся тонким событием-сырьём (канон §3). Совпадает с нашим ER (Обращение — отдельная сущность) и прототипом.
- [decision] **Источник Ticket = `crm_open_requests.csv`** (782, богатый: продукт/клиент/тариф/исполнитель), НЕ `sd_tasks` (42k журнал-метаданные без содержания, из техдолга).

## План инцидент-среза (первый, Фаза 1)
- [plan] domain: таблица Ticket + миграция. ingest: загрузчик Ticket из crm_open_requests с резолвом FK. application: агрегаты Ticket×продукт/клиент/команда/тариф/приоритет/возраст + timeline + темы. interface: `routers/incidents.py` по cio.py + монтаж main.py под auth. web: живой `ServiceDesk.tsx` (COO-слот) + api.ts. Сверка с `data/incident-phase0-baseline.json`.

- relates_to [[Helm — стратегия построения: вертикальный срез, не горизонталь; ручной прогон = проект пайплайна]]
- relates_to [[2026-07-06 — Helm: инцидент-срез Фаза 0 развёрнут на реальном стеке (изолированно)]]
- relates_to [[Helm — согласованный вердикт по сущностной модели (30 сущностей)]]