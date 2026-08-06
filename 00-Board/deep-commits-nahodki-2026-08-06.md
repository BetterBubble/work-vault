---
title: Глубокий поиск по 216 161 коммиту — неожиданные заделы для заявки на льготу
type: report
status: draft
created: 2026-08-06
tags:
- board
- инновации
- explore
permalink: tacticum/00-board/deep-commits-nahodki-2026-08-06
---

# Глубокий поиск по корпусу коммитов — что нашлось сверх аудита

**Источники:** `~/tacticum/helm/data/real/git/all_commits.csv` (216 161 коммит, 233 репо,
2003-06-06 … 2026-07-04) и `repo_activity_all.csv` (233 строки). Два потоковых прохода
регулярками (en+ru, `re.I`, границы слов) по полю `subject`. Аудит
[[audit-intellektualnyh-fich]] и [[recon-ai-corpus-2026-08-03]] прочитаны до старта — темы,
уже закрытые там (Terra/ASR/диаризация/перевод/TTS, DeepFilterNet, virtual bg, ADP/vLLM,
OpenSearch, GigaChat), здесь не повторяются, кроме мест, где нашёлся **новый угол**.

---

## 1. Главное: чего в аудите нет вообще

| # | Находка | Репозитории | Коммитов | Свежесть | Что это значит |
|---|---|---|---:|---|---|
| **1** | **Детекция дипфейков в ВКС** | `desktop/ivcs` (5), `ivcs/ivcs-server` (1), `web/iva-connect` (1) | **7** | **2026-04-10** | Ветка `VCSDESK-7122_deep_fake_bl` («бизнес-логика»), `VCSDESK-7128_indication`, сигнал **`deepfakesDetected`**, «метод учёта новых дипфейков», отложенное уведомление, попап на сервере (`VCSWEB-7727`), нотификации в вебе (`VCSWEB2-9140`). **Сквозной контур: сервер → десктоп → веб.** В аудите этого нет ни строкой |
| **2** | **ML-инференс на отечественном железе и ОС** | `webrt-sdk/ivawebrtc`, `webrt-sdk/iva-virtual-bg` | **10 Baikal + 12 Aurora** | **2026-06-16** | Сборки WebRTC-SDK (внутри которого DeepFilterNet и MediaPipe-сегментация) под **Baikal-M** (`imp-1072`, `imp-1102`, `imp-1183`, `imp-1669`, `imp-1683 «moved baikal to astra/arm»`) и под **ОС Аврора** (`imp-1091 add aurora support`, 2022-10-04 → `hotfix: aurora deepfilternet dep`, 2026-06-16). То есть нейросетевой шумодав крутится на российском ARM-процессоре и российской мобильной ОС |
| **3** | **DLP-контур с блокировкой «в разрыв»** | `iva/one/backend/{chats,users,admin-api-gateway}`, `ivcs/ivcs-server`, `web/iva-admin`, `imail/desktop/ivaplugins`, iOS, KMP | **25+** | **2026-07-02** | Интеграция с **InfoWatch** (`feat/10732-dlp-infowatch`), проверка контента синхронно в разрыв отправки/редактирования сообщений и ботов, **массово-параллельные проверки** (`VCSWEB-7748`), проверка тела REST-запросов (`VCSWEB-7545`), аудит DLP-проверок, политики безопасности в админке. Самая горячая тема июня 2026 |
| **4** | **Постквантовая криптография — была и снята** | `aves/external-java-libraries` | **1** | **2026-05-26** | `Drop frodo crypto support` — FrodoKEM (постквантовый KEM, финалист NIST) поддерживался в вендоренном java-стеке и удалён. Единственный след в корпусе; **как задел заявлять нельзя — это откат, а не наработка** |
| **5** | **Групповой simulcast с per-layer кап-контролем битрейта** | `iva-m/android/kmp`, `ivasdk/webrtc-kmp`, `mobile/ucim-android`, `desktop/ivcs` | **85** | **2026-06-28** | Не просто simulcast: `RtpSender.setSimulcastMaxBitrates — per-layer битрейт групп (iva21)`, `mungeSdpForSimulcasting` (SSRC-munge), `gate simulcast on isGroup (1:1 single-layer)`, кросс-платформенная `getSimulcastStats`. Собственный форк WebRTC с изменённой логикой адаптации потоков — **живая инженерия июня 2026**, в аудите не упоминалась |
| **6** | **LRGWEB — веб-интерфейс терминала Largo, живой проект** | `web/iva-admin` (221), `fork/sbc_certified/iva-admin` (77) | **298** | **2026-07-02** | Отдельный Jira-проект `LRGWEB` (тикеты до №262). Содержание: RTP MTU, PCAP-съём трафика с терминала, список подключённых камер, TLS/NET-события по вебсокету, LDAP, обновления прошивки, диагностика целостности. Плюс в `infrastructure/ansible`: `Add largo-web-ui repository` (2025-05-20) и **`Say bye-bye to largocodec and largocodec-orel repositories`** (2026-03-05) |

