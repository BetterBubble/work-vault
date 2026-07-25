---
title: deploy-rag1-quality-done-2026-07-21
type: note
permalink: tacticum/01-sessions/deploy-rag1-quality-done-2026-07-21-1
status: current
role: rag1-lead
project: helm / RAG#1
tags:
- rag1
- deploy
- done
- verify
- blocker
- iva-llm
---

# RAG#1 3 проблемы — ЗАДЕПЛОЕНО (2026-07-21), фул-тесты ждут iva_llm

Пользователь смержил (PR #83 corrclar, #84 convctx) и дал OK на деплой. Выкатка проведена по runbook.

## Деплой — УСПЕХ
- Бэкап БД до миграции: `/root/backup_helm_pre_convctx_2026-07-21.sql` (84 МБ).
- `/opt/helm` ff `ea295f8 → 739f611` под tacticum-deploy (дрейф не помешал).
- `SEED=0 bash scripts/deploy.sh` → `DEPLOY_EXIT_0`: образ пересобран, helm-helm-1 пересоздан, postgres healthy, миграция `e3d4c5b6a7f8 → f4e5d6c7b8a9 docs_conversation_turn` накатана.

## Verify (не-LLM) — ЗЕЛЁНО
- helm-helm-1 Up, alembic current = **f4e5d6c7b8a9 (head)**.
- Флаги **OFF**: docs_clarify_enabled=False, docs_conversation_context_enabled=False (turns=3), max_tokens=700 → **выкатка инертна** (поведение прода не изменилось).
- Таблица `docs_conversation_turn` создана.
- Retrieval smoke OK (5 чанков, топ desktop-ug-calls) — embed+Qdrant+Meili+rerank живые.

## ⚠️ БЛОКЕР сохраняется: iva_llm недоступен
vLLM генерации (`172.18.0.1:8790` → ssh-туннель → `10.0.196.12:8004`) даёт APIConnectionError. Сокет открыт, HTTP падает → удалённый vLLM/форвард туннеля лёг (сломалось ~13:15, judge-прогон 12:18-13:01 работал). **Задевает живую генерацию бота.** Нужен рестарт туннеля/удалённого vLLM со стороны ops (host-процесс `ssh -L`, pid ~75397, НЕ в compose — деплой его не чинит).

## Фул-тесты — АРМИРОВАНЫ (ждут LLM)
Оркестратор `/root/run_fulltests_when_ready.sh` (detached, pid 419195): health-poll iva_llm до 90 мин, при восстановлении сам гоняет:
1. golden-184 judge «после» → `/app/eval184_after.json` (сверка с baseline correctness 0.768 / faithfulness 0.982 — должно совпасть, дифф инертен).
2. `verify_clarify.py` — доказательство clarify (переспрос по делу, не ложно) на clarify_cases.
Результат → `/root/fulltest.log`, маркер `FULLTEST_EXIT_0/2`.

## Post-deploy (после зелёных фул-тестов + OK)
Включить clarify на тесте (`docs_clarify_enabled=1`), подтвердить «по делу» → оставить ON. Тогда прирост проблем 1+2 станет фактом. Аналогично conversation-context.

Baseline: [[baseline-rag1-golden184-judge-2026-07-21]]. Сводка: [[summary-rag1-quality-3issues-ready]].