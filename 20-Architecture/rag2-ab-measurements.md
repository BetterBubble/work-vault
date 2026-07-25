---
title: rag2-ab-measurements
type: report
permalink: tacticum/20-architecture/rag2-ab-measurements-1
tags:
- rag2
- ab-test
- measurements
- recall
- golden
- benchmark
- helm
---

## Замеры RAG#2 — A/B на реальном Qdrant (2026-07-17)

Конкретные числа для дальнейшей работы. Замеры: sidecar/overlay кода веток против живого корпуса Qdrant `10.16.0.19`, golden 45 кейсов (25 profile / 12 contracts / 8 negative), **14 positive размеченных**, 12 negative. Выборка мала — числа ориентировочные, но направление устойчиво. Метод и итоги замеров: заметка [[session-state]], карта корпуса [[rag2-corpus-map]].

## Baseline (main, floor OFF, дотяжка ON)
- positive(14): recall@1/3/5/10 = **0.643 / 0.857 / 0.929 / 0.929**, precision@5=0.186, MRR=0.732.
- semi-5 recall@5=1.0; orig-9 recall@10=0.889.
- **noise_kept_rate = 1.0** на 12 негативах (floor OFF — весь шум удержан, 0 no_answer). ← опора для floor.

## Дотяжка (exact-key retrieval) — главный фикс
- recall@1/5/10: **0.000/0.000/0.111 (OFF) → 0.889/0.889/0.889 (ON)** на orig-9; на расширенном positive recall@10=0.929. MRR 0.019→0.889.
- tenant-изоляция держит (чужой/пустой tenant → 0). В проде, дефолт ON.

## Hybrid ft-weight свип (14 positive) — снижение РОНЯЕТ recall
| ft_weight | recall@5 | recall@10 | MRR |
|---|---|---|---|
| **1.0** | **0.929** | **0.929** | 0.732 |
| 0.5 | 0.857 | 0.857 | 0.708 |
| 0.0 | 0.857 | 0.857 | 0.708 |
→ ft-канал net-полезен, оставить 1.0 (флаг off, не включать). Гипотеза «мёртвый ft вредит» — case-specific, на полном наборе неверна.

## Cross-rerank (+A2) — recall не меняет, helm-регресс
| | recall@10 | recall@1 | helm в top-10 | score-шкала |
|---|---|---|---|---|
| OFF | 0.929 | 0.643 | 3.00 | RRF ~0.016 |
| ON+A2 | 0.929 | 0.571 | 1.76 | 0.001–0.999 |
→ recall@10 без изменений, recall@1 хуже, helm −40% (A2 смягчает, не убирает). Ценность только score-база. Флаг off, не для recall.

## Floor τ-калибровка (по confidence 0..1, НЕ RRF-score)
- positive key-hits confidence ≥0.719; **8/14 позитивов — дотяжка-хиты confidence=None** (floor их НЕ режет — recall safe). Негативы: median 0.592, max 0.985 (перекрытие).
- τ=0.7 → noise_kept **1.0→0.833 (−17%)**, recall цел; τ=0.8 → −33% шума, но роняет 1 rerank-позитив.
→ **ФИНАЛ (re-validate на дотяжка-базе): τ=0.5, action=drop.** НЕ 0.7! τ=0.7 режет позитив a-graph-04 (дотяжка present-but-low rerank-хит confidence=0.505<0.7) → recall@10 0.929→0.857 просел. τ=0.5: тот же −17% шума (noise_kept 1.0→0.833), recall ЦЕЛ 0.929, 0 ложных срезов. noise_kept плато 0.833 на τ∈[0.4;0.7]. Прошлый замер τ=0.7 был на ветке БЕЗ дотяжки (caveat) — там a-graph-04 не было. **Урок: re-validate на прод-релевантной базе ловит ложные срезы (не включать порог вслепую по замеру на неполной базе).**

## Р-5 корень (не замер, но данные)
Changelog покрывает **377/8119 (4.6%)** EpicTask (IVAONE 212/7669=2.8%, IVAONEHALF 165/450=37%). extract гонялся узко под IVAONEHALF/velocity. Фикс — broad-extract + переингест.

## Relations
- relates_to [[session-state]]
- relates_to [[rag2-corpus-map]]
- part_of [[20-Architecture]]


## Р-5 ПЕРЕСМОТР (2026-07-17) — changelog УЖЕ в Qdrant, PAT НЕ нужен
**Прежний вывод «нужен bulk-extract 8119 через Jira PAT» — НЕВЕРЕН.** Проверка Qdrant `10.16.0.19`:
- `iva_jira__bge_m3_1024`: 319 303 точки, **319 047 has_changelog=true**. IVAONE — 87 097 точек, **87 053 с changelog**.
- **Переходы статусов с таймстемпами лежат в `payload.text`** (плоско): `[2024-03-22T09:52+0300] status: In Progress → In Review`. IVAONE-1175=11 переходов, IVAONE-3803=6. Парсится regex `\[ts\] status: X → Y`.
- Структурированного `changelog.histories` в payload НЕТ — только флаг + плоский текст переходов.
**Почему effort_hint null-медианы:** читает СТРУКТУРИРОВАННЫЙ changelog `[{field,from,to,created}]` из УЗКОГО корпуса EpicTask/`helm_mgmt` (400 pts, из velocity-дампа 379 задач — `data/iva/tasks_changelog/` 379). Богатый iva_jira игнорирует. Корень — источник, не объём выгрузки. Мой «4.6%» мерил узкий корпус.
**Локальные дампы:** `data/iva/tasks_rich/` = 8119 задач (IVAONE 7669+IVAONEHALF 450) но БЕЗ changelog (компактный мета: k/type/st/spent). `data/iva/tasks_changelog/` = 379 задач {k,chlog,cmts} (узкий).
**Решение Р-5 (без PAT/extract/туннеля 8443):** распарсить переходы из iva_jira.text → структура → наполнить EpicTask.changelog для IVAONE/IVAONEHALF → переингест → численные медианы. **УРОК: проверять, нет ли данных УЖЕ в проде (Qdrant payload), прежде чем планировать новую выгрузку — [[verify-data-credibility]].**


