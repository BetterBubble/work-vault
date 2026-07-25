---
title: impl-rag2-analyst-answer-backend
type: note
permalink: tacticum/00-board/impl-rag2-analyst-answer-backend
tags:
- rag2
- analyst
- triva
- demo
- helm
- backend
- implementer
---

# impl-rag2-analyst-answer-backend

status: draft (готово к ревью тимлида)
ветка: `feat/rag2-analyst-sintez` · worktree `/Users/bubblemac/tacticum/helm-wt-rag2-analyst-be`
база: main `9571b1f` · autonomy OFF (не пушено, не мержено, не деплоено)

## Коммиты
- `4b3a7f3` — A: синтез мозг-плана triva на `POST /api/rag2/answer`
- `0bb085d` — B/C: топология C4 + покрытие автотестами в `/answer`

## Что сделано

### A — синтез triva (demo-critical) ✅
Новый `POST /api/rag2/answer` — надмножество `/context` (все поля 1:1 для фолбэка) + `synthesis`/`synthesis_failed`/`topology`/`tests`. `/context` НЕ тронут.
- `Rag2Context.llm: TrivaLlm|None` — синтез через внутренний vLLM `triva` (`iva_llm_*`, туннель 127.0.0.1:8790). Нет креды / `rag2_synth_enabled=false` → None → `/answer` ведёт себя как `/context`.
- Промпт `SYNTH_PROMPT` с жёсткими рубриками: **Статус → Сделано / В процессе / Не начато → Как технически сделать → Кому задача → Что уточнить**; строго по контексту, цитаты `[n]` только на переданные источники (нумерация = `render_rag2_context`/`build_rag2_citations`, оба через `dedup_by_document` → согласовано с `citations`). Отдельная оговорка «про тесты — только покрытие, не зелёные %».
- Guardrail-first: нет hits / `no_answer` / `llm is None` → triva НЕ зовём, `synthesis=null`, `synthesis_failed=false`.
- Таймаут-guard: `asyncio.wait_for(rag2_synth_timeout_s=25с)` поверх `run_in_threadpool` — защита от стекинга ретраев Gateway (`_call_with_retry`, 6 попыток, до ~90с+). Ошибка/таймаут/пустой ответ → `synthesis_failed=true`, `synthesis=null`, retrieval валиден → **HTTP 200** (никогда 5xx/зависания).
- `sanitize_markdown` на выходе; `temperature=0.2`; `max_tokens=900` (env `HELM_RAG2_SYNTH_MAX_TOKENS`).

### B — топология C4 (важно) ✅
`infrastructure/rag2/topology.py::build_topology_summary` — fail-soft обёртка над хелперами `analyst_server`: по тексту запроса `retriever.retrieve_rids` → похожие требования → `requirement_detail` + `_enrich_containers` через `_with_session`. Компактная проза (контейнер [tech] — владелец · линии · архриски · релизы · компоненты) идёт И в промпт синтеза (рубрика «как сделать»), И в поле `topology`. Нет БД/ретривера/матча → None.

### C — Allure (stretch) ✅
`build_tests_summary` — ТОЛЬКО счётчики (`test_cases`/`automated`/`matched_by`) из `_allure_coverage`. БЕЗ «зелёных %». Снапшот `available=False` / нет матча → `tests=null`.

### D — docs-корпус (stretch) ❌ ОСОЗНАННО ОТРЕЗАН
Не делал: самый рискованный (дубли docs↔confluence, веса RRF, риск сломать живой ретрив RAG#2) при минимальной ценности для демо мозг-плана. Приоритет «режь при нехватке времени».

## Точная форма ответа `/api/rag2/answer`
Все поля `Rag2ContextOut` (`query, mode, context, citations[], structural, as_of, degraded, needs_confluence_body, disclaimer, answerable`) ПЛЮС:
- `synthesis: str|null` — мозг-план прозой с `[n]`
- `synthesis_failed: bool`
- `topology: str|null` — **строка** (компактная проза, не объект)
- `tests: str|null` — **строка** (счётчики покрытия)

## Изменённые/новые файлы
- `src/helm/interface/api/routers/rag2.py` — `Rag2AnswerOut`, `SYNTH_PROMPT`, `_synthesize`, `_build_synth_user_prompt`, `_build_topology`, `_build_tests`, эндпоинт `answer`
- `src/helm/infrastructure/rag2/service.py` — `Rag2Context.llm`, сборка `TrivaLlm`, поля синтеза в `Rag2Config`
- `src/helm/infrastructure/rag2/topology.py` — НОВЫЙ (B/C)
- `src/helm/config.py` — `rag2_synth_enabled/max_tokens/temperature/timeout_s`
- `tests/interface/test_rag2.py` — 7 новых тестов `/answer`

## Тесты — числа
`tests/interface/test_rag2.py`: **13 passed, 0 failed** (7 новых: успех-синтез, guardrail-no-hits→не зовёт triva, ошибка→failed+fallback+200, таймаут→failed+200, `llm=None`→как /context, 503, 422).
Смежные (не сломаны): `test_rag2_orchestrator` + `domain/test_rag2` + `test_rag2_search` = **83 passed**.
ruff — clean; mypy (rag2.py + topology.py + service.py) — Success no issues.

## Известные компромиссы (для PR/ревью)
1. Слой `infra→interface`: `topology.py` (infrastructure) импортирует `interface.mcp.analyst_server` + `cio` — лениво, внутри функций. Оправдано переиспользованием уже рабочих in-process хелперов под дедлайн; альтернатива (дублировать логику requirement_detail/_enrich_containers) дороже и рискованнее.
2. Таймаут-guard прерывает ожидание, но фоновый поток triva досчитывает (нельзя убить) — приемлемо для демо; ответ возвращается вовремя.
3. B/C зависят от `assistant_enabled` (Gateway+vLLM+Qdrant) и `session_factory` на проде; иначе fail-soft off (None).

## Что осталось / для тимлида
- Прогнать 2-3 живых вопроса против реальной triva заранее (проверить формат Gemma/рубрики) — риск качества, не покрывается юнит-тестами (LLM замокан).
- На проде проверить: `iva_llm_*` сконфигурен → синтез живой; снапшот Allure скорее `available=False` → `tests=null` (ок).
- Фронт пишется параллельно под зафиксированный контракт (`topology`/`tests` = строки).
