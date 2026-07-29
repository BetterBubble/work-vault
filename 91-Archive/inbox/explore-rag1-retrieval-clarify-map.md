---
title: explore-rag1-retrieval-clarify-map
type: explore
permalink: tacticum/00-board/explore-rag1-retrieval-clarify-map-1
tags:
- rag1
- docs-assistant
- retrieval
- clarify
- eval
- explore
archived-at: 2026-07-29 18:12
---

# explore-rag1-retrieval-clarify-map

status: draft
Репо: /Users/bubblemac/tacticum/helm @ ea295f8 (main). Только разведка, правок нет.
Задача: карта кода под 2 проблемы RAG#1 — (1) корректность на «активный звонок vs мероприятие», (2) калибровка clarify Ф5.

## 1. Пайплайн docs_ask end-to-end (поток данных)

Оркестратор: `DocsAssistant.ask` — `src/helm/application/docs_assistant.py:241-341`.
Порядок стадий (комментарий там же :209 «retrieve → (rerank) → cap → context → generate → guardrail»):

1. **known-gaps гейт** (:253-256) — `classify_known_gap`, вне охвата → отказ без ретрива/LLM.
2. **retrieval** (:257) — `self._search.search(question, limit=SEARCH_LIMIT=30, filters)`.
   - Реализация: `DocsSearch.search` — `src/helm/infrastructure/docs_assistant/search.py:107-133`.
   - Режим `hybrid` (`_hybrid` :158-204): параллельно Meili(fulltext) в ThreadPoolExecutor + embed(bge-m3)→Qdrant dense; слияние `rrf_fuse` (`fusion.py`, RRF_K=60); `CANDIDATE_K=60` тянется из каждого стора.
   - Опц. `query_rewrite` (:113-118, `expand_query`+synonyms, ДЕФОЛТ off) и near-dup dedup (:131-132).
   - Meili падает → semantic-only (:185-187), ответ не роняем.
3. **rerank** (:259-278) — только если reranker есть. Кап входа `rerank_candidate_cap=20` (:263-264 `chunks[:cap]`), затем `DocReranker.rerank` (`reranker.py:27-59`, Gateway `tacticum/rerank`, bge-reranker-v2-m3, текст = `doc_rerank_text` = heading_path ⊕ text). score реранка (0..1) кладётся в `DocChunk.score`. Опц. `rerank_floor` отсекает слабые (:276-278, ДЕФОЛТ None).
4. **cap-per-doc + context_limit** (:279) — `_cap_per_doc(chunks, max_chunks_per_doc=4)[:context_limit=10]`.
5. **confidence-гейт Ф2** (:287-300) — ТОЛЬКО при `allow_clarify=True`. `decide_retrieval_action` → "answer"|"not_found". not_found → отказ без LLM.
6. **context render** (:301) — `render_docs_context` (`domain/docs.py:83-102`).
7. **generate** (:302-328) — `self._llm.generate(system, user)`. LLM: `TrivaLlm` c `max_tokens=docs_answer_max_tokens=700` (`service.py:96`).
8. **guardrail + citations** (:329-341) — `decide_guardrail(len(chunks), text)`, `select_cited_chunks`.

Вход/каналы:
- Веб: `POST /api/docs/ask` — `src/helm/interface/api/routers/docs.py:223-286`; clarify-петля `_ask_with_clarify` :289-327 (channel="web", ключ=email).
- Бот: `src/helm/interface/api/routers/bot_support.py:121-168` (`run_docs_ask` plain=True; `_resolve_answer` clarify channel="bot", ключ=chatRoomId).
- MCP `docs_ask`: `interface/mcp/analyst_server.py:405` → зовёт docs_router.ask.

## 2. Grounding / генерация — где расходятся «мероприятие» и «звонок»

- SYSTEM_PROMPT — `docs_assistant.py:70-89` (строго по фрагментам, TL;DR+шаги, цитаты [n]).
- Контекст в промпт: `render_docs_context` (`domain/docs.py:83-102`) — каждый чанк как `[n] title · продукт · раздел(heading_path) · URL` + сниппет тела `_snippet` обрезан до `_CONTEXT_SNIPPET_LEN=1200` символов (:39). Тело режется — навигационный сигнал несут title/heading_path.
- User-промпт: `_build_user_prompt` (:204-205) = «Вопрос: … Фрагменты документации: …».
- Цитаты: `select_cited_chunks` (`domain/docs.py:195-260`) — оставляет только реально упомянутые [n], дедуп по документу (url→slug→title), перенумерация.

