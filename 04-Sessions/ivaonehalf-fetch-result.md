---
title: ivaonehalf-fetch-result
type: report
permalink: tacticum/04-sessions/ivaonehalf-fetch-result
tags:
- helm
- rag2
- ivaonehalf
- ingest
- corpus-gap
- blocked
---

# IVAONEHALF fetch/prep — пайплайн готов, ждёт Jira-креды (2026-07-17)

Воркер `ivaonehalf-fetch`, ночь 2026-07-17. Фаза 3 ночного плана. **В прод НИЧЕГО не писал** (Qdrant/epic_task не тронуты, guardrail соблюдён). Связано: [[rag2-ivaonehalf-gap]], [[rag2-r5-rebuild-and-datamap]], [[session-state]].

## TL;DR
Подготовка доингеста IVAONEHALF завершена и валидирована на прод-коде (паритет с 87k IVAONE доказан на синтетике). **Единственный блокер — Jira-креды**, которых на helm нет. Пользователь даст утром → запуск «одной командой». Решение лида: **Вариант B (iva-mcp) НЕ использовать** — нормализованный json ≠ сырой REST, риск тихой порчи паритета; кривой корпус хуже отсутствия.

## Что установлено (факты)
- Qdrant `iva_jira__bge_m3_1024` (10.16.0.19, только с helm): IVAONEHALF=**0**, IVAONE=**87097**. Живой Jira `project=IVAONEHALF` total=**577** (вырос с 574→577 за день).
- **Паритет 87k IVAONE**: точки строит `rag2/ingest.py:build_document` (+`iter_points`) из **tasks_rich-формата**, НЕ из `JiraRecord`. Прод-путь записи: `rag2_index.ingest_jira`→`jira.ingest`→`iter_points`. Параметры (сняты с живой точки + кода):
  - `tenant_id="iva"`, `jira_base_url="https://jira.iva.ru"`, `char_split(900,150)`, uid=`chunk_uid("iva",key,idx)`=`uuid5(NS_b2c3d4e5,"iva:{key}:{idx}")`, vector 1024/Cosine (BGE-M3).
  - payload-ключи: tenant_id, source_doc_id, key, title, project, epic, component, status, status_cat, assignee, assignee_login, reporter, type, priority, sprint, fix_versions, labels, parent, created, updated, url, as_of, source, attachments, has_comments, has_changelog, text, chunk_idx.
  - Конвенции локализованы: status="Закрыт"/"Ready for Test", type="Ошибка"/"Задача", priority="Высокий" — совпадают с нормализацией. `parent` = строка `"None"` (квирк jira_issue_to_record — воспроизводится, паритет точный).
  - `chunk_idx` у старых точек глобальный (4088) — артефакт старой версии; актуальный `iter_points` даёт per-task idx. uid по разным ключам с IVAONE не коллизирует → upsert аддитивен, IVAONE не тронется.
- **Маппер**: прод `jira_issue_to_record` ждёт **СЫРОЙ Jira REST** (`fields=*all&expand=changelog`). iva-mcp `jira_search` отдаёт НОРМАЛИЗОВАННЫЙ json (плоский, `status.category` текстом, `changelogs[].items[]` snake_case `from_string/to_string`) → прод-маппер его НЕ ест. Отсюда и решение против B.
- **Креды**: `fetch_jira_issues` — Basic-auth (`HELM_RAG2_JIRA_USER/PASSWORD`, группа browse-allprojects). На helm ОТСУТСТВУЮТ (нет в контейнере `helm-helm-1` env, нет в `/opt/helm/.env`) — подавались разово при исходном фетче. Туннель **8443**→jira.iva.ru жив.

## Пайплайн (готов, на helm `/root/ivaonehalf_prep/`)
- `fetch_raw.py` — фетч сырого REST на ХОСТЕ (host python3 без deps; curl `--resolve jira.iva.ru:8443:127.0.0.1` через туннель; креды из env, в контекст модели не попадают). Батчи 50, пагинация по total, ретраи. → `raw_issues.jsonl`.
- `prepare_items.py` — в контейнере `helm-helm-1` (PYTHONPATH=/app/src, прод deps). Прод-паритет 1-в-1: `jira_issue_to_record→validate→should_index(QualityConfig дефолт)→dedup_jira_records→build_document→char_split(900,150)→chunk_uid`. **БЕЗ вектора, БЕЗ записи.** Артефакты: `tasks_rich.jsonl` (вход для записи), `items_prepared.jsonl` ({id,payload} по чанкам), `changelog.jsonl` (`{k,chlog:[{f,t,d}]}` статус-переходы, d=полный ISO-таймстемп — для epic_task/Р-5).
- `prepare_items_ivamcp.py` — fallback-адаптер под нормализованный iva-mcp (НЕ использовать по решению лида; лежит на всякий).
- `run.sh` — оркестратор «одной командой».
- **Валидация**: смоук-тест prepare_items.py в контейнере на синтетической сырой issue — payload 1-в-1 с IVAONE (совпал квирк parent:"None"), changelog `[{f,t,d}]` с полными таймстемпами, chunk_idx per-task, uuid5 корректный. Синтаксис всех скриптов + прод-импорты зелёные.

## Команда запуска (утром, когда будут креды)
```bash
# на хосте helm (ssh-manager server=helm), Basic-auth:
HELM_RAG2_JIRA_USER='<user>' HELM_RAG2_JIRA_PASSWORD='<pass>' bash /root/ivaonehalf_prep/run.sh
# или PAT:
HELM_RAG2_JIRA_PAT='<pat>' bash /root/ivaonehalf_prep/run.sh
```
Если TLS-verify упадёт на туннеле (self-signed) — добавить `-k` в curl внутри `fetch_raw.py` (сейчас verify включён; прод использовал verify=True с SNI, т.е. cert валиден для jira.iva.ru → -k скорее не нужен).

## Ожидаемый объём
- Задач: **577** (минус отсев `should_index`: `🔴` в summary, метки∩{deprecated,draft}, `len(sum)+len(desc)<40` — у KMP-багов маловероятно; ожидаемо ~570+).
- Точек: **~1500** (оценка ~2.75 чанка/задача — текст = summary+desc+комменты+changelog+links+вложения; changelog у активных багов даёт основной объём). Точная цифра — из `counts.json` после реального прогона.
- С changelog: большинство (у KMP-багов есть импортные статус-переходы Open→Ready for Development→…→Closed с таймстемпами).

## Осталось (утром, после кредов)
1. Дать креды → `bash run.sh` → снять реальные counts.
2. **Запись (координированно с лидом, отдельный сигнал):** `tasks_rich.jsonl` → `rag2_index.ingest_jira` scope IVAONEHALF (эмбеддинг+аддитивный upsert в iva_jira + Meili) — тот же прод-код. Verify: IVAONEHALF findable, **IVAONE count == 87097 не изменился**, точечный search.
3. `changelog.jsonl` → UPDATE `epic_task.changelog` для IVAONEHALF-ключей (как Р-5 для IVAONE) → effort_hint по IVAONEHALF.
4. Обновить `jira_manifest.csv` (снять пометку «IVAONEHALF отсутствует»).

Блок того же класса, что [[rag2-ivaonehalf-gap]] Eva-wiki — нужны креды человека.
