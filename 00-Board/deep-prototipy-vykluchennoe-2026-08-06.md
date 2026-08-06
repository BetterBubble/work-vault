---
title: Глубокая разведка — прототипы и выключенное (кандидаты в заявку на льготу)
type: report
status: draft
role: explorer (разведчик)
created: 2026-08-06
tags:
- board
- инновации
- explore
permalink: tacticum/00-board/deep-prototipy-vykluchennoe-2026-08-06
---

# Прототипы и выключенное — что можно честно заявить

Дозаход поверх [[audit-intellektualnyh-fich]] и [[recon-ai-one-2026-08-03]]. Правило то же:
`НАЙДЕНО` = файл+символ; снимки repomix `--compress` от **2026-07-03**, тела функций местами
вырезаны, поэтому «не найдено» ≠ «нет».

Корпус: `~/tacticum/helm/data/real/git/` — `repomix/{rn,terra-core}.xml`, `all_commits.csv`
(216 161 коммит), `repo_activity_all.csv`, `repos_registry.csv`, `repos_manifest.csv`,
`~/tacticum/helm/data/real/jira/{jira_epics.csv,jira_issues.csv,ivaone/}`.

**Оговорка по методу:** MCP `helm-analyst` (analyst_search/analyst_context) в моей сессии
не поднялся — тулов нет в наборе. Confluence/Jira я смотрел по локальным выгрузкам, а они
**сэмпл**: 3 071 задача и 3 155 страниц вики, не весь трекер. Значит по Jira мой ответ
неполный по построению — см. §7.

---

## 1. `ione` — что это, чьё, живое ли

**Это не «репозиторий `rn`».** `rn/rn` в GitLab — *staged-срез* чужого монорепо: `README.md`
дословно — «Staged from the `iva-connect` monorepo via `tools/stage-rn-repo/stage.sh` in that
upstream workspace». Настоящее дерево живёт локально у автора: `DEPLOY.md` трижды даёт путь
сборки образов — `/Users/ev/ione/connect/{ai-gateway,bpm-service,call-control-gateway}/Dockerfile`.

**Атрибуция — прямая, из наших же реестров, а не мой вывод:**
`repos_manifest.csv` → `rn` = «Ядро 2.0 (RN-федерация, **прототип учредителя**)»;
`git/README.md` → «снапшот **учредителя** ~месячной давности, дорабатывается спецами ИВА,
**акционер разрабатывает дальше в upstream**». Автор в git — `EV <ev@EVMed.local>` и
`Evgeniy Terentyev <et@iva.ru>`; в `identity_gaps.csv` оба — «git без корп. email», в
`identity_map.csv` их нет.

**Хронология (`all_commits.csv`, `repo_activity_all.csv`: project_id 1452, 53 коммита,
8 контрибьюторов, 6 веток, 19 смерженных MR):**

| Дата | Кто | Что |
|---|---|---|
| 2026-04-16 | Evgeniy Terentyev `et@iva.ru` | `Initial commit` (пустой репо) |
| 2026-04-21 | EV | `Initial import: RN modules, Angular host, e2e, docs` — вываливание готового прототипа целиком |
| 2026-04-23 | EV ×3 | `Refresh: calendar MCU join flow…`, `Fixes: WS URL slashes, chat caps, Jump token flush…`, `Update: authed attachments, calls lifecycle, nav refresh, e2e coverage` |
| 2026-05-22 | EV | `Sync from sbox (2026-05-22): RN modules, Angular host, e2e refresh` — **последний коммит автора** |
| 2026-06-26 → 2026-07-03 | Савицкий, Литовкин, Платонов, Корнюшин, flashfire | ~47 коммитов за 7 дней, тикеты `IVAONE-12443…12525`, CI |

**Вывод по «прототип одного человека против почти продукта» — и то, и другое, но разное.**
Ядро (RN-модули, ассистент, гейтвеи) написано одним человеком вне ИВА и приезжает
squash-снимками. Штатная команда ИВА с 26.06 разрабатывает поверх него **продуктовые** фичи
(настройки, календарь, контакты, ВКС, опросы) — и **ни одного коммита по ИИ**: проверил
подстроками `ai|assist|llm|gpt|deepseek|qdrant|bpm|mini|rag|summ` по всем коммитам после
2026-06-01 — 0 попаданий. То есть ИИ-контур команда пока не трогает вообще.

