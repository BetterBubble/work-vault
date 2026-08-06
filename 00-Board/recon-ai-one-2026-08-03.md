---
title: Разведка ИИ-функционала IVA One (web + RN) — что уже есть в коде
type: note
status: draft
role: explorer (разведчик)
created: 2026-08-03
tags:
- recon
- ai
- iva-one
- rn
- audit
permalink: tacticum/00-board/recon-ai-one-2026-08-03-1
---

# Разведка: интеллектуальный функционал IVA One (web + RN)

Аудит по запросу руководителя заказчика: **что из AI/ML/LLM уже реализовано в коде**.
Правило разведки: `НАЙДЕНО` = доказательство наличия (файл+символ). `НЕ НАЙДЕНО В СНИМКЕ`
≠ доказательство отсутствия — снимки сделаны `repomix --compress`, тела функций местами
вырезаны (разделитель `⋮----`).

---

## 1. Что проверено

| Корпус | Что это на самом деле | Файлов | Метод |
|---|---|---|---|
| `/Users/bubblemac/tacticum/helm/data/real/git/repomix/iva-one.xml` | `iva-ui-kit` v2.0.0 «Iva One» — Angular 21.1.1 / Nx монорепо: чаты, почта, календарь, звонки, контакты | 12 652 | `rg -i` + индекс путей + чтение блоков |
| `/Users/bubblemac/tacticum/helm/data/real/git/repomix/rn.xml` | **составной репозиторий `ione`** — см. врезку ниже | 6 578 | то же |

**Важно: `rn.xml` — это не один продукт, а три слоя.** Это самая существенная деталь всей
разведки, без неё находки атрибутируются неверно:

1. `web/**` (3 532 + 1 415 файлов) — **вендоренная копия продукта IVA**: `web/package.json` →
   `"name": "ivcs-web-ui", "version": "25.0.0", "description": "Iva-Connect UI package",
   "author": "Hi-tech Ltd <info@iva-tech.ru>"`. Это веб-клиент IVA Connect / UC
   (конференции, SFU, DLP) — **другой** продукт, чем `iva-one.xml`.
2. `rn-chat/`, `rn-mail/`, `rn-calendar/`, `rn-tasks/`, `rn-disk/`, `rn-contacts/`,
   `rn-shared/` — новые RN-модули поверх этого хоста (линия «One 1.5 / План А»).
3. `connect/ai-gateway`, `connect/bpm-service`, `connect/call-control-gateway` — **свои
   бэкенд-сервисы, исходников которых в снимке НЕТ** (лежат вне дерева, в снимок попали
   только деплой-конфиги). Владелец по тегам в реестре — `@ev`; прод-конфиг ведёт на
   `uc.iva.ru`, админы мини-приложений — `et@iva.ru`, `s.televinov@iva.ru`.

Просеяно: ~3 300 хитов по ~45 ключевым словам (ru+en), из них вручную разобрано и
атрибутировано ~600; остальное отсечено как шум (см. §5 «Ложные срабатывания»).

---

## 2. Таблица находок

### 2.1. Продукт IVA — то, что уже в проде