**КОРЕНЬ ПРОБЛЕМЫ 1 (сигнал «активный/идущий звонок» теряется на retrieval):**
- Проблема РАНЬШЕ генерации — в ретриве/ранжировании. bge-m3 dense + Meili вытягивают кластер «мероприятие/событие», нужные страницы (desktop-ug-calls / one-web-ug-calls «Звонки → Добавление участника в активный звонок») не попадают в топ, либо вытесняются cap-per-doc/context_limit=10.
- Усугубляющий фактор — **synonyms.json:13** (`infrastructure/docs_assistant/query_rewrite.py` грузит его): одна группа синонимов склеивает `вкс / видеоконференция / конференция / совещание / мероприятие / встреча / meeting`. То есть «мероприятие» и «конференция/ВКС» считаются одним смыслом. `звонок/вызов/call` — отдельная группа (:14). При включённом query_rewrite это прямо размывает грань «звонок в чате» vs «запланированное мероприятие». НО `docs_query_rewrite_enabled=False` по умолчанию (config.py:120) — на проде выражается через эмбеддинги, не через rewrite.
- Модификатор «идущий/активный» — короткое прилагательное, теряется в dense-эмбеддинге длинного вопроса; ни rerank-текст (heading_path⊕text), ни синонимы его не усиливают. Точки, где сигнал мог бы сохраниться, но не сохраняется: (a) нет буста по heading «активный звонок»; (b) cap-per-doc=4 + context_limit=10 могут вытеснить единственную правильную страницу, если «мероприятие»-кластер плотнее; (c) reranker off по умолчанию (`docs_rerank_enabled=False`), т.е. грубый RRF-порядок уходит в генерацию как есть.

## 3. Clarify-механизм Ф5 — устройство и точки калибровки

Пивот: решение «переспросить» принимает LLM-генерация через маркер `[[CLARIFY]]`, НЕ score-гейт.

- **Инъекция инструкции**: `CLARIFY_INSTRUCTION` — `docs_assistant.py:111-118`; добавляется к system-промпту ТОЛЬКО при `allow_clarify` (:307-308). Текст сейчас про «разные продукты (MCU/One/Mail) или разные функции» — про темпоральную двусмысленность (активный звонок vs запланированное мероприятие) НЕ говорит. **← ГЛАВНАЯ ТОЧКА КАЛИБРОВКИ ПРОБЛЕМЫ 2**: формулировку триггера надо расширить на «одна функция в разных состояниях/контекстах».
- **Парсинг маркера**: `_split_clarify_marker` (:121-131) и применение (:314-325) — если генерация начинается с `[[CLARIFY]]`, весь остаток = уточняющий вопрос, LLM повторно не зовётся, возвращается `DocsAnswer(clarify=True)`. `_strip_clarify_marker` (:134-139) — страховка от случайного маркера в обычном ответе.
- **Гейт флагом**: `docs_clarify_enabled` (config.py:176, ДЕФОЛТ False). Каналы прокидывают `allow_clarify` только когда флаг ON: веб `docs.py:254-259`, бот `bot_support.py:142-143`. Флаг OFF → `allow_clarify` не прокидывается, `CLARIFY_INSTRUCTION` не добавляется, поведение 1:1 как раньше.
- **Петля состояния**: `resolve_with_clarify` — `interface/api/docs_clarify.py:34-89`; кап уточнений `DOCS_CLARIFY_MAX_ASKS=2` (`infrastructure/db/repository.py:2023`); склейка ответа с прошлым вопросом (:66); TTL `docs_clarify_ttl_seconds=900` (config.py:186).

**Остатки старого score-гейта:**
- `decide_retrieval_action` — `domain/docs.py:316-351`. Ветка "clarify"-по-score УБРАНА (докстринг :326-329): теперь возвращает только "answer"|"not_found". `allow_clarify` в сигнатуре сохранён, но на решение не влияет. По верхнему score: `top < tau_floor` → "not_found", иначе "answer" (замечание: `tau_answer` в теле функции больше не используется — только tau_floor).
- Пороги τ живут: config.py `docs_clarify_tau_answer=0.55` (:180), `docs_clarify_tau_floor=0.30` (:183) → `service.py:118-119` → `DocsAssistant.__init__` (:223-224, :237-239) → используются в вызове (:288-293). Фактически задействован только `tau_floor` (анти-галлюцинация not_found); `tau_answer` — мёртвый параметр после Ф5.
- `build_clarify_question`/`_clarify_facet` (`domain/docs.py:275-313`) — детерминированный конструктор уточнения дизъюнкцией по product/section/heading. НЕ вызывается из `ask` (осталось от Ф2, теперь вопрос строит LLM). Кандидат в мёртвый код.

