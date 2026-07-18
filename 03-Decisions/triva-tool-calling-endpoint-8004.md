---
title: triva tool-calling эндпоинт — 10.0.196.12:8004 (gateway переведён)
type: decision
permalink: tacticum/03-decisions/triva-tool-calling-endpoint-8004
date: 2026-07-16
status: applied (gateway), pending-eval (helm)
tags:
- triva
- tool-calling
- litellm
- gateway
- vllm
- iva
- decision
---

# Решение: triva для tool-calling ходит на 10.0.196.12:8004

## Контекст
tool-calling через `tacticum/triva` не работал: `tool_calls` пустой, вызов уезжал текстом в `content`.
Разобрано с нашей стороны (туннель+litellm чисты) → причина на апстриме. См. [[triva-proxy-tool-calls-discovery]].

## Что выяснилось (звонок/чат Елене Сакевич, 2026-07-16)
- `10.0.196.14:9034`, куда вёл наш туннель, — **тестовый ADP-сервис** (chat-only façade, без tools-шаблона и парсера). «Для агента не нужен».
- Рабочий tool-capable vLLM — **`10.0.196.12:8004`**, модель **`/models/triva_llm_instruct`**, серверный ключ **`sk-…IBSv11`** (выдан ИВА, не личный ключ Елены).

## Решение
- **Gateway переведён** на `10.0.196.12:8004` (туннель + litellm `tacticum/triva` + ключ в `.env` как `IVA_TRIVA_API_KEY`). Проверено end-to-end: внутренний путь + публичный `https://llm.cifragen.ru`, обычный чат + tools + стриминг → `tool_calls` приходят. Алиас `tacticum/triva` не менялся.
- **helm пока НЕ переключаем** — остаётся на `9034`. Переключение только через eval (golden-set A/B `9034` vs `8004`), т.к. helm-RAG валидировался против `9034` (ADR-0003). Стратегически helm тоже уводить на `8004` (т.к. `9034` — тестовый, могут выключить), но по цифрам.

## Ключевые факты для быстрой сверки
| | значение |
|---|---|
| рабочий host:port | `10.0.196.12:8004` |
| модель | `/models/triva_llm_instruct` |
| ключ | `sk-…IBSv11` → `gateway:/opt/cifragen/.env` (`IVA_TRIVA_API_KEY`), не в git |
| публичный вход агентов | `https://llm.cifragen.ru`, модель-алиас `tacticum/triva` |
| jump | `194.36.208.242` (adp_emb), `permitopen` = 9034 + 8004 |
| откат gateway | `*.bak-20260716-121021`, `authorized_keys.bak-20260716-121014` |

## project-hub: allowlist моделей на ключ (факт, проверено 2026-07-16)
project-hub / LiteLLMAdapter выдаёт виртуальным ключам **явный allowlist моделей** (litellm `models=[...]` на ключ). Проверено ключом helm: он допущен к `cheap/chat/code/flash-3.5/flash-3/embed/rerank/smart/long`, но **`tacticum/triva` в его allowlist НЕТ** (triva раньше ходила мимо Gateway, прямым туннелем). Ошибка при вызове: `key not allowed to access model … Tried to access tacticum/triva`.

**Следствие для агентов:** чтобы боевой ключ (не master) мог звать `tacticum/triva` (в т.ч. tool-calling), в project-hub нужно **добавить `tacticum/triva` в allowlist этого ключа**. Иначе — 400/403 «key not allowed», и это auth, НЕ проблема tools. Master-ключ litellm допущен ко всему (им и проверяли gateway).

## helm — план «по-человечески» (НЕ сделано; когда решим, под eval)
Сейчас helm ходит в triva **в обход Gateway**: прямой туннель `HELM_IVA_LLM_*` → `172.18.0.1:8790` (на `9034`), ключ `ivcs`. Одновременно у helm уже есть Gateway-доступ (`HELM_GATEWAY_BASE_URL=https://llm.cifragen.ru/v1`, `HELM_GATEWAY_API_KEY`) — используется для `tacticum/embed`, `tacticum/rerank`.

Целевой чистый вариант (единая дверь через Gateway, сырой ИВА-ключ спрятан):
1. **project-hub:** добавить `tacticum/triva` в allowlist ключа helm-репы.
2. **helm config:** переключить генерацию с `HELM_IVA_LLM_*` (прямой туннель) на Gateway — `base_url = HELM_GATEWAY_BASE_URL`, ключ `HELM_GATEWAY_API_KEY`, модель `tacticum/triva`. (в коде: клиент генерации → на gateway-клиент, как embed/rerank.)
3. **Убрать** на helm: systemd `iva-triva-tunnel.service`, `HELM_IVA_LLM_*` из `.env`, ключ `ivcs`. Хост `adp-jump` в ssh-config helm остаётся не нужен для triva (проверить, не используется ли ещё для чего-то).
4. Сырой ИВА-ключ `sk-…IBSv11` на helm НЕ кладём — он живёт только в Gateway (upstream). helm авторизуется своим project-hub-ключом.

**Гейт:** это переводит helm-генерацию на `8004` (через Gateway) → RAG валидировался против `9034` (ADR-0003) → **сначала eval** (golden-set A/B `9034` vs `8004`: grounding/вердикты/галлюцинации/латентность), переключать только если `8004 ≥ 9034`. helm-RAG использует plain-chat (tools не нужны) → срочности нет, это про чистоту архитектуры и уход с тестового `9034`.

Соответствует «компромиссу на будущее» из ADR-0003: зарегистрировать triva как внутренний тир самого Gateway.

## Открытый вопрос
Уточнить у Елены: `gpt-oss-120b` из её скрина vs `/models/triva_llm_instruct` из `/v1/models` — опечатка или отдельная модель.
