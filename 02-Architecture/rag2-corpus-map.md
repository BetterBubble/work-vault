---
title: rag2-corpus-map
type: reference
permalink: tacticum/02-architecture/rag2-corpus-map
tags:
- rag2
- corpus
- qdrant
- jira
- data
- helm
- ingest
---

## Карта реального корпуса RAG#2 (Qdrant)

Разведано 2026-07-17. Где реально лежат данные RAG#2 и сколько. Важно: НЕ путать векторный Qdrant с M1-CSV-срезами в `data/real/` (на этом легко ошибиться — см. урок ниже).

## Где данные
- **Qdrant и Meili — НЕ на helm, а на приватном `10.16.0.19`** (Qdrant :6333, Meili :7700), достижимы с прода helm. На самом helm — только app+traefik+postgres.
- Векторный стор наполнялся **полной выгрузкой через adp_emb** (туннель), не M1-скриптами.

## Observations
- [corpus] Qdrant, 6 коллекций (эмбеддинг bge-m3 1024), as_of 2026-07-10: `iva_jira__bge_m3_1024` 319 303 точки, `iva_confluence__bge_m3_1024` 92 374, `knowledge__bge_m3_1024` 80 274, `iva_docs__bge_m3_1024` 8 272, `helm_requirements__bge_m3_1024` 1 465, `helm_mgmt__bge_m3_1024` 400.
- [jira] `iva_jira` = **14 проектов** (points): VCSMOB 88174, IVAONE 87097, VCSWEB2 69601, VCSWEB 48232, IMP 8912, IVASBC 6744, IVAUC2 5473, SCORE 2030, VCSMASH 1796, LRGWEB 456, IVATR 428, IVACS 354, CEO 5, STRAT 1. Точки = чанки (задача+комментарии+changelog), НЕ число задач. Payload-поля: tenant_id, source_doc_id=key, project, status, type, url, as_of, source, text…
- [gap] **IVAONEHALF отсутствует в векторном корпусе** (count=0 по всем 6 коллекциям), хотя заложен в scope (`config.py` `rag2_jira_projects` пример `IVAONE,IVAONEHALF`), в графе (`graph.py` `IVA_PROJECTS`), в `tracks.py` (→ продукт "one"), в `data/vitrines/commits.csv` (53 строки). Вывод: проекта не было в выгрузке-источнике на момент снятия. Не блокер; довыгрузка — `extract_jira_tasks_local.py` (`DEFAULT_JQL = project in (IVAONE, IVAONEHALF)`). golden-кейс a-cross-01 (ждал IVAONE↔IVAONEHALF) → честный corpus-gap negative.
- [scope] Фактический scope Jira-ингеста: env `HELM_RAG2_JIRA_PROJECTS` не задан → дефолт `""` → «все проекты из выгрузки».
- [manifest] `data/real/jira/jira_manifest.csv` **обновлён 2026-07-17** реальными числами из Qdrant (14 проектов, points, as_of 2026-07-10). Старый M1-CSV-срез (≤300/проект, 2026-07-03) → бэкап `jira_manifest.m1-2026-07-03.csv.bak`.

## ⚠️ Урок (достоверность)
`data/real/jira/` содержал **M1-CSV-срез** (≤300 задач/проект, для управленческого графа/мониторов) — он выглядит как «манифест выгрузки», но это НЕ векторный корпус. Реальный корпус (полный, 319k точек) — только в Qdrant. Проверять полноту всегда по Qdrant (facet/count), не по CSV в `data/real`. Перекликается с [[verify-data-credibility]].

## Relations
- relates_to [[iva-contour-access-helm]]
- relates_to [[session-state]]
- part_of [[02-Architecture]]
