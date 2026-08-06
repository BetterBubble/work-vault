---
title: Инвентарь Terra (ИИ-ядро ИВА) — что реализовано, что готово и не подключено
type: report
status: draft
created: 2026-08-06
tags:
- board
- инновации
- terra
- explore
permalink: tacticum/00-board/deep-terra-inventar-2026-08-06
---

# Terra: полный инвентарь под заявку на льготу (ИНТЦ)

**Метод и оговорка.** Снимок `repomix/terra-core.xml` (523 файла, 11 МБ, срез до 2026-06-19)
сделан с `--compress`: тела функций и значения словарей местами вырезаны. **Найденное доказывает
наличие; ненайденное отсутствия НЕ доказывает.** Serena по этому репозиторию не поднималась —
рабочей копии кода на машине нет, есть только XML-снимок, поэтому разведка шла текстовым поиском
по снимку (это ограничение метода, не выбор). MCP `helm-analyst` в моём наборе инструментов **не
доступен** — Jira и Confluence читал из локальных выгрузок `~/tacticum/helm/data/real/{jira,confluence}`.

Ссылки вида `terra-core.xml:NNNNN` — номер строки в снимке, путь файла указан рядом.

---

## 1. Модели и движки внутри Terra

| Что | Модель/движок | Где в коде | Настройки |
|---|---|---|---|
| **ASR** | собственный форк faster-whisper/WhisperX: **CTranslate2 + Whisper** | `rapi/terradyne/asr.py` (класс `TerradyneModel(FTerraTerradyneModel)`, `terra-core.xml:21489`), `transcribe.py` | `MODELS_LOCATION`, `CUDA_ID`, `CPUONLY`, `AVAILABLE_COMPUTE_TYPES`, батчинг, `initial_prompt` |
| **VAD** | **silero-vad** (`load_vad_model`, `terra-core.xml:21673`) | `rapi/terradyne/vad.py` | `VAD_ON`, `VAD_OFF`, `SPEECH_CHUNK_SIZE`, `SPEECH_DURATION_THRESHOLD`, `SPEECH_SPLIT_PAUSE` (`rapi/online_model_adapter.py`, `terra-core.xml:24305-24307`) |
| **Выравнивание слов (word-level timestamps)** | **wav2vec2** по языкам, torchaudio-bundles + HF-модели | `rapi/terradyne/alignment.py:21162-21198` — `ALIGNER_LOCATION='/rapi/models/aligners/'`, `DEFAULT_ALIGN_MODELS_TORCH`, `DEFAULT_ALIGN_MODELS_HF`, `LANGUAGES_WITHOUT_SPACES=["ja","zh"]` | флаг **`USE_ALIGNER`** |
| **Диаризация** | **WeSpeaker**, веса `voxblink2_samresnet100` | `rapi/diarizator/config.py` (`terra-core.xml:20965`): `WESPEAKER_MODEL_DIR="/rapi/models/voxblink2_samresnet100"`, `DEVICE="cuda"`, `COMPUTE_TYPE="float16"`; пайплайн `diar_pipeline.py`, ~70 файлов `wespeaker/**` (spectral + **umap** кластеризация, PLDA, ONNX-экспорт) | `WESPEAKER_OPTIONS` |
| **Перевод** | **два семейства сразу**: `MBartAdapter` (**mBART-50**, карта на **53 языка**, `translator_service/model_adapter_impl.py:181965-182024`) и `HelsinkiAdapter_RU_EN`/`_EN_RU` (**MarianMT / Helsinki-NLP opus-mt**, веса в репо `translator_service/pretrained_{ru_en,en_ru}/`) | `MODEL_ADAPTER_MAPPING` (`:182046`), `NAMED_HANDLER` (`:182344`) | `TRANSLATOR_LANGS`, `AVAIL_MODES`/`CURRENT_MODE`, `MODELS_LOCATION`, `PROXY_LANG="en"` + `ROUTES_WITH_PROXY` — **перевод «любой→любой» через английский pivot** (`settings.py:182327-182331`) |
| **Определение языка** | **fasttext** lid.176 (две модели: `lid.176.ftz` малая / `lid.176.bin` большая) | `translator_service/language_detector.py:181874-181875` | `USE_BIG_MODEL` |
| **TTS** | **Silero TTS** (TorchScript-пакеты `torch.package`) | `tts_service/code/tts_engine.py:182716-182722` | `TTS_DEVICE` (cuda/cpu), `MODELS_DIR`, `clean_threshold: 100` |
| **LLM** | **своего нет**, Terra — HTTP-прокси к внешнему ADP (`blank130/iva_adp`, vLLM 0.19) | `rapi/adp_processing.py` | см. §3 |

