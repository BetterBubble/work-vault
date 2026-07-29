---
title: explore-rag1-conditional-rerank-optionC
type: note
permalink: tacticum/00-board/explore-rag1-conditional-rerank-option-c
status: draft
role: explorer
repo: helm (/Users/bubblemac/tacticum/helm)
tags:
- rag1
- rerank
- latency
- explore
- draft
archived-at: 2026-07-29 18:12
---

# Разведка: опция C «условный реранк» для RAG#1

Задача: оценить осуществимость варианта C — реранкать не всегда, а только на неоднозначной выдаче; на «уверенных» запросах пропускать реранк (~1.2с экономии). Только анализ, не реализация.

## 1. Где хук и какой сигнал доступен ДО реранка

Хук: `DocsAssistant.ask`, `src/helm/application/docs_assistant.py:216-219` — блок
`if self._reranker is not None: chunks = list(self._reranker.rerank(question, chunks, top_n=len(chunks)))`.
Решение «реранкать/нет» вставлялось бы прямо перед этим вызовом.

Единственный сигнал до реранка — `DocChunk.score` (`domain/docs.py:63`), который проставляет `search`:
- **hybrid**: RRF fused-score из `rrf_fuse` (`infrastructure/docs_assistant/fusion.py:12`, вызов `search.py:181`), `RRF_K=60`.
- **semantic-only fallback** (Meili упал, `search.py:170`): сырой score Qdrant (косинус/dot).

Надёжность сигнала — **низкая, для гейта непригоден**:
- RRF некалиброван и крошечный: `Σ w/(k+rank+1)`. Топ обоих каналов на rank0 ≈ `2/61 ≈ 0.033`; топ одного канала ≈ `1/61 ≈ 0.016`. Диапазон значений ~0.01..0.03.
- Абсолютная величина RRF отражает **совпадение рангов dense+ft каналов**, а не семантическую релевантность. Два совершенно разных по качеству запроса с одинаковой ранговой структурой дадут одинаковый top-score. «Уверенный топ» по RRF = «один док возглавил оба канала», что ортогонально тому, отвечает ли он на вопрос.
- Шкала **гетерогенна**: hybrid RRF (~0.03) vs fallback vector-score (иная шкала) — единый порог гейта поставить нельзя.

Вывод: надёжного дешёвого сигнала уверенности без реранка нет.

## 2. Конфликт с Ф2-clarify — ДА, лобовой и жёсткий

`decide_retrieval_action` (`domain/docs.py:315`, ветка `feat/rag1-ph2-clarify`) сравнивает `c.score` с калиброванными порогами `tau_answer=0.55`, `tau_floor=0.30` (config ph2: `docs_clarify_tau_answer/tau_floor`), шкала 0..1 от Gateway `/rerank`.

Иммунитет гейта (`return "answer"`) срабатывает ТОЛЬКО когда `score is None` у всех чанков. Но `search` **всегда** проставляет score (RRF). Значит при пропуске реранка:
- чанки несут RRF-score ~0.03 (не None) → иммунитет НЕ срабатывает;
- гейт: `top ≈ 0.03 < tau_floor = 0.30` → **`"not_found"` на КАЖДОМ fast-path запросе** → ложные отказы на «уверенных» запросах. Катастрофа.
- Обойти можно только принудительно обнулив score на пропуске (`score=None`) → тогда иммунитет → всегда `"answer"`, но clarify для этих запросов **умирает полностью**.

Любой из путей: C и Ф2-clarify несовместимы ровно на той доле трафика, которую C хочет ускорить.

Тот же слом — у анти-галлюцинационного `rerank_floor` (`ask` ~223-225, `config.docs_rerank_floor`): тоже калиброванный 0..1 порог, тоже сломается на RRF-score.

## 3. Риск качества

Пропуск реранка по некалиброванному RRF рискован: высокий RRF-топ НЕ коррелирует с реранкнутым топом — в этом и смысл кросс-энкодера (пере-скорит порядок). Выигрыш recall@5 даёт реранк именно на кейсах, где RRF-порядок неверен, но «выглядит уверенно». Определить per-query, переставил бы реранк выдачу или нет, **без запуска реранка нельзя**. Гейт не отличит «реранк не нужен» от «реранк критичен» → риск для recall на части трафика, неизмеримый дёшево.

## 4. Сложность C vs A

- **A (cap кандидатов)**: сейчас `rerank(..., top_n=len(chunks))` реранкует все ~30 кандидатов (`SEARCH_LIMIT=30`) — отсюда и ~1.2с (стоимость ≈ линейна по числу пар). Cap до ~10-12 → меньше пар → дешевле. Правка почти однострочная, **калиброванные score сохраняются для ВСЕХ запросов** → clarify-гейт и rerank_floor не тронуты. Детерминированно, без per-query решений, self-contained, низкий риск. Ветка `feat/rag1-ph3-rerankcap` — сейчас placeholder (== ph2, своих коммитов нет), заведена под это.
- **C (условный реранк)**: требует (a) калиброванного pre-rerank сигнала, которого нет; (b) нового порога под тюнинг/eval; (c) примирения с clarify-гейтом И rerank_floor (оба на калиброванной шкале); (d) нормализации гетерогенных шкал (hybrid RRF vs fallback vector). Высокая сложность, лобовое столкновение с двумя фичами, экономит 1.2с лишь на подмножестве, рискуя recall.

## Вывод

**C не оправдан.** Нет надёжного дешёвого сигнала до реранка (RRF ~0.01-0.03, некалиброван, меряет overlap каналов, не релевантность); лобовой конфликт с Ф2-clarify (`tau_floor=0.30` vs RRF `0.03` → ложные `not_found`) и с `rerank_floor`; риск recall неизмерим per-query. Вариант A даёт основную долю выигрыша латентности при доле сложности C и без конфликтов. Рекомендация разведки: C не строить, идти вариантом A.

## Файлы
- `src/helm/application/docs_assistant.py:200-279` (`ask`), хук 216-219, floor 223-225
- `src/helm/infrastructure/docs_assistant/search.py:181-187` (RRF), `:34-58` (score в DocChunk)
- `src/helm/infrastructure/docs_assistant/fusion.py:12` (`rrf_fuse`, RRF_K=60)
- `src/helm/infrastructure/docs_assistant/reranker.py:40-58` (калиброванный score 0..1)
- `src/helm/domain/docs.py:315-350` (`decide_retrieval_action`), `:63` (`DocChunk.score`)
- config ph2: `docs_clarify_tau_answer=0.55`, `docs_clarify_tau_floor=0.30`, `docs_rerank_floor`