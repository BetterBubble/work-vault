---
title: rag2-r5-rebuild-and-datamap
type: report
permalink: tacticum/02-architecture/rag2-r5-rebuild-and-datamap
tags:
- helm
- rag2
- effort_hint
- changelog
- datamap
- cleanup
---

# RAG#2 Р-5: пересборка effort_hint-changelog + карта данных

Разведка explorer (read-only, 2026-07-17). Ничего не менялось/не удалялось. Связано: [[rag2-corpus-map]], [[rag2-ab-measurements]].

## TL;DR
- **Р-5 реализуемо: ДА.** Реконструкция полного timestamped changelog из Qdrant `iva_jira` доказана на 3 задачах (IVAONE-1175 → active_days=20.9; IVAONE-3803 → 7.3; IVAONE-1 → 0.0, открыта). Jira PAT и новый extract НЕ нужны.
- **Root-cause null-медиан подтверждён БД:** `epic_task` = 8073 строки, но changelog есть только у **377** (старый velocity-дамп на 379), и он **date-only** (`{f,t,d:"2023-10-16"}` без времени). Два дефекта сразу: (1) покрытие 377/8073 → similar-search почти всегда бьёт по задачам с пустым changelog → active_days=None → median=None; (2) даже у 377 внутридневные длительности потеряны.
- Карта данных: **~9 групп** помечены на будущую чистку (устаревшие дампы/бэкапы/tmp). Рабочие корпуса (6 Qdrant-коллекций, Meili, `data/real/*`, `epic_task`, vitrines-current, `tasks_rich`, `iva_jira` как источник) — источник-истины, НЕ трогать.

---

## Раздел A — план пересборки Р-5 (пошагово для implementer)

### A0. Как это устроено (проверенные факты)
- **effort_hint читает `EpicTask.changelog` из SQL (Postgres), НЕ из Qdrant helm_mgmt.** Путь: `effort_hint` (`analyst_server.py:943`) → similar-задачи по `EpicTask.summary ILIKE` → `_status_durations(_normalize_epic_changelog(t.changelog))`.
- `_normalize_epic_changelog` (`analyst_server.py:736`) ждёт КОМПАКТНЫЙ формат `{f,t,d}` → маппит в `{field:"status", from, to, created:d}`.
- `_status_durations` (`analyst_server.py:681`) суммирует секунды в интервалах, где `to`-статус попадает в корзину `development` (`velocity.status_bucket`: In Progress/Ready for Development/Merge request/To release). `_parse_ts` (`:659`) корректно ест полный ISO с `+0300`. Нет валидных статус-переходов → `active_days=None` (не 0.0).
- **Ингест EpicTask:** `scripts/ingest_epic_tasks.py` → `_read_dirs()` мёржит `*.jsonl` по ключу `k` из каталогов `data/iva/tasks_rich` + `data/iva/tasks_changelog`; короткий ключ `chlog`→`changelog`. `load_epic_tasks` (`epic_tasks.py:51`) делает replace по `as_of`, пишет ТОЛЬКО таблицу `epic_task`.
- **Источник истории — Qdrant `iva_jira__bge_m3_1024` (319303 чанка).** Переходы лежат ПЛОСКО в `payload.text`: `[2024-03-22T09:52:22.672+0300] status: In Progress → In Review`. Чанки одной задачи связаны `payload.key` (== `source_doc_id`), порядок — `payload.chunk_idx`. Перекрытие соседних чанков — константа **150 символов**.

### A1. Реконструкция (проверено, воспроизводимо)
Скрипт (запускать НА сервере helm — Qdrant `10.16.0.19:6333` виден только оттуда):
1. scroll `iva_jira__bge_m3_1024` c фильтром `key in <ключи из tasks_rich>` (8073 задачи IVAONE+IVAONEHALF), группировать по `key`, сортировать по `chunk_idx`.
2. склеить `text` через `"\n"` (НЕ пустой — иначе строки на границе чанка сливаются в мусор вида `14:55:11.02:44.218`).
3. regex `\[([0-9T:+.\-]+)\]\s*status:\s*(.+?)\s*[→>]\s*([^\n\[]+)` → **дедуп по (ts,from,to)** (перекрытие 150 симв. даёт дубли).
4. на задачу писать JSONL-строку `{"k": key, "chlog": [{"f":from,"t":to,"d":ts_full_iso}, ...]}` — `d` = ПОЛНЫЙ таймстемп (не дата). `_parse_ts` его переварит.
5. Складывать в НОВЫЙ каталог `data/iva/tasks_changelog_v2/` (шарды `chlog_*.jsonl`), старый `tasks_changelog` не трогать.

Проверка `_status_durations` на реконструкции (реальный прогон): IVAONE-1175 → 21 переход, active_days=**20.9**, closed_at=2024-04-16; IVAONE-3803 → 22, **7.3**, closed_at=2025-02-24; IVAONE-1 → 3, **0.0** (открыта, честный ноль). Осмысленно.

