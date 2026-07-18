---
title: rag2-eva-prod-audit-SUMMARY
type: report
permalink: tacticum/00-inbox/rag2-eva-prod-audit-summary
tags:
- rag2
- eva
- prod-audit
- helm
- read-only
---

# RAG#2 / Ева — инвентарь прод-сервера helm (read-only аудит)

Дата аудита: 2026-07-17. Сервер `helm` (159.194.233.33), доступ через MCP ssh-manager. Строго read-only, секреты не читались.

## Топология: где что живёт

- **На самом хосте helm — только 3 контейнера**: `helm-helm-1` (app, :8000), `helm-traefik-1` (:80/:443), `helm-postgres-1` (:5432). **Qdrant и Meilisearch на этом сервере НЕ развёрнуты.**
- **Векторный стор RAG#2 вынесен на приватный хост `10.16.0.19`**: Qdrant `http://10.16.0.19:6333`, Meili `http://10.16.0.19:7700` (из env приложения `HELM_QDRANT_URL` / `HELM_MEILI_URL`). Оба достижимы с прода.
- LLM-шлюз: `HELM_GATEWAY_BASE_URL=https://llm.cifragen.ru/v1`. Rerank RAG#2 включён (`HELM_RAG2_RERANK_ENABLED=1`).

## Qdrant — фактические коллекции и объёмы (10.16.0.19:6333)

Все `status: green`. Замерено 2026-07-17:

- `iva_jira__bge_m3_1024` — **319 303** точек (Jira, tenant iva)
- `iva_confluence__bge_m3_1024` — **92 374** (Confluence, tenant iva)
- `knowledge__bge_m3_1024` — **80 274** (substrate: код/CV/доки, `knowledge`)
- `iva_docs__bge_m3_1024` — **8 272** (публичная документация ИВА, tenant iva)
- `helm_requirements__bge_m3_1024` — **1 465** (реестр требований)
- `helm_mgmt__bge_m3_1024` — **400** (управленческий контур: инициативы/эпики/SD, tenant helm, `as_of=2026-07-13`, fail-closed изоляция)

**Отдельной коллекции eva / contract / jump / api в Qdrant НЕТ.** Найдено 6 корпусов (рекон ожидал 3 — реально их вдвое больше: добавились iva_docs, knowledge, helm_requirements).

## Meilisearch (10.16.0.19:7700)

- Жив (`/health` = 200), требует Bearer-ключ. Индексы (из `config.py`): `iva_jira`, `iva_confluence`, `iva_docs`.
- **Число документов по индексам НЕ снято**: запрос с ключом заблокирован гардрейлом на чтение секрета. → **нужен доступ: значение `HELM_MEILI_KEY` либо read-only Meili-ключ**, чтобы снять `numberOfDocuments`.

## Данные Евы на диске: `/opt/helm/data/real/eva/` (свежесть — 2026-07-04)

Три CSV — это выгрузка **EVA-трекера (eva.iva.ru)**:
- `eva_tasks.csv` — 2.5M, **6 634 задачи**
- `eva_task_links.csv` — 80K, **2 332 связи** (depends_on / affects)
- `eva_projects.csv` — **25 проектов**, с колонками `total` / `exported`

**Полнота EVA-трекера на уровне выгрузки — практически 100%**: у всех непустых проектов `exported == total` (iva-messendzher-2.0 2507/2507, largo 1840/1840, ms/Provisioning 616/616, rm-integraczii 486/486, new-desktop-qt 455/455, IVA ONE 152/152 и т.д.). Единственный пробел — проект `iva-marketing` (1 задача, exported=0). Пустые проекты (iva-id, wiki, quality-management и пр.) имеют total=0.

## Как Ева попадает в систему (архитектурно, из исходников)

- `eva_source.py`: читает `data/real/eva/`, подключается в `real_source` как **ВТОРОЙ task-источник РЯДОМ с Jira** (не вместо), сигналы несут `source="eva"`. Питает **управленческий граф** (loader → инициативы/эпики/зависимости), а НЕ векторный RAG#2 напрямую.
- Векторно EVA видна только опосредованно — через корпус `helm_mgmt__bge_m3_1024` (400 точек, агрегированные инициативы/эпики/SD), не как 6634 отдельные задачи.
- `contract.py` — **это НЕ JUMP-контракты distrohost**, а входной протокол-интерфейс источников (`RawJiraIssue`/`IssueSource`, граница изоляции). Отдельного ingest под distrohost/JUMP-контракты на проде НЕТ.

## Три сущности Евы (из рекона) → что реально выгружено

1. **EVA-трекер (eva.iva.ru)** — ✅ **выгружен полно**: 25 проектов, 6634 задачи, 2332 связи, свежесть 04.07. Кормит управленческий контур.
2. **Eva-wiki (orionpro DOC-000245)** — ❌ **следов на проде нет**: ни файлов, ни коллекции. Пробел.
3. **distrohost JUMP-контракты** — ❌ **не выгружены**: ingest отсутствует, distrohost из прод-контура не резолвится. (В `git/repomix/jump.xml` лежит repomix ИСХОДНИКОВ репо jump — это код, не контракты.)

## Прочие корпуса `/opt/helm/data/real/` (для контекста дозагрузки)

git 223M (all_commits.csv 31M, merge_requests_all.csv 135K → `mr_source.py`, repomix 10 репозиториев, topology), service 6.8M, jira 2.5M, data-room 1M, crm 796K, identity 592K, sensitive 244K, monitor-gd 228K, confluence 216K, manual 40K.

## Пробелы / нужен доступ

- **Meili doc-counts** — нужен `HELM_MEILI_KEY` (или read-only ключ), гардрейл заблокировал.
- **Eva-wiki (DOC-000245)** — не выгружалась вовсе.
- **distrohost JUMP-контракты** — не выгружались (нет ingest + приватный контур).
- **Postgres** — БД управленческого контура, `role postgres` отсутствует (нужен корректный db-user, если понадобится инвентарь mgmt-графа).
- Данные Евы на диске от 04.07 — при дозагрузке проверить актуальность (сегодня 17.07).

## Relations
- part_of [[RAG#2]]
- relates_to [[Ева]]
