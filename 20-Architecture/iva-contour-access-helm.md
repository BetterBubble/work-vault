---
title: iva-contour-access-helm
type: guide
permalink: tacticum/20-architecture/iva-contour-access-helm-1
tags:
- helm
- iva
- tunnel
- adp_emb
- rag2
- ingest
- ops
- sources
---

## Доступ к контуру ИВА с helm (туннели, источники, доставка)

Разведано 2026-07-17. Как helm достаёт данные из внутреннего контура ИВА для RAG#2. Дополняет [[git-deploy-hygiene-helm]] и [[session-state]].

## Ключевой факт
- Прод-helm имеет публичный egress (ya.ru/google резолвятся), но **контур-хосты ИВА (10.x) напрямую НЕ видит** — только через SSH-туннель на джамп-хост.
- **`adp_emb`** (ssh-manager, `194.36.208.242`, hostname `cexfdpqllh`) = сервер ВНУТРИ контура ИВА; в ssh-config helm он же alias **`adp-jump`** (`/home/tacticum-deploy/.ssh/config`). Видит внутренние 10.x. Играет роль ssh-транзита (jump).

## Observations
- [tunnel] helm ходит в контур через SSH `-L` форварды на `adp-jump`, оформлены systemd-юнитами: `iva-sources-tunnel.service` (→ `10.22.0.10:443` jira/wiki, локальный форвард `127.0.0.1:8443`) и `iva-triva-tunnel.service` (→ `10.0.196.14:9034` triva vLLM, форвард `8790` на `127.0.0.1` и `172.18.0.1`). Так helm уже тянет Jira/Confluence для RAG#2.
- [source] `distrohost.msk` = `10.3.7.199` — JUMP-контракты. `Docs/Sessions.html` отдаёт **HTTP 200, 761 KB** pandoc-HTML. Источник для Р-4/Eva-C.
- [source] `beta.hi-tech.org` = `10.0.200.58` — OpenAPI-реестры. Спеки лежат напрямую как JSON (200): `/doc/api/clients-openapi.json` (v2.30.0, ~315 путей, 1.46 MB), `/doc/api/integration-openapi.json` (v1.30.0, ~54), `/doc/api/bot-openapi.json` (v1.30.0, ~4). `.html` Redoc-страницы → 404, `/doc/` → 403. Источник для Р-1.
- [source] `eva.iva.ru` = `10.6.10.9` — EVA-трекер; корень 302 nginx (нужна auth/API-эндпоинт). Источник Eva-A.
- [source] `eva-wiki.orionpro.org` = `10.3.0.245` — Eva-wiki документы (DOC-000245); корень 302 nginx (нужна auth). Источник Eva-B.
- [delivery] **Схема доставки Р-1/Р-4/Eva:** на helm добавить `-L` форварды через `adp-jump` к нужным хостам (по образцу `iva-sources-tunnel`); ингест-джоба крутится **на helm**, тянет спеки/HTML через туннель, пишет и индексирует на helm. Данные оседают на helm, не на adp_emb.
- [constraint] **adp_emb НЕ трогать** (указание пользователя 2026-07-17): ничего не менять/добавлять/ставить, использовать только как ssh-транзит read-only. Вся выгрузка/ингест — на helm.

## Relations
- relates_to [[git-deploy-hygiene-helm]]
- relates_to [[session-state]]
- part_of [[20-Architecture]]


## Туннель iva-contour-sources-tunnel создан (2026-07-17)
Новый systemd-unit `iva-contour-sources-tunnel.service` на helm (по образцу `iva-sources-tunnel`, User `tacticum-deploy`, через `adp-jump`, autorestart). Слушает на loopback + docker-bridge `172.18.0.1` (для контейнера `helm-helm-1`, сеть `helm_default`). Проверено: доставка работает (beta clients-openapi 200/1.46MB, distrohost Sessions 200/761KB с хоста helm).

Маппинг портов (из контейнера helm — `172.18.0.1:<port>`, с хоста — `127.0.0.1:<port>`; для HTTPS нужен правильный SNI/Host):
- **8444** → `10.0.200.58:443` beta.hi-tech.org (OpenAPI: `/doc/api/{clients,integration,bot}-openapi.json`)
- **8081** → `10.3.7.199:80` distrohost.msk (`/Docs/Sessions.html`)
- **8445** → `10.6.10.9:443` eva.iva.ru (EVA-трекер)
- **8446** → `10.3.0.245:443` eva-wiki.orionpro.org (Eva-wiki DOC)

Ингест-джоба на helm обращается к `172.18.0.1:<port>` с заголовком/SNI `Host: <исходный-хост>`. adp_emb — только транзит, не тронут.


## Ингест через туннель + ufw-урок (2026-07-17, разобрано детально)
**Критично: новый туннель-порт требует ufw-allow для контейнера.** Хост `helm` имеет `ufw active` + `INPUT policy DROP`. Контейнер (172.18.0.4) обращается к туннель-форварду через docker-bridge gateway `172.18.0.1:<port>` → попадает в INPUT хоста → **DROP, если нет правила** `ufw allow from 172.18.0.0/16 to any port <port>`.
- triva (8790) имел это правило давно → работал.
- Мой `iva-contour-sources-tunnel` (beta 8444 и др., создан 2026-07-17) правило НЕ получил → из контейнера не работал **с создания** (host-loopback `127.0.0.1:8444` при этом работал — lo не через INPUT-firewall). Это НЕ связано с rebuild (ошибочно подумал сначала).
- **Фикс:** `ufw allow from 172.18.0.0/16 to any port 8444 proto tcp` (копия правила triva). После — контейнер→8444 OK. Правило ufw персистентно, переживает rebuild. Для distrohost/eva/orionpro (8081/8445/8446) при их ингесте — добавить аналогично.

**Ингест api-реестров (Р-1) — рабочий рецепт:** `docker exec -e HELM_API_REGISTRY_TUNNEL_URL=https://172.18.0.1:8444 helm-helm-1 /app/.venv/bin/python -m helm.ingest.api_index` → фетчит 3 спеки через beta-туннель (SNI/Host=beta.hi-tech.org, tls_verify off) → пишет `data/real/api/*.operations.json` (410 операций после дедупа). Тул `api_registry_check` читает этот dir. Проверено вживую: «удалить сообщение чата»→removeMessage, «отозвать письмо»→not_found.
- ⚠️ **Эфемерно:** пишет в `/app/data/real/api` ВНУТРИ контейнера (нет volume-mount) → **сотрётся при следующем rebuild/recreate**. Для персистентности нужен volume `./data/real/api:/app/data/real/api` в `docker-compose.prod.yml` **через git** (прод авто-pull'ится — прямая правка прода дрейфит) + `HELM_API_REGISTRY_TUNNEL_URL` в compose environment (рядом с RERANK-флагами, они там в git).

## Уточнение к «rebuild ломает туннели» (из [[git-deploy-hygiene-helm]])
НЕВЕРНО. Рантайм RAG#2 использует Qdrant/Meili `10.16.0.19` НАПРЯМУЮ (не туннели) + Gateway публичный → rebuild рантайм не ломает. Туннели нужны только для ИНГЕСТА. После rebuild туннели рестартить для рантайма НЕ нужно. beta-туннель из контейнера не работал из-за ufw (см. выше), а не из-за rebuild/bridge.