---
title: explore-nashi-modeli-2026-08-06 — инвентаризация собственных LLM в контуре
type: report
status: draft
tags:
- board
- gost-docs
- models
- explore
permalink: tacticum/00-board/explore-nashi-modeli-2026-08-06
---

# Какие языковые модели у нас РЕАЛЬНО есть внутри контура

Разведка 2026-08-06 под задачу «генерация ГОСТ-документации без выхода данных наружу».
Read-only: чтение vault, `~/tacticum`, чтение 7 серверов из `ssh_list_servers`. Ничего не менял.
Значения ключей нигде не выписаны.

---

## Главное в двух строках

**Собственная генеративная модель у нас ровно одна — `triva` (семейство Gemma), и она не наша:
она стоит на железе ИВА внутри их сети, мы ходим к ней через SSH-туннель.** Всё остальное
генеративное в нашем контуре — внешние API (Gemini, DeepSeek, OpenAI, OpenRouter, DeepInfra).
**И есть два боевых автофолбэка, которые при падении `triva` молча уводят текст в Google Gemini.**

---

## 1. `triva` — единственная собственная генеративная модель. ЖИВА, проверена вызовом

Проверял через прод-`helm` (у контейнера есть креды и туннель).

| Параметр | Значение | Чем подтверждено |
|---|---|---|
| Имя модели | `/models/triva_llm_instruct` (+ `/models/triva_llm_instruct-backup`) | `GET /v1/models` → 200, см. §Подтверждение |
| Контекст | **131 072 токена (128K)**, `max_model_len=max_total_tokens=131072` | ошибка 400 на `max_tokens=999999` |
| API | OpenAI-совместимый, `/v1/chat/completions` | вызов → 200, ответ «Да.», usage 26 токенов |
| Перед моделью | **LiteLLM-роутер**, не голый vLLM | текст ошибки: `litellm.BadRequestError`, `Received Model Group=`, `Available Model Group Fallbacks=['/models/triva_llm_instruct-backup']` |
| Адрес | `10.0.196.14:9034` в сети ИВА, недостижим напрямую | `20-Architecture/Gemini→Gemma (triva) — локальная генерация в контуре ИВА.md:22` |
| Доступ | SSH-туннель `helm → adp-jump → triva`, systemd `iva-triva-tunnel.service`, порт `8790` на `127.0.0.1` и `172.18.0.1` | `systemctl is-active` → `active`; `ss -lntp` → два LISTEN на 8790 |
| Стриминг | **нет**, ответ отдаётся целиком | та же заметка, стр. 22 |
| Владелец | ИВА (их железо, их сеть). Не наш сервис | `iva-contour-access-helm.md:20-22` |

**Какое семейство модели.** Прямого доказательства из рантайма нет — `/v1/models` отдаёт только
путь. Три независимых косвенных совпадения:
- имя артефакта `triva_llm_instruct` совпадает с `s2s_realtime_translator/app/gemma_client.py:235:
  MODEL = "/app/triva_llm_instruct_5"`, где файл называется `Gemma 4 26B LLM client`
  (`00-Board/srv-core-2026-08-03.md:168-184`);
- комментарий в боевом конфиге шлюза: `# ---- self-hosted generation: triva (Gemma) в контуре ИВА
  через sidecar-туннель ----` (`/opt/cifragen/litellm/config.yml:92` на сервере `gateway`);
- **самоотчёт модели** на вопрос о базовой модели: «Моя базовая модель — Gemma 4». Самоотчёт —
  слабое доказательство (модель может повторять промпт-легенду), но совпадает с двумя прочими.

Читается как **внутренне дообученный instruct-вариант Gemma-4 ~26B, версия 5**. Размер параметров
**живьём не подтверждён** — только по README чужого репозитория.

## 2. `iva_adp` — есть, но у нас к нему доступа НЕТ

Всё, что ниже, — из vault (срез коммитов и репозиториев), **не из живого сервиса**: адреса ADP у
нас нет, вызвать его я не мог.

- vLLM 0.19.0, OpenAI-совместимый `/v1/chat/completions`, задачи `assistant` / `summary` /
  `planner` / `planner_adv` / `coder` / `diar_identification`; guided decoding, стриминг,
  Prometheus-метрики движка (`00-Board/srv-core-2026-08-03.md:88-121, 250-253`).