| Фича | Статус | Файл : символ | Что делает | Зрелость |
|---|---|---|---|---|
| **Транскрибация голосовых сообщений (ASR)** в чатах | НАЙДЕНО | `iva-one.xml` → `libs/chats/shared/feature-messages/src/lib/user/chat-room/contents/voice-message/transcription/transcription.service.ts` : `TranscriptionService.transcribe(uploadId)` + `transcription.directive.ts` : `TranscriptionDirective` + `transcription-trigger/`, `transcription-output/` | Кнопка «расшифровать» на голосовом сообщении; поллинг раз в секунду до `status: done`, текст разворачивается инлайн. API: `libs/messenger/shared/data-access/src/lib/file-upload/file-upload-api.service.ts` : `FileUploadApiService.transcribe(uploadId): Observable<TranscriptionResponse>` | **Прод**, за серверным фича-тогглом `speechToText` (`voice-message.component.html:68634`: `*libFeatureToggle="'speechToText'"`). Это ЕДИНСТВЕННЫЙ серверный тоггл в системе: `libs/shared/utils/src/lib/feature-toggle/feature-toggle.interfaces.ts` : `export type SystemFeatureKey = 'speechToText'` |
| **Живые субтитры конференции (real-time STT)** | НАЙДЕНО | `rn.xml` → `web/libs/conference/src/lib/store/subtitles/subtitles.service.ts` : `SubtitlesService.enable/disable/toggleSubtitles/setLanguage`; `subtitles.store.ts` : `SubtitlePhraseDTO`; UI `conference-session-shared/subtitles-wrapper/*` (обёртка, выбор языка, размер шрифта, режим наложения) | Фразы приходят событием `ConferenceSessionTranscriptionPhraseEvent`, рендерятся оверлеем | **Прод.** Лицензируется: строки `SUBSCRIPTION_RESTRICTION_TOTAL_EVENTS_WITH_SUBTITLES` / `LICENSE_RESTRICTION_...` в `web/libs/conference/assets/i18n/ru.json:290257` |
| **Машинный перевод субтитров** | НАЙДЕНО | `web/libs/conference/src/lib/store/subtitles/lang-code.ts` : `enum LangCode` (45 языков) + `SubtitlesAutoLangCode.AUTO`; `get-subtitles-available-languages.ts`; `subtitles.store.ts` : `SpeechRecognitionTranslationLanguagesDto`. Серверные настройки: `web/libs/data-access/src/lib/generated/iva-mcu/models/system-settings-constants.ts:381205-381206` : `TRANSCRIPT_TRANSLATION`, `TRANSCRIPT_TRANSLATION_LANGS`; `:381517` `SPEECH_RECOGNITION_API_TEXT_TRANSLATION_URL` | Пользователь выбирает язык субтитров, отличный от оригинала | **Прод**, но UI-выбор сужен до 5 вариантов: `ru.json:290244` → «Оригинальный / Русский / Английский / 中文 / العربية» |
| **Стенограмма мероприятия** (запись расшифровки в файл) | НАЙДЕНО | `iva-one.xml` → `libs/mcu/data-access/src/lib/generated/endpoints/conference-session/conference-session.service.ts` : `startTranscription()` / `stopTranscription()`; `rn.xml` → `web/libs/data-access/.../fn/conference-session/start-transcription.ts`. UI-строки: `ru-RU.json` → `room_settings_transcription` = «Автоматический старт стенограммы мероприятия»; `web/libs/conference/assets/i18n/ru.json:289794` → «Стенограмма остановлена. Вы можете найти ее в разделе Файлы.» | Модератор стартует стенограмму, результат — файл в разделе «Файлы» | **Прод**, лицензируется (`SUBSCRIPTION_MAX_EVENTS_WITH_TRANSCRIPT_DEFAULT`), есть аудит доступа (`AUDIT_ACCESS_TO_RECORD_TRANSCRIPT`), TTL хранения (`EVENT_TRANSCRIPT_STORAGE_DAYS`), формат файла (`TRANSCRIPT_FILE_FORMAT`) |
| **Подключаемый ASR-движок (серверная настройка)** | НАЙДЕНО | `web/libs/data-access/src/lib/generated/iva-mcu/models/system-settings-constants.ts:381513-381521` : `SPEECH_RECOGNITION_SERVICE`, `SPEECH_RECOGNITION_API_URL`, `SPEECH_RECOGNITION_API_KEY`, `SPEECH_RECOGNITION_API_POST_PROCESS_URL`, `SPEECH_RECOGNITION_TRANSLATION_LANGUAGES`, `SPEECH_RECOGNITION_MAX_PHRASE_DURATION`, `SPEECH_RECOGNITION_SUBTITLES_MAX_PHRASE_DURATION`, `SPEECH_RECOGNITION_PROXY_URI` | Распознавание речи — внешний сервис, адрес/ключ/прокси задаются админом. **Есть отдельный `_API_POST_PROCESS_URL`** — точка расширения под пост-обработку текста | **Прод** (контракт API MCU) |
| **Нейросетевое шумоподавление на устройстве** | НАЙДЕНО | `iva-one.xml` → `libs/ivcs-infrastructure/src/lib/vendors/noise-suppression-node/deep-filter/df.js` + `df_bg.wasm`, `rnnoise/rnnoise.js`, `noise-suppression.worker.ts`, `noise-suppression-node.ts`. `rn.xml` → `web/libs/common/media/src/lib/media/middleware/noise-suppression/*` + модель `web/libs/common/media/src/assets/noise-suppression/DeepFilterNet3_onnx.tar.gz` | DeepFilterNet3 (ONNX) и RNNoise, WASM в audio-worker, полностью локально | **Прод**, включено: `config.prod.json` → `"NOISE_SUPPRESSION_AVAILABLE": true`. Серверная настройка `NOISE_SUPPRESSION` |
| **Сегментация человека для виртуального фона / размытия** | НАЙДЕНО | `rn.xml` → `web/libs/common/media/src/lib/media/middleware/virtual-background/virtual-background.ts`, `tflite-simd.js`, `tflite.js`; модель `web/libs/common/media/src/assets/virtual-background/selfie_segmentation_landscape.tflite`; воркер `web/apps/app/src/app/services/virtual-background/vbg.worker.ts` : `virtual-bg.service.ts` | MediaPipe Selfie Segmentation через TFLite+WASM SIMD, локально в браузере | **Прод**: `config.prod.json` → `"SHOW_VIRTUAL_BACKGROUND": true`, `CONFERENCE_SESSION_SHOW_VIRTUAL_BACKGROUND_SEGMENTATION_WIDTH: 256` |
| **Распознавание объектов (видео)** | СЛЕДЫ | `web/libs/data-access/src/lib/generated/iva-mcu/models/system-settings-constants.ts:381705-381708` : `OBJECT_RECOGNITION_ENABLED`, `OBJECT_RECOGNITION_API_URL`, `OBJECT_RECOGNITION_LOGIN`, `OBJECT_RECOGNITION_PASSWORD` | Внешний сервис распознавания объектов, подключается по URL+логин/пароль | Только контракт настроек. **Ни одного использования этих констант в клиентском коде обоих корпусов** — фича серверная, в вебе не отражена |
| **Суммаризация стенограммы** | СЛЕДЫ (важно!) | `web/libs/data-access/src/lib/generated/iva-mcu/models/system-settings-constants.ts:381204` : `EVENT_TRANSCRIPT_SUMMARY = 'EVENT_TRANSCRIPT_SUMMARY'` | Серверная системная настройка «сводка стенограммы мероприятия» существует в контракте MCU API | **Настройка есть — потребителя нет.** `rg 'EVENT_TRANSCRIPT_SUMMARY'` даёт ровно одно совпадение на оба корпуса — само объявление enum. Ни один веб-клиент её не читает и ничего не рисует. Требует проверки в `ivcs-server.xml` / на живом MCU |
| **Боты в чатах (платформа)** | НАЙДЕНО | `iva-one.xml` → `libs/chats/shared/feature/src/lib/chat-room/send-message/chat-bot-keyboard/chat-bot-keyboard.component.ts` : `ChatBotKeyboardComponent.onButtonClick(text)`; вызывается из `send-message.component.html:61831`. Тип контакта: `libs/contacts/data-access/src/lib/contact/contacts-api.interfaces.ts:75606` : `ApiContactType = 'user' \| 'external' \| 'federation' \| 'resource' \| 'bot' \| 'personal'`. Серверный API документирован: во всех сгенерированных сервисах MCU шапка «Chat bot API is described here (/doc/api/bot.html)» | Bot — первоклассный тип контакта; боты присылают inline-клавиатуру (Telegram-подобно), нажатие отправляет текст в чат | **Прод.** Инфраструктура ботов есть, **ИИ внутри неё нет** |
| **Мини-приложения (SmartApps)** | НАЙДЕНО | `iva-one.xml` → `libs/messenger/shared/data-access/src/lib/smartapps/smartapps.types.ts` : `ApiSmartApp {id, displayName, avatar}`, `ApiSmartAppOpenLinkResponse {url}`; `smartapps-api.service.ts` : `SmartAppsApiService.openLink(id)`; `libs/messenger/feature/src/lib/navigation/services/navigation-smartapps.registrar.ts` : `NavigationSmartAppsRegistrar.register()`; вью `libs/messenger/shell/src/lib/smartapp/smartapp-view.component.ts` | Список приложений → вкладка в навигации → встроенный webview по выданному URL | **Прод, но минимально:** только id/имя/иконка/ссылка. Нет категорий, прав, каталога, поиска — **это контейнер мини-приложений, а не маркетплейс** |
| **Подбор свободных слотов для встречи** | НАЙДЕНО | `iva-one.xml` → `libs/calendar/feature/src/lib/events/create-edit-shared/freebusy/participants-business/get-busy-participants.ts`, `busy-participants.service.ts`, `scheduler/scheduler.component.ts`. UI-строки `calendar_event_slots_*` | Пересечение free/busy участников, окно 9:00–19:00 | **Прод.** Детерминированный алгоритм, не ML — но пользователем воспринимается как «умный подбор» |

