---
title: rag2-eval-baseline-v2
type: report
permalink: tacticum/00-inbox/rag2-eval-baseline-v2
tags:
- rag2
- eval
- baseline
- verifier
- helm
- meili
- retrieval
---

# RAG#2 eval — baseline v2 (прогон за гейтом, prod read-only)

Verifier. База `main` b5d1739 (сверил md5 eval-модулей контейнер = `/opt/helm` = main). Прогон **read-only** в проде: `helm-helm-1`, stores `10.16.0.19` (Qdrant :6333, Meili :7700). Данные не менялись. Секреты не выводились (Meili-ключ — из env внутри контейнера).

## TL;DR
- **Retrieval baseline снят (25 кейсов, 9 с ключами — малая выборка!): почти нулевой.** recall@1/3/5 = **0.0**, только recall@10 = hit@10 = **0.1111** (1 из 9), MRR=0.0185, nDCG@10=0.0396. Маршрутизация 1.0 везде. no_answer_rate=0.
- **Корень диагностирован** (не пробел корпуса): (1) федерация jira/confluence/helm — **round-robin RRF** (cross_rerank=OFF) → релевантный jira-корпус размывается; (2) внутри jira **точный ключ не всплывает** даже с per-corpus rerank+hybrid — запрос с «IVAONE-7752» возвращает IVAONE-6138/5261 выше цели. Ключи В индексе есть.
- **Критично для Р-2/Р-3:** floor по confidence на **RRF-fused score бесполезен** — все скоры ≈**0.016** (1/(60+1)), не разделяют релевантное/шум. Floor нужен на raw rerank score, а сейчас cross-rerank выключен.
- **Meili-эмпирика Р-3:** doc **208701103 есть** и в Qdrant, и в Meili → «нет в индексе» опровергнуто; корень — **camelCase-токенизация** (`messageSync` vs `message sync`) + burial.

## Как прогонял (для воспроизводимости)
Код в контейнере = main (md5 eval совпал с `/opt/helm`). golden скопирован в контейнер:
```
docker cp /opt/helm/tests/data/rag2_golden_profile.json helm-helm-1:/tmp/
for k in 1 3 5 10; do
  docker exec helm-helm-1 python3 -m helm.eval.rag2_eval \
    --golden /tmp/rag2_golden_profile.json --k $k --out /tmp/base_k$k.json
done
```
`--live` off, `--judge` off (детерминизм). Сырьё: на хосте `helm:/tmp/base_k{1,3,5,10}.json` (per-item+aggregate) и `.log`.
⚠️ `--out` пишет JSON **внутрь контейнера**; логи-редиректы — на хост. Скопировал JSON на хост через `docker cp`.

## Baseline-числа (main, «before», floor off)
25 кейсов профиля, retrieval считается по **9** размеченным (`expected_keys`), фактов для answer_in_context — **0/25** (не мерилось).

| k | recall@k | hit@k | nDCG@k | MRR | no_answer | route_mode_acc |
|---|---|---|---|---|---|---|
| 1 | 0.0000 | 0.0000 | 0.0000 | 0.0000 | 0.0 | 1.0 |
| 3 | 0.0000 | 0.0000 | 0.0000 | 0.0000 | 0.0 | 1.0 |
| 5 | 0.0000 | 0.0000 | 0.0000 | 0.0000 | 0.0 | 1.0 |
| 10 | **0.1111** | 0.1111 | 0.0396 | 0.0185 | 0.0 | 1.0 |

(MRR rank-free, но растёт с k, т.к. единственный хит появился только в top-10 — на ранге ~6.)
Пометка: **выборка 9/25 — статистически мала**, числа индикативны, не финальны. structural_coverage=0 (граф-ось не сработала на этих кейсах). latency ~6.5–9 c/кейс.