- Владелец — **sys-terra, Воронин Артём Сергеевич**; контейнер `adp-llm`
  (`00-Board/demo-hero-features-iva-part2.md:112`). Разработчик по коммитам — Елена Сакевич.
- **Модель под капотом неизвестна.** Единственная зацепка — шаблон чата
  `iva_adp/app/config.py:239: "<|start_header_id|>assistant<|end_header_id|>\n\n"`, это формат
  **Llama-3**, а не Gemma (`00-Board/srv-core-2026-08-03.md:155`). То есть ADP и `triva`, возможно,
  **разные модели**. Не проверено.
- Ограничителя нагрузки и квот в срезе не найдено вовсе (`srv-core:256`) — очередь видно, но
  перегрузку не предотвращают.

## 3. Terra — генеративной текстовой модели НЕТ

Terra = речь и перевод, не генерация текста. Состав по срезу `terra-core.xml` (523 файла):
ASR `rapi/terradyne/*` (форк faster-whisper на CTranslate2 + silero-vad), диаризация
`rapi/diarizator/*` (вендоренный WeSpeaker, ~70 файлов), MT `translator_service/pretrained_en_ru|ru_en`
(MarianMT), TTS `tts_service/*` (`00-Board/evidence-po-slaydam-2026-08-03.md:50`).
Внутри Terra `vllm` — 0 вхождений, `gpt` — 0, `langchain` — 0 (`evidence-po-slaydam:51`).
«Terra с языковым модулем» — терминологическая натяжка: языковой модуль это отдельный ADP за шлюзом.

## 4. `doc_translator` — Terra/triva в коде НЕ НАЙДЕНА

Задание говорило «Президент вчера прикрутил Terra к `doc_translator`». **В рабочей копии этого нет.**
- `git log` в `~/tacticum/doc_translator` — **один коммит**: `2026-06-18 vsyntot initial commit`.
  `git status` чист, кроме нетронутого `.serena/`.
- Греп по `terra|triva|iva|9034|8790|adp` (py/yml/yaml/md/toml/sh) даёт только совпадения на «IVA»
  как на название продукта в текстах документов и тестах. Ни одного вызова модели.
- Провайдер перевода — **LibreTranslate**: `backend/app/core/settings.py:30:
  LIBRETRANSLATE_URL: str = "http://localhost:5000"`.

Вывод: выкатка 05.08 с «Tacticum API + опция модели ИВА/Terra» (канон
`14-Canon/canon-gost-dokumentaciya…md:197-205`, `00-Board/komandy-gost-docs-2026-08-06.md:63`)
**в этой рабочей копии не лежит** — либо выкачена из другого места, либо не закоммичена.
Что именно там вызывается — **по коду сказать нельзя**, надо спрашивать Президента.

## 5. Шлюз `llm.cifragen.ru` — что наше, что чужое

Боевой конфиг: `/opt/cifragen/litellm/config.yml` на сервере `gateway` (155.212.134.20),
контейнер `cifragen-litellm-1`, рядом `cifragen-triva-tunnel-1` (Up 13 hours).

**Наше, наружу не уходит (2 тира):**
- `tacticum/triva` → `openai//models/triva_llm_instruct`, `api_base: http://127.0.0.1:8790/v1` (:93-97)
- `tacticum/embed` → `openai/bge-m3`, `api_base: http://embedding-api:8000/v1` — self-hosted TEI (:70-73)

**Внешнее (всё остальное):** `tacticum/cheap`, `chat`, `flash-3`, `flash-3.5`, `flash-3.5-flex`,
`vision`, `long`, `smart` → **Gemini**; `tacticum/code` → **DeepSeek**; `tacticum/transcribe` →
DeepInfra whisper-large-v3; `tacticum/rerank` → DeepInfra `Qwen/Qwen3-Reranker-4B`; фолбэк-депл-ты
`fb/gpt*` → **OpenAI**, `or/*` → **OpenRouter**.

Техдолг из ADR-0003 («зарегистрировать triva как внутренний тир Gateway») — **закрыт**, тир есть.

