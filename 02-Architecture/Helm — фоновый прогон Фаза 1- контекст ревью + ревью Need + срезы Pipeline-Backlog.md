---
title: 'Helm — фоновый прогон Фаза 1: контекст ревью + ревью Need + срезы Pipeline/Backlog'
type: note
permalink: tacticum/02-architecture/helm-fonovyi-progon-faza-1-kontekst-reviu-reviu-need-srezy-pipeline-backlog
tags:
- helm
- control-tower
- review
- slices
- pipeline
- backlog
- ready-not-deployed
- batch
---

# Helm — фоновый прогон Фаза 1 (2026-07-08)

4 фоновые задачи, все зелёные, PR-ready, **НЕ задеплоено**. Миграций нет ни в одной ветке; merge-tree по всем парам = 0 конфликтов → мержатся в любом порядке, деплой alembic no-op. Связано: [[Helm — операторская консоль ревью (Настройки как HITL)- дизайн]], [[Helm — аудит готовности срезов (что на проде + что следующее)]], ADR-0002.

## Ветки
- [done] **`feature/review-rollout`** (включает `feature/review-context`, 4 коммита): контекст/провенанс на карточках ревью (бредкрамб «откуда ячейка» + компонент·поколение·тип, `no_assessment` смягчён) + **раскатка ревью на Потребности (Need)** — очередь кандидатов → promote(→active)/edit/reject через generic `operator_review` + новый реестр `DECISION_APPLIERS`; клиенты — заготовка (сигнала «на ревью» в данных нет, критерий TBD). 745 passed.
- [done] **`feature/slice-pipeline`** (1 коммит): срез **Sales/Pipeline (CCO)** — роутер `/api/pipeline` (KPI/воронка/маржа/сделки из `sales_initiative`+`product_economics`, `DealPipeline`) + `Pipeline.tsx`, слот CCO. Данные реальные (316 сделок).
- [done] **`feature/slice-backlog`** (2 коммита): срез **Бэклог/Потребности (CPO)** — роутер `/api/backlog` (читает `Need`, сорт по весу спроса, разбивка по продукту) + `Backlog.tsx`, слот CPO. Инертен до прогона Need.

## Ключевые дефолты (помечены)
- [gotcha] Backlog/ревью-Need читают вес спроса/confidence **парсингом из `Need.description`** (инжест кладёт сводку туда, отдельных колонок нет). Кандидат на улучшение: вынести метрики в колонки `Need` миграцией.
- [decision] Клиенты-ревью — заготовка: нужен признак `match_method` (фаззи-title vs точный external_ref) у загрузчика `requirement_clients`, тогда критерий «на ревью» заработает.
- [decision] Need-ревью: promote→status=active + аудит в `operator_review`; reject→status=rejected (расширил статус-множество); edit правит title/description.

## Что осталось
- [todo] Смержить пачкой (порядок свободен, конфликтов/миграций нет) → **один аккуратный деплой** (bundle → `SEED=0`, before=after).
- [todo] Наполнение: прогнать `Need` на сервере (оживит Backlog + очередь ревью-Need); дождаться CSV «Заказчик» (клиентская ось + критерий клиентов-ревью).

## Relations
- relates_to [[Helm — операторская консоль ревью (Настройки как HITL)- дизайн]]
- relates_to [[Helm — аудит готовности срезов (что на проде + что следующее)]]