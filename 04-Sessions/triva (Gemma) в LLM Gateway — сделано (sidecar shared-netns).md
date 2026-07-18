---
title: triva (Gemma) в LLM Gateway — сделано (sidecar shared-netns)
type: note
permalink: tacticum/04-sessions/triva-gemma-v-llm-gateway-sdelano-sidecar-shared-netns
tags:
- triva
- gemma
- gateway
- litellm
- sidecar
- tunnel
- rag
- done
---

## Что сделано (2026-07-15) — задача 1Б
Тир `tacticum/triva` (self-hosted Gemma в контуре ИВА) поднят в LLM Gateway. Работает: `/v1/chat/completions` model=tacticum/triva → 200, ~0.5–2.6с. Все прочие тиры (embed/chat/rerank) целы.

## Архитектура
- vLLM triva: `10.0.196.14:9034` в сети ИВА, токен `Bearer ivcs`, достижим только с adp_emb (VPN).
- Туннель — **sidecar-контейнер** `triva-tunnel` (`kroniak/ssh-client`) в `/opt/cifragen/docker-compose.yml`. Делит netns с litellm (`network_mode: "service:litellm"`), форвардит `ssh -L 127.0.0.1:8790:10.0.196.14:9034 root@adp_emb`. litellm ходит `api_base: http://127.0.0.1:8790/v1`, `api_key: ivcs`.
- Ключ туннеля: `/root/.ssh/triva_tunnel` на gateway; на adp_emb в authorized_keys с ограничением `restrict,port-forwarding,permitopen="10.0.196.14:9034"` (только проброс к triva, ни шелла, ни др. портов).

## Сетевой урок (важно на будущее)
Контейнер gateway НЕ может надёжно ходить к triva через:
- host-bound туннель (`-L 172.18.0.1:8790` + systemd на хосте) → тело ответа зависает (MTU black-hole на VPN-пути + docker bridge; хост при этом работает);
- sidecar с `-L 0.0.0.0:8790` (cross-container) → ssh не открывает канал для не-loopback источника (GatewayPorts-нюанс) → reset.
**Рабочий паттерн:** sidecar делит netns с потребителем + `-L 127.0.0.1` (loopback). MSS-clamp НЕ понадобился.
**Операц. нюанс:** при рестарте litellm сайдкар надо рестартить следом (rejoin netns); `docker compose up -d` это делает через depends_on, `docker restart litellm` — нет.

## Контур данных
Промпт идёт litellm(cifragen) → sidecar → adp_emb → triva(ИВА). Наружу к третьим сторонам не уходит. Транзит через cifragen-хост — по указанию руководителя (подключить triva в Gateway); формально это тот самый «маршрут Gateway→сеть ИВА» из ADR-0003 (был техдолгом).

## Осталось
1. **Доступ ключам (project-hub):** добавить `tacticum/triva` (и `tacticum/rerank`) в allowlist RAG-групп. Сейчас ни у одного ключа их нет. triva — opt-in: helm/др. переходят на неё, когда захотят (пока helm юзает свой прямой туннель).
2. **helm при желании:** переключить генерацию с прямого туннеля на `tacticum/triva` (helm-side, отдельно).
3. Аналогично реранку — примирить с ADR (генерация через Gateway-транзит).

Связано: [[Gemini→Gemma (triva) — локальная генерация в контуре ИВА]], [[Реранкер в LLM Gateway — сделано (DeepInfra Qwen3-Reranker-4B)]], [[Доступ к двум ВПС- реранкер (gateway) + полная выгрузка данных (ИВА-adp_emb)]].
