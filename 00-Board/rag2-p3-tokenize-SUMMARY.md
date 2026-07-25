---
title: rag2-p3-tokenize-SUMMARY
type: note
permalink: tacticum/00-board/rag2-p3-tokenize-summary
tags:
- rag2
- helm
- meili
- tokenization
- explore
- P3
---

# RAG#2 Р-3 (T3.1): токенизация Meili camelCase/snake — разведка

Разведка explorer по `helm@main (b5d1739)`. Read-only. Код не менялся.
Цель: почему `analyst_search("messageSync")` не находит Confluence-страницу 208701103 и как чинить токенизацию.

## TL;DR (важная коррекция вводной)
Исходная гипотеза лида «дефолтный tokenizer Meili НЕ разбивает camelCase» **по докам Meili неверна**: токенайзер Meili — **Charabia** — по умолчанию **разбивает CamelCase** (и делает это симметрично на ингесте и на запросе, до typo-matching). Значит tokenizer camelCase, скорее всего, **НЕ корневая причина** пустого результата. Перед тем как чинить токенизацию, нужно эмпирически проверить фактическое поведение Meili и, главное, **лежит ли страница 208701103 в Meili-корпусе вообще**.

## Что подтверждено по коду (факты)

### Настройка индексов Meili (`infrastructure/rag2/fulltext.py`)
- Оба индекса — `iva_jira` и `iva_confluence` — обслуживает ОДИН класс `JiraFulltextStore` (`service.py:199-201` строит его для Confluence с `confluence_meili_index`).
- `ensure()` (`fulltext.py:49-56`): `POST /indexes` (primaryKey=id) + `PUT settings/filterable-attributes`. **Больше НИКАКИХ settings**: `separatorTokens`/`nonSeparatorTokens`/`dictionary`/`synonyms`/`typoTolerance`/`searchableAttributes` НЕ задаются → индекс работает на **дефолтах Meili** (все атрибуты searchable, typo-tolerance вкл, Charabia-сегментация).
- Запрос (`fulltext.py:84-89`): `q=query` как есть + tenant/аналитик-фильтры. Предобработки токенов НЕТ.

### Индексируемый текст Confluence (`infrastructure/rag2/confluence.py`, `ingest.py`)
- Единица = чанк страницы; в Meili кладётся `{"id": uid, **payload}` (`confluence.py:538-543`), searchable-поле = `payload["text"]` (markdown-чанк) + `title`/`breadcrumb`/`labels`/`space` (все searchable по дефолту).
- HTML→markdown (`html_to_markdown`) сохраняет текст как есть. **Никакого сплита camelCase/snake на ингесте** — `messageSync` уходит в индекс как есть.
- Best-effort по Meili: сбой полнотекста НЕ роняет Qdrant (`ingest.py:544-550`) → возможна ситуация «Qdrant есть, Meili пуст».

### Query-side (`infrastructure/rag2/search.py`)
- `JiraIndexSearch.search` → `text = query.strip()` без изменений; `_hybrid`/`_fulltext_only` шлют сырой `query` в Meili (`search.py:187,199`). **Query-side сплита/расширения в RAG#2 НЕТ.**
- Hybrid = dense(Qdrant) + fulltext(Meili) → `rrf_fuse` по uid (RRF_K=60), затем rerank, cap 2/doc, top-k.

### Прецедент query-expansion (только RAG#1, НЕ RAG#2)
- `docs_assistant/query_rewrite.py`: детерминированный словарный `expand_query` + лёгкий RU-стем + `synonyms.json`. RAG#2 его **не использует**. Готовый паттерн под вариант (в), если пойдём в query-side.

### Federated-ретрив (кандидат на «burial»)
- `application/rag2.py`: морда ищет по ТРЁМ корпусам (jira/confluence/helm) с ОДНИМ `limit`, `federate(...)` сливает в пул, опц. `cross_reranker`, затем top-k. Confluence-хит конкурирует с Jira/helm → может быть вытеснен ниже порога.
- Golden прямо помечает: «требует тел Confluence, которых **пока нет** -> future» (`tests/data/rag2_golden.json:15`) → на момент калибровки тела Confluence в индексе не было. Сильный сигнал, что причина — **наполнение корпуса**, а не токенайзер.

