---
title: lightrag-server-eval-2026-07-02
type: report
permalink: tacticum/01-sessions/lightrag-server-eval-2026-07-02-1
tags:
- lightrag
- zu_demo
- eval
- age
- postgres
- server
- teardown-pending
status: archived
updated: 2026-07-18
---

# LightRAG на zu_demo: замер сделан, реализация ОСТАВЛЕНА на сервере (teardown отложен) — 2026-07-02

Серверный замер graph-RAG (LightRAG) на боевом стенде ЗУ выполнен по [[plan-lightrag-server-eval]] (Ш0–Ш7). Ш8-teardown **сознательно отложен** решением пользователя — реализация живёт на стенде, снесём по инструкции ниже, когда потребуется.

## Итог замера (ADR-0002 D6)
- Tenant замера: **`cifragen`** (определён read-only: 69 доков / 577 чанков в `knowledge__bge-m3_1024`, единственный tenant с данными, fail-closed).
- Baseline semantic на сервере (`inprocess` = боевая `rag.search.search()`) воспроизвёл исторические числа: **recall@5=0.9667 / MRR=0.8595 / nDCG=0.8745**.
- LightRAG mix (Postgres+AGE, тот же корпус, множество документов ТОЧНО совпадает с индексом): **recall@5=0.8048 / MRR=0.6179 / nDCG=0.6282**.
- **Δ recall = −0.162.** Покейсно: 0 выигрышей графа, 6 регрессий (q7,q8,c2,c22,c24 → 0; c3 → 0.33), 29/35 идентичны. Граф выталкивает точный документ за top-5.
- **Вывод:** на фактологии ЗУ граф не помогает (отрицательная дельта) — подтверждает гипотезу ADR-0002 (D1). Прод оставить semantic; граф не продвигать. Решение о проде — за руководителем. Перепроверить, если появится корпус реляционных вопросов.
- Полный отчёт-артефакт: `~/tacticum/… lightrag-eval-report.md` (передан пользователю).

## Что развёрнуто на сервере (АРТЕФАКТЫ ДЛЯ TEARDOWN)
- **codex-git** (`/home/tacticum-deploy/codex-git`): checkout ветки **`migration/lightrag`** @ `bd38ab2` (working tree clean). Есть `stash@{0}: On main: lightrag-eval-dockerfile` — сохранённая чужая правка `COPY eval/` в Dockerfile.
- **Apache AGE 1.6.0** установлен в боевой контейнер `zu-deploy-postgres-1` (образ postgres:16): скопированы `age.so` (→ `/usr/lib/postgresql/16/lib/`), `age.control`, `age--1.6.0.sql` (→ `/usr/share/postgresql/16/extension/`). Собраны из official apache/age (ветка PG16) в эфемерном контейнере (ABI = 16.14/Debian13). Работает per-session `LOAD 'age'`, боевой Postgres НЕ перезапускался (shared_preload не трогали).
- **БД `codex`** (боевая, история чатов): `CREATE EXTENSION age`; AGE-граф `cifragen_chunk_entity_relation` (1302 узла / 1438 рёбер); 8 пустых таблиц `lightrag_doc_chunks, lightrag_doc_full, lightrag_doc_status, lightrag_entity_chunks, lightrag_full_entities, lightrag_full_relations, lightrag_llm_cache, lightrag_relation_chunks` (KV/doc-status ушли в файлы, таблицы забутстрапил общий PG-клиент — данных в них 0).
- **Qdrant**: коллекции `lightrag_vdb_chunks` (205), `lightrag_vdb_entities` (1302), `lightrag_vdb_relationships` (1438). Боевая `knowledge__bge-m3_1024` (577) НЕ тронута.
- **Контейнеры**: `lightrag-eval` (persistent, python:3.12-slim, сеть zu-deploy_default, mount codex-git:ro, deps+/work артефакты) и `age-builder` (exited, эфемерный билдер). Хост: `/tmp/age-build/` (артефакты AGE + скрипты).
- **swap**: `/swapfile` 2G включён (не в fstab).

## РАНБУК TEARDOWN (выполнить, когда решим убрать)
1. Qdrant: `DELETE /collections/lightrag_vdb_chunks|lightrag_vdb_entities|lightrag_vdb_relationships` (knowledge__bge-m3_1024 НЕ трогать).
2. `docker exec -i zu-deploy-postgres-1 psql -U codex -d codex`: `LOAD 'age'; SET search_path=ag_catalog,public; SELECT drop_graph('cifragen_chunk_entity_relation', true);` → `DROP TABLE` 8× lightrag_* → `DROP EXTENSION age CASCADE;` → при остатке `DROP SCHEMA ag_catalog CASCADE;`.
3. Удалить файлы AGE из боевого контейнера: `age.so`, `age.control`, `age--1.6.0.sql` (lib + extension каталоги).
4. `docker rm -f lightrag-eval age-builder`; `rm -rf /tmp/age-build`.
5. codex-git: `sudo -u tacticum-deploy git checkout main` + `git stash pop` (вернуть правку `COPY eval/`).
6. swap: `swapoff /swapfile && rm /swapfile` (или оставить).
7. Проверить демо: knowledge=577, codex-1 up, `/health`+`/docs`=200.

## Guardrails, соблюдённые в сессии
knowledge read-only; в боевую БД только артефакты LightRAG; codex-1 не пересобирался (`/ask` жив); `.env` не читали/не печатали (секреты в eval-контейнер через `--env-file`); строго один tenant `cifragen` (fail-closed); отказ политики (сборка внешнего кода) не обходили — прошли по явному разрешению пользователя.

## Relations
- part_of [[01-Sessions]]
- relates_to [[plan-lightrag-server-eval]]
- relates_to [[0002-zu-lightrag-graph-rag]]
- relates_to [[lightrag-codex-checkpoint-2026-07-01]]
- relates_to [[server-zu-demo]]