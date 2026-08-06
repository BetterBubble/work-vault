---
title: Разведка ИИ-функций — ВКС/MCU (ivcs-server) и мобильный KMP
type: note
status: draft
created: 2026-08-03
tags:
- recon
- ai
- вкс
- mcu
- ivcs-server
- kmp
- транскрипция
- субтитры
- terra
permalink: tacticum/00-board/recon-ai-vks-2026-08-03
---

# Разведка ИИ-функций: ВКС/MCU + мобильный клиент

Роль: explorer. Правок не вносил, только чтение.

## 1. Что проверено

| Источник | Объём | Как смотрел |
|---|---|---|
| `/Users/bubblemac/tacticum/helm/data/real/git/repomix/ivcs-server.xml` | 37 МБ, 12 729 файлов (Java/GWT, сервер ВКС) | `rg -i` по ~60 ключевым словам (en+ru) + сопоставление номеров строк с `<file path=...>`; чтение целиком найденных классов |
| `/Users/bubblemac/.../repomix/kmp.xml` | 38 МБ, 11 171 файл (Kotlin Multiplatform, клиент One) | то же |
| `/Users/bubblemac/.../repomix/load-runner.xml` | 247 КБ, 128 файлов | полный обзор списка файлов + сквозной греп |
| `topology/kmp/topology-review.md` | 88 строк | прочитан (карта 116 gradle-модулей / 34 блока) |
| SCIP `scip/ivcs-server/`, `scip/kmp/` | **пусто по нужным языкам** | `java/` и `kotlin/` — пустые каталоги. Символьный индекс недоступен, вся работа сделана по repomix |

**Ограничения снимка (важно для чтения выводов):**

- Repomix с `--compress`: тела функций местами вырезаны (`⋮----`). Например,
  **`shared/src/main/java/su/ivcs/videoconference/util/SystemSettingsConstants.java` сжат до одних
  javadoc-комментариев** — все имена системных настроек оттуда невидимы. Часть настроек я восстановил
  по i18n-интерфейсам, SQL и по сгенерированному OpenAPI-энуму в KMP.
- Из снимка **исключены веса моделей**: `*.pt, *.bin, *.onnx, *.pb, *.h5, *.tflite, *.pth, *.ckpt` и
  `*.jar/*.so/*.dll`. Но раздел `<directory_structure>` перечисляет **все** пути, включая бинарные, —
  я по нему прошёлся: файлов моделей нет ни в одном из двух репозиториев.
- «НЕ НАЙДЕНО В СНИМКЕ» ≠ «не существует».

## 2. Главный вывод одной строкой

**Интеллект в ВКС есть и работает в проде, но своего ML-кода в `ivcs-server` нет ни строчки.**
Весь ИИ — внешний сервис **IVA Terra**, к которому ходят по HTTP; `ivcs-server` — оркестратор
(включить/выключить, собрать фразы, сложить файлы, разослать события) и медиасервер (MCU) — транспорт
аудио. Мобильный KMP-клиент умеет управлять стенограммой и расшифровывать голосовые в чате, но своего
ML не содержит тоже.

## 3. Таблица находок

Легенда зрелости: **прод** — полный путь от UI до сохранения результата, с лицензией/квотами/аудитом ·
**интеграция** — код есть, вычисление во внешнем сервисе · **частично** — часть слоёв есть, UI нет.

