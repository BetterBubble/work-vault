---
title: rag2-p3-fix-design
type: note
permalink: tacticum/00-inbox/rag2-p3-fix-design
tags:
- rag2
- helm
- meili
- tokenization
- fix-design
- P3
- implementer-ready
---

# RAG#2 Р-3 (T3.1): дизайн фикса токенизации camelCase/snake — для implementer

Explorer, read-only. Дизайн реализации, готовый к передаче implementer'у. `helm@main (b5d1739)`.

## Корень (подтверждён эмпирикой verifier на живом Meili/Qdrant 10.16.0.19)
- doc 208701103 **ЕСТЬ** в `iva_confluence` (не «нет в индексе», не burial).
- Meili `separatorTokens=[]` → **camelCase не расщепляется**. Документный `messageSync` = единый токен `messagesync` (lowercase).
- `messageSync` → doc #1 (слитный запрос матчит слитный токен). `message sync` (раздельный запрос → токены `message`+`sync`) → **не в top-20** (не матчит слитный `messagesync`).
- Вывод: корень — токенизация, направление «раздельная форма ↔ слитная форма» не мостится.

## Критичный инсайт для приёмки (ОБЯЗАТЕЛЬНО учесть)
Приёмка двунаправленная: и `messageSync`, и `message sync` должны находить doc.
- **split** (`messageSync`→+`message sync`) закрывает: слитный запрос → находит и слитный (оригинал сохранён), и раздельный документ.
- **`message sync`→doc НЕ закрывается одним split** — запрос уже раздельный, бить нечего, а документ слитный `messagesync`. Нужен **join** (склейка соседних токенов `message sync`→+`messagesync`).
→ Вариант (а) обязан делать И split, И join. Чистый split приёмку №2 не проходит.

## Сравнение вариантов

| Критерий | (а) query-side split+join за флагом | (б) index-side split (доп. поле на ингесте) |
|---|---|---|
| Объём | малый: 1 чистый модуль + флаг + прокидка | средний: правка build-документа обоих корпусов |
| Переиндексация | **НЕ нужна** | **нужна** (Confluence + желательно Jira, прогон CLI на 10.16.0.19) |
| Обратимость | мгновенно (флаг off) | реиндекс назад |
| Покрытие | только формы, что есть в самом ЗАПРОСЕ (документы не меняем) | расщепляет ДОКУМЕНТЫ → любой запрос матчит; симметрично, лучший recall |
| Риск | join-эвристика может добавить шумный токен (митигируемо) | рост индекса; при смене логики — новый реиндекс |
| Итерируемость | высокая (крутить эвристику + мерить golden без инфры) | низкая (каждая правка = реиндекс) |

## РЕКОМЕНДАЦИЯ: вариант (а) — query-side split+join за флагом, дефолт OFF
Причины: doc уже в индексе (проблема чисто в мэтчинге форм), приоритет «дёшево/без реиндекса/обратимо/мерим на golden», быстрая итерация эвристики. Зеркалит существующий паттерн RAG#1 (`docs_query_rewrite_enabled` + `expand_query`).
**Оговорка:** если golden покажет недобор/шум от query-side join → эскалировать к (б) index-side как усилению (надёжнее, но реиндекс). Их можно комбинировать. Начинаем с (а).

## Точки правки (для implementer)

### 1. Новый чистый модуль `src/helm/infrastructure/rag2/query_expand.py`
- `expand_code_query(query: str, *, max_added: int = 16) -> str` — чистая, тестируемая (как `query_rewrite.expand_query`). Сохраняет оригинал, добавляет варианты в хвост, дедуп, лимит.
- **split**: для каждого токена детектить границы: camelCase (`[a-z0-9]→[A-Z]`, `[A-Z]+→[A-Z][a-z]`), буква↔цифра, snake `_`, kebab `-` → добавить расщеплённую форму через пробел (`messageSync`→`message sync`, `message_sync`→`message sync`).
- **join**: maximal-run из ≥2 подряд ЛАТИНСКИХ словных токенов → добавить слитную lowercase-форму (`message sync`→`messagesync`). Только латиница (кириллицу НЕ склеивать — шум); ограничить длину и `max_added`.
- Оба — идемпотентно и без дублей; кириллические/обычные RU-запросы проходят без изменений.
- Тесты (юнит, без инфры): `messageSync`→содержит `message sync`; `message sync`→содержит `messagesync`; `message_sync`→`message sync`; RU-запрос без латиницы → без изменений; идемпотентность.

### 2. `src/helm/infrastructure/rag2/search.py` — `JiraIndexSearch`
- В `__init__` добавить параметр `expand_code_terms: bool = False` → `self._expand_code_terms`.
- В `search()` (стр.150-151) после `text = query.strip()` и guard, ЗЕРКАЛЯ RAG#1 (`docs_assistant/search.py:112-118`):
  ```
  if self._expand_code_terms:
      try: text = expand_code_query(text)
      except Exception: log.warning(...)  # сбой не роняет поиск
  ```
- `text` уходит в `_semantic/_fulltext_only/_hybrid` (и эмбеддинг, и Meili) — как в RAG#1. **rerank НЕ трогаем**: `search.py:163` уже реранкует по исходному `query`, не по `text` — расширение не портит реранк. ✓
- (эмбеддинг расширенного текста нейтрален/слабо-полезен, как и в RAG#1; основной эффект — на Meili-ветке.)

### 3. `src/helm/config.py` — флаг (рядом с rag2-флагами ~171, стиль как `docs_query_rewrite_enabled` стр.113-115)
- `rag2_query_expand_code: bool = False` (env `HELM_RAG2_QUERY_EXPAND_CODE`), комментарий «за флагом: мерим эффект на golden, не форсируем».

### 4. `src/helm/infrastructure/rag2/service.py` — прокидка
- `Rag2Config`: добавить поле `query_expand_code: bool`; в `from_settings` — `query_expand_code=bool(g("rag2_query_expand_code", False))`.
- `build_rag2_context`: прокинуть `expand_code_terms=cfg.query_expand_code` в оба ft-корпуса — `JiraIndexSearch` для jira (стр.183) и confluence (стр.203). helm — semantic без Meili, можно не прокидывать.

## Приёмка (verifier добавит golden-кейсы)
- **Offline**: `python -m helm.eval.rag2_eval --k 5` по golden. Новые кейсы: query `messageSync` И `message sync` → ожидать page-id 208701103 в top-k. Мерить recall@k / MRR: флаг OFF (baseline) vs ON. Критерий: ON поднимает 208701103 в top-k для ОБОИХ запросов, без регресса общих метрик.
- **Live**: MCP `analyst_search("messageSync")` и `analyst_search("message sync")` (`interface/mcp/analyst_server.py`) на стеке с `HELM_RAG2_QUERY_EXPAND_CODE=true` → doc 208701103 выше порога в обоих.

## Якоря
- Паттерн-образец RAG#1: `docs_assistant/search.py:83-118` (флаг+вызов), `docs_assistant/query_rewrite.py` (чистый expand).
- Точки: `rag2/search.py:126-165`, `config.py:113-115,167-231`, `service.py:34-112,183-211`.
- Приёмка: `eval/rag2_eval.py`, `tests/data/rag2_golden.json`.
- Предыдущая разведка: [[rag2-p3-tokenize-SUMMARY]].
