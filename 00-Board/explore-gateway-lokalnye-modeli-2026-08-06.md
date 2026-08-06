---
title: 'Разведка: LLM-гейтвей — можно ли собрать контур «данные не уходят наружу»'
type: report
status: draft
tags:
- board
- gost-docs
- gateway
- explore
permalink: tacticum/00-board/explore-gateway-lokalnye-modeli-2026-08-06
---

# Разведка: гейтвей и локальные модели (2026-08-06)

Read-only разведка по запросу тимлида. Ничего не менял. Значения секретов не выписывались — только имена переменных.

Источники: сервер `gateway` (155.212.134.20), `/opt/cifragen/litellm/config.yml` (правлен 2026-08-01),
`/opt/cifragen/docker-compose.yml`, БД `litellm` в `cifragen-postgres-1`,
`~/tacticum/project-hub/project-hub/src/project_hub/adapters/litellm.py`.

## 1. Полный список моделей в `model_list`

**Наши собственные эндпоинты (трафик не покидает наш/заказчика контур):**

| Алиас | Куда ведёт | Комментарий |
|---|---|---|
| `tacticum/embed` | `http://embedding-api:8000/v1`, модель `openai/bge-m3` | контейнер `cifragen-embedding-api-1` на том же хосте, docker-сеть `tei`. Полностью локально |
| `tacticum/triva` | `http://127.0.0.1:8790/v1`, модель `openai//models/triva_llm_instruct` | **единственная локальная генеративная**. 8790 — sidecar SSH-туннель `cifragen-triva-tunnel-1`: `gateway → root@31.129.105.187 → 10.0.196.14:8004` (vLLM с Gemma в контуре ИВА). Живой, `/v1/models` отдаёт `/models/triva_llm_instruct` и `-backup` |

**Внешние API (трафик уходит в интернет, весь egress — через прокси `37.1.194.238:8888`, DE):**

| Алиас | Провайдер / модель |
|---|---|
| `tacticum/cheap` | Google Gemini `gemini-3.1-flash-lite` |
| `tacticum/chat` | Google Gemini `gemini-2.5-flash` |
| `tacticum/code` | DeepSeek `deepseek-v4-pro` |
| `tacticum/flash-3.5` | Google Gemini `gemini-3.5-flash` |
| `tacticum/flash-3` | Google Gemini `gemini-3-flash-preview` |
| `tacticum/flash-3.5-flex` | Google Gemini `gemini-3.5-flash`, `service_tier: flex` |
| `tacticum/smart` | Google Gemini `gemini-3.1-pro-preview` (gated) |
| `tacticum/vision` | Google Gemini `gemini-2.5-flash` (gated) |
| `tacticum/long` | Google Gemini `gemini-2.5-pro` (gated) |
| `tacticum/rerank` | DeepInfra `Qwen/Qwen3-Reranker-4B` |
| `tacticum/transcribe` | DeepInfra `openai/whisper-large-v3` (gated) |
| `fb/gemini-flash-lite`, `fb/gemini-flash`, `fb/gemini-pro` | Google Gemini |
| `fb/deepseek-flash`, `fb/deepseek-pro` | DeepSeek |
| `fb/gpt` (`openai/gpt-5.5`), `fb/gpt-mini` (`openai/gpt-5.4-mini`) | OpenAI |
| `or/gemini-3.5-flash`, `or/gemini-3-flash`, `or/auto` | OpenRouter |

Итого: 24 алиаса, из них **локальных — 2** (`embed`, `triva`), остальные внешние.
Ключи провайдеров в env: `GEMINI_API_KEY`, `DEEPSEEK_API_KEY`, `OPENAI_API_KEY`, `DEEPINFRA_API_KEY`,
`OPENROUTER_API_KEY`, `TEI_GATEWAY_API_KEY`, `IVA_TRIVA_API_KEY` (значений не смотрел).

## 2. Есть ли уже локальные модели

Да, но ровно одна генеративная — `tacticum/triva`. И **важная оговорка: это контур ИВА (заказчика), а не наш.**
vLLM крутится на `10.0.196.14:8004` внутри их сети, гейтвей ходит туда SSH-туннелем через `adp_emb`.
Своего vLLM/ollama/TGI на наших серверах нет: в `docker ps` на gateway только litellm, traefik,
postgres, redis, prometheus, grafana, туннель и TEI-эмбеддинги (`tei-service:dev`,
`ghcr.io/huggingface/text-embeddings-inference:cpu-1.6`).

За 7 дней по `LiteLLM_SpendLogs` triva реально работает: 253 вызова `openai//models/triva_llm_instruct` + 35 через алиас.

## 3. Механизм ключей — можно ли выдать ключ только на локальные модели

**Технически — да.** Ключ выдаётся `POST /key/generate` на прокси с master-key в Bearer; в теле
передаётся `models: [...]` — это allow-list, который прокси проверяет **на этапе аутентификации,
до роутинга**. Подтверждение из живых логов контейнера:

