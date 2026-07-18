---
title: rag2-dotyazhka-ab-result
type: report
permalink: tacticum/00-inbox/rag2-dotyazhka-ab-result
tags:
- rag2
- exact-key
- dotyazhka
- recall
- ab-test
- verifier
- helm
- tenant-isolation
status: archived
updated: 2026-07-18
---

# RAG#2 exact-key дотяжка — A/B результат (главный recall-фикс)

Verifier, изолированный прогон **read-only** (sidecar `helm-helm` + overlay ветки `feat/rag2-exact-key-boost` через PYTHONPATH; helm-helm-1 не нагружался; env-file без вывода секретов, overlay/patch/probe удалены). База main b5d1739, те же 9 размеченных golden. Deps ветки не тронуты. Флаг `HELM_RAG2_EXACT_KEY_BOOST` (дефолт ON).

## TL;DR
- **Главный recall-фикс работает: recall@10 0.111 → 0.778 (7×), recall@1 0 → 0.778, MRR 0.019 → 0.778.**
- **`fetch_by_keys` вживую работает** на `iva_jira` (IVAONE-7752 → 6 чанков, 2 ключа → 16).
- **tenant-изоляция держит (fail-closed):** чужой tenant=helm → 0, пустой tenant → 0, несуществующий ключ → 0. (бонус к #32)
- **Регресса нет:** keyless-кейс (a-cross-02) — no-op; RU/обычные не хуже.
- **⚠️ 1 баг-edge: a-graph-04 НЕ починился** (7/8 key-кейсов, не 8/8). Точная причина ниже — impl-фикс.

## 1. fetch_by_keys вживую + tenant-изоляция (прямой вызов store)
`JiraVectorStore.fetch_by_keys` на реальном `iva_jira__bge_m3_1024`:
| вызов | результат |
|---|---|
| `["IVAONE-7752"]`, tenant=iva | **6 чанков** ✅ |
| `["IVAONE-7752","IVAONE-3262"]`, tenant=iva | **16 чанков** (оба) ✅ |
| `["IVAONE-7752"]`, tenant=**helm** | **0** ✅ (ключ есть в iva, но чужой tenant → пусто) |
| `["IVAONE-7752"]`, tenant=**""** | **0** ✅ (fail-closed без сети) |
| `["IVAONE-DOESNOTEXIST"]`, tenant=iva | **0** ✅ |

→ Qdrant scroll с фильтром `key`(any)+`tenant_id` работает без keyword-индекса; **cross-tenant не течёт** (обязательный tenant в must-фильтре).

## 2. recall A/B (9 размеченных)
| метрика | OFF (boost=0) | ON (boost=1) |
|---|---|---|
| recall@1 | 0.000 | **0.778** |
| recall@3 | 0.000 | **0.778** |
| recall@5 | 0.000 | **0.778** |
| recall@10 | 0.111 | **0.778** |
| MRR | 0.019 | **0.778** |
| nDCG@10 | 0.040 | **0.778** |

7 из 9 кейсов → ожидаемый ключ на **#1** (пин). Из 8 key-in-query кейсов исправлено **7/8**.

## 3. Per-case (ранг ожидаемого ключа OFF → ON)
| кейс | key в запросе | OFF | ON |
|---|---|---|---|
| a-status-01 IVAONE-7752 | да | MISS | **@1** |
| a-status-02 IVAONE-3262 | да | MISS | **@1** |
| a-status-04 IVAONE-4430 | да | @6 | **@1** |
| a-graph-01 IVAONE-655 | да | MISS | **@1** |
| a-graph-02 IVAONE-6118 | да | MISS | **@1** |
| a-graph-04 IVAONE-6206 | да | MISS(fed) | **MISS** ⚠️ |
| a-cross-02 IVAONE-4357/4309 | **нет** | MISS | MISS (no-op, корректно) |
| a-temporal-01 IVAONE-12541 | да | MISS | **@1** |
| a-temporal-02 IVAONE-1 | да | MISS | **@1** |

## ⚠️ Баг-edge: a-graph-04 (impl-фикс)
Диагностировано (fetch/extract работают на всех уровнях):
- `issue_keys_in_query(q)` → `('IVAONE-6206',)` ✅
- `store.fetch_by_keys(['IVAONE-6206'])` → 4 чанка ✅; `search.fetch_by_keys(['IVAONE-6206'])` → 4 docs ✅
- Но в `answer()` IVAONE-6206 **не попадает в hits** (top5 без него), тогда как a-graph-01/status-01 (тоже index/структурные) — пин @1.

**Корневая причина (высокая уверенность):** `_exact_key_dotyazhka` фетчит ТОЛЬКО отсутствующие ключи:
```
present = {h.key.upper() for h in jira_hits}
missing = [k for k in keys if k.upper() not in present]
if not missing: return jira_hits   # <-- пропуск fetch
```
Для a-graph-04 IVAONE-6206 **был в jira-выдаче, но на низком ранге** (baseline-диаг: jira full(rerank,30)**@13**) → `present`=True → дотяжка НЕ фетчит/не поднимает вперёд. Дальше `federate` round-robin (3 корпуса) уносит jira@13 за `pool` → в финальном `index_hits` его нет → `boost_exact_keys` **нечего пинить** (пин работает только по тем, кто дожил до index_hits). Для 7 промахов ключ был полным MISS → дотяжка фетчит и ставит в голову jira → federate ~#3 → пин @1.

**Т.е. дыра ровно там, где hybrid извлёк ключ, но низко** — `present`-проверка обезоруживает дотяжку, а федерация всё равно топит.

**Кандидаты-фиксы (impl):**
1. Считать `present` по позиции: дотягивать/поднимать ключ, если он НЕ в топ-N jira (а не «есть где-либо»); либо всегда prepend найденный ключ в голову jira.
2. `boost_exact_keys` — если ключ отсутствует в финальном `index_hits`, до-фетчить и вставить (пин с гарантией наличия), а не только реордерить присутствующие.
3. Простейшее: применять дотяжку-prepend безусловно для распознанных ключей (present-check убрать) — fetch дёшев, дедуп есть.

Оценка: закрытие даст **8/8 key-кейсов** (recall ~0.889 на 9; на key-кейсах 1.0).

## Вердикт
- **fetch_by_keys вживую — работает.** ✅
- **tenant-изоляция — держит (fail-closed).** ✅ (в зачёт #32)
- **recall-фикс — подтверждён, крупный: 0.111 → 0.778.** ✅ Главный recall-корень (retrieval-miss exact-key) закрыт для явных ключей.
- **Регресса нет** (keyless no-op). ✅
- **Остаётся 1 edge (a-graph-04)** — present-but-low-rank; impl-фикс поднимет до 8/8. Рекомендую пофиксить до мержа (или зафиксировать как известный follow-up).
- Семантический корень (a-cross-02, ключа в запросе нет) дотяжкой НЕ закрывается — это отдельный трек (hybrid-ребаланс/entity-aware), как в `rag2-retrieval-miss-diag`.

## Артефакты
- Сырьё армов: `helm:/tmp/dot_off.json`, `helm:/tmp/dot_on.json` (per-case ordered keys+source+score).
- Связанные: `00-Inbox/rag2-retrieval-miss-diag`, `00-Inbox/rag2-cross-rerank-ab-result`, `00-Inbox/rag2-eval-baseline-v2`.


---

## UPDATE: edge-фикс (aca27a2) — перегон, итог 8/8

Перегнал ON с edge-фиксом (present-but-low ключи переставляются в топ jira-пула до federate). Тот же изолированный sidecar, read-only, за собой убрал.

| метрика | OFF | ON (edge-fix) |
|---|---|---|
| recall@1 | 0.000 | **0.889** |
| recall@5 | 0.000 | **0.889** |
| recall@10 | 0.111 | **0.889** |
| MRR | 0.019 | **0.889** |
| nDCG@10 | 0.040 | **0.889** |

Per-case: **a-graph-04 (IVAONE-6206) MISS → @1** (edge починен); 7 прежних остались @1; a-cross-02 (keyless) → MISS (корректный no-op). **8/8 key-in-query кейсов на #1.**

tenant-изоляция при перегоне снова держит (чужой tenant=helm → 0, пустой → 0).

**Вердикт: дотяжка готова к мержу.** Единственная оставшаяся MISS (a-cross-02) — keyless, семантический трек, дотяжкой не адресуется. Сырьё: `helm:/tmp/dot_off.json`, `dot_on2.json`.
