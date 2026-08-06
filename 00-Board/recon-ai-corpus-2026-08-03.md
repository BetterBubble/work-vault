---
title: Разведка интеллектуальных фич по всему корпусу репозиториев ИВА + Terra
type: note
status: draft
created: 2026-08-03
tags:
- recon
- ai
- ml
- llm
- terra
- adp
- gitlab
- corpus
- audit
permalink: tacticum/00-board/recon-ai-corpus-2026-08-03
---

# Разведка: интеллектуальные (AI/ML/LLM) фичи в корпусе ИВА

**Короткий вывод.** Гипотеза руководителя заказчика подтверждается частично и с важной поправкой.
В коде ИВА **есть много «умного», но почти всё это — речевые и сигнальные ML-технологии** (ASR,
диаризация, машинный перевод, TTS, нейрошумодав, сегментация фона), причём **зрелые, в проде и с
2020 года**. **LLM-часть существует, но она молодая, изолированная и в один автор**: это
`blank130/iva_adp` (сервер на vLLM с задачами assistant/summary/planner) и три пилота рядом.
**RAG и векторного поиска нет нигде** — ни в Terra, ни в корпусе коммитов: 0 упоминаний
qdrant/faiss/milvus/langchain, слово `RAG` в terra-core встречается ровно 2 раза и оба — токены в
словаре токенизатора. Terra — **не LLM-модуль**, а ASR/диаризация/перевод/TTS + **шлюз** к внешнему
ADP.

---

## 1. Что проверено и каким методом

| Источник | Объём | Что сделано |
|---|---|---|
| `/Users/bubblemac/tacticum/helm/data/real/git/all_commits.csv` | **216 161 коммит**, 233 репо, 2003-06…2026-07 | два потоковых прохода регулярками (en+ru, границы слов `\b`) |
| `/Users/bubblemac/tacticum/helm/data/real/git/repo_activity_all.csv` | 233 репо | сверка активности, контрибьюторов, MR |
| `/Users/bubblemac/tacticum/helm/data/real/git/merge_requests_all.csv` | 1115 MR | скан заголовков — **дал ~0 нового** (в файле лишь последние ~8 MR на репо) |
| `/Users/bubblemac/tacticum/helm/data/real/git/repomix/terra-core.xml` | 523 файла, 11 МБ | разбор дерева, чтение ключевых модулей, счётчики библиотек |
| `/Users/bubblemac/tacticum/helm/data/real/git/repomix/fedot_gardener.xml` | 1734 файла, 22 МБ | идентификация проекта |

**Проход 1 (широкий, 15 тем):** 3 346 сырых хитов из 216 161.
**Проход 2 (узкий, только однозначные ML-маркеры** — llm/vllm/gpt/openai/gigachat/whisper/vosk/
silero/wespeaker/pyannote/onnx/tflite/mediapipe/deepfilternet/rnnoise/diariz/asr/stt/tts/ocr/nlp/
rag/qdrant/faiss/embedding/neural/torch/inference/transcri/subtitle/speech/промпт/нейросет/диариз/
транскри/стенограмм/субтитр/шумоподавл/…**):** **622 хита в 44 репозиториях**.

**Что отсеяно и почему (это ~80 % сырых хитов):**

| Ложняк | Объём | Природа |
|---|---|---|
| `перевод`/`translation` | ~1 610 из 1 684 | локализация i18n, а также **перевод звонка** (call transfer) и **синхронный перевод живым переводчиком** (роль «переводчик», языковые дорожки — это НЕ машинный перевод) |
| `prompt` | 30 из 61 | OIDC-параметр `prompt=none/login/consent` в Keycloak (`iva/core/id/server`), SIP/POP3 prompt |
| `summary`/`digest` | 75 из 82 | `SUMMARY.adoc` в доках Keycloak, DIGEST-MD5-аутентификация, `Calendar: summary` в почте |
| `classif`/`cluster` | 55 из 60 | кластеризация серверов (Infinispan, Quartz), maven-classifier |
| `embed`/`retrieval` | 125 из 183 | embedded-сервер, `go:embed`, «retrieval» в смысле «получение данных» |
| `agent`/`bot`/`ml`/`ai` | ~50 | user-agent, gitlab-runner, `fine-tuning of layout`, `bottom` |
| `gpu`/`torch` | ~30 | GPU-рендеринг видео в qtwebengine/Chromium |

