---
title: report-rag1-ph1-brevity — Ф1 реализована (draft)
type: report
permalink: tacticum/00-board/report-rag1-ph1-brevity-f1-realizovana-draft
tags:
- rag1
- implementer
- phase1
- review-needed
---

# RAG#1 бот — Фаза 1 (краткость + параллелизация). Отчёт implementer

status: draft — на ревью ГД. НЕ мержено, НЕ запушено, НЕ задеплоено.

## Worktree / ветка
- Worktree: `/Users/bubblemac/tacticum/helm-wt-rag1-ph1`
- Ветка: `feat/rag1-ph1-brevity` (от main = eaf10f8)
- Коммиты: `45b3aec` (Ф1a), `ca57a0d` (Ф1b)

## Ф1a — сжатый промпт + max_tokens
`src/helm/application/docs_assistant.py` (SYSTEM_PROMPT):
- Блок ПОЛНОТА («дай РАЗВЁРНУТЫЙ пошаговый ответ… не сокращай инструкцию») удалён.
- Вставлен блок КРАТКОСТЬ: структура **TL;DR (1-2 строки) → короткие шаги-буллеты (только действия) → цитаты [n]**; для простых вопросов шаги можно опустить.
- СОХРАНЕНЫ без изменений: правило цитат `[n]`, anti-hallucination «не нахожу», блок ФОРМАТ/анти-LaTeX, `PLAIN_SUFFIX`.

`src/helm/config.py`: `docs_answer_max_tokens` 1536 → **700**. Проверено: env `HELM_DOCS_ANSWER_MAX_TOKENS` переопределяет (env_prefix `HELM_`), быстрый откат без деплоя. Дефолт после правки подтверждён = 700.

## Ф1b — параллелизация Qdrant ∥ Meili
`src/helm/infrastructure/docs_assistant/search.py` (`_hybrid`):
- Meili (нужен только текст) стартует в `ThreadPoolExecutor(max_workers=1)` СРАЗУ — параллельно `embed`+Qdrant. embed→store.search остаются последовательны (Qdrant нужен вектор), но идут параллельно уже запущенному Meili. Sync httpx отпускает GIL на I/O.
- Инварианты сохранены: сбой `embed` (None→[]) / Qdrant пробрасывается наружу; сбой Meili → semantic-only на `dense` (лог warning). RRF-слияние (`fusion.py`) не тронуто.
- Добавлен импорт `from concurrent.futures import ThreadPoolExecutor`.
- Executor локальный в методе (разрешённый вариант). Общий через конструктор не делал: локальный безопаснее по конкурентности (router уже гоняет search в run_in_threadpool; свой поток на запрос, без глобального лимита воркеров) и без вопросов lifecycle/shutdown.

Поведенческое замечание: Meili теперь вызывается и в вырожденных случаях (embed вернул None / Qdrant упал), т.к. стартует раньше. На корректность результата не влияет (инварианты те же), тесты зелёные.

## Тесты (venv основного репо, PYTHONPATH на worktree/src)
- pytest: **583 passed, 17 skipped** (infrastructure + interface + application + domain). Целевой `tests/infrastructure/test_docs_search.py` — зелёный.
- ruff: **All checks passed** (по 3 изменённым файлам).
- mypy: **Success: no issues found in 3 source files**.

## Сомнения / на внимание ГД
- **max_tokens=700 может резать длинные инструктивные ответы.** Сам не повышал (в скоупе). Если после калибровки на реальном корпусе увидим обрывы пошаговых инструкций — быстрый откат `HELM_DOCS_ANSWER_MAX_TOKENS=1536` без деплоя; либо подобрать промежуточное значение.
- Краткость промпта и лимит токенов взаимодействуют: если TL;DR+буллеты окажутся слишком куцыми для сложных сценариев — тюнить оба параметра совместно (отдельная задача, не Ф1).
- Скоуп строго Ф1a/Ф1b: кэш/стриминг/clarify не трогал.
