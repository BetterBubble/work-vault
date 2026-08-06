---
title: srv-rnlive-2026-08-03
type: note
permalink: tacticum/00-board/srv-rnlive-2026-08-03
status: draft
created: '2026-08-03'
role: explorer (разведчик)
subject: Разведка репозитория rn-live с живого сервера — ИИ-контур IVA One 1.5-RN
tags:
- recon
- ai
- rn-live
- assistant
- rag
- audit
---

# Разведка: `rn-live` (живой сервер, `/srv/helm/repos/rn-live`)

ИИ-контур линии IVA One 1.5-RN по выгрузке с живого сервера. Репозиторий в прежний аудит не входил.

## 0. Дисциплина доказательств и честная граница этой работы

Работал **только по выгрузке грепа** `srv-rnlive.txt` (401 строка формата `файл:строка:текст`).
Самого репозитория у меня нет — ни локально, ни по SSH. Поэтому:

- `НАЙДЕНО` = строка есть в выгрузке, привожу `файл:строка`.
- `НЕ ВИДНО В ВЫГРУЗКЕ` ≠ «нет в репозитории». Выгрузка — срез по ИИ-словам, не полный обход.
- **Выгрузка обрезана, и это важно.** Лид передал частоты `semantic` 58, **`rerank` 55**,
  `embedding` 16, `qdrant` 11, `LLM` 5, `GPT` 4. В моём файле фактически: `semantic` 20,
  `embedding` 20, `qdrant` 12, `llm` 4, `gpt` 4, **`rerank` — 0 строк**. То есть ни одного
  из 55 совпадений по `rerank` в мой срез не попало, а `semantic` попал на треть.
  **Про переранжирование я не могу сказать ничего доказательного** — см. §8, запрос Q1.
  Гипотеза (не факт): в снимке `rn` месячной давности `rerank()` жил в
  `rn-shared/src/search/SearchService.ts` и был **лексическим** ранжированием поверх MiniSearch
  (зафиксировано в `00-Board/recon-ai-one-2026-08-03.md:92`). Если это тот же символ — «зрелый
  поисковый контур» здесь ни при чём. Но подтверждать нечем.

Базой для сравнения беру `00-Board/recon-ai-one-2026-08-03.md` (разбор снимка `rn`) — ниже
пишу **дельту**, а не пересказ.

---

## 1. Что это за репозиторий и чем отличается от `rn`

**Вывод: это не другой продукт и не другая линия. Это тот же монорепозиторий `ione`, только
рабочая копия на сервере, а не repomix-снимок.**

Доказательство — состав дерева. В `rn-live` лежит **и** веб-SPA, **и** RN-модули, **и** тулинг
деплоя, то есть ровно тот же набор, что в `rn`:

| Что | Строка из выгрузки |
|---|---|
| Angular-SPA целиком | `rn-live/web/apps/app/src/app/features/rn-modules/rn-chat-host.component.ts:61` |
| Тот же `DEPLOY.md` с той же §6.8 | `rn-live/web/DEPLOY.md:1009` |
| Тот же compose ИИ-шлюза | `rn-live/web/tools/deploy/host-scripts/docker-compose.ai-gateway.yml:3` |
| Тот же реестр воркэраундов | `rn-live/docs/WORKAROUNDS-REGISTRY.md:488` (`AI-WR-001`) |
| RN-модули | `rn-chat`, `rn-mail`, `rn-calendar`, `rn-tasks`, `rn-disk` (`rn-live/rn-tasks/HANDOFF.md:243` и др.) |

**Главное отличие от `rn` — техническое, но полезное:** в `rn` номера строк были
repomix-сцепленные (`DEPLOY.md:429677`), здесь — **настоящие номера в настоящих файлах**
(`DEPLOY.md:1009`). То есть `rn-live` пригоден для точной цитаты, а `rn` — нет.

**Признаки, что это именно развёрнутая копия, а не голый клон:** в `web/apps/app/src/assets/`
лежат **собранные бандлы** модулей — `rnchat.5a7b070d.js`, `rnchat.9561fa44.js` (две разные
сборки чата), `rnmail.b2770207.js`, `rnchat-element.js`, `rnmail-element.js`,
`rnbpm-element.js`, `rncalls-element.js`, `rncontacts-element.js`, `rndisk-element.js`,
`rntasks-element.js`, `rnminiapps-element.js`, `rncommunications-element.js`
(`rn-live/web/apps/app/src/assets/*.js:1–2`). Артефакты сборки лежат в дереве исходников —
так выглядит рабочий стенд, а не чистый репозиторий.

**Чего я не могу сказать честно:** новее ли `rn-live`, чем месячный снимок `rn`. `package.json`,
`README`, `git log`, даты — **в выгрузку не попали вообще**. Ветка, тег, дата последнего коммита —
неизвестны. Запросы Q2/Q3 в §8.