**Зрелость — по тестам, а не по ощущению.** В срезе **205 файлов e2e** (Playwright), из них
по ИИ и смежному: `chat-assistant-real.spec.ts`, `chat-assistant-draft-real.spec.ts`,
`mail-assistant-task-real.spec.ts`, `mail-assistant-real-probe.spec.ts`,
`assistant-calendar-confirmation-real.spec.ts`, `communications-assistant-smoke.spec.ts`
+ 6 отдельных playwright-конфигов к ним; по BPM — 4 спеки; по мини-приложениям — 10, включая
три `miniapps-hr-tek-*` **против живого HR Tek**. Суффикс `-real` = прогон против живого
бэкенда, логин из `.env`, с обязательным «skip loudly» при отсутствии учёток.

**Чего в GitLab-срезе НЕТ, хотя в upstream есть** (по `DEPLOY.md`, который перечисляет
модули для деплоя): `rn-communications`, `rn-calls`, `rn-miniapps`, `rn-bpm` — а также
исходники всех трёх Node-сервисов. В срезе только `rn-shared/src/{bpm,miniapps}/` (типы,
клиенты, SDK, mock-гейтвей). **Практический смысл: без доступа к `/Users/ev/ione/` заявлять
готовность ИИ-слоя нельзя — половина кода вне досягаемости.**

---

## 2. Векторная память с учётом прав — что именно и что значит «выключена»

Схема разобрана по `web/tools/deploy/host-scripts/docker-compose.ai-gateway.yml` и §6.8
`DEPLOY.md` (полный скаффолд `/opt/ai-gateway/.env`, 40+ переменных).

- **Движок:** Qdrant (`qdrant/qdrant:latest`), коллекция `assistant_pointer_memory`,
  `AI_GATEWAY_QDRANT_URL=http://qdrant:6333`.
- **Эмбеддер:** `michaelf34/infinity:latest`, модель `BAAI/bge-m3`, 1024 измерения,
  `AI_GATEWAY_EMBEDDINGS_PATH=/embeddings`.
- **Права — не декларация, а параметры:** `AI_GATEWAY_RAG_REQUIRE_ACL_FINGERPRINT=true`,
  `AI_GATEWAY_RAG_TENANT_ID`, `AI_GATEWAY_RAG_CLASSIFICATION=internal`,
  `AI_GATEWAY_RAG_POINTER_TTL_DAYS=30`, `AI_GATEWAY_RAG_MIN_SCORE=0.60`.
- **Память указателей, а не копий** — дословно из `DEPLOY.md`: «Store embeddings plus opaque
  source refs and safe metadata only. Raw chat messages, emails, videoconference minutes,
  prompts, and generated memory text remain in source systems and are **fetched
  just-in-time after IVA authorization**»; и про индексацию: «The raw source text is fetched
  to create embeddings and **discarded before Qdrant persistence**».
- **Источники:** `AI_GATEWAY_RAG_SOURCE_TYPES=chat_message,email_message`; сканер берёт
  20 чатов × 20 сообщений и 30 писем × 4 ящика, лимит 80 источников.

**«Выключена» — это флаг, а не недоделка.** Три независимых гейта, все `false` в скаффолде:
`AI_GATEWAY_RAG_ENABLED` (поднимает профиль compose `rag` с Qdrant и эмбеддером),
`AI_GATEWAY_RAG_RECOMMENDATIONS_ENABLED` (второй гейт — «operators can index/test pointers
before the browser-facing assistant uses them»), `AI_GATEWAY_RAG_RECENT_INDEX_ENABLED`.
Включение описано как штатная операция: `sudo env COMPOSE_PROFILES=rag docker compose up -d
--force-recreate api qdrant embeddings`. Плюс есть смоук-скрипты `npm run smoke:ai-gateway:rag`
и `smoke:ai-gateway:cache`, и внутренняя ручка `POST /internal/rag/index-recent`
(намеренно не выведена через nginx).