## 4. Eval / golden RAG#1

- CLI: `python -m helm.eval.docs_eval` — `src/helm/eval/docs_eval.py`. Пример прогона (:6-8):
  `docker exec helm-helm-1 python -m helm.eval.docs_eval --golden /app/golden_iva_rag1.json --k 5 --limit 200 --out /app/eval_run.json`.
  Флаги: `--golden`/env HELM_DOCS_GOLDEN, `--k`, `--limit`, `--compare` (sweep режимов semantic/hybrid/hybrid+rerank через `sweep.py`), `--judge` (платно), `--judge-model`, `--out`.
- Метрики: `runner.py` (`evaluate`/`aggregate`/`slices`) + `metrics.py`: `recall@k`, `hit@k`, `mrr`, `ndcg@k` (:69-108); `answer_in_context@k` / `context_hit@k` (покрытие key_facts в текстах топ-k, :202+). «Слепые» кейсы (чанки есть, ответа в них нет) отдельно печатаются (`docs_eval.py:84-92`).
- Judge (`--judge`): `judge.py` — `faithfulness`/groundedness, `answer_relevance`, `correctness` (vs ideal_answer; нет эталона → null). Промпт :43-52, парс :137-161.
- **Golden-формат**: `eval/golden.py:27-104`, `GoldenCase`. Поля кейса:
  `id` (`<slug>#<n>`), `query`, `relevant_doc_ids` (`[slug]` = DocChunk.slug), `product`/`section`/`source_url` (срезы), `qtype`/`role`/`difficulty` (оси срезов), `ideal_answer` (опц.), `ideal_answer_key_facts` (опц. `list[str]` — атомарные факты для answer_in_context@k). Парсер терпим, принимает массив или `{"rows":[...]}`.
- Golden-файлы (по постановке, лежат ВНЕ репо helm, монтируются в /app контейнера): `iva-rag1-docs/golden/golden_iva_rag1.eval200.json` (184 кейса с эталонами) и `golden_iva_rag1.json` (1306, без эталонов). В самом репо helm их нет (проверено find по репо) — только парсер/харнес.
- Для сборки ambiguous-кейсов «идущий звонок vs мероприятие»: кейс = `{id, query, relevant_doc_ids:[<slug активного звонка>], ideal_answer_key_facts:[...]}`. relevant_doc_ids должны указывать на slug'и desktop-ug-calls / one-web-ug-calls / connect-ios-ug-calls / connect-android-ug-calls (раздел «Добавление участника в активный звонок»).

## 5. Конфиг (все дефолты — src/helm/config.py, класс Settings, env-префикс HELM_)

- `docs_search_mode="hybrid"` (:108), `iva_docs_qdrant_collection="iva_docs__bge_m3_1024"` (:102), `iva_docs_tenant="iva"` (:104).
- `docs_query_rewrite_enabled=False` (:120), `docs_near_dup_dedup_enabled=False` (:121), `docs_rerank_enabled=False` (:122).
- `docs_max_citations=10` (:129), `docs_context_limit=10` (:134), `docs_max_chunks_per_doc=4` (:139).
- `docs_answer_max_tokens=700` (:145), `docs_known_gaps_enabled=True` (:152).
- `docs_rerank_floor=None` (:162), `docs_rerank_candidate_cap=20` (:170).
- clarify: `docs_clarify_enabled=False` (:176), `docs_clarify_tau_answer=0.55` (:180), `docs_clarify_tau_floor=0.30` (:183), `docs_clarify_ttl_seconds=900` (:186).

## Риски / наблюдения
- `tau_answer` фактически мёртв после Ф5 (в `decide_retrieval_action` не используется) — при калибровке не путать его с триггером clarify; триггер теперь в тексте `CLARIFY_INSTRUCTION`.
- synonyms.json склейка «мероприятие=конференция» — потенциальный вред при включении query_rewrite; для проблемы 1 стоит держать в уме даже при off.
- reranker и query_rewrite off по умолчанию — на проде retrieval = чистый RRF(dense+Meili) без переранжирования, самое слабое звено для тонких различий «активный vs запланированный».
- Проверить нельзя: реальные payload'ы Qdrant/Meili и фактические топ-k на этом вопросе (рантайм, не код).