## 6. РИСК: два боевых автофолбэка уводят текст в Google

Это главное для задачи «данные не должны уходить наружу».

1. **В самом шлюзе:** `router_settings.fallbacks: - { "tacticum/triva": ["tacticum/cheap"] }`
   (`config.yml:140`). `tacticum/cheap` = `gemini/gemini-3.1-flash-lite`. То есть **запрос к
   «нашей» модели через шлюз при недоступности triva автоматически уходит в Gemini** — молча,
   без ошибки вызывающему.
2. **На проде helm:** `HELM_RAG2_SYNTH_FALLBACK_LLM_BASE_URL=https://llm.cifragen.ru/v1`,
   `HELM_RAG2_SYNTH_FALLBACK_LLM_MODEL=tacticum/cheap`, ключ выставлен, primary timeout 45с.
   Механизм описан в `helm/src/helm/config.py:388-394`: «если она недоступна/тайм-аутит/ошибается —
   синтез АВТОМАТИЧЕСКИ уходит на этот fallback». **Фолбэк взведён, не выключен.**

Для документации, идущей в Минобороны, оба пути надо гасить явно: звать `triva` напрямую
(`http://<bind>:8790/v1`), либо снимать фолбэк с тира. Само по себе «мы ходим в свою модель через
шлюз» **гарантии контура не даёт**.

## 7. Другие собственные генеративные модели — не найдено

Свип по `~/tacticum` (py/yml/yaml/toml/env) паттернами `vllm|ollama|llama.cpp|text-generation|qwen|
mistral|saiga|yandexgpt|gigachat|gemma|triva`: содержательных попаданий нет — только `pnpm-lock.yaml`,
QA-манифесты, `tei_service/models.yaml` (эмбеддеры) и `KB-Brownfield-Bootstrap` (клиенты к API).
`ollama`, `llama.cpp`, `saiga`, `yandexgpt`, `gigachat` — **ноль** развёртываний.
По корпусу ИВА (216 161 коммит) `gemma`/`deepseek`/`saiga`/`ollama` — тоже ноль
(`00-Board/devils-advocate-2026-08-03.md:301, 357`).

## 8. Железо: GPU у нас НЕТ ни на одном сервере

Проверены все 7 серверов из `ssh_list_servers` + джамп-хост контура:

| Сервер | Хост | `nvidia-smi` |
|---|---|---|
| `helm` | 159.194.233.33 | `not found` |
| `gateway` | 155.212.134.20 | `not found` |
| `zu_demo` | 159.194.212.2 | `not found` |
| `tacticum_prod` | 159.194.224.59 | `not found` |
| `project_hub` | 45.141.79.157 | `not found` |
| `teststand` | 38.180.236.39 | `not found` (и docker нет) |
| `adp_emb` / `adp-jump` | 194.36.208.242, hostname `uslgvhuwxc` | `command not found` |

`adp_emb` напрямую по SSH **не отвечает** (`Timed out while waiting for handshake`) — достучался
через `helm` штатным ключом `tacticum-deploy`. Его адреса: `31.129.105.187 10.33.134.137 10.16.0.23`.
GPU там нет — это транзит, а не машина с моделью.

**Вывод по железу: своих GPU у нас ноль. Вся генерация — либо на железе ИВА (`10.0.196.14`,
недоступно нам ни по SSH, ни для замера), либо во внешних API.** Поднять свою модель под ГОСТ-задачу
сегодня не на чем.

---

## ВЕРДИКТ

Внутри контура у нас есть **ровно одна доступная собственная генеративная модель — `triva`,
семейство Gemma-4, instruct-дообученная, 128K контекста, OpenAI-совместимая, без стриминга.**
Она тянет: Q&A по данным ИВА (15 боевых вызовов: grounded 12/12, вердикты 12/12, negative 3/3,
0 галлюцинаций), структурирование свободного текста в черновик требования (`intake.py:156`),
синтез «мозг-плана» RAG#2 (~15-25с на ответ ~600 токенов). 128K контекста означает, что в один
запрос влезает ГОСТ-документ целиком вместе с образцом — для задачи техписа это скорее хорошо.

