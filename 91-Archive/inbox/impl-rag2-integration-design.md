---
title: impl-rag2-integration-design
type: report
permalink: tacticum/00-board/impl-rag2-integration-design
tags:
- rag2
- design-mode
- conversation
- integration
- analyst
- demo
- roskosmos
- implementer
archived-at: 2026-07-29 18:12
---

# impl-rag2-integration-design

status: draft
from: implementer (RAG#2 /analyst)
to: teamlead/director
branch: feat/rag2-integration — worktree /Users/bubblemac/tacticum/helm-wt-rag2-integration
commits: b5a9093 (DESIGN-режим) + 778d2a7 (fix генерации conversation_id) · base 993f1d2

## Задача 1 — DESIGN-режим (commit b5a9093)
Наложить DESIGN-режим синтеза на финальную ветку `feat/rag2-integration` (где conversation уже корректный), взяв ТОЛЬКО design-части из `feat/rag2-design-mode` (5a2ddb3). Целиком тот коммит мержить нельзя (тащил сломанный дубль conversation).

### Что наложено (по файлам)
- **src/helm/domain/rag2.py** (+65): `is_design_query()` + маркеры (`_DESIGN_ADD_VERBS`, `_DESIGN_HOWTO_RE`, `_DESIGN_MARKERS`), в `__all__`. Детект по ИНФИНИТИВУ добавления после «как» ИЛИ маркерам «куда встраивать/что затронет/кого подключить». Форма «как реализовАН» и 6 статус-актов «Что по X сделано…» → status (регресс-1:1).
- **src/helm/infrastructure/rag2/topology.py** (+102): `build_design_context(app, query, *, enabled)` + хелперы. Блок D: прецеденты Jira (related_tasks) + аналоги Confluence (nearest_spec) через `rag2_router.search` + `api_registry_check`. Fail-soft.
- **src/helm/interface/api/routers/rag2.py**: `SYNTH_PROMPT_DESIGN` + `_build_design_context()`; интеграция с conversation БЕЗ дублирования — `_synthesize`/`_build_synth_user_prompt` получили keyword `system`/`design`; в `answer` — `design_q = is_design_query(query)`, блок D в общий `asyncio.gather`, выбор `SYNTH_PROMPT_DESIGN if design_q else SYNTH_PROMPT`.
- Тесты: 3 на детектор (domain), 4 на выбор промпта + блок D (interface) + `_RecordingLlm.last_system`.
- config.py для design НЕ трогал (в 5a2ddb3 его не было; переиспользован `topology_timeout_s`).

### Эталон АКТ 1b (ЕСИА)
«Как нам добавить в IVA ID вход через ЕСИА? Куда встраивать, что затронет и кого подключить?» → is_design_query=True, выбор SYNTH_PROMPT_DESIGN + блок D (юнит-тесты). Живой прогон — за лидом.

## Задача 2 — fix генерации conversation_id (commit 778d2a7)
На проде первый запрос БЕЗ `conversation_id` возвращал `conversation_id=null` → фронт-лента не получала id из эхо-поля и цепочка диалога не стартовала. Fix: `/answer` при включённой памяти (флаг + БД) и отсутствии входящего id ГЕНЕРИТ новую нить (`uuid4().hex`) и возвращает её эхом; фронт подхватывает → follow-up шлёт этот id → контекст работает.

### Правки
- **_conversation_on**: теперь проверяет ТОЛЬКО флаг + фабрику сессий (не требует id). Сигнатура сменилась (убран параметр conversation_id) — единственный вызов в `answer` обновлён.
- **answer**: `conversation_id = incoming_id or (uuid4().hex if conv_on else "")`. Историю тянем ТОЛЬКО при ВХОДЯЩЕМ id (свежая нить заведомо пуста → лишний запрос к БД не делаем).
- **Кэш пресетов не сломан**: при пустой истории кэш-ключ без hist-суффикса → тёплый кэш-хит работает как для одиночного; ход пишется после ответа (в т.ч. на кэш-хите). Флаг off / нет БД → conv_on False → эхо null, нить не заводится (поведение как раньше).
- **config.py**: подтверждён дефолт `rag2_conversation_context_enabled=True` (env `HELM_RAG2_CONVERSATION_CONTEXT_ENABLED`; на проде не задан → память on, потому стор и писал ходы). Env на прод задавать НЕ обязательно — дефолт уже on. Комментарий обновлён под генерацию id.

### Тесты (test_rag2_conversation.py)
- новые: первый запрос без id → сгенерён новый id в эхо + ход записан под ним; e2e (id из эхо первого ответа → follow-up с ним понимает контекст: история в промпте + обогащённый ретрив); флаг off → эхо null, ход не пишется.
- обновлён регресс-тест: id-less запрос при вкл. памяти заводит НОВУЮ нить на каждый запрос (эхо непустое, id1≠id2), кэш/промпт как для одиночного (llm.calls==1, нет блока истории).

## Числа тестов (venv helm, PYTHONPATH на worktree)
- test_rag2_conversation.py: **16 passed**, 0 failed.
- Широкий регресс `-k "rag2 or docs or conversation"` по всему tests: **477 passed**, 1350 deselected, 0 failed.

## Чистота
- ruff на rag2.py (domain+router) + topology + config + все тесты: **All checks passed (clean)**.
- mypy на domain/rag2.py, router/rag2.py, topology.py, config.py: **0 ошибок в самих файлах**. Остаточные 21 ошибка mypy — пред-существующие в ЧУЖИХ модулях (req_matrix, gateway, cio, analyst_server), одинаковы на baseline HEAD, мной не внесены.

## Статус
Коммиты b5a9093, 778d2a7 в feat/rag2-integration. НЕ пушил / НЕ мержил / НЕ деплоил — за пользователем/лидом после ревью. Env `HELM_RAG2_CONVERSATION_CONTEXT_ENABLED` задавать не обязательно (дефолт True).