```
key not allowed to access model. This key can only access models=['tacticum/cheap', ...,
'tacticum/triva']. Tried to access tacticum/flash-3.5-flex
```
(`user_api_key_auth` → `ProxyException`, HTTP 401/403)

То есть ключ с `models: ["tacticum/triva", "tacticum/embed"]` физически не сможет вызвать Gemini/OpenAI.

Текущая раскладка (10 ключей в БД, `LiteLLM_VerificationToken`): у всех в списке внешние тиры;
`tacticum/triva` есть только у `control-tower` и `tacticum-agents`. У всех 10 команд
(`LiteLLM_TeamTable`) `models = {}` — команда не ограничивает, ограничивает ключ.

**Три оговорки, каждая ломает гарантию:**

1. **`router_settings.fallbacks` содержит `{"tacticum/triva": ["tacticum/cheap"]}`** — то есть при
   недоступности triva запрос по конфигу уходит в Gemini. Это ровно то, чего заказчик не хочет.
   Строка есть в `/opt/cifragen/litellm/config.yml`. Применяется ли к fallback-модели allow-list
   ключа — **не проверял**; но саму строку в любом случае надо снимать для такого ключа/контура.
2. **Через project-hub локально-только ключ создать нельзя.** В
   `.../adapters/litellm.py:17` константа `_OPEN_TIERS` (cheap, chat, code, flash-3.5, flash-3,
   embed, rerank) **всегда** добавляется в `models` и в `ensure_service_key` (:346), и в
   `sync_key_models` (:279). Локальный ключ придётся делать мимо хаба — напрямую master-key'ом.
3. **Прокси-egress общий на весь гейтвей** (`HTTP_PROXY`/`HTTPS_PROXY` на контейнере,
   `NO_PROXY=localhost,127.0.0.1,embedding-api,tei-gte,postgres,redis,langfuse.cifragen.ru`).
   Сетевой изоляции «этот ключ не имеет выхода в интернет» нет — гарантия только на уровне
   allow-list ключа.

## 4. Логирование — чем доказывать

- **БД LiteLLM есть**: postgres `cifragen-postgres-1`, база `litellm`, ~90 таблиц.
  `LiteLLM_SpendLogs` — 313 939 строк, с 2026-06-21 по сейчас.
- В логе на каждый запрос есть: `model`, `model_group`, `custom_llm_provider`, **`api_base`**,
  `api_key` (хеш), `team_id`, `requester_ip_address`, токены, стоимость, время.
  По `api_base`/`custom_llm_provider` **видно, куда реально ушёл запрос** — например, triva-вызовы
  идут с `api_base = http://127.0.0.1:8790/v1/`, `custom_llm_provider = openai`. Это и есть
  доказательная база «наружу не ушло».
- **Содержимое не хранится**: за последние 2 суток из 2691 записи `messages` и `response`
  заполнены в **0** (LiteLLM пишет их только при явно включённом store-prompts). Плюс — меньше
  чувствительных данных в БД; минус — по SpendLogs нельзя показать, ЧТО именно отправляли.
- `LiteLLM_ErrorLogs` пуст (0 строк).
- **Langfuse** (`success_callback`/`failure_callback`) — `https://langfuse.cifragen.ru`,
  DNS → `62.113.107.239`, то есть **наш собственный хост**, не SaaS-облако. Traces обычно содержат
  input/output — значит текст промптов, скорее всего, лежит там. Включён ли захват содержимого и
  какая ретенция — **не проверял** (SSH-доступа к 62.113.107.239 в конфиге нет).
- В `LiteLLM_SpendLogs` за 7 дней засветились `gpt-5.5` (1) и `claude-sonnet-4-6` (1) — то есть
  внешние вызовы через гейтвей идут, в т.ч. по моделям вне списка алиасов.

## 5. `LiteLLMAdapter` — откатит ли ручную правку

Файл: `/Users/bubblemac/tacticum/project-hub/project-hub/src/project_hub/adapters/litellm.py`

- `_OPEN_TIERS` (:17) — 7 внешних+локальных тиров, всегда в списке.
  `_GATED_TIERS` (:22) — `use:llm:smart|vision|long|triva|transcribe` → соответствующий алиас.
- `sync_key_models` (:248) — прямо документирует поведение: *«The hub is the authority: tiers absent
  from capabilities are dropped, which also reverts manual grants made straight in the LiteLLM admin
  panel»*. Правит существующий ключ через `/key/update` **без ротации секрета**.
- **Кто вызывает** (call-sites; Serena символьных ссылок не нашла — вызовы идут через
  адаптер-реестр, поэтому проверял и текстом):
  - `sync_key_models` → `project-hub/src/project_hub/cli/reconcile_llm.py:147` (шаг 4)
  - `ensure_service_key` → `services/llm.py:112`, и `rotate_service_key` (:381)