### 2.2. Слой `ione` / RN поверх IVA Connect — LLM-контур, задеплоенный на `uc.iva.ru`

> Это не часть релиза `ivcs-web-ui 25.0.0`. Это надстройка, живущая в том же репозитории.
> **Атрибуцию (чей это результат для целей аудита) нужно решить лиду — я фиксирую факты.**

| Фича | Статус | Файл : символ | Что делает | Зрелость |
|---|---|---|---|---|
| **AI Gateway на DeepSeek** | НАЙДЕНО (конфиг; исходников сервиса в снимке нет) | `rn.xml` → `web/tools/deploy/host-scripts/docker-compose.ai-gateway.yml:423400-423505`; `web/DEPLOY.md:429677` «### 6.8 AI Gateway — DeepSeek assistant (S1)»; `web/tools/deploy/deploy-uc.sh:425790+` | Node 22-alpine контейнер `iva-ai-gateway:current` на `127.0.0.1:8098`, публично — `https://uc.iva.ru/api/assistant/v1/{healthz,recommendations,chat/respond}`. Модели `deepseek-v4-flash` / `deepseek-v4-pro`, режимы `simple` / `thinking`. Ключ провайдера — только в `/opt/ai-gateway/.env` (chmod 600), в бандл не попадает. Авторизация: nginx пробрасывает `Authorization`, гейтвей валидирует против `${AI_GATEWAY_IVA_ONE_BASE_URL}/api/v1/me` | **Прод-конфиг включён:** `web/apps/app/src/assets/config/config.prod.json:174428` → `"RN_ASSISTANT_ENABLED": "true"`, `"RN_ASSISTANT_API_BASE": "https://uc.iva.ru/api/assistant/v1"`. Сам сервис помечен как **S1 / alpha** (`DEPLOY.md:428962`: возвращает `500`, пока не заполнен `/opt/ai-gateway/.env`) |
| **RAG / корпоративная память коммуникаций** | НАЙДЕНО (конфиг) | тот же compose: сервисы `qdrant` (`qdrant/qdrant:latest`, коллекция `assistant_pointer_memory`) и `embeddings` (`michaelf34/infinity:latest`, модель `BAAI/bge-m3`, 1024 dim), профиль `rag` | Индексируются **указатели** (`source_ref`) на сообщения чатов и письма — `AI_GATEWAY_RAG_SOURCE_TYPES=chat_message,email_message`, сырой текст в Qdrant не пишется (`DEPLOY.md:429867` «raw source text is fetched to create embeddings and discarded before Qdrant persistence»), TTL 30 дней, обязательный ACL-fingerprint, порог `MIN_SCORE=0.60` | **Выключено по умолчанию в проде:** `AI_GATEWAY_RAG_ENABLED=false`, `AI_GATEWAY_RAG_RECOMMENDATIONS_ENABLED=false`, `AI_GATEWAY_RAG_RECENT_INDEX_ENABLED=false`. Механизм построен, включается оператором |
| **Ассистент в чате («UC Assistant»)** | НАЙДЕНО | `rn.xml` → `rn-chat/src/assistant/client.ts` : `requestAssistantChatResponse(...)`, `AssistantChatResponse {answer, botPost, sourceRefs[]}`; `rn-chat/src/assistant/types.ts` : `AssistantChatConfig`, `AssistantChatRequestContext {visibleTitle, visibleSummary, selectedText, user}`; `rn-chat/src/web/customElement.tsx:51224` : атрибуты `assistantApiBaseUrl` / `assistantBotUserId` / `assistantBotEmail` | Ассистент — **обычный директ-чат** с ботом (`RN_ASSISTANT_BOT_EMAIL=etdev@iva.ru`, id `27ce3ec0-…`). Сообщение уходит в обычный бэкенд чата, затем браузер зовёт `/chat/respond`; ответ гейтвей публикует обратно от имени бота через bot API (`X-Iva-Bot-Api-Token`). Ответ несёт `sourceRefs` — цитаты RAG | **Прод-конфиг включён.** Есть E2E против живого бэкенда: `e2e/chat-assistant-real.spec.ts`, `e2e/playwright-chat-assistant-real.config.ts` |
| **Суммаризация письма + черновик задачи («Summarize» в почте)** | НАЙДЕНО | `rn.xml` → `rn-mail/src/utils/assistantBriefWithTask.ts` : `MailBrief`, `MailTaskDraft`, `MailBriefCitation`, `MailBriefWithTaskResult`, `MailAssistantError`; UI — `rn-mail/src/panels/MailRightRail.tsx`; гард `rn-mail/src/panels/__tests__/MailRightRail.assistantTaskDraft.guard.test.ts` | Один round-trip к навыку `mail-brief-with-task`: гейтвей валидирует вывод модели через zod и стримит SSE-событие с **брифом** (`summary`, `decisions`, `blockers`, `followUps`) **и черновиком задачи разом**, обе части с цитатами. Раньше было два вызова — `summarize-next-actions` + `mail-draft-task` | **Реализовано, покрыто E2E против живого Jump-бэкенда:** `e2e/mail-assistant-task-real.spec.ts` — «CM-110 / TASKS-004 — AI-suggested task created in Mail must be findable in the Tasks module», задача реально сохраняется через `TasksBridge.createTask` |
| **Черновик ответа на письмо** | НАЙДЕНО | `rn-mail/src/utils/assistantDraftResponse.ts` : `MailAssistantDraftResponseRequest {assistantBrief, assistantSourceRefs}`, `buildSourceRefs()`, `formatAssistantBrief()`, `parseAssistantSseText()`; вход — `rn-mail/src/compose/ComposeDock.tsx:68871` : `getAssistantApiBase()` | Генерация ответа поверх уже полученного брифа, с прокидыванием тех же цитат | Реализовано; E2E `e2e/chat-assistant-draft-real.spec.ts` |
| **Подтверждение календарного события ассистентом** | НАЙДЕНО (E2E) | `e2e/assistant-calendar-confirmation-real.spec.ts` + `e2e/playwright-assistant-calendar-confirmation-real.config.ts` | Сценарий: ассистент → создание события в календаре, тест проверяет по `eventUid` на живом бэкенде и подчищает за собой | Реализовано на уровне сценария |
| **«Коммуникационная вики» — производные факты с цитатами** | СЛЕДЫ | `docker-compose.ai-gateway.yml:423447-423451` : `AI_GATEWAY_WIKI_ENABLED`, `AI_GATEWAY_WIKI_DIR=/var/lib/ai-gateway/wiki`, `AI_GATEWAY_WIKI_REQUIRE_SOURCE_REFS=true`, том `wiki-data`; описание `DEPLOY.md:429880` | Паттерн `raw sources -> wiki -> schema`: гейтвей хранит выведенные факты как markdown с обязательными ссылками на источник, в публичные рекомендации отдаёт только совпавшие компактные страницы | **Выключено** (`AI_GATEWAY_WIKI_ENABLED=false`). Ближайшее к «корпоративной памяти» из презентации |
| **Маркетплейс мини-приложений** | НАЙДЕНО | `rn-shared/src/miniapps/types.ts` : `MiniAppCatalogItem {id, name, shortDescription, owner, category, runtime, badges, favorite, recent, disabledReason}`, `MiniAppCategory` (`hr\|sales\|finance\|it\|operations`), `MiniAppPermission {kind: read\|write\|approve\|admin}`, `MiniAppField` (динамические формы: text/date/select/boolean, `multi`, группы полей); `rn-shared/src/miniapps/client.ts`, `sdk.ts`, `launchIntent.ts`; хост `web/apps/app/src/app/features/rn-modules/rn-mini-apps-host.component.ts` | Полноценная модель каталога: категории, права, избранное, недавние, два рантайма (`api` / `webview`), декларативные формы действий | **Прод-конфиг включён:** `config.prod.json:174426` → `"RN_MINIAPPS_API_BASE": "https://uc.iva.ru"`. Это и есть маркетплейс — **но в линии RN, не в `iva-one`** |
| **Движок бизнес-процессов (BPM) — «агент действий»** | НАЙДЕНО | `rn-shared/src/bpm/types.ts` : `BpmStepType` (`task\|approval\|condition\|wait\|notify\|miniapp-action\|create-task\|create-event\|send-mail\|open-chat`), `BpmAutomationConfig {module, actionId, miniAppId, requiredFields}`, `BpmSourceModule`, `BpmCollaborationKind`; деплой `web/tools/deploy/host-scripts/docker-compose.bpm.yml` под `/api/v1/bpm/` и `/api/v1/miniapps/` | Шаблоны процессов выполняют действия в чате/почте/календаре/задачах/мини-приложениях | **Прод-конфиг включён:** `"RN_BPM_ENABLED": "true"`. **Детерминированный workflow-движок, не LLM** — но функционально это «агент, который делает действия» |
| **Транскрибация голосовых в RN-чате** | НАЙДЕНО | `rn-chat/src/hooks/useVoiceTranscription.ts` : `useVoiceTranscription` (состояния `idle\|loading\|ready\|error`, поллинг 1 c, таймаут 60 c, module-level кеш по `uploadId`); компоненты `VoiceTranscribeButton.tsx`, `VoiceTranscriptView.tsx` | Порт `TranscriptionDirective`/`TranscriptionService` из iva-one на React-хук — по собственному докблоку «Mirrors iva-one-master … 1:1 in semantics» | Реализовано, есть тесты |

