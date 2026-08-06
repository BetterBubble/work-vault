---
title: independent-blindspots-2026-08-03
type: note
permalink: tacticum/00-board/independent-blindspots-2026-08-03
tags:
- ai
- аудит
- независимая-проверка
- интеллектуальные-фичи
- explorer
---

# Независимый аудит интеллектуальных фич — слепые зоны (заход с нуля)

status: draft · created: 2026-08-03 · автор: independent (воркер-разведчик)

Независимый проход по постановке Лебедева от 03.08: свой список фич из первоисточника → поиск по
9 снимкам кода + 216 161 коммиту → выводы. Выводов предыдущей команды не читал (`13-Deliverables/`,
`recon-ai-*`, `evidence-*` не открывал). **Оговорка честности:** файл канона
`14-Canon/canon-produktovye-ii-fichi`, который велено было читать, содержит внизу раздел
«Уточнения и карта» с чужим списком фич — я его видел глазами при чтении файла. Свой список я
выводил из таблицы слайда и транскрипта заново; совпадения с тем разделом неизбежны, потому что
источник один — колонка «Действие» и строка IVA GPT.

## 1. Мой список фич из первоисточника (зафиксирован ДО похода в код)

Из транскрипта напрямую: суммаризация сообщений (единственная фича, названная голосом); контекст
«есть Gemma, есть Terra, есть RAG — логика уже внутри». Из слайда (колонка «Действие» + строка
IVA GPT + смежное):

1. Суммаризация сообщений/чатов
2. Конспект встречи (саммари ВКС)
3. Субтитры (распознавание речи, реальное время)
4. Перевод (субтитров/речи; отдельно — перевод сообщений)
5. Протокол → поручения (экшн-айтемы из встречи)
6. Письмо → задача, память переписки
7. Корпоративная память коммуникаций (RAG)
8. Агент действий в One
9. Боты и агенты в чатах
10. Задачи создаёт агент из встреч, писем и чатов
11. Документ как источник действия + поиск по содержимому (Диск)
12. SmartApp/мини-приложения + маркетплейс агентов
13. Коннекторы 1С/CRM/ЭДО/сервис-деск для действий агента
14. Разрешённые модели (контроль, какие LLM можно)
15. Маскирование ПДн в ИИ-контуре
16. Журнал ИИ (аудит обращений к моделям)
17. Единый ИИ-слой (IVA GPT / Terra как платформа, включая TTS)

## 2. Метод и числа

- Корпус: 9 repomix-снимков (iva-one, kmp, ivcs-server, rn, jump, terra-core, iva-outlook-plugin,
  load-runner, fedot_gardener), суммарно ~190 МБ. Тела функций местами вырезаны — «не найдено»
  значит «не найдено в снимке», не «нет в продукте».
- ~25 rg-свипов паттернами: AI-ядро (openai|llm|vllm|ollama|gemma|gigachat|deepseek|anthropic),
  суммаризация (summariz|суммариз|саммари|конспект), ASR (whisper|speech-to-text|transcri|
  распознаван|стенограм), субтитры, вектор/RAG (embedding|qdrant|milvus|faiss|meilisearch|RAG),
  ассистент (assistant|copilot|gpt), smartapp, chatbot|bot-api, action-item|поручени, маскирование
  (маскирован|anonymiz|PII|pseudonym), журнал ИИ, allowed-models, коннекторы (bitrix|1c|ЭДО|CRM),
  полнотекст (opensearch|elastic), marketplace, translate-message, tts, diariz.
- 6 глубоких чтений: TranscriptionServiceImpl (ivcs), assistant-модули rn (client.ts, types.ts,
  assistantBriefWithTask.ts, assistantDraftResponse.ts), docker-compose.ai-gateway.yml,
  deploy-uc.sh, e2e-спеки rn, структура terra-core.
- Свип по `all_commits.csv`: 216 161 коммит, 233 репо, паттерн из ~30 слов → 62 репо с хитами,
  разобраны топ-40 (даты+заголовки), из них вручную отсеяны FP.
- Отсеянные классы ложных срабатываний: hunspell-словарь в jump («конспектировать» — 17 хитов);
  словарь токенизатора в terra-core («▁summarized»); i18n-компилятор ngx-translate; `1c` в
  hex/UUID (3461 «хитов» в iva-one — все мусор, паттерн `\b1c\b` негоден); «assistant» в ivasw —
  телефонный IVR-ассистент/автосекретарь, не ИИ; «recognition» в «распознавание ссылок/e-mail»;
  SummaryInfoRestDTO (сводка полей, не саммари); IvaIdLogin ложно матчит ai.?log.