**Оговорка честности:** выключено — потому что не заполнены `AI_GATEWAY_QDRANT_API_KEY` и
`AI_GATEWAY_INTERNAL_INDEX_TOKEN` («operator fills in»). Работоспособность контура в живую
я не проверял — исходников гейтвея в корпусе нет.

**Корпоративная память коммуникаций** («коммуникационная вики») — тот же гейтвей, режим
`AI_GATEWAY_WIKI_ENABLED=false`, том `wiki-data`, `AI_GATEWAY_WIKI_DIR=/var/lib/ai-gateway/wiki`,
`AI_GATEWAY_WIKI_REQUIRE_SOURCE_REFS=true`, паттерн «`raw sources -> wiki -> schema`»:
выведенные факты хранятся markdown-страницами с обязательной ссылкой на источник, наружу
отдаются только совпавшие компактные страницы. Ручки `/internal/wiki/*` под внутренним токеном.
**Это не «косвенные следы» — это описанный режим с параметрами; но кода я не видел.**

---

## 3. `tts_service` — поправка к аудиту

**В аудите ошибка: это не отдельный репозиторий.** Это каталог внутри `terra/terra-core`
(`repomix/terra-core.xml`): `tts_service/code/{tts.py,tts_engine.py,config.py,
language_detector.py,rabbitmq_consumer.py,rabbitmq_publisher.py,web.py}`, `Dockerfile_tts`,
`build_image_tts_service.sh`, плюс прототип `tests/tts-proto/` и `tests/tts-proto/task_prompt.txt`
— исходное ТЗ, по которому сервис писала LLM.

- **Модель — Silero TTS** (`snakers4/silero-models`), TorchScript-пакеты, грузятся через
  `torch.package.PackageImporter(model_path).load_pickle("tts_models","model")`.
  Открытые веса, on-prem, никакого облака.
- **Языки — 5:** ru, en, de, es, fr (`README` прототипа + `MODELS_CONFIG` в
  `scripts/download_models.py`, где на каждый язык свой `speaker`). Автоопределение языка —
  `fasttext-predict` (`fastdetect-model` и `fastdetect-model-small` в образе).
- **Клонирования тембра НЕТ.** В `tts_engine.py` только `text_to_speech(text, language,
  output_path)` и `model.save_wav(...)`; ни reference-аудио, ни speaker-эмбеддингов.
  Голоса — фиксированные спикеры Silero.
- **API:** FastAPI + gunicorn/uvicorn; `POST /tts` (`request_id`, текст **до 250 символов**,
  код языка), `GET /download/{request_id}`, `GET /status/{request_id}`; между вебом и
  процессором — RabbitMQ; инференс на CUDA (`TTS_DEVICE=cuda`), torch 2.6.0+cu118.
- **Потребитель ЕСТЬ, вопреки прежнему выводу.** В `terra-core/rapi/api.py` (публичное API
  Terra) есть прокси: `target_url = "http://terra_tts_worker:8000/tts"`. Коммиты:
  2025-10-29 «TTS done single sync endpoint /tts» → 2025-10-30 «integrated TTS service to API»
  + «fixed integration TerraAPI and TerraTTS» → 2025-12-16 «removing outdated sound files
  every 100 processed requests» → 2026-05-28 «TTS: update packages for avoid vulnerabilities».
  Автор всех — Валерий Большунов. Образ `terra_tts_worker:v5.1`, есть отчёт Trivy и
  зафиксированные версии pypi (`info/pypi-versions/`) — то есть сервис прошёл релизный конвейер.
- **Продуктового потребителя нет:** ни в `rn.xml`, ни в `iva-one` вызовов `/tts` не найдено.
  Готовый кирпич без витрины.

---

## 4. DeepSeek-шлюз — куда ходит и есть ли «сменные модели»

`DEPLOY.md` §6.8, скаффолд `/opt/ai-gateway/.env`:

```ini
AI_GATEWAY_DEEPSEEK_BASE_URL=https://api.deepseek.com
AI_GATEWAY_DEEPSEEK_MODEL=deepseek-v4-flash
AI_GATEWAY_DEEPSEEK_THINKING_MODEL=deepseek-v4-pro
AI_GATEWAY_DEEPSEEK_ALLOWED_MODELS=deepseek-v4-flash,deepseek-v4-pro
AI_GATEWAY_DEEPSEEK_DEFAULT_MODE=simple
AI_GATEWAY_DEEPSEEK_ALLOWED_MODES=simple,thinking
AI_GATEWAY_DEEPSEEK_THINKING_EFFORT=high
DEEPSEEK_API_KEY=...              # operator fills in
```

