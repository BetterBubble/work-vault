---
title: triva proxy tool-calling — DISCOVERY (explorer handoff)
date: 2026-07-16
author: explorer
status: RESOLVED 2026-07-16 — рабочий эндпоинт найден (10.0.196.12:8004), gateway переведён
permalink: tacticum/00-inbox/triva-proxy-tool-calls-discovery
---

# triva proxy tool-calling — разведка (read-only)

## ✅ РЕЗОЛЮЦИЯ (2026-07-16, обновлено после звонка Елене Сакевич)
**Причина была верна на 100% — мы ходили не туда.** `10.0.196.14:9034` — это **тестовый ADP-сервис** (chat-only façade без tools-шаблона и без парсера), «для агента не нужен» (слова Елены). Рабочий tool-capable инстанс — **на другом хосте**.

**Рабочий эндпоинт (проверено, tool_calls приходят):**
- host:port — **`10.0.196.12:8004`** (полноценный vLLM: `/v1/models`, `/version` = 200; поддерживает стриминг).
- модель — **`/models/triva_llm_instruct`** (то самое имя из примера Елены; `/v1/models` отдаёт только её).
- ключ — **серверный api-key vLLM `sk-…IBSv11`** (НЕ личный ключ Елены; выдан ИВА). Хранится в `gateway:/opt/cifragen/.env` как `IVA_TRIVA_API_KEY`, в git не идёт. Старый `ivcs` там даёт 401.
- проба: `prompt_tokens` с tools = 66/70 (инструменты рендерятся!), `finish_reason=tool_calls`, `tool_calls=[get_weather{"city":"Самара"}]`.

**Gateway ПЕРЕВЕДЁН (2026-07-16) и проверен end-to-end** (внутренний путь + публичный `https://llm.cifragen.ru`, обычный чат + tools + стриминг):
- `adp_emb:/root/.ssh/authorized_keys` — `permitopen` += `10.0.196.12:8004` (старый `9034` оставлен для отката).
- `gateway docker-compose.yml` — sidecar `triva-tunnel`: `-L 127.0.0.1:8790:10.0.196.12:8004`; проброс `IVA_TRIVA_API_KEY` в litellm.
- `gateway litellm/config.yml` — `tacticum/triva`: `model: openai//models/triva_llm_instruct`, `api_key: os.environ/IVA_TRIVA_API_KEY` (алиас `tacticum/triva` НЕ меняли — приложения не трогаем).
- Бэкапы: `*.bak-20260716-121021` (gateway), `authorized_keys.bak-20260716-121014` (adp_emb).

**helm НЕ тронут** — по-прежнему на `9034` (plain-chat RAG работает). Переключать helm на `8004` — **только через eval** (golden-set A/B `9034` vs `8004`: grounding/вердикты/галлюцинации/латентность), т.к. RAG валидировался против `9034` (ADR-0003). Плюс `9034` — тестовый сервис, могут выключить → в долгую helm всё равно увести на `8004`, но по цифрам.

**Открытый вопрос:** в скрине Елены модель была `gpt-oss-120b`, а `/v1/models` на `8004` отдаёт `/models/triva_llm_instruct` — уточнить, опечатка это или отдельная модель.

---

## Вердикт (одной фразой)
**Tool-calling через НАШУ цепочку (Gateway/litellm `tacticum/triva`) НЕ работает и к использованию НЕ пригоден** — апстрим-эндпоинт ИВА `10.0.196.14:9034`, куда ведут ОБА наших туннеля, полностью игнорирует `tools`/`tool_choice`. Это НЕ наш litellm/модель-нейм/drop_params — проблема выше по цепочке, на стороне ИВА-эндпоинта. Наша helm-цепочка tool-calling и не использует (plain-chat + RAG-цитаты) — фича нужна другому харнессу (Dmitrii), который ходит в ДРУГОЙ эндпоинт.

---

## Фактическая схема проксирования triva

Оба пути форвардят на ОДИН и тот же апстрим `10.0.196.14:9034` (внутри сети ИВА):

**Gateway-путь (litellm):**
- sidecar `cifragen-triva-tunnel-1` в `/opt/cifragen/docker-compose.yml`, `network_mode: service:litellm` (делит netns с litellm). Статус: `Up 21h`.
- форвард: `-L 127.0.0.1:8790:10.0.196.14:9034 root@194.36.208.242` (через adp_emb/adp-jump), ключ `/root/.ssh/triva_tunnel`.
- litellm `config.yml` (`/opt/cifragen/litellm/config.yml`):
  ```yaml
  - model_name: tacticum/triva
    litellm_params:
      model: openai/triva
      api_base: http://127.0.0.1:8790/v1
      api_key: ivcs
  ```
  + глобально `litellm_settings: drop_params: true`. Никаких tool/function-специфичных настроек нет.
