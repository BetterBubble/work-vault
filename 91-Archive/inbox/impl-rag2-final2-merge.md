---
title: impl-rag2-final2-merge
type: report
permalink: tacticum/00-board/impl-rag2-final2-merge
tags:
- rag2
- merge
- fallback
- conversation
- design
- final
- implementer
archived-at: 2026-07-29 18:12
---

# impl-rag2-final2-merge

status: draft
from: implementer (RAG#2 /analyst)
to: teamlead/director
branch: feat/rag2-final2 — worktree /Users/bubblemac/tacticum/helm-wt-rag2-final
merge-commit: d5b4160 (свёл feat/rag2-synth-fallback 0ea9fde в базу bf8bdd6)

## Задача
Завершить мерж `feat/rag2-synth-fallback` в `feat/rag2-final2` (финальная ветка на прод). Конфликт: fallback-цепочка primary→fallback правит `_synthesize`, где уже conversation-история + выбор design-промпта.

## Разрулено (3 конфликтных файла)

### src/helm/config.py
Слиты ОБА набора настроек: conversation (`rag2_conversation_context_enabled/_turns/_ttl`) + fallback (`rag2_synth_fallback_llm_base_url/_api_key/_model`, `rag2_synth_primary_timeout_s`). Ничего не потеряно.

### src/helm/interface/api/routers/rag2.py
Ключевое сведение в `_synthesize` = **выбор промпта (design/status) + история (conversation) + цепочка primary→fallback вместе**:
- `_run_synth_llm` (введён fallback-веткой) получил параметры `history` / `system` / `design` и прокидывает их в `_build_synth_user_prompt` + `llm.generate(system=...)`. Раньше он хардкодил `SYNTH_PROMPT` и не знал про историю/design.
- `_synthesize` передаёт `history=history, system=system, design=design` в ОБА вызова `_run_synth_llm` — и primary (bounded `synth_primary_timeout_s`), и fallback (`synth_timeout_s`). То есть design-вопрос идёт с `SYNTH_PROMPT_DESIGN` + блоком D, follow-up — с историей, на ЛЮБОМ звене цепочки.
- Обратная совместимость сохранена: `llm_fallback is None` → одиночный primary под общим guard (как раньше).

### tests/interface/test_rag2.py
Сохранены ОБА набора тестов: design (4 — выбор промпта + блок D) и fallback (6 — primary-ok / error→fallback / empty→fallback / timeout→fallback / оба-fail→synthesis_failed / primary-only-без-fallback). Хелперы слились чисто: `_context(llm_fallback=, settings=)`, `_FakeLlm(fail=/sleep=/calls)`, `_RecordingLlm(last_system/last_user)` — сосуществуют.

### Авто-мерж (без конфликта, из fallback-ветки)
`service.py` (`Rag2Config.synth_fallback_llm_*`, `synth_primary_timeout_s`, `Rag2Context.llm_fallback`, `synth_fallback_llm_resolved`, `build_rag2_context` со вторым портом), `gateway.py` (`GatewayClient(chat_retry=False)` fast-fail), `test_rag2_synth_llm.py` — приняты как есть.

## Числа тестов (venv helm, PYTHONPATH на worktree)
- Целевые файлы (test_rag2.py + test_rag2_conversation.py + domain/test_rag2.py + test_rag2_synth_llm.py + llm/test_gateway.py): **117 passed**, 0 failed.
- Широкий регресс `-k "rag2 or docs or conversation or gateway"` по всему tests: **535 passed, 2 skipped**, 1313 deselected, 0 failed. (2 skipped — не связанные с задачей.)

## Чистота
- ruff на rag2.py + config.py + service.py + gateway.py + оба теста: **All checks passed (clean)**.
- mypy на routers/rag2.py, config.py, service.py: **0 ошибок в самих сведённых файлах**. Остаточные 21 ошибка mypy — пред-существующие в ЧУЖИх модулях (req_matrix, gateway.py:121, cio, analyst_server), одинаковы на baseline, мной не внесены (0 новых).

## Статус
Merge-коммит d5b4160 в feat/rag2-final2. НЕ пушил / НЕ мержил в main / НЕ деплоил — за лидом. Целевая прод-env для fallback: primary=triva на `iva_llm_*`, fallback=`HELM_RAG2_SYNTH_FALLBACK_LLM_*` (gateway `tacticum/cheap`); conversation `HELM_RAG2_CONVERSATION_CONTEXT_ENABLED` дефолт True.
