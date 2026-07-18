---
title: Gemini→Gemma (triva) — локальная генерация в контуре ИВА
type: note
permalink: tacticum/02-architecture/gemini-gemma-triva-lokalnaia-generatsiia-v-konture-iva
tags:
- llm
- gemma
- triva
- vllm
- tunnel
- adr-0003
- rag
- contour
- setup
---

## Суть решения (ADR-0003, принят 2026-07-14)
Генерация ответов RAG идёт на **локальной Gemma** (self-hostable Google), а **не на облачном Gemini** (Google Cloud). Причина — контур данных: чувствительные данные ИВА (требования, переписка заказчиков) наружу не уходят (Слайд 10). Внешние чат-тиры платформенного Gateway (`tacticum/chat` → Gemini/DeepSeek через egress) для этого запрещены.

**НЕ нарушает контур и разрешено:** retrieval (Qdrant/Meili) и reranker (`bge-reranker-v2-m3` через self-hosted TEI/Gateway) — это Tacticum-инфра, текст к третьим сторонам не уходит.

## Как реализовано у нас (helm)
- `triva` = **дообученная Gemma на внутреннем vLLM** в сети ИВА, OpenAI-совместимый API, адрес **10.0.196.14:9034**, достижим только из контура ИВА. **SSE/стриминг не поддерживает** — ответ отдаётся целиком.
- Доступ — через **SSH-туннель** `helm → adp-jump → triva`. adp-jump = **194.36.208.242** (user `tacticum`), точка входа в сеть ИВА.
- Туннель держит **systemd-юнит `iva-triva-tunnel.service`** на helm:
  ```
  ExecStart=/usr/bin/ssh -N -T -o BatchMode=yes -o ExitOnForwardFailure=yes \
    -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o StrictHostKeyChecking=accept-new \
    -F ~/.ssh/config \
    -L 127.0.0.1:8790:10.0.196.14:9034 \
    -L 172.18.0.1:8790:10.0.196.14:9034 \
    adp-jump
  Restart=always
  ```
  Два `-L`: loopback (хост-дебаг) и docker-bridge gateway `172.18.0.1` (чтобы контейнер приложения видел туннель).
- Приложение зовёт triva как обычный OpenAI-endpoint:
  ```
  HELM_IVA_LLM_BASE_URL=http://172.18.0.1:8790/v1
  HELM_IVA_LLM_MODEL=triva
  HELM_IVA_LLM_API_KEY=<ключ>
  ```

## Как настроить на другом сервере/репе (напр. agents)
1. **SSH-доступ к adp-jump** (194.36.208.242, user tacticum) — ключ выдаёт руководитель; запись `Host adp-jump` в `~/.ssh/config` с `IdentitiesOnly yes`.
2. **Поднять свой туннель** (systemd по образцу выше): `-L <bind>:8790:10.0.196.14:9034 adp-jump`. bind = loopback + адрес, который видит контейнер agents (docker-bridge gateway их сети, а не 172.18.0.1 — он у каждого хоста свой; посмотреть `docker network inspect <net>` → Gateway).
3. **Указать приложению OpenAI-endpoint:** `BASE_URL=http://<bind>:8790/v1`, `MODEL=triva`, ключ triva. Клиент — любой OpenAI-совместимый.
4. **Учесть:** стриминга нет (ответ целиком; если код ждёт SSE — отключить/обрабатывать ошибку). `/v1/models` может отдавать 404 — это норм для этого vLLM, работает `/v1/chat/completions` (`/v1/completions`).
5. Проверка живости: `curl http://<bind>:8790/v1/chat/completions -H "Authorization: Bearer <key>" -d '{"model":"triva",...}'`.

## Важные оговорки
- «Меняем Gemini на Gemma» — это **наша трактовка курса**, дословной фразы в транскрипте созвона нет (Gemma там упомянута один раз как агент по коду). Разбор — в `docs/iva-knowledge-vision.md §6`.
- Качество: 15 боевых вызовов, grounded 12/12, вердикты 12/12, negative 3/3, 0 галлюцинаций, ~1с. **triva/Gemma годится для Q&A как есть, свой Qwen не нужен.** Главный риск качества — не модель, а **ретрив** (реранкер/hybrid/чанкинг/eval).
- Техдолг/цель: зарегистрировать triva как **внутренний тир самого Gateway** (нужен маршрут Gateway→сеть ИВА) — тогда соблюдается правило «LLM Gateway везде» без утечки; до этого прямой вызов vLLM через туннель.

Источник: `helm/docs/adr/0003-*.md`, `helm/docs/iva-knowledge-rag-concept-v3.md §6.3`, `helm/docs/iva-knowledge-vision.md §6`. Связано: [[Доступ к двум ВПС- реранкер (gateway) + полная выгрузка данных (ИВА-adp_emb)]].
