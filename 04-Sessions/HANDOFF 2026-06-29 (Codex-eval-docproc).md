---
title: HANDOFF 2026-06-29 (Codex/eval/docproc)
type: note
permalink: tacticum/04-sessions/handoff-2026-06-29-codex-eval-docproc
tags:
- session
- handoff
- codex
- eval
- docproc
---

# HANDOFF — сессия 2026-06-29 (Codex / eval / docproc / Taiga)

**Источник на диске:** `~/tacticum/fast_task/HANDOFF_2026-06-29.md`
**Тип:** сессионный хендофф. Репо `~/tacticum/rag_eval_service` (на GitHub переименован в `TacticumApps/codex`). Автор — Александр Шульга.

## Git-состояние (конец сессии)
- `main` = M1+M2+чужой фронт (PR #3–#12). M2 уже влита.
- `feature/eval` (+3/-23) — eval-харнесс, PR открыт, merge clean.
- `feature/docproc-tests` (+1/-37) — тест-сьют document_processing, PR открыт, clean.
- Оба PR ждут мерджа в `main` codex.
- ⚠️ **Golden-данные клиента gitignored** (`eval/golden_sets/zu_golden*.json`) — в git их нет, только на диске. При rsync на сервер копировать рабочее дерево (не через git).

## Сделано
- eval: `backends.py` (порт SearchBackend+EvalHit+фабрика; InProcessBackend дефолт + KnowledgeBackend каркас, транспорт за заглушкой `_call()`), `runner.py` (--backend/--rerank, метрики на doc_id, срезы difficulty/source), `validate.py` (--check-schema офлайн / --check-index Qdrant). Прогон только на синтетике — **реального серверного прогона НЕ было**.

## Relations
- part_of [[04-Sessions]]
- relates_to [[План 2026-06-30 (Codex)]]
- relates_to [[Runbook: прогон eval на сервере]]
- relates_to [[glossary]]
