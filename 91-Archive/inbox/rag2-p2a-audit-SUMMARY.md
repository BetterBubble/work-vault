---
title: rag2-p2a-audit-SUMMARY
type: report
permalink: tacticum/00-board/rag2-p2a-audit-summary
tags:
- rag2
- rerank
- confidence
- p2a
- audit
- helm
status: archived
updated: 2026-07-18
---

# Р-2a: нормированный confidence + порог отсечки шума — аудит и дизайн (read-only)

База: **`main` (b5d1739)**. Задача T2.1 — карта для implementer'а, код НЕ трогал.

## 1. Карта 14 тулов: кто отдаёт скор/ранжирование клиенту
Все ретрив-тулы делегируют в `rag2_router.search`/`context` (`interface/api/routers/rag2.py`) и сериализуют через DTO. Ключ — `Rag2HitOut.score: float | None` (:54).

**Над raw-ретривом (score/ранг в ответе):**
- `analyst_search` (asrv:254) → `Rag2SearchOut.results[*].score` — **сырой скор в ответе**.
- `related_tasks` (asrv:289) → `[h.model_dump()]` c `score` — **сырой скор**.
- `nearest_spec` (asrv:779) → `specs[*].score = h.score` — **сырой скор** (Confluence-хиты).
- `constraints` (asrv:1126) → items БЕЗ score, но **порядок = ретрив-ранг** (title/space/url/snippet/kind).
- `contradiction_check` (asrv:1162) → candidates БЕЗ score, **порядок = семантическая близость**.
- `analyst_context` (asrv:272) → `Rag2ContextOut.citations` (`Rag2CitationOut`) — **score НЕ выводится**, но состав/порядок цитат = ретрив (порог повлияет на то, что попадёт в контекст).

