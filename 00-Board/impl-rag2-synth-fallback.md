---
title: impl-rag2-synth-fallback
type: report
permalink: tacticum/00-board/impl-rag2-synth-fallback
status: draft
tags:
- rag2
- implementer
- synth
- fallback
---

# impl-rag2-synth-fallback

status: draft · роль: implementer · ветка `feat/rag2-synth-fallback` (worktree `helm-wt-rag2-fallback`, от main e7f075e) · commit **0ea9fde**

## Что сделано
Синтез «мозг-плана» на `POST /api/rag2/answer` теперь работает по цепочке **primary → fallback**: primary-LLM пробуется под коротким bounded-таймаутом; при недоступности / ошибке / таймауте / пустом результате синтез **автоматически** уходит на fallback-LLM. Целевой прод-режим: primary = triva (приватная в контуре ИВА), fallback = gateway `tacticum/cheap`.

Обратная совместимость: **fallback не задан → поведение 1:1 как раньше** (одиночный primary/`synth_llm_resolved` под общим `synth_timeout_s`, без цепочки). Guardrail-first (нет hits → LLM не зовём), синтез-guard (общий таймаут), кэш синтез-ответов, conversation-кэш — не тронуты.

## Файлы (все в worktree `/Users/bubblemac/tacticum/helm-wt-rag2-fallback`)
- `src/helm/config.py` — новые настройки:
  - `rag2_synth_fallback_llm_base_url` / `_api_key` / `_model` (env `HELM_RAG2_SYNTH_FALLBACK_LLM_*`, дефолт None);
  - `rag2_synth_primary_timeout_s: float = 12.0` (bounded попытка primary; применяется ТОЛЬКО при сконфигуренном fallback).
- `src/helm/infrastructure/rag2/service.py`:
  - `Rag2Config` — поля `synth_fallback_llm_*`, `synth_primary_timeout_s`; свойство `synth_fallback_llm_resolved -> tuple|None` (None → fallback выключен; model фолбэчит на `iva_llm_model`);
  - `Rag2Context.llm_fallback: TrivaLlm | None = None` (дефолт None — старые сборки/тесты целы);
  - `build_rag2_context` — при заданном fallback строит второй порт (gateway cheap) и primary в fast-fail режиме; без fallback — primary как раньше.
- `src/helm/llm/gateway.py` — `GatewayClient(chat_retry=False)`: `chat` минует `_call_with_retry` (6 ретраев, backoff до 30с) и строит OpenAI-клиент с `max_retries=0` → connection refused / таймаут отдаёт управление мгновенно, не копя ретраи. Дефолт `chat_retry=True` — прочие клиенты не затронуты.
- `src/helm/interface/api/routers/rag2.py` — `_synthesize` реализует цепочку (вынесен `_run_synth_llm` с параметром `timeout_s`): primary под `synth_primary_timeout_s`, fallback под `synth_timeout_s`; оба упали/пусты → исключение/пустая строка наружу → эндпоинт ставит `synthesis_failed=true` (как раньше), HTTP 200.
- `tests/interface/test_rag2.py` — helper `_context(..., llm_fallback=...)` + 6 тестов цепочки.
- `tests/infrastructure/test_rag2_synth_llm.py` — helper `_settings(fallback_*)` + 6 тестов конфига/сборки портов.

## Как логируется, какой LLM сработал
Логгер `helm.interface.api.routers.rag2` (`_synthesize`):
- primary успех: `INFO` «RAG#2 синтез: отработал primary LLM»;
- primary упал/таймаут: `WARNING` «RAG#2 синтез: primary недоступен (<exc>) → fallback LLM»;
- primary пуст: `WARNING` «RAG#2 синтез: primary вернул пусто → fallback LLM»;
- fallback успех: `INFO` «RAG#2 синтез: отработал fallback LLM»;
- без fallback: `INFO` «... отработал primary LLM (fallback не сконфигурирован)».

## Целевая env-конфигурация (primary triva + fallback cheap)
Primary (triva) остаётся на общих `iva_llm_*` — их НЕ трогаем, `rag2_synth_llm_*` НЕ задаём (тогда `synth_llm_resolved` = `iva_llm_*` = triva):
```
HELM_IVA_LLM_BASE_URL=<endpoint triva в контуре, напр. http://triva.internal:8004/v1>
HELM_IVA_LLM_API_KEY=<ключ triva>
HELM_IVA_LLM_MODEL=triva
```
Fallback (gateway cheap):
```
HELM_RAG2_SYNTH_FALLBACK_LLM_BASE_URL=<url gateway, напр. https://llm.cifragen.ru/v1>
HELM_RAG2_SYNTH_FALLBACK_LLM_API_KEY=<project-hub ключ тенанта>
HELM_RAG2_SYNTH_FALLBACK_LLM_MODEL=tacticum/cheap
```
Опционально: `HELM_RAG2_SYNTH_PRIMARY_TIMEOUT_S=12` (короткая попытка мёртвой triva до ухода на fallback).

Итог включённого режима: triva лежит (connection refused) → primary fast-fail → fallback `tacticum/cheap` синтезирует план; `synthesis_failed` только если ОБА недоступны.

## Проверка (числа)
- `tests/interface/test_rag2.py` + `tests/infrastructure/test_rag2_synth_llm.py`: **42 passed** (из них 12 новых: 6 цепочки на /answer — primary-ok / error→fallback / empty→fallback / timeout→fallback / оба-fail→synthesis_failed / primary-only-без-fallback; 6 конфиг/сборка портов).
- Смежные rag2/gateway/llm/assistant (`-k "rag2 or gateway or llm or assistant"`): **452 passed, 2 skipped** (2 warning — не связанный aiosqlite teardown в чужих тестах).
- ruff: clean (изменённые файлы). mypy: clean по изменениям (остаётся 1 пред-существующий baseline `gateway.py:121 no-any-return` в НЕтронутой `_batch_limit_from_error`).

## Границы соблюдены
Тронуты только rag2-синтез/config/gateway. conversation/design/docs-corpus/tests-матч, `/context`, RAG#1 — не трогал. Не пушил, не мержил, не деплоил.
