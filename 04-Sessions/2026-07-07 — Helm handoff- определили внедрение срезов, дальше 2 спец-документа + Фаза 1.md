---
title: '2026-07-07 — Helm handoff: определили внедрение срезов, дальше 2 спец-документа
  + Фаза 1'
type: note
permalink: tacticum/04-sessions/2026-07-07-helm-handoff-opredelili-vnedrenie-srezov-dalshe-2-spets-dokumenta-faza-1
tags:
- helm
- control-tower
- handoff
- session
- slice
- phase-1
---

# 2026-07-07 — Helm handoff: определили внедрение срезов, дальше 2 спец-документа + Фаза 1

Чекпойнт для передачи в новую сессию (контекст переполнен).

## Где мы сейчас
- [state] Определили **как внедрять вертикальные срезы в готовый код Helm**: вертикаль по слоям (ingest→domain→application→interface→web), переиспользуя готовое. Разведка code-scout завершена, инвентарь получен и учтён.
- [decision] **Обращение = отдельная таблица `Ticket`** (не раздуваем Signal), источник — **`crm_open_requests`** (не sd_tasks). Подробности → [[Helm — внедрение срезов в код: вертикаль по слоям, Ticket отдельной таблицей]].
- [state] Придуманы **два вспомогательных документа** (Карта срезов + Реестр данных по сущностям) → [[Helm — два вспомогательных документа для срезов: Карта срезов + Реестр данных по сущностям]]. Согласовано, что делаем их первыми — окупаются на всех срезах.

## Следующие шаги (порядок)
- [todo] 1. **Собрать 2 документа** `data/slices-map.md` + `data/data-registry.md` (реестр наполнить реальными колонками из data-файлов; карту — по 7 вью). Основа: аудит data/real (память), `concept-entity-provenance.md`, `concept-entities.md`.
- [todo] 2. **Фаза 1 инцидент-среза**: таблица Ticket + alembic → загрузчик из crm_open_requests с резолвом FK → агрегаты → `routers/incidents.py` по образцу cio.py → живой экран ServiceDesk (COO-слот) → сверка с `data/incident-phase0-baseline.json` (числа = прототип).
- [todo] 3. (висит, не срочно) **PR Шага 0** wave-1a-backend → main — чтобы Фаза 1 стартовала от смерженного main. Пуш только по явной команде оператора, показать diff до пуша.

## Ключевые артефакты
- [reference] Прототип Фазы 0 живой: https://helm.tacticum.ru/incidents (изолированно, БД incidents, боевое не тронуто). Файлы прототипа на сервере `/opt/helm/_incidents` + `_front`.
- [reference] Репо-материалы: `data/concept-entities.md` (ER 30 сущностей/44 связи), `data/concept-entity-provenance.md` (вердикт), `data/slice-build-phases.md` (4 фазы), `data/incident-phase0-baseline.json` (эталон сверки), `data/incident-phase0-summary.md`, `data/incident-phase0-vs-phase1.md`, `data/needs-first-pass.md`, `data/incidents-first-pass.md`.
- [reference] Черновик разведки: `00-Inbox/explore-helm-инвентарь-слоёв-для-инцидент-среза`.

## Инструменты для новой сессии
- [note] Serena активировать на проекте helm. Тиммейты-агенты: explorer/implementer/verifier + Agent Team + Workflow (fan-out). Память basic-memory (vault tacticum) — прочитать 03-Decisions/04-Sessions/02-Architecture по helm.

- relates_to [[Helm — внедрение срезов в код: вертикаль по слоям, Ticket отдельной таблицей]]
- relates_to [[Helm — два вспомогательных документа для срезов: Карта срезов + Реестр данных по сущностям]]