Постобработка стенограммы: **словарь регулярок-«мусорных фраз»** `rapi/cleaning_patterns.txt`
(`terra-core.xml:23131-23149` — вычищает «Субтитры … DimaTorzok», «Продолжение следует» и т.п.
галлюцинации Whisper) + параметр `CLEANING_PATTERNS`. Есть корпус hot-words для промпта
(`tests/HotWords/transcribe.txt`).

## 2. TTS: подтверждено частично, тезис «ни к чему не подключён» надо уточнить

**Что есть.** Отдельный микросервис: FastAPI `POST /tts` + RabbitMQ-очередь + воркер на Silero
(`tts_service/code/{web,tts,tts_engine}.py`). Язык **определяется автоматически** fasttext-ом
(`web.py:182914`), лимит текста — **250 символов** (`web.py:182876`), ответ — WAV.
Прототип: `tests/tts-proto/README.md:56223-56229` — языки **ru, en, de, es, fr**; эндпоинты
`POST /tts`, `GET /download/{request_id}`, `GET /status/{request_id}`; GPU-докер (`runtime: nvidia`).

**Статус зрелости — выше, чем «прототип в тестах».** Jira **IVATR-145 «Разработка Text-To-Speech
сервиса» — Закрыт (2025-11-19)**. Коммиты `terra-core`: `Stable Text-To-Speech service` (2025-10-28),
**`integrated TTS service to API` (2025-10-30)**, `fixed integration TerraAPI and TerraTTS` (2025-10-30),
`Created image terra_tts_worker:v3.0` (2025-10-31), поддержка живёт до 2026-05-28. В релизе **v5.1
образ собирается и сканируется**: `info/pypi-versions/terra_tts_worker:v5.1.txt`,
`info/trivy-reports/report_terra_tts_worker:v5.1.*`, `Dockerfile_tts`, `build_image_tts_service.sh`.
Есть страница Confluence **`160711129` «IVA Terra API /tts — Преобразование текста в звук речи»**.

**Чего нет.** **Ни одного потребителя за пределами Terra**: в 216 161 коммите слово `tts` вне
`terra-core` встречается только у `blank130/s2s_realtime_translator`; в `ivcs-server`, One, мобильных —
ноль. Клонирования тембра / few-shot voice cloning в Terra **нет**: Silero — фиксированные дикторы,
голос выбирается как `speaker` при скачивании модели (`scripts/download_models.py:182981`,
`MODELS_CONFIG`). Конкретные имена голосов в снимке вырезаны компрессией.
**Единственный след voice-prompt** — коммит s2s от 2026-05-29 «add **prompt pcm** for tts module»
(другой репозиторий, снимка нет) — это похоже на zero-shot TTS с референс-аудио, но **не проверено**.

## 3. Задачи LLM-сервера ADP: кто есть и у кого есть потребитель

| Задача | Что делает | Где вызывается | Потребитель |
|---|---|---|---|
| **`summary`** | краткое изложение встречи | `adp_client/adp_client.py` класс `ADPSummary` (`terra-core.xml:909`); вставляется в протокол репликой участника **ID=11111111** (`:975`) | **ЕСТЬ** — протокол встречи, гейт `SUMMARY_PARTICIPANT_ID` (пустой ID = запросы не шлются, `:960`) |
| **`planner`** | поручения «что / срок / ответственный» (XML `tasks/task/description|deadline|responsible`, `:830`) | `ADPPlanner` (`:903`), реплика участника **ID=00000000** (`:973`) | **ЕСТЬ**, гейт `PLANNER_PARTICIPANT_ID` |
| **`diar_identification`** | по стенограмме определяет, кто такие `voice_N`, и **заменяет имена спикеров на реальные ФИО** (`:978-998`), сохраняя оригинал в `orig_participants` | `ADPDiarization` (`:911`), включается только если в протоколе есть метки `voice_` (`compute_voice_identification`, `:923`) | **ЕСТЬ**, но узкий: Jira IVATR-165 (закрыт 2026-03-04), артефакт `ADP_VOICES_LIST` в метриках для правки имён в админке |
| **`assistant`** | диалоговый ассистент со стримингом, история на сервере или на клиенте (`history_mode: server\|client`) | только через **ADP API GATE** `/adp_services`, `/adp_services_stream` (`rapi/adp_processing.py`; примеры `tests/adp_client/send_post_adp_assistant.sh:31943`) | **НЕТ продуктового потребителя.** UI не сделан: **IVATR-149 «IVA GPT WEB компонент» — Открыто**, **IVATR-151 «IVA GPT Bot развертывание» — Открыто** |
| **`code_docs` (RAG!)** | вопросно-ответная система по документам | в terra-core нет (живёт в ADP) | **НЕТ.** Но: **IVATR-163 «RAG: реализовать пайплайн вопросно-ответной системы по документам» — Закрыт (2026-02-13)**, **IVATR-164 «RAG: переход со статического RAG на 3 tools для задачи code_docs» — Закрыт (2026-06-11)**, IVATR-166 «RAG: тестовый датасет для retrieval+generation» — Ready for Development. Прежний вывод «RAG нет нигде» верен только для кода Terra |

