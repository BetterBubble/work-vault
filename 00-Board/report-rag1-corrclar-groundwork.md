---
title: report-rag1-corrclar-groundwork
type: note
permalink: tacticum/00-board/report-rag1-corrclar-groundwork-1
status: draft
role: implementer
branch: feat/rag1-correctness-clarify
tags:
- rag1
- clarify
- golden
- docs-assistant
- draft
---

# RAG#1 correctness + clarify groundwork (части A-D)

## Ветка / worktree
- Ветка: `feat/rag1-correctness-clarify` (от `main` ea295f8)
- Worktree: `/Users/bubblemac/tacticum/helm-wt-rag1-corrclar`
- Коммиты (2): `6d5a301` (clarify-калибровка), `be7dbef` (удаление мёртвого кода)
- Retrieval/rerank/cap/synonyms/context_limit НЕ трогал (отдельный шаг тимлида).

## Часть A. Ambiguous-golden (замер корректности)
Новый файл (базовый eval200 не тронут): `/Users/bubblemac/tacticum/iva-rag1-docs/golden/golden_iva_rag1.ambiguous.json`
- 10 кейсов, схема идентична eval200 + служебные поля `ambiguity_axis`/`expect_clarify` (парсер `helm.eval.golden.GoldenCase` их игнорирует; проверил — 10/10 парсятся, у всех relevant+ideal+key_facts).
- Разбивка: 5 `temporal-active-call` (relevant = *-ug-calls), 2 `control-call` (однозначный звонок), 3 `control-event` (однозначное мероприятие, relevant = event pages).
- Активный звонок → slugs: `one-web-ug-calls`, `one-android-ug-calls`, `one-ios-ug-calls`, `connect-android-ug-calls`. ideal_answer VERBATIM-заземлены по corpus md (calls.md, секции «Участники → Добавить участников»: web стр.225, android стр.140, ios стр.204, connect-android стр.117).
- Контроль-мероприятие → slugs: `one-ios-ug-events-event-planning`, `one-web-ug-events-add-events`, `one-android-ug-events-event-create` (заземлено по event_planning/add_events/event_create).
- Все relevant_doc_ids сверены со списком `_latest_slugs.json` (0 неизвестных). `python -m json.tool` — VALID.

## Часть B. Калибровка clarify (docs_assistant.py)
`CLARIFY_INSTRUCTION` (src/helm/application/docs_assistant.py) переписана: было — триггер только «разные продукты/функции». Стало — 3 условия неоднозначности:
1. Продукт/функция (прежний триггер сохранён: IVA MCU / IVA One / IVA Mail).
2. **Темпоральная**: запрос допускает ОБЕ трактовки — «действие сейчас в ИДУЩЕМ/активном звонке (звонок в чате)» ИЛИ «в ЗАПЛАНИРОВАННОМ мероприятии/событии/ВКС-встрече». Пример в промпте: «добавить человека в звонок/в ВКС» без указания времени → переспросить.
3. **Звонок vs конференция**: неясно, звонок в чате IVA One или конференция/мероприятие на IVA MCU.

Прецизионный анти-ложный блок: НЕ переспрашивать, если явно «во время звонка / в идущем / активном звонке / в текущем вызове» (→ отвечать про активный звонок) ЛИБО явно «мероприятие/событие/встреча/запланировать/создать ВКС-встречу» (→ отвечать про мероприятие), а также при явно указанном продукте. Формат маркера `[[CLARIFY]]` и его парсинг (`_split_clarify_marker`, ~:121), гейт `docs_clarify_enabled` — НЕ тронуты.

Валидационный набор (для ручной прод-проверки тимлидом, НЕ автотест): `/Users/bubblemac/tacticum/iva-rag1-docs/golden/clarify_cases.json` — 10 кейсов {query, expect_clarify, why}: 5 двусмысленных (true, вкл. «Как добавить человека в идущий ВКС?») + 5 однозначных (false). JSON VALID.

## Часть C. Уборка мёртвого кода (domain/docs.py)
Перед удалением подтверждал `find_referencing_symbols`:
- **`build_clarify_question`** — референсы ТОЛЬКО в `__all__` и в тестах (test_docs_clarify_gate.py), в проде не вызывается → **УДАЛЕНО** вместе с приватными хелперами `_clarify_facet`, `_CLARIFY_TOP_K`, `_CLARIFY_MAX_OPTIONS` (они использовались только внутри неё). Убрано из `__all__`.
- **`tau_answer`** (параметр `decide_retrieval_action`) — в теле не читался (решение по `tau_floor`) → **УДАЛЁН** из сигнатуры + call-site в docs_assistant.py + тестовый helper. Docstring подчищен.
- **ОСТАВЛЕНО осознанно (шире плана, не расширял молча):** цепочка конфигурации `clarify_tau_answer` (config.py `docs_clarify_tau_answer` + env `HELM_DOCS_CLARIFY_TAU_ANSWER`, routers bot_support.py/docs.py, service.py DTO, конструктор DocsAssistant, поле `self._clarify_tau_answer`). После удаления параметра функции эта цепочка кормит мёртвое значение, но её удаление трогает 5+ файлов (публичный конфиг-контракт + 2 роутера + infra DTO + test_analyst_mcp.py). Это выходит за объём A-D — **не трогал, оставил на решение тимлида** (отдельная задача «выпилить clarify_tau_answer конфиг end-to-end»).
- `tau_floor` — живой, НЕ тронут. Контракт `answer|not_found` не изменён.

## Часть D. Тесты
- `tests/domain/test_docs_clarify_gate.py` — переписан без `build_clarify_question`/`tau_answer` (удалены 5 тестов на удалённую функцию; helper `_decide` без tau_answer; сохранены все тесты гейта).
- `tests/application/test_docs_assistant.py` — добавлен `test_clarify_instruction_covers_temporal_and_product_ambiguity` (проверяет темпоральный+продуктовый триггеры и наличие анти-ложного «не переспрашивай»). Парсинг `[[CLARIFY]]` покрыт прежними тестами — не сломан.
- Команда: `uv run pytest tests/domain tests/application tests/interface -q`
- Результат: **910 passed, 19 skipped** (warnings пред-существующие: Starlette deprecation, event-loop-closed — не связаны с правками). Ruff по 4 изменённым файлам: All checks passed.

## Для тимлида
- Не мержено/не запушено/не деплоено.
- Открытый вопрос: выпиливать ли `clarify_tau_answer` конфиг-цепочку целиком (см. Часть C) — могу дополнить эту же ветку, если скажешь.