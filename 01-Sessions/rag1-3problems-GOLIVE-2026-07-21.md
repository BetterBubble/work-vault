---
title: rag1-3problems-GOLIVE-2026-07-21
type: note
permalink: tacticum/01-sessions/rag1-3problems-golive-2026-07-21
status: current
role: rag1-lead
project: helm / RAG#1
tags:
- rag1
- golive
- done
- clarify
- conversation-context
- prod
---

# RAG#1 — 3 проблемы ЗАКРЫТЫ И ЖИВЫ в проде (2026-07-21)

Прод `helm.tacticum.ru`, `main = df6e2fc`. Флаги clarify + conversation-context ВКЛючены, всё проверено на реальном LLM.

## Что задеплоено и включено
- Стек ph1-ph5 (ранее) + calibrated-v1 clarify + conversation-context (PR #83/#84) + **clarify v2** (PR #85, `df6e2fc`).
- `.env`: `HELM_DOCS_CLARIFY_ENABLED=true` + `HELM_DOCS_CONVERSATION_CONTEXT_ENABLED=true` (turns=3). Бэкапы: `/root/backup_helm_pre_calibv2_2026-07-21.sql`, `/root/helm.env.bak-calibv2`. Откат флагов — мгновенный через env.

## Финальный смоук на проде (задеплоенный код, флаги ON)
- **Clarify v2: 17/18** — флагманский «идущий ВКС» ПЕРЕСПРАШИВАЕТ ✅, однозначные 11/11, **0 ложных** ✅. 1 пограничный промах «добавить в звонок» (benign — отвечает корректно).
- **Обычный ответ цел:** однозначный запрос → нормальный ответ, без лишнего clarify.
- **Conversation-context:** follow-up «А на Android как?» с историей понят (тема подхвачена), без истории — терялся.
- Health: контейнер Up, postgres healthy, alembic head, живой docs_ask отвечает.

## Итог по 3 проблемам
1. **Корректность** — retrieval-фикс измеренный тупик; закрыта поведенчески: двусмысленный кейс (=баг Лебедева) теперь переспрашивает. golden-184 correctness 0.778 (было 0.768) — регресса нет.
2. **Clarify** — v2 калибровка, живо, 17/18.
3. **Контекст чата** — conversation-context живо, follow-up понимается.

## Follow-up (не блокеры)
- clarify: 1 пограничный кейс «добавить в звонок» (не критично).
- conversation-context: ретрив для переформулированного multi-turn запроса можно улучшить (ответ «нет на Android», хотя страница есть) — тема понята, но ретрив недотягивает.
- **Архитектура (ждёт руководителя):** helm→Gateway через Project Hub (грант `tacticum/triva`) — сейчас helm ходит своим ssh-туннелем (работает); переключение = чистка, не блокер. Сообщение руководителю подготовлено.
- vLLM triva на 10.0.196.12:8004 восстановлен ~15:29.

Связи: [[rag1-3problems-full-validation-2026-07-21]] · [[fulltest-rag1-after-clarify-verify-2026-07-21]] · [[deploy-rag1-quality-done-2026-07-21]]