### A2. Наполнение effort_hint-корпуса (что запускает implementer)
```
# на сервере helm, из /opt/helm (или локально с последующей доставкой tasks_changelog_v2)
uv run --no-sync python scripts/ingest_epic_tasks.py \
  --dir data/iva/tasks_rich,data/iva/tasks_changelog_v2 --as-of <YYYY-MM-DD>
```
- Пишет ТОЛЬКО таблицу `epic_task` (replace по as_of). **Qdrant `iva_jira` и все RAG-корпуса НЕ трогаются** — guardrail соблюдён.
- Старый `data/iva/tasks_changelog` в `--dir` НЕ передавать (иначе date-only перезатрёт по мёржу — last-non-empty wins).
- После ингеста прогнать effort_hint на 2-3 запросах → медианы должны стать не-null.

### A3. Замечания/риски
- `EpicTask.changelog` потребляется ТОЛЬКО `_normalize_epic_changelog` (effort_hint) — полный таймстемп в `d` безопасен, других читателей нет. `velocity.py`/VelocityStat — отдельный ингест, не затрагивается.
- ~44 IVAONE-чанка имеют `has_changelog=false` — у этих задач истории не будет (ожидаемо).
- Скрипт реконструкции — НОВЫЙ (в репо нет). Не путать с `scripts/extract_jira_changelog.py` (тот тянет из Jira API, требует PAT — здесь НЕ используем).

---

## Раздел B — карта данных с вердиктами

Легенда: 🟢 источник-истины (НЕ трогать) · 🟡 устаревший/ложный путь · 🔴 мусор-tmp. Чистка ОТЛОЖЕНА — только описание.

### Qdrant (10.16.0.19:6333) — все 🟢, дублей/устаревших НЕТ
| Коллекция | points | Вердикт |
|---|---|---|
| iva_jira__bge_m3_1024 | 319303 | 🟢 RAG#2 + источник реконструкции Р-5 |
| iva_confluence__bge_m3_1024 | 92374 | 🟢 RAG#2 |
| knowledge__bge_m3_1024 | 80274 | 🟢 RAG#1/knowledge |
| iva_docs__bge_m3_1024 | 8272 | 🟢 публичные доки (docs_ask) |
| helm_requirements__bge_m3_1024 | 1465 | 🟢 реестр требований |
| helm_mgmt__bge_m3_1024 | 400 | 🟢 витрина «Управление задачами» (RAG-срез; effort_hint фактически на SQL) |

Meili-индексы — 🟢 (не инспектировал глубоко по указанию, рабочие). adp_emb — не трогал (SSH-транзит, Jira-дампов нет).

