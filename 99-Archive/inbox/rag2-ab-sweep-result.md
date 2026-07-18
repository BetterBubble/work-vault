---
title: rag2-ab-sweep-result
type: report
permalink: tacticum/00-inbox/rag2-ab-sweep-result
tags:
- rag2
- ab-test
- hybrid
- floor
- cross-rerank
- verifier
- eval
status: archived
updated: 2026-07-18
---

# RAG#2 A/B свипы (hybrid / floor / cross) на 45-golden

Verifier, read-only, изолированные sidecar-прогоны (overlay main cec722f для hybrid/cross, confidence-ветка для floor; helm-helm-1 не трогал, env-file без вывода секретов, артефакты убраны). Golden — расширенный набор Task #4 (14 positive / 12 negative). Метки сверены.

## TL;DR — ни один флаг не даёт recall-выигрыша
Главный recall-выигрыш **уже дала дотяжка** (0.111→0.929, в main). Три свипа:
- **hybrid ft-weight↓ — РОНЯЕТ recall** (0.929→0.857). Оставить ft=1.0 (дефолт).
- **cross-rerank — recall@10 не меняет** (0.929), recall@1/5 чуть хуже; helm-регресс 3.0→1.76 (A2 смягчает, но не убирает). Ценность — только score-база для floor.
- **floor τ≈0.7 — noise_kept 1.0→0.833 (−17%)**, recall безопасен (dotyazhka-хиты conf=None не режутся). Модест, но чистый плюс.

## 1. Hybrid ft-weight свип (14 positive)
| ft-weight | recall@1 | @3 | @5 | @10 | MRR |
|---|---|---|---|---|---|
| **1.0 (дефолт)** | 0.643 | 0.857 | **0.929** | **0.929** | 0.732 |
| 0.5 | 0.643 | 0.786 | 0.857 | 0.857 | 0.708 |
| 0.0 | 0.643 | 0.786 | 0.857 | 0.857 | 0.708 |

**Вывод:** снижение веса ft-канала (Meili) **ухудшает** recall (−0.07 @10, 1 кейс выпал). Моя прошлая гипотеза «мёртвый ft вредит» (retrieval-miss-diag, кейсы a-status-04/graph-04) — case-specific, на полном positive-наборе ft net-полезен (semi/confluence-кейсы выигрывают от лексики). **Ft-downweight НЕ включать.** NL-гейтинг (ft↓ только для длинных NL) мог бы помочь точечно, но на метке net нейтрально-негативно.

## 2. Cross-rerank дельта (14 positive)
| | recall@1 | @3 | @5 | @10 | MRR | helm в top-10 |
|---|---|---|---|---|---|---|
| OFF (baseline) | 0.643 | 0.857 | 0.929 | **0.929** | 0.732 | **3.00** |
| ON (cross+A2) | 0.571 | 0.857 | 0.857 | **0.929** | 0.71 | **1.76** |

- **recall@10 не изменился** (0.929), recall@1/5 чуть просели. Cross-rerank recall не поднимает (подтверждает прошлый A/B: корень был retrieval-miss, закрыт дотяжкой).
- **helm-регресс:** OFF 3.0 → ON 1.76 helm-хитов в top-10. Source-aware A2 смягчает (раньше без A2 было 0), но helm всё равно частично вытесняется.
- **Score-база:** cross даёт min=0.001 / med=0.858 / max=0.999 (vs вырожденный RRF 0.016) — реальная шкала (полезно, но floor использует `confidence`, не этот score).

## 3. Floor τ-калибровка (confidence 0..1, per-corpus rerank)
Confidence-раздача (floor_run):
- **positive key-hits:** n=6, min=**0.719**, med=0.936, max=0.989 (высокая). *8 из 14 позитивов — дотяжка-хиты с `confidence=None` → floor их НИКОГДА не режет.*
- **negative hits:** n=80, min=0.002, med=0.592, max=**0.985** (перекрытие с позитивами — rerank переуверен на off-topic).

τ-свип (drop-action, 12 негативов / 14 позитивов):
| τ | noise_kept (neg) | recall@10 (pos) |
|---|---|---|
| 0.0 (off) | 1.000 | база |
| 0.5 | 0.833 | цел |
| 0.70 | **0.833** | **цел** |
| 0.72 | 0.833 | −1 позитив |
| 0.80 | 0.667 | −1 позитив |