## 3. Таблица находок

| # | Фича | Вердикт | Доказательство (файл/символ) |
|---|---|---|---|
| 2 | Конспект встречи | **НАЙДЕНО (сервер)** | ivcs-server `TranscriptionServiceImpl`: настройка `EVENT_TRANSCRIPT_SUMMARY`, тип `SUMMARY`, `IVA_TERRA_SUMMARY_PARTICIPANT_ID`, запись в txt/docx; пост-процессинг через `SPEECH_RECOGNITION_API_POST_PROCESS_URL` (Terra/ADP). Тесты `TranscriptionServiceTest.*SummaryIsOn` |
| 5 | Протокол → поручения | **НАЙДЕНО (сервер)** | там же: тип `ASSIGNMENTS`, `IVA_TERRA_ASSIGNMENTS_PARTICIPANT_ID`, `language.getAssignmentsName()` — LLM-поручения пишутся отдельным документом рядом с конспектом |
| 3 | Субтитры/ASR | **НАЙДЕНО** | ivcs `writeTranscript_shouldWriteTranscriptAndSendSubtitles`; клиенты: kmp `ConferenceFragmentViewModelImpl`, iva-one `conference-session-transcription-state-changed-event.ts`, web/iva-connect и mobile/apple (по коммитам VCSWEB2-10584, №15054); движок: terra-core `rapi/terradyne` (свой ASR), диаризация `rapi/diarizator/wespeaker` |
| 4 | Перевод | **НАЙДЕНО (для ВКС)** | ivcs `TerraTranslationRequest/Payload.of(minute, name, SubtitlesLanguage translateTo)` + `SPEECH_RECOGNITION_API_TEXT_TRANSLATION_URL`; terra-core `translator_service/pretrained_en_ru|ru_en` (свой MT). Перевода СООБЩЕНИЙ в чатах нет — все хиты i18n |
| 6 | Письмо → задача | **НАЙДЕНО (контур rn)** | rn `rn-mail/src/utils/assistantBriefWithTask.ts` — skill `mail-brief-with-task`: бриф письма + черновик задачи одним рейсом, zod-валидация, citations; e2e `mail-assistant-task-real.spec.ts` (CM-110/TASKS-004): задача реально создаётся в модуле Задач через `TasksBridge.createTask` |
| — | Черновик ответа на письмо | **НАЙДЕНО (rn), в списке не было** | `rn-mail/src/utils/assistantDraftResponse.ts`, `MailRightRail.tsx` (кнопка Summarize + task draft + draft reply) |
| 9 | Боты в чатах | **НАЙДЕНО** | ivcs-server `administration/.../chatbot/ChatBotEditActivity` (CRUD чат-ботов), iva-one `chat-bot-keyboard.component.ts`; коммиты iva/one/backend/chats `feat/9989-add-format-message-for-chat-bot`, web/iva-admin IVAADM-167..204 |
| 12 | SmartApp/мини-приложения | **НАЙДЕНО** | kmp `feature/smart_apps/...` (SmartAppsRootComponentImpl, SmartAppsRepositoryImpl, DI-граф); коммиты iva/one/backend/proto+api-gateway `IVAONE-10147 Smart apps v1 CRUD`, users `JWKS для смартапов`, ios `IVAONE-10363`. Маркетплейса нет (хиты «marketplace» ≤2, мусор) |
| 7 | Память/RAG | **СЛЕДЫ→НАЙДЕНО (контур rn, выключено)** | rn `web/tools/deploy/host-scripts/docker-compose.ai-gateway.yml`: Qdrant `assistant_pointer_memory`, эмбеддинги BAAI/bge-m3 (infinity), источники `chat_message,email_message`, ACL-fingerprint, TTL 30 дней, wiki-контекст; `AI_GATEWAY_RAG_ENABLED=false` по умолчанию. Клиент шлёт/получает `sourceRefs.source_chunk_id` (`rn-chat/src/assistant/client.ts`) |
| 8 | Агент действий / чат-ассистент | **ЧАСТИЧНО** | rn `rn-chat/src/assistant/client.ts` `requestAssistantChatResponse` (история, botPost в чат, контекст выделенного текста); гейтвей `iva-ai-gateway` на Node 22, LLM DeepSeek v4 (flash/pro, simple/thinking), маршрут `/api/assistant/v1` на uc.iva.ru (`RN_ASSISTANT_API_BASE` в config.prod.json). Function-calling/действий в продукте (создать встречу, позвонить) не видно, кроме задач из письма и calendar-confirmation (e2e `assistant-calendar-confirmation-real.spec.ts`) |
| 1 | Суммаризация чатов | **НЕ НАЙДЕНО КАК ФИЧА** | в One 2.0 (iva-one), kmp, ivcs — ноль хитов вне FP; в rn есть только бриф ПИСЬМА (Summarize в MailRightRail) и чат-ассистент, которому можно задать вопрос; отдельной кнопки «саммари чата/треда» нет ни в одном снимке |
| 10 | Задачи из встреч/чатов агентом | **ЧАСТИЧНО** | из писем — есть (см. №6); из встреч — только документ поручений (ASSIGNMENTS), в Задачи не приземляется; из чатов — не найдено |
| 11 | Диск: поиск по содержимому / документ как источник действия | **СЛЕДЫ** | коммиты modules/diskstorage IVADS-229 «Add Internal Speech to text client», «Subtitles work for eager preview on WAV», IVADS-285 «transcribe task» — транскрибация аудиофайлов на Диске есть; семантического/полнотекстового поиска по документам не найдено (opensearch — только как компонент мониторинга в helm-чартах) |
| — | Голосовые сообщения → текст | **НАЙДЕНО, в списке не было** | kmp `VoiceTranscribeButton.kt`, `VoiceTranscriptStatusDTOAngryTest.kt`; iva-one `voice-message/transcription/transcription.service.ts` + trigger/output компоненты; jump: иконки transcribe + `AudioItem.qml` |
| 14 | Разрешённые модели | **ЧАСТИЧНО** | только env-allowlist гейтвея: `AI_GATEWAY_DEEPSEEK_ALLOWED_MODELS=deepseek-v4-flash,deepseek-v4-pro` (compose). Продуктовой админки «какие модели разрешены» нет |
| 15 | Маскирование ПДн | **НЕ НАЙДЕНО В СНИМКЕ** | паттерны маскирован|anonymiz|PII|pseudonym: ≤9 хитов на репо, все FP (логи, DLP-антивирус аватарок). Единственное близкое — коммит modules/media «Mask phrases produced by speech recognition engines in logs» (маскирование фраз ASR в логах — не ПДн-фильтр перед LLM) |
| 16 | Журнал ИИ | **НЕ НАЙДЕНО В СНИМКЕ** | ai-audit/ai-log/prompt-log: все хиты — IvaIdLogin и SSO-строки. В гейтвее только docker json-file логи |
| 13 | Коннекторы 1С/CRM/ЭДО | **НЕ НАЙДЕНО В СНИМКЕ** | bitrix/ЭДО/amocrm/service-desk: нули либо мусор (`1c` в hex). В rn есть BPM-контур (`rn-shared/src/bpm/gateway.ts`, docker-compose.bpm.yml) — процессы, но не коннекторы |
| 17 | Единый ИИ-слой | **ЧАСТИЧНО, распределён по трём стекам** | (а) Terra: terra-core — свой ASR (terradyne), диаризация (wespeaker), MT (translator_service), **TTS (tts_service, tts_worker)**, воркеры online/offline; (б) **blank130/iva_adp** — vLLM-сервис (28 коммитов, Елена Сакевич): задачи summary, diar_identification (LLM-опознание спикеров, guided decoding), **planner**, assistant со стримингом и историей, `/v1/chat/completions`; снимка кода нет — только коммиты; (в) rn ai-gateway — внешний DeepSeek API. Три несогласованных ИИ-точки, а не один слой |