## Вероятные корневые причины (по убыванию правдоподобия)
1. **Страницы 208701103 нет в Meili/Qdrant-корпусе** (Confluence-ингест не прогнан / вне выгрузки / Meili best-effort упал / as_of 2026-07-10). — проверять первым.
2. **Burial в federated+cross-rerank+top-k**: страница находится, но не проходит порог в смешанной с Jira/helm выдаче.
3. **Токенизация** (snake_case `message_sync`, поведение Charabia в конкретной версии Meili, термин внутри code-макроса/вложения). camelCase по докам уже бьётся — под сомнением.

## Варианты фикса (риск/объём/переиндексация)
- **(а) Meili index settings** (`separatorTokens`/`dictionary`/`nonSeparatorTokens` в `ensure()`):
  - **Ограничение**: `separatorTokens` добавляет символ-разделитель — для camelCase **бесполезен** (между `e` и `S` нет символа). `dictionary` может форсить отдельные термины, но это ручной список, не общий сплит.
  - Объём: малый (1 PUT в `ensure()`). Риск: низкий по коду, но **требует переиндексации** обоих корпусов, чтобы применилось к уже записанным докам. Помогает в основном snake/разделителям, не camelCase.
- **(б) Предобработка на ингесте** (добавлять расщеплённую форму `messageSync`→`message sync` в доп-поле/текст):
  - Надёжно и симметрично контролируемо; **требует полной переиндексации Confluence-корпуса** (и по-хорошему Jira). Объём: средний (утилита сплита + правка `build_*_document`). Риск: рост индекса, дубли токенов.
- **(в) Query-side expansion** (бить camelCase/snake в запросе, добавлять варианты — по образцу `expand_query` RAG#1):
  - **Самый дешёвый и без переиндексации**, обратимо, за флагом (как в RAG#1). Объём: малый. Риск: минимальный, можно мерить на golden. Работает независимо от того, что уже в индексе.

**Предварительная рекомендация:** НЕ коммитить фикс токенизации вслепую. Сначала (0) эмпирически: проверить, что реально возвращает Meili на `messageSync`/`message sync` и есть ли doc страницы 208701103 в индексе. Если корневая причина — токенайзер, то самый дешёвый и безопасный путь — **(в) query-side split за флагом** (нет переиндексации, паттерн уже есть в RAG#1). (а) отдельно почти не решает camelCase; (б) — если нужен контроль на ингесте, но ценой переиндексации.

## Приёмка (до/после)
- **Offline harness**: `src/helm/eval/rag2_eval.py` (`python -m helm.eval.rag2_eval --golden ... --k 5`) считает recall@k / MRR / answer_in_context@k по golden (`tests/data/rag2_golden*.json`). Кейса под `messageSync`/208701103 сейчас НЕТ — нужно добавить golden-кейс (query→ожидаемый page-id) и мерить recall@k до/после.
- **Live**: MCP `analyst_search` (`src/helm/interface/mcp/analyst_server.py`) — прямой прогон `analyst_search("messageSync")` на боевом стеке; сравнить с прямым CQL.

## Файлы-якоря
- `infrastructure/rag2/fulltext.py:49-56,84-89` — настройка индекса и запрос
- `infrastructure/rag2/service.py:194-211` — сборка Confluence-корпуса (тот же store)
- `infrastructure/rag2/confluence.py:325-367,532-543` — build документа + запись в Meili
- `infrastructure/rag2/search.py:147-219` — query-side + hybrid/RRF
- `application/rag2.py:150-185` — federate трёх корпусов + cross-rerank
- `docs_assistant/query_rewrite.py` — готовый паттерн query-expansion (RAG#1)
- `eval/rag2_eval.py`, `tests/data/rag2_golden.json:15` — приёмка + сигнал «тел Confluence пока нет»