| # | Фича | Статус | Файл : символ | Что делает | Зрелость |
|---|---|---|---|---|---|
| 1 | Стенограмма мероприятия (транскрипция) | **НАЙДЕНО** | `server/src/main/java/su/ivcs/videoconference/ejb/impl/TranscriptionServiceImpl.java` : `start()` / `stop()` / `writeTranscript()` | Старт/стоп по сессии, лок медиа-комнаты, проверка настройки домена `EVENT_TRANSCRIPT`, проверка лицензии и квоты подписки, аудит, рассылка события участникам. Фразы пишутся асинхронно в 2 потока | **прод**, интеграция с Terra |
| 2 | Приём распознанных фраз от MCU | **НАЙДЕНО** | `server/src/main/java/su/ivcs/media/listeners/MediaCallbackServiceListener.java` : L417129 → `transcriptionService.writeTranscript(participantId, info.getPhrase(), ...)` | Медиасервер отдаёт готовую фразу с `participantId`; сервер парсит JSON Terra и кладёт в БД | **прод** |
| 3 | «Протокол встречи → поручения» | **НАЙДЕНО** | `TranscriptionServiceImpl.java` : `createDocumentsToWriteTranscription()`, `writeProcessedInformation()`; тип `ejb/impl/transcription/model/TerraTranscriptionType.java` (`FULL_TRANSCRIPTION` / `ASSIGNMENTS` / `SUMMARY`) | По окончании стенограммы дергается Terra `/minutes`; из ответа раскладываются **три** документа: полная стенограмма, **поручения**, **конспект**. Включается настройкой домена `EVENT_TRANSCRIPT_SUMMARY` | **прод**, вычисление в Terra |
| 4 | Конспект (SUMMARY) | **НАЙДЕНО** | там же; фильтр по `IVA_TERRA_SUMMARY_PARTICIPANT_ID` | Terra возвращает конспект как «реплику» служебного участника; сервер её вылавливает и пишет отдельным файлом | **прод**, вычисление в Terra |
| 5 | Живые субтитры в мероприятии | **НАЙДЕНО** | `server/src/main/java/su/ivcs/media/business/impl/ParticipantMediaServiceImpl.java` : `startSubtitling(SubtitlesLanguage)`; REST `api/rest/client/resource/conference/ConferenceSessionResource.java` : `startSubtitling()` | Каждый участник включает субтитры себе и выбирает язык; фразы летят `ConferenceSessionTranscriptionPhraseEvent` через `server2ClientBus` | **прод** |
| 6 | Машинный перевод субтитров на лету | **НАЙДЕНО** | `TranscriptionServiceImpl.java` : `requestTranslation()` | Для каждого языка, который запросили участники, фраза уходит POST'ом на Terra `/translate_to_any` и рассылается уже переведённой; язык `ORIGINAL` идёт без перевода | **прод**, вычисление в Terra |
| 7 | Перевод готовой стенограммы (пакетом) | **НАЙДЕНО** | `TranscriptionServiceImpl.java` : `tryToTranslateTranscription()`, `handlePostProcessedTranslationResponse()`; модели `ejb/impl/transcription/model/TerraTranslationRequest.java`, `TerraTranslationPayload.java` | Итоговый протокол переводится на языки из `TRANSCRIPT_TRANSLATION_LANGS`, каждый язык — своя папка с тремя файлами | **прод**, вычисление в Terra |
| 8 | Выбор системы распознавания | **НАЙДЕНО** | `shared/src/main/java/su/ivcs/videoconference/dto/media/SpeechRecognitionSystem.java`; значения видны по вызовам: **`IVA_TERRA`, `STC`, `YANDEX`** (`server/src/test/java/su/ivcs/videoconference/ejb/impl/TranscriptionServiceTest.java` L660474/660478/660965) | Настройка `speech_recognition_service`. Постобработка (поручения/конспект/перевод) — **только для `IVA_TERRA`** (`isSpeechRecognitionServiceIsIvaTerra()`) | **прод** |
| 9 | Автостарт стенограммы | **НАЙДЕНО** | `TranscriptionServiceImpl.java` : `tryToAutoStartTranscription()`; фичи `ConferenceFeatureType.TRANSCRIBE_AT_START`, `AUTO_START_TRANSCRIBING_ON_USER_LEVEL` | Стенограмма стартует сама при переходе сессии в ACTIVE | **прод** |
| 10 | Экспорт протокола в TXT/DOCX | **НАЙДЕНО** | `TranscriptionServiceImpl.java` : `writeDocument()`, `writeToDocxFile()`, внутренний `DocxWriter`; `shared/.../dto/media/TranscriptFileFormat.java` | Через Apache POI `XWPFDocument`; формат — настройка домена `TRANSCRIPT_FILE_FORMAT` | **прод** |
| 11 | Поиск по стенограммам (админка) | **НАЙДЕНО** | `server/src/main/java/su/ivcs/services/i/repositories/ConferenceTranscriptRepositoryImpl.java` : `findAllByParams()`; UI `gwt/vcs/.../adminevent/transcription/EventTranscriptionTabActivity.java` | Постраничный поиск фраз по мероприятию в админ-панели | **прод**, обычный SQL (не векторный поиск) |
| 12 | Расшифровка голосовых сообщений в чате | **НАЙДЕНО** | `core/network/src/commonMain/kotlin/service/VoiceFileService.kt` : `transcribeVoice()` → `POST {uploads}/{uploadId}/transcribe`; UI `feature/chat2/impl/.../message/voice/VoiceTranscribeButton.kt`; логика `.../chat_feed/ChatFeedComponentImpl.kt` : `onTranscribeVoiceClick()` | Кнопка у голосового; клиент **опрашивает бэк** с растущей задержкой (до 20 с) пока статус не `DONE`. Модель статусов — `core/model/.../file/VoiceTranscriptDTO.kt` (`PROCESSING`/`DONE`) | **прод** на клиенте; сам ASR — на бэке мессенджера (другой репозиторий) |
| 13 | Фича-флаг расшифровки голосовых | **НАЙДЕНО** | `core/network/src/commonMain/kotlin/models/responses/system/properties/ResponseFeatures.kt` : `speechToText: Boolean`; состояние `feature/chat2/.../chat_feed/VoiceTranscriptState.kt` : `speechToTextIsEnabled` | Сервер сообщает клиенту, включена ли расшифровка | **прод** |
| 14 | Управление стенограммой с мобильного | **НАЙДЕНО** | `feature/ucim/conference-connect/src/androidMain/.../restService/api/ConferenceSessionApiProcessing.kt` : `startTranscription()`/`stopTranscription()`; кнопка `.../controlPanel/components/ConferenceButtonCreator.kt` (L570607, гейт `isStartTranscriptionEnabled()`) | REST `POST conference-sessions/{id}/transcription/start|stop` (роль модератора) | **прод** |
| 15 | Индикатор «идёт стенограмма» на мобильном | **НАЙДЕНО** | `feature/conference-multiplatform-ui/.../underlay/ConferenceIndicators.kt` (L350946); `feature/ucim/conference-connect/.../view/ConferenceScreenFragment.kt` (L598798); попап `.../eventsProcessors/IncomingMessageToConferenceSessionTranscriptionStateChangedEventProcessor.kt` | Индикатор + всплывашка при старте/остановке | **прод** |
| 16 | Разбор MCU-пушей о транскрипции | **НАЙДЕНО** | `core/centrifuge/src/commonMain/.../client/McuConferenceSessionStateJsonMapper.kt` : `TRANSCRIPTION_TYPES`, `TRANSCRIPTION_PHRASE_TYPES`, `TRANSLATION_DIRECTION_TYPES`; модель `McuConferenceSessionStatePush.kt` : `Transcription`, `TranscriptionPhrase` | Парсит из `/json/` MCU и состояние, и **отдельные фразы субтитров** (`ConferenceSessionTranscriptionPhraseEvent`), с тестами в `McuConferenceSessionStateJsonMapperTest.kt` | **частично**: транспорт готов и покрыт тестами, **рендера субтитров на экране конференции в KMP я не нашёл** |
| 17 | REST субтитров в KMP-клиенте | **НАЙДЕНО** | `feature/ucim/conference-connect/src/androidMain/.../ConferenceSessionApiProcessing.kt` : `startSubtitling()`/`stopSubtitling()` (L556393, L556401); DTO `StartSubtitlingRequestRestDTO.kt` | Сгенерированный клиент под ivcs API | **частично**: API-обвязка есть, вызывающего UI не нашёл |
| 18 | Приватность записи/стенограммы: индикация на видеополотне | **НАЙДЕНО** | `server/src/main/java/su/ivcs/media/business/impl/MediaStageSubscriptionServiceImpl.java` (L414770–414800) | На картинку MCU подмешиваются иконки `RECORD_ICON`/`TRANSCRIBE_ICON` (видно **всем**, включая SIP/H.323-терминалы) + текстовая плашка «Стенограмма мероприятия начата» первые `RECORD_TRANSCRIBE_NOTIFICATION_TTL_MS` | **прод** |
| 19 | Приватность: голосовое уведомление VVoIP | **НАЙДЕНО** | `TranscriptionServiceImpl.java` : `notifyParticipants()` → `conferenceVVoIPAudioNotificationService.pushNotification(ivrType, ...)`; гейт — настройка `VVOIP_VIDEO_NOTIFICATIONS` | Участникам по SIP/H.323 проигрывается IVR-сообщение о старте стенограммы | **прод** |
| 20 | Приватность: не сохранять стенограмму | **НАЙДЕНО** | `TranscriptionServiceImpl.java` : `needToSaveResource()` + `deleteTranscriptionFor()` | При `CLEAR_ROOM_RESOURCES_AFTER_PARTICIPANTS_EXIT` и нуле онлайн-участников стенограмма удаляется, а не сохраняется | **прод** |
| 21 | Приватность: срок хранения | **НАЙДЕНО** | `server/src/main/java/su/ivcs/videoconference/jobs/transcription/TranscriptionCleanerJob.java` → `deleteTranscriptions(olderThenDateInMs)`; настройка `event_transcript_storage_days` | Плановая чистка старых стенограмм | **прод** |
| 22 | Приватность: права и аудит | **НАЙДЕНО** | `server/src/main/java/su/ivcs/videoconference/controller/ConferenceSessionControllerImpl.java` : `startTranscription()` (L459466+) | Требует роль модератора **и** право загрузки ресурсов **и** профильную фичу `ToggledProfileFeature.TRANSCRIPT_MANAGEMENT`; факт старта/стопа пишется в аудит (`ConferenceSessionRecordStateChangeInfo`, `ResourceObjectInnerType.TRANSCRIPT`) | **прод** |
| 23 | Лицензирование ИИ-функций | **НАЙДЕНО** | `server/src/main/java/su/ivcs/license/business/LicenseServiceImpl.java` : `checkEventsWithTranscriptRestriction()`, `checkEventsWithSubtitlesRestriction()`; `license/util/RestrictionCheckUtil.java` | Отдельные лимиты лицензии и подписки на число одновременных мероприятий со стенограммой и с субтитрами. **Это продаётся отдельно** | **прод** |
| 24 | Диагностика ИИ-подсистемы | **НАЙДЕНО** | `shared/.../dto/alert/info/SpeechRecognitionErrorInfo.java`, `SpeechRecognitionExecuteErrorInfo.java`; `TranscriptionServiceImpl.onTranscriptionError()` | Системные алерты «настройки распознавания некорректны» / «ошибка распознавания», со счётчиком повторов, видны в админке | **прод** |
| 25 | Шумоподавление / эхо / АРУ | **НАЙДЕНО, но это не ИИ** | `feature/webrtc/impl/src/androidMain/.../StreamPeerConnectionFactory.kt` (L813696–813700), `.../WebRtcFeaturesManagerImpl.kt` (чёрные списки моделей устройств), `feature/webrtc/impl2/src/jvmMain/.../PlatformMedia.jvm.kt` (L819062–819072) | Штатный WebRTC APM: `setUseHardwareNoiseSuppressor`, `googNoiseSuppression`, `NoiseSuppression.Level.MODERATE`, highpass. **Классический DSP, нейросетевого шумодава (RNNoise/Krisp) нет** | прод (DSP) |
| 26 | Активный спикер | **НАЙДЕНО, но это не ИИ** | `ConferenceFeatureType.ACTIVE_SPEAKER_INDICATION`; настройка `LAYOUT_SPEAKER_ACTIVATION_DELAY_MS` (`MediaConferenceServiceBean.java` L413012); в вебе `gwt/vcs/.../webrtc/service/impl/StreamVoiceActivityTrackerServiceImpl.java` | Детекция по энергии аудио на MCU/клиенте, а не ML | прод (DSP) |
| 27 | Виртуальные фоны | **СЛЕДЫ** | Настройка `VIRTUAL_BACKGROUNDS` — `feature/ucim/ivcs-rest-api/src/commonMain/kotlin/su/ivcs/restapi/model/MapEntryRestDTOSystemSettingsConstantsString.kt` L707974 (энум сгенерирован из OpenAPI ivcs-сервера); строка UI `feature/webrtc/rtc-core/src/commonMain/kotlin/ivac/l10n.kt` L828274 «Виртуальные фоны» | Настройка в API ВКС существует, но **ни в `ivcs-server`, ни в KMP нет кода сегментации/наложения фона**. Реализация — в веб-клиенте (не мой корпус). В `ivcs-server.xml` строка `VIRTUAL_BACKGROUND` не встречается вовсе — вероятная причина: `SystemSettingsConstants.java` сжат до комментариев | **не подтверждено** |

