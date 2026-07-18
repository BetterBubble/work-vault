---
title: rag2-r5-rebuild-result
type: report
permalink: tacticum/04-sessions/rag2-r5-rebuild-result
tags:
- helm
- rag2
- effort_hint
- changelog
- r5
- result
---

# Р-5: результат пересборки effort_hint-changelog (2026-07-17)

Реализация плана [[rag2-r5-rebuild-and-datamap]] (раздел A). Фон: [[rag2-ab-measurements]] (Р-5 ПЕРЕСМОТР). Read-only по Qdrant, запись ТОЛЬКО в `epic_task.changelog`.

## Что сделано
- Реконструирован полный **timestamped** changelog из Qdrant `iva_jira__bge_m3_1024` (scroll 319303 чанков, склейка по `key`+`chunk_idx` через "\n", regex статус-переходов, дедуп по (ts,from,to)).
- Записан в `epic_task.changelog` **хирургическим UPDATE** (только эта колонка) — НЕ через reingest, чтобы не задвоить снапшот (один as_of=2026-07-10, `_search_epic_by_terms` не фильтрует as_of) и не потерять 4 задачи, которых нет в `tasks_rich` (IVAONE-1213/1214/1215/3999). Все 8073 строки целы, 0 потерь.
- Скрипты (worktree `feat/r5-changelog-recon`): `scripts/rebuild_changelog_from_qdrant.py` (реконструкция, stdlib), `smoke_r5_durations.py`, `checkpoint_before_after.py`, `gen_changelog_update_sql.py`.

## Охват (до → после)
- non-empty changelog в `epic_task`: **377 → 6676** задач (из 8073).
- Реконструкция дала переходы для **6471** задач (7104 имели чанки в iva_jira; +205 старых date-only остались нетронутыми у ключей без реконструкции → 6676 итог).
- На реальном запросе покрытие среди совпавших по термам: **1306/1507 (87%)** для «push-уведомления».

## Медианы ДО → ПОСЛЕ (одинаковый ILIKE-набор, логика 1:1 с effort_hint; проверено на текущей БД)
- «push-уведомления в чате» (16/20 покрыто): active_days null → **24.8 дн**; lead_time null → **138.7 дн**.
- «экспорт списка пользователей» (19/20): active_days 0.0 → **23.8 дн**; lead_time null → **151.7 дн**.
- Эталоны (реальные `_status_durations`): IVAONE-1175→20.9, IVAONE-3803→7.3, IVAONE-1→0.0 — совпали.
- Глобально по реконструкции: active_days median 1.1 (p75 6.0, max 930), lead_time median 28.5 (n=5476).

## Найденные дефекты effort_hint (код, вне scope Р-5 — на воркер code-fixes, задача #13)
1. **mixed-tz TypeError.** `created` (rich `cr`) = date-only naive, `closed_at` из changelog = tz-aware → `closed_ts - created_ts` падает. Обошёл на уровне данных: реконструированный `d` пишется БЕЗ tz-оффсета (все переходы ИВА в +0300 → active_days неизменны, lead_time корректный naive−naive). Правильный фикс — привести обе метки к одному виду в коде.
2. **`LIMIT 20` без `ORDER BY` в `_search_epic_by_terms`.** После MVCC-UPDATE покрытые строки ушли в конец heap, непокрытые — в начале; seq-scan LIMIT 20 выхватывает непокрытые → живой effort_hint отдаёт null НЕСМОТРЯ на 87% покрытие. Доказано: тот же WHERE `LIMIT 20` → 0/20 покрыто; `ORDER BY key LIMIT 20` → 16/20 покрыто (дало бы 24.8/138.7). Фикс — добавить ORDER BY и/или предпочтение `changelog<>''`.

## Не покрыто
- **IVAONEHALF (~450 задач)** — их нет в `iva_jira`, реконструкция невозможна; отдельный трек выгрузки из Jira (задачи #15/#14).
- 4 задачи вне `tasks_rich` (см. выше) — сохранены как были (пустой/старый changelog).

## Артефакты / откат
- Бэкап: `/opt/helm/data/backup_epic_task_2026-07-17.sql` (pg_dump -t epic_task, 8.3MB, все 8073 строки).
- SQL записи: `/tmp/iva/r5_changelog_update.sql` (6471 UPDATE, dollar-quoting), реконструкция: `/tmp/iva/tasks_changelog_v2/chlog_v2.jsonl`.
- Откат: restore из бэкапа (TRUNCATE epic_task + psql < backup).

## Вердикт
Данные пересобраны и верифицированы (запись легла точно: текущая БД на фикс-наборах даёт эталонные медианы). effort_hint считает численные медианы для покрытых similar-задач. Полное «работает из коробки» в живом туле наступит после код-фикса #2 (ORDER BY) — иначе LIMIT-выборка под-сэмплит покрытые задачи.
