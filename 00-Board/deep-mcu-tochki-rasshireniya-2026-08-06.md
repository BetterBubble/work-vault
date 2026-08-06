---
title: Глубокая разведка ВКС/MCU — точки расширения, медиастек, ML в медиатракте
type: report
status: draft
created: 2026-08-06
tags:
- board
- инновации
- mcu
- explore
permalink: tacticum/00-board/deep-mcu-tochki-rasshireniya-2026-08-06-1
---

# ВКС/MCU: что спроектировано, что занято, что пустует

Роль: explorer, только чтение. Корпус — `repomix/ivcs-server.xml` (37 МБ, снят с `--compress`),
`git/all_commits.csv` (216 161 коммит, включая репозитории, снимков которых нет),
`git/repo_activity_all.csv`, `jira/jira_issues.csv` (по 300 задач на проект — выборка, не полный дамп).
**Ненайденное не доказывает отсутствия**: тела функций местами вырезаны, дамп Jira неполный.

Предыдущее по теме, не повторяю: [[recon-ai-vks-2026-08-03]] (Terra, стенограмма, субтитры,
поручения, конспект, лицензирование ИИ поштучно), [[audit-intellektualnyh-fich]].

## 0. Поправка к рамке задачи

**`SPEECH_RECOGNITION_API_POST_PROCESS_URL` не пустует — он занят.** Это настройка домена,
указывающая на Terra `/minutes`; по ней уже ходит `TranscriptionServiceImpl` за протоколом,
поручениями и конспектом (доказано в [[recon-ai-vks-2026-08-03]], разделы 3 и 4).
Ценность её не в пустоте, а в том, что это **сменный адрес**: контракт «отдай сессию — получи
обработанный текст» вынесен в настройку, и подставить туда свой сервис (наш `helm`-конвейер,
LLM-постобработку) можно без правки кода ВКС. То же верно для `SPEECH_RECOGNITION_API_URL`
и `SPEECH_RECOGNITION_API_TEXT_TRANSLATION_URL`; есть `SPEECH_RECOGNITION_PROXY_URI` и флаг
`trustAllTlsCertificates` — то есть подстановка стороннего сервиса в закрытом контуре
предусмотрена изначально.

Настоящая **незанятая** точка расширения нашлась в другом месте — см. §1.

## 1. ГЛАВНАЯ НАХОДКА: в ВКС уже есть конвейер «кадр участника → внешний CV-сервис»

Это не описано ни в аудите, ни в прошлой разведке. В `ivcs-server` работает **проверка
подлинности участника (детекция дипфейков)** через VisionLabs LUNA Platform.

| Что | Где |
|---|---|
| Клиент к LUNA | `server/src/main/java/su/ivcs/verification/business/LunaPlatformVerificationServiceImpl.java` : `verifyVideoFrameAsync()`, `performVerification()` |
| Запрос | `POST {OBJECT_RECOGNITION_API_URL}/6/sdk` с телом `image/jpeg`, Basic-auth; параметры `detect_face=1`, **`estimate_deepfake=1`**, `multiface_policy=0` |
| Периодический опрос | `server/src/main/java/su/ivcs/videoconference/jobs/authenticity/ParticipantAuthenticityCheckJob.java` — quartz по сессии, берёт **всех участников с включённым видео**, режет на чанки, планирует после коммита |
| Оркестрация и батч | `server/src/main/java/su/ivcs/videoconference/business/ConferenceSessionParticipantAuthenticityCheckServiceImpl.java` : `checkFor()`, `fetchVideoFrameForParticipant()` |
| **Снятие кадра с MCU** | `GET https://{node}/api/rs/media/proxy/api/conference/{mediaConferenceId}/participant/{participantId}/stream/PRIMARY/preview?server=..&time=..`, внутренний заголовок `AUTHENTICITY_CHECK_SERVICE` (гейт — `isInternalAuthenticityCheckRequest()`) |
| Настройки домена | `OBJECT_RECOGNITION_ENABLED`, `OBJECT_RECOGNITION_API_URL`, `OBJECT_RECOGNITION_LOGIN`, `OBJECT_RECOGNITION_PASSWORD`; интервал проверки — на уровне мероприятия, `setAuthenticityCheckInterval()` |
| Стенд | `07_stands_specific_settings.sql` L726375–726378: `http://10.0.206.188:5000`, `root@visionlabs.ai` |
| Результат в UI | `VerificationStatus` (`NONE`/`REAL`/`FAKE`) → колонка «Подлинность» в админке мероприятия, событие `ConferenceSessionParticipantAuthenticityChangedEvent` модераторам, строка «Лицо участника … распознано как ненастоящее» |

