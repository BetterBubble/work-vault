---
title: report-rag1-clarify-calibv2
type: report
permalink: tacticum/00-board/report-rag1-clarify-calibv2
status: draft
role: implementer
---

# report-rag1-clarify-calibv2

## Задача
Заменить `CLARIFY_INSTRUCTION` в `src/helm/application/docs_assistant.py` на калиброванный v2-текст (17/18 на прод-LLM против 7/10 у v1) + обновить тест инструкции.

## Ветка / worktree
- Репо: `/Users/bubblemac/tacticum/helm`
- Ветка: `feat/rag1-clarify-calib-v2` (от `origin/main` @ 739f611)
- Worktree: `/Users/bubblemac/tacticum/helm-wt-rag1-calibv2`
- Коммит: `32d031f` — feat(rag1): калиброванный v2-текст CLARIFY_INSTRUCTION

## Изменения (кратко)
1. **`src/helm/application/docs_assistant.py`** — тело константы `CLARIFY_INSTRUCTION` заменено на v2-текст СЛОВО В СЛОВО из scratchpad. Стиль сохранён: parenthesized string concatenation, ведущий `\n\n`, маркер подставлен через `+ CLARIFY_MARKER +` (2 вхождения `[[CLARIFY]]` → обе через константу, литерал не хардкодился). Ничего кроме константы не тронуто (парсинг маркера, гейт, логика `ask` — без изменений).
   - Суть v2: главный признак двусмысленности — ТЕМПОРАЛЬНЫЙ СЦЕНАРИЙ. ТЕРМИН (ВКС/видеоконференция/конференция/совещание/встреча) сам по себе не различает сценарий → переспрос (даже при «идущий/идёт» рядом с ВКС). Слова «звонок/вызов» с явным маркером активности → отвечать про активный звонок. Явные маркеры (запланировать/создать/в календаре/заранее) → отвечать про мероприятие. Продукт/устройство сам по себе — не повод переспрашивать.
2. **`tests/application/test_docs_assistant.py`** — тест `test_clarify_instruction_covers_temporal_and_product_ambiguity` подогнан и усилен (не ослаблен): сохранены прежние ассерты (темпоральный триггер идущий/актив + запланир + мероприят/событ, конференц, iva mcu/iva one, «не переспрашивай»); добавлены проверки наличия маркера `CLARIFY_MARKER` в тексте и ключевого отличия v2 — `"вкс"` и `"не различают"` (термин сам по себе двусмыслен).

Примечание: `replace_symbol_body` Serena продублировала присваивание (оставила старое тело хвостом `) = (...)`) — вручную удалил дубликат, финал синтаксически корректен.

## Тесты
- Команда: `uv run python -m pytest tests/application tests/domain tests/interface -q`
- Результат: **920 passed, 19 skipped**, 9 warnings (предупреждения — чужой legacy: Starlette 422 deprecation, не связано с правкой).
- Целевой тест точечно: `test_clarify_instruction_covers_temporal_and_product_ambiguity` — 1 passed.
- Ruff: `uv run ruff check src/helm/application/docs_assistant.py tests/application/test_docs_assistant.py` → All checks passed.

## Границы
Тронуты только `CLARIFY_INSTRUCTION` и её тест. Не мержил/не пушил/не деплоил.