---
title: Разведка ИИ-функционала — IVA Mail (jump + iva-outlook-plugin)
type: note
status: draft
created: 2026-08-03
tags:
- recon
- ai-audit
- iva-mail
- jump
- outlook-plugin
- explore
permalink: tacticum/00-board/recon-ai-mail-2026-08-03
---

# Разведка: интеллектуальные фичи в IVA Mail

**Главный вывод одной строкой:** в корпусе почты (десктоп-клиент `jump` + `iva-outlook-plugin`)
**собственного ИИ/ML/LLM-кода нет ни одной строки**. Весь ИИ-слой компании (конспекты, субтитры,
перевод, распознавание объектов, DLP) живёт в **сервере ВКС (IVCS/IVA One)** — и это доказано
файлом внутри корпуса Outlook-плагина, а не догадкой.

---

## 1. Что проверено и как

| Корпус | Объём | Метод |
|---|---|---|
| `/Users/bubblemac/tacticum/helm/data/real/git/repomix/jump.xml` | 8.4 МБ, 2907 файлов, 294 984 строк, C++/Qt | `rg -i` по ~70 паттернам + резолв каждого хита в файл через индекс `<file path=`; отсев словарей орфографии |
| `/Users/bubblemac/tacticum/helm/data/real/git/repomix/iva-outlook-plugin.xml` | 3.6 МБ, 208 файлов, 86 363 строк, C# | тот же метод, ~40 паттернов |
| SCIP `scip/jump/{cpp,js,ts,python}/index.scip` (44 МБ cpp) | символьный индекс | `strings \| rg -io` по AI-токенам — ловит символы, тела которых вырезал `--compress` |
| SCIP `scip/iva-outlook-plugin/csharp/index.scip` | 1.1 МБ | то же |
| `topology/jump/topology-review.md` | 16 блоков | чтение целиком |

**Просеяно хитов:** порядка 1500 сырых совпадений; после отсева ложных срабатываний осталось
**ноль** признаков собственного ИИ в почте и **один** внешний источник правды (см. §4).

**Отсеянные ложные срабатывания (важнее, чем набор хитов):**

- `LLM` (44 хита в jump) — все до единого подстроки: `fi**llM**ode` (Qt Image), `delayTi**llM**s`,
  `a**llM**ailboxRights`, `insta**llM**anager`, `scro**llM**ultiplier`. Настоящего LLM нет.