Контракт гейта (`rapi/adp_processing.py:22833-22855`): `{service, task: request|response, message,
chat_id, task_id, lang}`; список разрешённых сервисов — **параметр `ADP_API_GATE_AVAILABLE_SERVICES`**;
SSE-стрим с заголовками `X-Inference-ID`, `X-Chat-ID` (`tests/adp_client/terra_adp_api_v2_test.py:32019`).
У ADP есть и **OpenAI-совместимый** `/v1/chat/completions` (вызов в тесте закомментирован, `:32028`).

## 4. Речь-в-речь (S2S)

Прототип **есть, отдельным репозиторием** `blank130/s2s_realtime_translator` (5 коммитов,
2026-05-07…2026-06-02, автор Сакевич Елена; снимка кода нет — только реестр и коммиты):
`init project` (2026-05-21) → `update la-2; correct commiter; off agc in frontend` (2026-05-25) →
`add new langs; correct replay for frontend; add timestamps for trim; **add prompt pcm for tts module**`
(2026-05-29) → **`add eval scripts; add metrics for v0.0.3`** (2026-06-02). Есть фронтенд, «коммиттер»
(логика решения, когда фиксировать сегмент — типовой узел для потоковой задержки), тримминг по таймштампам.
Jira: **IVATR-179 «Исследование и оптимизация S2S real time translation pipeline» — Закрыт (2026-06-03)**,
**IVATR-184 «s2s: оптимизация модуля tts в пайплайне (стриминговый tts)» — Закрыт (2026-06-23)**.
**Конкретных чисел задержки я не нашёл** — метрики есть в репозитории s2s (`eval scripts`, v0.0.3), но
снимка кода нет и MCP-доступа к Confluence у меня не было. Это дыра, её закрывает выгрузка репо s2s.

## 5. Пустующие точки расширения (спроектировано, не занято)

- **`ADP_API_GATE_AVAILABLE_SERVICES`** — белый список сервисов гейта. Новая LLM-задача подключается
  строкой в конфиге, без кода в Terra (`rapi/adp_processing.py:22828`).
- **`upstream_url` в `prepare_request`** (`:22857`) — гейт умеет проксировать на **произвольный
  внешний URL**, а не только на ADP. Готовый механизм подключения любой сторонней модели.
- **`SPEECH_RECOGNITION_API_POST_PROCESS_URL`** на стороне MCU (`ivcs-server`, Jira VCSWEB-4688,
  пример значения `https://terra.iva.ru/minutes`) — плагинная точка постобработки стенограммы.
- Полный список ключей `PARAMS[...]`, найденных в снимке: `ADP_API_GATE_{ANSWER,QUESTION}_URL`,
  `ADP_API_GATE_AVAILABLE_SERVICES`, `ADP_CLIENT_URL`, `ANSWER_ATTEMPT`, `ANSWER_TIMEOUT`,
  `AVAIL_MODES`, `AVAILABLE_COMPUTE_TYPES`, `CLEANING_PATTERNS`, `COMBINE_SOUND_BY_PARTICIPANT`,
  `CPUONLY`, `CUDA_ID`, `CURRENT_MODE`, `FERNET_KEY_FILEPATH`, `FORCE_EXTEND_MINUTES`,
  `MAX_PROCESSING_TIME`, `MODELS_LOCATION`, `OFFLINE_LANG_TO_TRANSLATE`,
  `ONLINE_{DB_URL,MAX_SEC,STATES_PATH,STORE_TYPE,TIMEOUT}`, `PLANNER_PARTICIPANT_ID`, `PROMPT`,
  `SECONDS_IN_QUESTION`, `SHOW_NATIVE_TEXT`, `SUMMARY_PARTICIPANT_ID`, `TRANSLATE_REQ_QUEUE_NAME`,
  `TRANSLATOR_TIMEOUT`, `TWICE_TEXT_IN_DOUBLE_SOUND`, `USE_ALIGNER`, `VAD_ON`, `VAD_OFF`,
  `XML_ARCH_PATH`, `RAW_AUDIO_ARCH_PATH`.