**Ходит НАРУЖУ, в облако DeepSeek** — `https://api.deepseek.com`, по ключу. Это прямой факт,
а не догадка (в аудите этот пункт был «смягчён до неизвестности» — теперь он закрыт).

**Абстракция «сменные модели» есть, но частичная и её надо назвать честно.** Что реально есть:
базовый URL, имена моделей, whitelist разрешённых моделей и whitelist режимов — всё
переменными окружения; ключ живёт только в `/opt/ai-gateway/.env` (chmod 600) и **никогда**
не попадает в Angular-конфиг, атрибуты RN-элементов, браузерное хранилище и JS-бандл.
Публичный `/healthz` намеренно урезан: «must not expose provider names, model names, upstream
URLs, collection names, or actor details». Чего НЕТ: интерфейса провайдера — все переменные
названы `DEEPSEEK_*`, то есть по именам это один провайдер с настраиваемым адресом, а не
пул провайдеров. Смена на локальный vLLM = подмена `AI_GATEWAY_DEEPSEEK_BASE_URL`, **если**
клиент внутри написан по OpenAI-совместимому контракту — а этого по коду не проверить,
исходников гейтвея в срезе нет.

Прочее по контуру: контейнер `iva-ai-gateway:current` (Node 22-alpine) на `127.0.0.1:8098`,
наружу `https://uc.iva.ru/api/assistant/v1/{healthz,recommendations,chat/respond}`; авторизация
— nginx пробрасывает `Authorization`, гейтвей валидирует против
`${AI_GATEWAY_IVA_ONE_BASE_URL}/api/v1/me`; ответ публикуется обратно в чат от имени бота
через `X-Iva-Bot-Api-Token` (без него ответ приходит в браузер с `botPost.persisted=false`).
Метрики `[ai-gateway:metric]` санитизированы: запрещено логировать промпты, тексты источников,
email участников, id чатов/писем и `reasoning_content`. В проде включено:
`RN_ASSISTANT_ENABLED=true`, бот `etdev@iva.ru` / `27ce3ec0-4421-4f42-b40b-c371575b280b`.

---

## 5. BPM-движок и маркетплейс

**BPM (`rn-shared/src/bpm/{types,gateway,httpGateway,mockGateway,index}.ts` + сервис
`iva-bpm-service:current` на `127.0.0.1:8096`, `/opt/bpm/docker-compose.yml`).** Модель
шире, чем считалось в первой разведке:
`BpmStepType` = `start|task|approval|condition|wait|notify|miniapp-action|create-task|
create-event|send-mail|open-chat|end`; `BpmTemplateStatus` = `draft|published|retired`;
`BpmInstanceStatus` = `active|blocked|completed|cancelled`; политики видимости
(`BpmVisibilityPolicy` — `all|groups|private` + `editableByRoles`), фазы с SLA
(`BpmWorkflowPhase.slaPolicyId`), артефакты коллаборации (`BpmCollaborationArtifact`).
**Новое, чего в аудите нет:** `BpmSemanticSource` с полями `embeddingStatus`
(`indexed|pending|stale|failed`), `embeddingModel`, `confidence` — то есть векторная память
спроектирована не только под ассистента, но и как слой поиска источников для процессов.
И `BpmExternalIntegration` с готовым `BpmJiraIntegrationConfig` (маппинг статусов и полей,
`syncDirection: bpm-to-jira|jira-to-bpm|bidirectional`, webhook-события).
API: `GET|POST /api/v1/bpm/{templates,instances,reports,signals}`, идемпотентность через
клиентский `requestId`. Хранилище — JSON-файл на томе `/data`. В проде включено:
`RN_BPM_ENABLED=true`, `RN_BPM_USE_MOCK=false`, доступ по группе `bpm`. Тесты: 4 e2e,
из них 3 `-real` (`bpm-case-status-real`, `bpm-collaboration-hub-real`, `bpm-real-cross`).
**LLM внутри нет** — это детерминированный движок; «агент» получится, если сверху поставить
выбор шагов моделью.