---

## 2. Таблица интеллектуальных репозиториев

### 2.1 Ядро: специализированные AI/ML-репозитории

| Репо | Что это на самом деле | Коммитов | Контриб. | Последняя активность | Статус | Доказательство |
|---|---|---:|---:|---|---|---|
| `terra/terra-core` | **ASR + диаризация + машинный перевод + TTS + шлюз к ADP.** Не LLM. | 1086 | 6 | 2026-06-19 | **живой**, trunk-only (0 MR) | см. §4 |
| `blank130/iva_adp` | **Сервер LLM-задач на vLLM**: OpenAI-совместимый `/v1/chat/completions`, задачи `assistant`/`summary`/`planner`/`diar_identification`, промпты в Jinja, guided decoding, redis, rabbit, prometheus | **28** | **2** | **2026-07-02** | **живой, но однонитевый** | `2026-04-30 · update vllm to version 0.19.0`; `2026-02-25 · add task diar_identification` |
| `blank130/iva-adp-client` | Поставочная сборка ADP клиенту (docker-compose, GPU), версии v2.0→**v5.0** | 12 | 2 | 2026-06-17 | живой | `2026-06-17 · add backend for structured output in llm engine` |
| `blank130/diarization_pipeline` | Диаризация + **идентификация говорящих через LLM** | **5** | 2 | **2026-07-03** | живой пилот | `2025-12-30 · add llm identification to pipeline`; `2026-06-23 · diar: relative min_cluster_size in cluster()` |
| `blank130/s2s_realtime_translator` | **Speech-to-speech перевод в реальном времени** (ASR→MT→TTS), eval-метрики | **5** | 2 | 2026-06-02 | ранний пилот | `2026-05-29 · add prompt pcm for tts module`; `2026-06-02 · add eval scripts; add metrics for v0.0.3` |

> **Критично:** все четыре `blank130/*` — **один автор, Сакевич Елена Витальевна** (второй
> контрибьютор — технический аккаунт инициализации). Ни одного MR, ни одного jira-ключа, ни одного
> ревью. Это персональная R&D-площадка, а не продуктовая линия. Bus factor = 1.

### 2.2 Речевой стек в проде (зрелый, кросс-продуктовый)

| Репо | Роль в интеллектуальном контуре | Коммитов | Контриб. | Посл. активность | Статус |
|---|---|---:|---:|---|---|
| `ivcs/ivcs-server` (MCU) | оркестратор транскрипции/субтитров, интеграция ASR-движков, постобработка Terra | 6921 | 19 | 2026-07-03 | живой |
| `modules/media` | **подключение ASR-движков к медиапотоку** (STC/ЦРТ, Yandex SpeechKit, IVA Terra), VAD | 3336 | 5 | 2026-06-29 | живой |
| `web/iva-connect` | UI субтитров, стенограммы, **интеллектуального шумоподавления**, синхроперевода | 6664 | 17 | 2026-07-03 | живой |
| `desktop/ivcs` | UI стенограммы/субтитров, **шумоподавление DeepFilterNet**, размытие фона | 22649 | 9 | 2026-07-03 | живой |
| `mobile/apple/messenger` | субтитры, синхроперевод, VirtualBackground | 10767 | 10 | 2026-07-03 | живой |
| `modules/diskstorage` | **Terra как S2T-сервис для файлов на диске** (IVADS-229/285) | 3688 | 4 | 2026-07-01 | живой |
| `web/iva-admin` + `fork/sbc_certified/iva-admin` | **админка Terra** (jira-проект IVATR), релизы TERRA 2.0→5.0 | 1842 | 15 | 2026-07-02 | живой |
| `iva/one/backend/{api-gateway,chats,proto,users}`, `iva/core/sdk-go` | транскрибация в One: фичетоглы, `GetSubtitles`, чат-бот API | — | — | 2026-07 | живой |
| `iva-m/android/kmp` | кросс-платформенная стенограмма/перевод в конференции | 1301 | 20 | 2026-07-04 | живой |

### 2.3 Компьютерное зрение и нейро-аудио (клиентские SDK)

