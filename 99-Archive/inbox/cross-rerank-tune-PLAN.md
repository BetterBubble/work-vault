---
title: Cross-rerank RAG#2 — тюнинг регресса на helm-корпусе (PLAN)
type: handoff
author: explorer (team helm)
date: 2026-07-16
scope: READ-ONLY разведка + дизайн. Реализацию делает implementer позже.
permalink: tacticum/00-inbox/cross-rerank-tune-plan
status: archived
updated: 2026-07-18
---

# Cross-rerank RAG#2 — почему опускает helm и как чинить

Разведка по коду в контейнере `helm-helm-1:/app/src/helm` (main, PR#53). Всё подтверждено чтением кода, не гипотезой.

## TL;DR / рекомендация
**Корневая причина — асимметрия текста: helm-хиты эмбеддятся по обогащённому тексту (тело + управленческая сводка покрытия), а кросс-энкодер судит их по ОБЕДНЁННОМУ тексту (только title+description, БЕЗ сводки).** Тот самый сигнал, что поднял helm в топ по RRF, не виден реранкеру → он честно ставит низкий score → роняет.

Рекомендованный подход: **(г) обогатить text helm-хита перед реранком** — собрать source-aware текст для helm в адаптере реранка, переиспользовав поля, которые кросс-энкодеру сейчас не показывают. Чинит причину, не трогает domain-RRF, не влияет на путь jira/confluence (→ +17% сохраняется). Как дешёвая страховка первым шагом допустима **(б) rank-floor**.

---

## 1. Диагноз (с точками кода)

Поток реранка:
- `application/rag2.py:178` — `index_hits = federate(by_source, limit=pool)` — RRF-слияние, каждый `JiraDoc.score` = RRF fused_score, порядок списка = RRF-ранг.
- `application/rag2.py:182-186` — под флагом `cross_reranker.rerank(query, index_hits, top_n=len(index_hits))` **перезаписывает** `index_hits` (и `.score`) выдачей кросс-энкодера. Здесь же, ДО вызова, ещё доступен исходный RRF-порядок/score (важно для б/а).
- `infrastructure/rag2/reranker.py:31` — `texts = [jira_rerank_text(d) for d in docs]` — ЕДИНАЯ функция текста для всех источников (jira/confluence/helm).
- `domain/rag2.py:114-119` `jira_rerank_text(doc)` — возвращает `title ⊕ doc.text` (а при `title in body` — просто `doc.text`).

**Что реально лежит в `doc.text` у helm** (ключевой факт) — `ingest/helm_mgmt_index.py:201-205`:
```python
for idx, piece in enumerate(pieces):
    # context (Компоненты/Заказчики/Покрытие: KMP—покрыто; ...) — только в EMBED-текст
    embed_text = f"{piece}\n\n{context}" if (idx == 0 and context) else piece
    payload = {**base_payload, "text": piece, "chunk_idx": idx}
```
- `piece` = чанк из `head = title + description` (`:173-174`).
- `context` = человекочитаемая управленческая сводка (целевое поколение, компоненты, заказчики, **Покрытие по трекам** — `_requirement_context` `:140-155`).
- **`embed_text` (что видит эмбеддер/вектор) = piece + context.** **`payload["text"]` (что потом видит реранкер) = piece БЕЗ context.**

Мэппинг payload→JiraDoc: `infrastructure/rag2/search.py:73` `_doc_from_payload` — `text = payload["text"]` (= piece, без сводки). Значит `jira_rerank_text` для helm отдаёт кросс-энкодеру только title+description требования.

**Механизм регресса:** вектор helm-требования посчитан по богатому тексту (piece+сводка покрытия) → требование хорошо матчит семантический запрос → высокий RRF-ранг → входит в топ. Но кросс-энкодер оценивает его по короткому title+desc (управленческая формулировка вида «1.0-xxx», без сводки, которая и дала матч) → недооценивает релевантность → опускает, иногда ниже топ-10 (роняет единственный лучший ответ). У jira/confluence такой асимметрии нет: их `payload["text"]` = полное тело (`ingest.py:180-193` — summary+desc+комментарии+история+связи), т.е. кросс-энкодер видит тот же богатый текст → судит честно. Отсюда систематический перекос против helm.

**Диагноз одной фразой:** helm попадает в топ по обогащённому embed-тексту (тело+сводка покрытия), а кросс-энкодер ранжирует его по обеднённому `payload["text"]` (title+desc без сводки) — `helm_mgmt_index.py:201-205` пишет сводку только в embed_text; `reranker.py:31`+`domain/rag2.py:114` кормят кросс-энкодер полем `text` → helm недооценён и опущен.

---

## 2. Таблица подходов

| # | Суть | Точки правки | Слой | Pro | Con / риск |
|---|------|--------------|------|-----|-----------|
| **(г) обогатить helm-текст** ⭐ | Для helm-хита строить текст реранка = title+desc **+ управленческая сводка** (покрытие/компоненты/трек/тип), чтобы кросс-энкодер судил по тому же сигналу, что дал матч | source-aware билдер текста в `infrastructure/rag2/reranker.py` (заменить строку 31 на branch по `d.source`); поля `verdict/track/entity_type/component/status` УЖЕ на JiraDoc (`search.py:73`); опц. домапить `generation/client/verdicts` из payload | infra (+опц. search) | Чинит КОРНЕВУЮ причину; jira/confluence не трогает → +17% цел; НЕ трогает domain-RRF; реингест НЕ нужен (поля уже в payload) | Меняет score всех helm-хитов (не только 4 регресс) — но в честную сторону, риск низкий; формат обогащения надо подобрать (не раздуть/не зашумить); часть полей (`client/verdicts-dict/generation`) сейчас не на JiraDoc — нужен маленький домап в `search.py` |
| **(б) rank-floor** | Не давать документу упасть ниже исходного RRF-ранга (clamp): после реранка финальный ранг = учёт исходной RRF-позиции, стабильная пересортировка | `application/rag2.py:182-186` — захватить RRF-порядок ДО rerank, после — clamp | application | Минимально-инвазивно, чисто application, без domain; прямо защищает «лучший ответ» от провала; generic (любой источник) | Пластырь: не убирает саму недооценку helm, только катастрофические падения; ослабляет часть законных перестановок кросс-энкодера → может слегка подрезать выигрыш; нужен подбор «порога» |
| **(а) source-aware пул** | helm НЕ мешать в кросс-энкодерный пул: реранкать только jira+confluence, helm домёржить по его RRF-позиции | `application/rag2.py:182-191` — сплит helm/не-helm до rerank, домёрж после | application | Полностью снимает регресс на helm; jira/confluence реранк не трогается → +17% цел; без domain | helm НИКОГДА не выигрывает от кросс-энкодера (если реранк корректно поднял бы helm — потеряем); нужно проверить, нет ли среди 11/16 top-1 выигрышей поднятого helm-дока (если есть — риск потери) |
| **(в) бонус score по источнику** | Прибавлять helm фиксированный/относительный бонус к score после реранка | `application/rag2.py` post-process или `reranker.py` | app/infra | Просто | Произвольный вес, хрупко, может пере-поднять нерелевантный helm; калибровка на глаз; риск средний — не рекомендую |

⭐ = рекомендованный.

---

## 3. Рекомендация + почему

**(г) обогащение helm-текста перед реранком, реализованное в `infrastructure/rag2/reranker.py` (source-aware билдер, без правки domain).**

Почему лучший баланс «убрать регресс, сохранить +17%»:
- Атакует **причину**, а не симптом: кросс-энкодер начинает видеть тот же управленческий сигнал (покрытие/компоненты/трек), что дал helm высокий recall → систематическая недооценка снимается для ВСЕХ helm-хитов, а не только 4 сломанных.
- **Не трогает** путь jira/confluence и domain-RRF (`federate`) → выигрыш на 12/16 остаётся математически нетронутым.
- **Реингест не нужен**: сводка-поля уже лежат в Qdrant payload (`helm_mgmt_index.py:176-199`: `verdict/verdicts/component/client/generation/track/entity_type/status`), просто сейчас не доезжают до реранка. Минимальный домап в `search.py` для 2-3 недостающих полей + branch в билдере текста.

Минимальный вариант (совсем без `search.py`): собрать обогащение из уже-мэпнутых полей JiraDoc (`entity_type/verdict/track/component/status`) — этого достаточно, чтобы поднять score; домап `generation/client/verdicts-dict` — как follow-up при недоборе.

Если хочется страховку первым коммитом — добавить **(б) rank-floor** как дешёвый предохранитель от провала лучшего ответа (application-only), затем (г) убирает саму недооценку. (г) и (б) ортогональны и совместимы.

Не рекомендую (в) — хрупкая калибровка. (а) рабочая, но лишает helm потенциального выигрыша от кросс-энкодера и требует проверки, не было ли среди выигрышей поднятого helm.

---

## 4. Как проверить после правки (A/B, набор B)

Драйвер уже есть — CLI `helm.eval.rag2_eval` поверх `build_rag2_eval_wiring` (`eval/rag2_adapters.py:77`) → `Rag2Orchestrator.answer(q, k=10)`.
- Запуск в `helm-helm-1` (Gateway/Qdrant достижимы). Флаг — env: `-e HELM_RAG2_CROSS_RERANK_ENABLED=1` (за него читается `rag2_cross_rerank_enabled`, `service.py:100`).
- Golden/набор B: `--golden <путь к 16-запросному набору B>` (профильный дефолт `tests/data/rag2_golden_profile.json`, `rag2_eval.py:53`); k=10.
- Verifier гонял эфемерно, сырые артефакты `out_off.json` / `out_on.json` — в scratchpad его сессии (сравнить с ними).

Что мерить (as PR#53):
1. **Регресс ушёл:** на Q2/Q6/Q7/Q8 helm-корпус НЕ опускается — лучший helm-ответ (ключ `1.0-xxx`) снова в топ-10 (в идеале top-1, где так было в off).
2. **Выигрыш цел:** общий nDCG@10 ≈ 0.71 (не ниже off=0.608, желательно держать ~+17%); top-1 ≈ 11/16 не деградирует.
3. Три прогона: `off` (флаг снят) · `on` (текущий, воспроизвести регресс) · `tuned` (правка) — сверить попарно per-query, что 4 helm-регресса исчезли, а 12 не-helm выигрышей на месте.

---

## ⚠️ INJECTION
Не обнаружено. В прочитанном коде/докстрингах инъекций «системных правил» нет.

---

## Приложение: карта файлов (сервер helm, `/app/src/helm/`)
- `application/rag2.py` — `Rag2Orchestrator.answer`; :178 federate; :182-186 применение cross_reranker (перезапись index_hits); :136 поле `cross_reranker: DocsReranker | None`.
- `domain/rag2.py` — :114 `jira_rerank_text` (title⊕text); :62 `JiraDoc` (поля helm: `entity_type/verdict/track`); :~330 `federate` (RRF, `FEDERATE_RRF_K=60`, `.score`=fused).
- `infrastructure/rag2/reranker.py:31` — `JiraReranker.rerank`, `texts = [jira_rerank_text(d) ...]` (единая точка текста для всех источников) ← точка правки (г).
- `infrastructure/rag2/search.py:73` — `_doc_from_payload` (`text = payload["text"]`, маппинг helm-полей).
- `infrastructure/rag2/mgmt_vectorstore.py` — Qdrant helm-корпуса (`with_payload=True`, весь payload возвращается).
- `ingest/helm_mgmt_index.py:158-205` — сборка helm-документа; :204 `embed_text=piece+context`, :205 `payload["text"]=piece` ← корень асимметрии.
- `infrastructure/rag2/service.py:100,109-111,170-181` — флаги/сборка reranker.
- `eval/rag2_eval.py`, `eval/rag2_adapters.py:77`, `eval/rag2_runner.py` — драйвер A/B.