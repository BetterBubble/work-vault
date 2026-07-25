---
title: diag-rag1-issue1-correctness-realdata
type: note
permalink: tacticum/00-board/diag-rag1-issue1-correctness-realdata-1
status: current
role: rag1-lead
project: helm / RAG#1
tags:
- rag1
- diagnosis
- correctness
- retrieval
- real-data
---

# RAG#1 проблема 1 (корректность) — диагноз на реальных данных (прод helm, 2026-07-21)

Диагностика через `docs_eval` + точечный retrieval-probe на **прод-контейнере helm-helm-1**, golden `golden_iva_rag1.ambiguous.json` (10 кейсов: 5 «активный звонок» + 2 контроль-звонок + 3 контроль-мероприятие). Воспроизведён живой баг Лебедева.

## Агрегаты (ambiguous-10, k=10)
aic@10 0.92 · hit@10 0.90 · **mrr 0.58 · ndcg 0.66** → правильная страница ИЗВЛЕКАЕТСЯ, но НЕДОРАНЖИРОВАНА (ранги 2-3), кроме 1 recall-miss (iOS).

## Per-item (ранг правильной `*-ug-calls`)
- web#1 ранг 1 ✅ · android#1 ранг 3 · **ios#1 MISS (нет в топ-8, aic 0.40)** · web#2 ранг 3 · connect-android#1 ранг 2.
- контроль-event: ios ранг 1, android ранг 1, web#1 ранг 6.

## Что стоит ВЫШЕ правильной страницы (probe, упорядоченные slug)
**Механизм 1 — event-кластер перебивает звонок (ЭТО БАГ):** для «добавить участника в активный вызов» топ занимают `*-ug-event-interface-*-participant` / `*-event-management-user-actions`. Худший `amb-active-call-ios#1`: ранги 1-5 — сплошь event-interface-participant (connect-ios, desktop, web, mcu, connect-android), правильной `one-ios-ug-calls` нет в топ-8. Семантика «добавить **участника**» ≈ «участник **мероприятия**»; темпоральный сигнал «идущий/активный» слишком слаб.
**Механизм 2 — sibling-продукты (benign):** правильный звонок часто ниже call-страниц ДРУГИХ продуктов (connect/one-web/desktop). Топик верный, путается только продукт → лечится clarify.

## Два корня
1. **Лексика/синонимы:** `synonyms.json` группа ВКС = `[вкс, видеоконференция, конференция, совещание, **мероприятие**, встреча, meeting]` — отдельно от `[вызов, звонок, call]`. Запрос «идущий ВКС» лексически расширяется в «мероприятие» → тянет event-кластер. `--compare`: hybrid mrr **0.342** (ХУЖЕ) vs semantic **0.661** vs hybrid+rerank **0.583** — лексический путь ухудшает ранжирование на этих кейсах.
2. **Семантика:** bge-m3 сам ставит event-participant страницы высоко для «добавить участника».

## Вывод по фиксам
- **Clarify (проблема 2) — правильный ПЕРВИЧНЫЙ рычаг:** двусмысленность (активный звонок vs мероприятие; какой продукт) реальна и измерима. Откалибровано в ветке `feat/rag1-correctness-clarify`. Бот переспрашивает вместо угадывания. Низкий риск регресса (gated OFF до валидации на тесте post-deploy).
- **Ранжирование (проблема 1) — кандидат: правка синонимов ВКС↔мероприятие.** Реальный, но РИСКОВЫЙ рычаг: глобальная правкa синонимов/режима может уронить широкий golden-1306 (прод-оптимум hybrid+rerank recall@5 0.949). Требует A/B на golden-184 (retrieval-only быстро, judge дорого) ПЕРЕД любым прод-изменением. Глобально менять режим ретрива нельзя.

## Артефакты на проде
`/app/golden_ambiguous.json`, `/app/amb_diag.json`, `/app/probe_rank.py` (temp, сотрутся при rebuild). Локально golden: `iva-rag1-docs/golden/golden_iva_rag1.ambiguous.json`.

Связи: [[plan-rag1-quality-3issues-korrektnost-clarify-kontekst-chata-novomu-lidu]] · [[explore-rag1-retrieval-clarify-map]] · [[report-rag1-corrclar-groundwork]]