| Репо | Что | Коммитов | Посл. | Статус |
|---|---|---:|---|---|
| `webrt-sdk/ivawebrtc` | RNNoise → **DeepFilterNet** нейрошумодав; виртуальный фон на MediaPipe | 857 | 2026-06-16 | живой |
| `webrt-sdk/iva-virtual-bg` | выделенный компонент virtual background: **tflite → onnx**, GPU-ускорение Windows | 42 | 2026-06-19 | **живой, активно пилится в 2026** |
| `ivasdk/webrtc-kmp` | **новьё: виртуальный фон на MediaPipe Selfie Segmentation** для Android/Web/iOS (Vision+CoreImage) | 252 | **2026-07-03** | живой, ветка `iva-fork` |
| `ivasdk/webrtc-java` | **`iva-ns`: порт-кит движков шумоподавления (DFN/RNNoise)** для Android-seam | 330 | **2026-07-03** | живой, ветка `iva-main` |

### 2.4 Проверенные кандидаты, оказавшиеся НЕ интеллектуальными

| Репо | Что на самом деле |
|---|---|
| `ivasw/audio/ru-default` | **набор WAV/MP3-файлов IVR-подсказок** (`add call beep audio`, `add speak-after-signal audio`). Ни грамма ML. 59 коммитов, jira SWL. |
| `iva/one/external/opensearch` | зеркало OpenSearch с одним `.gitlab-ci.yml`. **3 коммита, 1 автор, 2026-06-01.** Пустышка. |
| `modules/conversion` | конвертация документов через **LibreOffice** (с 2017). Не OCR, не ML. |
| `ivaqa/iva-ai-qa-assist` | QA-дашборд по тестам; «AI» в названии — потому что **написан агентом Codex** (ветки `codex/fix-dashboard-date-range`, `codex/manual-dashboard-ops`). Не ИИ-продукт. 13 коммитов, 2026-06-16…17, автор Брейкин Никита. |
| `fedot_gardener` (github, внешний) | **НЕ репозиторий ИВА.** `github.com/dsolonko/fedot_gardener` — LLM-приложение про уход за растениями: LangChain + **Gemini via Vertex AI** + DeepSeek + Langfuse-трейсинг, evaluation-датасеты. Внутри лежат артефакты `.tacticum/runs` (597 файлов) — следы **нашего** RE-пайплайна. К активам ИВА отношения не имеет; в `repos_registry.csv:2` помечен `external (внешний github)`. |

---

## 3. Хронология тем по коммитам

