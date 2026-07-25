---
title: Helm — wave-1b data-ingest отложен (не влит в main)
type: decision
permalink: tacticum/21-decisions/helm-wave-1b-data-ingest-otlozhen-ne-vlit-v-main
tags:
- helm
- control-tower
- wave-1b
- data-ingest
- deferred
- decision
---

## Решение (2026-07-07)

- [decision] **wave-1b data-ingest НЕ вливаем в main сейчас.** Точка правды — **сервер**; сервер = Фаза 1 (ServiceDesk), wave-1b на нём нет и текущему срезу не нужен.
- [context] wave-1b = ingest для БОЛЬШОГО графа (не для инцидент-среза): `eva_source` (EVA-задачи), `git_source`+`load_all_commits` (216k коммитов), `mr_source` (MR), repo→product (233 репо), сшивка в `real_source`. Нужен будущим срезам: **Task Management (PMO), Ценность людей, Дефекты/CIO** — им нужны задачи/коммиты/MR/repo. Инциденту (ServiceDesk) — нет.
- [reference] Работа сохранена: ветки `origin/wave-1a-backend` (старый PR, закрыть как устаревший, НЕ удалять) и локальная `feature/wave1b-data-ingest` (HEAD `38fb96e`). Реальная поверхность конфликтов с main мала — 3 файла (`ingest/loader.py`, `re_enrich.py`, `real_source.py`); остальное — новые файлы.
- [gotcha] Деплой wave-1b на сервер: **миграций НЕ добавляет** (схему не трогает), ServiceDesk не ломает (он на `sd_request`, изолирован). НО меняет графовые витрины (Портфель/Gaps/Задачи) непроверенными данными и требует залить **552М git-данных** (`data/real/git`, runbook их намеренно не льёт) → тяжёлый seed, риск падения.
- [plan] **Пере-интегрировать свежо от main + проверить на сервере**, когда возьмёмся за срез, которому wave-1b нужен (задачи/люди/дефекты). Не раньше.

- relates_to [[Helm — техдолг по данным (чего не хватает в data, по срезам)]]
