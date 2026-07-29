---
title: report-rag1-ph5-clarify-llm
type: note
permalink: tacticum/00-board/report-rag1-ph5-clarify-llm
status: draft
tags:
- rag1
- clarify
- docs-assistant
- report
archived-at: 2026-07-29 18:12
---

# RAG#1 Ф5 — clarify по маркеру LLM (вместо score-гейта)

## Что и где
- **Worktree**: `/Users/bubblemac/tacticum/helm-wt-rag1-ph5-clarifyllm`
- **Ветка**: `feat/rag1-ph5-clarify-llm` (поверх `feat/rag1-ph4-evalexport`)
- **Коммит**: `0c5588b` — не мержено, не пушено.

## Реализация
**Маркер + инструкция** (`src/helm/application/docs_assistant.py`):
- `CLARIFY_MARKER = "[[CLARIFY]]"`, `CLARIFY_INSTRUCTION` (суффикс к SYSTEM_PROMPT, идёт после PLAIN_SUFFIX).
- Инструкция добавляется в системный промпт **только при `allow_clarify=True`**. При False промпт идентичен прежнему.
- `ask`: после генерации при `allow_clarify` парсим сырой вывод `_split_clarify_marker` — ведущий `[[CLARIFY]]` → `DocsAnswer(clarify=True, clarify_question=<очищенный вопрос>, text=тот же, evidence=(), reason="clarify")`. **LLM повторно не зовём** (calls==1). Иначе — обычный ответ, маркер срезается `_strip_clarify_marker` (страховка при False/пустом вопросе). Парсинг до `sanitize_markdown`; чистим только сам вопрос.

**decide_retrieval_action** (`src/helm/domain/docs.py`): ветка `clarify`-по-score убрана (мёртвая — скоры бимодальны). Сохранены `answer` и анти-галлюцинационный `not_found` (top < tau_floor → честный отказ без LLM). Параметр `allow_clarify` оставлен в сигнатуре для совместимости, на решение не влияет. `build_clarify_question` осталась (детерминированная, ещё под тестами), но из пайплайна ask больше не зовётся.

**Не тронуто**: стор `docs_clarify_pending`, оркестратор `interface/api/docs_clarify.py` (работает по `answer.clarify`), петли в роутерах, кап ≤2, флаг `docs_clarify_enabled` (дефолт False), двухконтурный allowlist, краткость Ф1, cap реранка Ф3.

## Тесты
- Юнит парсинга: `[[CLARIFY]] вопрос?` → чистый вопрос; обычный текст/пустой маркер → None; срез просочившегося маркера.
- `ask`: маркер → clarify=True (LLM зван 1 раз, evidence пуст, вопрос без маркера); нет маркера → обычный ответ; инструкция инъектится только при allow_clarify; при allow_clarify=False `system == SYSTEM_PROMPT`; просочившийся маркер при False срезан; not_found-гейт (score<floor) сохранён без LLM.
- Оркестратор `test_docs_clarify.py` (мок ask_fn): неоднозначный → pending+вопрос; ответ на уточнение → склейка → ответ; кап 2; дедуп messageId — **цел без изменений**.
- Domain-гейт: зона неоднозначности теперь → "answer".

**Результат**: целевые 59 тестов — pass. Полный `tests/application+interface+domain` — **905 passed, 19 skipped** (warnings предсуществующие). `ruff`/`mypy` по изменённым файлам — clean (прочие ошибки ruff/mypy — в несвязанных предсуществующих файлах, мои файлы не затронуты).

## Подтверждение
При `docs_clarify_enabled=False` роутеры не прокидывают `allow_clarify=True` → clarify-инструкция в промпт не идёт, маркер не парсится (а если просочится — срезается), decide_retrieval_action не меняет уверенные ответы. **Поведение 1:1 с прежним.** Не мержить.