---
title: session-state
type: note
permalink: tacticum/04-sessions/session-state
status: current
updated: 2026-07-18
tags:
- session
- checkpoint
- helm
- rag2
---

# session-state — текущий чекпойнт

Обновлён **2026-07-18** (после ночной работы). Направление — **RAG#2 / RAG#1 для ИВА на helm**. Прод `helm.tacticum.ru`, `main = c036842`, сервис жив.

> Вся хроника доработки RAG#2 (16→18.07, промежуточные апдейты + ночная работа) вынесена в архив: [[session-state-chronicle-2026-07-17-18]]. Здесь — только текущее состояние.

## Где стоим (прод, проверено на реальных данных)
- helm `main = c036842` задеплоен; unit-тесты 1619 passed / 30 skipped (аудит подтвердил — не обманки).
- ТЗ RAG#2-доработка: **Р-1 / Р-2 (floor×pin) / Р-4-JUMP / Р-5 (effort_hint) / дотяжка / blockers — 🟢 зелёные вживую.**
- Метрики: RAG#2 recall@10 0.973, key_lookup 1.00, MRR 0.697; RAG#1 recall@5 0.949 (hybrid+rerank — прод-оптимум).
- Бэкап epic_task: `/opt/helm/data/backup_epic_task_2026-07-17.sql`.

## Следующий шаг / ждёт пользователя
1. **IVAONEHALF** (577 задач, corpus-gap ИВА-1.5): нужны Jira-креды → `bash /root/ivaonehalf_prep/run.sh` (пайплайн готов, паритет 1-в-1 доказан).
2. **Eva-wiki** (Р-4, 2-я часть): orionpro session-креды + HTML-парсер вики.
3. **Р-3 code-term**: единственный неизмеренный пункт ТЗ — можно замерить.

## НЕ включать — доказано замером на реальных данных (не возвращаться к этому)
- **cross-rerank** и **hybrid ре-баланс ft_weight** — recall не дают.
- **F2 near_dup_dedup** — вредит RAG#1 (0.930→0.877).
- **F3 query_rewrite** — вредит на полном golden 1306 (−0.019; ночной +0.01 был артефакт узкой выборки).
- **floor τ=0.7** — маргинально (+1 OOD из 40); прод-τ оставлен **0.5**.

## Ключевые операционные факты
- Сервер **helm**: ssh-manager алиас `helm`. `/opt/helm` — git-репо, `pull` только под `tacticum-deploy` (deploy-key), не root. Деплой = bundle → ff-merge → `SEED=0 bash scripts/deploy.sh` (rebuild; **volume-mount ненадёжен — только rebuild**) → verify `getsource`. Env `/opt/helm/.env`.
- **Git:** push только feature-ветку явно; PR не создаём (материалы → пользователь мержит); **без AI-подписей**; не в main.
- **Команда:** делегировать; plan-approval; общение SendMessage + заметки.

## Ключевые заметки
[[rag2-ab-measurements]] · [[night-work-summary]] · [[git-deploy-hygiene-helm]] · [[rag2-r5-rebuild-and-datamap]] · [[rag2-ivaonehalf-gap]]