- **Сценарий отката, который надо иметь в виду:** `reconcile_llm.py` шаг 2 (:88-103) «усыновляет»
  ключи, которых нет в БД хаба, создавая запись с `capabilities=None`. На следующем прогоне шаг 4
  вызовет `sync_key_models(capabilities=set())` → `models` станет ровно `_OPEN_TIERS`, то есть у
  ручного «локального» ключа **отберут `tacticum/triva` и добавят Gemini/DeepSeek**.
- **Смягчает:** `reconcile-llm` — ручная CLI-команда (`cli/__main__.py:44`), крона/таймера под неё
  на сервере `project_hub` нет (`crontab -l` пуст, в `systemctl list-timers` только системные).
  Обходится и структурно: ключ в команде, не привязанной к проекту хаба (`litellm_team_id`),
  reconcile не увидит вовсе — он итерирует только проекты с `litellm_team_id is not null` (:53).

## 6. Живое состояние `docker ps` на `gateway`

```
cifragen-triva-tunnel-1       kroniak/ssh-client:latest                Up 13 hours
cifragen-litellm-1            ghcr.io/berriai/litellm:main-stable      Up 13 hours (healthy)   4000/tcp
cifragen-embedding-worker-1   tei-service:dev                          Up 2 weeks
cifragen-embedding-api-1      tei-service:dev                          Up 2 weeks (healthy)    8000->8000
cifragen-tei-gte-1            hf/text-embeddings-inference:cpu-1.6     Up 2 months (healthy)
cifragen-prometheus-1         prom/prometheus:v2.55.0                  Up 2 months
cifragen-grafana-1            grafana/grafana:11.3.0                   Up 2 months             3000->3000
cifragen-traefik-1            traefik:v2.11                            Up 2 months             80/443/8080
cifragen-postgres-1           postgres:16-alpine                       Up 2 months (healthy)
cifragen-redis-1              redis:7-alpine                           Up 2 months (healthy)
```
Публичный домен гейтвея — `llm.cifragen.ru` (traefik + Let's Encrypt).

---

**ВЕРДИКТ: технически — да, но контур получится узкий и не бесплатный.**

Механизм для гарантии есть и работает: allow-list `models` на виртуальном ключе проверяется на
аутентификации, до роутинга (подтверждено живым 403 в логах). Ключ с `models: ["tacticum/triva",
"tacticum/embed"]` во внешнюю модель уйти не сможет. Цена вопроса — четыре пункта:

1. **Модель одна и она чужая.** Единственная локальная генеративная — `tacticum/triva` (Gemma в
   контуре ИВА через SSH-туннель). Своего инференса у нас нет вообще. «Наши модели» в понимании
   заказчика надо уточнить: если ИВА — свои, контур есть уже сегодня; если требуется наш хостинг —
   его нужно поднимать с нуля.
2. **Снять fallback `tacticum/triva → tacticum/cheap`** из `config.yml`, иначе падение triva
   отправляет данные в Google. Это правка конфига гейтвея (прод).
3. **Ключ выдавать мимо project-hub**, напрямую master-key'ом, и держать его в команде без
   `litellm_team_id`, иначе `reconcile-llm` рано или поздно вернёт в него внешние тиры.
4. **Доказательство — по `api_base`/`custom_llm_provider` в `LiteLLM_SpendLogs`**, содержимое там
   не хранится. Если заказчику нужна не только гарантия, но и аудит содержимого — источник Langfuse
   на нашем хосте, и его настройки надо отдельно проверить.

**Проверено / Данные / Подтверждение / НЕ проверено**

- *Проверено:* состав `model_list` и провайдеры; наличие/адрес локальных эндпоинтов; живость triva
  (`/v1/models` через туннель); allow-list на 10 ключах и пустые `models` на 10 командах; факт
  enforcement allow-list; наличие и схема БД LiteLLM; отсутствие содержимого в SpendLogs; код
  `LiteLLMAdapter` и его call-sites; отсутствие крона у `reconcile-llm`; `docker ps`.
- *Данные:* вывод команд приведён в тексте по каждому пункту; ссылки на файлы и строки — на месте.
- *Подтверждение:* enforcement подтверждён не документацией, а живой ошибкой из
  `docker logs cifragen-litellm-1`; локальность triva — ответом `/v1/models` на `127.0.0.1:8790`;
  отсутствие промптов в БД — счётчиком по 2691 записи за 2 суток.
- *НЕ проверено:* (а) применяется ли allow-list ключа к fallback-модели (проверка требует живого
  запроса — это уже не read-only); (б) пишет ли Langfuse тела запросов/ответов и какая там ретенция
  (нет SSH к 62.113.107.239); (в) политика и SLA vLLM triva со стороны ИВА (чей это сервер
  юридически, кто видит логи на их стороне); (г) выдерживает ли triva нагрузку техписа —
  производительность не мерил.