## 4. Ключевое: внешний сервис ИИ — IVA Terra

Это ответ на вопрос «где живёт интеллект».

**Адрес (стенд разработки), файл конфигурации:**
`service/src/deb/ivcs-server-dev/usr/share/ivcs-server/sql-scripts/dev/07_stands_specific_settings.sql`,
строки ~726328–726342:

```
speech_recognition_service                    = 'IVA_TERRA'
speech_recognition_api_url                    = 'https://10.6.19.23:9001/process'
speech_recognition_api_post_process_url       = 'https://10.6.19.23:9001/minutes'
speech_recognition_api_text_translation_url   = 'https://10.6.19.23:9001/translate_to_any'
speech_recognition_api_key                    = <зашифрованное значение, здесь не привожу>
event_subtitles                               = 'true'
event_subtitles_langs                         = af,ar,az,bn,ca,cs,de,en,es,et,fa,fi,fr,gl,gu,he,hi,hr,
                                                id,it,ja,ka,kk,km,ko,lt,lv,mk,ml,mn,mr,my,ne,nl,pl,ps,
                                                pt,ro,ru,si,sl,sv,sw,ta,te,th,tl,tr,uk,ur,vi,xh,zh  (52 языка)
```

Один хост, три ручки — распознавание, «минутки» (протокол/поручения/конспект) и перевод.
На проде адрес задаётся администратором в настройках домена; в снимке прод-значения нет.

