---
title: rag2-golden-ready
type: report
permalink: tacticum/00-inbox/rag2-golden-ready
tags:
- rag2
- golden
- eval
- ready
- contracts
- negative
- verifier-handoff
---

# RAG#2 golden — готовые-к-вставке JSON-фрагменты (для verifier)

База `main`. Все ключи/факты извлечены детерминированно из `data/iva/` (as_of корпуса). **Профиль-файлы НЕ трогал — владелец verifier.** Схема кейсов зеркалит `rag2_golden_profile.json`. Три набора → три файла (решение лида): дораметка → `rag2_golden_profile.json`, контрактные → `rag2_golden_contracts.json`, negative → `rag2_golden_negative.json`.

---

## A. Детерминированная дораметка (в `rag2_golden_profile.json`, замена существующих кейсов)

### A1. a-status-03 — ЧИСТО детерминирован ✅ (15 epic-key из epics.json)
Метод: `epics.json` rows где `status ∈ {Приостановлено, Backlog}`. Проверяемо: 7 Приостановлено + 8 Backlog = 15.
```json
{
  "id": "a-status-03", "role": "analyst", "category": "статус-жизненный-цикл",
  "query": "Какие эпики IVAONE находятся в статусе «Приостановлено» и «Backlog»?",
  "expected_mode": "index", "expected_structural": false, "expected_temporal": false,
  "expected_needs_confluence_body": false, "requires": ["helm"],
  "expected_keys": ["IVAONE-867","IVAONE-1616","IVAONE-3075","IVAONE-3287","IVAONE-3774","IVAONE-4422","IVAONE-9387","IVAONE-870","IVAONE-969","IVAONE-1059","IVAONE-1205","IVAONE-3864","IVAONE-6663","IVAONE-10490","IVAONE-11258"],
  "expected_sources": ["epics.json"],
  "ideal_answer": null,
  "ideal_answer_key_facts": ["Приостановлено: 7 эпиков (IVAONE-867, 1616, 3075, 3287, 3774, 4422, 9387)","Backlog: 8 эпиков (IVAONE-870, 969, 1059, 1205, 3864, 6663, 10490, 11258)","Всего 15 эпиков в двух статусах"],
  "difficulty": "easy"
}
```

### A2. a-cross-01 — ⚠️ КОРПУС-ГЭП (НЕ размечается positive сейчас)
Метод дал **0 связей**: в `jira_issue_links.csv` (5512 Blocks) **проект IVAONEHALF ОТСУТСТВУЕТ полностью** (0 связей его касаются). Кросс-проектные связи в целом есть (2546), но не IVAONE↔IVAONEHALF. → В текущем срезе корпуса правильный ответ = «прямых связей IVAONE↔IVAONEHALF в корпусе нет». **Решение за лидом/verifier:** (вариант 1) держать как negative-now (empty) c флагом; (вариант 2) чинить пополнением корпуса IVAONEHALF, тогда → semi (прогон). Черновик под вариант 1:
```json
{
  "id": "a-cross-01", "role": "analyst", "category": "кросс-проектные",
  "query": "Есть ли кросс-проектные зависимости между IVAONE и IVAONEHALF — задачи, блокирующие друг друга через проекты?",
  "expected_mode": "hybrid", "expected_structural": true, "expected_temporal": false,
  "expected_needs_confluence_body": false, "requires": ["jira","helm"],
  "expected_keys": [], "expected_no_answer": true,
  "expected_sources": ["jira_issue_links.csv"],
  "ideal_answer": null,
  "ideal_answer_key_facts": ["В текущем срезе корпуса проект IVAONEHALF отсутствует (0 связей)","Прямых кросс-проектных Blocks-связей IVAONE↔IVAONEHALF не найдено"],
  "difficulty": "hard",
  "_note": "КОРПУС-ГЭП: IVAONEHALF нет в срезе data/iva. Станет positive+semi после пополнения корпуса."
}
```

### A3. Агрегатные (keys неприменимы → `expected_keys:[]` + key_facts, мерить answer_in_context)