---

## 2. Темы задания — результаты по каждой

### 2.1 Видео и кодеки

| Маркер | Хитов | Топ-репо | Посл. |
|---|---:|---|---|
| `codec` | 240 | `modules/media` 35, `modules/sdp` + `fork/.../sdp` 26+26, `ivcs-server` 25 | 2026-01-28 |
| `simulcast` | **85** | `desktop/ivcs` 37, `mobile/apple/messenger` 11, `ucim-android` 8, `kmp` 7 | **2026-06-28** |
| `bitrate` / `битрейт` | 115 / 30 | `modules/media` 44, `ivcs-server` 12, `ivawebrtc` 12 | 2026-06-26 |
| `h264` | 65 | `modules/media` 19 | 2025-12-26 |
| `h265` / `hevc` | 11 / 2 | `modules/sdp` 4 | 2026-01-28 |
| `nack` / `fec` / `pli` | 17 / 14 / 6 | `modules/media`, `modules/sdp` | 2025-10-15 |
| `jitter` / `congestion` | 12 / 12 | `desktop/ivcs`, `modules/media`, `rtpgw` | 2025-07-07 |
| `потер*` (потери) | 141 | `mobile/apple/messenger` 53, `desktop/ivcs` 30 | 2026-07-02 |
| **`av1`** | **0** | — | — |
| `vp9` | **1** | `ivcs-server` | 2026-01-28 |
| `packet loss` | 2 | `ivcs-server` | 2019 |
| `concealment` / `frame drop` / `bandwidth estimation` | **0** | — | — |

**Вывод.** Кодеки — H.264/H.265/VP8, классика. **AV1 нет вообще, VP9 упомянут один раз.**
Своих алгоритмов сокрытия потерь (concealment), собственной bandwidth estimation в текстах
коммитов не видно. Единственное, что тянет на инженерный задел, — **групповой simulcast с
управлением битрейтом по слоям** (см. §1.5). Ложняк: `gcc` (50 хитов) — это компилятор в
`qtwebengine`, не Google Congestion Control.

### 2.2 Компьютерное зрение

| Маркер | Хитов | Топ-репо | Посл. |
|---|---:|---|---|
| `mediapipe` | 3 | `ivasdk/webrtc-kmp` 2, `ivawebrtc` 1 | **2026-07-03** |
| `tflite` / `onnx` | 3 / 1 | `webrt-sdk/iva-virtual-bg` | 2026-03-06 |
| `segmentation` | 3 | `webrtc-kmp` 2, `ivawebrtc` 1 | 2026-07-03 |
| `blur` / `размыт` | 23 / 7 | `iva-virtual-bg` 5, `ivawebrtc` 3, `webrtc-kmp` 3, `desktop/ivcs` 7 | 2026-06-16 |
| `face` / `landmark` / `gaze` / `pose` / `emotion` | **3 / 1 / 0 / 0 / 0** | ложняки (`Faces` шрифты, `landmark` в Keycloak) | — |
| `framing` / `кадрирован` | 2 / 1 | `id/server` (не про камеру), `iva-connect` 1 | 2025-08-01 |
| `gesture` / `жест` | 24 / 68 | почти всё — жесты тач-интерфейса | 2026-05-12 |
| `opencv` | 1 | `autotest3/autocore` (тест-фреймворк) | 2024-05-03 |

