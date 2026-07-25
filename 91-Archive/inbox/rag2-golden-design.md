---
title: rag2-golden-design
type: report
permalink: tacticum/91-archive/inbox/rag2-golden-design
tags:
- rag2
- golden
- eval
- p2a
- contracts
- negative
- helm
status: archived
updated: 2026-07-18
---

# RAG#2 golden — доразметка + контрактные Р-2a + negative (черновик)

База `main`. Файл: `tests/data/rag2_golden_profile.json` (25 кейсов, схема `id/role/category/query/expected_mode/expected_structural/expected_temporal/expected_needs_confluence_body/requires/expected_keys/expected_sources/ideal_answer/ideal_answer_key_facts/difficulty`). Данные корпуса ЕСТЬ офлайн в `data/iva/` (epics.json, req_realization.json, client_blockers.json, jira_issue_links.csv, tasks_rich/*.jsonl, wiki_reqs_*.json) → часть разметки детерминирована. **Golden НЕ правил — только черновики ниже.**

## 1. Разметка 25 кейсов: таблица + метод
`expected_keys` заполнено у **9/25**: a-status-01/02/04, a-graph-01/02/04, a-cross-02, a-temporal-01/02. Разбор 16 неразмеченных по методу получения ключей:

| id | mode | requires | src (провенанс) | метод разметки | вердикт |
|---|---|---|---|---|---|
| a-status-03 | index | helm | epics.json | filter rows status∈{Приостановлено(7),Backlog(8)} → 15 epic-key | **детерминирован офлайн** ✅ |
| a-cross-01 | hybrid | jira,helm | tasks_rich/*, links(live) | jira_issue_links.csv: source_project≠target_project → пары key | **детерминирован офлайн** ✅ |
| a-graph-03 | index | helm | tasks_rich/* | links.csv: Blocks=5512 (агрегат-счёт) | агрегат → keys=[], мерить фактами |
| s-coverage-01 | index | helm | req_realization.json | verdict dist met828/partial405/absent151/planned416 (счёт) | агрегат → keys=[], факты |
| s-coverage-02 | index | helm | req_realization,wiki_reqs_15 | filter generation=1.5 & verdict=planned → titles | агрегат/по title (нет issue-key) |
| s-coverage-03 | hybrid | jira,helm | req_realization,req_jira_status | join req→jira-status по адр.книге | semi (нужен join+сверка) |
| s-similar-02 | index | helm | client_blockers,tasks_rich | grep «offline/защищённый сегмент» → task-key | **semi-офлайн (grep+сверка)** |
| s-similar-01 | index | jira,helm | tasks_rich/* | grep «голосовые сообщения/iOS» → task-key | semi (нужен прогон/сверка) |
| s-similar-03 | index | jira,helm | tasks_rich/* | grep «лицензирование/состав ПО» → task-key | semi |
| s-feature-01 | index | helm | req_realization,wiki_reqs_10 | адр.книга: req по title (нет key) | по фактам, не keys |
| s-feature-02 | index | jira,helm | tasks_rich,wiki_reqs_15 | Calls/MCU задачи → task-key | semi |
| s-feature-03 | hybrid | jira,helm | wiki_reqs_10,req_realization,tasks_rich | Почта: req+задачи | semi |
| a-rationale-01 | live | confluence | confluence_get_page(body,**future**) | тела Confluence НЕ выгружены | **BLOCKED → negative/future** |
| a-rationale-02 | live | confluence | confluence body future | — | **BLOCKED → negative/future** |
| a-reqtext-01 | hybrid | confluence,helm | confluence body future + wiki_reqs_10 | частично (helm-часть) | partial, тело — future |
| a-reqtext-02 | live | confluence | confluence body future | — | **BLOCKED → negative/future** |

**Ключевой нюанс:** `req_realization.json` строки БЕЗ issue-key (поля `title/generation/verdict/clients`). Значит требование-центричные кейсы (coverage/feature) **в принципе не размечаются `expected_keys`** — их мерить `answer_in_context@k`/`ideal_answer_key_facts`, а не recall@k. Это надо явно сказать verifier'у: recall/nDCG применимы ТОЛЬКО к jira/epic-key кейсам.

**Итог по офлайн-размечаемости:**
- **Детерминировано офлайн сейчас** (чтением data/iva): **2** (a-status-03, a-cross-01) → доводит keys-покрытие с 9 до **11/25**.
- **Semi-офлайн (grep-черновик + ручная сверка top-k):** ~6 (s-similar-01/02/03, s-feature-02/03, s-coverage-03) — черновик офлайн, но «правильный релевантный набор» = либо суждение, либо прогон ретрива по корпусу с ручной приёмкой.
- **Агрегат/по-фактам (keys неприменимы):** 4 (a-graph-03, s-coverage-01/02, s-feature-01) → ставим `expected_keys:[]` осознанно + `ideal_answer_key_facts`.
- **BLOCKED (Confluence body future):** 3–4 (a-rationale-01/02, a-reqtext-02, частично a-reqtext-01) → до выгрузки тел Confluence = negative.

## 2. Контрактные Р-2a golden (≥10, черновик)
Корпус `api/contract` (JUMP/distrohost) ещё НЕ существует (подтверждено рекон: `messageSync/mailboxFind` нигде в src). → **сейчас все = negative (ожидаем not-found), после появления `source_type=api/contract` переразмечаются в positive** (`expected_keys` = doc-id контракта). Помечены `expected_source_type: "api"`, `expected_not_found_until_corpus: true`.

```
{id:"c-jump-recall-01", role:"support", category:"contract",
 query:"Есть ли в JUMP команда отзыва (recall) уже отправленного письма?",
 expected_mode:"index", requires:["api"], expected_source_type:"api",
 expected_not_found_until_corpus:true, expected_keys:[], difficulty:"medium"}
{id:"c-jump-msgsync-02", query:"Какие параметры у операции messageSync в контракте JUMP (cursor/limit/since)?", ...}
{id:"c-jump-mailboxfind-03", query:"Что возвращает mailboxFind — поля ответа и обязательные аргументы?", ...}
{id:"c-jump-chatdel-04", query:"Каким методом контракта удаляется сообщение из чата?", ...}
{id:"c-jump-msgedit-05", query:"Есть ли в API метод редактирования уже отправленного сообщения чата?", ...}
{id:"c-jump-errors-06", query:"Какие коды ошибок у операции отправки письма в JUMP?", ...}
{id:"c-jump-attach-07", query:"Поддерживает ли контракт вложения в сообщении чата и какой лимит размера?", ...}
{id:"c-jump-folders-08", query:"Метод получения списка папок почтового ящика (mailbox folders)?", ...}
{id:"c-jump-markread-09", query:"Есть ли операция массовой отметки писем прочитанными?", ...}
{id:"c-jump-session-10", query:"Как в JUMP инициировать сессию — endpoint Sessions.html?", ...}
{id:"c-jump-webhook-11", query:"Есть ли подписка/webhook на новые сообщения чата в контракте?", ...}
{id:"c-jump-paging-12", query:"Как устроена пагинация в messageSync — курсор или offset?", ...}
```
(12 кейсов, боль пилота — почта JUMP/чаты/сессии. При появлении корпуса: `expected_not_found_until_corpus:false` + `expected_keys` = doc-id, напр. `DOC-000245`/`JUMP:messageSync`.)

## 3. Negative-набор (доказать «шум отсечён»), черновик 8 кейсов
Правильный ответ = ничего выше порога / `no_answer`. Схема: +`expected_no_answer: true`, `expected_keys: []`, `expected_below_floor: true`. Метрика — `precision@k`/`noise_kept_rate` (см. `rag2-p2a-audit-SUMMARY`). Идея взята из `known_gaps.json` (32 антипримера RAG#1, `expected=no_answer`) — но эти под аналитика RAG#2.

```
{id:"n-ooc-pricing-01", category:"negative-oocorpus",
 query:"Сколько стоит лицензия IVA One на 100 пользователей?",
 expected_mode:"index", expected_no_answer:true, expected_below_floor:true, expected_keys:[]}   # цены вне SoR (known_gaps)
{id:"n-ooc-roadmap-02", query:"Когда выйдет IVA One 3.0 и что в роадмапе на 2027?", ...}           # сроки/роадмап вне корпуса
{id:"n-ooc-competitor-03", query:"Чем IVA MCU лучше Zoom и Teams по качеству видео?", ...}          # конкуренты
{id:"n-nomatch-04", query:"Какая задача в IVAONE про интеграцию с подводной лодкой?", ...}          # заведомо нет сущности
{id:"n-lexnoise-05", query:"статус", ...}                                                            # один общий токен → лексический шум, ничего не подтверждено
{id:"n-hr-06", query:"Какая зарплата у исполнителя задачи IVAONE-655?", ...}                         # HR/персональное вне корпуса
{id:"n-tenant-07", query:"Покажи требования проекта Залог Успеха (cifragen)", ...}                   # другой tenant → fail-closed пусто (двойная польза: изоляция)
{id:"n-greeting-08", query:"привет, что ты умеешь?", ...}                                            # не ретрив-запрос → нет хитов выше порога
```
Плюс переиспользовать `expected_needs_confluence_body=future`-кейсы (a-rationale-01/02, a-reqtext-02) как **negative до выгрузки Confluence** — уже в профиле, просто добавить им `expected_no_answer:true` пока тел нет.

## Что блокирует
- **Контрактные positive:** нет корпуса `api/contract` (нет источника JUMP/distrohost, нет `source_type`). Сейчас только negative-режим. Разблок = доступ distrohost.msk + ингест контрактов (пересекается с Р-1/Р-4).
- **Semi-кейсы (recall):** «правильный набор keys» надёжно закрывается только прогоном ретрива по живому/локальному корпусу Qdrant + ручной приёмкой top-k (офлайн-grep даёт лишь черновик-кандидатов). Полный прогон = прод-гейт/локальный индекс (как A/B cross-rerank).
- **Confluence-body кейсы:** заблокированы до выгрузки тел Confluence в корпус.
- **req-центричные кейсы:** `expected_keys` неприменимы (нет issue-key у требований) → нужна договорённость с verifier мерить их `answer_in_context`, иначе они ложно тянут recall вниз.

## Открытые вопросы
1. Deterministic-разметку (a-status-03, a-cross-01) вношу как готовый черновик keys прямо сейчас, или ждём прогон-верификацию тоже?
2. Контрактные — держать negative-до-корпуса в ОДНОМ файле с флагом, или отдельный `rag2_golden_contracts.json`?
3. Negative-набор — расширять профиль или отдельный файл (verifier'у для precision удобнее отдельный)?
4. Порог tenant-изоляции (n-tenant-07) — это golden Р-2a или отдельный isolation-тест (у verifier он уже есть)?