- **«Сменные модели» перевода уже механизм, а не хардкод**: `MODEL_ADAPTER_MAPPING` + `NAMED_HANDLER`
  + `AVAIL_MODES`/`CURRENT_MODE` — три режима загрузки моделей и распределение направлений перевода
  между инстансами (`translator_service/settings.py:182332-182344`). Добавить новую модель = адаптер + конфиг.
- **Шаблоны протоколов**: IVATR-167 «Выбор шаблона для формирования протокола стенограммы» —
  Ready for Development; Confluence `195432645` «Управление шаблонами отчетов», `195432647` «Проект
  порядка управления шаблонами». Спроектировано, не реализовано.

## 6. Зрение/видео внутри Terra — НЕТ

Счётчики по снимку: `mediapipe` **0**, `opencv`/`cv2` **0**, `yolo` **0**, `face`/`retinaface`/
`insightface` **0**, `emotion` **0**. Единственное — `onnxruntime` (7 хитов), и он используется
для speaker-эмбеддингов WeSpeaker, не для CV. CV в корпусе ИВА живёт **вне Terra**: виртуальный фон
(`ivawebrtc`, `iva-virtual-bg`, MediaPipe selfie-segmentation → ONNX, jira/imp-2002, живо в 2026-07)
и нейрошумодав (DeepFilterNet/RNNoise, `webrtc-java`, 2026-07-03). См. [[recon-ai-corpus-2026-08-03]].

## 7. Что в работе прямо сейчас (Jira IVATR, локальная выгрузка)

- **IVATR-187** «Диаризация: ресерч методов кластеризации спикер-эмбеддингов и стабилизации оценки
  числа спикеров» — **Development in Progress** (создан 2026-07-03, приоритет Высокий).
- **IVATR-150** «IVA Terra: API для транскрибации аудиофайла» — **Development in Progress**.
- **IVATR-144** «Изучение использования ИИ в разработке» — Development in Progress.
- Открыты: **IVATR-149** (IVA GPT WEB компонент), **IVATR-151** (IVA GPT Bot), **IVATR-171**
  («Тестирование qwen3.5 для модуля adp»), **IVATR-153** (устранить искажения слов), **IVATR-157**
  (диаризация: оптимизация RTF), **IVATR-142** (мусор при транскрибации китайского),
  **IVATR-123** (API: перевод на арабский и китайский), **IVATR-169** (дистрибутив Terra 5.0).
- Закрыты недавно и важны как задел: **IVATR-181** «ASR benchmark: Терра ТПУ vs базовая Терра»
  (2026-06-11) — есть **методика сравнения ASR-моделей**; **IVATR-186** «Демо-стенд IVA Terra»
  (2026-06-30); **IVATR-176** «Макет рекомендаций по ВКС при написании письма» (2026-04-22) —
  LLM-подсказки в почте; **IVATR-175** «Тестирование модели **gemma**» (2026-04-09) — «гемма»
  руководителя существует, просто в Jira, а не в коммитах.
- Confluence-задел: `IVAADP/208699919` «Диаризация + идентификация», `IVAADP/208699922` «Демо IVA
  Terra» (обе 2026-07-02), `IVATerra/188481912` «Релиз 6.0», `IVATerra/160711129` «API /tts».

---

## КАНДИДАТЫ В ЗАЯВКУ (ранжировано по дешевизне)

