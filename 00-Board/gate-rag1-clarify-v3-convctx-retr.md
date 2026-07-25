---
title: gate-rag1-clarify-v3-convctx-retr
type: note
permalink: tacticum/00-board/gate-rag1-clarify-v3-convctx-retr
tags:
- gate
- rag1
- controller
- docs_assistant
---

# Гейт: feat/rag1-clarify-v3 + feat/rag1-convctx-contextual-retrieval

Контролёр (read-only). База обеих веток = origin/main `df6e2fc` (задеплоено). Каждая = ровно 1 коммит поверх origin/main.

## Ветка 1 — feat/rag1-clarify-v3 (01d4f06) — ВЕРДИКТ: PASS

- **Гит-чистота: PASS.** `git status` чист. diff --stat: только `src/helm/application/docs_assistant.py` (+51/-37, 72 стр.) и `tests/application/test_docs_assistant.py` (+11/-5). Секретов/.env/бинарников/мусора нет. AI-подписей в теле коммита нет.
- **Скоуп: PASS.** Весь diff docs_assistant.py — внутри одной строковой константы CLARIFY_INSTRUCTION (хунк @@ -109,39 +109,47 @@, ничего вне неё). `grep -c "CLARIFY_INSTRUCTION ="` = 1. ast.parse OK. Тест трогает только `test_clarify_instruction_covers_temporal_and_product_ambiguity` (комментарий + новые assert'ы про «разветвление»/«3+ продукта»/«переспроси продукт»; правка `не переспрашивай`→`не переспрашивать`). Разрастания нет.
- **Тесты: PASS.** `pytest tests/application tests/domain tests/interface` → **920 passed, 19 skipped** (21.99s). Совпадает с заявленным 920. Только warnings (deprecation FastAPI 422, event-loop-close в teardown) — не падения.
- **Достоверность/acceptance: PASS с оговоркой.** Автотест проверяет лишь НАЛИЧИЕ подстрок в тексте инструкции, не реальное поведение LLM. Заявленный результат «18/18 на живом LLM» — самоотчёт воркера, в этом гейте не аудируется (нет доступа к прод-LLM). Флаг тимлиду: если 18/18 критичен для приёмки — нужен независимый прогон verifier'а на живом LLM, а не только зелёный pytest.
- **AI-подписи: PASS.** Нет.

## Ветка 2 — feat/rag1-convctx-contextual-retrieval (247d31a) — ВЕРДИКТ: PASS

- **Гит-чистота: PASS.** `git status` чист. diff --stat: только `src/helm/application/docs_assistant.py` (+15/-2) и `tests/application/test_docs_assistant.py` (+37, новые тесты). Секретов/мусора нет. AI-подписей нет.
- **Скоуп: PASS.** Изменения только в теле `DocsAssistant.ask`: при непустой history строится `search_query` (склейка прошлых вопросов + текущий), по нему идут `self._search.search(...)` и `self._reranker.rerank(search_query, ...)`. Пустая/None history → `search_query == question` (1:1). Генерация (промпт), guardrail (confidence-floor), CLARIFY_INSTRUCTION — НЕ тронуты (в diff 0 упоминаний CLARIFY_INSTRUCTION). ast.parse OK.
- **Тесты: PASS.** `pytest tests/application tests/domain tests/interface` → **923 passed, 19 skipped** (22.04s). Совпадает с заявленным 923.
- **AI-подписи: PASS.** Нет.

## Конфликт мержа — РИСК НИЗКИЙ, конфликта НЕТ

Обе трогают `docs_assistant.py` и `test_docs_assistant.py`, но в непересекающихся регионах:
- clarify-v3 → константа CLARIFY_INSTRUCTION (~стр. 109–155) + тест-функция ~стр. 508–537.
- convctx-retr → тело `ask()` (~стр. 315–345) + НОВЫЕ тест-функции (append).

Проверка `git merge-tree --write-tree`:
- clarify-v3 в origin/main → clean (exit 0).
- convctx-retr в origin/main → clean (exit 0).
- **обе ветки вместе (база origin/main) → clean (exit 0)**, дерево `b32770d`. В объединённом дереве присутствуют ОБА изменения (`search_query = question` =1, `rerank(search_query` =1, «РАЗВЕТВЛЕНИЕ ПО ПРОДУКТУ» =1, assert «разветвл» в тесте =1), конфликт-маркеров 0, ast.parse обоих файлов OK.

**Рекомендованный порядок мержа:** любой (симметрично). Предпочтительно clarify-v3 → затем convctx-retr (чистая пользовательская фича раньше ретрив-фикса), но порядок на конфликт не влияет.

**Рекомендация:** после второго мержа один раз прогнать полный pytest на итоговом main — merge-tree чист, но комбинированное состояние ни одним из воркеров не тестировалось совместно (риск низкий, изменения независимы; ожидаемо 923 passed).

## Итог
- feat/rag1-clarify-v3 — PASS (оговорка: «18/18 live LLM» не аудирован в гейте).
- feat/rag1-convctx-contextual-retrieval — PASS.
- Конфликт мержа — НЕТ (clean, дерево b32770d); порядок любой.
