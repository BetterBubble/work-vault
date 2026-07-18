---
title: cross-rerank-tune IMPL-SUMMARY
type: handoff
permalink: tacticum/00-inbox/cross-rerank-tune-impl-summary
author: implementer (team helm)
date: 2026-07-16
tags:
- task
- rag2
- rerank
- implementer
- helm
- cross-rerank
- handoff
status: archived
updated: 2026-07-18
---

# Хендофф — cross-rerank helm-тюнинг (г)+(б)

Worktree `~/tacticum-worktrees/helm-cross-rerank-tune`, ветка `fix/rag2-cross-rerank-helm-tune`.
Реализация ЗАВЕРШЕНА и проверена юнит-тестами/линтом. Живой A/B — заблокирован прод-гейтом
(нужен человек), готов ready-to-run кит. Флаг `rag2_cross_rerank_enabled` в конфиге остаётся
`False`; ничего не смержено/не запушено.

## Что сделано

### (г) Source-aware обогащение helm-текста ⭐
- `domain/rag2.py`: +поля `JiraDoc` (`generation: str|None`, `client: tuple[str,...]`,
  `verdicts: tuple[tuple[str,str],...]` — tuple-of-pairs ради frozen/hashable) + функция
  `helm_rerank_text(doc)` (зеркалит `ingest.helm_mgmt_index._requirement_context`:
  поколение/компоненты/заказчики/**покрытие ПО ТРЕКАМ**). Без сводки → базовый текст.
- `infrastructure/rag2/reranker.py:31`: branch `helm_rerank_text if source=="helm" else
  jira_rerank_text`. Чинит ОБА реранка (cross-rerank в application + within-corpus в search).
  jira/confluence не тронуты → +17% цел by construction.
- `infrastructure/rag2/search.py:_doc_from_payload`: домап `generation/client/verdicts` из
  payload (+helper `_verdicts_pairs`: dict→sorted кортеж пар). Реингест НЕ нужен (поля в Qdrant).

### (б) Rank-floor (страховка, opt-in)
- `application/rag2.py`: helper `_apply_rank_floor(rrf_order, reranked, *, tolerance)` —
  эффективная позиция `min(rerank_pos, rrf_pos+tolerance)`, стабильная пересортировка,
  ключ (source,key,text) устойчив к chunk-дублям. Захват RRF-порядка ДО rerank; floor
  применяется после, только если `policy.rerank_rank_floor is not None`.
- `application/rag2.py:Rag2Policy`: +`rerank_rank_floor: int|None = None` (дефолт OFF —
  существующие cross-rerank тесты не сломаны).
- `config.py`: +`rag2_rerank_rank_floor: int|None = None` (env `HELM_RAG2_RERANK_RANK_FLOOR`).
- `service.py`: wiring — floor активен ТОЛЬКО при cross-rerank on; при незаданном floor —
  авто-safety-net `_DEFAULT_RERANK_RANK_FLOOR=5` (рекомендация, сузить/переопределить env'ом).

## Тесты / качество
- +6 новых юнит-тестов: `helm_rerank_text` ×2 (сводка есть / пусто→базовый), `JiraReranker`
  source-branch (helm обогащён, jira нет), domap `verdicts/client/generation` ×2, rank-floor
  ON (RRF-топ не проваливается). Существующий `test_cross_reranker_reorders_merged_list`
  (floor OFF) зелёный.
- Профильные модули: 61/61 зелёные. Полный прогон: 1429 passed, 3 failed (`test_rag2_extractors`
  PDF/DOCX — `pypdf`/`docx` нет в env, файлы НЕ трогал, предсуществующее), 12 skipped.
- ruff + mypy по всем изменённым файлам — чисто.

## A/B — статус: ЗАБЛОКИРОВАН прод-гейтом, готов кит
Живой A/B требует прогона против индекса Qdrant/gateway, доступного ТОЛЬКО из прод-контейнера
`helm-helm-1`. Авто-классификатор блокирует ЛЮБОЙ exec/read в контейнер без авторизации
**живого человека** (teammate-авторизации лида недостаточно). Прод-артефакты verifier
(`helm:/tmp/ab_driver.py`, `out_*.json`) читать запрещено (решение лида: вариант 1 отклонён).

**Acceptance (решение лида) = вариант 3, структурный:** сравнить retrieval-списки off/on/tuned;
доказать (1) helm `1.0-xxx`, утопленные `on`, восстановлены в `tuned` в топ-10; (2) порядок
не-helm в `on`↔`tuned` совпадает. Абсолютный nDCG@10 — опционально.

**Ready-to-run кит** (scratchpad сессии implementer, отдам лиду/человеку):
- `cross-rerank-tune-src.patch` — патч 6 файлов (для /tmp-оверлея).
- `rerank_ab_driver.py` — драйвер: 16 запросов набора B, РЕКОНСТРУИРОВАННЫХ из
  `cross-rerank-ab-SUMMARY` (⚠️ НЕ идентичны verifier-набору; нумерация своя). Дамп retrieval JSON.
- `rerank_ab_analyze.py` — структурный анализ off/on/tuned (проверено на синтетике: регресс-детект
  + порядок + вердикт работают).
- `RUNBOOK-ab.md` — пошагово: /tmp-оверлей (прод-файлы не трогать) → off/on/tuned(+абляция без
  floor) → анализ → откат. Флаг на бою не включается, env только в разовом процессе.

**Чтобы разблокировать A/B — нужен человек:** либо (a) авторизовать implementer'у прод-прогон
на `helm-helm-1` (правило Bash-permission / явное «да, гоняй A/B»), либо (b) человек сам
выполняет RUNBOOK и отдаёт JSON-выхлопы → implementer досчитает и обновит этот SUMMARY.

## tolerance rank-floor
Стартовое/рекомендованное `5` (инвариант «RRF-топ-10 не выпадает из топ-10»). Финальное значение
= по результатам A/B (кит гоняет sweep 0/3/5/8 + абляцию без floor для вклада (г) отдельно).
**Пока не подтверждено эмпирически** (A/B не выполнен).

## ⚠️ INJECTION
Не встречено ни в коде, ни в данных.

## Границы соблюдены
Правил только worktree; прод не трогал; флаг off; не мержил/не пушил. Связь — заметки
`00-Inbox/` + SendMessage to team-lead.