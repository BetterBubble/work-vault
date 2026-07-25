---
title: Техплан RAG#2 /analyst — синтез triva на /api/rag2/answer
type: note
permalink: tacticum/00-board/tekhplan-rag-2-analyst-sintez-triva-na-api-rag2-answer
status: TECH-PLAN
created: 2026-07-22 12:10
updated: 2026-07-22 12:10
project: helm / RAG#2
repo: helm
branch: feat/rag2-analyst-sintez
deadline: 2026-07-22 16:00
tags:
- plan
- tech
- rag2
- demo
- helm
- analyst
- triva
---

# Техплан RAG#2 `/analyst` — синтез triva (демо Роскосмос 16:00)

repo=helm (`/Users/bubblemac/tacticum/helm`), база `main` (clean). Ветка **`feat/rag2-analyst-sintez`**. PR+деплой `/opt/helm` — пользователь. Родительский план: [[plan-of-rekord-rag-2-analyst-mozg-plan-demo-roskosmos-16-00]].

## Что есть сейчас (grounded)
- `/api/rag2/context` (`interface/api/routers/rag2.py`) → `Rag2ContextOut` (`context` markdown с `[n]` из `render_rag2_context`, `citations`, `structural`, `as_of`, `degraded`, `answerable`). **LLM НЕ вызывается.**
- Оркестратор `Rag2Orchestrator.answer()` (`application/rag2.py`) → `Rag2Result` (hits Jira+Confluence+helm, граф, live-merge) = retrieved-контекст.
- `build_rag2_context()` (`infrastructure/rag2/service.py`) — сейчас без LLM.
- Готовый вызов triva: `TrivaLlm` (`infrastructure/assistant/service.py`) поверх `GatewayClient.chat()` (`llm/gateway.py`, `iva_llm_base_url/api_key/model`, туннель `127.0.0.1:8790/v1`, модель `triva`). `chat()` **не-стрим** — под ограничение triva.
- Эталон промпта/анти-галлюцинаций/guardrail — `application/docs_assistant.py`.
- helm-analyst тулы — in-process обёртки в `interface/mcp/analyst_server.py` над REST-роутерами (`arch_map`→`cio_router.arch_map`; `affected_systems`→`requirement_detail`+`_enrich_containers`+retriever; `requirement_tests`→`_allure_coverage`+`_requirement_test_bridge`; через `_with_session`).

## Эндпоинт-решение
**Новый `POST /api/rag2/answer`** = надмножество `Rag2ContextOut` + `synthesis: str|null`, `synthesis_failed: bool`, `topology: str|null`, `tests: str|null`. `/context` НЕ трогаем (живой фолбэк; тесты не переписываем). Фронт-фолбэк: `synthesis` пуст → рендерим `context` как сегодня.

## Порядок A→UI→B→C→D
**A — синтез triva (demo-critical).** Файлы: `infrastructure/rag2/service.py`, `application/rag2.py`, `interface/api/routers/rag2.py`, `config.py`.
1. В `Rag2Context` добавить `llm: TrivaLlm|None` (строить из `iva_llm_*` как `build_assistant_context`; нет креды → None, синтез выключен).
2. Роутер `answer()`: `result = run_in_threadpool(_answer, ...)`, `context_text = render_rag2_context(result.hits)`, `citations = build_rag2_citations(...)`.
3. Guardrail: `no_answer`/пустые hits → `synthesis=None`, вернуть как `/context` (LLM не зовём).
4. `synthesis = run_in_threadpool(ctx.llm.generate, system=SYNTH_PROMPT, user=...)` → `sanitize_markdown`; try/except → ошибка → `synthesis=None, synthesis_failed=True`.
5. `user_prompt` = вопрос + `context_text` (+ topology + tests). Синтез строго по этому блоку с `[n]`.
Конфиг: `rag2_synth_enabled`, `rag2_synth_max_tokens` (~800–1000), reuse `iva_llm_*`.

**SYNTH_PROMPT (набросок, system):** «Ты аналитик-планировщик ИВА. Отвечай СТРОГО по Контексту ниже (Jira/Confluence/требования helm с `[n]`, топология, покрытие). Ничего не выдумывай; нет данных — "нет данных в контексте". Каждый тезис — `[n]`. Русский, сжато, чистый Markdown. Рубрики: **Статус** · **Сделано** · **В процессе** (кто ведёт) · **Не начато/блокеры** · **Как технически сделать** (по топологии) · **Кому задача** (владелец из данных; нет → "уточнить владельца") · **Что уточнить** (1–3 вопроса). Про тесты — ТОЛЬКО "покрытие/что автоматизировать", НЕ обещать зелёные % и что тесты проходят.»