**Маркетплейс (`rn-shared/src/miniapps/*` + тот же `bpm-service`).** Модель каталога:
`MiniAppCatalogItem` (категория `hr|sales|finance|it|operations`, `runtime` `api|webview`,
`badges`, `favorite`, `recent`, `disabledReason`), права `MiniAppPermission`
(`read|write|approve|admin`), декларативные формы `MiniAppField`. API:
`GET /api/v1/miniapps/catalog` · `/manifest` · `POST /actions/<actionId>` ·
`/artifacts/.../file` (подписанный прокси документов).
**Главное, чего в аудите нет: у маркетплейса есть реальный первый коннектор к внешней
корпоративной системе — VK HR Tek.** `/opt/bpm/.env`:
`HR_TEK_BASE_URL=https://public-api.vkdoc.mail.ru/api/v1`, `HR_TEK_API_KEY`,
`HR_TEK_COMPANY_LEGAL_ID`, `HR_TEK_IDENTITY_CLAIM=snils`; сервис хранит маппинг
HR Tek `X-User-Id` ↔ актор ИВА. Дословно из `DEPLOY.md`: «Today: HR Tek employee
self-service». E2E против живого HR Tek: `miniapps-hr-tek-home-real`,
`miniapps-hr-tek-reports-real`, `miniapps-hr-tek-vacation-submit-liveness`,
плюс `calendar-vacation-miniapp-real`. **Это ровно тот класс работы («коннектор к внешней
системе»), который в концепте оценён в 25–40 ч/д как несделанный.**

---

## 6. Остальное выключенное — что нашёл и чего не нашёл

**Нашёл:**
- Три RAG-гейта и вики-гейт (§2) — единственные «выключено флагом» по ИИ.
- `RN_CHAT_FOLDERS_REMOTE_ENABLED=false` (папки чатов на сервере) — не ИИ, но выключено
  в обоих конфигах, prod и staging.
- `EVENT_TRANSCRIPT_SUMMARY` — системная настройка MCU «сводка стенограммы»: объявление
  в контракте есть, **потребителя в вебе нет ни одного** (подтверждаю прошлую находку).
- `OBJECT_RECOGNITION_ENABLED` / `_API_URL` / `_LOGIN` / `_PASSWORD` — контракт под внешний
  сервис распознавания объектов, использований в клиентах нет.
- `SPEECH_RECOGNITION_API_POST_PROCESS_URL` — спроектированная точка расширения под
  пост-обработку стенограммы, пустует.
- **Спроектированная и не построенная архитектура ИИ:** `AI-WR-001` предполагал
  `rn-shared/src/ai/` со скелетом `aiBridge.getCapabilities()` и гейтом на каждую фичу.
  **Каталога `rn-shared/src/ai/` в срезе нет вообще** — вместо capability-gate автор
  построил прямые HTTP-вызовы гейтвея из `rn-chat/src/assistant/` и `rn-mail/src/utils/`.
  То есть план апреля отменён самим ходом работы, а не заброшен.

**Не нашёл (и это тоже результат):** ни одного закомментированного `provider`/регистрации
сервиса и ни одного `TODO`/`FIXME`/`HACK` рядом с ИИ-словами — прогнал по всем 6 578 файлам
среза, 4 попадания и все ложные (совпадения внутри `WORKAROUNDS-REGISTRY.md` и в
`mpegts.js.map`). ИИ-код в срезе без «долговых» пометок.

---

## 7. Отменённые задачи по ИИ

По локальной выгрузке Jira (`jira_epics.csv`, 3 071 задача в `jira_issues.csv`) —
**три приостановленных эпика прямо по теме**, и это сильнее, чем отдельные задачи:

| Эпик | Имя | Статус | Обновлён |
|---|---|---|---|
| `IVAONE-1616` | **«Интеграция с AI»** (epic_name `AI`) | **Приостановлено** | 2025-09-03 |
| `IVAONE-9387` | **«БФТ: Чат боты»** | **Приостановлено** | 2026-03-11 |
| `IVAONE-3287` | «Поиск» (внутри — `IVAONE-1062` «Умный поиск во внешних ресурсах и приложениях») | **Приостановлено** | 2025-10-21 |

