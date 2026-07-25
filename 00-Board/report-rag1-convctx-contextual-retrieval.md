---
title: report-rag1-convctx-contextual-retrieval
type: note
permalink: tacticum/00-board/report-rag1-convctx-contextual-retrieval
tags:
- rag1
- docs-assistant
- implementer
- report
---

# report-rag1-convctx-contextual-retrieval

status: draft
роль: implementer

## Задача
Баг с прода: conversation-context не помогал follow-up'ам, т.к. RETRIEVAL шёл по сырому текущему вопросу. История входила только в промпт генерации, не в ретрив. Пример: «Как создать мероприятие?» → «у меня iOS» → search("у меня iOS") находил общие iOS-страницы → guardrail not_found.

## Ветка / worktree
- Ветка: `feat/rag1-convctx-contextual-retrieval` (от origin/main = df6e2fc)
- Worktree: `/Users/bubblemac/tacticum/helm-wt-rag1-convctx-retr`
- Коммит: `247d31a`

## Изменения (diff кратко)
`src/helm/application/docs_assistant.py` — метод `DocsAssistant.ask`:
1. После known-gaps-гейта (остался на сыром question) вычисляем `search_query`: при непустой `history` — склейка `" ".join(прошлые вопросы в хронологии) + " " + question`; иначе `search_query = question`.
2. `self._search.search(question, ...)` → `self._search.search(search_query, ...)`.
3. Реранк `self._reranker.rerank(question, ...)` → `rerank(search_query, ...)` (ранжируем по тому же контекстному запросу).

Не тронуто: known_gaps (сырой question), `_build_user_prompt`/`_render_history` (сырой question + блок истории), clarify-логика, guardrail, confidence-гейт. При пустой/None history — search_query == question, поведение 1:1.

Тип history без изменений: `Sequence[tuple[str, str]] | None` (сигнатура ask прежняя).

## Тесты
`tests/application/test_docs_assistant.py`:
- `test_history_builds_contextual_search_query` — при непустой history `search.search` вызывается с контекстным запросом «Как создать мероприятие? а на телефоне? у меня iOS».
- `test_empty_or_none_history_searches_raw_question` — при None и `[]` history поиск по сырому вопросу.
- `test_history_contextual_query_also_used_for_rerank` — реранк получает тот же контекстный запрос (расширил `_RecordingReranker` полем `received_query`).

## Прогон
Команда: `uv run --project . pytest tests/application tests/domain tests/interface -q`
Результат: **923 passed, 19 skipped** (warnings — предсуществующие deprecation/event-loop, не связаны с правкой).
Ruff: `ruff check` по изменённым файлам — All checks passed.

## Границы
Только контекстный ретрив/реранк в `ask()` + тесты. Не мержил, не пушил, не деплоил.