**Агрегирующие / без ретрив-скора:** `arch_map`, `arch_container`, `affected_systems` (matched_by+requirements), `requirement_coverage` (матрица покрытия), `who_to_involve` (люди), `effort_hint` (медианы changelog), `gap_questions` (чеклист), `docs_ask` (RAG#1: `DocsAskOut` = answer+citations, **без score клиенту**).

➡️ Единая точка нормировки/порога = формирование `Rag2Result.hits` в `application/rag2.py:answer` + сериализация `Rag2HitOut` — покрывает analyst_search/related_tasks/nearest_spec и (через `out.results`) constraints/contradiction_check/context разом.

## 2. Шкалы численно — три разные, несопоставимые (подтверждаю: единый фикс-порог невозможен)
`JiraDoc.score` заполняется на РАЗНЫХ стадиях РАЗНОЙ шкалой:
- **Dense cosine (Qdrant)** ~0.5–0.9. Фикстуры: `test_docs_search.py:60/91-92` (0.9/0.5), `test_rag2_search.py:62` (0.7). Достигает клиента только в semantic-only корпусе (helm — всегда semantic).
- **RRF-fused** = `Σ 1/(k+rank+1)`, `RRF_K=60` (per-corpus, `search.py:40,215`) и `FEDERATE_RRF_K=60` (межкорпус, `domain/rag2.py:321`). Топ одного источника = **1/(60+0+1) ≈ 0.0164**.
- **Rerank `relevance_score`** ∈ **0..1** (Cohere-совм. `/rerank`, `assistant/reranker.py:88` `item["relevance_score"]`). Фикстуры `test_reranker.py:41-42,68-70,86`: 0.1/0.4/0.5/0.9/1.0.

Три несовместимые шкалы (0.5–0.9 vs ~0.016 vs 0..1) в ОДНОМ поле `score` → **любой единый числовой порог ломается** на смене флагов/корпуса. Подтверждено.

## 3. «Шум ~0.016» — это RRF-fused, НЕ rerank (ключевой инсайт)
Порядок в `answer` (`application/rag2.py:169-210`): federate `index_hits = federate(by_source)` **ПЕРЕЗАПИСЫВАЕТ** `score` на RRF-fused (`domain/rag2.py:344` `replace(..., score=scores[ident])`). Затем опц. cross-rerank перезапишет снова; per-corpus rerank-скор (0..1) **теряется на federate — используется только для ПОРЯДКА внутри корпуса**.
Итог для клиента по умолчанию (per-corpus rerank ON, cross OFF): **score = federate RRF ≈ 0.0164 даже у ЛУЧШЕГО хита** — это ранговый артефакт, а не низкая релевантность. Именно он «выглядит как шум/неподтверждение». При cross-rerank ON → клиент видит cross `relevance_score` 0..1.
⚠️ Следствие: наивный порог на текущем `score` (~0.016) **отрежет ВСЁ, включая топ**. Резать нужно НЕ этот артефакт.

## 4. Точки вставки + предлагаемый дизайн
**Принцип:** ввести ОТДЕЛЬНЫЙ калиброванный `confidence` ∈ 0..1, не мешать его с ранговым `score`; источник confidence — rerank-relevance (0..1, калибруема), которую сейчас теряет federate.

**(а) Нормировка confidence**
- `domain/rag2.py`: +поле `JiraDoc.confidence: float | None` (frozen dataclass) + чистый хелпер `normalize_confidence(raw, *, scale)` — для rerank-relevance: clamp[0..1] (уже 0..1) или мягкая калибровка (sigmoid по логиту, если перейдём на raw-logit эндпоинт; изотоническая — later, нужен лейблинг golden). Для fused-без-rerank шкала иная → отдельная ветка (напр. rank-based confidence `1/(1+rank)` вместо RRF-суммы) ЛИБО `confidence=None` (честно «нет калибровки»).
- `infrastructure/rag2/reranker.py:JiraReranker.rerank` (:31-39): писать relevance в `doc.confidence` (0..1), а `score`/порядок — как сейчас. Тогда federate НЕ затрёт confidence.
- `domain/rag2.py:federate` (:324-344): при `replace(...)` СОХРАНЯТЬ `confidence` (перезаписывать только fused `score`). Для cross-rerank — писать cross-relevance в `confidence` в `application/rag2.py:answer` после cross-rerank.

**(б) Порог отсечки шума**
- `application/rag2.py:answer`: ПОСЛЕ rerank/cross-rerank и ДО `hits = merged.hits[:k]` — партиция по `confidence`: `below = confidence < policy.noise_floor`. Действие настраиваемое: (1) `drop` (убрать) ИЛИ (2) `tag` — не резать, но проставить `below_noise_floor=True` (безопаснее для recall, решение аналитику). Рекомендую **tag по умолчанию**, drop — опционально.
- `application/rag2.py:Rag2Policy`: +`noise_floor: float | None = None`, +`noise_action: str = "tag"`.
- Порог применять к `confidence` (0..1), НЕ к `score`. Дефолт off (`None`) — не ломать текущее.

**(в) Единообразный проброс в тулы**
- `interface/api/routers/rag2.py:Rag2HitOut`: +`confidence: float | None = None`, +`below_noise_floor: bool = False`; `_hit_out` (:123-140) домапить из JiraDoc. → сразу в analyst_search/related_tasks/nearest_spec.
- `constraints`/`contradiction_check` (`analyst_server.py`) — фильтруют `out.results`; порог применится upstream, для них добавить опц. отброс/пометку по тем же полям.
- `config.py`: +`rag2_confidence_enabled: bool=False`, +`rag2_noise_floor: float|None=None` (env `HELM_RAG2_NOISE_FLOOR`), +`rag2_noise_action`. `service.py:build_rag2_context` — прокинуть в `Rag2Policy`.

## 5. Как измерить «шум отсечён, реальные хиты целы»
- **Реальные хиты целы:** существующий golden (68 кейсов, `expected_sources`/`retrieval_mode`/`difficulty`) + harness `eval/rag2_runner.py`: `recall@k`/`hit@k`/`nDCG@k`/`answer_in_context@k` (`eval/metrics.py:69-94`) НЕ должны просесть после порога. Гонять off/on-floor sweep.
- **Шум реально режется:** сейчас метрики только по recall/ранжу — **precision/ложных-срабатываний НЕТ**. Нужен новый сигнал: (1) добавить в metrics.py `precision@k` / `noise_kept_rate`; (2) набор «должно быть пусто/отказ» — переиспользовать кейсы `needs_confluence_body=future` и `domain/known_gaps.json` (RAG#1) как negative/OOD, где ожидаем `no_answer`/пустую выдачу выше порога.
- Калибровка порога: sweep `noise_floor` (напр. 0.1/0.2/0.3 по confidence) на golden+negative → выбрать точку max precision при recall ≈ baseline.

## Открытые вопросы
1. Источник confidence: остаёмся на Cohere-совм. `relevance_score` (0..1, уже есть) или переходим на raw-logit + собственная sigmoid-калибровка? Первое — быстрее, второе — точнее.
2. По умолчанию `tag` (пометить `below_noise_floor`) или `drop`? Рекомендую tag (не бьёт recall, решение аналитику).
3. Confidence при выключенном rerank (fused-only) — считать rank-based или честно `None`? 
4. Negative-набор для precision — расширять golden или отдельный файл? Нужен лейблинг «ожидаемо пусто».
5. Порог — глобальный или per-source (helm/jira/confluence разные шкалы даже внутри confidence)? Скорее per-source после калибровки.