### 2.3. «Умное», но без ИИ (важно не выдать за ИИ)

| Фича | Файл : символ | Механизм |
|---|---|---|
| Локальный полнотекстовый поиск по всем модулям | `rn-shared/src/search/LocalIndex.ts` (`import MiniSearch from 'minisearch'`), `SearchService.ts` : `rerank()`, `providers/*`, `SearchOmnibox.tsx` | MiniSearch (BM25 + префикс + fuzzy), **не векторный** |
| Поиск по содержимому вложений | `rn-mail/src/search/AttachmentTextIndex.ts` : `AttachmentTextIndexHit`, `isAttachmentTextIndexable()`, `buildSnippet()` | Извлечение текста вложения + лексический индекс |
| Расширение поискового запроса | `rn-shared/src/search/queryExpansion.ts` : `buildSearchTextVariants()`, `transliterateCyrillicToLatin()`, `transliterateLatinToCyrillic()` | Транслитерация и варианты раскладки, словарь правил |
| Разбор задачи из естественного языка | `rn-shared/src/tasks/nlParse.ts` : `ParsedQuickAdd`, грамматика дат/времени/приоритетов/`#меток`/`@исполнителей`/повторов, ru+en | Регулярки, **не LLM**. По собственному докблоку «grammar is intentionally small and non-greedy» |
| Проверка орфографии | `rn-shared/src/spellcheck/core.ts` (`import nspell from 'nspell'`), `types.ts` : `SpellcheckLanguage = en\|ru\|be\|id\|kk\|uz\|tg`, `SpellcheckSuggestionSource` включает `'keyboard-layout'`; «исправить всё» — `rn-mail/src/spellcheck/mailFixAll.ts` | Hunspell-словари + детект неверной раскладки |
| Подбор времени встречи в RN | `rn-calendar/src/components/SuggestTimeDialog.tsx` → `rn-calendar/src/utils/findCommonFreeSlots.ts` | Пересечение free/busy |
| «Рекомендации людей» | `rn-shared/src/people/peopleRecommendations.ts`, `PeopleRecommendationStore.ts`, `rn-chat/src/utils/mentionPeopleRecommendations.ts` : `enrichMentionMembersWithPeopleRecommendations()` | Недавние/частые контакты, ранжирование по recency |
| Правила обработки почты | `iva-one.xml` → `libs/mail/data-access/src/lib/rule.consts.ts`, `rule.helpers.ts`, `libs/jump/data-access/src/lib/jump-api-rule.interfaces.ts` | Фильтры «если → то», задаются пользователем |
| DLP (контроль утечек) | `rn.xml` → `web/libs/dlp/src/lib/components/*`, `ResourceScanAccessState`; настройки `DLP_SAVEFILE_ON_DLP_DETECTED` | Серверное сканирование ресурсов, вердикт приходит клиенту |

