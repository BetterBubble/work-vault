---
title: impl-rag2-docs-corpus
type: report
permalink: tacticum/00-board/impl-rag2-docs-corpus
status: draft
repo: helm
worktree: /Users/bubblemac/tacticum/helm-wt-rag2-docs
branch: feat/rag2-docs-corpus
commit: da667ba98d96a734af03cfe9da83af348d225040
role: implementer
task: блок D техплана RAG#2 /analyst
tags:
- rag2
- helm
- docs
- impl
- demo
- analyst
---

# impl-rag2-docs-corpus — блок D (корпус документации в ретрив RAG#2)

## Что сделано
Расширил федеративный ретрив RAG#2 `/analyst`: при **включённом флаге** подмешивает коллекцию `iva_docs__bge_m3_1024` (публичная дока ИВА, та же, что RAG#1 `/docs`) к Jira/Confluence/helm. **Дефолт OFF — нулевой регресс.**

## Имя env-флага (главное)
**`HELM_RAG2_DOCS_CORPUS_ENABLED`** (поле `rag2_docs_corpus_enabled`, дефолт `False`).

Сопутствующие env (дефолты рабочие, менять не обязательно):
- `HELM_RAG2_DOCS_QDRANT_COLLECTION` = `iva_docs__bge_m3_1024`
- `HELM_RAG2_DOCS_TENANT` = `iva`
- `HELM_RAG2_DOCS_MEILI_INDEX` = `iva_docs` (hybrid; нет Meili → semantic-only)
- `HELM_RAG2_DOCS_RRF_WEIGHT` = `0.6` (вес docs в RRF, <1 чтобы не топил Jira/Confluence)

## Как включить для замера на проде
```
HELM_RAG2_DOCS_CORPUS_ENABLED=true
```
(опц. подстроить `HELM_RAG2_DOCS_RRF_WEIGHT`). Перезапуск сервиса. Коллекция `iva_docs__bge_m3_1024` уже наполнена RAG#1 — реиндекс не нужен.

## Файлы (все в worktree helm-wt-rag2-docs)
- `src/helm/config.py` — флаг + коллекция/tenant/meili/RRF-вес docs.
- `src/helm/application/rag2.py` — порт `docs` в `Rag2Orchestrator`, слот docs в `_corpora_hits`; веса `federate` и дедуп — только при активном порте (`self.docs is not None`); `Rag2Policy.docs_rrf_weight`.
- `src/helm/infrastructure/rag2/service.py` — сборка docs-стора (`DocsVectorStore`) за флагом, `source_override="docs"`, проброс веса в policy.
- `src/helm/infrastructure/rag2/search.py` — `source_override` в `_doc_from_payload` + `JiraIndexSearch` (payload доков НЕ несёт `source` → без override стал бы «jira»; форсируем «docs»). Метод `_to_doc`.
- `src/helm/domain/rag2.py` — `SOURCE_LABELS["docs"]="Документация ИВА"` + meta-ветка docs; per-source веса в `federate` (по умолчанию no-op); `drop_docs_duplicates` (дедуп docs↔Confluence/Jira по норм. заголовку).
- Тесты: `tests/domain/test_rag2.py`, `tests/application/test_rag2_orchestrator.py`, `tests/infrastructure/test_rag2_search.py`.

## Механика OFF vs ON
- **OFF** (`docs=None` в оркестраторе): слот docs не добавляется в `_corpora_hits`; `federate` вызывается без весов (`weights=None` → идентично прежнему); `drop_docs_duplicates` не вызывается. Набор источников и порядок = как раньше.
- **ON**: docs ищется изолированно (свой tenant/scope, fail-soft), хиты помечены `source="docs"`, сливаются RRF с весом 0.6, дубли docs↔Confluence по заголовку схлопываются (остаётся первоисточник).

## Числа тестов
- Затронутые + смежные rag2 (`domain/test_rag2` + `application/test_rag2_orchestrator` + `infrastructure/test_rag2_search`): **94 passed**.
- Расширенный rag2 (+confluence/extractors/interface/eval): **183 passed**.
- Полный `tests/`: **1784 passed, 31 skipped**.
- ruff: чисто на всех моих файлах (3 предсуществующие ошибки — в чужих `db/models.py`, `ingest/requirements.py`, не мои).
- mypy: `Success: no issues found in 5 source files`.

Новые тесты: OFF-регресс (набор источников), ON docs-source в ретриве, `source_override` тег, `drop_docs_duplicates`, вес docs в RRF (не топит Jira), docs-ключи не уходят в live/graph.

## Риски (отмечено)
- **Дубли docs↔Confluence:** реализован `drop_docs_duplicates` — дедуп по нормализованному заголовку (lower+схлоп пробелов), оставляет первоисточник (Jira/Confluence/helm), выкидывает docs-дубликат. Эвристика по заголовку; разный контент с совпадающим заголовком тоже схлопнется (риск низкий — docs/вики заголовки специфичны). Не роняет существующий ретрив (применяется только при активном docs).
- **Вес docs в RRF:** дефолт 0.6 (<1) — фоновый вклад, чтобы не топить Jira/Confluence. Настраивается env.
- **as_of:** docs-хиты не имеют `as_of` в payload и НЕ участвуют в live/graph (`candidate_keys` берёт только `source=="jira"`). Поэтому `as_of`/live-merge не затронуты; docs получают `as_of=corpus_as_of` через merge как index-хиты. Влияния на дисклеймеры нет.
- **/context vs /answer:** и `/context`, и `/answer` используют один `Rag2Orchestrator.answer()`. При **OFF** `/context` не меняется (проверено регресс-тестами). При **ON** docs подмешивается в ОБА эндпоинта осознанно (единый ретрив) — так и задумано, отдельного гейта на `/context` не делал.

## Границы (не трогал)
conversation-слой, design-режим, synth-изоляцию, RAG#1 `/docs` (ingest/сервис) — не менял. `DocsVectorStore` переиспользован из `docs_assistant` (только импорт, read-only контракт search).

## Дальше
Не пушил / не мержил / не деплоил. Тимлид ревьюит; включение флага на проде — пользователь.