**Как устроен путь звука (важно для оценки трудоёмкости любых доработок):**

1. `ivcs-server` не касается аудио. Он собирает `SpeechRecognitionSettings` (id системы, URL, прокси,
   токен, максимальная длительность фразы, id сессии) —
   `server/src/main/java/su/ivcs/media/util/MediaDataHelper.java` : `getSpeechRecognitionSettings()`,
   и отдаёт их **медиасерверу**: `server/src/main/java/su/ivcs/media/business/impl/MediaConferenceServiceBean.java`
   : `updateMediaConferenceParameters()` → `mediaService.setConferenceParameters(...)`.
   Параметры `online=<субтитры активны>` / `offline=<стенограмма активна>` дописываются в URL.
2. **Аудио в Terra отправляет MCU, и его исходников в корпусе нет.** Это отдельный maven-артефакт
   `su.ivcs.services.media:api` со `scope=provided` (`gwt/vcs/pom.xml`, L355980). Отдельный репозиторий.
3. Обратно фразы приходят колбэком в `MediaCallbackServiceListener` → `writeTranscript(...)`.

**Формат обмена (реверсится по коду, без доступа к Terra):**

- Онлайн-фразы — JSON-массив, поля `id, text, timestamp, timestamp_float, duration, conference_id,
  participant_id, post_ids, status`. Пример прямо в шапке
  `server/src/main/java/su/ivcs/videoconference/model/transcription/TerraIvaTranscriptionResult.java`.
