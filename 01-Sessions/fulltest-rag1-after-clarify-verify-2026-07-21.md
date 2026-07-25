---
title: fulltest-rag1-after-clarify-verify-2026-07-21
type: note
permalink: tacticum/01-sessions/fulltest-rag1-after-clarify-verify-2026-07-21-1
status: current
role: rag1-lead
project: helm / RAG#1
tags:
- rag1
- fulltest
- after
- clarify
- verification
- real-data
---

# RAG#1 фул-тесты «после» + clarify-верификация (2026-07-21, vLLM восстановлен)

vLLM triva поднялся ~15:29, оркестратор сам прогнал обе проверки (FULLTEST_EXIT_0).

## [1] golden-184 judge «после» vs baseline — РЕГРЕССА НЕТ
| метрика | до | после | Δ |
|---|---|---|---|
| judge.correctness | 0.768 | **0.778** | +0.010 |
| judge.faithfulness | 0.982 | **0.984** | +0.002 |
| judge.answer_relevance | 0.952 | 0.947 | −0.005 |
| aic@5 | 0.894 | 0.892 | ~0 |
| recall@5 | 0.973 | 0.973 | 0 |
Всё в пределах шума → деплой (calibrated clarify + conversation-context, флаги OFF) **инертен и безопасен**. Качество не просело. Артефакт `/app/eval184_after.json`.

## [2] clarify-верификация (флаг ON, реальный LLM, clarify_cases) — 7/10, НЕ ГОТОВ
- **Двусмысленные (expect clarify): 3/5 (recall 0.60).** ПРОМАХ на флагманском «Как добавить человека в идущий ВКС?» (ответил, не переспросил) + «пригласить на встречу». Это ровно баг Лебедева — clarify его НЕ ловит.
- **Однозначные (expect answer): 4/5 — 1 ложное** («во время идущего группового звонка…» → переспросил про продукт).
- Вывод: LLM сильно триггерит на ПРОДУКТОВУЮ двусмысленность (даже лишне), слабо — на ТЕМПОРАЛЬНУЮ (идущий звонок vs мероприятие), а именно темпоральная — суть бага.

## РЕШЕНИЕ
- **clarify в проде ОСТАВЛЯЕМ OFF** — калибровка не прошла порог «по делу, не ложно» (плановое условие включения). Нужна ещё итерация CLARIFY_INSTRUCTION: усилить темпоральный триггер (идущий ВКС/звонок/встреча), приглушить лишний продуктовый переспрос. Затем повторить verify_clarify.py.
- Деплой стека остаётся валидным (безопасен, инертен, качество не просело).
- conversation-context функционально ещё не проверен (нужен multi-turn прод-тест) — отдельно.

Связи: [[baseline-rag1-golden184-judge-2026-07-21]] · [[deploy-rag1-quality-done-2026-07-21]] · [[summary-rag1-quality-3issues-ready]]