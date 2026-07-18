---
title: Реранкер в LLM Gateway — сделано (DeepInfra Qwen3-Reranker-4B)
type: note
permalink: tacticum/04-sessions/reranker-v-llm-gateway-sdelano-deep-infra-qwen3-reranker-4-b
tags:
- rerank
- gateway
- litellm
- deepinfra
- rag
- adr-0005
- done
---

## Что сделано (2026-07-15)
Задача 1А — реранкер на LLM Gateway (`llm.cifragen.ru`, хост TEI/Gateway 155.212.134.20, стек `/opt/cifragen`). Доступ к серверам получен, заведены в ssh-manager: `gateway` (155.212.134.20, root/пароль) и `adp_emb` (194.36.208.242, root/пароль).

**Развёрнуто:** тир `tacticum/rerank` в litellm (`/opt/cifragen/litellm/config.yml`), путём **как эмбеддинги — через DeepInfra** (решение руководителя: контур уже «наружу», т.к. `tacticum/embed` bge-m3 и так проксируется в DeepInfra).
- Модель: **`Qwen/Qwen3-Reranker-4B`** — т.к. **DeepInfra больше НЕ отдаёт `bge-reranker-v2-m3`** (в каталоге только Qwen3-Reranker 0.6B/4B/8B + nvidia). Качество: та же модель локально/облако = идентично.
- Правки: `config.yml` (+ модель), `docker-compose.yml` (прокинут `DEEPINFRA_API_KEY` в litellm). Бэкапы `*.bak-preRerank`. Рестарт litellm — healthy, `embed`/`chat` не задеты.
- Тест: боевой `/v1/rerank` ранжирует корректно (релевант 0.9999).

**Доступ ключей:** allowlist ключей рулит **project-hub (US #66)**. Временно выдал `tacticum/rerank` ключу helm (`key_alias=control-tower`) через litellm `/key/update` (модели+tier_budgets сохранены) → helm получает 200. **Это времянка** — project-hub может затереть при пересинке.

## Осталось
1. **project-hub (руководитель):** добавить `tacticum/rerank` в набор моделей RAG-групп (control-tower/re/agents/arch-mcp + knowledge_rag), как `tacticum/embed`. Не гейтед. Тогда времянка станет постоянной. Провижинер локально не найден — нужен от руководителя (репо/хост).
2. **Включить в helm:** `HELM_DOCS_RERANK_ENABLED=true` (+ `assistant_rerank_enabled`, `rag2_rerank_enabled` по месту) + замер eval. Helm уже написан под `tacticum/rerank` (Cohere `/v1/rerank`), менять код не надо.
3. **⚠️ Примирить ADR-0005:** он предписывает реранк **self-hosted TEI в tei_service** и **отклоняет облачный rerank** (суверенитет/RU). Мы — через DeepInfra. Тот же дрейф, что и эмбеддинги. Нужно обновить/заместить ADR (решение руководителя).

## Дальше по плану
Задача 2 — выкачка данных Jira/Confluence через adp_emb (доступ есть). См. [[Доступ к двум ВПС- реранкер (gateway) + полная выгрузка данных (ИВА-adp_emb)]], модель генерации [[Gemini→Gemma (triva) — локальная генерация в контуре ИВА]].