**Что в `rn-live` есть, а в разборе `rn` не фигурировало ни разу** (кандидаты в «новое», но без
дат это остаётся «не было в прошлом отчёте», а не «появилось позже»):

| Находка | Строка |
|---|---|
| **`CONFIG__MODEL="openai/gpt-oss-120b"`** в CI | `rn-live/web/.gitlab-ci.yml:166` |
| **Стриминговый эндпоинт `/api/assistant/v1/chat/stream`** (SSE) | `rn-live/e2e/mail-assistant-real-probe.spec.ts:66,83,143`; `rn-live/rn-mail/src/utils/__tests__/assistantDraftResponse.test.ts:70` |
| **UI «Semantic Sources» в конструкторе бизнес-процессов** | `rn-live/e2e/bpm-builder.spec.ts:26,40,47,50` |
| **Индикатор «ассистент печатает» от бэкенда** | `rn-live/rn-chat/src/views/__tests__/ChatView.assistantTyping.guard.test.ts:7` |
| **Markdown-разметка в ответах ассистента** | `rn-live/rn-chat/src/utils/__tests__/inlineMarkup.test.ts:4,12,30` |
| **ИИ в компактном ответе на письмо** (InlineReplyBox / FloatingReplyDock) | `rn-live/rn-mail/src/components/InlineReplyBox.tsx:65,203,235,440`; `FloatingReplyDock.tsx:90` |
| **Полная русская локализация ассистента** (≈45 ключей) | `rn-live/rn-mail/src/i18n/ru.ts:148–194`, `rn-live/rn-chat/src/i18n/ru.ts:91–92` |
| **Уточняющие вопросы к письму (follow-up)** | `rn-live/rn-mail/src/i18n/ru.ts:160–166`; `rn-live/e2e/mail-assistant-real-probe.spec.ts:63–71` |
| **«Ассистентские чипы» в Диске** | `rn-live/rn-disk/src/views/DiskView.tsx:2385` |

---

## 2. Поисковый контур: настоящий семантический поиск или конфигурация под будущее

**Короткий ответ: механизм настоящий, но это не «поиск по смыслу для пользователя» — это
retrieval, который кормит ассистента. И он opt-in.**

### 2.1. Что именно построено

| Компонент | Доказательство | Что делает |
|---|---|---|
| **Векторная база** | `web/tools/deploy/host-scripts/docker-compose.ai-gateway.yml:88–89` — сервис `qdrant`, образ `qdrant/qdrant:latest`; `:97` том `qdrant-data:/qdrant/storage`; `:120` объявление тома | Настоящий Qdrant с персистентным хранилищем, а не мок |
| **Адрес из шлюза** | `docker-compose.ai-gateway.yml:56` — `AI_GATEWAY_QDRANT_URL: "${AI_GATEWAY_QDRANT_URL:-http://qdrant:6333}"`; дублируется в `web/DEPLOY.md:1108` | Шлюз ходит в Qdrant по внутренней сети compose |
| **Эмбеддинги** | `web/DEPLOY.md:1063–1064`: «`AI_GATEWAY_RAG_ENABLED=true` starts private Qdrant **and BGE-M3 embedding services** through the compose `rag` profile» | Модель эмбеддингов — **BGE-M3**, поднимается своим контейнером |
| **Изоляция контура** | `web/DEPLOY.md:1026`: «RAG services (`qdrant`, `embeddings`) run only through the private compose»; `:1057`: «RAG / memory services must stay backend-only. Expose Qdrant, embeddings, or…» | Векторная база наружу не торчит принципиально |
| **Что хранится** | `web/DEPLOY.md:1199`: «…fetched to create embeddings and discarded before Qdrant persistence» | **Сырой текст в Qdrant не пишется.** В базу уходит вектор + указатель на источник, оригинал выбрасывается |
| **Ручная раскатка** | `web/DEPLOY.md:1137`: `ssh uc 'cd /opt/ai-gateway && sudo env COMPOSE_PROFILES=rag docker compose up -d --force-recreate api qdrant embeddings'` | Инструкция написана под живой хост `uc` — контур поднимали руками, не гипотетически |
| **Что индексируется** | `web/DEPLOY.md:1151`: «…covered by the assistant policy. The scanner indexes bounded recent…» (строка обрезана в выгрузке) | Сканер индексирует **ограниченную свежую** выборку — не весь архив |
| **Внешний список источников** | `web/DEPLOY.md:1184`: `AI_GATEWAY_SCENARIO_REFS_FILE=/secure/path/assistant-sources.json` | Часть источников задаётся файлом на диске сервера |