## Диагностика корня (прогон оркестратора на 3 кейсах, k=10)
Извлечённые ключи = round-robin по корпусам с одинаковым fused-score:
```
a-status-01 exp=IVAONE-7752 HIT=False → [160713585(conf), 1.0-411(helm), IVAONE-6138(jira), 160705099(conf), 1.0-335(helm), IVAONE-5261(jira), ...]  все score=0.016
a-status-02 exp=IVAONE-3262 HIT=False → [..., IVAONE-11250, ..., IVAONE-11390, ..., IVAONE-9022]  0.016
a-status-04 exp=IVAONE-4430 HIT=True  → [..., IVAONE-4429, ..., IVAONE-4430(#6), ..., IVAONE-4431]  0.016
```
Наблюдения:
1. **Round-robin федерации.** `federate(by_source, limit=pool)` сливает 3 корпуса по РАНГУ (RRF, `cross_rerank_enabled=False`) → в top-10 попадает лишь ~3 jira-дока. Даже jira#1 оказывается на общем ранге ~3, jira#2 ~6, jira#3 ~9. Confluence/helm занимают 2/3 выдачи независимо от релевантности вопроса.
2. **Точный ключ не побеждает внутри jira.** Запрос содержит «IVAONE-7752», но jira отдаёт IVAONE-6138/5261/7862 выше цели. Per-corpus rerank (`rag2_rerank_enabled=True`, `tacticum/rerank`) + hybrid Meili **не бустят точное совпадение ключа** — семантический реранк топит лексический сигнал. Это hypothesis (b) burial, но на стадии per-corpus.
3. **Fused score неинформативен.** Все хиты ≈0.016 = `1/(rrf_k=60 + rank0 + 1)`. RRF схлопывает шкалу → **порог по этому score не отделит шум**. Для floor>0 нужен либо raw cross-rerank score (сейчас OFF), либо per-corpus rerank score до RRF.

Конфиг подтверждён: `rag2_search_mode=hybrid`, `rag2_rerank_enabled=True`, `rag2_cross_rerank_enabled=False`.

## Meili/Qdrant эмпирика (Р-3, doc 208701103 «Документация JUMP», space IVAQA)
- **Qdrant `iva_confluence__bge_m3_1024`** (92 374 точки): doc есть — **6 чанков** по `page_id/key/source_doc_id=208701103`. Терм `messageSync` в тексте ровно 1 раз (chunk 3, camelCase); `message sync` (с пробелом) — 0.
- **Meili `iva_confluence`** (92 374 док): doc есть.
  - `q="messageSync"` → estTotal=**2**, doc 208701103 (chunk 3) на **#1**.
  - `q="message sync"` → estTotal=**1000** (matchingStrategy=all: **22**), doc **не в top-20**.
  - Meili settings: `searchable=['*']`, `separatorTokens=[]`, `nonSeparatorTokens=[]`, `dictionary=[]` → **camelCase не расщепляется**, `messageSync` = один токен.
- **Вывод по гипотезам аудита:**
  - (а) «страницы нет в индексе» — **ОПРОВЕРГНУТО** (есть и в Qdrant, и в Meili).
  - (в) **токенизация — ПОДТВЕРЖДЕНА как корень**: пользовательский `message sync` не матчит camelCase-токен `messageSync`; точный терм даёт #1. Фикс: настроить Meili `separatorTokens`/dictionary или нормализовать camelCase при индексации/запросе.
  - (б) burial в cross-rerank — для `message sync` провал уже на **лексике** (до реранка); для точного терма doc #1. Отдельно подтверждён burial на **per-corpus rerank** в jira-кейсах выше.

## Дизайн доказательного A/B: negative-набор + precision@k / noise_kept_rate
Цель — доказать «шум отсечён» при **floor>0 vs floor=0**, не сломав recall на позитивах.