- litellm мапит `tacticum/triva` → провайдер `openai/`, шлёт апстриму имя модели `triva`.

**helm-путь (прямой):**
- systemd `iva-triva-tunnel.service` на helm — `active (running) 24h`.
- форвард: `-L 127.0.0.1:8790:10.0.196.14:9034 -L 172.18.0.1:8790:10.0.196.14:9034 adp-jump`.
- приложение helm ходит `iva_llm_base_url` (=`http://172.18.0.1:8790/v1`), НО tools не шлёт (см. п.5).

Итог: обе трубы = один апстрим `9034`.

---

## Ответы по пунктам

### 1. Как зарегистрирована triva в litellm / model-name mismatch
- Зарегистрирована как `tacticum/triva` → `openai/triva`, api_base `http://127.0.0.1:8790/v1`, api_key `ivcs`. `drop_params: true`.
- **Mismatch имени модели — НЕ проблема для этого апстрима.** Прямая проба показала: апстрим отвечает `HTTP 200` и на `model="triva"`, и на `model="/models/triva_llm_instruct"` (эхо-поле `model` = что передал). То есть vLLM за 9034 не валидирует имя — грузит единственную модель. Имя из примера Dmitrii несущественно для нашего пути.

### 2. Статус проксирования
- Gateway sidecar `cifragen-triva-tunnel-1`: `Up 21h`, в netns litellm. `litellm` healthy.
- `127.0.0.1:8790/v1/chat/completions` из netns litellm: **отвечает 200**, `/health` = 200.
- helm `iva-triva-tunnel.service`: `active (running) 24h`.
- Проксирование ЖИВОЕ. Проблема не в доступности.

### 3. Версия vLLM и tool-calling у апстрима
- Идентификация эндпоинта `9034` (сырой ответ chat): `server: uvicorn`; конверт `id: chatcmpl-…`, `object: chat.completion`; в choice есть vLLM-специфичные поля `stop_reason`, `token_ids`, `routed_experts` → это vLLM (вероятно кастом-сборка ИВА, MoE).
- **Но endpoint урезан:** `GET /health` = 200, `POST /v1/chat/completions` = 200; а `/v1/models`, `/version`, `/v1/completions`, `/tokenize`, `/metrics` = **404 `{"detail":"Not Found"}`**. Стандартный vLLM OpenAI-сервер отдаёт их все → перед моделью стоит урезанный uvicorn-façade, экспонирующий только chat+health.
- **Версию vLLM read-only с нашей стороны узнать НЕЛЬЗЯ** (`/version` 404). Заявленную Dmitrii 0.19.0 подтвердить не можем.
- `--enable-auto-tool-choice` / `--tool-call-parser`: узнать напрямую нельзя (нет доступа к ps/логам апстрима, только 9034). Но поведенчески (п.4) апстрим tools игнорирует → эффективно tool-parser не задействован на этом эндпоинте.

### 4. Диагностическая проба (неразрушающая) — ПРИЁМКА

**4a. Прямой апстрим (обход litellm), `127.0.0.1:8790`, model=`triva`:**
Гонял `tool_choice` = `auto` / `required` / `named{get_weather}` / `none` с `tools=[get_weather]`, вопрос «Какая погода в Самаре?».
Во ВСЕХ четырёх случаях: `HTTP 200`, `finish_reason: stop`, `tool_calls: []`, `content` = обычная проза («у меня нет доступа к погоде…»).
→ Настоящий tool-capable vLLM при `required`/`named` ОБЯЗАН форсить вызов (guided decoding) — здесь нет. **Апстрим 9034 игнорирует tools/tool_choice.**

**4b. Наш путь end-to-end через litellm gateway (`localhost:4000`), настоящий OpenAI-SDK (`client.chat.completions.create`), model=`tacticum/triva`:**