Права доступа, TTL и порог отсечения (`ACL-fingerprint`, 30 дней, `MIN_SCORE=0.60`) в **мой**
срез не попали — они зафиксированы в предыдущем разборе `rn`
(`recon-ai-one-2026-08-03.md:78`). Своими глазами здесь я их не видел; запрос Q4.

### 2.2. Поисковый контур виден пользователю — и это новое

В сквозных тестах конструктора бизнес-процессов есть **экранные надписи** про семантику:

```
e2e/bpm-builder.spec.ts:26:  await expect(page.getByText('4 collaboration objects · 2 semantic sources')).toBeVisible();
e2e/bpm-builder.spec.ts:40:  await expect(page.getByText('Semantic Sources', { exact: true })).toBeVisible();
e2e/bpm-builder.spec.ts:47:  await expect(page.getByText('Indexed semantic sources', { exact: true })).toBeVisible();
e2e/bpm-builder.spec.ts:50:  await expect(page.getByText('Semantic readiness')).toBeVisible();
e2e/bpm-real-cross.spec.ts:100: await expect(page.getByText('Semantic Sources', { exact: true })).toBeVisible();
```

Заголовок теста: «runs document signing workflow with **collaboration and semantic context**»
(`e2e/bpm-builder.spec.ts:22`). То есть в UI есть панель «семантические источники», счётчик
проиндексированных источников и индикатор «семантической готовности», и это проверяется в том
числе тестом с суффиксом `-real` (`bpm-real-cross.spec.ts`), то есть против живого бэкенда.

### 2.3. И ассистент реально ждёт RAG-цитат

```
e2e/chat-assistant-real.spec.ts:45: test.describe('UC Assistant chat real RAG e2e @real', () => {
e2e/chat-assistant-real.spec.ts:51: test('caller answers inside the normal assistant direct chat with chat and mail RAG sources', …
```

Сквозной сценарий против живого бэкенда ожидает, что ответ ассистента опирается на источники
**из чатов и из почты одновременно**. Плюс карточки источников в почте:
`mail-right-rail-assistant-source-card` (`e2e/mail.spec.ts:864`,
`e2e/mail-chat-discussion-real.spec.ts:313`) — правда, в обоих местах ожидается `toHaveCount(0)`,
то есть в этих конкретных прогонах карточек источников быть НЕ должно.

### 2.4. Отсев ложных срабатываний — чтобы слово `semantic` не раздули

Из 20 попавших ко мне совпадений `semantic` **к поиску по смыслу относятся только 5** (§2.2).
Остальные 15 — слово в обычном значении:

- **токены дизайн-системы:** `rn-chat/src/theme.ts:5,135` («semantic groups» цветов),
  `rn-tasks/src/components/TaskContextMenu.tsx:11`, `rn-tasks/src/components/LinkedSourceCard.tsx:7`;
- **семантика даты/сущности:** `rn-tasks/src/bridge/__tests__/vtodoMapper.test.ts:242`,
  `docs/WORKAROUNDS-REGISTRY.md:1740`, `rn-tasks/src/hooks/__tests__/destinations.test.ts:152`;
- **семантика API/поведения:** `docs/WORKAROUNDS-REGISTRY.md:933,953`,
  `web/apps/app/src/app/features/rn-modules/rn-call-bridge-adapter.ts:675`,
  `rn-mail/src/components/EmailContextMenu.tsx:142`;
- **доступность:** `rn-chat/src/components/VoiceTranscribeButton.tsx:122`;
- **протокол WebRTC:** `a=msid-semantic:WMS …` в SDP-моке
  (`web/libs/common/web-rtc/src/lib/utils/sdp-bitrate-transformer/server-spd.mock.ts:8,121`).

Аналогично `embedding`: из 20 совпадений **19 — про встраивание RN в хост-приложение**
(«host embedding behavior», «Inline script embedding», `ReactRootView`), и лишь строки
`DEPLOY.md:1064,1199` — про векторные эмбеддинги. Слово-омоним, на нём легко ошибиться в обе
стороны.

---

## 3. Ассистент: что умеет и чем включается

### 3.1. Возможности (все — НАЙДЕНО)