**Вывод.** CV в корпусе — **ровно одна задача: сегментация человека для виртуального фона**
(MediaPipe Selfie Segmentation, миграция tflite→onnx, GPU-ускорение). **Ни распознавания лиц,
ни автокадрирования, ни трекинга взгляда, ни эмоций.** Единственное исключение — детекция
дипфейков (§1.1), но по текстам коммитов **не видно, где живёт сам детектор** — в клиентах
только приём сигнала и индикация.

### 2.3 Аудио-ML

| Маркер | Хитов | Топ-репо | Посл. |
|---|---:|---|---|
| `deepfilter` / `rnnoise` | 5 / 4 | `ivawebrtc`, `desktop/ivcs`, `webrtc-java` | **2026-07-03** |
| `шумоподав` | 37 | `web/iva-connect` 30, `desktop/ivcs` 7 | 2025-12-17 |
| `aec` / `agc` / `эхо` | 7 / 10 / 8 | `ivawebrtc`, `kmp`, `ucim-android` | 2026-06-22 |
| `vad` | 7 | `modules/media` 5 | 2026-06-24 |
| `diariz` | 4 | `terra-core` | 2026-03-03 |
| `wake word` / `voiceprint` / `mos` | **0 / 0 / 0** | — | — |

**Вывод.** Всё сходится с аудитом; нового нет, кроме **порт-кита `iva-ns` (DFN/RNNoise) для
Android-seam** (`ivasdk/webrtc-java`, 2026-07-03) и **сборки под Аврору** (§1.2). Wake-word,
голосовых отпечатков и объективной оценки качества речи (MOS) в корпусе нет.

### 2.4 Железо и ускорители

| Маркер | Хитов | Репо | Посл. |
|---|---:|---|---|
| **`baikal`** | **10** | `ivawebrtc` 8, `iva-virtual-bg` 2 | **2024-09-05** |
| `astra linux` | 43 | `desktop/ivcs` 31, `ivcs-server` 2, `infrastructure/ansible` 2, `docs/docs-sbc` 2 | 2025-12-03 |
| `elbrus` | **3** | `ivasw/its/sip_signalling` (`elbrus compilation`, ветка `elbrus`) | 2023-05-04 |
| `gpu` | 35 | `qtwebengine` 16+12, `desktop/ivcs` 3, `ivawebrtc` 2 | 2026-06-05 |
| `arm64` / `aarch64` | 30 / 3 | `qtwebengine` 13, `servercore` 5 | 2025-06-05 |
| `cuda` | **3** | **только `terra/terra-core`** | 2025-12-09 |
| `int8` | 1 | `terra-core` | 2024-04-09 |
| `npu` / `tensorrt` / `openvino` / `quantiz` | **0** | — | — |
| `альт линукс` | 1 | `desktop/ivcs` | 2024-08-05 |

**Вывод.** **NPU, TensorRT, OpenVINO, квантизации — нуля.** CUDA — только в Terra, 3 коммита.
Зато **Baikal-M и Эльбрус реально собираются** (Baikal — с WebRTC-SDK и виртуальным фоном,
Эльбрус — SIP-сигналлинг). Плюс Astra Linux (43) и ОС Аврора (§1.2). Это «отечественная
аппаратно-программная платформа», а не ускорение ИИ.

### 2.5 Largo и терминалы

