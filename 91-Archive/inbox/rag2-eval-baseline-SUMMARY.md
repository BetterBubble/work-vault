---
title: rag2-eval-baseline-SUMMARY
type: report
permalink: tacticum/91-archive/inbox/rag2-eval-baseline-summary
tags:
- rag2
- eval
- baseline
- verifier
- helm
status: archived
updated: 2026-07-18
---

# RAG#2 eval — baseline на `main` (b5d1739)

Verifier-разбор харнесса `helm/src/helm/eval`. База: `main` b5d1739, репо `/Users/bubblemac/tacticum/helm`. Код НЕ правился (read-only измерения).

## TL;DR
- **Метрики (`metrics.py`) корректны** — сверил вручную recall/hit/MRR/nDCG на 3 кейсах, всё сходится (в т.ч. multi-relevant nDCG@3 = 0.9197).
- **Route-only baseline снят оффлайн** на профильном golden (25 кейсов): `route_mode/structural/temporal/conf_body_acc = 1.0000`, 0 расхождений.
- **Retrieval baseline (recall@k/MRR/nDCG) НЕ снят — заблокирован дважды:**
  1. нужен full pipeline = Gateway (embed/rerank) + Qdrant с наполненным индексом (+опц. Meili) → прод/стенд-гейт;
  2. **даже с бэкендом golden не размечен под ретрив**: `expected_keys` заполнены лишь в **9 из 25** кейсов профиля; 68-кейсовый файл вообще другой схемы (см. ниже) и даёт **0** размеченных ретрив-кейсов.

## Как запускается харнесс
CLI-модуль: **`python -m helm.eval.rag2_eval`** (файл `src/helm/eval/rag2_eval.py`). `rag2_runner.py` — библиотека функций (`evaluate_routes`/`evaluate`/`aggregate`), CLI там нет.

Команда (оффлайн route-only):
```
.venv/bin/python -m helm.eval.rag2_eval --golden tests/data/rag2_golden_profile.json --route-only --out /tmp/run.json
```
Команда (full pipeline, на сервере в контейнере — как ингест):
```
docker exec helm-helm-1 python -m helm.eval.rag2_eval \
    --golden /app/tests/data/rag2_golden_profile.json --k 5 --out /app/rag2_eval_run.json
```
Флаги: `--golden PATH` (иначе env `HELM_RAG2_GOLDEN`, иначе дефолт), `--k K` (дефолт 5), `--limit N`, `--route-only`, `--live` (по умолч. выкл — детерминизм), `--judge` (платно, LLM), `--judge-model`, `--out PATH`.

### Две плоскости оценки
1. **route-only** (`evaluate_routes`) — БЕЗ бэкенда. Чистый роутер `domain.rag2.classify` над query, сверка `expected_mode/structural/temporal/needs_confluence_body`. Мерит только маршрутизацию. **Работает оффлайн уже сейчас.**
2. **full pipeline** (`evaluate` через `Rag2Orchestrator`) — retrieval-метрики по совпадению issue-key хитов с `expected_keys` (recall@k/hit@k/MRR/nDCG@k), `answer_in_context@k`, покрытие графа, degraded/no_answer, опц. judge.

## Что нужно для full pipeline (гейт)
Гейт в `infrastructure/rag2/service.py:114` — `Rag2Config.enabled`:
```
enabled = gateway_base_url AND gateway_api_key AND qdrant_url
```
Если не сконфигурирован → `build_rag2_eval_wiring` вернёт None → CLI **молча деградирует до route-only** (`_run_full` → `_run_route_only`). Т.е. «полный» прогон без бэкенда просто не даст retrieval-чисел.

Нужно для прогона retrieval:
- **Gateway** (embed `bge-m3`/rerank `tacticum/rerank`) — `gateway_base_url` + `gateway_api_key` (llm.cifragen.ru);
- **Qdrant** (`qdrant_url`) с **наполненными коллекциями**: `iva_jira__bge_m3_1024`, `iva_confluence__bge_m3_1024`, `helm_mgmt__bge_m3_1024`;
- **Meili** (опц.) для `hybrid`; без него режим честно падает в `semantic`;
- env-настройки `HELM_*` (см. `Rag2Config.from_settings`: `rag2_*`, `gateway_*`, `meili_*`, `qdrant_url`).

Оффлайн/локально full pipeline **не прогнать** — нужен прод/стенд с ингестированным индексом за гейтом секретов.

## Снятый baseline (оффлайн, route-only)
| golden | кейсов | mode_acc | struct_acc | temporal_acc | conf_body_acc | расхожд. |
|---|---|---|---|---|---|---|
| `rag2_golden_profile.json` | 25 | **1.0000** | 1.0000 | 1.0000 | 1.0000 | 0 |
| `rag2_golden.json` | 68 | — (0 опред.) | — | — | — | n/a |

Сырьё: `/tmp/rag2_baseline/route_only_25.json`, `/tmp/rag2_baseline/route_only_68.json`.

