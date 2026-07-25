---
title: rag2-cross-rerank-ab-result
type: report
permalink: tacticum/91-archive/inbox/rag2-cross-rerank-ab-result-1
tags:
- rag2
- eval
- ab-test
- cross-rerank
- verifier
- helm
- retrieval
status: archived
updated: 2026-07-18
---

# RAG#2 cross-rerank A/B — результат (A0/A1/A2)

Verifier. Изолированный прогон **read-only**, sidecar из образа `helm-helm` (`docker run --rm --network host --env-file <из helm-helm-1, не печатался> --entrypoint python3`) — **helm-helm-1 НЕ трогался**, Qdrant/Meili только чтение, секреты не выводились. Overlay ветки через `PYTHONPATH`-prepend (подтверждён: `reranker.__file__ = /tmp/helm-tuned-src/...`). Кейсы — те же 9 размеченных golden, что и baseline. Deps ветки не менялись (только `src/helm/*`+тесты). После прогона все временные артефакты и env-file удалены.

## TL;DR
- **Cross-rerank НЕ чинит recall.** recall@10 = **0.1111** во всех армах (A0/A1/A2). 8 из 9 ожидаемых ключей **отсутствуют даже в пуле k=30** → это **retrieval-miss (ключ не извлечён), а не burial реранка**. Cross-rerank лишь переупорядочивает пул — помочь принципиально не может.
- **Микро-выигрыш от cross-rerank есть:** единственный извлечённый ключ (IVAONE-4430) поднялся off@6 → on@4 (recall@5: 0→0.111, MRR 0.0185→0.0278).
- **helm-регресс ПОДТВЕРЖДЁН:** cross-on полностью вытесняет helm-корпус (off: 3 helm в top-10 → on: **0**). A2 source-aware возвращает ровно **1 helm на фикс-ранг 8** через rank-floor (форс-инъекция с низким скором).
- **Главный побочный выигрыш — скор-база для floor τ:** off = вырожденные ~0.016 (floor невозможен) → on/tuned = **реальные 0.72–0.97** (floor осмыслен). Но позитив (0.84) в диапазоне шума (0.72–0.97) → τ по скору один не разделит; нужен negative-set для калибровки.
- **A2 = A1 по recall** на этой выборке (source-aware влияет на helm-корпус, а тут позитивы — jira-ключи).

## Таблица recall (9 размеченных, малая выборка)
| метрика | A0 off (baseline) | A1 on (main+cross) | A2 tuned (ветка+cross) |
|---|---|---|---|
| recall@1 | 0.0 | 0.0 | 0.0 |
| recall@3 | 0.0 | 0.0 | 0.0 |
| recall@5 | 0.0 | **0.1111** | **0.1111** |
| recall@10 | 0.1111 | 0.1111 | 0.1111 |
| hit@5 | 0.0 | 0.1111 | 0.1111 |
| nDCG@10 | 0.0396 | 0.0479 | 0.0479 |
| MRR | 0.0185 | 0.0278 | 0.0278 |

Улучшение только в перемещении **одного** уже-извлечённого дока (IVAONE-4430) в top-5. recall@10 неизменен — новых релевантных не добавилось.

## Per-case (ранг ожидаемого ключа; MISS = нет в top-10)
| кейс | expected | off | on | tuned |
|---|---|---|---|---|
| a-status-01 | IVAONE-7752 | MISS | MISS | MISS |
| a-status-02 | IVAONE-3262 | MISS | MISS | MISS |
| a-status-04 | IVAONE-4430 | @6 | **@4** | **@4** |
| a-graph-01 | IVAONE-655 | MISS | MISS | MISS |
| a-graph-02 | IVAONE-6118 | MISS | MISS | MISS |
| a-graph-04 | IVAONE-6206 | MISS | MISS | MISS |
| a-cross-02 | IVAONE-4357/4309 | MISS | MISS | MISS |
| a-temporal-01 | IVAONE-12541 | MISS | MISS | MISS |
| a-temporal-02 | IVAONE-1 | MISS | MISS | MISS |