| Тема | Появилась | Последний коммит | Репозитории | Примеры subject |
|---|---|---|---|---|
| **Транскрипция конференций** | **2020-11-25** | 2026-03-24 | ivcs-server, modules/media | `[VCSWEB-2138] Initial support for conference transcribe`; `[VCSWEB-2139] Table for conference transcript` |
| **Внешние ASR-движки** | **2020-10-19** | 2026-04-20 | modules/media, ivcs-server | `Implement integration with speech recognition service`; `Introduce stc speech recognition engine` (2021-01-13); `Introduce yandex speech kit recognition engine` (2021-01-15); `Allow Yandex as speech recognition system` (2022-07-06) |
| **Свой движок IVA Terra как ASR** | **2022-12-06** | 2025-08-14 | modules/media, ivcs-server | `Add support of Iva Terra recognition engine` (modules/media, 2022-12-06); `Add support for Iva Terra speech recognition system` (ivcs-server, 2022-12-14); `Turn on dev IVA Terra as speech recognition engine by default` (2024-02-05) |
| **Стенограмма (UI)** | 2022-05-06 | 2026-06-25 | web/iva-connect, desktop/ivcs | `VCSWEB2-1299 Управление стенограммой`; `Добавлена бизнес-логика настройки мероприятия "стенограмма"` (2022-07-05) |
| **Субтитры в реальном времени** | **2024-10-11** | **2026-07-01** | iva-connect, ivcs-server, desktop, mobile, kmp | `VCSWEB2-5543 Субтитры в мероприятии`; `[VCSWEB-5118] Subtitles in conference session`; `[VCSWEB-7493] Support for additional subtitles languages` (2026-03-24) |
| **Машинный перевод стенограмм** | **2024-12-02** | 2026-04-20 | ivcs-server, iva-connect, terra-core | `[VCSWEB-5917] Translation for transcription post processed files`; `TRANSCRIPT_TRANSLATION system setting`; `Терра: поддержка перевода на китайский и арабский языки` (2025-07-11); `feat: VCSWEB2-9906 Поддержка новых языков распознавания речи` (2026-04-20) |
| **Нейрошумодав** | **2023-02-07** (RNNoise) → **2024-02-19** (DeepFilterNet) | **2026-07-03** | ivawebrtc, desktop/ivcs, iva-connect, webrtc-java | `jira/imp-1206, use RNNoise`; `jira/imp-1491, add deepfilternet noise suppressor`; `Шумоподавление DeepFilterNet` (desktop, 2024-03-06); `VCSWEB2-9305 … Интеллектуального шумоподавления`; `iva-ns: NS engine port kit (DFN/RNNoise sources) for B2/C2` (2026-07-03) |
| **Виртуальный фон (сегментация)** | **2021-10-12** | **2026-07-03** | ivawebrtc, iva-virtual-bg, webrtc-kmp | `jira/imp-844, add virtual background overlay`; `jira/imp-976, Virtual background. Use mediapipe.`; `jira/imp-2002: moved from tflite model to onnx` (2026-03-06); `Camera virtual background: BlurVideoProcessor (MediaPipe selfie-segmentation)` (2026-07-02) |
| **ADP (LLM-сервисы) — первые следы** | **2024-02-13** | 2026-07-02 | ivcs-server, terra-core, blank130 | `[VCSWEB-4980] Add test adp bot` (MCU, 2024-02-13); `ADP Client init commit` (terra, 2024-07-23); `implemented question to ADP/summary and response handling` (2024-07-29) |
| **ADP API GATE в Terra** | **2025-11-10** | 2025-12-12 | terra-core | `implemented real adp_handler in adp_processing.py`; `IVATR-158: ADP API GATE implemented /adp_services_stream` (2025-12-05) |
| **Собственный LLM-сервер (vLLM)** | **2024-07-01** (init) → рабочий **2025-08-22** | **2026-07-02** | blank130/iva_adp | `add prompts in jinja format`; `update vllm to version 0.19.0`; `add render to multimodal format; correct parse summary` |
| **Диаризация** | **2025-12-08** (пилот) → **2026-02-17** (в Terra) | 2026-07-03 | diarization_pipeline, terra-core | `offline_worker: added speaker diarization` (terra, 2026-02-17); `Using transcriber vad_segments for diarization` (2026-02-26) |
| **S2S realtime перевод** | **2026-05-07** | 2026-06-02 | s2s_realtime_translator | самый молодой пилот |
| **AI-ревью кода в CI** (внутренний инструмент, не продукт) | **2025-12-24** | **2026-06-10** | iva-m/devops/gitlab, iva/one/backend/*, ivcs-server, mobile/ucim-android, ios/messenger, helm-charts | `feat(ci): add review prompts`; `feat: add ai reviewer` (Медведев Василий, 2025-12-25); `Support new scheme of AI review` (ivcs-server, 2026-05-29); `IVAONE-0 Добавлены правила для ИИ` (2026-06-10) |
| **AGENTS.md / скиллы для AI-агентов** | 2026-02-04 | 2026-07-02 | docs/docs-one, iva-m/android/kmp | `feat: add agents.md`; `Скиллы для AI` (2026-06-17); `feat(arch-guard): checkNoBlankValueObjects + AGENTS.md правила против LLM-дрейфа` (2026-07-02) |
| **GigaChat** | 2025-10-30 | 2025-10-30 | autotest/mcu-ui/services/main | `Add GigaChat integration with new routes and services` — **ровно один коммит, один автор (a.rashchupkin), продолжения нет** |
| **Чат-бот-платформа** (интеграционная поверхность, не ИИ) | 2024-01-24 | 2025-12-25 | ivcs-server, iva-admin, One backend, mobile, desktop | `[VCSWEB-4970] The chatbot avatar is not sent to the chat`; `IVAADM-64 Создание/редактирование чат-бота`; токены, вебхуки, `externalUrl` |
| **OpenSearch / Elastic** (лексический поиск, не семантический) | 2023-12-13 | 2026-06-09 | one/backend/{chats,users,address-book}, helm-charts | `feat: add elasticsearch library`; `IVAMSG2-2537 switch to opensearch` (2024-09-02); `add addressbook opensearch` (2026-01-28) |

### Что живо прямо сейчас (коммиты 2026-06…07)
Субтитры/стенограмма (iva-connect, kmp, ivaplugins), виртуальный фон (webrtc-kmp, iva-virtual-bg),
шумодав (webrtc-java), ADP (iva_adp, iva-adp-client), диаризация (diarization_pipeline), AI-ревью в CI.

### Что мертво / заглохло
GigaChat (1 коммит, 2025-10-30). `iva/one/external/opensearch` (3 коммита, 2026-06-01).
`ivaqa/iva-ai-qa-assist` (2 дня в июне 2026). Terra последний коммит **2026-06-19** — темп упал:
в 2024 году было ~600 коммитов, за весь 2026 — заметно меньше сотни.

---

## 4. Terra: что это на самом деле

**Итог: Terra — это ASR-платформа с диаризацией, машинным переводом и синтезом речи, плюс
HTTP-шлюз к внешнему LLM-сервису ADP. Сама Terra модели-LLM не запускает и RAG не содержит.**

Формулировка из презентации «ИИ-модуль агентного контура» коду не соответствует: агентов в Terra
нет, а «RAG» из фразы на созвоне («у тебя есть гемма, есть эра, есть RAG») в коде отсутствует.

### Архитектура (523 файла, Python, `repomix/terra-core.xml`)

| Компонент | Путь | Что делает |
|---|---|---|
| **terradyne** — собственный ASR-движок | `rapi/terradyne/{asr,transcribe,alignment,vad,tokenizer,feature_extractor,audio}.py` | форк стека faster-whisper/WhisperX: **CTranslate2** + модели **Whisper** + **silero-vad 6.2.0** + wav2vec-элайнер по языкам. `rapi/terradyne/asr.py` — класс `TerradyneModel(FTerraTerradyneModel)`, батчинг, `initial_prompt`, детект языка с порогом |
| **Диаризация** | `rapi/diarizator/diar_pipeline.py` + `rapi/diarizator/wespeaker/**` (~70 файлов) | вендоренный **WeSpeaker**: speaker-эмбеддинги, `diar/spectral_clusterer.py`, `diar/umap_clusterer.py`, PLDA, ONNX-инференс. Появилось в проде **2026-02-17** |
| **Переводчик** | `translator_service/{translator_engine,model_adapter_impl,language_detector}.py` | **Helsinki-NLP/opus-mt (MarianMT)**, веса лежат в репо: `translator_service/pretrained_en_ru/config.json` → `"_name_or_path": "Helsinki-NLP/opus-mt-en-ru"`, `"architectures": ["MarianMTModel"]`. Детект языка на **fasttext**. 53 языка (`2026-02-05`) |
| **TTS** | `tts_service/code/{tts_engine,tts,language_detector}.py` | синтез речи, отдельный воркер, с 2025-10-29 |
| **ADP API GATE** | `rapi/adp_processing.py` | **прокси**, а не LLM. Валидирует `{service: "assistant/summary/planner", task: "request/response", message, chat_id, task_id}` и проксирует на `ADP_API_GATE_QUESTION_URL` / `ADP_API_GATE_ANSWER_URL`. Есть SSE-стриминг (`process_adp_stream`, `BackendStreamResponse`) |
| **ADP Client** | `adp_client/adp_client.py`, `adp_client/api.py` | эндпоинт `/extend_minutes` — «обогащает» протокол встречи ответами ADP |
| **Оркестрация** | `rapi/{inference_model,minutes_worker1,processing,online_model_adapter}.py`, `settings_holder/**` | offline/online воркеры, RabbitMQ, Redis, Postgres, лицензирование (`terra_license.py`, `license_dispatcher.py`), шифрование протоколов (Fernet) |
| **Админка** | `spectacular/**` (159 файлов) | Django + DRF-spectacular: аккаунты, клиенты, **лицензии**, Swagger |

### Доказательства отсутствия RAG/векторов в Terra

Счётчики по `terra-core.xml`: `qdrant` **0**, `faiss` **0**, `milvus` **0**, `langchain` **0**,
`vllm` **0**, `gpt` **0**, `llama` **0**, `gigaam` **0**, `vosk` **0**.
`\bRAG\b` — **2 попадания, оба вида `"rag": 21796` в JSON-словаре токенизатора** (строки 78415 и
141070). `embedding` (86) — **исключительно speaker-эмбеддинги WeSpeaker**
(`utils/embedding_processing.py`, `diar/extract_emb.py`), не текстовые. `OpenAI(` — **1 раз**, в
тестовом файле `tests/adp_client/*` для дёрганья ADP по OpenAI-совместимому протоколу, причём
вызов **закомментирован** (`# call_chat_completions()`).

### Где живёт настоящий LLM
В `blank130/iva_adp` — **vLLM 0.19.0**, OpenAI-совместимый `/v1/chat/completions`, `history_mode:
server|client` (своя БД истории чата), guided decoding / structured output, Jinja-промпты,
chunking-стратегии для длинных текстов, задачи `assistant` / `summary` / `planner` /
`diar_identification`. **Названия конкретных весов модели в данных не встречаются ни разу** —
модель подаётся через конфиг, в коммитах не зафиксирована. «Гемма» из фразы руководителя в
216 161 коммите **не встречается ни разу** (`gemma`/`гемма` — 0 хитов).

### Кто пишет Terra
`Большунов Валерий` (697) / `Valery Bolshunov` (400) — **это один человек с двумя написаниями,
т.е. ~1097 из 1086 коммитов default-ветки**. Плюс `art` (2). **Bus factor Terra = 1.**
0 merge requests за всю историю — разработка ведётся прямо в trunk без ревью.

---

## 5. Jira-проекты, связанные с ИИ (точки входа в трекер)

В 216 161 коммите — 133 разных jira-префикса. Релевантные:

| Префикс | Коммитов | Диапазон | Где | Что там про ИИ |
|---|---:|---|---|---|
| **IVATR** | **254** | 2024-05-21…2026-03-19 | `web/iva-admin` (125), `fork/sbc_certified/iva-admin` (123), `terra/terra-core` (6) | **Собственный проект IVA Terra.** Первый тикет IVATR-5 (init, 2024-05-21). Живые: IVATR-133 «Завершить сбор данных / Повторная обработка в Информации о транскрибации», IVATR-148 (cleanup), IVATR-152 (cuda cache), **IVATR-158 «ADP API GATE /adp_services_stream»** |
| **VCSWEB** | 5365 | 2017-06…2026-07 | `ivcs/ivcs-server` | **бэкенд транскрипции**: VCSWEB-2138/2139/2141/2142/2192 (2020-11…12, запуск), VCSWEB-5118 (субтитры), **VCSWEB-5917 (перевод стенограмм)**, VCSWEB-6316 (ограничение функционала Terra), VCSWEB-7493 (языки субтитров), **VCSWEB-4980 (ADP-бот)** |
| **VCSWEB2** | 8308 | 2020-10…2026-07 | `web/iva-connect`, `web/iva-core` | **фронт**: VCSWEB2-5543 (субтитры), VCSWEB2-4169 (шумоподавление), VCSWEB2-7415/7516 (**RnD: переключение моделей шумоподавления**), VCSWEB2-7715 (Терра: китайский и арабский), VCSWEB2-9906 (новые языки распознавания), VCSWEB2-6342 (синхроперевод) |
| **VCSDESK** | 8157 | 2022-02…2026-07 | `desktop/ivcs` | VCSDESK-4573 (индикаторы транскрипции), VCSDESK-6483 (размытие фона), VCSDESK-7170 (переводы субтитров) |
| **IVADS** | **3474** | 2025-02…2026-07 | `modules/diskstorage` | **IVADS-229 «Add Terra as S2T service»** (2025-06-17) — Terra как движок расшифровки файлов на диске; IVADS-285 (задачи transcribe), IVADS-338/414 (multipart в Terra), IVADS-71 (conferenceId в Terra) |
| **IVAONE** | 7159 | 2025-01…2026-07 | One (backend/web/desktop/mobile) | IVAONE-5215 (**фичетогл отключения транскрибации**), IVAONE-10044 (hide transcription), IVAONE-11948 (ошибка транскрибации), IVAONE-8766 (Terra в HELM), IVAONE-10625 (OpenSearch TLS), IVAONE-0 (правила для ИИ) |
| **DOCS** | 807 | 2025-02…2026-07 | `docs/docs-terra`, `docs/docs-playbook` | **DOCS-1078 «Конвертация IVA Terra»**, DOCS-1386 (Terra 3.0), DOCS-1494 (Terra 4.0), **DOCS-1551 (Terra 5.0)**, DOCS-1633 (Terra 5.0 offline), DOCS-1399/1424 (публикация Terra на сайт и снятие с прода) |
| **IMP** | (в тексте, без префикса-ключа) | 2019…2026 | `webrt-sdk/*` | у webrt-sdk своя нумерация вида `jira/imp-NNNN`: imp-844/976 (virtual bg), imp-1206 (RNNoise), imp-1491 (DeepFilterNet), **imp-2002/2003/2004/2027/2031 (onnx, GPU, оптимизация — 2026)** |
| **VCSMOB** | 6463 | 2017-08…2026-07 | mobile | чат-бот-клавиатура, субтитры, синхроперевод |
| **SWL / SWM** | 16230 / 5124 | 2020…2026 | `ivasw/**` | **ИИ там нет.** `assistant` в SWL — это **телефонный сервис «ассистент»** рядом с voicemail/IVR (SWL-996 `added service assistant model`, SWL-2135 `ivr_item notify`), не ИИ |

**Рекомендуемые точки входа в трекер:** `IVATR` (Terra целиком), `IVADS-229` (Terra↔диск),
`VCSWEB-5917` + `VCSWEB2-7715` (машинный перевод), `VCSWEB2-7415/7516` (RnD шумодава),
`VCSWEB-4980` (ADP-бот в MCU), `IVAONE-5215` (управление транскрибацией), `DOCS-1078/1551` (доки Terra).

---

## 6. Чего в данных не видно и где доискать

1. **Сам ADP-сервис (LLM-бэкенд) в выгрузке отсутствует как код.** Есть только клиент
   (`blank130/iva-adp-client`) и шлюз (`terra-core/rapi/adp_processing.py`). Репозитория с
   реализацией самого движка ADP в списке 233 активных нет — либо он внутри `blank130/iva_adp`
   целиком (28 коммитов на такой объём функциональности выглядит подозрительно мало → вероятно
   squash-коммиты «Major update with extensive changes across multiple modules»), либо живёт вне
   GitLab. **Проверять:** repomix-снимок `blank130/iva_adp` (его нет — снято только 9 реп).
2. **Какая LLM-модель реально крутится.** В 216k коммитов нет ни одного имени весов. Ответ — в
   `.env`/конфигах ADP на стендах, в docker-compose `iva-adp-client`, либо у Сакевич Е. В.
3. **7 репозиториев с ошибкой доступа** при выгрузке (`README-gitapi.md`: «475 с активностью…
   233 с коммитами… 7 ошибок доступа»). Что там — неизвестно. Плюс **235 проектов «активны лишь по
   issue/MR без коммитов»** — среди них могут быть трекерные проекты ИИ-инициатив без кода.
4. **Только default-ветки.** `all_commits.csv` — это default-branch. У `desktop/ivcs` 5556 веток, у
   `ivcs-server` 968. Незамерженный ИИ-эксперимент в фиче-ветке в данные **не попал**.
5. **Только subject коммита.** Тело коммита, диффы и содержимое файлов для 224 из 233 репо
   недоступны — repomix снят лишь по 9 репозиториям. Вывод «в репо X нет ИИ» опирается на тексты
   коммитов, а не на код. Для `blank130/*` кода нет вообще.
6. **Группа `blank*`** (`blank130`, `blank150`, `blank23`, `blank60`) — судя по всему, персональные
   песочницы (`blank130` = Сакевич). Стоит выяснить, кто такие `blank150`, `blank23`, `blank60`, и
   нет ли ещё `blankNNN` среди 235 отфильтрованных «активных без коммитов».
7. **Спорная зона «синхронный перевод».** В `web/iva-connect`/`web/iva-core`/`mobile` это фича с
   ролью **живого переводчика** и языковыми дорожками (VCSWEB2-1074 с 2022-03), а не машинный
   перевод. Но рядом есть VCSWEB2-5874 «API URL перевода текста» — то есть машинный контур туда же
   заведён. Где проходит граница — по коммитам не восстановить, нужен трекер или разговор с командой.
8. **`docs/docs-terra`** (35 коммитов, 4 автора, до 2026-05-29) — единственная документация продукта
   Terra в корпусе. Это самый быстрый способ узнать заявленную функциональность Terra 5.0 и сверить
   с кодом; **DOCS-1633 «IVA Terra 5.0 mode offline»** был ещё открыт на момент выгрузки.