**Калибровка: τ≈0.7** — noise_kept 1.0→**0.833 (−17%)** при сохранённом recall (dotyazhka conf=None + rerank-позитивы ≥0.719 выживают). τ=0.8 режет больше шума (−33%) но роняет rerank-позитив (min 0.719). **Рекомендую τ=0.7, action=drop.**
Ограничение: floor не отсекает шум чисто — негативы имеют high-confidence хиты (rerank переуверен). −17% при τ=0.7 — потолок без потери recall.

## ⚠️ Caveats (честно)
- **floor_run — на confidence-ветке (база БЕЗ дотяжки)** → её recall@10=0.429 (exact-key кейсы отсутствуют), НЕ прод-репрезентативно. **τ-калибровка (confidence-раздача) валидна**, но финальную floor-валидацию гнать на **confidence-ветке, ребейзнутой на main cec722f** (дотяжка+hybrid). Тогда dotyazhka-позитивы (conf=None) подтверждённо не режутся.
- Hybrid-свип 3-точечный (1.0/0.5/0.0), 0.5=0.0 → тренд монотонный, 0.7/0.3 не нужны.
- Выборка мала (14 pos / 12 neg). Прогоны медленные (Gateway congested, ~30мин/25 кейсов) → сузил hybrid до positive-сабсета, гнал 4 арма параллельно.

## Рекомендации под включение флагов
| флаг | вердикт |
|---|---|
| `HELM_RAG2_HYBRID_FT_WEIGHT` | **оставить 1.0** (снижение роняет recall) |
| `HELM_RAG2_CROSS_RERANK_ENABLED` | recall не даёт, helm-регресс (−40% helm-хитов); включать ТОЛЬКО если нужна score-база под floor и helm-регресс приемлем |
| `HELM_RAG2_NOISE_FLOOR` | **τ=0.7, action=drop** — модест −17% шума, recall safe; сначала re-validate на дотяжка-базе |
| `HELM_RAG2_EXACT_KEY_BOOST` (дотяжка) | **главный recall-фикс, уже в main, ON** (0.111→0.929) |

## Связанные
- `00-Inbox/rag2-golden-expand-result`, `rag2-dotyazhka-ab-result`, `rag2-cross-rerank-ab-result`, `rag2-retrieval-miss-diag`.


---

## UPDATE: floor re-validate на ДОТЯЖКА-базе (confidence-ветка ребейз на cec722f) — τ=0.5, НЕ 0.7

Прогон OFF vs `NOISE_FLOOR=0.7 drop` на 45-golden, дотяжка-база (6ebe807, cec722f-предок), rerank ON, таймаут-защита (0 скипов).

**Находка: τ=0.7 роняет recall** (0.929→0.857) — режет **a-graph-04** (IVAONE-6206), у которого key-hit confidence=**0.505** (present-but-low rerank, НЕ conf=None). τ-свип на дотяжка-базе:
| τ | recall@10 (14 pos) | noise_kept (12 neg) |
|---|---|---|
| 0.0 (off) | 0.929 | 1.000 |
| 0.40 | 0.929 | 0.833 |
| **0.50** | **0.929** | **0.833** |
| 0.505 | 0.857 | 0.833 |
| 0.70 | 0.857 | 0.833 |
| 0.80 | 0.786 | 0.750 |

key-hit confidence позитивов (возр.): a-graph-04 **0.505**, a-rationale-01 0.719, a-status-04 0.84, s-similar-03 0.912, s-similar-01 0.96, s-coverage-03 0.962, a-reqtext-01 0.989. Дотяжка conf=None: **6 из 14** (floor не режет — подтверждено).

**Вывод: правильный τ = 0.5** (не 0.7). noise_kept 1.0→0.833 (**−17%**), recall@10 **ЦЕЛ 0.929**, 0 ложных срезов. τ=0.7 даёт ТОТ ЖЕ noise_kept (0.833), но зря режет a-graph-04 → строго хуже. noise_kept плато 0.833 на τ∈[0.4;0.7] → −17% это потолок без потери recall (негативы имеют high-conf шум до 0.985, rerank переуверен).

**Рекомендация под деплой: `NOISE_FLOOR=0.5`, action=drop** (не 0.7). Re-validate на дотяжка-базе поймал ложный срез a-graph-04 при 0.7 — ровно то, ради чего гоняли.