**Почему это важно.** Готов весь дорогой обвяз для *любой* компьютерно-зрительной фичи:
периодический джоб по сессии, батчинг, асинхронный `CompletableFuture`-конвейер, снятие JPEG
с MCU по участнику, **вынесенный в настройку адрес внешнего сервиса**, статус на участнике в БД,
рассылка события модераторам, колонка в админке. Заменить LUNA на свою модель — это подмена URL
и парсера ответа. Это и есть настоящая незанятая точка расширения.

## 2. Остальные точки расширения и интеграции

- **Вебхук-шина ботов** — `server/src/main/java/su/ivcs/videoconference/ibus/WebHookBus.java`,
  `ibus/webhook/WebHookTarget.java` (адрес берётся из `BotProfile.getExternalUrl()`),
  `httpPushService.send(...)` с `onSuccess`/`onFailure`, буферизация, системный алерт
  `WebHookConnectionErrorInfo` при недоступности. То есть исходящая доставка событий во внешний
  сервис уже построена и эксплуатируется. Настройки бота — `webhookURL`, `webhookToken`.
- **Внешние адреса в настройках** (все — настройки домена, меняются админом без пересборки):
  `SPEECH_RECOGNITION_API_*`, `OBJECT_RECOGNITION_API_URL`, `EXTERNAL_BILLING_API_URL`,
  `EXCHANGE_GW_EWS_API_URL`, `AVSCAN_ICAP_URL` (антивирус по ICAP), `EXTERNAL_AUTH_*`,
  `TAT_OPENID_AUTH_*`, `LDAP_URL`, `PUSH_PROXY_URL`, `NATS_URL`/`MESSAGEBROKER_URL`,
  `webrtc_screenshare_extension_url`.
- **Prometheus-экспортер** в `ivcs-server` (`createPrometheusExporterServer()`,
  `UsageStatisticsExporterJob`, гейджи вида `ivcs_license_events_with_transcript`) плюс
  встроенная страница мониторинга на chart.js + `chartjs-plugin-datasource-prometheus`.
  Экспортируется **утилизация и лицензии**, не покадровое качество связи.
- **Zabbix** — настройки `zabbix_enabled`, `zabbix_servers`, PSK.

## 3. Медиастек

Исходников медиасервера и SDK в снимках нет — они в отдельных репозиториях. Восстановлено по
реестру репозиториев и текстам коммитов.

| Репозиторий | Объём | Активность |
|---|---|---|
| `modules/media` (MCU) | 3336 коммитов, 5 контрибьюторов | до 2026-06-29 |
| `webrt-sdk/ivawebrtc` (свой форк WebRTC, C++) | 857 коммитов, 18 контрибьюторов | до 2026-06-16 |
| `ivasdk/webrtc-kmp` (`iva-fork`) | 252 коммита | до 2026-07-03 |
| `ivasdk/webrtc-java` (`iva-main`) | 330 коммитов | до 2026-07-03 |
| `ivasw/its/media/rtpgw`, `media_utils`, `rtc`, `modules/turn`, `fork/sbc_certified/turn` | 368/146/11/267/290 | до 2026-06 |

**Кодеки** (по `StreamStatisticItem.getCodec()` и i18n/тестам `ivcs-server`): видео — H.264, H.265
(`modules/media` 02f59c04, 2018-08-07 «Add H265 codec support»), VP8, VP9; аудио — Opus, AAC,
G.711/G.722/G.723/G.726/G.728/G.729, Speex. То есть и WebRTC-, и SIP/H.323-набор.