| Возможность | Доказательство |
|---|---|
| **Ассистент в чате как обычный директ-чат** | `rn-chat/src/panels/ChatRoom.tsx:1` — `requestAssistantChatResponse`; `rn-chat/src/views/ChatView.tsx:294` — `mergeAssistantChat()` подмешивает строку «UC Assistant» в список чатов; `:956,976` — displayName `UC Assistant`; `:970` — резолв бота по e-mail |
| **Ответ бота сохраняется в истории чата** | `rn-chat/src/panels/ChatRoom.tsx:875` — `throw new Error('Assistant bot reply was not persisted')`; проверка: `e2e/chat-assistant-real.spec.ts:810` — `expect(body.botPost?.persisted, 'assistant bot reply should be persisted by IVA bot API').toBe(true)` |
| **Индикатор «ассистент думает»** | `rn-chat/src/views/__tests__/ChatView.assistantTyping.guard.test.ts:8,16` — «surfaces assistant response requests through the active room header», «mirrors the same assistant working state into the chat list typing map»; строка UI `'mail.rightRail.assistant.thinking': 'Думаю...'` (`rn-mail/src/i18n/ru.ts:182`) |
| **Markdown в ответах** | `rn-chat/src/utils/__tests__/inlineMarkup.test.ts:4,12,30` — жирный, курсив, инлайн-код, заголовки |
| **Сводка письма** | `rn-mail/src/i18n/ru.ts:185` — «Создать сводку», `:186` — «Суммировать письмо и показать полезные следующие шаги»; `:168–170` — разделы «Решения», «Блокеры», «Follow-up» |
| **Черновик задачи из письма** | `rn-mail/src/i18n/ru.ts:193` — «AI-предложение задачи», `:194` — «Готовим черновик задачи…»; `e2e/mail-assistant-task-real.spec.ts:86–104` — заголовок задачи → создание → «открыть в Задачах» |
| **Черновик ответа на письмо** | `rn-mail/src/utils/assistantDraftResponse.ts:105,235,281`; `rn-mail/src/i18n/ru.ts:171` — «Черновик ответа» |
| **Уточняющий вопрос к письму (follow-up)** | `rn-mail/src/i18n/ru.ts:160–166` («Спросите об этом письме…», «Спросить»); `e2e/mail-assistant-real-probe.spec.ts:63–71` |
| **Быстрые действия из сводки** | `rn-mail/src/i18n/ru.ts:171–174` — «Черновик ответа», «Создать событие», «Создать задачу», «Обсудить» |
| **ИИ в компактном ответе (inline-переписывание)** | `rn-mail/src/components/InlineReplyBox.tsx:65` — «Assistant gateway for compact compose AI actions»; тест `rn-mail/src/components/__tests__/InlineReplyBox.test.tsx:219` — «rewrites compact reply notes with assistant email context» |
| **Создание календарного события ассистентом** | `e2e/assistant-calendar-confirmation-real.spec.ts:24,49–56` — payload `proposalId`/`chatId`/`messageId`, реальное создание и уборка события |
| **Переход из почты в чат с ассистентом** | `rn-mail/src/i18n/ru.ts:188–189` — «Открыть чат с ассистентом», «Продолжить в чате» |

Промпт-политика видна дословно (`rn-mail/src/utils/assistantDraftResponse.ts:235`):

> `'Use the assistant brief when provided, and use only the selected email plus authorized source snippets.'`

«**only … authorized source snippets**» — ограничение по правам доступа зашито в сам промпт.

### 3.2. Чем включается и выключается

| Флаг / параметр | Где | Значение |
|---|---|---|
| `assistant-enabled` (атрибут хоста) | `web/apps/app/src/app/features/rn-modules/rn-communications-host.component.ts:59`; проверка в смоуке `e2e/communications-assistant-smoke.spec.ts:52` | Ассистент в коммуникациях включается атрибутом хоста |
| `RN_ASSISTANT_API_BASE` | `web/apps/app/src/app/settings.ts:735` = `https://uc.iva.ru/api/assistant/v1` | **Дефолт зашит на прод-адрес** |
| `RN_ASSISTANT_BOT_DISPLAY_NAME` | `web/apps/app/src/app/settings.ts:738` = `UC Assistant` | |
| `AI_GATEWAY_RAG_ENABLED` | `web/DEPLOY.md:1063` | RAG — **opt-in**, по умолчанию контур не поднимается |
| `AI_GATEWAY_IVA_BOT_API_TOKEN` | `web/DEPLOY.md:1082` — «optional but needed for persisted UC Assistant bot replies» | Без него ответ приходит в браузер, но **в историю чата не ложится** (`DEPLOY.md:1042`) |

Деградация без настройки описана явно: `rn-mail/src/i18n/ru.ts:175` — «Ассистент не настроен»,
`:150` — «Шлюз ассистента не настроен», `:151` — «Ассистенту нужна активная сессия»; в коде —
`rn-mail/src/utils/assistantDraftResponse.ts:224,227`. Тест это тоже допускает:
`e2e/mail.spec.ts:865` ожидает **любую** из трёх надписей —
`Assistant is not configured|Generate brief|Brief ready`.

---

## 4. Какая языковая модель и где она считает

**Это самый спорный пункт, и я развожу доказанное и недоказанное.**

### 4.1. Что доказано выгрузкой

