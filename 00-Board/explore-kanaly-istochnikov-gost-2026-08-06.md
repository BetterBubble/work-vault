---
title: Разведка — каналы доступа к источникам (Jira/Confluence/файлы) под ГОСТ-документацию
type: report
status: draft
created: 2026-08-06
tags:
- board
- gost-docs
- explore
permalink: tacticum/00-board/explore-kanaly-istochnikov-gost-2026-08-06
---

# Каналы доступа к источникам: чем читаем, чем пишем, что прибито к ИВА

**Метод:** только чтение. Репозитории `helm`, `platform`, `tacticum-dev`, `project-hub`,
`doc_translator` + заметки vault. На серверы не ходил, ничего не запускал.

**Важно про вершины.** Локальные клоны отстают от `origin`, и живой код iva-write в
рабочем дереве отсутствует. Всё ниже читал из `origin/main`:

| Репозиторий | локальный HEAD | `origin/main` |
|---|---|---|
| `~/tacticum/helm` | `5c52811` | `708cac5` (Merge PR #114 `fix/iva-write-bot-intent`) |
| `~/tacticum/tacticum-dev` | `b612fd90` (03.08) | `cd0ddfc7` |
| `~/tacticum/platform` | рабочее дерево | — |

---

## 1. iva-write — канал записи (Jira + Confluence)

**Где код.** Репозиторий `helm`, всё в `origin/main`, в рабочем дереве этих файлов нет:

| Файл | Что делает |
|---|---|
| `helm/src/helm/interface/mcp/iva_write_surface.py` | MCP-поверхность канала. `MOUNT_PATH = "/mcp/iva-write"` (`:78`) |
| `helm/src/helm/interface/mcp/iva_write_tools.py` | два собственных тула: `iva_write_connect`, `iva_write_issue_key` (`:29-33`) |
| `helm/src/helm/application/iva_write.py` | поток согласия OAuth: `start_consent` (`:158`), `complete_consent` (`:347`), `refresh_credential` (`:458`), `issue_credentials` (`:652`) — единственная функция, достающая секрет из хранилища |
| `helm/src/helm/interface/api/routers/iva_write.py` | ручки согласия `/api/iva/oauth/start\|callback\|status\|revoke\|refresh` (`:75, :240, :299, :577, :619, :648`) + выдача ключа `/api/iva/key/{handout}` (`:80, :496`) |
| `helm/src/helm/application/bot_iva_write.py`, `helm/src/helm/infrastructure/bot_iva_write/` | бот выдачи доступа (мессенджер) |
| `helm/src/helm/main.py:60,181-218` | монтирование sub-app и роутеров |
| `helm/alembic/versions/bot330_bot_iva_write.py`, `mrg350_iva_write_pmb.py` | миграции |

**Что умеет.** Канал — не свой клиент Atlassian, а **прокси перед контейнером
`ghcr.io/sooperset/mcp-atlassian:0.23.0`** (`helm/docker-compose.prod.yml:156-157`).
То есть доступен весь тулсет `mcp-atlassian`: создание/правка страниц Confluence,
заведение и обновление задач Jira, комментарии, вложения. Плюс два своих тула
(`iva_write_tools.py`): `iva_write_connect` — проверить готовность канала и получить
ссылки на согласие; `iva_write_issue_key` — выпустить человеку новый ключ Tacticum.
Поверхность подмешивает их в `tools/list` и перехватывает по имени (`iva_write_tools.py:12-16`,
`iva_write_surface.py:675 _handle_our_tool`, `:711 _rpc_tools`).

**От чьего имени.** От имени самого сотрудника, не робота. Человек один раз даёт согласие
по OAuth 2.0 (Atlassian **Data Center**, пути `/rest/oauth2/latest/authorize|token` —
`helm/src/helm/infrastructure/iva_oauth/client.py:47-49`), его личный токен хранится
зашифрованным и подставляется на каждый запрос парой заголовков
(`iva_write_surface.py:92-99`: `X-Atlassian-Jira-Url` / `X-Atlassian-Jira-Personal-Token`
и то же для Confluence). Наш `Authorization` наверх **вырезается** (`:100-116`) — иначе
`mcp-atlassian` молча ушёл бы на глобальную учётку. Окружение контейнера намеренно пустое
по кредам (`docker-compose.prod.yml:148-155`, там же список «что сюда нельзя добавлять никогда»).

**Адрес канала:** `https://helm.tacticum.ru/mcp/iva-write`.
Ловушка (записана в манифесте лейна, `tacticum-dev` `templates/iva-write-base/manifest.yaml:15-17`):
прежний адрес `mcp.tacticum.ru/iva-write/mcp` **не отвечает**, но манифест с ним валидируется.

**Как канал доезжает до ролей.** Лейн `iva-write-base` (создан 05.08, в `origin/main`
`tacticum-dev`): `templates/iva-write-base/manifest.yaml` — ингредиент `helm-iva-write`
(`:120-133`), `transport: http`, `url: https://helm.tacticum.ru/mcp/iva-write`,
`env_required: [TACTICUM_TOKEN]`, `auth_type: bearer`. Отдельного ключа для записи нет —
тот же личный `phk_*`. Лейн несёт канал **и тексты к нему** (`instruction_pack` для
`CLAUDE.md`/`AGENTS.md`), подключается ролям одной строкой в `depends_on`.
Документация: `docs/user_manuals/iva-write-base-profile-quickstart.md`,
`docs/runbooks/iva-write-rollout-to-roles.md`, `docs/adr/0058-...md`.

**Предшественник, который ещё жив в части лейнов:** `iva-atlassian-write` —
локальный `uvx mcp-atlassian` по stdio на **личном PAT** каждого человека
(`templates/iva-analysis-fr/manifest.yaml:240-250`, env `JIRA_URL`, `JIRA_PERSONAL_TOKEN`,
`CONFLUENCE_URL`, `CONFLUENCE_PERSONAL_TOKEN`). Объявлен также в `iva-architect-mcp`,
`iva-qa-mcp`, `iva-techwriter-mcp`. Важно: **`allowed_tools` в манифестах нигде не
исполняется** — ни рендерерами, ни шлюзом (разбор в [[iva-write-picture-2026-07-27]] §1.3),
так что «ролевые ограничения» там комментарий, а не механизм.

---

## 2. Read-каналы — чем читаем Jira и Confluence

Каналов чтения **три, и они разной природы**.

### 2.1. `iva-read` — живое чтение Jira/Confluence ИВА

- **Что это:** тот же `mcp-atlassian`, но **read-only инстанс на ADP-хосте заказчика**,
  доступный через обратный SSH-туннель `:19010`.
- **Адрес для агента:** `https://mcp.tacticum.ru/iva-read/mcp`
  (`tacticum-dev/templates/tacticum-analysis-core/manifest.yaml:192-202`,
  `env_required: [TACTICUM_TOKEN]`, `auth_type: bearer`).
- **Конфиг шлюза:** `platform/services/runtime/mcp_runtime/deploy/traefik-mcp-gateway.example.yml`
  — роутер `mcp-iva-read` (`:45-50`), сервис `svc-iva-read → http://172.20.0.1:19010` (`:54`),
  цепочка middleware `forward-auth → ratelimit(10/30) → drop-auth → strip-prefix` (`:49`).
  `mcp-drop-auth` (`:19-20`) снимает клиентский Bearer: backend ходит по **сервисному PAT
  из своего env**, а не от имени человека.
- **Код резолва ключей:** `platform/services/runtime/mcp_runtime/src/mcp_runtime/infrastructure/dev_catalog.py`
  (`DEV_KEY_SCOPE = "iva-read"`, `:30-32`), `.../resolver.py:5`, `.../composition.py:37`,
  телеметрия `.../infrastructure/telemetry.py:1-3`.
- **Тулы (по комментарию манифеста, `templates/iva-system-analyst/manifest.yaml:93-97`):**
  `jira_search`, `jira_get_issue`, `confluence_search`, `confluence_get_page` и т.д. — read-срез
  `mcp-atlassian` против Jira 10.3 / Confluence 9.2.
- **Известный дефект:** `iva-read` отвечает **403** — тенант `iva` в записи реестра против
  `tacticum` в ключе ([[report-iva-write]], раздел «Что заблокировано»).

### 2.2. `helm-analyst` — факт-база, а НЕ живой Jira

- **Адрес:** `https://helm.tacticum.ru/mcp/analyst`
  (`tacticum-dev/templates/tacticum-analysis-core/manifest.yaml:180-190`).
- **Код:** `helm/src/helm/interface/mcp/analyst_server.py` (`analyst_search` `:371`,
  `analyst_context` `:389`, `docs_ask` `:430`, всего ~19 тулов).
- **Ключевое отличие:** это **оффлайн-индекс**, а не запрос в Jira в момент вызова.
  Данные приезжают ингестом (`helm/src/helm/ingest/rag2_extract.py`) — постраничная выгрузка
  REST Jira/Confluence через SSH-туннель в Qdrant/Meili, с курсором инкрементального
  до-индексирования (`docker-compose.prod.yml:94-97`, `data/real/rag2`).
  Вложения pdf/docx в ингесте **не разбираются** — `rag2_extract.py:15`:
  «Вложения pdf/docx — TODO(Ф2), см. `infrastructure/rag2/extractors.py::TODO_SUFFIXES`».

### 2.3. `tacticum-mcp` (kb_*) — знание о коде, не о вики

`https://mcp.tacticum.dev/mcp`, тулы `kb_discover`, `kb_get_task_context`,
`kb_get_code_context`, `kb_verify_api_exists` и др.
Наполняется отдельным пайплайном: репозиторий `KB-Brownfield-Bootstrap` (пакет `tacticum_re`,
команда `kb-upload`, `cli/main.py:475`) → приём и индексация в
`tacticum-dev/apps/backend/src/backend/knowledge/`. Подробно — [[explore-analyst-indexing-2026-08-03]].
К Jira/Confluence отношения не имеет.

---

## 3. Привязка к инстансу — главный вопрос

**Хорошая новость: адреса систем НЕ зашиты в код, это настройки.**
`helm/src/helm/config.py:586-650` и `helm/.env.example:19-42`:

| Переменная | Назначение |
|---|---|
| `HELM_IVA_OAUTH_JIRA_BASE_URL` / `..._CONFLUENCE_BASE_URL` | логический адрес систем (Host, SNI, ссылки) |
| `HELM_IVA_OAUTH_JIRA_CONNECT_URL` / `..._CONFLUENCE_CONNECT_URL` | физический адрес подключения (локальный конец SSH-туннеля) |
| `HELM_IVA_OAUTH_JIRA_CLIENT_ID` / `_SECRET`, то же для Confluence | регистрация OAuth-приложения, **раздельная у каждой системы** |
| `HELM_IVA_OAUTH_PUBLIC_BASE_URL` | наш адрес, из него собирается `redirect_uri` — обязан посимвольно совпасть с регистрацией |
| `HELM_IVA_WRITE_MCP_URL`, `HELM_IVA_WRITE_JIRA_URL`, `HELM_IVA_WRITE_CONFLUENCE_URL` | адреса для контейнера `mcp-atlassian` (`docker-compose.prod.yml:81-83`) |
| `HELM_IVA_OAUTH_SCOPE` (`iva_oauth_scope`, дефолт `WRITE`), `..._PKCE_ENABLED`, `..._VERIFY_TLS` | параметры потока |

**Плохая новость: прибито другое — тип и топология, а не имя хоста.**

1. **Только Atlassian Data Center.** `iva_oauth/client.py:3-6` — пути DC зашиты явно,
   у Cloud другой поток. Чужой отдел на Cloud → канал не заработает без нового кода.
2. **Ровно две системы.** `SYSTEMS = frozenset(WHOAMI_PATHS)` = `{jira, confluence}`
   (`client.py:52-59`). Третьей системы (например, другого трекера) модель не знает.
3. **Одна пара «инстанс Jira + инстанс Confluence» на весь helm.** Настройки скалярные,
   не по тенантам: второй контур одновременно с ИВА нынешним кодом **не поднимется** —
   нужен либо второй деплой helm, либо перевод настроек в мультиинстансные.
4. **Сетевой путь через туннель.** `docker-compose.prod.yml:176-178` —
   `extra_hosts: jira.iva.ru:172.18.0.1`, `wiki.iva.ru:172.18.0.1`; systemd-юнит
   `iva-sources-tunnel`. Под чужой контур нужен свой туннель/маршрут и свои `extra_hosts`.
5. **Регистрация OAuth-приложения — действие администратора чужого контура.** Он выдаёт
   `client_id`/`client_secret` и вписывает наш `redirect_uri`. Это внешняя зависимость,
   не наша работа.
6. **`iva-read` прибит жёстче:** upstream — конкретный порт туннеля в traefik-конфиге
   (`svc-iva-read → 172.20.0.1:19010`), сервисный PAT в env инстанса на ADP
   (`/opt/iva-mcp/env`, см. `helm/scripts/extract_confluence_15.py:11`). Под другой контур —
   отдельный инстанс `mcp-atlassian`, отдельный роутер, отдельная запись реестра.

**Что придётся сделать под чужой контур (запись):** отдельный деплой канала (или
мультиинстансность в конфиге) + туннель/сетевой доступ + регистрация OAuth-приложения
администратором того контура + четыре секрета + новый `mcp_server_spec` в лейне с другим
`url`. **Кода при этом почти не трогается — если у них Data Center.** Если Cloud — это
новый поток аутентификации, то есть настоящая разработка.

---

## 4. Файловые источники — что читает документы с диска

**В контуре ИВА/аналитика — ничего.** Ни watch-папки, ни загрузчика документов из каталога
в helm/platform/tacticum-dev я не нашёл. Ингест RAG#2 ходит по REST, вложения pdf/docx
явно помечены как TODO (`rag2_extract.py:15`).

**Но за пределами этого контура готовые инструменты есть, и их два класса.**

### 4.1. `doc_translator` (`~/tacticum/doc_translator`) — полноценный экстрактор документов

Платформа перевода с извлечением текста и обратной сборкой с сохранением вёрстки.
Экстракторы по форматам — `backend/app/domains/processing/extractors/`:
`docx_extractor.py`, `pdf_extractor.py`, `pptx_extractor.py`, `xlsx_extractor.py`,
`xls_extractor.py`, `rtf_extractor.py`, `odt_extractor.py`, `epub_extractor.py`,
`md_extractor.py`, `srt_extractor.py` + конвертеры legacy-форматов
(`doc_converter.py`, `ppt_converter.py`, `xls_converter.py` — через LibreOffice).
Плюс `backend/app/domains/documents/text_extractor.py`.
Вход — загрузка файла через API, хранение в MinIO, обработка Celery-джобой
(`README.md`, раздел «Primary flow»). **Watch-папки нет** — только upload.

### 4.2. `project-hub/word-mcp` — чтение И запись `.docx` как MCP-тулы

Обёртка над `SecurityRonin/docx-mcp`, деплой `https://project.cifragen.ru/word`.
Это самое близкое к «генерации ГОСТ-документа» из того, что у нас есть:
- **чтение:** `open_document`, `get_headings`, `search_text`, `get_paragraph`, `get_tables`,
  `get_styles`, `get_properties`, `get_images`, `audit_document`;
- **запись:** `create_document`, `create_from_markdown`, `insert_text`, `set_formatting`,
  `add_table`, `modify_cell`, `add_list`, `add_footnote`, `edit_header_footer`,
  `add_page_break`, `add_section_break`, `add_cross_reference`, `merge_documents`,
  `set_document_protection`;
- **шаблоны (наша надстройка):** `upload_template`, `list_templates`, `delete_template`,
  `create_from_template` — шаблоны `.docx`/`.dotx` в `/data/templates/`
  (`word-mcp/README.md`, раздел «Custom», `src/word_mcp_wrapper/templates_tools.py`);
- **файлы:** каталог `WORD_DATA_DIR` (`/data`), HTTP `GET /files` и `GET /files/{name}`;
  выгрузка в MinIO с presigned-ссылкой (`src/word_mcp_wrapper/minio_tools.py`).

Рядом: `project-hub/excel-mcp` (`read_excel`, `write_excel`, `update_excel`,
`get_sheet_names`, `analyze_excel`), `project-hub/wiki-mcp` (Wiki.js GraphQL),
`project-hub/taiga-mcp` (~35 тулов трекера Taiga), `project-hub/arch-mcp`,
`project-hub/transcription-mcp`.

### 4.3. Приём произвольного текста в индекс

`platform/services/data/knowledge_rag` — MCP `knowledge` (`knowledge.search`,
`knowledge.ingest`, `knowledge.list_collections`, реестр
`platform/services/runtime/mcp_runtime/registry.json:12-21`).
`application/ingest.py:1-11` — режимы `semantic` / `fulltext` / `passthrough`,
каждая запись штампуется тенантом. **Своего блоб-хранилища нет** — принимает уже
извлечённый текст, файлы не парсит.

### 4.4. Скрипты, читающие с диска/хоста

`helm/scripts/` — `collect_adp_digitized.py` (обход клонов на ADP-хосте, read-only),
`extract_confluence_15.py` (GET `body.storage` страниц Confluence, креды из
`/opt/iva-mcp/env`), `extract_jira_tasks_local.py`, `ingest_knowledge.py`,
CSV-лоадеры в `helm/src/helm/ingest/` (`csv_source.py`, `loader.py`).

---

## 5. Секреты и доступы

Карта имён — [[dostupy-karta]] (`20-Architecture/dostupy-karta.md`). Значения только в
Keychain macOS (`secret get <имя>`) и в `/opt/helm/.env` на сервере; в git — никогда.

**Уже заведено (имена):**

| Имя / переменная | От чего | Где живёт |
|---|---|---|
| `TACTICUM_TOKEN` (`phk_*`) | личный ключ человека: `helm-iva-write`, `helm-analyst`, `iva-read`, `tacticum-mcp` | env машины человека |
| `iva-mcp-token` | read-канал Confluence+Jira заказчика, `mcp.tacticum.ru` | Keychain (карта доступов) |
| `helm-analyst-token` | собственный MCP аналитика | Keychain |
| `iva-confluence-pat` | сервисный **read-only** PAT Confluence | на `adp_emb`, `/opt/iva-mcp/env` |
| `HELM_KEYSTORE_MASTER_KEY` | шифрование хранилища личных доступов (`openssl rand -base64 32`). Потеря = все согласия заново | `/opt/helm/.env` |
| `HELM_IVA_OAUTH_JIRA_CLIENT_ID` / `_SECRET` | регистрация OAuth-приложения в Jira DC | `/opt/helm/.env` |
| `HELM_IVA_OAUTH_CONFLUENCE_CLIENT_ID` / `_SECRET` | то же для Confluence DC (регистрация раздельная) | `/opt/helm/.env` |
| `HELM_IVA_WRITE_SERVICE_KEY` | служебный ключ внутренней ручки шлюза; не задан → 401 fail-closed (`config.py:576-582`) | `/opt/helm/.env` |
| `HELM_RAG2_JIRA_USER` / `_PASSWORD` | служебная учётка ингеста; ею же бот сверяет логин→почту | `/opt/helm/.env` |
| `HELM_IVA_BOT_IVA_WRITE_TOKEN` / `_WEBHOOK_SECRET` | бот выдачи доступа | `/opt/helm/.env` |
| `JIRA_PERSONAL_TOKEN` / `CONFLUENCE_PERSONAL_TOKEN` (+ `JIRA_URL`, `CONFLUENCE_URL`) | старый канал `iva-atlassian-write` (личный PAT, stdio) | env машины человека |
| `MCP_API_KEY` | Bearer к `word-mcp`/`excel-mcp`/`taiga-mcp`, `/opt/project/infra/.env` | сервер `project.cifragen.ru` |
| `taiga-token`, `wiki-token` | трекер Taiga, Wiki.js `cifragen` | Keychain |

**Что понадобится завести под чужой контур:**
1. `client_id` + `client_secret` для их Jira **и отдельно** для их Confluence — выдаёт их
   администратор, он же вписывает наш `redirect_uri`.
2. Сетевой доступ: туннель/маршрут до их инстансов + `extra_hosts` для контейнера.
3. Если нужен и read-канал — сервисный read-only PAT их контура + свой инстанс
   `mcp-atlassian` + запись в реестре шлюза.
4. Ключ шифрования хранилища переиспользуется (он наш, не их).
5. Никаких новых ключей на стороне людей: `TACTICUM_TOKEN` остаётся тем же.

---

ВЕРДИКТ: **Читаем тремя каналами** — `iva-read` (живой read-only `mcp-atlassian` через
шлюз `mcp.tacticum.ru`, сейчас отдаёт 403 из-за тенанта), `helm-analyst` (оффлайн-индекс
Jira/Confluence, не живой запрос) и `tacticum-mcp`/`kb_*` (про код, не про вики).
**Пишем одним** — `iva-write` (`https://helm.tacticum.ru/mcp/iva-write`, прокси перед
`mcp-atlassian` с личным OAuth-доступом человека) и его предшественником
`iva-atlassian-write` (локальный stdio на личном PAT, ещё живёт в части лейнов).
**Файловых источников в контуре ИВА нет вовсе**, но за его пределами есть `doc_translator`
(экстракторы docx/pdf/pptx/xlsx/rtf/odt/epub) и `word-mcp` (чтение и **запись** `.docx`
с шаблонами — прямой кандидат под генерацию ГОСТ-документа).
**Под чужой контур:** адреса и креды конфигурируются (`HELM_IVA_OAUTH_*`), прибиты не
хосты, а три вещи — только Atlassian **Data Center**, ровно **две системы** (jira+confluence),
**одна пара инстансов на весь helm** (мультитенантности нет). Работа: отдельный деплой или
мультиинстансность конфига + туннель + регистрация OAuth-приложения их администратором +
новый `mcp_server_spec` в лейне. Кода почти не трогаем — **если у них Data Center**;
Atlassian Cloud = новый поток аутентификации, то есть настоящая разработка.

Проверено: `helm` (`origin/main` `708cac5`) — `iva_write_surface.py`, `iva_write_tools.py`,
`application/iva_write.py`, `routers/iva_write.py`, `iva_oauth/client.py`, `config.py`,
`.env.example`, `docker-compose.prod.yml`, `ingest/rag2_extract.py`, `analyst_server.py`,
`scripts/`; `platform` — `mcp_runtime` (traefik-конфиг, `registry.json`, `dev_catalog.py`),
`knowledge_rag`; `tacticum-dev` (`origin/main` `cd0ddfc7`) — `templates/iva-write-base/`,
`tacticum-analysis-core`, `iva-analysis-fr`, `iva-system-analyst`; `project-hub` —
`word-mcp`, `excel-mcp`, `wiki-mcp`, `taiga-mcp`; `doc_translator` — экстракторы;
vault — `dostupy-karta`, `report-iva-write`, `iva-write-picture-2026-07-27`,
`grabli-iva-write-2026-08-05`, `explore-analyst-sources/indexing-2026-08-03`.

Данные: каналов к Jira/Confluence — **4** (`iva-write`, `iva-atlassian-write` (legacy),
`iva-read`, `helm-analyst`); систем в модели — **2** (Jira DC, Confluence DC);
собственных тулов канала записи — **2**; форматов документов у `doc_translator` — **10**
экстракторов + 3 конвертера; MCP-серверов `project-hub` — **7**;
env-переменных под контур ИВА в `.env.example` — **14**.

Подтверждение: `/Users/bubblemac/tacticum/helm/src/helm/interface/mcp/iva_write_surface.py`
(в `origin/main`) · `/Users/bubblemac/tacticum/helm/src/helm/config.py:586-650` ·
`/Users/bubblemac/tacticum/helm/.env.example:11-60` ·
`/Users/bubblemac/tacticum/helm/docker-compose.prod.yml:77-83,148-179` ·
`/Users/bubblemac/tacticum/platform/services/runtime/mcp_runtime/deploy/traefik-mcp-gateway.example.yml:17-54` ·
`/Users/bubblemac/tacticum/tacticum-dev/templates/iva-write-base/manifest.yaml` (в `origin/main`) ·
`/Users/bubblemac/tacticum/project-hub/word-mcp/README.md` ·
`/Users/bubblemac/tacticum/doc_translator/backend/app/domains/processing/extractors/`

НЕ проверено:
- **Живая работоспособность** ни одного канала — на серверы не ходил, `curl` не делал.
- **Что за контур у смежного отдела:** Data Center или Cloud, тот же `jira.iva.ru` или
  другой сервер, другое пространство или другой инстанс. От этого зависит вся оценка §3.
- **Тулсет `mcp-atlassian` 0.23.0 поимённо** — читал наши обёртки и манифесты, исходники
  пакета не открывал.
- **`project-hub` на проде:** развёрнут ли `word-mcp` сейчас, кому выдан `MCP_API_KEY`.
- **`doc_translator` как источник для агента:** MCP-интерфейса у него нет, только HTTP API;
  можно ли его звать агентом — не выяснял.
- **403 у `iva-read`** — причину взял из [[report-iva-write]], сам не воспроизводил.
- Ветки, кроме `origin/main`; рабочие деревья `.claude/worktrees` игнорировал.