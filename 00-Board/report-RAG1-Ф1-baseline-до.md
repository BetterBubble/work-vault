---
title: report-RAG1-Ф1-baseline-до
type: report
permalink: tacticum/00-board/report-rag1-f1-baseline-do
tags:
- rag1
- baseline
- eval
- verifier
- draft
---

# report-RAG1-Ф1-baseline-до

status: draft
роль: verifier (read-only, замеры на реальных данных)
репо: `~/tacticum/helm` @ `eaf10f8` (main, дерево чистое)
дата: 2026-07-20

## TL;DR (вердикт)
Локально снять числовой baseline **невозможно**: и ретрив, и генерация требуют прод-бэкендов ИВА, недостижимых с ноутбука. Что зафиксировано твёрдо — **структурный baseline «до»** (промпт-многословие, `max_tokens=1536`, последовательный ретрив embed→Qdrant→Meili). Числовой baseline (retrieve_latency p50/p90, длина ответа, LLM-время) **нужно снимать на helm** — команда ниже. Данных не выдумывал.

## Что реально запускается локально vs требует helm

**Локально доступно:**
- helm импортируется в `~/tacticum/helm/.venv` (Python 3.13 в pycache; venv рабочий).
- golden-сет присутствует: `~/tacticum/iva-rag1-docs/golden/golden_iva_rag1.json` — **1306 кейсов**, у всех есть `relevant_doc_ids`; **ideal_answer / ideal_answer_key_facts — НЕТ ни у одного** (0/1306).
- Gateway `llm.cifragen.ru:443` — TCP доступен.

**Требует прод-helm (локально недоступно):**
- `Settings().docs_assistant_enabled == False` с текущим `helm/.env`: заданы только gateway_*; `iva_llm_base_url`, `qdrant_url`, `meili_url` — не заданы (None).
- Qdrant `10.16.0.19:6333` — **UNREACHABLE** с ноутбука (внутренний контур).
- `build_docs_assistant_context(Settings())` → **None** → `build_eval_wiring` → None → `docs_eval` печатает `error: ассистент доков не сконфигурирован (нужны Gateway, vLLM, Qdrant)` и выходит с кодом 3. Проверено фактическим запуском на 3 кейсах.
- Генерация (`TrivaLlm`) строится из `iva_llm_base_url` (внутренний vLLM контура ИВА) — не сконфигурирован, недостижим. Длину ответа локально измерить нельзя.

Вывод: eval-harness `docs_eval` — «толстый клиент» к прод-инфре (embed через Gateway + Qdrant + Meili + vLLM). Локальных фикстур/оффлайн-режима ретрива нет by design (adapters.py импортирует реальную инфру).

## Baseline «до» — что зафиксировано

### 1. Длина ответов — структурный факт (числа НЕ измерены)
Генерация локально недоступна → средняя длина `answer.text` не снята. Зафиксирован источник многословия «до» для сравнения после правок краткости:
- `SYSTEM_PROMPT` (`application/docs_assistant.py:61-79`) — блок **ПОЛНОТА** прямо требует: «дай РАЗВЁРНУТЫЙ пошаговый ответ… Не сокращай инструкцию до одной ссылки — приводи сами действия». Тот же промпт идёт и в eval-генерацию (`adapters.py` `IvaAnswerGenerator`, `SYSTEM_PROMPT`).
- Потолок: `docs_answer_max_tokens = 1536` (`config.py:144`), применяется в `service.py:91` (`TrivaLlm(max_tokens=…)`). Это верхняя граница длины ответа «до».
- **Метрика длины для сравнения после Ф1 (краткость): средняя/медиана len(answer.text) в символах и токенах — снимается только с `--judge` на helm (генерирует ответы), иначе метрики длины нет.**

### 2. Латентность — НЕ измерена локально
`retrieve_latency_ms` (runner.py:118-120 — единый wall-clock вокруг `retriever.retrieve`) требует живого конвейера embed+Qdrant+Meili. Qdrant недостижим → p50/p90 не сняты. Агрегатор считает `retrieve_latency_ms_mean` и `_p95` (runner.py:181-182); p50/p90 в текущем агрегаторе НЕТ — есть mean и p95 (учесть при сравнении).