**a-graph-03** (Blocks total + топ-эпики):
```json
{
  "id": "a-graph-03", "role": "analyst", "category": "связи-зависимости-граф",
  "query": "Сколько всего связей Blocks между задачами в корпусе и какие эпики затронуты сильнее всего?",
  "expected_mode": "index", "expected_structural": false, "expected_temporal": false,
  "expected_needs_confluence_body": false, "requires": ["helm"],
  "expected_keys": [], "expected_sources": ["jira_issue_links.csv"],
  "ideal_answer": null,
  "ideal_answer_key_facts": ["Всего связей Blocks в корпусе: 5512","Все связи типа Blocks (label=blocks)","Больше всего блокирует: IVAONE-2857 (14), IVASERV-1293 (13), IVACODEC-525 (10)"],
  "difficulty": "medium"
}
```

**s-coverage-01** (распределение вердиктов):
```json
{
  "id": "s-coverage-01", "role": "support", "category": "покрытие-требований",
  "query": "Сколько требований IVA One 1.0 имеют вердикт met, partial, planned, absent?",
  "expected_mode": "index", "expected_structural": false, "expected_temporal": false,
  "expected_needs_confluence_body": false, "requires": ["helm"],
  "expected_keys": [], "expected_sources": ["req_realization.json"],
  "ideal_answer": null,
  "ideal_answer_key_facts": ["Поколение 1.0: met 477, partial 303, absent 76, planned 0 (всего 856)","В 1.0 статуса planned нет — все требования либо реализованы, либо частично/отсутствуют","Весь корпус требований (1.0+1.5): met 828, partial 405, planned 416, absent 151"],
  "difficulty": "easy"
}
```
> ⚠️ Формулировка кейса про «1.0» — а planned есть только в 1.5. Verifier: либо сузить факты до 1.0 (planned=0), либо переформулировать query на «весь корпус». Оставил оба факта.

**s-coverage-02** (planned в 1.5):
```json
{
  "id": "s-coverage-02", "role": "support", "category": "покрытие-требований",
  "query": "Какие требования IVA One 1.5 ещё только запланированы (planned) и не начаты?",
  "expected_mode": "index", "expected_structural": false, "expected_temporal": false,
  "expected_needs_confluence_body": false, "requires": ["helm"],
  "expected_keys": [], "expected_sources": ["req_realization.json","wiki_reqs_15.json"],
  "ideal_answer": null,
  "ideal_answer_key_facts": ["Поколение 1.5: planned = 416 требований","В 1.5 также: met 351, partial 102, absent 75 (всего 944)","Примеры planned 1.5: «Административная панель: Дашборд текущей активности», «Интеграция модулей (Connect, MCU, SFU, LDAP)», «Корректный учёт лицензий»"],
  "difficulty": "medium"
}
```

**s-feature-01** (адресная книга 1.0):
```json
{
  "id": "s-feature-01", "role": "support", "category": "статус-фичи",
  "query": "Что уже реализовано по фиче «Адресная книга» в IVA One 1.0?",
  "expected_mode": "index", "expected_structural": false, "expected_temporal": false,
  "expected_needs_confluence_body": false, "requires": ["helm"],
  "expected_keys": [], "expected_sources": ["req_realization.json","wiki_reqs_10.json"],
  "ideal_answer": null,
  "ideal_answer_key_facts": ["В корпусе ~78 требований со словом «адресная книга» (met + partial)","met (реализовано): просмотр/редактирование/удаление/сортировка записей; поиск и сортировка в глобальной адресной книге","partial (частично): действия из карточки контакта в один клик; создание мероприятия с контактами"],
  "difficulty": "medium"
}
```

---