### Где floor (текущее состояние)
Порога сейчас **нет**: `application/rag2.py:223` → `no_answer = not hits and structural is None` (отдаёт всё, что нашлось). `JiraDoc.score` после `federate` = **RRF-fused (~0.016)** — по нему floor не работает. Ввод floor: параметр `rag2_score_floor` в `Rag2Config`/`RagPolicy`, применять к хитам **после rerank, до федерации** (на per-corpus rerank score) либо включить `cross_rerank` и резать по его score. Иначе floor нерелевантен.

### 1. Negative-набор (кейсы «ожидается not-found/шум»)
Схема кейса (в тот же `cases[]`), новый флаг:
```json
{ "id": "neg-ood-01", "role":"analyst", "category":"negative",
  "query":"<вне корпуса / несуществующая фича>",
  "expected_mode":"index", "expected_keys":[], "expected_not_found": true,
  "difficulty":"hard" }
```
Два подтипа: (a) **hard-negative** — правдоподобный, но без релевантного дока (тест отсечения пограничного шума floor>0); (b) **out-of-domain** — явно вне домена (санити).
Точки правки: `eval/rag2_golden.py` — поле `expected_not_found: bool|None` в `Rag2GoldenCase` (~стр.44) + парс `_bool_or_none(item.get("expected_not_found"))` (~стр.128).

### 2. Метрики в `metrics.py`
- **`precision_at_k(retrieved, relevant, k)`** — чистая ф-я рядом с `recall_at_k` (~стр.69): `|retrieved[:k] ∩ rel| / min(k, len(retrieved[:k]))`; пустой retrieved → 0.0. На позитивах ловит, не выкинул ли floor истинные хиты.
- **`noise_kept_rate`** (агрегат по negative-кейсам, в `rag2_runner.aggregate`): доля negative-кейсов, где пайплайн **всё равно вернул ≥1 хит выше floor / не выдал no_answer**. Ниже = лучше. Комплемент — **`correct_rejection_rate = 1 − noise_kept_rate`** (fail-closed на негативах).
- Опц. **`false_positive@k`** — число хитов на negative-кейсе (ожидалось 0).

### 3. Точки правки runner
`rag2_runner.evaluate` (~стр.262): ветка по `case.expected_not_found` — для негативов считать noise (вернул ли выше-floor хиты / no_answer). `aggregate` (~стр.321): добавить `noise_kept_rate`/`correct_rejection_rate`; recall/precision разделить на positive-only.

### 4. Протокол A/B (before/after порога)
Один снапшот корпуса/Qdrant для обоих прогонов; live/judge off. Матрица приёмки:
- **Позитивы:** recall@k/hit@k/nDCG@k **не должны просесть** (или в пределах допуска); precision@k — не выросли ли потери.
- **Негативы:** `noise_kept_rate` **должен упасть** (главный выигрыш), `correct_rejection_rate` вырасти.
- Trade-off: floor режет и шум, и хвост recall — смотреть оба разреза; правка порога особенно бьёт по `no_answer_rate ↔ recall`.

## Вердикт (acceptance #32)
- Харнесс/метрики валидны; маршрутизация 1.0.
- **Retrieval baseline снят, но КРИТИЧЕСКИ низкий** (recall@10=0.111 на 9 точках) — это честный «floor off / before». **Не acceptance-годен сам по себе**, но это и есть отправная точка: у Р-2/Р-3 есть куда расти.
- **Блокеры доказательства до правок confidence/порога:**
  1. floor нельзя вешать на RRF-fused score (≈0.016) — сначала включить cross-rerank или резать по per-corpus rerank score;
  2. корень низкого recall — федерация round-robin + burial точного ключа, а не сам порог; порог этого не починит;
  3. golden ретрив-беден (9/25, факты 0/25) — нужна разметка + negative-набор (≥10).
- Meili-токенизация (Р-3) — подтверждённый отдельный фикс, не зависящий от floor.

## Связанные
- Предыдущая заметка: `tacticum/00-Inbox/rag2-eval-baseline-SUMMARY` (оффлайн route-only, разбор харнесса).