| Факт | Строка |
|---|---|
| Раздел инструкции деплоя называется **«AI Gateway — DeepSeek assistant (S1)»** | `web/DEPLOY.md:1009` |
| «The AI assistant uses a dedicated Node service so **LLM provider credentials** and …» | `web/DEPLOY.md:1011` |
| Шлюз владеет `/api/assistant/v1/*` и **слушает только `127.0.0.1:8098`** | `web/tools/deploy/host-scripts/docker-compose.ai-gateway.yml:3` |
| Каталог развёртывания — `/opt/ai-gateway/`, «AI Gateway / DeepSeek assistant (§6.8)» | `web/DEPLOY.md:1493` |
| Состав: «api + optional private qdrant / embeddings services» | `web/DEPLOY.md:1494` |
| Публичный адрес шлюза | `web/DEPLOY.md:1172` — `AI_GATEWAY_PUBLIC_URL=https://uc.iva.ru/api/assistant/v1`; проверка живости `:1016` — `curl -sk https://uc.iva.ru/api/assistant/v1/healthz` |
| **В CI объявлена модель `openai/gpt-oss-120b`** | `web/.gitlab-ci.yml:166` — `export CONFIG__MODEL="openai/gpt-oss-120b"` |

### 4.2. Что из этого следует, а что нет

**Генерация — почти наверняка внешний провайдер, не локальные веса.** Опора — формулировка
`DEPLOY.md:1011`: «**LLM provider credentials**». Учётные данные провайдера нужны тому, кто
ходит наружу; локально поднятой модели ключ провайдера не нужен. Плюс название раздела —
DeepSeek, то есть конкретный внешний вендор. Это **сильный, но косвенный** довод: самой строки
с базовым URL провайдера и именем переменной ключа в моём срезе нет.

**Retrieval — точно локальный.** Qdrant и эмбеддинги BGE-M3 поднимаются своими контейнерами в
том же compose (`docker-compose.ai-gateway.yml:88–89`, `DEPLOY.md:1064,1137`) и наружу не
выставляются (`DEPLOY.md:1057`). Здесь сомнений нет: **векторная часть считается на своём
железе**.

**Про `openai/gpt-oss-120b` предупреждаю отдельно — на этой строке легко ошибиться.**
`gpt-oss-120b` — открытая модель с публичными весами, префикс `openai/` в такой записи означает
**имя семейства в реестре моделей** (так адресуются модели в OpenRouter и в OpenAI-совместимых
серверах вроде vLLM), а **не** обращение к API OpenAI. Но лежит эта строка в `.gitlab-ci.yml`,
то есть в **сборочном конвейере**, а не в рантайме шлюза. Правдоподобны минимум два чтения:
(а) конвейер прогоняет ИИ-проверку кода/ревью своей моделью; (б) это конфигурация модели самого
продукта, задаваемая на сборке. **Различить их по одной строке нельзя**, и выдавать (б) за факт
я не буду. Запрос Q5 — единственный способ закрыть.

**Итог для аудита, без натяжки:** утверждение «по коду не видно, облачная модель или локальная»
**больше не полностью верно**. Видно, что: (1) генерация вынесена в отдельный сервис, которому
нужны **учётные данные провайдера**; (2) вендор в документации назван — **DeepSeek**;
(3) векторный поиск и эмбеддинги — **локальные, в приватном compose**; (4) в CI фигурирует
**открытая модель `gpt-oss-120b`**, роль которой не установлена. Полного доказательства
(базовый URL провайдера + имя модели в рантайме) в срезе нет — оно в `/opt/ai-gateway/.env` и в
исходниках `connect/ai-gateway`, которых в выгрузке не было.

---

## 5. Зрелость

### 5.1. Сквозные тесты — их много, и они против живого бэкенда

В срезе **13 spec-файлов и 6 отдельных playwright-конфигов**, посвящённых ассистенту:

| Сценарий | Файл | Против чего |
|---|---|---|
| RAG-ответ в чате | `e2e/chat-assistant-real.spec.ts` (+ `e2e/playwright-chat-assistant-real.config.ts`) | **живой** (`@real`), `https://uc.iva.ru/api/assistant/v1` (`:16`) |
| Черновик письма | `e2e/chat-assistant-draft-real.spec.ts` (+ конфиг) | **живой** |
| Сводка + задача из письма | `e2e/mail-assistant-task-real.spec.ts` (+ конфиг) | **живой** Jump-бэкенд |
| Зонд почтового ассистента | `e2e/mail-assistant-real-probe.spec.ts` (+ конфиг) | **живой**, стрим `chat/stream` |
| Событие в календаре из подтверждения | `e2e/assistant-calendar-confirmation-real.spec.ts` (+ конфиг) | **живой**, с уборкой за собой |
| Обвязка хоста коммуникаций | `e2e/communications-assistant-smoke.spec.ts` (+ конфиг) | статическая проверка исходника хоста |
| Правая панель почты | `e2e/mail.spec.ts`, `e2e/mail-embedded.spec.ts` | мок (`page.route('**/api/assistant/v1/chat/stream'…)`, `mail-embedded.spec.ts:467`) |
| Семантические источники в BPM | `e2e/bpm-builder.spec.ts`, `e2e/bpm-real-cross.spec.ts` | смешанно |
| Обсуждение письма в чате | `e2e/mail-chat-discussion-real.spec.ts` | **живой** |