**Simulcast — есть, своё.** `imp-821` (2021-09-09) «add simulcast support for publishing»,
`imp-910` (2021-11-30) «changing max bitrate of first and second simulcast layers», `imp-1713`
(2025-08-04) «WebRTC API v2 additions: simulcast, codecs filtering», `imp-1959` (2025-09) — фиксы
максимального битрейта. На MCU — «Switch to simulcast video decoder» (2021-10-20).
В `webrtc-kmp` 267fe957 (2026-06-27) `RtpSender.setSimulcastMaxBitrates` — побайерные битрейты.
**SVC/AV1 в найденном не встречается вовсе.** Открытый дефект `IMP-1996` — «webrtc. ios. Не работает
симулкаст на VideoToolbox энкодере».

**Борьба с потерями — есть, частично собственная.**
FEC внедрён в 2018 (`modules/media` 702c1cce «Implement FEC support»), в 2019 добавлена
**собственная схема `IVA FEC`** — `0924065d` «Add support for IVA FEC schema», `4e8a9986` «Always use
OF_RS_2d8 IVA FEC type instead of OF_RS_2d4» (Рид—Соломон), «Enable FEC for H323». Для WebRTC-стека
`ulpfec`/`red` наоборот **принудительно выключены** (07e721fd, 2024-02-01, «Force disable ulpfec and
red codecs for WebRTC stack»), а с 2025-10 через `MediaConferenceParameters` можно включить
**Opus in-band FEC** (278f56c9). NACK и PLI — есть (`nackMode`, `NackMode`, `pliCount`), REMB — есть.

**Адаптация битрейта — своя, не стоковая WebRTC.** «Implement more accurate bitrate estimation for
WebRtcSession» (2024-12-25), «Migrate to new bitrate estimator from imp» (2025-02-07), «Skip bitrate
decrease if real bitrate is less than estimated one», «Do no apply WebRTC abr heuristics if max
bitrate decreased», настраиваемое сглаживание (`bitrate smoothing paced delay`), `abrMode` и
`fecSchema` в SIP/H.323-сессиях, TMMB для VoIP.

## 4. ML в медиатракте прямо сейчас

| Что | Где исполняется | Доказательство |
|---|---|---|
| **DeepFilterNet** (нейрошумодав) | нативно в `ivawebrtc` (десктоп/веб через модуль), с 2026-06 и на Аврору | `imp-1491` 2c84a1dd (2024-02-19) «add deepfilternet noise suppressor»; 20b42604 (2026-06-16) «hotfix: aurora deepfilternet dep» |
| **RNNoise** | там же, включён раньше DFN | `imp-1206` 4d8f542a (2023-02-07) «use RNNoise»; `imp-1950` (2025-08) фикс краша |
| Порт шумодава в Java-SDK | `ivasdk/webrtc-java` | 26d47110 (2026-07-03) «iva-ns: NS engine port kit (DFN/RNNoise sources) for B2/C2» |
| **MediaPipe Selfie Segmentation** → виртуальный фон и блюр | клиент | `imp-976` (2022-02-01) «Virtual background. Use mediapipe»; `imp-911` (2021-11-17) «add real background blur» |
| **Переход с TFLite на ONNX** | клиент | `IMP-2002` «ivawebrtc dll. Переход с tflite на onnx модель» (Закрыт) |
| **Выделенная библиотека инференса `iva-virtual-bg.dll`** | клиент, отдельный компонент | `IMP-2004` (закрыт) «Вынести логику работы с виртуальным фоном в iva-virtual-bg»; классы `VirtualBackgroundFilter`, `ONNXSegmentationBackend` (`IMP-2018`) |
| **GPU-инференс**: препроцесс, инференс, постпроцесс вынесены на GPU | клиент, Windows и Linux | `IMP-2019`/`IMP-2020`/`IMP-2021` (Решено, 2026-04), `IMP-2034` (Закрыт, linux) |
| **Три backend'а исполнения**: WebGPU EP (`IMP-2039`, закрыт), Vulkan (`IMP-2040`, открыт), OpenVINO для Intel (`IMP-2038`, Ready for Dev) | клиент | Jira, июнь 2026 |
| Виртуальный фон в мобиле/вебе — **делается прямо сейчас** | KMP: iOS `BlurVideoProcessor` (Vision+CoreImage), web `canvas + MediaPipe (js/wasmJs)`, автоустановка фабрики | `ivasdk/webrtc-kmp` a0959d32, 7588c2b5, eb977f17, ab5fdbad — все 2026-07-02/03 |
| Виртуальный фон встроен в QtWebEngine/chromium десктопа | десктоп | `IMP-1995`, `IMP-2009`, `IMP-2016`, `IMP-2017` |
| Шумоподавление в «ВКС модуле» | десктоп | `IMP-2013`, **Development in Progress** |