Рядом, для калибровки: приостановлены также `IVAONE-3075` «FEATURE TOGGLES»,
`IVAONE-4422` «Версионирование», `IVAONE-3774` «Федерация»; `IVAONE-6785` «IVA Room» —
`Rejected`. То есть паузы — регулярный организационный приём в этом трекере, а не
приговор технике.

**Чего подтвердить НЕ смог.** Ранее заявленные `IVAONE-1034`, `IVAONE-2733`, `IVAONE-2735`
(«чат-бот саммари и протоколов», формулировка отмены «Отменена ввиду смешения курса по
ботам», 29.11.2024) **в локальной выгрузке отсутствуют** — выгрузка это сэмпл, а MCP
`helm-analyst` мне недоступен. Их статус и причина остаются на прежнем источнике и
требуют перепроверки через MCP. Дочерних задач у `IVAONE-1616` в выгрузке тоже нет —
либо все закрыты, либо не попали в сэмпл.

**Читается так:** курс по ботам и по ИИ в One останавливали **организационно** (эпик
целиком на паузу, формулировка про «смешение курса»), а не потому, что уперлись в технику.
Это и есть искомый признак «работа частично сделана и брошена по решению, а не по
невозможности».

---

## КАНДИДАТЫ В ЗАЯВКУ

| Что | Готовность | Что мешает включить | Насколько честно заявлять как новое |
|---|---|---|---|
| **Векторная память с учётом прав** (Qdrant + bge-m3, `assistant_pointer_memory`, ACL-fingerprint, TTL 30 дн., память указателей) | **~80 %**: контур описан параметрами, деплой-профиль и смоуки есть; в живую не проверялось | 3 флага `false` + два незаполненных секрета (`QDRANT_API_KEY`, `INTERNAL_INDEX_TOKEN`); исходники гейтвея вне GitLab | **Честно как «доработка задела»**: заявлять сам факт векторного поиска с правами — правда; заявлять как *построенное в отчётном периоде* — нельзя, задел чужой и старше |
| **Маркетплейс мини-приложений + коннектор к VK HR Tek** (каталог, права, формы, подписанные артефакты, SNILS-идентификация) | **~85 %**: включено в проде (`RN_MINIAPPS_API_BASE`), 10 e2e, из них 3 против живого HR Tek | Ничего не мешает — работает; ограничение в том, что живёт в линии RN, а не в поставляемом `iva-one` | **Очень честно**: это законченная интеграция с внешней корпоративной системой, и именно она в концепте числилась несделанной |
| **BPM-движок** (12 типов шагов, SLA-фазы, политики видимости, идемпотентность, готовый конфиг интеграции с Jira) | **~80 %**: `RN_BPM_ENABLED=true`, mock выключен, 4 e2e | Хранилище — JSON-файл на томе, не БД; UI-модуль `rn-bpm` вне GitLab-среза | **Честно**, но не как ИИ: это детерминированный движок. Заявлять как «исполнитель действий, к которому достраивается ИИ-планировщик» |
| **Саммари письма + черновик задачи одним вызовом** (`mail-brief-with-task`, zod-валидация вывода модели, SSE, цитаты) | **~75 %**: E2E `mail-assistant-task-real` создаёт задачу и находит её в модуле задач | Зависит от внешнего DeepSeek и ключа; UI только в линии RN | **Честно как прототип с доказанным E2E**; «в продукте» — нет |
| **TTS на Silero** (5 языков, RabbitMQ, CUDA, образ `terra_tts_worker:v5.1`, прокси в публичном API Terra) | **~90 % как сервис, 0 % как фича**: релизный конвейер пройден, ручка в API Terra есть | Нет ни одного продуктового потребителя; лимит 250 символов на запрос; клонирования тембра нет | **Честно**: «синтез речи в контуре, готов к встраиванию». Заявлять голосового ассистента нельзя |
| **Два ML-инференса в браузере ежедневно** (DeepFilterNet3 ONNX/WASM, MediaPipe Selfie Segmentation TFLite) | **100 %, в проде**: `NOISE_SUPPRESSION_AVAILABLE=true`, `SHOW_VIRTUAL_BACKGROUND=true` | Ничего | **Абсолютно честно и проверяемо руками** — единственный ML, работающий локально у пользователя без внешних сервисов. Самый дешёвый пункт заявки |
| **ИИ-шлюз с изоляцией секретов и санитизацией телеметрии** (ключ только в `.env` 600, урезанный `/healthz`, запрет логировать промпты и `reasoning_content`) | **~70 %** | — | **Честно как «архитектура суверенного шлюза»**; но прямо сказать, что сегодня он ходит в облако `api.deepseek.com`, и суверенность — это переключение адреса, а не текущее состояние |
| **Приостановленные эпики `IVAONE-1616` (AI), `IVAONE-9387` (чат-боты)** | задел частичный | Организационное решение, не техника | **Честно как «возобновление работ по существующему заделу»** |