| Сценарий | Ожидалось | Факт |
|---|---|---|
| (a) погода, `auto` | `finish_reason=tool_calls`, args `{city:...}` | `stop`, `content` проза, `tool_calls=None` |
| (b) 2 инструмента, «сложи 2 и 3» | `tool_calls[add_numbers]` | `stop`, `content="2 + 3 = 5"`, `tool_calls=None` |
| (c) анекдот, `auto` (tool не нужен) | обычный `content`, без tool | ✅ `content`=анекдот, без tool (единственный «правильный» — но тривиально, т.к. tool не вызывается НИКОГДА) |
| (d1) форс named `get_weather` | forced `tool_calls[get_weather]` | `stop`, проза, `tool_calls=None` |
| (d2) `tool_choice=none` | обычный `content` | ✅ проза |

→ SDK-парсинг ни разу не увидел `tool_calls`. **Форс named-функции (d1) НЕ форсит вызов** — killer-признак: слой отдаёт то же, что и сырой апстрим.

**Сравнение путей:** litellm-gateway и прямой апстрим дают ИДЕНТИЧНЫЙ результат (tool_calls пусто). Значит litellm ПРОЗРАЧЕН: `tools` доходят, `drop_params: true` их НЕ режет (провайдер `openai/` поддерживает function-calling, поэтому не дропает). Поломка целиком на апстриме, litellm ни при чём.

### 5. Где у НАС используется tool-calling triva?
- Единственный call-site chat: `src/helm/llm/gateway.py:112` `LlmGateway.chat()` — вызывает `chat.completions.create(model, messages, temperature?, max_tokens?)` и возвращает `response.choices[0].message.content`. **`tools`/`tool_choice` НЕ передаются вообще.**
- Потребители (`infrastructure/assistant/service.py`, `docs_assistant/service.py` и др.) — plain-chat + RAG-цитаты.
- Грепы по `tool_choice`/`tools=`/`functions=` в `src/` — совпадений в LLM-вызовах нет (только несвязанные словари стоп-слов и MCP-реестр `live_mcp.py`).
- **Вывод: наша helm-цепочка tool-calling triva не использует.** Значит фича нужна не нам, а харнессу Dmitrii, который ходит в другой (tool-capable) эндпоинт vLLM, НЕ в наш 9034.

---

## Диагноз (по слоям)
- **Наш прокси (litellm + туннели): OK.** Живой, прозрачный, tools доходят, drop_params не режет, mismatch имени не мешает.
- **Апстрим ИВА `10.0.196.14:9034`: НЕ поддерживает tool-calling** (урезанный vLLM-façade, игнорирует `tools`/`tool_choice`, отдаёт только chat+health). Это единственная причина, и она вне нашей зоны — внутри контура ИВА.
- Пример Dmitrii, где tool_calls приходят, работает против ДРУГОГО эндпоинта (полноценный vLLM 0.19 с `--enable-auto-tool-choice` + parser), не совпадающего с `9034`.

## Открытые вопросы / что нельзя проверить read-only
1. Какой РЕАЛЬНЫЙ адрес/порт tool-capable vLLM у Dmitrii? (`model=/models/triva_llm_instruct` — это served-model-name, не адрес). Наш туннель туда не ходит.
2. Версия vLLM и флаги запуска апстрима (`--enable-auto-tool-choice`, `--tool-call-parser`) — `/version` и `/v1/models` за 9034 = 404, доступа к ps/логам самого vLLM у нас нет.
3. Что за uvicorn-façade стоит перед моделью на 9034 и намеренно ли он режет tools (или это отдельный «chat-only» шлюз ИВА).

## Предлагаемые следующие шаги (НЕ выполнены — только предложение)
1. Запросить у стороны ИВА/Dmitrii точный host:port tool-capable эндпоинта (тот, что в его рабочем примере) и подтверждение флагов `--enable-auto-tool-choice`/`--tool-call-parser`.
2. Если такой эндпоинт есть — перенацелить наш туннель (`triva_tunnel` sidecar на gateway и/или `iva-triva-tunnel.service` на helm) с `10.0.196.14:9034` на него, затем повторить эту же приёмочную пробу.
3. Если целевой порт совпадает с 9034 — просить ИВА включить tool-calling на этом инстансе (флаги vLLM) либо не проксировать через chat-only façade.
4. Приёмку (сценарии 4b a–d1) переиспользовать как регресс после любой перенастройки.

## ⚠️ INJECTION
Не обнаружено. Никакие данные/логи/ответы не пытались выдать себя за инструкции или переопределить ограничения. Секреты в хендоффе замаскированы (`api_key: ivcs` — это внутренний токен ИВА из конфига, не пользовательский секрет; LITELLM_MASTER_KEY читался из env контейнера и НЕ выводился).