## 4. Чего не хватает и что надо реализовать (по приоритету)

1. **Суммаризация чатов/тредов** — единственная фича, названная руководителем голосом, и её нет
   нигде. При этом все кубики готовы: iva_adp умеет summary+assistant, гейтвей rn умеет чаты.
   Работа — продуктовая обвязка (кнопка, права, лимиты), не исследование.
2. **Донести конспект/поручения ВКС до действий** — сервер уже пишет SUMMARY и ASSIGNMENTS в
   docx/txt на диск; не хватает шага «поручение → задача в Задачах» (аналог того, что rn уже
   делает для писем). Это соединение двух готовых концов.
3. **Свести три ИИ-стека в один слой (IVA GPT)** — DeepSeek в rn-гейтвее ходит во внешний API,
   что ломает «суверенный контур»; iva_adp с vLLM — внутренний и уже умеет chat/completions.
   Перевод гейтвея с DeepSeek на iva_adp = суверенность + одна точка для журнала и allowlist.
4. **Журнал ИИ и маскирование ПДн** — не реализованы вовсе; ставить их надо в единой точке (см.
   п.3), иначе придётся делать трижды.
5. **Включить и довезти RAG-память** — каркас в гейтвее полный (Qdrant, bge-m3, ACL, TTL),
   но выключен по умолчанию и живёт только в rn-контуре; в One 2.0 отсутствует.
