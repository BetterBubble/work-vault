---
title: RAG#1 — чеклист улучшений vs ЗУ (нюансы codex)
type: note
permalink: tacticum/20-architecture/rag-1-cheklist-uluchshenii-vs-zu-niuansy-codex
tags:
- iva
- rag
- rag1
- чеклист
- качество
- codex
- ux
---

## Назначение
Что не повторять из RAG ЗУ (codex) и как сделать RAG#1 (документация ИВА) круче/удобнее. Сверено по коду `rag_eval_service` main + памяти. Формат: было в ЗУ → делаем в RAG#1. ✅ = уже улучшено у нас.

## ТОП-приоритет (must, максимальный эффект)
A2 реранкер · A1 структурный чанкер · A5 богатые метаданные · B1 answer-eval+judge · B5 юнит-тесты · C1 сниппет+якорь-URL · C3/C4 фильтры+версия · D3 grounding/отказ.

## A. Retrieval-качество
- **A1 Чанкинг [must]:** ЗУ — слепой char-slicing 900/150 (рвёт заголовки/таблицы; табличный fix только в ветке `fix/table-chunking`, не в main). RAG#1 — структурный по Markdown (H1-H3 границы, абзацы, списки), record-aware таблицы, sentence-aware overlap. Корпус ИВА — чистый .md, использовать.
- **A2 Reranker [must]:** ЗУ — всегда off (NotImplementedError), признан главным незакрытым рычагом (ADR-0002). RAG#1 — ✅ поднять bge-reranker-v2-m3 поверх hybrid; candidate_k 20→50-100.
- **A3 Hybrid [nice]:** ЗУ дефолт semantic (hybrid без реранкера проигрывал). RAG#1 — hybrid+reranker, A/B на golden; полнотекст ловит коды/API/названия продуктов.
- **A4 Query rewrite [nice]:** ЗУ нет. RAG#1 — лёгкий rewrite аббревиатур (МСУ/MCU) — сначала померить.
- **A5 Метаданные [must]:** ЗУ payload беден (tenant/doc/title/page). RAG#1 — ✅ пробросить весь frontmatter (product/section/breadcrumb/version/slug/url/kind) + heading-path → фильтры, якоря, срезы.
- **A6 Контекстный retrieval [nice]:** префикс heading-path/breadcrumb к чанку (contextual-lite), дёшево из frontmatter.

## B. Eval / достоверность
- **B1 Answer-eval/LLM-judge [must] — ГЛАВНЫЙ пробел ЗУ:** ЗУ только retrieval doc-level (слеп — «recall@5=1.0, а ответа в тексте нет»). RAG#1 — ✅ answer-in-context@k + judge faithfulness/groundedness + correctness vs ideal_answer.
- **B2 Chunk-level/graded метрики [nice]:** ЗУ doc-level binary. RAG#1 — relevant_chunk_ids.
- **B3 Golden [✅]:** ЗУ 35 кейсов. RAG#1 — 1306, 100% покрытие. КАВЕАТ: синтетический → человеческая выверка 100-200 до боевого eval.
- **B4 Антипримеры [must]:** ✅ 32 known_gaps, гейтить «честный отказ».
- **B5 Юнит-тесты [must]:** ЗУ — в rag/bff/eval НОЛЬ тестов. RAG#1 — покрыть chunker/RRF/citations/metrics + тест fail-closed изоляции.

## C. UX / фронт
- **C1 Цитаты [must]:** ЗУ — title+«стр.N»+presigned S3, без сниппета и inline-привязки; дедуп «1 док=1 цитата» скрывает разные места. RAG#1 — сниппет фрагмента + якорь-URL прямо на раздел iva.ru/docs (у нас `url` в frontmatter), inline-нумерация [1][2].
- **C2 Подсветка [nice]:** совпадений запроса в сниппете.
- **C3 Фильтры [must]:** ЗУ нет. RAG#1 — по продукту/разделу/версии (16 продуктов ИВА).
- **C4 Версия/актуальность [must]:** индексируем version=latest, в цитате показывать версию (корпус = 796 latest + 1348 версионных).
- **C5 Мобильность [nice]:** ЗУ история `hidden md:block` (нет на мобиле). RAG#1 — drawer.
- **C6 Сохранить из ЗУ:** стриминг NDJSON, фидбэк 👍/👎+коммент, состояния loading/error/401/«не нашлось», переименование/удаление истории.

## D. Надёжность / операционка
- **D1 Fallback [✅ сохранить]:** hybrid→semantic при падении Meili, Meili-сбой не роняет Qdrant, LLM-ретраи backoff.
- **D2 Изоляция tenant [✅ сохранить+тест]:** fail-closed.
- **D3 Grounding/отказ [must]:** системный промпт «только из контекста, иначе честно не нашёл» + гейт known_gaps. Критично для support (не выдумывать API/настройки).
- **D4 Обновление корпуса [must]:** ЗУ — ручная переиндексация, грабли с транслит-title. RAG#1 — ✅ стабильные id `<slug>#<n>`, инкрементальный ingest по slug, corpus_version в payload, title из frontmatter.
- **D5 Near-duplicate [nice]:** ИВА — `*-introduction` vs `-pdf`, glossary iOS/WEB — канонизация/дедуп.
- **D6 Cap чанков/документ [nice]:** параметр ретрива (НЕ diversity=1), против краудинга.

## Отношения
- part_of [[Концепт: три RAG для ИВА на общем движке]]
- relates_to [[Codex (rag_eval_service) — слои поверх retrieval для RAG#1]]
- relates_to [[RAG#1 ИВА — корпус iva.ru/docs + golden-set готовы]]
- relates_to [[verify-data-credibility]]