Таймауты выставлены под настоящую модель, а не под мок: 30 с, 90 с, **120 с**
(`e2e/mail-embedded.spec.ts:126`, `e2e/mail-assistant-real-probe.spec.ts:51`,
`e2e/mail-assistant-task-real.spec.ts:84`). Так пишут, когда с той стороны действительно долго
думают.

### 5.2. Но зелёный прогон здесь НЕ доказывает работу ассистента — важная оговорка

Все «живые» тесты **мягко пропускаются**, если бэкенд недоступен, и падают только при явном
жёстком флаге:

```
e2e/chat-assistant-real.spec.ts:797  … 'Set UC_ASSISTANT_CHAT_REAL_REQUIRED=1 to hard-gate this real backend flow.'
e2e/chat-assistant-draft-real.spec.ts:147 … 'Set UC_ASSISTANT_DRAFT_REAL_REQUIRED=1 …'
e2e/assistant-calendar-confirmation-real.spec.ts:129 … 'Set UC_ASSISTANT_CALENDAR_CONFIRMATION_REQUIRED=1 …'
```

Плюс `test.skip` по отсутствию учётных данных (`e2e/chat-assistant-real.spec.ts:48`,
`e2e/mail-assistant-task-real.spec.ts:56`, `e2e/mail-assistant-real-probe.spec.ts:25`).
**Вывод: наличие тестов доказывает проектную зрелость, но не факт работы контура.**
Доказательством работы был бы сохранённый результат прогона — см. §5.4 и запрос Q6.

### 5.3. Развёртывание

- Отдельный compose ИИ-шлюза: `web/tools/deploy/host-scripts/docker-compose.ai-gateway.yml`
  (сервисы `api`, `qdrant`, `embeddings`; том `qdrant-data`, `:120`).
- Отдельная цель деплоя: `web/DEPLOY.md:67` — «build + roll AI Gateway only (assistant hot-fix;
  SPA untouched)»; `:133` — «Only files under `ai-gateway/` … Builds `iva-ai-gateway:<tag>`,
  ships the image, recreates `/opt/ai-gateway/docker-compose.yml`, and smoke-tests …»,
  флаг `--skip-ai-gateway` (`:1030`).
- Смоук после деплоя: `curl -sk https://uc.iva.ru/api/assistant/v1/healthz` (`:1016`),
  проверка `/recommendations` (`:1020`).
- Метрики: `web/DEPLOY.md:1158` + `docs/HANDOFF-WEB-RN-INTEGRATION.md:32–38` —
  `[ai-gateway:metric] {"service":"assistant","event":"assistant_request",…}`, и отдельно
  оговорено, что аналитика ассистента **не** идёт через браузерный Umami.

**Это не прототип на коленке.** Отдельный образ, отдельный цикл выкатки, health-check,
санитизированные метрики и hot-fix-путь мимо SPA — признаки эксплуатируемого сервиса.

### 5.4. Косвенный след реальных прогонов

`e2e/chat-assistant-draft-real.spec.ts:127`:
`path.resolve(process.cwd(), '.deploy-state/assistant-real/email-draft-playwright-results.json')`

Результаты «живого» прогона пишутся в `.deploy-state/` — каталог состояния деплоя. Если эти
файлы физически лежат на сервере, они и есть **прямое доказательство, что ассистент отвечал на
живом контуре, с датой**. Запрос Q6 — по ценности для аудита он первый.

---

## 6. Чего не хватает, чтобы это стало продуктом