## ⚠️ Находка: два golden-файла, разные схемы
- **`rag2_golden_profile.json` (25)** — **дефолт CLI** (`_DEFAULT_GOLDEN`), схема совпадает с загрузчиком `rag2_golden.py`. Поля `expected_mode/structural/temporal/needs_confluence_body` = 25/25. `expected_keys` (issue-key для ретрива) = **9/25**. `ideal_answer`/`_key_facts` = **0/25** → `answer_in_context@k` не определён нигде.
- **`rag2_golden.json` (68)** — **другая (старая) схема**, загрузчик её поля НЕ читает:
  - `retrieval_mode` вместо `expected_mode`; голые `temporal`/`needs_confluence_body` без префикса `expected_` → все ожидания = None;
  - `expected_sources` = **имена файлов корпуса** (`req_realization.json`, `wiki_reqs_10.json`), а не issue-key; `expected_keys` отсутствуют.
  - Итог: route-only даёт 0 определённых; retrieval-метрики undefined для всех 68. **Файл в текущем виде раннером не измеряется.**

Т.е. «68 golden» из рекона ≠ рабочий набор харнесса. Рабочий — профиль на 25, и он ретрив-неполон.

## Проверка метрик (ручная сверка)
`metrics.py` — чистые функции, сверил на .venv:
- recall@k = |top-k ∩ rel| / |rel|; hit@k = 1 если есть пересечение; mrr = 1/ранг первого релевантного; ndcg@k — binary, ideal = min(|rel|,k).
- Кейс `[A,B,C,D]`, rel={C}, k=3: hit@3=1.0, recall@3=1.0, **ndcg@3=0.5** (1/log2(4)), mrr=0.333 — сходится.
- Кейс rel вне top-3: hit=0, ndcg=0, mrr=0.25 — сходится.
- Multi-relevant `[C,B,D,E]`, rel={C,D}, k=3: **ndcg@3=0.9197** — сходится с ручным (dcg=1+1/log2(4), ideal=1+1/log2(3)).
Вывод: метрикам можно доверять как опоре для before/after.

⚠️ Нюанс k: `--k` — один горизонт за прогон. Для **k=1/3/5/10** нужно **4 прогона** с разным `--k` (recall@k/hit@k/ndcg@k считаются на одном k; mrr — rank-free, инвариантен).

## План воспроизводимого A/B (baseline vs после правок confidence/порога)
Retrieval-числа — только за гейтом. Предлагаю тимлиду:
1. **Разметить golden под ретрив** — довести `expected_keys` (issue-key) хотя бы до всех 25 профиля + добавить ≥10 контрактных кейсов Р-2a. Без этого recall/MRR/nDCG статистически пусты (сейчас максимум 9 точек).
2. **Зафиксировать корпус** — один и тот же `corpus_as_of` и снапшот Qdrant-коллекций для baseline и after (иначе A/B несопоставим). `--live` держать **off** (детерминизм); judge — off (детерминизм + не платить).
3. **Прогон на стенде/проде за гейтом** внутри контейнера:
   ```
   docker exec helm-helm-1 python -m helm.eval.rag2_eval \
       --golden /app/tests/data/rag2_golden_profile.json --k 5 --out /app/base_k5.json
   ```
   повторить для `--k 1|3|10`. Сохранить `--out` артефакты обоих прогонов (before/after правок порога).
4. **Дельта** — сравнить `aggregate` из JSON: `recall@k`, `hit@k`, `mrr`, `ndcg@k`, `answer_in_context@k`, `no_answer_rate`, `degraded_rate`, `structural_coverage`. Правки confidence/порога особенно бьют по `no_answer_rate` ↔ `recall` (trade-off — смотреть оба).
5. Изоляция тенантов (cross-tenant leak / fail-closed scope_filter) — **отдельная проверка**, в этот baseline не входила; если нужна к acceptance — дайте знать, что считать утечкой и где scope_filter.

## Место для ≥10 контрактных golden (Р-2a) — только формат/место, кейсы НЕ добавлял
- **Файл**: дописать в `cases[]` профиля `tests/data/rag2_golden_profile.json`, либо отдельный файл и гонять через `--golden` / env `HELM_RAG2_GOLDEN`.
- **Схема кейса** (как читает `rag2_golden.py`):
  ```json
  {
    "id": "analyst-<категория>-<n>",
    "role": "analyst|support",
    "category": "<подкатегория>",
    "query": "<вопрос роли>",
    "expected_mode": "index|live|hybrid",
    "expected_structural": false,
    "expected_temporal": false,
    "expected_needs_confluence_body": false,
    "requires": ["jira|confluence|helm"],
    "expected_keys": ["IVAONE-XXXX"],   // ОБЯЗАТЕЛЬНО для retrieval-приёмки
    "expected_sources": ["..."],
    "ideal_answer": "...",              // для answer_in_context@k / judge
    "ideal_answer_key_facts": ["..."],
    "difficulty": "easy|medium|hard"
  }
  ```
- Ключевое для приёмки #32: у контрактных кейсов **`expected_keys` обязаны быть заполнены** — иначе retrieval-метрики по ним undefined и приёмка не докажет улучшение.

## Вердикт (acceptance)
- **Харнесс исправен, метрики верны** — опора для before/after валидна.
- **Route-only baseline: снят** (профиль 25, всё 1.0).
- **Retrieval baseline: заблокирован** — (а) прод/стенд-гейт (Gateway+Qdrant+индекс), (б) golden ретрив-неполон (9/25, 68-файл несовместим). Приёмка #32 по retrieval-числам **пока не проходит** — нужна разметка golden + прогон за гейтом.