- Постобработка: `GET /minutes/{transcriptionSessionId}?token=..&name=<имя мероприятия>`. Пока считает —
  отдаёт JSON-статус (`TerraTranscriptionStatus`: `IN_QUEUE`/`RUNNING`), сервер перепланирует опрос
  quartz-джобом `TranscriptionPostProcessingJob` каждые 5 с, максимум 10 минут
  (`su.ivcs.transcription.postProcessing.maxDurationInMinutes`). Готовый результат отдаётся **XML'ом**
  — в коде это прокомментировано буквально: `// final result is presented as xml - thanks for terra`
  (`TranscriptionServiceImpl.java` L~489790).
- Поручения и конспект приезжают **не отдельными полями**, а как реплики двух служебных участников
  с зарезервированными id — `IVA_TERRA_ASSIGNMENTS_PARTICIPANT_ID` и `IVA_TERRA_SUMMARY_PARTICIPANT_ID`
  (`TranscriptionServiceImpl.java` L489871, L489880). Хрупкий контракт, стоит держать в голове.
- Перевод: `POST /translate_to_any`, тело — `TerraTranslationRequest` (`payload[]` с `start/end/
  participantId/participantName/fromLang/toLang/content`, плюс `token`).
- Есть поддержка HTTP-прокси до Terra (`SPEECH_RECOGNITION_PROXY_URI`) и флаг
  `su.ivcs.transcription.postProcessing.trustAllTlsCertificates` — то есть Terra ставится в закрытый
  контур, и это предусмотрено.