---

## 3. Первоисточник о том, чего НЕТ (самое ценное для аудита)

В `rn.xml` → `docs/WORKAROUNDS-REGISTRY.md` есть три авторские записи от **2026-04-18**,
прямо описывающие пробел. Это не мой вывод, а зафиксированное в репозитории утверждение
команды — и оно снимает риск «сказали, что нет, а оно есть».

**`AI-WR-001`** (строка 12956), дословно:

> «The IVA Connect tree does not contain any LLM client, any GPT post-processing pipeline,
> or an "AI bot" implementation. The generic chat-bot plumbing (`users-main` identity,
> `chats-main` webhook delivery, `admin-api-gateway-main` GraphQL CRUD,
> `iva-admin-develop/projects/one/dashboard/chatbots` admin UI) is fully built but has zero
> AI inside it.»

**`AI-WR-002`** (строка 12974), дословно:

> «The conference transcription pipeline (`iva-admin-develop/projects/terra` admin + MCU
> subtitles) produces raw `.txt` / `.xml` transcripts and stops there — no summarisation,
> no entity extraction, no GPT post-processing.»

Там же перечислено, что было **запланировано и не сделано** (все — за capability-флагом
`aiBridge.getCapabilities()`, который возвращает `null`, пока внешний сервис `iva-ai-bot`
не зарегистрирован): compose-assist, summarise-thread, smart-reply chips, triage hint,
NL search, NL event entry, agenda-from-thread, slash-меню ИИ в чате, room summary,
conference recap card, голосовая диктовка, daily digest, экран Settings → AI с раскрытием
провайдера/региона/retention.

