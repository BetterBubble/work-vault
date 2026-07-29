---
title: gate-rag1-corrclar-convctx
type: report
permalink: tacticum/00-board/gate-rag1-corrclar-convctx-1
tags:
- rag1
- gate
- controller
- clarify
- conversation-context
- helm
archived-at: 2026-07-29 18:12
---

# gate-rag1-corrclar-convctx

status: draft
Роль: controller (гейт перед мержем, read-only). Репо: `/Users/bubblemac/tacticum/helm`, main = `ea295f8` (задеплоенный прод RAG#1).
Проверены две несмёрженные/непушеные ветки в отдельных worktree'ах.

## ИТОГ
- **feat/rag1-conversation-context** — **PASS** (блокеров нет).
- **feat/rag1-correctness-clarify** — **PASS** (блокеров нет).
- **Конфликт мержа обеих в main** — риск НИЗКИЙ: git авто-мержит без конфликтов (`git merge-tree --write-tree` exit 0).

---

## Ветка 1: feat/rag1-conversation-context (worktree helm-wt-rag1-convctx, HEAD 2fa4ad6)

1. **Гит-чистота — PASS.** `git status` чист. 3 коммита (4dc2b1a, c0db010, 2fa4ad6), автор Александр Шульга. `diff --stat`: 11 файлов, +557/-26. Секретов/.env/ключей/бинарников нет; трекнутого мусора (`__pycache__`/`.DS_Store`/`.serena`/`.pyc`) нет (есть НЕтрекнутый `__pycache__/*.pyc` от новой миграции — вне индекса, ок). Все файлы по задаче.
2. **Скоуп — PASS.** Только conversation-context: model `DocsConversationTurn`, миграция, repo (add_turn/get_recent_turns/purge_expired_turns), config (3 флага), `docs_assistant.py` (`_render_history` + `ask(history=...)`), проводка `docs_clarify.py`/`bot_support.py`/`docs.py`, тесты. Clarify/retrieval-логика по существу НЕ тронута: в `docs_assistant.py` только добавлен параметр `history` и вызов `_build_user_prompt(...,history)`; `CLARIFY_INSTRUCTION`/`decide_retrieval_action` не менялись. `domain/docs.py` не тронут.
3. **Тесты — PASS. Подтверждено 111 passed** (0 failed), команда из отчёта. Совпадает с заявленным.
4. **Флаги-безопасность — PASS.** `docs_conversation_context_enabled: bool = False` (env `HELM_DOCS_CONVERSATION_CONTEXT_ENABLED`). При OFF каналы историю не собирают/не пишут, промпт 1:1. Гейт в проводке `(clarify OR context) AND session_factory`.
5. **Миграции — PASS.** `f4e5d6c7b8a9`, `down_revision = e3d4c5b6a7f8`. На main единственный head = `e3d4c5b6a7f8`; после миграции единственный head = `f4e5d6c7b8a9`. Ветвления/множественных head нет, цепочка линейна. Оговорка (не блокер): `alembic upgrade` на sqlite не гоняется (env.py → postgres); соответствие модели проверено `test_models_metadata`.
7. **AI-подписи — PASS.** В телах коммитов футеров атрибуции нет.

## Ветка 2: feat/rag1-correctness-clarify (worktree helm-wt-rag1-corrclar, HEAD be7dbef)

1. **Гит-чистота — PASS.** `git status` чист. 2 коммита (6d5a301, be7dbef), автор Александр Шульга. `diff --stat`: 4 файла, +45/-103. Секретов/мусора/бинарников нет. Файлы по задаче.
2. **Скоуп — PASS.** Проблемы 1+2: калибровка `CLARIFY_INSTRUCTION` (темпоральная + звонок-vs-конференция + анти-ложные условия) в `docs_assistant.py`; удаление мёртвого `build_clarify_question` (+`_clarify_facet`/`_CLARIFY_TOP_K`/`_CLARIFY_MAX_OPTIONS`) и параметра `tau_answer` в `domain/docs.py` + call-site. Retrieval/rerank/synonyms/context_limit НЕ тронуты (по решению лида) — подтверждено: изменены только `docs_assistant.py`, `domain/docs.py`, 2 тест-файла. `tau_floor` живой, контракт `answer|not_found` сохранён. Ambiguous-golden — в отдельном репо `iva-rag1-docs`, в helm не попал (корректно). Известный осознанный хвост (не блокер): конфиг-цепочка `clarify_tau_answer` оставлена (кормит удалённый параметр) — отдельная задача, задокументирована в отчёте.
3. **Тесты — PASS. Подтверждено 910 passed, 19 skipped**, команда `uv run pytest tests/domain tests/application tests/interface -q`. Совпадает с заявленным.
4. **Флаги-безопасность — PASS.** `docs_clarify_enabled: bool = False`. Изменение `CLARIFY_INSTRUCTION` инжектится только при `allow_clarify=True` (за флагом). Деплой с флагом OFF поведенчески инертен.
5. **Миграции — н/д** (ветка миграций не добавляет).
7. **AI-подписи — PASS.** Футеров атрибуции нет.

---

## КОНФЛИКТ МЕРЖА (пункт 6)
Обе ветки трогают `src/helm/application/docs_assistant.py` и `tests/application/test_docs_assistant.py`.
- Пересечение вычислено; `git merge-tree --write-tree feat/rag1-correctness-clarify feat/rag1-conversation-context` → **exit 0, конфликтов нет**. Трёхсторонняя симуляция от базы main — маркеров `<<<<<<<` нет.
- Причина безопасности: хунки не пересекаются текстово. В `docs_assistant.py` corrclar меняет `CLARIFY_INSTRUCTION` (~стр.109) и вызов `decide_retrieval_action` (~стр.300, убирает `tau_answer`); convctx добавляет `_render_history`/`_build_user_prompt` (~стр.201) и `history` в сигнатуру `ask` (~стр.281) + вызов `generate` (~стр.340). В тест-файле оба ДОБАВЛЯЮТ новые тест-функции в разных местах.
- **Риск: НИЗКИЙ, авто-мерж чистый в любом порядке.** Семантически изменения ортогональны (convctx = память диалога; corrclar = формулировка clarify + уборка мёртвого параметра), после мержа `ask()` содержит оба.

### Рекомендованный порядок мержа
Порядок не критичен (авто-мерж в обе стороны). Рекомендация для минимума неожиданностей:
1. Сначала **feat/rag1-correctness-clarify** (меньше, domain-уборка + формулировка).
2. Затем **feat/rag1-conversation-context** (несёт миграцию, шире по проводке).
После первого мержа перед вторым: повторно прогнать `git merge-tree`/целевые тесты на актуальном main (страховка, если main продвинется другими ветками). Миграцию convctx на прод применять по runbook отдельно; флаги на проде оставить OFF — оба изменения тогда поведенчески инертны.

## Блокеры
Нет. Обе ветки PASS по всем применимым гейтам.