| Пробел | Доказательство |
|---|---|
| **Навигация по цитатам не сделана** — нельзя ткнуть в источник и перейти к нему | `e2e/source-citations-real.spec.ts:4` — `test.skip('assistant citation navigation needs stable source ids for chat, mail, and calendar results')`. Тест написан и **отключён**: нет стабильных id источников |
| **RAG выключен по умолчанию** — без него ассистент работает без корпоративной памяти | `web/DEPLOY.md:1063` — «RAG deployment is opt-in» |
| **Без токена бот-API ответы не сохраняются** в истории чата | `web/DEPLOY.md:1042` — «Without it, the browser still receives the assistant answer with…» (строка обрезана), `:1082` |
| **Follow-up в части сборок выключен** | `e2e/mail-embedded.spec.ts:149–150` — кнопка `…-action-ask-follow-up` и тред `…-follow-up-thread` ожидаются с `toHaveCount(0)` при видимом поле ввода (`:151`) |
| **Карточки источников в почте не показываются** в проверяемых сценариях | `e2e/mail.spec.ts:864`, `e2e/mail-chat-discussion-real.spec.ts:313` — `mail-right-rail-assistant-source-card` → `toHaveCount(0)` |
| **Планировщик встреч в композере события не сделан** | `docs/HANDOFF-WEB-RN-INTEGRATION.md:952` и `rn-calendar/HANDOFF.md:346` — «`fetchFreeBusy`/`fetchTimeSlots` are implemented in the bridge but no scheduling UI in the EventComposer yet» |
| **Конспект конференции и полировка стенограммы по-прежнему не сделаны** | `docs/WORKAROUNDS-REGISTRY.md:499` — `AI-WR-002: No LLM/GPT step in the existing Terra transcription pipeline; conference recap + transcript polish need iva-ai-bot to expose those endpoints`; `:506` — «neither layer has any LLM/GPT post-processing» |
| **ИИ-возможности спрятаны за capability-флагом, который возвращает `null`** | `docs/WORKAROUNDS-REGISTRY.md:489` — «All AI affordances stay **hidden** until the bot service exists. `AiBridgeApi.getCapabilities()` returns `null` when no service is registered for the active workspace» |
| **Сам ИИ-шлюз в этот репозиторий не входит** | В выгрузке нет ни одного файла из `ai-gateway/` — только его compose и раздел в `DEPLOY.md`. Код, который зовёт модель, лежит отдельно |

**Отдельно важно:** `docs/WORKAROUNDS-REGISTRY.md:488` в `rn-live` содержит **ту же** запись
`AI-WR-001` («The IVA Connect tree does not contain any LLM client, any GPT post-processing
pipeline, or an "AI bot" implementation»), что и месячный снимок `rn`. То есть **авторское
признание пробела в живой копии не отозвано и не переписано**. Это ограничивает толкование
находок: ИИ-контур живёт **рядом** с IVA Connect (надстройка `ione`/RN + внешний шлюз), а не
внутри самого Connect.

---

## 7. Что это меняет в выводе аудита «поиска по смыслу нет ни в одном поставляемом продукте»

Формулирую аккуратно, потому что тут легко перегнуть в обе стороны.

**Что подтверждается.** Внутри клиентского кода поиска по смыслу как не было, так и нет. Все
совпадения `semantic` в модулях `rn-*` — про токены темы, семантику дат и API, доступность и
SDP (§2.4). Локальный поиск в снимке `rn` был лексическим (MiniSearch), и **опровергнуть это
моим срезом нечем** — строк `rerank` в него не попало ни одной.

**Что опровергается.** Утверждение в формулировке «поиска по смыслу нет **нигде**» — неверно.
Векторный контур существует физически: Qdrant с персистентным томом, отдельный сервис
эмбеддингов на BGE-M3, инструкция подъёма на живом хосте `uc`, политика «сырой текст не
сохраняем», сквозные тесты, ожидающие RAG-источники из чатов и почты, и **видимая пользователю
панель «Semantic Sources / Indexed semantic sources / Semantic readiness»** в конструкторе
бизнес-процессов.

**Точная переформулировка, которую я предлагаю вместо прежней** (решение за лидом — я даю
только формулировку по фактам):

> Поиск по смыслу **не входит в коробочные продукты линии IVA Connect** и не реализован в
> клиентском коде — там лексический индекс. Но в надстройке `ione`/RN, развёрнутой на
> `uc.iva.ru`, построен полноценный векторный retrieval-контур (Qdrant + BGE-M3, приватный
> compose, TTL и ограничение по правам), который **питает ассистента, а не заменяет
> пользовательский поиск**, и **включается оператором** — по умолчанию выключен.

Три оговорки, без которых формулировка станет рекламой:

1. Это **retrieval для ассистента**, а не строка поиска «найди по смыслу». Отдельного
   пользовательского входа в семантический поиск в срезе не видно.
2. Индексируются **указатели на ограниченную свежую выборку**, а не архив
   (`DEPLOY.md:1151,1199`).
3. Контур **opt-in** (`DEPLOY.md:1063`), и «работает» ≠ «включено у заказчика».

---

## 8. ЗАПРОСЫ НА СЕРВЕР

Всё — в `/srv/helm/repos/rn-live`. По убыванию ценности.