**`AI-WR-003`** (12992) — двухступенчатое согласие на микрофон для голосовых ИИ-функций.

Хронология, которую это задаёт: апрель 2026 — пробел зафиксирован, ставка на внешний
`iva-ai-bot`; позже — вместо него в той же ветке появился собственный `connect/ai-gateway`
на DeepSeek (§2.2). `PLAN-AI-INTEGRATION.md`, на который ссылаются все три записи,
**в снимок не попал**.

---

## 4. Чего не нашли и где доискать в живом коде

| Фича из презентации | Вердикт по снимкам | Где смотреть дальше |
|---|---|---|
| **Суммаризация сообщений/встреч** (названа руководителем главной) | Настройка `EVENT_TRANSCRIPT_SUMMARY` в контракте MCU есть, потребителя нет ни в одном веб-клиенте. Для почты — реализована, но в слое `ione`/RN (§2.2) | `ivcs-server.xml` (сервер MCU) — есть ли реализация за настройкой; репозиторий **Terra** (`iva-admin-develop/projects/terra/src/app/dashboard/transcriptions/`, модель `read-transcription-detail.ts` с `result_txt_path`/`result_xml_path`) — в мой корпус не входит |
| **Единый ИИ-слой «IVA GPT»** | `НЕ НАЙДЕНО В СНИМКЕ.` `rg -i 'iva.?gpt\|нейросет\|искусственн'` — 0 содержательных совпадений на 19 230 файлов | Маркетинговые материалы / роадмап, не код |
| **Маскирование персональных данных (PII)** | `НЕ НАЙДЕНО В СНИМКЕ` как продуктовая функция. Все совпадения — соседние понятия: `docs/logging.md:21645` «Маскировка чувствительных данных (LogSanitizer)» — санитайзер **логов**; `libs/ui-kit/src/lib/inputs/input-phone/ADR.md` — маска ввода телефона; `sendDefaultPii: false` — настройка Sentry | `ivcs-server.xml`, `jump.xml` (почта), DLP-бэкенд. В обоих веб-клиентах — нет |
| **Журнал ИИ** | `НЕ НАЙДЕНО В СНИМКЕ.` Ближайшее: серверная настройка `AUDIT_ACCESS_TO_RECORD_TRANSCRIPT` (аудит доступа к стенограмме) и санитайзные метрики гейтвея `[ai-gateway:metric]` (`DEPLOY.md:429849`), которым прямо запрещено содержать тексты промптов, id чатов и почт | Админ-панель `iva-admin-develop`, сервер |
| **Классификация / тональность / intent-детект сообщений** | `НЕ НАЙДЕНО В СНИМКЕ.` `sentiment`/`тональн` — 0 содержательных; «Важные письма» (`profile_notifications_header_mails_important`) — пользовательский флаг важности, не классификатор | — |
| **Векторный/семантический поиск на клиенте** | `НЕ НАЙДЕНО В СНИМКЕ.` Все 227 совпадений `vector` в rn — `react-native-vector-icons`; `semantic` — про семантику версий и API. Векторный поиск существует только в бэкенд-контуре `ai-gateway` (§2.2) и выключен | — |
| **Маркетплейс агентов** (не мини-приложений) | `НЕ НАЙДЕНО В СНИМКЕ.` В `MiniAppCatalogItem` нет ни поля про агента/модель, ни категории `agent` | — |
| **Конспекты встреч в интерфейсе** | `НЕ НАЙДЕНО В СНИМКЕ.` В `web/libs/conference` нет ни компонента, ни i18n-ключа про конспект/резюме/итоги встречи. `rg 'конспект'` — 0 на оба корпуса | `ivcs-server.xml`, Terra |
| **Исходники самого `ai-gateway`** | Вне снимка: живут в `connect/ai-gateway/` (по `DEPLOY.md:429692` — `/Users/ev/ione/connect/ai-gateway/Dockerfile`), в repomix не попали | Отдельный клон/снимок этого каталога. **Рекомендую запросить его — это единственное место, где реально написан LLM-код** |