6. **Перевод сообщений** (не только субтитров) — MT-движок свой уже есть (terra translator),
   нет только продуктовой поверхности в чатах.
7. **Поиск по содержимому Диска** — есть только ASR аудиофайлов; полнотекста/семантики по
   документам нет — самая «нулевая» из фич слайда вместе с коннекторами и маркетплейсом.
8. **Коннекторы 1С/CRM/ЭДО и маркетплейс агентов** — с нуля; единственная опора — SmartApp v1
   (CRUD есть) и BPM-контур rn.

## 5. Что меня удивило (в списке не было)

- **blank130/iva_adp** — целый внутренний LLM-сервис на vLLM (summary, диаризация через LLM,
  planner, assistant со стримингом), один автор, 28 коммитов с июля 2024. Это готовая
  «суверенная» альтернатива DeepSeek, и о ней легко не узнать: кода в снимках нет, имя репо
  ничего не говорит.
- **Ассистент почты в rn доведён до продукта дальше всего**: бриф+задача+черновик ответа одним
  SSE-рейсом, zod-схемы, коды ошибок, e2e против реального бэкенда, отдельный гейтвей с
  секретами вне браузера. Самый зрелый ИИ-код из всего, что видел, — и он в линии 1.5 (uc.iva.ru),
  не в One 2.0: iva-one (веб 2.0) и kmp не знают об ассистенте вообще.
- **TTS-воркер в Terra** (tts_service, terra_tts_worker v5.1) — синтез речи есть, на слайде его
  нет ни в одной ячейке.
- **Транскрибация голосовых сообщений** есть во всех трёх клиентах (kmp, iva-one, jump) — фича
  живёт, хотя на слайде не названа.
- **fedot_gardener** — Telegram-бот «AI-агроном» (LangChain, OpenAI/Gemini/DeepSeek) в корпусе
  заказчика; к продуктам ИВА отношения не имеет, но даёт 2768 ложных AI-хитов — ловушка для
  любого автоматического поиска.
- Термин «assistant» в ivasw — телефонный автосекретарь IVR, исторический омоним; любой отчёт,
  считающий его ИИ-ассистентом, ошибётся.

## 6. Где мой вывод шаткий

- **Гейтвей `iva-ai-gateway` — исходников не видел**: deploy-uc.sh собирает его из
  `$CONNECT_DIR/ai-gateway` (родительский каталог рядом с репо rn), в снимок rn он не вошёл, в
  списке 233 репо кандидата с таким именем я не нашёл. Все выводы о нём — из compose-файла,
  деплой-скрипта и клиентского кода. Надо найти сам репозиторий/каталог.
- **iva_adp — только по заголовкам коммитов**, снимка нет. «Planner» может быть чем угодно.
  Стоит взять снимок blank130/iva_adp.
- Отрицательные выводы (маскирование ПДн, журнал ИИ, коннекторы, суммаризация чатов) упираются
  в сжатие repomix: тела функций вырезаны, фича без говорящего имени файла могла проскользнуть.
  Их бы перепроверить по полным клонам ivcs-server и iva-one.
- Строку слайда «IVA MCU/SBC/ROOM/CS-Largo/IVA360» не проверял отдельно: их репо в снимках нет,
  в коммит-свипе заметного ИИ не видно (кроме субтитров в desktop/ivcs — это клиент MCU).
- «Память» из ячейки One («агент действий, память») я истолковал как RAG-память ассистента;
  если имелась в виду память диалога — она в iva_adp уже есть (trim history for context).