- `summar` (598 хитов в C#) — XML-doc комментарии `/// <summary>`. В jump — `SUMMARY` из RFC 5545
  (заголовок события календаря) в `Plugins/CalendarPlugin/UI/EventComposerWidget.ui`.
- `digest` — `TrustedCertificate."Digest"` (отпечаток сертификата S/MIME), а не «дайджест писем».
- `GPT`, `NLP`, `PII`, `OCR`, `DLP` в Outlook-плагине — base64-блобы иконок внутри `.resx`
  (`rbnIva.en.resx`, `ConfirmationCodeFormWithFatalError.resx`). Классический мусор.
- `prompt` — `<ErrorReport>prompt</ErrorReport>` в `.csproj` и `"PromptEnabled"` в инсталляторах.
- `RAG` — подстрока в `d**rag**`, `f**rag**ment`. `vector` — `std::vector`. `bot` — иконки и
  `contact.type === "bot"` в чужом UI-ките.
- `translat` — `installTranslations`, `bodyTranslations`, `subjectTranslations` = **локализация Qt**,
  не машинный перевод. Проверено по SCIP-символам.

**Нулевые паттерны (0 хитов в jump вообще):** `openai`, `gemma`, `ollama`, `vllm`, `yandexgpt`,
`gigachat`, `qdrant`, `faiss`, `milvus`, `elastic`, `opensearch`, `semantic`, `rerank`, `classif`,
`sentiment`, `тональн`, `NLP`, `spam`, `спам`, `smart reply`, `умный ответ`, `autocomplete`,
`распознав`, `PII`, `маскир`, `anonymiz`, `DLP`, `assistant`, `ассистент`, `agent`, `агент`,
`inference`, `onnx`, `torch`, `tensorflow`, `neural`, `нейросет`, `машинн`, `аудит`,
`конфиденц`, `переписк`, `conversation`.

---

## 2. Таблица находок

### 2.1 Фичи из презентации руководителя

| Фича | Статус | Файл : символ | Что делает | Зрелость |
|---|---|---|---|---|
| **Поиск по содержимому писем** | **НАЙДЕНО** | `Plugins/MailPlugin/Resources/MailsDatabaseScripts/47.sql:280184` — `CREATE VIRTUAL TABLE "MailsSearchTokensFtsIndex" USING fts5(... tokenize = 'trigram')`; наполнение — `Plugins/MailPlugin/Core/Serializers/EmailSerializer.cpp:278024` `EmailSerializer::insertSearchTokens(const Email&)`; ввод — `Plugins/MailPlugin/Core/EmailItemModel.cpp:278381` `EmailItemModel::setSearchText(const QString&)`; UI — `Plugins/MailPlugin/UI/MailWidget.ui:290290` `searchInput` | Локальный полнотекстовый поиск SQLite **FTS5 с триграммным токенайзером** по 4 полям: `Subject`, `Names`, `Peers`, `Text`. Индекс поддерживается триггерами `MailsSearchTokens_InsertIntoFts` / `_DeleteFromFts` (стр. 280459, 280466). Зависимость закреплена: `vcpkg.json:294972` — `"sqlite3", "features": ["fts5","unicode"]` | **Прод.** Но это подстрочный/лексический поиск. Семантики, эмбеддингов, ранжирования по релевантности — **нет** |
| **Письмо → задача** | **НЕ НАЙДЕНО В СНИМКЕ** | — | Ни трекера, ни создания задач. `task-tracker*.svg` есть только как **иконки** в `Libraries/LibUi/Resources/IconsV2/` и в чужом `commonlibs` — кода за ними нет. Слово «задача»/`task` в коде = `LibBase/Thread.cpp` (пул потоков) | — |
| **Память переписки** | **НЕ НАЙДЕНО В СНИМКЕ** (есть сырьё) | `MailsDatabaseScripts/47.sql:280031` — колонки `"InReplyTo"`, `"ReplyReferences"` в таблице `Mails` | Заголовки RFC 5322 сохраняются в БД, то есть **техническая основа для тредов есть**. Но ни группировки в треды, ни «памяти» (сводка контекста, история отношений с адресатом) в коде нет: `conversation`, `ThreadId`, `ConversationId` — 0 хитов | Сырьё, фичи нет |
| **Контроль утечек / DLP в почте** | **НЕ НАЙДЕНО В СНИМКЕ** | — | В почтовом клиенте нет ни DLP, ни маскирования ПДн, ни классификации по грифу. Ближайшее по смыслу, но **не ИИ**: отзыв отправленного письма (`Plugins/MailPlugin/UI/RevokeSentEmailWidget.h:290403`, `enum class Mode { RevokeAndDelete, RevokeAndReplaceWithNewOne }` + `Libraries/LibJump/Methods.cpp` `MessageRevoke::method()` + окно отзыва `MailSettings::maxAgeToRevoke()`), ACL почтовых ящиков (`Plugins/MailPlugin/UI/ACL/MailboxRightsWidget.h`), S/MIME (`Plugins/MailPlugin/Core/Smime/SmimeHelper.h`), санитайзер HTML `Resources/JsScripts/DomPurifyHook.js` | — |
| **Аудит писем** | **НЕ НАЙДЕНО В СНИМКЕ** | — | `аудит`/`audit` в jump — **0 хитов**. В Outlook-плагине есть `OutlookPluginIVA-en/Infrastructure/AuditLog.cs` (`AuditLog.NewOperationId()`, `LinkOperationId`, `Info/Warn/Error/Write`) — но это журнал действий **самого плагина** (создание/правка конференций), не аудит писем | — |
| **Конспекты / субтитры / перевод / маскирование ПДн** | **НЕ НАЙДЕНО** в почте, **НАЙДЕНО во внешнем API** | см. §4 | В коде почты — ничего. В спецификации сервера ВКС, лежащей внутри корпуса Outlook-плагина, — весь набор | см. §4 |

### 2.2 Что интеллектуального в почте всё-таки есть (детерминированное, не ИИ)

| Механизм | Статус | Файл : символ | Что делает | Зрелость |
|---|---|---|---|---|
| **Правила обработки писем** | **НАЙДЕНО** | `Plugins/MailPlugin/Core/Rule.h` — `enum class Condition`, `enum class Operator`, `enum class Action`, `struct ConditionObject`, `struct ActionObject`, `struct Rule`; UI — `Plugins/MailPlugin/UI/Settings/Rules/RuleConditionWidget.cpp` (`setOperatorsForCurrentCondition()`, `localizedConditionText(Condition)`); серверная сторона — `Core/JumpBackend/JumpRuleParser.h`, `JumpRuleSerializer.h`, RPC `ruleAdd`/`ruleSet`/`ruleList`/`ruleDelete` | Классические фильтры «условие → действие», хранятся на сервере. **Правила пишет человек**, никакой автоклассификации/обучения нет | Прод, детерминированный |
| **Проверка орфографии** | **НАЙДЕНО** | `Plugins/MailPlugin/UI/QML/RichTextEdit.qml:286555` — `property var spellcheckSuggestions`, `:286802` `spellcheckSuggestions = request.spellCheckerSuggestions`; словари `PluginHost/Resources/Spellcheck/{en_GB,ru_RU}.{aff,dic}` | Встроенный спелчекер Qt WebEngine на **Hunspell-словарях** (ru + en). Подсказки замены при наборе письма | Прод. Словарный, не ML |
| **Приоритет письма** | **НАЙДЕНО** | `Plugins/MailPlugin/Core/Priority.h`, `Libraries/LibPluginCore/UI/PriorityComboBox.h` | Приоритет **выставляет отправитель вручную** (заголовок X-Priority). Автоопределения важности нет | Прод, ручной |
| **Предложение другого времени встречи** | **НАЙДЕНО** | `Plugins/MailPlugin/UI/Banners/SuggestTimeWidget.h:280716` — `SuggestTimeWidget::reply()`, `startAndEnd()`, `comment()`; RPC `calObjPropose` | iCalendar COUNTER-предложение по приглашению в письме. Слово `suggest` в названии обманчиво: **выбор времени делает человек**, подбора слотов нет | Прод, не ИИ |
| **Расшифровка голосового сообщения** | **СЛЕДЫ (чужой код, в почте не подключён)** | `Libraries/commonlibs/UiToolKit/qt/qml/Ui/Components/Extra/Media/AudioItem.qml:30719` — `transcribeIndicatorVisible`, `:30733` `signal transcribeClicked`, `:30857` `transcribeBtn`; иконки `LibUi/Icons/Filled#transcription` | Кнопка «расшифровать» на аудиосообщении — **UI-поверхность транскрибации**. Но: `Libraries/commonlibs` — это **git-submodule** (`.gitmodules:294811`, топология помечает `vendored` — «чужой код, не решение проекта»), а `AudioItem.qml` **не используется никаким кодом почты**: единственные ссылки — регистрация в `qmldir` и `.qrc` | Не фича почты. Это UI-кит мессенджера IVA One, приехавший сабмодулем |

### 2.3 Полная поверхность серверного протокола почты (доказательство от обратного)

Клиент общается с почтовым сервером через `LibJump`. Полный список методов извлечён из
`Libraries/LibJump/Methods.h` (инлайновые `method()`) и `Libraries/LibJump/Methods.cpp`
(вынесенные) — **64 метода, ни одного интеллектуального**:

```
accountGetMyRights accountReadRights accountUpdateRights addrBookClose addrBookOpen
busyTimesList calendarGetMyRights calendarOpen calendarReadRights calendarUpdateRights
calObjAccept calObjCancel calObjDecline calObjList calObjPropose calObjPublish calObjSync
close contactDelete contactList contactRead contactSave contactSync delayedDelivery
getServerInfo getSessionInfo identityDelete identityList identitySet login mailboxFind
mailboxGetMyRights mailboxReadRights mailboxUpdateRights messageSave messageSend messageSync
namespaceList namespaceSet ping preferenceGet preferenceSet ruleAdd ruleDelete ruleList
ruleSet runCommand signatureDelete signatureList signatureSet
+ из .cpp: CalendarClose DirectoryFind MailboxCreate MailboxRemove MailboxRename MailboxOpen
  MailboxClose MessageCopy MessageDelete MessageFlag MessageList MessageMove MessageRead
  MessageRevoke MessageDispNotify
```

Отдельно важно: **метода поиска на сервере нет вообще**. Поиск целиком клиентский, по локальной
SQLite. Значит поиск работает только по тому, что уже синхронизировано на устройство.

---

## 3. Неожиданное

1. **Поиск лексический и локальный, и это архитектурное ограничение, а не недоделка.** Триграммный
   FTS5 по локальной БД + отсутствие серверного метода поиска = семантический поиск нельзя добавить
   «сбоку», нужен серверный компонент. Это самое дешёвое место для встраивания ИИ и самое дорогое
   по архитектуре.
2. **Копирайт `ООО «АйМэйл»`** в шапке каждого файла `jump` (не «ИВА»). Почта — отдельное
   юрлицо/команда, что объясняет полное отсутствие общего ИИ-слоя с ВКС.
3. **`Libraries/commonlibs` — сабмодуль общего UI-кита с ВКС/мессенджером.** В нём иконки
   `transcribe`, `transcription-*`, `bot`, `chat-bot`, `task-tracker` и кнопка расшифровки аудио.
   Соблазн принять их за фичи почты велик — **это чужие ассеты, в почте не подключены**.
   Иконка ≠ фича.
4. **`runCommand`** в протоколе — обобщённый метод. Тело в снимке вырезано; чем он ограничен,
   по снимку не видно.
5. **Отзыв отправленного письма с окном по времени** (`maxAgeToRevoke`) — единственная реально
   «контролирующая утечку» механика в почте, и она полностью ручная.

---

## 4. Внешний ИИ-слой: где он на самом деле (НАЙДЕНО, но это ВКС, не почта)

Файл `doc/dependencies/clients-openapi.json` внутри корпуса Outlook-плагина — это спецификация
клиентского API **сервера ВКС (IVCS)**. Она содержит промышленный ИИ-контур:

| Возможность | Доказательство (`doc/dependencies/clients-openapi.json`) |
|---|---|
| **Транскрибация встреч** | `:8770` `/conference-sessions/{id}/transcription/start` → `operationId: startTranscription`; `:9026` `/transcription/stop`; события `ConferenceSessionTranscriptionStateChangedEvent` (`:34165`), `ConferenceSessionTranscriptionPhraseEvent` (`:36320`) |
| **Конспект встречи** | настройка `EVENT_TRANSCRIPT_SUMMARY` (`:40687`) — рядом с `EVENT_TRANSCRIPT`, `EVENT_TRANSCRIPT_STORAGE_DAYS`, `TRANSCRIPT_FILE_FORMAT` |
| **Субтитры в реальном времени** | `:8729` запрос субтитров; `EVENT_SUBTITLES`, `EVENT_SUBTITLES_LANGS`, `SPEECH_RECOGNITION_SUBTITLES_MAX_PHRASE_DURATION` |
| **Машинный перевод расшифровки** | `TRANSCRIPT_TRANSLATION`, `TRANSCRIPT_TRANSLATION_LANGS`, `SPEECH_RECOGNITION_API_TEXT_TRANSLATION_URL`, `SPEECH_RECOGNITION_TRANSLATION_LANGUAGES` |
| **ASR как внешний сервис** | `SPEECH_RECOGNITION_SERVICE`, `SPEECH_RECOGNITION_API_URL`, `SPEECH_RECOGNITION_API_KEY`, `SPEECH_RECOGNITION_API_POST_PROCESS_URL`, `SPEECH_RECOGNITION_PROXY_URI`, `SPEECH_RECOGNITION_MAX_PHRASE_DURATION`. **Ключевое: ИИ подключается по URL+ключу, движок внешний и сменный** |
| **Распознавание объектов (CV)** | `OBJECT_RECOGNITION_ENABLED`, `OBJECT_RECOGNITION_API_URL`, `OBJECT_RECOGNITION_LOGIN`, `OBJECT_RECOGNITION_PASSWORD` |
| **DLP (контроль утечек) — настоящий** | 18 настроек `DLP_*`: `DLP_SCAN_ENABLED`, `DLP_SILENT_MODE`, `DLP_FULL_AUDIT`, `DLP_CHECK_CHAT_MESSAGE`, `DLP_CHECK_MESSAGE`, `DLP_SCAN_ON_ATTACHED`, `DLP_SAVEFILE_ON_DLP_DETECTED`, `DLP_MAX_THREADS`… Плюс схема `ResourceDlpScanResult` (`:20953`) с состояниями `NO_NEED_TO_CHECK / NOT_CHECKED_YET / SCAN_ENGINE_ERROR / VERIFIED / DLP_DETECTED` и ошибка `BLOCKED_BY_SECURITY_CHECK` при отправке сообщения (`:1488`) |
| **Антивирусное сканирование** | ~22 настройки `AVSCAN_*` через ICAP (`AVSCAN_ICAP_URL`, `AVSCAN_TRY_DETECT_UNCHECKABLE_FOR_DRWEB`) |
| **Аудит доступа** | `AUDIT_ACCESS_TO_RECORD_TRANSCRIPT`, `AUDIT_ACCESS_TO_EVENT_DOCUMENTS`, `AUDIT_LOG_LEVEL`, `AUDIT_FULL_*` |
| **Грифы секретности** | `SecurityLevel`: `UNCLASSIFIED / CONFIDENTIAL / SECRET / TOP_SECRET` (`:24096`, `:40683`) |

Второй файл `doc/dependencies/internal-openapi.json` — по AI-паттернам **пусто**
(`transcript` 0, `recogn` 0, `dlp` 0, `subtitle` 0).

**Вывод для аудита:** между ИИ-контуром ВКС и почтой **нет ни одной точки соединения**. Почта не
вызывает ни один из этих API. Если руководителю показывали «конспекты и контроль утечек» — это
демо ВКС, и в почте этого нет.

---

## 5. Что НЕ найдено и где доискивать в живом коде

Снимки сделаны с флагом `--compress`: пути, структура и **сигнатуры** сохранены, а **тела функций
местами вырезаны** (разделитель `⋮----`). Поэтому «не найдено» ≠ «не существует». Чтобы закрыть
вопрос окончательно, в живом репозитории надо посмотреть:

| Что доискать | Где именно | Почему снимок не отвечает |
|---|---|---|
| **Значения enum'ов правил** | `Plugins/MailPlugin/Core/Rule.h` — тела `enum class Condition/Operator/Action`; `Plugins/MailPlugin/UI/Settings/Rules/EmailRuleCriteria.h` (**в снимке остался только копирайт**) | Тела вырезаны. Если среди `Condition` есть что-то вроде «похоже на спам» или «категория» — вывод по автоклассификации меняется |
| **Строки интерфейса** | `Locale/MailPlugin_ru.ts` и все 105 файлов `Locale/*.ts` — **в снимке пустые** (`jump.xml:69539` содержит только открывающий/закрывающий тег) | `--compress` вычистил XML-содержимое. Русские строки UI — самый быстрый способ увидеть фичу, которой не видно по именам символов. **Это главная слепая зона разведки** |
| **Границы `runCommand`** | `Libraries/LibJump/Methods.h` (класс `RunCommand`), `Plugins/MailPlugin/Core/MailCommandQueue/` | Тело `payload()` вырезано; какие команды допустимы — не видно |
| **Серверная часть почты** | Репозитория почтового сервера в корпусе **нет вообще** — есть только клиент | Клиент видит 64 RPC-метода; что сервер умеет сверх них (например, серверный поиск или классификация), по клиенту не определить. **Обязательно проверить отдельно** |
| **Веб-клиент и мобильные клиенты почты** | вне корпуса | Проверен только десктоп. ИИ-фичи часто выкатывают сначала в веб |
| **Фича-флаги / серверные preference** | RPC `preferenceGet`/`preferenceSet`, `getServerInfo` | Список ключей задаёт сервер; выключенная за флагом фича в клиенте не видна |
| **Плагинная система** | `Libraries/LibPluginCore/KnownPluginNames.h` — **в снимке пустой namespace**; `Libraries/LibPluginSystem/include/ICustomPlugin.h` | Хост умеет грузить внешние плагины. Если ИИ-функционал поставляется отдельным плагином, в этом репозитории его и не будет по определению |

---

## 6. Как это перепроверить (воспроизводимость)

```bash
# Индекс путей файлов внутри repomix-снимка
rg -n '^<file path=' jump.xml > jump_files.txt

# Хит → файл: ближайший предыдущий <file path=...> по номеру строки
rg -n -i '<паттерн>' jump.xml

# Символы, чьи тела вырезал --compress, но которые остались в SCIP
strings -n 6 scip/jump/cpp/index.scip | rg -io '[A-Za-z_]*(summar|llm|embedding|transcri|classifi)[A-Za-z_]*' | sort -u
```

Скрипт-резолвер хитов в файлы:
`/private/tmp/claude-501/-Users-bubblemac-tacticum-vault/69191caa-cd1f-458e-b45d-a5883b6f3cc2/scratchpad/hits.py`