---

## 5. Ложные срабатывания, которые пришлось отсеять

Записываю, чтобы следующий разведчик не потратил на них время и чтобы аудит не раздули.

- **`summary` (459 хитов в iva-one) — почти весь шум.** Это либо JSDoc-тег `@summary` в
  сгенерированном orval-коде, либо DTO вида `ConferenceSessionSummaryInfoRestDTO` —
  *сводная карточка конференции* (владелец, число участников, длительность), а не
  ИИ-выжимка. Проверено чтением `libs/mcu/data-access/src/lib/generated/models/conference-session-summary-info-rest-dt-o.ts`.
- **`translation` ≠ перевод.** `LiveTranslationRestDTO` — это **трансляция** (поля
  `Live stream server URL`, `Stream name`, `flvMainContentUrl`), а `WEBINAR_*_TRANSLATION_FPS`
  / `USE_FLV_TRANSLATION_FOR_*` — параметры видеотрансляции.
- **«Синхронный перевод» в UI — человек, а не машина.** `room_settings_simultaneous_translation`
  = «Синхронный перевод», но рядом `create_event_member_role_translator` = «Переводчик»,
  `create_event_member_role_translator_dialog_title` = «Назначить переводчиком», а
  `TranslationLanguageType` описан как *«User's broadcast language (for conferences with
  simultaneous interpretation feature)»*. Это назначение живого синхрониста и раздача
  языковых дорожек. Машинный перевод в продукте есть — но только у субтитров (§2.1).