### Локально `~/tacticum/helm/data/` (всего 636M)
| Путь | Что | Размер | Вердикт |
|---|---|---|---|
| data/real/* | боевой корпус-исходник (git/jira/confluence/eva/…) | 607M | 🟢 источник ингеста |
| data/iva/tasks_rich/ | 8119 rich-карточек (IVAONE 7669 + IVAONEHALF 450), формат `k/type/st/...` | 9.2M | 🟢 источник EpicTask |
| data/iva/tasks_changelog/ | 377 задач, changelog **date-only** `{f,t,d}` | 200K | 🟡 будет superseded после Р-5 (tasks_changelog_v2). До Р-5 — активный источник, НЕ удалять сейчас |
| data/iva/*.json (epics, req_realization, signal_active_salvage, wiki_reqs_*, …) | промежуточные витрины/выгрузки | ~1.8M | 🟢 держать (входы ингеста/дашборда) — точечно проверить перед любой чисткой |
| data/iva/deploy_bundle.tgz | архив-снапшот доставки | 1.6M | 🟡 артефакт разовой доставки — кандидат на удаление (проверить, что распакован/не нужен) |
| data/iva/task_snapshot_bundle.tgz | архив-снапшот задач | 1.6M | 🟡 артефакт-снапшот — кандидат |
| data/iva/tasks/ | пустой каталог (0B) | 0B | 🔴 пустышка (старый дефолт `--dir`); безопасно удалить |
| data/_superseded/ (jira-loose, repos-csv-only, repos-shallow) | явно устаревшее по имени | 4.2M | 🟡 superseded — кандидат (подтвердить, что нет уникальных данных vs data/real) |
| data/_wiki_extract/ | pages_index.csv/spaces.csv выгрузка вики | 9.0M | 🟡 разовая выгрузка — вероятно superseded RAG-корпусом confluence; проверить актуальность |
| data/example/ | демо-CSV (teams/goals/…) | 64K | 🟢 фикстуры-примеры (держать) |
| data/*.rtf (Созвон…) | транскрипты созвонов | 264K | 🟢 держать (или перенести в память) |
| разбросанные .DS_Store | macOS-мусор | — | 🔴 удалить |

### Прод `/opt/helm/data/` (396M) + `/tmp` (963M)
| Путь | Что | Размер | Вердикт |
|---|---|---|---|
| /opt/helm/data/real/* | боевой корпус | 238M | 🟢 источник-истины |
| /opt/helm/data/vitrines/*.csv (commit_effort, commits, conformance…) | текущие витрины дашборда | ~55M | 🟢 актуальные |
| /opt/helm/data/vitrines/*.bak, *.bak-0708/0709/0709b | старые бэкапы витрин (commit_effort ×3, commits ×2, conformance) | ~100M | 🟡 устаревшие бэкапы — кандидат (подтвердить, что дашборд читает не-.bak) |
| /opt/helm/data/_staging/rag2_pilot | стейджинг RAG#2 | 8.5M | 🟡 проверить, нужен ли ещё пилот-стейджинг |
| /opt/helm/data/{confluence,requirements,repos,reports,roadmap.csv,jira_*.csv} | входы/выходы витрин | ~1.5M | 🟢 держать |
| /tmp/*.py (check_*, ceo_extract*, build_scope, backfill_*) | 49 dev-скретч скриптов | — | 🔴 tmp-мусор — кандидат |
| /tmp/*.log (deploy*.log, base_k{1,3,5,10}.log, eval_driver, helm-build) | логи прогонов | — | 🔴 tmp-мусор — кандидат |
| /tmp (всего) | скретч | 963M | 🔴 разово почистить (подтвердить, что нет активного пайплайна) |
| /opt/helm/.env.bak.1784126288, .env.deploybak | бэкапы .env (СЕКРЕТЫ!) | ~1K | 🟡 удалить ОСТОРОЖНО (содержат секреты; убедиться, что текущий .env рабочий, затем стереть) |
| /opt/helm/_llmt2.log | пустой лог (0B) | 0B | 🔴 удалить |

### Итог по чистке
**~9 групп кандидатов** (ничего не удалено): (1) data/_superseded/, (2) data/iva/*.tgz ×2, (3) data/iva/tasks/ пустой, (4) data/_wiki_extract/, (5) .DS_Store, (6) prod vitrines *.bak*, (7) prod /tmp (.py+.log), (8) prod .env-бэкапы (осторожно), (9) prod _llmt2.log. Условный 10-й: data/iva/tasks_changelog/ — только ПОСЛЕ Р-5.

**Перед любым удалением подтвердить:** нет уникальных данных в `_superseded`/`_wiki_extract` vs `data/real` и Qdrant; дашборд читает не-.bak витрины; `/tmp` не вход активного пайплайна; текущий `.env` рабочий.

---

## Аудит полноты Р-1/Р-4 (read-only, 2026-07-17)

Задача: убедиться, что «зелёный» реестр = ПОЛНЫЙ корпус, а не только приёмочные кейсы. Независимый пересчёт источник vs реестр (не полагаясь на готовые validate_*.py — те ссылаются на worktree-пути).

### Р-1 API — полнота ПОДТВЕРЖДЕНА, потерь нет
Парсер `api_openapi.parse_openapi` считает операцию = path × HTTP-метод (`_HTTP_METHODS` = все 8: get/put/post/delete/patch/head/options/trace), без пропуска deprecated, без дедупа → реестр обязан совпасть с источником один-в-один.

| Реестр | paths (источник) | ops источник (paths×methods) | manifest | operations.json (факт на проде) | Вердикт |
|---|---|---|---|---|---|
| clients | 315 | 342 | 342 | 342 | ✅ |
| integration | 54 | 72 | 72 | 72 | ✅ |
| bot | 4 | 5 | 5 | 5 | ✅ |
| **ИТОГО** | 373 | **419** | **419** | **419** | ✅ полностью |

- Источник (локальные спеки в scratchpad `api_specs/*.json`) = манифест = фактические `operations.json` — **419**, полное совпадение. Потерь/дублей/схлопываний нет; все операции прошли `parse_openapi`.
- Примечание: заявленные «410» — устаревшее число; текущие спеки (clients v2.30.0, integration/bot v1.30.0) дают 419. Реестр полнее заявленного, не беднее.

### Р-4 JUMP — полнота ПОДТВЕРЖДЕНА, потерь нет
Парсер `jump_contracts.parse_sessions_html`: команда = h3-секция, чей заголовок проходит `_is_command_name` (`^[A-Za-z][A-Za-z0-9]*$` — одиночный алфанумерик-токен), дедуп по имени (первая встреча).

- Sessions.html: всего `<h3>` = **105** → прошли фильтр = **101** → уникальных = **101** = `JUMP.commands.json` (101, дублей 0). ✅
- 4 отброшенных h3 — заведомо НЕ команды (проза/кириллица): «Логин», «Message MIME part retrieving using HTTP request», «Uploading files to the JUMP session file storage using HTTP request», «Задачи VTODO». Корректно исключены, команд среди них нет.
- Дедуп ничего не схлопнул (0 дублей). Строчные-однословные имена (`close`, `ping`, …) — валидные команды JUMP, не ложные срабатывания.

### Итог аудита
**Полнота обоих реестров подтверждена — потерь 0.** Р-1: 419/419 (источник=манифест=факт). Р-4: 101/101 (105 h3 − 4 прозы). «Зелёный» отражает полный корпус операций/команд, а не только приёмочные кейсы. Приёмочные проверки матчинга (validate_r1/r4.py) — отдельный вопрос качества матчера, не полноты; здесь не гонялись (retrieval на живом сервисе не трогал, guardrail соблюдён).
