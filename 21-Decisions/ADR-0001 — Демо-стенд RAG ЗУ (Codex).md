---
title: ADR-0001 — Демо-стенд RAG ЗУ (Codex)
type: decision
permalink: tacticum/21-decisions/adr-0001-demo-stend-rag-zu-codex-1
tags:
- ADR
- codex
- rag
- zu
- decision
- isolation
---

# ADR-0001 — Демо-стенд RAG для клиента «Залог Успеха» (Codex)

- **Статус:** Accepted (ратифицирован в grill-сессии 2026-06-25)
- **Дата:** 2026-06-25
- **Репозиторий:** `rag_eval_service` (= Tacticum **Codex**)
- **Источник на диске:** `~/tacticum/fast_task/0001-zu-demo-rag-architecture.md`
- **Контекст-арбитр:** `docs/concept/architecture.md` (§1.1 водораздел, §3.3.1 Knowledge/RAG, §3.3.3 Document Processing, §3.4 Tenancy, §7 запреты App)

## Контекст
Клиенту **ЗУ** делаем демо-стенд: оператор КЦ логинится через Tacticum IdP (project-hub OIDC), видит чат и историю, задаёт вопрос и получает быстрый ответ по своим данным (Word/Excel/PDF, аудиозвонки) со ссылками на исходные документы. Цель ADR — зафиксировать архитектуру демо как **tracer-bullet (вертикальный срез)**, намеренно НЕ трогая репо `platform`, но без выбрасываемого кода. Сверка на момент решения: `rag_eval_service` — пустой каркас; `doctranslator` — зрелый extract-домен (донор); `agents` RAG — BYO-коллекция без фильтра tenant (латентная брешь); `tei_service` — bge-m3/1024 через DeepInfra; `project` — OIDC/Gateway/transcription MCP; стенд `zu_demo` (159.194.212.2).

## Решения
- **D1 — Tracer-bullet, своя шкала.** Тонкий вертикальный срез, развязанный с консолидацией платформы (ADR-0005 Phase 1). Полная Phase 1 в критпуть демо не ставится.
- **D2 — `platform` НЕ трогаем (техдолг).** В демо только `rag_eval_service`, `project`, `tei_service`. `agents` не трогаем; `doctranslator` — донор-копипаст. Вынос knowledge_rag/document_processing/контракта в platform — зафиксированный техдолг после демо.
- **D3 — Карта кода.** Codex = App+движок (ingestion, docproc, RAG-ретрив, Ask BFF, история, цитаты, OIDC-клиент, eval). `project` = OIDC+Gateway+transcription. `tei_service` = эмбеддинги. Инфра (Qdrant/MinIO/Postgres) на `zu_demo`. Фронт — `zu - figma`.
- **D4 — Docproc extract-only, донор doctranslator.** Vendored: docx/xlsx/pdf + OCR, IR `ExtractedDocument`. Реконструкторы/TM не переносим. Чанкинг — на RAG-слое Codex.
- **D5 — Аудио в MVP, видео нет.** Звонки → transcription MCP → текст+тайминги → ingest как источник. Видео — fast-follow.
- **D6 — RAG-ретрив: naive semantic, написан заново.** chunk → bge-m3/1024 (порт `Embedder`) → Qdrant, коллекция `knowledge__bge-m3_1024`; v1 только semantic; `mode` заложен под hybrid позже; reranker — тёмный порт (bge-reranker-v2-m3, не поднят).
- **D7 — Изоляция логическая, fail-closed с первого дня.** Коллекция ЗУ + обязательный payload `tenant_id=zu` + fail-closed фильтр в Qdrant. `StoreRouter` — порт (позже выделенный Qdrant/on-prem без смены контракта). ЗУ — отдельный тенант в project-hub.
- **D8 — Эмбеддинги/LLM через имеющееся.** Embedder = tei bge-m3/1024 (сейчас DeepInfra ⚠ для on-prem нужен локальный bge-m3, без ПДн во внешний API). LLM ответа — LiteLLM Gateway, стриминг.
- **D9 — Цитаты и «открыть документ».** Оригиналы в MinIO; payload цитаты `{tenant_id, source_doc_id, title, storage_uri, page/section, timestamp}`. BFF → `{answer, citations[]}` с presigned MinIO. PDF `#page=N`, docx/xlsx — скачать, аудио — транскрипт на таймкоде. Подсветка спана — отложена.
- **D10 — История чатов на стороне App** (Postgres Codex, scope по user_id). Naive RAG = одношаговый retrieve→answer.
- **D11 — Auth: project-hub OIDC + AuthScope end-to-end.** Фронт — OIDC authorization-code. Операторы — юзеры тенанта `zu`. BFF передаёт AuthScope; `ScopedRepository` держит fail-closed `tenant_id=zu`.
- **D12 — Работы/фазы/eval-гейт.** Шульга: RAG-ретрив + docproc + eval-harness + golden-set. Заказчик ADR: фронт. Фазы M0 (каркас+инфра) → M1 (тонкий срез: один PDF end-to-end) → M2 (docx/xlsx, аудио, батч, история) → M3 (eval+тюнинг) → M4 (харднинг+прогон). Eval-гейт **мягкий** (recall@k/MRR/nDCG, не блокирует выпуск).

## Открытые вопросы
- O1. Владелец «среднего слоя» (Ask BFF/оркестрация/чаты/OIDC-клиент/MinIO/деплой) — Шульга=движок, заказчик=фронт, середина без владельца.
- O2. UX «открыть аудио»: транскрипт на таймкоде vs проигрывание (реком. — транскрипт для MVP).
- O3. Согласие владельца `doctranslator` на vendoring.
- O4. Заведение тенанта ЗУ в project-hub (UI-only): кто/когда.

## Последствия
**+** Срок демо развязан с консолидацией платформы; инвариант fail-closed+AuthScope демонстрируется клиенту; Codex заводится по-настоящему, eval-harness получает дом; код не выбрасывается (кандидат на вынос в platform).
**−** Техдолг: knowledge_rag/document_processing/контракт временно в App (осознанно, нужен cutover); эмбеддинги демо во внешний DeepInfra (для on-prem нужен локальный bge-m3); дубль docproc-кода (vendored) до выноса.

## Relations
- part_of [[21-Decisions]]
- relates_to [[ADR-0002 — LightRAG (graph-RAG) для ЗУ]]
- relates_to [[TARGET — целевая архитектура knowledge v2.3]]
- relates_to [[Водораздел: платформенный knowledge vs Codex]]
- relates_to [[Runbook: прогон eval на сервере]]
- relates_to [[lightrag-в-codex]]
- relates_to [[glossary]]