**Q6 (самый ценный — доказательство, что ассистент реально отвечал).**
```
ls -la /srv/helm/repos/rn-live/.deploy-state/ 2>/dev/null
ls -la /srv/helm/repos/rn-live/.deploy-state/assistant-real/ 2>/dev/null
head -c 4000 /srv/helm/repos/rn-live/.deploy-state/assistant-real/*.json 2>/dev/null
```

**Q5 (модель — закрыть неоднозначность `gpt-oss-120b`).**
```
sed -n '120,220p' /srv/helm/repos/rn-live/web/.gitlab-ci.yml
grep -n 'CONFIG__\|MODEL=' /srv/helm/repos/rn-live/web/.gitlab-ci.yml
```

**Q7 (модель и адрес провайдера — прямое доказательство «облако или локально»).**
```
grep -rniE 'AI_GATEWAY_(LLM|MODEL|PROVIDER|OPENAI|DEEPSEEK|API_KEY|BASE_URL)[A-Z_]*' /srv/helm/repos/rn-live/web/ | head -60
sed -n '1,130p' /srv/helm/repos/rn-live/web/tools/deploy/host-scripts/docker-compose.ai-gateway.yml
sed -n '1000,1220p' /srv/helm/repos/rn-live/web/DEPLOY.md
```
Значения ключей **не нужны и не запрашиваются** — нужны только имена переменных и хост/URL.

**Q1 (переранжирование — 55 совпадений, ни одного в срезе).**
```
grep -rn --include='*.ts' --include='*.tsx' -i 'rerank' /srv/helm/repos/rn-live/ | grep -v node_modules | head -60
sed -n '1,80p' /srv/helm/repos/rn-live/rn-shared/src/search/SearchService.ts
grep -rn -i 'minisearch' /srv/helm/repos/rn-live/rn-shared/src/search/ | head
```
Ключевой вопрос: `rerank()` — это сортировка лексических хитов или обращение к
cross-encoder / вектору.

**Q2 (что это за копия: ветка, дата, версия).**
```
git -C /srv/helm/repos/rn-live log -5 --date=iso --pretty='%h %ad %an %d %s'
git -C /srv/helm/repos/rn-live status -sb | head -3
git -C /srv/helm/repos/rn-live remote -v
head -40 /srv/helm/repos/rn-live/package.json
```

**Q3 (насколько «живее» снимка `rn` — есть ли свежие ИИ-коммиты).**
```
git -C /srv/helm/repos/rn-live log --since=2026-06-25 --date=short --pretty='%ad %s' | grep -iE 'assist|rag|qdrant|embed|llm|gateway|semantic|rerank' | head -40
```

**Q4 (политика RAG: права, TTL, порог — своими глазами).**
```
grep -rn 'AI_GATEWAY_RAG\|MIN_SCORE\|ACL\|TTL\|SOURCE_TYPES\|RETENTION' /srv/helm/repos/rn-live/web/tools/deploy/host-scripts/docker-compose.ai-gateway.yml /srv/helm/repos/rn-live/web/DEPLOY.md | head -50
```

**Q8 (панель «Semantic Sources» — где реализация, а не тест).**
```
grep -rn 'Semantic Sources\|Indexed semantic sources\|Semantic readiness\|semantic source' /srv/helm/repos/rn-live/rn-shared /srv/helm/repos/rn-live/web/apps --include='*.ts' --include='*.tsx' --include='*.html' | head -30
```

**Q9 (есть ли рядом исходники самого шлюза — их не было ни в одном снимке).**
```
ls -la /srv/helm/repos/ | head -40
ls -la /srv/helm/repos/rn-live/ | head -40
find /srv/helm/repos -maxdepth 3 -type d -name 'ai-gateway' 2>/dev/null
```

**Q10 (прод-конфиг: включён ли ассистент и RAG у заказчика).**
```
grep -n 'RN_ASSISTANT\|RN_BPM\|RN_MINIAPPS\|AI_' /srv/helm/repos/rn-live/web/apps/app/src/assets/config/config.prod.json
```

---

## 9. Итог одной строкой

`rn-live` — тот же монорепозиторий `ione`, но рабочей копией на сервере и с пригодными для
цитирования номерами строк; ИИ-контур в нём **шире, чем показал разбор снимка `rn`** (стрим
`chat/stream`, follow-up к письму, ИИ в компактном ответе, панель семантических источников в
BPM, полная русская локализация ассистента, индикатор «печатает»), векторный поиск **реально
построен и локален**, генерация **почти наверняка внешняя (DeepSeek по документации)**, но
окончательное доказательство про модель лежит вне репозитория — в `/opt/ai-gateway/.env` и в
исходниках `connect/ai-gateway`, и его надо дозапросить (Q5, Q7).

## Relations
- relates_to [[recon-ai-one-2026-08-03]]
- relates_to [[recon-ai-corpus-2026-08-03]]