## Решающий диагностик: k=30 пул (retrieval-miss vs rerank-burial)
Прогнал `answer(k=30)` (cross-on). **8 из 9 ключей отсутствуют и в top-30** (только IVAONE-4430 @4). Значит ожидаемые доки **не попадают в пул кандидатов вообще** — провал на стадии **извлечения/candidate-generation**, до реранка. Запрос буквально содержит «IVAONE-7752», но exact-key jira-док не извлекается (при том что он ЕСТЬ в Qdrant/Meili iva_jira — см. baseline-v2). Вероятные корни (retrieval-side): лексический (Meili) exact-key матч по key-формату не срабатывает / тонет в dense-фьюжене; per-corpus лимит кандидатов; вес гибрида. **Cross-rerank этого не лечит.**

## helm-регресс (детально)
| | off | on | tuned |
|---|---|---|---|
| helm в top-10 (все кейсы) | 3 (ранги 2,5,8) | **0** | 1 (ранг 8) |

- **on**: cross-rerank скорит helm-mgmt (`1.0-xxx`) низко для jira-issue-запросов → вылетают полностью. Это тот самый регресс, под который сделан A2.
- **tuned (A2)**: rank-floor принудительно возвращает 1 helm на ранг 8 с его ИСХОДНЫМ (низким) скором (0.007–0.35), вытесняя jira-кандидата с ранга 8 на 9. На этой выборке recall не пострадал (позитив на ранге 4), но структурно A2 меняет jira-слот на helm-слот в хвосте. На запросах, где ответ ИМЕННО в helm (набор B кита), A2 нужен; здесь его эффект нейтрален к recall.

## Распределение rerank-скоров (для выбора floor τ, Р-2a)
Кейс a-status-04, top-10:
- **off:** `[0.0164, 0.0164, 0.0164, 0.0161, 0.0161, 0.0161, 0.0159, …]` — вырожденный RRF, **floor невозможен**.
- **on:** `[0.969, 0.919, 0.881, 0.840(IVAONE-4430✓), 0.814, 0.803, 0.780, 0.776, 0.774, 0.716]` — реальный диапазон.
- **tuned:** как on + инъекция helm `0.0078` на ранге 8.

Вывод по τ: cross-rerank **необходим** как score-база (без него floor не на чём считать). Но на этом кейсе истинный позитив (0.84) неотличим по скору от нерелевантного хвоста (0.72–0.97) — весь top-10 «на тему». Значит **floor по абсолютному rerank-скору сам по себе не отсечёт шум чисто** → нужен negative-set (кейсы not-found) для калибровки τ и метрики noise_kept_rate (дизайн в `rag2-eval-baseline-v2`).

## Вердикт
- **Гипотеза «cross-rerank > round-robin RRF → recall↑» — НЕ подтверждена** на этой выборке. Cross-rerank чинит порядок и score-базу, но не recall: корень — **retrieval-miss (exact-key доки не в пуле)**, глубже федерации.
- **A2 source-aware** решает свою узкую задачу (не даёт helm-корпусу занулиться), но на recall jira-ключей нейтрален. Под мерж — как фикс helm-регресса, НЕ как способ поднять recall.
- **Что даёт cross-rerank под прод (за/против):** (+) осмысленные скоры → открывает floor/confidence Р-2a; (+) чуть лучше ранг извлечённого; (−) обнуляет helm-корпус без A2-флора; (−) recall не растёт. Т.е. включать cross-rerank имеет смысл ВМЕСТЕ с A2 (source-aware) и как предпосылку для floor — но проблему recall это не закрывает.
- **Следующий шаг (корень recall) — retrieval-side, НЕ rerank:** почему exact-key jira-доки не в пуле k=30. Диагностика: (1) прямой Meili-поиск iva_jira по «IVAONE-7752» и по key-фильтру; (2) per-corpus candidate limit и вес dense↔lexical в гибриде; (3) exact-key boost/роутинг по распознанному ключу (роутер уже извлекает `keys` — можно точечно дотягивать по key-фильтру).

## Артефакты
- Сырьё армов: `helm:/tmp/probe_{off,on,tuned}.json` (per-case ordered keys+source+score).
- Baseline: `helm:/tmp/base_k{1,3,5,10}.json`.
- Связанные заметки: `00-Board/rag2-eval-baseline-v2`, `00-Board/rag2-eval-baseline-SUMMARY`.