**Вывод по §4: в клиенте ВКС уже стоит полноценный движок нейросетевого инференса на ONNX Runtime
с выбором GPU-бэкенда, асинхронным конвейером кадров и пре/постпроцессингом на GPU.** Используется
он ровно под одну задачу — сегментацию человека для фона. Это самая недооценённая инфраструктура
из найденного.

## 5. Управление камерой

- **FECC (Far End Camera Control) — реализовано**, но только для SIP/H.323-терминалов и только
  из админки: `gwt/.../adminevent/translation/fecc/FarEndCameraControlWidget.java`,
  `WatchParticipantTranslationDialogActivity` шлёт `UP/DOWN/LEFT/RIGHT/ZOOM_IN/ZOOM_OUT`
  → `ConferenceSessionController.sendFeccCommand()` → `MediaConferenceServiceBean.sendFeccComand(command, duration=8, participant)`
  → медиасервер. На MCU — «Implement FECC support» (2018-05-03), параметр `feccDisabled` в VoipSession.
- **Автокадрирования / speaker tracking нет.** Слова `autoframing`, `speaker tracking` в корпусе
  не встречаются.
- **Активный говорящий — детекция по энергии звука, не ML**: `ConferenceFeatureType.ACTIVE_SPEAKER_INDICATION`,
  настройка `LAYOUT_SPEAKER_ACTIVATION_DELAY_MS`, в вебе `StreamVoiceActivityTrackerServiceImpl`.
- Зум в вебе — программное кадрирование мозаики (`ZoomSlider`, `zoomFocusCoordinateX/Y`,
  `zoom_step_size`), не управление оптикой.

## 6. Телеметрия качества связи — есть, и она историческая

Модель на поток — `server/src/main/java/su/ivcs/videoconference/model/StreamStatisticItem.java`:
`codec`, `encryptionEnabled`, `sampleRate`, `channels`, `width`, `height`, `fps`, `actualFps`,
**`jitterMs`, `rttMs`, `totalPacketsLost`, `packetsLostPercent`, `bitrateLimit`, `actualBitrate`**,
`lastRtcpReportDate`, **`fecUsed`, `fecRestoredPackets`, `nackUsed`, `nackProcessedRequests`**.

**Ключевое: у медиасервера есть API истории по временному окну.**
`server/src/main/java/su/ivcs/videoconference/business/ParticipantStatisticServiceImpl.java` :
`download()` →
`GET http://{mediaServerIp}:11452/api/statistics/participant/{mediaServerParticipantId}?from={ms}&to={ms}`,
ответ — список `ParticipantStatisticInfo` с полем `gatherTime` и вложенным массивом потоков.
`getStatistics()` склеивает PRIMARY и SECONDARY, сортирует по времени; выгрузка — CSV
(`converter/ParticipantStatisticInfoCSVConverter.java`). Привязка «участник → медиасервер → id»
хранится в `MediaParticipantHistory`.

То есть **размеченный временной ряд качества связи по каждому потоку уже снимается и уже
выгружается**. Чего нет: долговременного хранилища этих рядов на стороне `ivcs-server`
(в схеме БД `videoconference.media_participant_statistic` только вход/выход, IP, user-agent,
причина отключения — покадровых метрик там нет) и экспорта их в Prometheus.
Срок хранения на медиасервере по снимку **не установлен — не проверял**.