## B. Контрактные Р-2a → `rag2_golden_contracts.json` (12 кейсов, negative-до-корпуса)
Корпуса `api/contract` нет → сейчас все `expected_not_found_until_corpus:true`, `expected_keys:[]`. После ингеста контрактов JUMP: снять флаг + проставить `expected_keys` = doc-id (напр. `JUMP:messageSync`, `DOC-000245`). Схема = профиль + `expected_source_type:"api"`.
```json
[
 {"id":"c-jump-recall-01","role":"support","category":"contract","query":"Есть ли в JUMP команда отзыва (recall) уже отправленного письма?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["api"],"expected_source_type":"api","expected_not_found_until_corpus":true,"expected_keys":[],"expected_sources":["distrohost.msk/Docs/*"],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"medium"},
 {"id":"c-jump-msgsync-02","role":"support","category":"contract","query":"Какие параметры у операции messageSync в контракте JUMP (cursor, limit, since)?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["api"],"expected_source_type":"api","expected_not_found_until_corpus":true,"expected_keys":[],"expected_sources":["distrohost.msk/Docs/*"],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"medium"},
 {"id":"c-jump-mailboxfind-03","role":"support","category":"contract","query":"Что возвращает mailboxFind — поля ответа и обязательные аргументы?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["api"],"expected_source_type":"api","expected_not_found_until_corpus":true,"expected_keys":[],"expected_sources":["distrohost.msk/Docs/*"],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"medium"},
 {"id":"c-jump-chatdel-04","role":"support","category":"contract","query":"Каким методом контракта удаляется сообщение из чата?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["api"],"expected_source_type":"api","expected_not_found_until_corpus":true,"expected_keys":[],"expected_sources":["distrohost.msk/Docs/*"],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"medium"},
 {"id":"c-jump-msgedit-05","role":"support","category":"contract","query":"Есть ли в API метод редактирования уже отправленного сообщения чата?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["api"],"expected_source_type":"api","expected_not_found_until_corpus":true,"expected_keys":[],"expected_sources":["distrohost.msk/Docs/*"],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"medium"},
 {"id":"c-jump-errors-06","role":"support","category":"contract","query":"Какие коды ошибок у операции отправки письма в JUMP?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["api"],"expected_source_type":"api","expected_not_found_until_corpus":true,"expected_keys":[],"expected_sources":["distrohost.msk/Docs/*"],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"hard"},
 {"id":"c-jump-attach-07","role":"support","category":"contract","query":"Поддерживает ли контракт вложения в сообщении чата и какой лимит размера?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["api"],"expected_source_type":"api","expected_not_found_until_corpus":true,"expected_keys":[],"expected_sources":["distrohost.msk/Docs/*"],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"medium"},
 {"id":"c-jump-folders-08","role":"support","category":"contract","query":"Каким методом получить список папок почтового ящика (mailbox folders)?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["api"],"expected_source_type":"api","expected_not_found_until_corpus":true,"expected_keys":[],"expected_sources":["distrohost.msk/Docs/*"],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"medium"},
 {"id":"c-jump-markread-09","role":"support","category":"contract","query":"Есть ли операция массовой отметки писем прочитанными?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["api"],"expected_source_type":"api","expected_not_found_until_corpus":true,"expected_keys":[],"expected_sources":["distrohost.msk/Docs/*"],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"easy"},
 {"id":"c-jump-session-10","role":"support","category":"contract","query":"Как в JUMP инициировать сессию — endpoint Sessions?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["api"],"expected_source_type":"api","expected_not_found_until_corpus":true,"expected_keys":[],"expected_sources":["distrohost.msk/Docs/Sessions.html"],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"medium"},
 {"id":"c-jump-webhook-11","role":"support","category":"contract","query":"Есть ли подписка или webhook на новые сообщения чата в контракте?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["api"],"expected_source_type":"api","expected_not_found_until_corpus":true,"expected_keys":[],"expected_sources":["distrohost.msk/Docs/*"],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"hard"},
 {"id":"c-jump-paging-12","role":"support","category":"contract","query":"Как устроена пагинация в messageSync — курсор или offset?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["api"],"expected_source_type":"api","expected_not_found_until_corpus":true,"expected_keys":[],"expected_sources":["distrohost.msk/Docs/*"],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"medium"}
]
```

---