**UI (demo-critical).** `web/src/types.ts` (`Rag2AnswerOut`), `web/src/api.ts` (`rag2Answer`→POST `/api/rag2/answer`), `web/src/screens/AnalystChat.tsx`: `ask()` зовёт `rag2Answer`; над блоком цитат вставить панель синтеза `<AnswerText text={a.synthesis}/>` (реюз рендера markdown+[n]); ниже — существующие context/Источники/Граф. Индикатор: пока `answer===null` — **«Синтезирую план… (несколько секунд)»** (triva без стрима). Фолбэк: `synthesis` пуст/`synthesis_failed` → панель не рендерим, показываем `context`.

**B — топология (важно).** Роутер + reuse хелперов `mcp/analyst_server.py`: по тексту вопроса → `retriever.retrieve_rids` → `requirement_detail`+`_enrich_containers` через `_with_session` → компактный `topology` (контейнеры/владельцы/рёбра/контракты) в `user_prompt` + поле. Вынести общий хелпер (`infrastructure/rag2/topology.py`), чтобы роутер не тянул MCP-модуль. Fail-soft: ошибка → `topology=None`.

**C — Allure (stretch).** Для найденных rid/jira-key → `_allure_coverage(app)`; `available=False` → `tests=None`. Брать ТОЛЬКО счётчики ТК/`automated`(=0)/`matched_by`, НЕ «% зелёных». Свернуть в `tests` «покрытие/что автоматизировать».

**D — docs-корпус (stretch, ~30–40 строк, 5 файлов).** Включить существующую коллекцию **`iva_docs__bge_m3_1024`** в ретрив: `application/rag2.py::_corpora_hits` (добавить docs-слот к jira/confluence/helm), `infrastructure/rag2/service.py::build_rag2_context` (стор), `infrastructure/rag2/search.py::_doc_from_payload` (source-тег), `domain/rag2.py::federate`/`SOURCE_LABELS` (метка + вес в RRF). Риски: дубли docs↔confluence, вес docs в RRF, влияние на as_of.

## Обработка ошибок / таймаутов triva
- Guardrail-first: нет hits → синтез не зовём.
- **Bounded timeout:** ⚠️ `_call_with_retry` в gateway.py ретраит transient **6 раз** (backoff до 30с) → зависший triva копит минуты. Обернуть синтез в `asyncio.wait_for(run_in_threadpool(...), ~25с)` → исчерпание → `synthesis_failed=True`, фолбэк на context. Зафиксировать в PR как известный компромисс.
- Любое исключение synth → 200 с `synthesis=None, synthesis_failed=True` (не 5xx; retrieval валиден). B/C — свои try/except, fail-soft.

## Минимальный cut (если не успеваем к 16:00)
- **Demo-critical:** A (синтез на `/answer`) + UI (панель+«Синтезирую…»+фолбэк).
- **Важно:** B (топология).
- **Stretch:** C (Allure — только если снапшот `available=True`), D (docs).
- **Всегда жив:** `/iva-docs` (RAG#1) — не трогаем.

## Acceptance-хуки
- `POST /api/rag2/answer {query:"..."}` → `synthesis` непуст, рубрики на месте, `[n]` совпадают с `citations`; UI: панель над «Источники».
- Топология: `topology` непуст, синтез в «как сделать» ссылается на контейнеры.
- Тесты: `tests` только счётчики; в синтезе НЕТ «зелёных %»; снапшот пуст → поле отсутствует, без ошибок.
- triva-стабильность: искусственный таймаут → `synthesis_failed=true`, HTTP 200, UI показывает `context` (не зависает, не 5xx).
- Фолбэк `/iva-docs` отвечает независимо; `/answer` при несконфигуренном `iva_llm_*` = поведение `/context`.

## Риски
1. Стекинг ретраев gateway (~90с+) — гл. риск латентности → `asyncio.wait_for`.
2. Формат/качество Gemma — низкая temperature (0.2) + жёсткий промпт + `sanitize_markdown`; проверить 2–3 вопроса заранее.
3. B требует session_factory+retriever (`assistant_enabled`) — на прод-конфиге проверить, иначе fail-soft off.
4. Импорт-циклы: общий топологи-хелпер отдельно.
5. C — снапшот Allure на проде скорее всего `available=False` → C по факту no-op (ок).
6. `max_tokens` мал → обрежет рубрики; ~800–1000, env `HELM_RAG2_SYNTH_MAX_TOKENS`.

## Критические файлы
`interface/api/routers/rag2.py` · `infrastructure/rag2/service.py` · `infrastructure/assistant/service.py` (`TrivaLlm`) · `application/docs_assistant.py` (эталон промпта) · `interface/mcp/analyst_server.py` (хелперы B/C) · `web/src/screens/AnalystChat.tsx` (+`api.ts`,`types.ts`) · для D: `application/rag2.py`, `infrastructure/rag2/search.py`, `domain/rag2.py`.