Дополнительно: клиент сам детектирует обрывы медиа —
`gwt/vcs/.../media/MediaConnectionIssueWatcher.java` регистрирует `ConnectionIssueType.MEDIA_CONNECTION`,
переживает перезагрузку через cookie; сервер пишет `/var/log/ivcs-server/statistic/connection_issues.log`
с суточной ротацией. Это готовая **разметка «здесь было плохо»** — то, чего обычно не хватает для
обучения предсказателя деградации.

## 7. Запись мероприятий

- `server/src/main/java/su/ivcs/videoconference/business/ConferenceRecordingServiceImpl.java` :
  `createRecordDocuments()` — файлы MP4 (или FLV по умолчанию, MP3 для аудио) кладутся
  `Document`'ами в папку сессии, имя = название мероприятия + дата старта записи, срок хранения из
  подписки (`getMaxRecordingStorageDays()`), уровень доступа наследуется от сессии. Есть `RECORD_AUTO_START`,
  автостоп-джоб, отдельная «data»-запись (демонстрация), превью (`ThumbnailResource`), HLS и FLV-плеер.
- **Индекса содержимого, глав и таймкодов у записи нет.**
- Но **стенограмма лежит рядом и она таймкодирована**: таблица
  `videoconference.conference_transcript` — `conference_session_id`, `conference_session_participant_id`,
  `phrase`, `phrase_timestamp`, `duration_ms`, `timestamp_float`, `external_id`, индекс
  `(conference_session_id, phrase_timestamp)`. Поиск по фразам в админке —
  `ConferenceTranscriptRepositoryImpl.findAllByParams()`, обычный SQL.
  Ключ связи «запись ↔ фраза» есть (сессия + дата старта записи), **связки в UI нет**.

## 8. Что в работе сейчас (Jira, выборка 300 задач на проект — не полный дамп)

- `IMP-2013` «ВКС модуль. Реализовать функцию шумоподавления» — **Development in Progress**.
- `IMP-2040` «iva-virtual-bg.dll Перенос image processing на Vulkan» — **Открыто**;
  `IMP-2038` OpenVINO для Intel, `IMP-2001`/`IMP-2010`/`IMP-2011`/`IMP-2012` (асинхронное
  декодирование и наложение фона, декодирование из BLOB) — **Ready for Development**.
- `IMP-1996` «webrtc. ios. Не работает симулкаст на VideoToolbox энкодере» — Ready for Development.
- `IMP-2008` «Рассинхрон аудио-видео при включении виртуальных фонов» — Ready for Development.
- `VCSWEB-7531` «Доработка постобработки стенограммы» — In Review (2026-02-24).
- `VCSWEB-6714` / `VCSWEB-6646` «Транскрибирование в звонках чата» — In Review / Analytics&Design.
- `VCSWEB-7881` / `VCSWEB-7810` «Кастомизированный список языков синхронного перевода» — в работе.
- `VCSWEB-7363` «Получение агрегированных данных по активностям в мероприятии» — In Review.

## КАНДИДАТЫ В ЗАЯВКУ

