---
title: rag2-p3-meili-validation
type: report
permalink: tacticum/00-board/rag2-p3-meili-validation-1
tags:
- rag2
- p3
- query-expand
- meili
- validation
- verifier
- helm
---

# RAG#2 P3 query-expand — прод-валидация на живом Meili

Verifier, контроль-режим (юнит-логику сверяем с живым Meili). Read-only: прямые Meili-запросы + sidecar overlay ветки `feat/rag2-query-expand-code` (флаг `HELM_RAG2_QUERY_EXPAND_CODE`). helm-helm-1 не нагружался сверх мелких probe; env-file без вывода секретов, overlay/патч/probe удалены после. База main b5d1739 (overlay из `/opt/helm/src`). Ветка source-only (deps не тронуты).

## Вердикт
**Р-3 работает на реальном Meili. Юнит-логика НЕ расходится с живым поведением.** Split+join даёт слитный токен `messagesync`, который матчит нормализованный индексный токен → «message sync» (с пробелом) поднимает doc **208701103**. Регресса на RU-запросах нет. Нюанс: в полном confluence-**hybrid** (dense+Meili) doc находится и без фикса (dense спасает), а expand **улучшает ранг**; на **чистом Meili** фикс решающий (None→#1).

## 1. Что делает фикс (`expand_code_query`, проверено локально)
Строковое расширение (не LLM), применяется в начале `search()` к тексту до dense и ft (и для jira, и для confluence-адаптера — `service.py`).
- `messageSync` → `messageSync message sync` (слитная форма = сам токен, уже был; добавляет разбитую).
- `message sync` → `message sync messagesync` ← **ключевое: добавлен слитный токен**.
- `message_sync` / `message-sync` → `… message sync messagesync`.
- `Как настроить messageSync в почте` → `… message sync` (RU + latin — латиницу обрабатывает).
- **`Политика хранения записей конференций` → без изменений** (RU без латиницы → no-op, антирегресс).

## 2. Прямой Meili `iva_confluence` (10.16.0.19:7700) — лексический канал
| запрос | rank doc 208701103 |
|---|---|
| RAW `message sync` | **None** (не в top-20) |
| EXPANDED `message sync messagesync` | **#1** ✅ |
| RAW `messageSync` | #1 |
| EXPANDED `messageSync message sync` | #1 (регресса нет) |

→ Слитный токен `messagesync` матчит нормализованный индексный `messageSync`→`messagesync`. Live-поведение = юнит-логике. **Меняет None → #1.**

## 3. Интеграционный прогон (overlay + флаг, confluence-адаптер hybrid dense+Meili)
overlay грузится подтверждён (`query_expand.__file__ = /tmp/helm-p3-src/...`), `query_expand_code` off/on переключается флагом.
| запрос | OFF rank | ON rank |
|---|---|---|
| `message sync` | 5 | **3** |
| `Как работает message sync в почте JUMP` | 2 | **1** |
| `политика хранения записей конференций` (RU-контроль) | None | None (top5 идентичны) |

Наблюдения:
- В полном **hybrid** doc находится и БЕЗ фикса (dense bge-m3 семантически матчит «message sync»↔«messageSync»-контент) → rank 5/2. Expand **улучшает ранг** (5→3, 2→1), не спасает с нуля в этом пути.
- **RU-контроль: top5 идентичны ON/OFF** → регресса нет (expand не трогает RU).
- Практический эффект Р-3 **сильнее всего там, где dense не помогает** (чистый Meili / code-токен без семантического контекста) — там None→#1; где dense уже вытягивает — улучшение ранга.

## Остаточные риски / что отметить
- **Join бьёт по любым соседним латинским словам:** «data base»→«database», «user name»→«username». Обычно безвредно (доп. токен, оригинал в голове, дедуп+лимит `max_added=16`), но на чисто-латинских НЕ-code фразах может слегка сдвинуть ранжирование. RU-запросы (основная масса аналитиков ИВА) не затронуты. Рекомендую 1-2 latin-noise контроль-кейса в golden при приёмке.
- Оценка меры: golden под Р-3 сейчас — фактически один кейс (doc 208701103). Для честной приёмки нужно ≥3–5 code-token кейсов (разные camelCase/snake/API-имена) + latin-noise контроль.

## Итог по приёмке Р-3
- **Механизм подтверждён на живом Meili** (не только юнит): split+join → `messagesync` → матч. ✅
- **Регресса на RU нет.** ✅
- Под мерж: фикс валиден, дёшев (query-side, реиндекс не нужен), за флагом. Рекомендация — включать; ценность максимальна для lexical-only/слабо-семантических code-запросов.
- Для полной приёмки #Р-3: расширить golden code-кейсами + latin-noise контроль (см. выше).

## Связанные
- `00-Board/rag2-retrieval-miss-diag` (Meili separatorTokens=[], hybrid-баланс), `00-Board/rag2-eval-baseline-v2`.