| Маркер | Хитов | Репо | Посл. |
|---|---:|---|---|
| `lrgweb` | **298** | `web/iva-admin` 221, `fork/sbc_certified/iva-admin` 77 | **2026-07-02** |
| `largo` / `ларг` | 42 / 3 | `iva-admin` 20+18, `ansible` 3, `terra-core` 1 | 2026-03-05 |
| `h323` / `h.323` | 275 / 14 | `voip-signalling-gateway` 70+69, `modules/media` 64 | 2026-06-15 |
| `sip` | 826 | `voip-signalling-gateway`, `jain-sip` | 2026-06-15 |
| `remote control` | **28** | `ivcs-server` 26, `iva-connect` 2 | 2026-01-29 |
| `camera control` / `ptz` / `hdmi` / `пульт` / `ir` | **2 / 0 / 0 / 0 / 0** | — | 2023-03-17 |
| `ivacodec` | **0** | — | — |

**Вывод.** Кода самого терминала Largo в корпусе 233 активных репо **нет** — только его
веб-GUI (`LRGWEB` внутри `iva-admin`) и следы в ansible: репозитории `largocodec` и
`largocodec-orel` **удалены 2026-03-05**, `largo-web-ui` добавлен 2025-05-20. **PTZ, HDMI,
ИК-пульта в текстах нет.** `remote control` (28) — это удалённое управление рабочим столом
при демонстрации экрана (`VCSWEB-5607…5611`, `VCSDESK` `[RDC]`), не управление терминалом.

### 2.6 Поиск и память

| Маркер | Хитов | Репо | Посл. |
|---|---:|---|---|
| `fts` | 86 | **только `imail-mirror/servercore`** | 2026-06-17 |
| `opensearch` / `elastic` | 44 / 41 | `iva-one-helm`, `one/backend/chats` 24 | 2026-06-09 |
| `vector` / `вектор` | 6 / 1 | всё ложняки (`std::vector`, SDP) | — |
| `qdrant` / `faiss` / `rag` / `rerank` / `knn` | **0 / 1 / 0 / 0 / 0** | `faiss` — 1 хит в `terra-core` 2024-02-14 | — |
| `semantic` | 37 | всё «семантика SDP/протокола» | — |

**Вывод.** Подтверждает аудит: **векторного поиска и RAG в поставляемом коде нет.** Новое —
**86 коммитов FTS в `imail-mirror/servercore`**: у почтового сервера есть собственный
полнотекстовый поиск, и это стоит сверить с обещанием заказчику «поиск по документам в
III кв. 2026» (в аудите сказано, что серверного поиска в почте нет вообще — вывод делался
по клиенту `jump`, а не по `servercore`). **Расхождение, требующее проверки.**

### 2.7 Спутник и сети

| Маркер | Хитов | Топ-репо | Посл. |
|---|---:|---|---|
| **`satellite` / `спутник` / `starlink` / `low orbit`** | **0** | — | — |
| `sfu` | 457 | `iva-connect` 152, `desktop/ivcs` 142, `apple/messenger` 115 | 2026-07-02 |
| `mcu` | 454 | `iva-one` 71, `kmp` 65 | 2026-07-03 |
| `quic` | 452 | `qtwebengine` 330 (Chromium upstream), `id/server` 30, `desktop/ivcs` 24 | 2026-07-02 |
| `turn` / `stun` / `relay` | 403 / 21 / 38 | `modules/turn`, `sbc-cfg-server`, `ivcs-server` | 2026-06-09 |
| `p2p` | 212 | `apple/messenger` 65, `iva-connect` 37 | 2026-06-30 |
| `latency` / `rtt` / `задержк` | 4 / 2 / 56 | `rtpgw`, `apple/messenger` 25 | 2026-05-29 |

**Вывод.** **Спутниковой темы нет ни в одном из 216 161 коммита.** Заявлять нечего.
QUIC — почти весь из вендоренного Chromium, свой транспорт на QUIC не просматривается.

### 2.8 AutoML / ML-платформа

