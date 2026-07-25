---
title: baseline-rag1-golden184-judge-2026-07-21
type: note
permalink: tacticum/01-sessions/baseline-rag1-golden184-judge-2026-07-21-1
status: current
role: rag1-lead
project: helm / RAG#1
tags:
- rag1
- baseline
- golden-184
- judge
- correctness
- real-data
---

# RAG#1 baseline golden-184 (judge, полный) — 2026-07-21

Полный judge-прогон на прод-контейнере helm-helm-1 (задеплоенный стек ph1-ph5), golden `golden_eval200.json` = 184 кейса с эталонами, k=5, judged=184. Артефакт `/app/eval184_baseline.json` (сотрётся при rebuild).

## Числа (ТВЁРДЫЙ baseline на полных 184)
| метрика | значение |
|---|---|
| judge.correctness | **0.768** |
| judge.faithfulness | **0.982** |
| judge.answer_relevance | 0.952 |
| answer_in_context@5 | 0.894 |
| context_hit@5 | 0.940 |
| recall@5 | 0.973 |
| hit@5 | 0.973 |
| mrr | 0.873 |
| ndcg@5 | 0.897 |

Срезы: role admin n=101 aic 0.89; end_user n=38 aic 0.93; difficulty typical n=106 aic 0.91, hard n=59 aic 0.86.

## Контекст
- Прежний ориентир «до» (60 verbose кейсов): aic@5 0.879 / correctness 0.796 / faithfulness 0.963 — согласуется, теперь есть полный 184.
- Faithfulness 0.982 — галлюцинаций почти нет. Correctness 0.768 ограничен корпусом/двусмысленностью, не форматом.
- **Дифф к деплою (calibrated clarify + conversation-context) ИНЕРТЕН** (флаги OFF, retrieval не тронут) → «после» == этот baseline. Реальный прирост — при включении clarify на тесте post-deploy.
- Латентность в прогоне (3601мс) — под нагрузкой judge, НЕ показатель живой латентности.

Связи: [[summary-rag1-quality-3issues-ready]] · [[diag-rag1-issue1-correctness-realdata]] · [[session-state]]