Чего она НЕ даёт: она **не наша** — стоит на железе ИВА, доступ через туннель, падает
(в `helm` под это заведён целый обходной механизм), стриминга нет, квот и приоритетов на ней не
видно. `iva_adp` — вторая генеративная точка, вероятно на Llama-3, но **доступа к ней у нас нет**.
Terra генеративной модели не содержит вовсе. Своих GPU — ноль.

**Для требования «данные только через наши модели» ключевое: сегодня два боевых автофолбэка
(§6) уводят текст в Google Gemini при падении triva. Пока они взведены, формулировка
«гоняем через свою модель» неверна.**

**Проверено:** 7 серверов (`nvidia-smi`, `docker ps`) + `adp-jump` через helm; живой вызов triva
(`/v1/models`, `/v1/chat/completions`, проба контекста); боевой конфиг LiteLLM на `gateway`;
env прод-контейнера `helm-helm-1`; `~/tacticum/doc_translator` (git log, greps, settings);
свип `~/tacticum` по 11 паттернам движков; ~12 заметок vault.

**Данные:** собственных генеративных моделей доступно — **1** (`triva`); из них с известным
семейством — 1 (Gemma-4, ~26B по чужому README, живьём не подтверждено); контекст — **131 072**
токена; served-моделей на эндпоинте — **2** (основная + `-backup`); тиров шлюза всего — **19**,
из них не выходящих наружу — **2**; GPU у нас — **0** на 7 серверах; автофолбэков наружу — **2**.

**Подтверждение (команды и вывод):**
```
helm$ systemctl is-active iva-triva-tunnel.service   → active
helm$ ss -lntp | grep 8790                           → LISTEN 127.0.0.1:8790, 172.18.0.1:8790 (ssh pid 4156866)
helm$ docker exec helm-helm-1 … GET $BASE/models     → 200 {"data":[{"id":"/models/triva_llm_instruct"},
                                                              {"id":"/models/triva_llm_instruct-backup"}]}
helm$ … POST /chat/completions max_tokens=999999     → 400 litellm.BadRequestError: max_model_len=
                                                          max_total_tokens=131072 …
                                                          Available Model Group Fallbacks=
                                                          ['/models/triva_llm_instruct-backup']
helm$ … POST /chat/completions «работаешь?»          → 200 model=/models/triva_llm_instruct
                                                          content='Да.' usage total_tokens=26
gateway$ docker ps                                   → cifragen-litellm-1 (Up 13h, healthy),
                                                          cifragen-triva-tunnel-1 (Up 13h)
gateway$ grep … /opt/cifragen/litellm/config.yml:140 → - { "tacticum/triva": ["tacticum/cheap"] }
helm$ docker exec helm-helm-1 env | grep RAG2_SYNTH  → FALLBACK_LLM_MODEL=tacticum/cheap,
                                                          FALLBACK_LLM_BASE_URL=https://llm.cifragen.ru/v1
doc_translator$ git log --oneline                    → 1 коммит: 2026-06-18 vsyntot initial commit
```

**НЕ проверено / осталось:**
1. **Реальное семейство и размер `triva`** — прямого пруфа нет, только имя артефакта + комментарий
   + самоотчёт. Даст `ls /models` на самой машине ИВА, куда доступа нет.
2. **Модель под `iva_adp`** и его адрес — доступа нет вовсе. Шаблон Llama-3 намекает, что это НЕ
   Gemma, то есть моделей у ИВА может быть две. Спрашивать Воронина.
3. **Что именно выкачено в `doc_translator` 05.08** — в рабочей копии этого кода нет (§4).
   Вопрос Президенту: откуда выкатывалось и что там за «модель ИВА/Terra».
4. **GPU под `triva`** — сколько карт, какие, есть ли запас под второго потребителя. Замерить
   нечем: `10.0.196.14` по SSH нам недоступен. Без этого нельзя обещать, что ГОСТ-нагрузка
   не подвинет ВКС-стенограммы: ограничителя нагрузки в ADP не найдено (`srv-core:256`).
5. **Политика ИВА по нашим данным на их железе** — юридически «внутри контура» != «внутри нашего
   контура». Для документации с грифом это отдельный вопрос, кодом не решается.