---

## ВЕРДИКТ

Самая ценная находка дня — не новая фича, а **правильная рамка**: `ione` это не «прототип,
который можно продуктивизировать», а **чужое дерево вне GitLab**, от которого нам виден
staged-срез без четырёх модулей и без исходников всех трёх Node-сервисов. Всё, что мы
называем «построено и выключено», доказано **деплой-конфигами и тестами, а не кодом**.
Для заявки это значит: заявлять надо по объектам, где доказательство самодостаточно —
маркетплейс с HR Tek, BPM, TTS в Terra, два браузерных инференса, — а RAG и шлюз подавать
как задел с явным условием «включается оператором».

**Проверено:** структура и авторство `rn/rn` (53 коммита, 8 контрибьюторов, 19 MR, 5 коммитов
EV — вываливания снимков, последний 22.05); отсутствие ИИ-коммитов у команды ИВА после 26.06;
205 e2e-файлов и их разбивка; параметры RAG/вики/DeepSeek целиком; состав и API `tts_service`
+ его прокси в `terra-core/rapi/api.py`; типы BPM и мини-приложений; коннектор HR Tek;
отсутствие `rn-shared/src/ai/`; отсутствие ИИ-долга в `TODO/FIXME`; три приостановленных
эпика в Jira.

**Данные:** `~/tacticum/helm/data/real/git/repomix/{rn,terra-core}.xml` (срез 2026-07-03,
`--compress`), `all_commits.csv`, `repo_activity_all.csv`, `repos_registry.csv`,
`repos_manifest.csv`, `git/README.md`, `identity/identity_gaps.csv`,
`jira/{jira_epics.csv,jira_issues.csv,ivaone/tasks_open_ivaone.csv}`.

**Подтверждение:** ключевые куски — дословные цитаты из `web/DEPLOY.md` §6.6/§6.8,
`web/tools/deploy/host-scripts/docker-compose.ai-gateway.yml`, `docs/WORKAROUNDS-REGISTRY.md`
(`AI-WR-001/002`), `web/apps/app/src/assets/config/config.prod.json`, `README.md` репозитория
`rn`, `terra-core/rapi/api.py`, `terra-core/tests/tts-proto/{README.md,task_prompt.txt}`.

**НЕ проверено:**
1. Работает ли что-либо из этого **сейчас на `uc.iva.ru`** — я в контур не ходил, всё по конфигам.
2. Исходники `ai-gateway`, `bpm-service`, `call-control-gateway` и модули `rn-communications`,
   `rn-calls`, `rn-miniapps`, `rn-bpm` — вне корпуса. Без них «есть абстракция сменных
   моделей» остаётся гипотезой по именам переменных.
3. `IVAONE-1034/2733/2735` и причина отмены 29.11.2024 — в локальной выгрузке Jira их нет,
   MCP `helm-analyst` недоступен. Нужен повторный заход с рабочим MCP.
4. Дочерние задачи эпика `IVAONE-1616` и точная причина его приостановки.
5. Правовой статус кода учредителя (чьи исключительные права) — вопрос не мой, но для
   заявки на льготу он первичен по отношению ко всей таблице кандидатов.

## Связано

[[audit-intellektualnyh-fich]] · [[recon-ai-one-2026-08-03]] · [[recon-ai-corpus-2026-08-03]] ·
[[explore-chto-uzhe-est-dlya-zayavki-2026-08-06]]