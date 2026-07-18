---
title: LightRAG локальная проверка — сырой прогон (verifier)
date: 2026-07-01
tags:
- lightrag
- eval
- verifier
- draft
- adr-0002
permalink: tacticum/00-inbox/lightrag-local-verify-raw
status: archived
updated: 2026-07-18
---

# LightRAG (migration/lightrag) — локальная проверка, сырьё

Worktree: `/Users/bubblemac/tacticum-worktrees/rag_eval_service-lightrag`
Ветка: `migration/lightrag` (4 коммита поверх 52525c4)
Окружение: venv python3.12.13 (в system python нет pip/pytest; py3.14 ensurepip падает), pytest 9.1.1,
lightrag-hku **1.5.4**, neo4j:5-community в docker (localhost:7687, neo4j/testpassword).

## Результаты по пунктам

### П.1 Юнит-тесты адаптера LightRAGBackend — PASS
- `pytest eval/tests` → 20 passed (8 smoke implementer2 + 12 verifier).
- Мок: фейковый `lightrag` модуль (QueryParam) + мокнутый `_ensure_rag`/`aquery`.
- Проверено: маппинг retrieved→EvalHit, порядок+дедуп, проброс mode/only_need_context/chunk_top_k,
  дефолтный режим, rerank→NotImplementedError.
- Замечание (минор, не блокер): `LightRAGBackend._loop = asyncio.new_event_loop()` не закрывается →
  `PytestUnraisableExceptionWarning` (Invalid file descriptor) при GC. Для короткоживущего eval-процесса
  не критично; для чистоты — закрывать loop в конце (напр. describe/close или atexit).

### П.2 Изоляция fail-closed — HOLDS
- `search(tenant_id="")` → `[]` БЕЗ построения графа (_ensure_rag не вызван).
- `build_lightrag("")` → ValueError.
- Кросс-tenant: отдельный LightRAG на tenant (self._rags per-tenant), выдачи A/B не смешиваются.
- На реальном Neo4j (п.5): workspace B (пустой) → пусто. Изоляция workspace держит.

### П.3 Офлайн-валидатор схемы — PASS
- `python -m eval.validate --golden eval/golden_sets/example.json --check-schema` → exit 0, «Схема OK. 2/2».

### П.4 Регресс semantic-пути — PASS (регресса нет)
- Ядро НЕ тронуто: `eval/runner.py`, `eval/metrics.py`, `rag/search.py`, `rag/store.py` — не в диффе (byte-identical к baseline 52525c4).
- `eval/backends.py`: +85/−0 — чисто additive, InProcessBackend не задет.
- Фабрика: `get_backend('inprocess')`→InProcessBackend, `EVAL_BACKEND=lightrag`→LightRAGBackend, дефолт→InProcessBackend.

### П.5 Docker-Neo4j смоук — ИНФРА OK, но НАЙДЕН БЛОКЕР маппинга
Инфра работает: Neo4JStorage коннектится к localhost:7687, ingest 2 доков проходит, 12 стораджей
инициализируются/финализируются, workspace-изоляция B→пусто.

**БЛОКЕР: `extract_doc_ids` несовместим с форматом retrieved-контекста lightrag-hku 1.5.4.**
`aquery(mode="naive", only_need_context=True)` возвращает СТРОКУ вида:
```
Document Chunks (... reference_id ...):
```json
{"reference_id": "1", "content": "Договор залога недвижимости. ..."}
```
Reference Document List (...):
```
[1] doc-zalog-1
```
```
- В чанках НЕТ поля `file_path` (в 1.5.4 это `reference_id` + `content`).
- doc_id (`doc-zalog-1`) живёт в секции **Reference Document List** как `[reference_id] doc_id`.
- `extract_doc_ids` грепает `"file_path": "..."` → для 1.5.4 всегда `[]`.

**Последствие:** `LightRAGBackend.search()` вернёт пустой список doc_id на ЛЮБОМ запросе →
recall@k / MRR / nDCG = **0.0** по всему golden-set. На сервере это выглядело бы как полный провал
vs baseline 0.967 — но это баг маппинга, а не качества графа.

**Направление фикса (implementer, НЕ verifier):** `extract_doc_ids` должен парсить формат 1.5.4 —
секцию `Reference Document List` (строки `[N] <doc_id>`), сохраняя порядок/ранг документов, и/или
маппить reference_id из Document Chunks → doc_id. Старый `file_path`-путь оставить как fallback для
других версий/форматов.

## Граница «осталось на сервере» (подтверждено)
- Реальный граф по 70 докам через боевой gateway-LLM (O1 нагрузка, O2 ПДн).
- Полный замер recall@k/MRR/nDCG vs baseline 0.967 (ADR-0002) — на zu_demo.
- Совместимость с боевой коллекцией Qdrant (O3, `--check-index`), RAM/swap.
- mix/local/global-режимы с реальным LLM (мок-LLM keyword-extraction не даёт → mix локально не мерится).

## Вердикт (первый прогон)
Локальная часть НЕ готова к серверному тесту из-за блокера в extract_doc_ids (recall@k=0 на 1.5.4).
После фикса маппинга — повторить п.5 смоук (ingest→aquery→doc_id непусто) и тогда green-light на сервер.
Инфра/изоляция/регресс/схема — в порядке.

---

## ПОВТОРНЫЙ ПРОГОН после фикса f7dfabe — БЛОКЕР ЗАКРЫТ

`extract_doc_ids` переписан: `_extract_via_reference_list` парсит секцию «Reference Document List»
(`[N] <doc_id>`), ранг по порядку reference_id чанков (N→doc_id, дедуп); fallback — порядок ref-списка;
затем `_extract_via_file_path` как второй fallback. `LightRAGBackend.close()/__del__` закрывает loop.

Результаты:
- `pytest eval/tests` → **22 passed** (10 smoke incl. 2 новых на формат 1.5.4 + 12 verifier). Warning про
  незакрытый event-loop больше не появляется.
- Повторный п.5 смоук на живом Neo4j: `[A] extract_doc_ids -> ['doc-zalog-1']` (непусто, doc_id корректен);
  `[B] чужой/пустой workspace -> []` (изоляция держит); VERDICT: PASS.
- Оговорка: fake-эмбеддинги дают ретрив только на точный текст чанка (diff-cos ~0.04 < порог 0.2),
  на живом смоуке подтверждён 1 документ; корректность РАНГА при нескольких doc покрыта юнит-тестами
  парсера (reference_id order), не живым ретривом.

## ФИНАЛЬНЫЙ ВЕРДИКТ
Локальная часть — **GREEN-LIGHT к серверному тесту**. Блокер extract_doc_ids закрыт, маппинг непуст и
ранжирован, изоляция держит, регресс/схема/юниты чистые, event-loop warning ушёл. На сервере остаётся:
реальный граф по 70 докам через боевой LLM (O1/O2), полный замер recall@k vs 0.967 (ADR-0002), боевой
Qdrant (O3, --check-index), RAM/swap, mix/local/global с реальным LLM (сверить фактический формат
ref-списка на боевом прогоне — парсер толерантен, но подтвердить стоит).