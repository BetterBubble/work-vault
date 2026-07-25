---
title: report-rag1-ph4-evalexport
type: report
permalink: tacticum/00-board/report-rag1-ph4-evalexport
status: draft
tags:
- rag1
- eval
- implementer
- draft
---

# RAG#1 Ф4: экспортёр веб-истории DocsQa → golden-набор для docs_eval

## Изоляция
- Worktree: `/Users/bubblemac/tacticum/helm-wt-rag1-ph4-evalexport`
- Ветка: `feat/rag1-ph4-evalexport` (стек поверх `feat/rag1-ph3-rerankcap`)
- Коммит: `8fa300f`. НЕ мержил/пушил.

## Что сделано
1. `src/helm/infrastructure/db/repository.py` — новая read-функция `list_all_docs_qa(session, *, limit=None, with_feedback=False)`: вся история Q&A по всем пользователям, свежие сверху; опц. лимит; при `with_feedback` жадный `selectinload` фидбэка (без N+1/lazy вне сессии). Только чтение.
2. `src/helm/eval/docs_qa_export.py` — новый модуль:
   - Чистая `build_export(rows, *, with_feedback)` → `ExportResult` (кейсы + статистика n_raw/n_cases/n_duplicates/n_empty).
   - CLI `python -m helm.eval.docs_qa_export --out <path> [--limit N] [--with-feedback]` (или env `HELM_DOCS_QA_EXPORT_OUT`; без `--out` → stdout).
   - Своя сессия БД, `engine.dispose()` в finally.

## Формат вывода
JSON-документ `{"meta": {...}, "rows": [...]}` (парсер `helm.eval.golden` читает ключ `rows`). Кейс:
- `id` = `docsqa#<id>`
- `query` = текст вопроса
- `relevant_doc_ids` = уникальные `slug` из снимка цитат `DocsQa.citations[*].slug` (= `DocChunk.slug`)
- `tenant_id` (по умолчанию `iva`), `source: "docs_qa"`, `not_found: bool`
- `feedback: {"up": n, "down": m}` — только при `--with-feedback` и наличии оценок
- `ideal_answer` НЕ выставляется (эталонов нет).
Дедуп по нормализованному вопросу (нижний регистр + схлопнутые пробелы), остаётся самый свежий. Пустые вопросы отброшены. В `meta.note` — честная граница набора.

## Запуск (на сервере, как ингест)
`docker exec helm-helm-1 python -m helm.eval.docs_qa_export --out /app/docs_qa_live.json --with-feedback`

## Тесты
- `tests/eval/test_docs_qa_export.py` (9): пустая история→пусто; маппинг query+slug; дедуп; отброс пустых; not_found→пустые doc_ids; фидбэк только с флагом; выход парсится `parse_golden`.
- `tests/infrastructure/test_docs_qa_list_all.py` (3): пусто; порядок+лимит; жадный фидбэк (доступ после `expunge_all`).
- `pytest tests/eval tests/infrastructure` → 305 passed. `ruff` clean, `mypy` clean (изменённые файлы).
- Прим.: в infra-прогоне ResourceWarning `Event loop is closed` от teardown aiosqlite — baseline-шум харнесса, не падение.

## Ограничения (что измеримо)
Набор БЕЗ эталонных ответов → пригоден для retrieval-метрик (recall/hit/mrr/ndcg@k по `relevant_doc_ids` из снимка цитат) и латентности/длины. НЕ годится для judge-качества ответа (нужен курируемый golden с `ideal_answer`/`ideal_answer_key_facts`). `relevant_doc_ids` — снимок «что показали», не независимый эталон релевантности: возможен self-fulfilling bias. `not_found`-кейсы идут с пустыми doc_ids (полезны для латентности, вырождены для recall). Фидбэк 👎 в набор попадает как метка, но не корректирует relevant_doc_ids.