## C. Negative-набор → `rag2_golden_negative.json` (8 кейсов)
Правильный ответ = ничего выше порога / no_answer. Поля: `expected_no_answer:true`, `expected_below_floor:true`, `expected_keys:[]`. Метрика — `precision@k`/`noise_kept_rate`.
```json
[
 {"id":"n-ooc-pricing-01","role":"analyst","category":"negative-oocorpus","query":"Сколько стоит лицензия IVA One на 100 пользователей?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":[],"expected_no_answer":true,"expected_below_floor":true,"expected_keys":[],"expected_sources":[],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"easy","_reason":"цены вне SoR (ср. known_gaps RAG#1)"},
 {"id":"n-ooc-roadmap-02","role":"analyst","category":"negative-oocorpus","query":"Когда выйдет IVA One 3.0 и что в роадмапе на 2027 год?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":[],"expected_no_answer":true,"expected_below_floor":true,"expected_keys":[],"expected_sources":[],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"easy","_reason":"сроки/роадмап вне корпуса"},
 {"id":"n-ooc-competitor-03","role":"analyst","category":"negative-oocorpus","query":"Чем IVA MCU лучше Zoom и Microsoft Teams по качеству видео?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":[],"expected_no_answer":true,"expected_below_floor":true,"expected_keys":[],"expected_sources":[],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"easy","_reason":"сравнение с конкурентами вне SoR"},
 {"id":"n-nomatch-04","role":"analyst","category":"negative-nomatch","query":"Какая задача в IVAONE про интеграцию с подводной лодкой?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["jira"],"expected_no_answer":true,"expected_below_floor":true,"expected_keys":[],"expected_sources":[],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"medium","_reason":"заведомо несуществующая сущность"},
 {"id":"n-lexnoise-05","role":"analyst","category":"negative-lexnoise","query":"статус","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":["jira"],"expected_no_answer":true,"expected_below_floor":true,"expected_keys":[],"expected_sources":[],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"hard","_reason":"один общий токен — лексический шум, ничего не подтверждено выше порога"},
 {"id":"n-hr-06","role":"analyst","category":"negative-oocorpus","query":"Какая зарплата у исполнителя задачи IVAONE-655?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":[],"expected_no_answer":true,"expected_below_floor":true,"expected_keys":[],"expected_sources":[],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"medium","_reason":"HR/персональные данные вне корпуса"},
 {"id":"n-tenant-07","role":"analyst","category":"negative-isolation","query":"Покажи требования проекта «Залог Успеха» (cifragen)","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":[],"expected_no_answer":true,"expected_below_floor":true,"expected_keys":[],"expected_sources":[],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"hard","_reason":"другой tenant → fail-closed изоляция (двойная польза: #32 isolation)"},
 {"id":"n-greeting-08","role":"analyst","category":"negative-nonquery","query":"привет, что ты умеешь?","expected_mode":"index","expected_structural":false,"expected_temporal":false,"expected_needs_confluence_body":false,"requires":[],"expected_no_answer":true,"expected_below_floor":true,"expected_keys":[],"expected_sources":[],"ideal_answer":null,"ideal_answer_key_facts":[],"difficulty":"easy","_reason":"не ретрив-запрос — нет хитов выше порога"}
]
```

---

## Заметки verifier'у (важно перед вставкой)
1. **a-status-03** — единственный ЧИСТО детерминированный positive (15 keys проверены). Вставляй как есть.
2. **a-cross-01** — НЕ positive: IVAONEHALF отсутствует в срезе корпуса. Дал negative-версию; решение (negative vs чинить корпус) — за тобой/лидом.
3. **Агрегатные** (a-graph-03, s-coverage-01/02, s-feature-01) — `expected_keys:[]` намеренно, НЕ считать recall@k, только `answer_in_context`. s-coverage-01: формулировка про «1.0», а planned есть только в 1.5 — сверь query/факты.
4. **Контрактные/negative** используют доп. поля (`expected_source_type`, `expected_not_found_until_corpus`, `expected_no_answer`, `expected_below_floor`, `_reason`) — если harness строго валидирует схему профиля, добавь их в схему ИЛИ держи отдельные файлы (что и выбрано).
5. **Semi-кейсы (6)** намеренно НЕ финализировал — доразметка после baseline-прогона по корпусу.
6. `_note`/`_reason`/`_comment` — служебные, при строгой схеме вычисти или заведи в meta.