| Маркер | Хитов | Репо |
|---|---:|---|
| `fedot` / `automl` / `mlflow` | **0 / 0 / 0** в корпусе ИВА | — |
| `whisper` / `torch` / `inference` | 3 / 3 / 9 | `terra-core` |
| `vllm` / `llm` | 2 / 5 | `blank130/*`, `kmp` 1 |
| `finetune` / `дообуч` | 15 / 0 | всё ложняки (`fine-tuning of layout`) |

**Вывод.** Ни AutoML, ни MLOps-платформы, ни следов дообучения моделей. **Обучения моделей
у заказчика в корпусе не видно вообще** — только инференс готовых весов.

---

## 3. `fedot_gardener` — проверено, к заказчику отношения не имеет

`repos_registry.csv:2` помечен `external (внешний github)`,
URL `https://github.com/dsolonko/fedot_gardener.git`. Метрики реестра: **123 коммита, 5
участников, 351 файл**, Python 108 / Shell 31, диапазон **2025-06-07 … 2026-05-26**, 31 ветка,
13 тегов, 71 релиз. Авторы: dmitry solonko (43), Dmitry Solonko (26), dsolonko (22),
vorotovalexey (19), Taisia Zykova (8). **В `all_commits.csv` его нет** (grep по `fedot` — 1
хит, и тот не из этого репо; ни один из 233 активных репозиториев GitLab заказчика не
называется `fedot_gardener`).

**Вопреки названию, FEDOT (AutoML ИТМО) там ни при чём** — по разбору снимка в
[[recon-ai-corpus-2026-08-03]] это LLM-приложение про уход за растениями (LangChain + Gemini
via Vertex AI + DeepSeek + Langfuse). **В заявку не годится: это не актив заказчика.**

---

## 4. Репозитории, которых нет в аудите

| Репо | Коммитов | Контриб. | Посл. | Что это |
|---|---:|---:|---|---|
| `imail-mirror/servercore` | **7292** | 10 | **2026-07-04** | Ядро почтового сервера (C++). **86 коммитов FTS**, QUIC 12, keyword 6. Самый активный репо корпуса и **не разобран ни в одном отчёте** |
| `ivasw/ism/ics3` | 4211 | 20 | 2026-07-04 | Телефонная платформа (SWM). `кодек` 27, `голос` 36, `лиц*` 11 |
| `blank60/disaster` | 181 | 1 | 2026-06-21 | **Disaster Recovery для MCU**: репликация БД (`MEDIA_TABLES_EXCLUDE`), lsyncd-синхронизация filestorage, `lr_recovery.py`, `ivcs-lr-post-upgrade`, сборка **Nuitka**, `LR_MONITORING.md`. Персональный репо (Паркин Борис). Не ИИ, но реальный инженерный задел по отказоустойчивости |
| `aves/external-java-libraries` | 137 | 1 | 2026-05-28 | Вендоренный java-стек с **бэкпортами CVE-фиксов** (CVE-2026-5588, CVE-2026-42198), `Drop frodo crypto support` |
| `aves/postinstall` | 6 | 1 | 2026-06-05 | Postinst-скрипты под **Astra Linux** (`/etc/astra/build_version`) |
| `live/ivcs` · `live/media` · `live/load-runner` | 1127 · 545 · 73 | 7 · 4 · 2 | 2026-06-30 | **Live-дистрибутив продукта** («Implement Live System for IVCS», с 2017): загрузочный образ, `Switch base distro to debian trixie` (2026-04-07), релизный конвейер 3.240 / 28.12 |
| `sgaidukov/its_utils` | 197 | 1 | 2026-06-03 | Личные devops-утилиты, **`agents.md` обновляется** — след работы с ИИ-агентами |
| `blank23/web_ui_mcu_learning` | 186 | 3 | 2026-06-03 | **Ложный след:** «learning» здесь не ML. Коммиты вида `test_47`, `fix_27`, `test_54` — учебный репозиторий по UI-тестам MCU |
| `blank150/request-docs` | 14 | 2 | 2026-06-23 | Прокси/пермишены для WS-подписок (IVAONE) |
| `inartific/iva-outlook-plugin` | 223 | 13 | 2026-06-02 | Второй Outlook-плагин рядом с `web/iva-outlook-plugin` (297) |

