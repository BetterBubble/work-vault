---
title: report-rag1-max-asks-1
type: report
permalink: tacticum/00-board/report-rag1-max-asks-1
status: draft
---

# report-rag1-max-asks-1

## Ветка
`feat/rag1-clarify-max-asks-1` (worktree: `/Users/bubblemac/tacticum/helm-wt-rag1-maxasks`, база `origin/main` 6b8b64b). Коммит `9ee65f8`. НЕ мержено/не пушено.

## Задача
Снизить кап уточнений `DOCS_CLARIFY_MAX_ASKS` 2 → 1. Политика «спросить один раз, затем ответить» — лучший UX саппорт-бота (доказано сквозным тестом на проде: при cap=2 бот на «создать мероприятие»→«у меня iOS» переспрашивает дважды; при cap=1 после одного уточнения склейка+контекстный ретрив дают ответ).

## Что изменено
1. `src/helm/infrastructure/db/repository.py` (~2023): константа `DOCS_CLARIFY_MAX_ASKS = 2` → `= 1`. Комментарий над ней обновлён: «одно уточнение, затем отвечаем как есть». Строки 2072/2075 (кламп `ask_count` в `1..DOCS_CLARIFY_MAX_ASKS`) не трогал — логика корректна при любом капе.
2. `tests/interface/test_docs_clarify.py`: cap-тест переведён с 2 уточнений на 1.
   - `test_cap_two_clarifications_then_forced_answer` → `test_cap_one_clarification_then_forced_answer`.
   - Скрипт ассистента `[clarify, clarify, answer]` → `[clarify, answer]`. Теперь: 1-е входящее → уточнение, `ask_count==1` (кап исчерпан); 2-е входящее → `allow_clarify=False` (форс), `clarify is False`, pending снят. Смысл сохранён — проверяем, что после исчерпания капа бот форсированно отвечает и ожидание очищается.
   - Docstring модуля: «кап ≤2» → «кап=1».
   - `test_ambiguous_then_answer_merges_and_clears` не менял: он и так делает ровно 1 уточнение + ответ, проходит под cap=1 без правок.

## Границы
Логику clarify (`resolve_with_clarify`), `ask()`, guardrail, conversation-context — не трогал. Комментарии «≤2» в `src/helm/interface/api/docs_clarify.py` (строки 11, 88) оставил как есть — вне заявленного объёма (только константа + зависящие тесты); они не влияют на поведение, но при желании тимлид может освежить формулировку.

## Тесты
Команда: `PYTHONPATH=src <helm.venv>/python -m pytest tests/application tests/domain tests/interface -q`
(venv воркти отсутствует → импорт helm направлен на src воркти через PYTHONPATH, основной editable-install указывает на src главного репо).

Результат: **923 passed, 19 skipped** (warnings — предсуществующие deprecations FastAPI/asyncio, не связаны с правкой). Отдельно `test_docs_clarify.py`: 6 passed. Ruff по затронутым файлам: clean.