## 5. Что НЕ найдено и где доискать в живом коде

**Не найдено в снимке вообще (ни строки):** `LLM`, `GPT`, `openai`, `GigaChat`, `YandexGPT`, `prompt`
(в ИИ-смысле), `whisper`, `vosk`, `kaldi`, `mediapipe`, `tensorflow`, `tflite`, `onnx`, `torch`,
`embedding`, `qdrant`/`milvus`/`faiss`, `диаризац`/`diariz`, `эмоц`/`emotion`, `детекция лиц`/`face
detection`, `жест`/`gesture`. В `pom.xml` `ivcs-server` — ни одной ML-библиотеки (единственное
«медийное» — внешний бинарь `ffmpeg-static` для превью и склейки файлов). В KMP `build.gradle.kts` —
тоже ни одной; `OpenCV`/`JavaCV` в `feature/chat2/impl` используется **только** для захвата камеры при
записи видео-заметок на desktop (`VideoNoteRecorder.jvm.kt` : `OpenCVFrameGrabber`), никакого зрения.

Где доискать (по убыванию ценности):

1. **Сам сервис IVA Terra** — здесь весь ИИ. В корпусе есть `repomix/terra-core.xml` (11 МБ), он **не
   входил в мою зону**; проверить, тот ли это Terra, — первое, что стоит сделать.
2. **Медиасервер MCU** — артефакт `su.ivcs.services.media:api`, отдельный репозиторий, в снимках его
   нет. Именно он гонит аудио в Terra и может содержать VAD/шумодав/сегментацию.
3. **Бэкенд мессенджера** (ручка `POST {uploads}/{uploadId}/transcribe`) — где-то в `iva-one`/`rn`,
   не мой корпус. Там же ответ на «какой ASR расшифровывает голосовые в чате — Terra или другой».
4. **Веб-клиент ВКС** — рендер живых субтитров и **виртуальные фоны** (сегментация в браузере).
   Настройка `VIRTUAL_BACKGROUNDS` в API есть, кода нигде в моей зоне нет.