### Что такое `aves` по коммитам
30 хитов `\baves\b` в 6 репо. `AVES add 384kBit` (`ivcs-server`, 2021-03-31),
`[VCSWEB-6550] Remove "IVA MCU" phrase from i18n to make it more suitable for IVA Aves`
(2025-05-27), `Drop aves support` (`infrastructure/ansible`, 2024-10-21), `Hide /aves/source
location with password in debian-repository` (2026-04-09), тесты `aves` в
`autotest/mcu-ui/restapitests`. **Отдельной кодовой базы AVES в корпусе нет** — это вариант
поставки на общем коде MCU (`ivcs-server`) плюс сборочная обвязка в группе `aves/`.
**Для заявки по продукту AVES это существенно: технический задел придётся брать из `ivcs-server`.**

---

## 5. НАХОДКИ ДЛЯ ЗАЯВКИ

Ранжировано по неожиданности и по тому, насколько тянет на «инновационную фичу».

1. **Детекция дипфейков в видеоконференции** — сквозной контур сервер→клиенты, март–апрель
   2026, отдельные Jira-задачи. Формулируется как «обнаружение синтетически подменённого
   изображения участника в реальном времени». **Обязательно доуточнить, где детектор** —
   в корпусе видна только доставка сигнала и индикация.
2. **Нейросетевой инференс на отечественной аппаратно-программной платформе** — DeepFilterNet
   (шумоподавление) и MediaPipe-сегментация (виртуальный фон) собраны и работают на
   **Baikal-M / Astra Linux / ОС Аврора**. Это редкое сочетание «ML + импортозамещённое
   железо», и оно доказывается коммитами 2022→2026.
3. **Групповой simulcast с покадровым управлением битрейтом по слоям** — собственный форк
   WebRTC (`ivasdk/webrtc-kmp` `iva fork:` ветка), SDP-munging, кросс-платформенная
   статистика по слоям. Живая разработка июня 2026.
4. **DLP-контур с синхронной блокировкой контента «в разрыв»** — на всех клиентах, включая
   сообщения ботов и тело REST-запросов, с массово-параллельной проверкой. Июнь 2026.
5. **Disaster Recovery для ВКС** (`blank60/disaster`) — репликация, восстановление,
   post-upgrade, Nuitka-сборка. Не ИИ, но отвечает на «непрерывность сервиса».
6. **Полнотекстовый поиск в ядре почтового сервера** (86 коммитов, `servercore`) — если он
   серверный, это меняет картину по обещанию «поиск по документам в III кв. 2026».

### Чего в коде нет — не заявлять
**Спутниковая связь (0), AV1 (0), NPU/TensorRT/OpenVINO/квантизация (0), wake-word и голосовой
ассистент (0), распознавание лиц / эмоций / позы / взгляда (0), автокадрирование (0),
жестовый язык (0), AutoML/MLflow/FEDOT (0), блокчейн (0), обучение и дообучение моделей (0),
RAG и векторный поиск (0/1), PTZ/HDMI/ИК-пульт (0).**

---

## ВЕРДИКТ

Шесть заделов, которых нет в аудите; два из них — дипфейк-детекция и ML на Baikal/Аврора —
прямо ложатся в рамку «инновационная фича». Одновременно закрыты семь тем задания как
пустые: спутник, AV1, NPU-ускорители, голосовой ассистент, лицевая биометрия, AutoML,
управление терминалом. Кода терминала Largo в корпусе нет — есть только его веб-GUI
(298 коммитов, живой), а репозитории `largocodec` удалены в марте 2026. AVES отдельной
кодовой базы не имеет.

