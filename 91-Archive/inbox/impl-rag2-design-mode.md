---
title: impl-rag2-design-mode
type: report
permalink: tacticum/00-board/impl-rag2-design-mode
tags:
- rag2
- design-mode
- analyst
- demo
- roskosmos
- implementer
archived-at: 2026-07-29 18:12
---

# impl-rag2-design-mode

status: draft
from: implementer (RAG#2 /analyst)
to: director/teamlead
branch: feat/rag2-design-mode · worktree: /Users/bubblemac/tacticum/helm-wt-rag2-design
commit: 5a2ddb3

## Что сделано
DESIGN-режим синтеза для `/api/rag2/answer`: на вопрос-ПРОЕКТИРОВАНИЕ («как ДОБАВИТЬ в фичу X возможность Z») морда отвечает проектными рубриками, а не пересказом статуса. Две правки по КОЛ ГД + подмешивание прецедентов.

### 1. Детектор design vs status (чистый, domain)
`is_design_query(query)` в `src/helm/domain/rag2.py`.
- Триггеры: инфинитивы добавления после «как» (добавить/встроить/внедрить/реализовать/интегрировать/подключить/обеспечить/настроить/сделать…) ИЛИ императивные маркеры «куда встраивать», «что затронет», «кого подключить».
- Ключевое отличие от статуса: только ИНФИНИТИВ. «Как реализовАН вход через ЕСИА?» → status (форма результата). «Как реализовАТЬ …» → design.
- Статус-акты «Что по X сделано, что в работе, что не начато?» (6 канон-актов демо) под детектор НЕ подпадают → регресс-1:1.

### 2. SYNTH_PROMPT_DESIGN (роутер)
`SYNTH_PROMPT_DESIGN` в `src/helm/interface/api/routers/rag2.py` рядом с `SYNTH_PROMPT`. Рубрики: **Куда встраивать** (по топологии) · **Что затронет** (рёбра/контракты/смежные требования) · **Прецедент** (related_tasks/nearest_spec) · **Кому задача** (владелец контейнера) · **Что уточнить** (честные пробелы, в т.ч. «API-ручки нет»). Те же анти-галлюцинации + запрет LaTeX. Выбор промпта в `answer`: `SYNTH_PROMPT_DESIGN if design_q else SYNTH_PROMPT`. Status-путь не тронут.

### 3. Контекст для design-вопроса (блок D, fail-soft)
`build_design_context(app, query, enabled=)` в `src/helm/infrastructure/rag2/topology.py` (рядом с топологией/тестами). Собирается ТОЛЬКО для design-вопроса:
- прецеденты Jira (как `related_tasks` — broad-recall, фильтр source=jira);
- постановки-аналоги Confluence (как `nearest_spec` — source=confluence);
- детерминированный матч API (`api_registry_check` → есть операция METHOD+path, либо честное «готовой операции НЕ найдено» для рубрики «Что уточнить» — эталон ЕСИА: ручки «добавить IdP» нет).
Подмешивается во ВХОД синтеза отдельной секцией «ПРЕЦЕДЕНТЫ И СМЕЖНЫЕ ТРЕБОВАНИЯ». Под-таймаут (как топология), недоступность/нет матча/enabled=False → None (синтез и retrieval не страдают). Зовётся параллельно в `asyncio.gather` вместе с retrieval/topology/tests.

## Файлы
- `src/helm/domain/rag2.py` — `is_design_query` + маркеры (в `__all__`).
- `src/helm/interface/api/routers/rag2.py` — `SYNTH_PROMPT_DESIGN`, `_build_design_context`, выбор промпта + gather блока D в `answer`; `_synthesize`/`_build_synth_user_prompt` получили параметры `system`/`design`.
- `src/helm/infrastructure/rag2/topology.py` — `build_design_context` + хелперы строк (`_design_task_line`/`_design_spec_line`/`_design_api_line`).
- `tests/domain/test_rag2.py` — 3 теста детектора (design-акты True, статус-акты+«реализован» False, пусто False).
- `tests/interface/test_rag2.py` — 4 теста (+`_RecordingLlm.last_system`): design→design-промпт, status→status-промпт (регресс), блок D подмешан для design, НЕ собран для status.

## Числа тестов
- `tests/domain/test_rag2.py` + `tests/interface/test_rag2.py`: **63 passed** (0 failed).
- Смежные rag2 (domain+interface+analyst_mcp+synth_llm+orchestrator): **174 passed**.
- ruff: мои файлы (domain/rag2.py, topology.py, оба теста) — **All checks passed**. mypy: проверенные мои модули (domain/rag2.py, topology.py) — 0 ошибок.

## Как детектится design (кратко)
`is_design_query`: regex `\bкак\b[^.?!\n]{0,40}?<инфинитив-добавления>` ИЛИ маркеры «куда встраивать/добавлять», «что затронет», «кого подключить/привлечь». Эталон ЕСИА («Как нам добавить в IVA ID вход через ЕСИА? Куда встраивать, что затронет и кого подключить?») → True. Проверено на 3 кандидатах из `demo-design-question-candidates` + 6 статус-актах.

## ВАЖНО — конфликт со-жительства в worktree (нужно решение тимлида)
`src/helm/interface/api/routers/rag2.py` в ЭТОМ worktree одновременно правил другой implementer (conversation-слой: `conversation_id`, история). Его код (5 функций: `_render_rag2_history`, `_conversation_on`, `_history_fingerprint`, `_load_rag2_history`, `_store_rag2_turn` + правки `answer`/поля) присутствует в рабочем дереве, НЕ в HEAD, и переплетён с моим в общих `answer`/`_synthesize`/`_build_synth_user_prompt` — отделить в git нельзя. Поэтому мой коммит 5a2ddb3 НЕИЗБЕЖНО несёт и его WIP.
Его код сейчас СЛОМАН (не моя зона, я его не трогал):
- F821 / mypy name-defined: `hashlib`, `datetime`, `UTC`, `timedelta`, `repo` не импортированы;
- `Settings` не имеет `rag2_conversation_context_enabled/_turns/_ttl`;
- нет методов `repo.add_rag2_turn` / `repo.get_recent_rag2_turns`.
Это делает ruff/mypy на `rag2.py` НЕ чистыми — из-за его строк (591-645), не моих. В рантайме безопасно: память диалога off по умолчанию (`getattr(..., False)`), conv-пути в тестах не исполняются, 63/174 зелёные.

Две вынужденные правки в общем файле (нужны эндпоинту и моим тестам):
- восстановил утерянный им декоратор `@router.post("/rag2/answer", response_model=Rag2AnswerOut)` — без него весь /answer давал 404 (ломались и мои, и прежние тесты);
- добавил `from collections.abc import Sequence` — его код (и мой) используют, импорта не было.

Рекомендация: свести коммиты (conversation-слой достроить импорты/Settings/repo-методы ЛИБО откатить его WIP из этого файла), затем прогнать ruff/mypy на rag2.py целиком.