| Фича | Какой задел уже есть | Чего не хватает | Дешевизна |
|---|---|---|---|
| **Предсказание деградации канала и упреждающее снижение качества** | временной ряд `rtt/jitter/loss/fec/nack/bitrate/fps` по каждому потоку, API истории по окну `:11452/api/statistics/participant/{id}?from&to`, CSV-выгрузка, лог `connection_issues.log` как готовая разметка инцидентов, свой оценщик битрейта в MCU (точка приложения решения) | хранилище рядов, обучение модели, петля обратной связи в `abrMode`/`MediaConferenceParameters` | **очень высокая** — самое дорогое (измерение + разметка) уже есть |
| **Вторая ONNX-модель в клиенте** (жесты, поднятая рука, присутствие/отвлечение, качество кадра) | `iva-virtual-bg` с ONNX Runtime, выбором GPU-бэкенда (WebGPU/Vulkan/OpenVINO), пре- и постпроцессом на GPU, асинхронным конвейером кадров; уже встроен в chromium десктопа, портируется в KMP и веб | сама модель и UI-потребитель | **очень высокая** — движок инференса уже в проде |
| **Автокадрирование / speaker framing** | детекция лиц уже вызывается (`detect_face=1` в LUNA), FECC-команды PTZ работают, программный зум мозаики есть, активный говорящий определяется | связать детекцию лица с рамкой кадра; на клиенте — ещё одна модель в тот же конвейер | **высокая** |
| **Любая CV-аналитика по участникам** (пересчёт присутствующих, контроль внимания, ПДн-маскирование лица) | весь конвейер §1: джоб по сессии, батч, снятие JPEG с MCU по участнику, сменный адрес сервиса, статус в БД, событие модераторам, колонка в админке | замена URL и парсера ответа | **очень высокая** |
| **Поиск по записям и переход к моменту** | таймкодированная `conference_transcript` с индексом по (сессия, время), запись MP4 в той же папке сессии, дата старта записи, HLS/FLV-плеер, поиск фраз в админке | связка «фраза → позиция в плеере» и пользовательский UI; для семантики — эмбеддинги | **высокая** |
| **Своя LLM-постобработка стенограммы вместо/поверх Terra** | `SPEECH_RECOGNITION_API_POST_PROCESS_URL` — сменный адрес; контракт «сессия → протокол/поручения/конспект» отработан; наш `helm`-конвейер уже делает transcribe→dialogue→summarize | сервис, отвечающий XML в формате Terra (контракт хрупкий: результат приезжает репликами служебных участников) | **высокая**, но упирается в формат |
| **Исходящие события мероприятия во внешние системы** | `WebHookBus` + `WebHookTarget` + `httpPushService` с ретраями и системным алертом; платформа ботов с `externalUrl` | адресаты за пределами ботов, набор событий | средняя |
| **AV1 / SVC** | нет ничего | всё | низкая — не кандидат |

## ВЕРДИКТ

Ключевая зацепка задачи сформулирована неточно: адрес постобработки текста **занят**, его ценность —
в сменяемости, а не в пустоте. Зато нашлись две неучтённые инфраструктуры, каждая дороже её:
**конвейер «кадр участника с MCU → внешний CV-сервис»** в `ivcs-server` (сейчас — детекция дипфейков
через VisionLabs LUNA) и **движок ONNX-инференса на GPU в клиенте** (сейчас — только виртуальный фон).
Плюс уже снимаемый и выгружаемый временной ряд качества связи с готовой разметкой инцидентов.
Три самые дешёвые фичи в заявку — предсказание деградации канала, вторая модель в клиентском
инференс-движке и CV-аналитика по участникам поверх готового конвейера.

**Проверено:** `ivcs-server.xml` (греп + чтение классов целиком), `all_commits.csv` по 4 медиа-репозиториям
(4775 коммитов), `repo_activity_all.csv`, `jira_issues.csv` по IMP/VCSWEB/VCSMOB/VCSWEB2/IVACS.

**Данные:** файлы и строки указаны по каждому пункту; коммиты — по sha и дате; задачи — по ключу и статусу.

**Подтверждение:** LUNA-конвейер прочитан по трём классам целиком, а не по грепу. Схема
`conference_transcript` и URL статистики MCU взяты из исходников, а не восстановлены по смыслу.

**НЕ проверено:**
1. Исходников `modules/media` (MCU), `ivawebrtc`, `iva-virtual-bg`, `webrtc-kmp` в снимках нет —
   всё по §3, §4 восстановлено **по текстам коммитов и заголовкам Jira**, код не читал.
2. Включён ли `OBJECT_RECOGNITION_ENABLED` у кого-то на проде и лицензируется ли проверка
   подлинности — не смотрел. В снимке только адрес стенда.
3. Срок хранения статистики на медиасервере (`:11452`) — неизвестен. Без него оценка «телеметрия
   историческая» верна для окна, длину которого я не знаю.
4. Дамп Jira — по 300 задач на проект. Список §8 — не полный перечень открытых задач.
5. Веб-клиент ВКС (рендер субтитров, потребитель виртуального фона в браузере) в мой корпус
   не входил.
6. Есть ли в MCU собственный ML (VAD, шумодав на стороне сервера) — по коммитам не видно,
   исходников нет.