**Проверено**
- Полный проход по `all_commits.csv` (216 161 строка) двумя скриптами, ~90 регулярных
  выражений по 8 темам задания + 22 дополнительным темам.
- Реестр `repo_activity_all.csv` (233 репо) — сверка групп, авторов, активности, дат.
- `repos_registry.csv` — статус `fedot_gardener`.
- Точечные выборки коммитов по каждой значимой находке (дипфейк, Baikal, Эльбрус, Аврора,
  DLP, LRGWEB, simulcast, AVES).

**Данные**
216 161 коммит · 233 активных репо · диапазон 2003-06-06…2026-07-04 · 30 просканированных тем.
Ключевые числа: дипфейк **7** коммитов / 3 репо / посл. **2026-04-10**; Baikal **10** / 2 репо;
Аврора **12** продуктовых (13 хитов `id/server` — AWS Aurora Postgres, ложняк); Эльбрус **3**;
Astra Linux **43**; DLP **25+** / 8 репо / посл. **2026-07-02**; LRGWEB **298** / посл.
**2026-07-02**; simulcast **85** / посл. **2026-06-28**; FTS в `servercore` **86**;
CUDA **3** (только Terra); satellite **0**; AV1 **0**; NPU/TensorRT/OpenVINO **0**;
wake-word **0**; qdrant/rag **0**; fedot в корпусе ИВА **0**.

**Подтверждение (команды)**
```
python3 scan.py                       # два потоковых прохода, см. scratchpad/scan.py и out.json
grep -i "baikal\|байкал" all_commits.csv | cut -d, -f1,3,7-
grep -iE "deepfake|дипфейк|deep.?fake" all_commits.csv | cut -d, -f1,3,7-
grep -iE "\bdlp\b|data ?loss|утечк" all_commits.csv | cut -d, -f1,3,7- | sort -t, -k2
grep -i "lrgweb" all_commits.csv | cut -d, -f1,3,7- | sort -t, -k2 | tail -25
grep -iE "aurora|аврор" all_commits.csv | cut -d, -f1 | sort | uniq -c | sort -rn
grep -iE "largo|ларг" all_commits.csv | grep -vi lrgweb | cut -d, -f1,3,7-
grep -iE "elbrus|эльбрус" all_commits.csv | cut -d, -f1,3,7-
grep -E "^(aves|inartific|live|blank[0-9]+)/" repo_activity_all.csv | cut -d, -f1,4,5,7,8,9
grep -iE "fedot|gardener" repos_registry.csv
```

**НЕ проверено**
1. **Где живёт детектор дипфейков.** В коммитах видна только доставка сигнала
   `deepfakesDetected` и индикация. Своя ли модель, чья, где считает — по корпусу
   не восстановить. Смотреть: Jira `VCSDESK-7122`, `VCSDESK-7128`, `VCSWEB-7727`,
   `VCSWEB2-9140`; репозитория с детектором среди 233 активных не видно.
2. **Serena не поднималась** — работа шла по CSV грепом и Python-скриптами, а не по символам:
   кода этих репозиториев локально нет (repomix снят лишь по 9 из 233), символьная разведка
   технически невозможна. Значит вывод «в репо X темы нет» опирается на **тексты коммитов**,
   а не на код.
3. **Только default-ветки и только `subject`.** Тела коммитов и диффы недоступны.
   У `desktop/ivcs` 5556 веток, у `ivcs-server` 968 — незамерженное в данные не попало.
4. **FTS в `imail-mirror/servercore`** — 86 коммитов зафиксированы, но серверный ли это поиск
   и по каким объектам, не проверено. Это прямое расхождение с выводом аудита «серверного
   поиска в почте нет вообще» и его надо снять до подачи заявки.
5. **`largocodec` / `largocodec-orel`** — удалены в марте 2026, их истории в выгрузке нет.
   Если для заявки нужен задел по кодекам терминала, источник придётся искать вне GitLab.
6. **Confluence и Jira не открывались** — только корпус git.