| Что заявляем | Почему дёшево (задел) | Чего не хватает до работающей фичи |
|---|---|---|
| **1. Голосовое озвучивание протокола и поручений («слушай встречу вместо чтения»)** | TTS-сервис собран, интегрирован в Terra API, образ `terra_tts_worker` в релизе v5.1, API задокументирован (Confluence 160711129) | снять лимит 250 символов (нарезка на предложения + склейка), кнопка в UI ВКС/One. Модели уже локальные, GPU уже есть |
| **2. Мультиязычные субтитры «любой→любой»** | mBART-50 на **53 языка** + pivot через английский **уже в коде** (`ROUTES_WITH_PROXY`), fasttext-детект языка, MCU уже умеет субтитры (VCSWEB2-5543) | включить направления в `TRANSLATOR_LANGS`, замерить задержку, выбор языка в UI. Кода почти не нужно — конфиг |
| **3. Автоматическая идентификация участников по голосу (кто говорил, а не «Спикер 2»)** | WeSpeaker + `diar_identification` в ADP **уже работают** и подменяют имена в протоколе; IVATR-165 закрыт | реестр голосовых отпечатков сотрудников (enrollment), UI подтверждения имени, приватность/согласие |
| **4. Диалоговый ИИ-ассистент по встрече («спроси у протокола»)** | задача `assistant` в ADP готова, стриминг SSE, гейт `/adp_services_stream` в проде с 2025-12, история диалога на сервере | UI: IVATR-149/151 открыты. Это единственный разрыв — бэкенд есть целиком |
| **5. Вопросы к корпоративным документам (RAG)** | пайплайн RAG в ADP реализован и переведён на tool-calling (IVATR-163, IVATR-164 закрыты), есть задача на тестовый датасет | подключение к реальному хранилищу документов ИВА, оценка качества, продуктовый вход. В Terra кода нет — всё на стороне ADP |
| **6. Синхронный перевод речи с озвучкой (speech-to-speech)** | прототип `s2s_realtime_translator` v0.0.3 с фронтендом, стриминговым TTS (IVATR-184) и eval-скриптами; все три кирпича (ASR, MT, TTS) — свои | замеры задержки под нагрузкой, интеграция в MCU-поток, продакшн-стойкость. Самая «инновационная» и самая дорогая из списка |
| **7. Умные шаблоны протокола (протокол под тип встречи)** | движок сборки протокола, `planner`/`summary`, спроектированное управление шаблонами (IVATR-167, Confluence 195432645/195432647) | сам механизм шаблонов не реализован — только проект |
| **8. Подсказки по ВКС при написании письма** | макет сделан (IVATR-176, закрыт), ADP-ассистент есть, почта своя (`jump`/imail) | интеграция в почтовый клиент; вне Terra |

---

## ВЕРДИКТ

Terra — **не «ИИ-модуль агентного контура»**, а зрелая речевая платформа (ASR + VAD + выравнивание
слов + диаризация + перевод на 53 языка + TTS) и **шлюз** к внешнему LLM-сервису ADP. Под заявку
это выгодно: **готовые кирпичи уже лежат в релизе и не выведены в продукт** — TTS (есть всё, кроме
потребителя), мультиязычный перевод через pivot (есть всё, кроме конфига и UI), диалоговый ассистент
и RAG (бэкенд есть, UI открыт в Jira). Самое дешёвое «звучит инновационно» — озвучивание протоколов
и перевод «любой→любой»; самое эффектное и дорогое — синхронный перевод речи с озвучкой.

**Проверено:** состав движков и моделей, параметры и флаги, контракт ADP-гейта, статус TTS,
отсутствие CV внутри Terra, открытые задачи IVATR.
**Данные:** `~/tacticum/helm/data/real/git/repomix/terra-core.xml` (523 файла), `all_commits.csv`
(216 161 коммит), `repo_activity_all.csv` (233 репо), `jira/jira_issues.csv` (проект IVATR),
`confluence/pages_index.csv` (пространства IVATerra, IVAADP).
**Подтверждение:** каждая строка выше — с указанием файла и номера строки снимка, номера Jira или
id страницы Confluence.
**НЕ проверено:** (1) числа задержки S2S — метрики лежат в `blank130/s2s_realtime_translator`,
снимка репозитория нет; (2) имена конкретных голосов Silero и содержимое словарей конфигов —
вырезаны `--compress`; (3) тела страниц Confluence — локально выгружена ровно одна
(`IVAADP_131664812.md`), MCP `helm-analyst` мне недоступен; (4) есть ли voice cloning в s2s —
единственный след «prompt pcm» в тексте коммита, код не читал; (5) какая именно LLM крутится в
ADP — имя весов не зафиксировано ни в коде, ни в коммитах (в Jira тестировали gemma и qwen3.5).