- **`llama` (97 хитов в rn)** — испанское `llamada` («звонок») в `assets/i18n/es.json`.
- **`ai`** ловит `mail`, `available`, `detail`; **`bot`** ловит `bottom`; **`model`** ловит
  MVC и `MediaQualityDTO`; **`agent`** ловит `user-agent`
  (`web/apps/app/src/app/services/user-agent/iva-user-agent.service.ts` — 29 хитов).

---

## 6. Неожиданное — интеллектуальное, чего в списке фич не было

1. **Два ML-инференса уже крутятся у пользователя в браузере, каждый день.** DeepFilterNet3
   (ONNX/WASM) для шумоподавления и MediaPipe Selfie Segmentation (TFLite/WASM SIMD) для
   виртуального фона. Это единственный ML в продукте, который работает **сейчас, локально,
   без внешних сервисов и без вопросов о передаче данных**. В презентации его нет, а для
   разговора «у нас уже есть ИИ на устройстве» это самый сильный аргумент.
2. **`SPEECH_RECOGNITION_API_POST_PROCESS_URL`** — в контракте MCU уже заложена точка
   расширения под пост-обработку распознанного текста отдельным сервисом. То есть место
   под LLM-полировку стенограммы спроектировано, просто пустует.
3. **Субтитры умеют 45 языков и авто-детект** (`LangCode`, `SubtitlesAutoLangCode.AUTO`),
   а UI показывает только 5. Разрыв между возможностью и витриной.
4. **ASR полностью отвязан от вендора** — `SPEECH_RECOGNITION_SERVICE` + URL + ключ + прокси.
   Движок распознавания меняется настройкой, без правки клиента.
5. **BPM-движок (§2.2) — это готовый «агент действий» без ИИ.** `BpmStepType` уже умеет
   `create-task`, `create-event`, `send-mail`, `open-chat`, `approval`, `condition`,
   `miniapp-action`. Чтобы получить «агента, который делает», не хватает только слоя,
   который выбирает шаги — то есть LLM поверх уже готового исполнителя.
6. **В RAG-контуре сознательно выбрана «память указателей», а не копий.** Коллекция
   называется `assistant_pointer_memory`, в Qdrant кладутся эмбеддинги и `source_ref`,
   сырой текст выбрасывается, требуется ACL-fingerprint, TTL 30 дней. Для разговора с
   безопасностью заказчика это сильная позиция, и она уже реализована, а не обещана.
7. **Лицензирование ИИ-функций уже существует.** `SUBSCRIPTION_MAX_EVENTS_WITH_TRANSCRIPT_DEFAULT`,
   `SUBSCRIPTION_MAX_EVENTS_WITH_SUBTITLES_DEFAULT` + локализованные ошибки превышения.
   Значит стенограмма и субтитры — не эксперимент, а продаваемая функция с квотами.
8. **Детект неверной раскладки клавиатуры** как отдельный источник подсказок
   (`SpellcheckSuggestionSource = 'keyboard-layout'`) — мелочь, но пользователем читается
   как «умный ввод».

---

## 7. Ограничения этой разведки

- Снимки статичны на **2026-07-03**. Всё, что вмержено после, здесь не видно.
- `--compress` вырезал тела функций: я подтверждал наличие по сигнатурам, импортам,
  конфигам, строкам локализации и E2E-тестам. Ни одно «НЕ НАЙДЕНО» из §4 не следует
  читать как «этого нет в продукте» — только как «этого нет в этих двух снимках».
- Исходников `connect/ai-gateway` (единственного места с реальным LLM-кодом) в корпусе
  нет — все выводы о нём сделаны по деплой-конфигам, `DEPLOY.md` и клиентским контрактам.
- Серверные корпуса (`ivcs-server.xml`, `jump.xml`, `kmp.xml`, `terra-core.xml`) — вне моей
  зоны; по `EVENT_TRANSCRIPT_SUMMARY`, PII и журналу ИИ ответ, скорее всего, там.