---
title: Helm — клиентский слой Этапа 2 (готово в ветке, ждёт данных)
type: note
permalink: tacticum/20-architecture/helm-klientskii-sloi-etapa-2-gotovo-v-vetke-zhdiot-dannykh
tags:
- helm
- conformance
- stage-2
- clients
- pilots
- RequirementClient
- ready-not-deployed
---

# Helm — клиентский слой Этапа 2 (готово в ветке, ждёт данных)

Фоновая Задача 1 (2026-07-08). **Ветка `feature/client-dimension`** (worktree `~/tacticum-worktrees/helm-client-dim`), 4 коммита, **675 тестов зелёные**, НЕ задеплоено. Даёт ось «Клиент» дэшу многомерного соответствия. Связано: [[Helm — Этап 2- развилка источника связи «клиент↔требование»]], [[Helm — Conformance v2- 3-этапный план (многомерная матрица требований 1.0+1.5)]].

## Что построено
- [done] **Модель `RequirementClient`** (`infrastructure/db/models.py`): `requirement_id`↔`client_code` + `pilot_critical:bool` + `provenance:str="заказчик"` + `source_ref`. **PK = `(requirement_id, client_code, provenance)`** (provenance в ключе → источники «заказчик»/«спрос» сосуществуют). `Client.is_pilot:bool` добавлен. Миграция **`a2b3c4d5e6f7`** (down_revision `cf1a2b3c4d5e`, новый head).
- [done] **Refdata-инжест** `ingest/clients.py::load_clients` — upsert `Client(code,name,is_pilot)` из CSV (гибкие заголовки); пилот-флаг из встроенного справочника `PILOT_CLIENTS` (TRN Транснефть · MVD МВД · VEB ВЭБ.РФ · ECH Еврохим · HT-MC Метро · **FORTE Фортеинвест — код провизорный**) + синонимы. `scripts/seed_clients.py`.
- [done] **Загрузчик «Заказчик»** `ingest/requirement_clients.py::load_requirement_clients` — матч требования по `external_ref` → фолбэк по нормализованному title (1.5 на хеше); `RequirementClient(provenance="заказчик")`; **scoped-replace** (полный дамп авторитетен); счётчик `unmatched` (лог, не падает). `scripts/seed_requirement_clients.py --file`.
- [done] **Дэш B3** (`cio.py`+`ConformanceV2.tsx`): фильтр `client` + `client_summary` (свод на клиента: сделано/частично/не сделано/% — как свод по трекам) + `pilot_critical` в ячейках; фронт — чип «Клиент», таблица `ClientSummary`, вид «Требования по пилоту» со столбцом «Критично».

## Дефолты (пометки)
- [gotcha] Код Фортеинвеста `FORTE` — провизорный, уточнить; ЦБ/Мосбиржа кодов пока нет.
- [decision] Пилот-флаг детерминирован справочником (не CSV). Scoped-replace «Заказчик» — по всему provenance. FK проставляется только если резолвится в существующий справочник.

## Чтобы активировать (когда будут данные)
- [todo] Дождаться CSV «Заказчик» от руководителя → сверить с ожидаемыми колонками в docstring `requirement_clients` → при отличии подправить наборы заголовков.
- [todo] На сервере: `alembic upgrade head` (до `a2b3c4d5e6f7`) → `seed_clients.py --file …` → `seed_requirement_clients.py --file …`. `client` в проде сейчас 0 — дэш-ось пуста до заливки. Деплой — bundle→SEED=0, с твоего «ок».