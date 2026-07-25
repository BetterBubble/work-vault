---
title: cross-rerank-tune IMPL-PLAN
type: handoff
permalink: tacticum/91-archive/inbox/cross-rerank-tune-impl-plan
author: implementer (team helm)
date: 2026-07-16
tags:
- task
- rag2
- rerank
- implementer
- helm
- cross-rerank
- plan
status: archived
updated: 2026-07-18
---

# План правки cross-rerank helm-тюнинг — (г)+(б)

Worktree `~/tacticum-worktrees/helm-cross-rerank-tune`, ветка `fix/rag2-cross-rerank-helm-tune` (от origin/main a0435f6, PR#53 на месте). Диагноз PLAN-заметки подтверждён дословным чтением кода. **Код НЕ трогал — жду одобрения.**

## Подтверждение диагноза (реальные строки)
- `ingest/helm_mgmt_index.py:204` — `embed_text = f"{piece}\n\n{context}"`, а `:205 payload["text"]=piece` (сводки НЕТ). Сводка = `_requirement_context` (`:140-155`): Целевое поколение / Компоненты / Заказчики / **Покрытие: TRACK — покрыто; …**.
- `infrastructure/rag2/search.py:73 _doc_from_payload` — `text=payload["text"]` (обеднённый). Из helm-полей мапит только `entity_type/verdict(агрег.)/track/component/status`.
- `domain/rag2.py:113 jira_rerank_text` — `title ⊕ text`; единый билдер для всех источников.
- `infrastructure/rag2/reranker.py:31` — `texts=[jira_rerank_text(d) for d in docs]` — одна точка текста и для cross-rerank (application), и для within-corpus rerank (`search.py:162`).
- `application/rag2.py:182-186` — под флагом `cross_reranker.rerank(...)` перезаписывает `index_hits`; RRF-порядок доступен ДО вызова (для б).

**Недобор полей (важно):** на `JiraDoc` НЕТ `generation`, `client`, и главное — `verdicts` (dict track→verdict). Есть только `verdict` — агрегированный tuple ЗНАЧЕНИЙ без привязки к треку. Значит «Покрытие по трекам» (главный сигнал, поднявший helm в RRF) без домапа не восстановить → домап в `search.py` обязателен, не опционален.

---

## (г) Source-aware обогащение helm-текста ⭐ основное

**1. `domain/rag2.py` — расширить `JiraDoc` (аддитивно, дефолты — обратная совместимость):**
```python
generation: str | None = None
client: tuple[str, ...] = ()
verdicts: tuple[tuple[str, str], ...] = ()   # (track, verdict) — hashable, frozen-safe
```
(tuple-of-pairs, а не dict — чтобы frozen-dataclass остался hashable.)

**2. `domain/rag2.py` — добавить `helm_rerank_text(doc)` рядом с `jira_rerank_text`** (чистая функция, домен — как существующий билдер):
```python
_VERDICT_RU = {"met": "покрыто", "partial": "частично", "absent": "нет", "na": "неприменимо"}

def helm_rerank_text(doc: JiraDoc) -> str:
    """helm-требование для кросс-энкодера: title+desc + управленческая сводка
    (поколение/компоненты/заказчики/покрытие по трекам) — тот же сигнал, что дал
    высокий RRF-ранг по embed-тексту. Без сводки → базовый текст."""
    base = jira_rerank_text(doc)
    lines = []
    if doc.generation: lines.append(f"Целевое поколение: {doc.generation}")
    if doc.component:  lines.append(f"Компоненты: {', '.join(doc.component)}")
    if doc.client:     lines.append(f"Заказчики: {', '.join(doc.client)}")
    if doc.verdicts:
        cov = "; ".join(f"{t.upper()} — {_VERDICT_RU.get(v, v)}"
                        for t, v in sorted(doc.verdicts))
        lines.append(f"Покрытие: {cov}")
    return f"{base}\n\n" + "\n".join(lines) if lines else base
```
Формат зеркалит `_requirement_context` → кросс-энкодер видит ровно тот текст, по которому считался вектор.

**3. `infrastructure/rag2/reranker.py:31` — branch по источнику:**
```python
texts = [helm_rerank_text(d) if d.source == "helm" else jira_rerank_text(d) for d in docs]
```
jira/confluence не трогаются (у них `payload["text"]` = полное тело) → +17% математически цел. Domain-RRF (`federate`) не трогаю.

**4. `infrastructure/rag2/search.py:_doc_from_payload` — домапить недостающие helm-поля:**
```python
generation=_opt("generation"),
client=_str_tuple(payload.get("client")),
verdicts=tuple(sorted((str(t), str(v)) for t, v in (payload.get("verdicts") or {}).items())),
```
Реингест НЕ нужен — поля уже в Qdrant payload (`helm_mgmt_index.py:176-199`).

## (б) Rank-floor (страховка, ортогонально, application-only)

**5. `domain/rag2.py`? — нет.** Чистый helper в `application/rag2.py` (без domain):
```python
def _apply_rank_floor(rrf_order, reranked, *, tolerance):
    """Не давать документу упасть более чем на `tolerance` позиций ниже его RRF-ранга.
    Составной ключ (source,key,text) — устойчив к chunk-дублям одного key."""
    pos = {}
    for i, d in enumerate(rrf_order):
        pos.setdefault((d.source, d.key, d.text), i)
    def eff(j, d):
        return min(j, pos.get((d.source, d.key, d.text), j) + tolerance)
    return [d for _, d in sorted(enumerate(reranked),
                                 key=lambda p: (eff(p[0], p[1]), p[0]))]
```
- `tolerance=0` → строгий floor (никаких падений ниже RRF); больше → свобода реранка. Стартовое значение подберу по A/B; целевой инвариант — **документ из RRF-топ-10 не выпадает из топ-10** (защита «лучшего ответа»).
- Стабильная пересортировка: при равном `eff` — порядок реранка (реранк судит перестановки внутри «этажа»).

**6. Включение floor — через `Rag2Policy`, opt-in (дефолт OFF):**
```python
rerank_rank_floor: int | None = None   # None → floor выключен
```
и в `answer()` (182-186), внутри `if self.cross_reranker`, ПЕРЕД rerank захватить `rrf_order = list(index_hits)`, после — если `policy.rerank_rank_floor is not None`, прогнать `_apply_rank_floor`.
- **Почему opt-in:** существующий `test_cross_reranker_reorders_merged_list` требует полного реверса. Дефолт-OFF в `Rag2Policy` → юнит-тесты с сырым `_POLICY` не ломаются. Floor включаю в tuned только через service-wiring (см. ниже).

**7. Wiring tuned-режима:** в `infrastructure/rag2/service.py` (где строится `Rag2Policy`/оркестратор под флагом `rag2_cross_rerank_enabled`) при включённом флаге проставить `rerank_rank_floor=<tuned>`. Прод-конфиг флага остаётся `False` → на бою ничего не меняется. (Точную точку сборки policy уточню при реализации — в плане пометка.)

---

## Тесты (расширю, не сломав существующие)
- `tests/infrastructure/test_reranker.py` — новый: helm-док с `generation/component/client/verdicts` → в `fake.docs` попадает текст со сводкой «Покрытие: …»; jira-док → без сводки (регресс-гард на асимметрию).
- `tests/infrastructure/test_rag2_search.py` — `_doc_from_payload` мапит `generation/client/verdicts` из payload.
- `tests/application/test_rag2_orchestrator.py` — новый: с `rerank_rank_floor=0` и `_ReverseReranker` RRF-топ-док не падает ниже floor; существующие 3 cross-rerank теста (дефолт floor OFF) остаются зелёными.
- `tests/domain/` — `helm_rerank_text`: сводка присутствует / пустые поля → базовый текст.

## A/B (эфемерно, `helm-helm-1`, прод не трогаю) — ⚠️ нужен ответ тимлида
Драйвер `helm.eval.rag2_eval` поверх `build_rag2_eval_wiring`, набор B (16 запросов), k=10. Три прогона:
- **off** — main-код, флаг снят.
- **on** — main-код, `-e HELM_RAG2_CROSS_RERANK_ENABLED=1` (воспроизвести регресс Q2/Q6/Q7/Q8).
- **tuned** — МОЙ код + флаг on.

**Операционный нюанс (отличие от verifier):** verifier только тоглил env на деплой-коде. Для **tuned** нужен МОЙ патч В контейнере. Предлагаю: ephemeral `docker cp` 3-4 патченых файла в `helm-helm-1:/app/src` (через ssh-manager) → прогон eval с флагом → откат (`docker cp` оригиналов обратно / restart контейнера). Прод-конфиг и флаг на бою НЕ трогаю, всё эфемерно и обратимо. **Подтверди, что такой ephemeral-оверлей кода приемлем** (иначе tuned A/B доказать нельзя — env-тоглом код не подменить).

Что доказываю: (1) регресс ушёл — helm `1.0-xxx` на Q2/Q6/Q7/Q8 обратно в топ-10; (2) nDCG@10 ≈0.71 (≥ off 0.608), top-1 ≈11/16 не деградирует. Плюс юнит-тесты затронутых модулей зелёные.

## Границы / безопасность
- Правлю ТОЛЬКО worktree. Прод не трогаю, флаг в конфиге остаётся off. Не мержу/не пушу.
- ⚠️ INJECTION: в прочитанном коде/данных инъекций не встретил.

## Файлы под правку (итог)
1. `domain/rag2.py` — +3 поля JiraDoc, +`helm_rerank_text` (+`_VERDICT_RU`).
2. `infrastructure/rag2/reranker.py` — branch по source (стр.31).
3. `infrastructure/rag2/search.py` — домап `generation/client/verdicts`.
4. `application/rag2.py` — `_apply_rank_floor` + захват RRF-порядка + policy-поле.
5. `infrastructure/rag2/service.py` — wiring floor в tuned (точка уточняется).
6. Тесты (4 файла выше).

**Открытые вопросы тимлиду:**
1. Ephemeral `docker cp`-оверлей кода для tuned A/B — ок? (см. нюанс выше)
2. Стартовый `tolerance` rank-floor — оставляешь на мой подбор по A/B (предлагаю инвариант «RRF-топ-10 не выпадает из топ-10»)?