### 3. Последовательность ретрива «до» (структурный факт для Ф1-параллелизации)
`search.py` `_hybrid` (строки 157-187) — три сетевых вызова **строго последовательно**:
1. `vec = self._embed(query)` (:162) — Gateway embed (bge-m3);
2. `dense = self._store.search(vec, …)` (:163) — Qdrant;
3. `ft = self._fulltext.search(query, …)` (:167) — Meili;
затем RRF (`rrf_fuse`). Qdrant и Meili сейчас идут один за другим — это кандидат на параллелизацию (embed обязан быть до Qdrant; Meili можно пустить параллельно с embed+Qdrant).

## БЛОКЕР по качеству
golden `golden_iva_rag1.json` (1306 кейсов) **без ideal_answer/key_facts** → `answer_in_context@k` будет **undefined для всех** (`n_scored=0`). Считаемы только retrieval-метрики (recall@k / hit@k / MRR / nDCG@k) — `relevant_doc_ids` есть у всех 1306. `--judge` даст faithfulness/answer_relevance, но **correctness требует ideal_answer** → на этом golden неполна. Для качественного baseline «до» (не только латентность/длина) нужен golden с эталонами либо принять, что качество меряем только retrieval-метриками.

## Точная команда для полного замера на helm

Golden скопировать в контейнер, затем (по образцу ингеста):

```
docker cp ~/tacticum/iva-rag1-docs/golden/golden_iva_rag1.json helm-helm-1:/app/golden_iva_rag1.json

# A) Латентность ретрива + retrieval-метрики (дёшево, без LLM):
docker exec helm-helm-1 python -m helm.eval.docs_eval \
    --golden /app/golden_iva_rag1.json --k 5 --out /app/eval_baseline_retrieval.json

# B) + длина ответа и judge (ПЛАТНО, генерит ответы; длину брать из rows[].generated_answer):
docker exec helm-helm-1 python -m helm.eval.docs_eval \
    --golden /app/golden_iva_rag1.json --k 5 --limit 100 --judge \
    --out /app/eval_baseline_judge.json
docker cp helm-helm-1:/app/eval_baseline_judge.json ./
```

Из артефакта: `aggregate.retrieve_latency_ms_mean` / `_p95`; длина «до» = агрегировать `len(rows[i].generated_answer)` (символы) по строкам с непустым ответом. p50/p90 в агрегаторе нет — считать по `rows[].retrieve_latency_ms` вручную из JSON.

## Инструментация Ф0 (спроектировано, НЕ внедрено — я read-only)
Единый `retrieve_latency_ms` не даёт разбивки по 4 стадиям (embed / Qdrant / Meili / LLM). Предложение под env-флагом `HELM_DOCS_TIMING` (по умолчанию off, нулевой оверхед в проде):
- таймеры `perf_counter` вокруг: `_embed` (:162), `_store.search` (:163), `_fulltext.search` (:167) в `search.py._hybrid`; и вокруг `_llm.generate` в `docs_assistant.py:210`.
- аккумулировать в контекстный объект/лог `stage=embed|qdrant|meili|llm dur_ms=…`; в eval — прокинуть в `CaseResult` доп. поля `embed_ms/qdrant_ms/meili_ms` и агрегировать p50/p90 по стадиям.
- это даёт «до»-разбивку, на которой видно выигрыш параллелизации Qdrant∥Meili и эффект сокращения длины (llm_ms).

## Достаточность baseline для сравнения после Ф1
- **Для краткости (длина ответа):** структурного «до» недостаточно для числового «до/после» — нужен прогон B на helm (с `--judge`), иначе сравнивать нечем.
- **Для параллелизации (латентность):** нужен прогон A на helm; и желательно Ф0-инструментация, иначе увидим только суммарный `retrieve_latency`, а не вклад Qdrant∥Meili.
- **Кто запускает:** прогон на helm — тот, у кого доступ к контейнеру/контуру ИВА (деплой-узел, алиас ssh `helm`). Я (verifier) с ноутбука его выполнить не могу — Qdrant недостижим.

## Наблюдения / observations
- [observation] Локальный eval RAG#1 невозможен: Qdrant 10.16.0.19 недостижим, docs_assistant_enabled=False #blocker
- [observation] golden_iva_rag1.json 1306 кейсов без ideal_answer/key_facts → answer_in_context undefined, correctness-judge неполон #golden-gap
- [observation] Ретрив embed→Qdrant→Meili последовательный (search.py:162-167) — цель Ф1-параллелизации #baseline-структура
- [observation] max_tokens=1536 + промпт «РАЗВЁРНУТО, не сокращай» — источник многословия «до» #baseline-длина