## Аудит полноты Р-1/Р-4 (r5-and-datamap, 2026-07-17) — потерь 0
Проверка источник → реестр (read-only по файлам, retrieval не гоняли):
- **Р-1 API: 419/419** — clients 342 + integration 72 + bot 5 (все 8 HTTP-методов). Источник (paths×methods) = manifest = фактические `operations.json`. Ни потерь, ни дублей, ни схлопываний. **Число «410» устарело — реестр = 419** (полнее заявленного). Прод: `/opt/helm/data/real/api/`.
- **Р-4 JUMP: 101/101** — Sessions.html 105 h3 → 101 команда (4 отброшены верно: проза «Логин»/MIME/VTODO, команд среди них нет), дублей 0. Прод: `/opt/helm/data/real/contracts/`.
Вывод: Р-1/Р-4 зелёные по ПОЛНОТЕ корпуса, не только по приёмочным кейсам. Детерминированные, риска артефакта узкой выборки нет.


## ШИРОКИЙ ПЕРЕМЕР (640 реальных IVAONE-кейсов, 2026-07-17) — floor drop ВРЕДИТ
Прогоны на сервере (`/tmp/run_{A,B,P,C}.json`, golden 640: 300 key_lookup + 300 title_dense + 40 negative_ood). Метод: live Rag2Orchestrator, k=10, env-override дотяжки/floor.

| Прогон | recall@10 | MRR | key_lookup(300) | title_dense(300) |
|---|---|---|---|---|
| A baseline (дотяжка off, floor off) | 0.617 | 0.167 | **0.303** | 0.93 |
| B дотяжка ON (floor off) | **0.967** | 0.642 | **1.00** | 0.933 |
| P ПРОД (дотяжка + floor τ=0.5 drop) | 0.703 | 0.429 | **0.463** | 0.943 |
| C dense-only (ft0, rerank off) | 0.977 | 0.230 | 0.993 | 0.96 |

**ВЫВОДЫ:**
1. **Дотяжка подтверждена СИЛЬНЕЕ узкой оценки:** key_lookup 0.303→1.00. Прежние 0.11→0.93 занижали. 🟢
2. **🔴 floor τ=0.5 drop в ПРОДЕ роняет key_lookup 1.00→0.463** — эмпирическое подтверждение дефекта floor×pin (floor дропает запиненные exact-key). Активный вред: по ключу аналитик находит задачу в ~46% вместо 100%.
3. **КОРРЕКЦИЯ:** прежний вывод «floor τ=0.5 safe» верен ТОЛЬКО на узком golden. На широком — floor массово рушит key. Урок повторно: узкая выборка врёт ([[verify-data-credibility]]).
4. dense-only силён по recall (0.977) но слаб по MRR (0.23 vs дотяжка 0.64) — дотяжка ценна ранжированием на rank1.
**СЛЕДСТВИЕ:** floor×pin фикс (ветка `fix/rag2-audit-fixes`, pinned иммунны к drop) — не «хорошо бы», а СРОЧНО в прод. До деплоя фикса прод недо-отдаёт key-запросы.


## 🟢 floor×pin VERDICT — ФИКС РАБОТАЕТ (2026-07-18, deploy-verify на c2796a5)
Пост-деплой прогон на фикшеном коде (golden 640, дотяжка ON, floor drop, live):
- **key_lookup τ=0.5 (прод): 0.463 (старый) → 1.00 (новый).** floor×pin (JiraDoc.pinned иммунитет) восстановил полностью — запиненные exact-key floor не трогает. Аналитик по ключу 100% (было ~46%). **Р-2 зелёный вживую.**
- effort_hint численный (ORDER BY), тулы живы. Все 5 фиксов подтверждены на реальном проде.

**τ-рекалибровка (новый код):**
| τ | recall@10 | MRR | key_lookup | title_dense | OOD no_ans |
|---|---|---|---|---|---|
| 0.5 (прод) | 0.9733 | 0.6969 | 1.00 | 0.9467 | 0.950 (38/40) |
| 0.6 | 0.9733 | 0.7021 | 1.00 | 0.9467 | 0.950 (38/40) |
| 0.7 | 0.9750 | 0.7067 | 1.00 | 0.9500 | 0.975 (39/40) |
**key_lookup=1.0 на ВСЕХ τ** (пин-иммунитет держит при любом пороге). **Рекомендация τ=0.7 (маргинальна: +1 OOD-негатив, без вреда key/title, регрессий ноль). РЕШЕНИЕ: прод-τ оставлен 0.5** (работает, выигрыш 0.7 маргинальный) — τ=0.7 опция пользователю. Сервис цел весь прогон (CPU пики 12%, MEM<450MiB). Отчёт [[deploy-verify-result]].