5. **`SystemSettingsConstants.java` в живом репозитории** — в снимке он выпотрошен компрессией; там
   полный список настроек, включая, вероятно, ИИ-относящиеся, которых я не увидел.
6. **Живой прод-конфиг домена** — чтобы узнать реальные адреса Terra и какие фичи включены у заказчиков
   (`EVENT_TRANSCRIPT`, `EVENT_TRANSCRIPT_SUMMARY`, `TRANSCRIPT_TRANSLATION`, `EVENT_SUBTITLES`).
7. **KMP: рендер субтитров.** Парсер `TranscriptionPhrase` в `core/centrifuge` есть и покрыт тестами,
   а потребителя я не нашёл — либо фича в работе, либо потребитель появится позже. Проверить в живом
   репозитории поиском использований `McuConferenceSessionStatePush.TranscriptionPhrase`.

## 6. Неожиданное

- **Поручения из встречи («протокол → поручения») уже в проде**, отдельным файлом, с переводом на
  выбранные языки и с ограничением по лицензии. В презентации это выглядит как план, в коде — как
  работающая функция минимум с 2024 года (дата в примере JSON — июль 2024).
- **ИИ-функции лицензируются отдельно и поштучно**: лимиты «сколько мероприятий одновременно со
  стенограммой» и «сколько с субтитрами» живут и в лицензии, и в подписке, с отдельными кодами ошибок
  и записью в аудит. Это готовая коммерческая упаковка, а не техническая заглушка.
- **Система распознавания заменяемая**: кроме `IVA_TERRA` в коде фигурируют `STC` (ЦРТ) и `YANDEX`.
  Но поручения/конспект/перевод завязаны на Terra жёстко — при других вендорах остаётся только голая
  расшифровка (`isSpeechRecognitionServiceIsIvaTerra()`).
- **Перевод субтитров идёт по каждой фразе синхронно с речью** — на каждую фразу для каждого языка
  отдельный HTTP-POST в Terra. Заявленные 52 языка субтитров упираются в этот сетевой путь, что
  выглядит как узкое место при больших мероприятиях.
- **`server/src/main/java/su/ivcs/videoconference/ejb/impl/TranscriptionServiceImpl.java` — файл
  примерно на 800 строк, делающий всё сразу**: старт/стоп, лицензии, HTTP-клиент к Terra,
  quartz-планирование, генерация DOCX, работа с ФС. Если задача выльется в доработку — она вся упрётся
  в этот класс.
- **Ложные срабатывания, которые легко принять за ИИ (перепроверил, это не ИИ):**
  - `TranslationDirection`, `interpreterLanguagesPair`, `SIMULTANEOUS_INTERPRETATION` — это **живой
    синхронный переводчик-человек** в мероприятии, не машинный перевод. Более того, субтитры и
    синхронный перевод **взаимоисключающие**: `ConferenceSessionControllerImpl.startSubtitling()`
    кидает ошибку при `SIMULTANEOUS_INTERPRETATION`.
  - `WebinarTranslationInfoDTO`, `LiveTranslationRestDTO`, `use_flv_translation_for_*` — «translation»
    здесь в смысле **трансляции** (вещания), не перевода.
  - Каталог `AI common/skills/**` в `kmp.xml` и `AGENTS.md` в `ivcs-server` — это инструкции для
    ИИ-помощников разработчиков (Claude Code/Cursor), к продукту отношения не имеют.
  - `api/bot-api-doc/` в `ivcs-server` — «IVA Chat Bot API» для сторонних ботов. Сам по себе не ИИ, но
    это **готовая точка подключения** внешнего ассистента в чат, если такая задача возникнет.
  - `feature/smart_apps` в KMP — WebView-контейнер для мини-приложений, никакого ML.
  - `load-runner` — чистая нагрузочная генерация SIP/H.323/RTP, ИИ-следов ноль.