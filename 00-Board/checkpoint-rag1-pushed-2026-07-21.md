---
title: checkpoint-rag1-pushed-2026-07-21
type: note
permalink: tacticum/00-board/checkpoint-rag1-pushed-2026-07-21-1
status: current
role: rag1-lead
project: helm / RAG#1
tags:
- rag1
- checkpoint
- pushed
- awaiting-merge
- blocker
---

# RAG#1 3 проблемы — запушено, ждёт мержа/деплоя (2026-07-21)

Ветки запушены (OK пользователя получен). Полная сводка: [[summary-rag1-quality-3issues-ready]]. Baseline: [[baseline-rag1-golden184-judge-2026-07-21]].

## Запушено (новые ветки на origin, PR создаёт пользователь)
1. **feat/rag1-correctness-clarify** — калибровка clarify (темпоральная+продуктовая двусмысленность) + уборка мёртвого score-кода. 910 passed, гейт PASS. Флаг docs_clarify_enabled=False.
   PR: https://github.com/TacticumApps/helm/pull/new/feat/rag1-correctness-clarify
2. **feat/rag1-conversation-context** — стор docs_conversation_turn + инжект истории, миграция f4e5d6c7b8a9. 111 passed, гейт PASS. Флаг HELM_DOCS_CONVERSATION_CONTEXT_ENABLED=False.
   PR: https://github.com/TacticumApps/helm/pull/new/feat/rag1-conversation-context

Порядок мержа: corrclar → convctx (конфликта нет, merge-tree exit 0). Оба инертны в проде (флаги OFF).

## ⚠️ БЛОКЕР инфры (обнаружен 2026-07-21 ~13:15)
`iva_llm` (vLLM генерации ответов) ходит через ssh-туннель `172.18.0.1:8790 → 10.0.196.12:8004` (jump adp-jump). Сокет открывается, но HTTP до vLLM падает `APIConnectionError`. **Сломалось в последние ~15 мин** (judge-прогон в 12:18-13:01 через тот же iva_llm работал — 184 ответа). **Задевает живую генерацию бота.** Похоже удалённый vLLM/форвард туннеля отвалился — нужен рестарт со стороны ops. Не наш код.

## Осталось (по восстановлении LLM + OK)
1. **Pre-deploy clarify-верификация** (доказательство прироста проблемы 2): probe `verify_clarify.py` готов (на проде /app), гоняет откалиброванную инструкцию через живой LLM на clarify_cases (5 двусмысл. expect clarify / 5 однозначных expect answer). **Заблокирован недоступным iva_llm.** Прогнать по восстановлении.
2. Деплой (после мержа PR): ff corrclar→main, затем convctx→main → `SEED=0 bash scripts/deploy.sh` → `alembic upgrade heads` → verify. Флаги OFF.
3. Post-deploy: включить clarify на тесте, подтвердить «по делу, не ложно» → оставить ON.

## Прод-артефакты (temp, сотрутся при rebuild)
/app: golden_ambiguous.json, golden_eval200.json, eval184_baseline.json, verify_clarify.py, clarify_cases.json, clarify_instruction.txt, ab_synonyms.py, probe_rank.py.

Follow-up: recall-miss one-ios-ug-calls; конфиг-цепочка clarify_tau_answer.