---
title: Разведка — helm docs-чат как движок под чужой корпус (ГОСТ-документация)
type: report
status: draft
tags:
- board
- gost-docs
- rag
- explore
permalink: tacticum/00-board/explore-helm-kb-gost-2026-08-06
---

# Разведка: можно ли положить чужой корпус на механизм helm docs-чата

Дата: 2026-08-06. Роль: explorer (read-only). Репо: `/Users/bubblemac/tacticum/helm`
(Serena активирована, python). Сервер: `helm` 159.194.233.33 — только чтение.

---

## 1. Что такое helm.tacticum.ru/docs

**Уточнение по URL: страницы `/docs` нет.** Живой чат по документации ИВА — это
**`https://helm.tacticum.ru/iva-docs`** (проверено: `/iva-docs` → 200, `/docs` → 404).
Путь `/docs` был занят Swagger'ом и страницу с него увели:
`src/helm/main.py:245-252` (комментарий прямо об этом), отдаётся `web/dist/docs.html`.
Рядом `/analyst` → 200 (`src/helm/main.py:259-261`, RAG#2 для аналитиков).

- **Сервис:** это не отдельный сервис, а **модуль внутри монолита helm** (FastAPI).
  Один процесс отдаёт и API, и SPA (`src/helm/main.py:263` — `StaticFiles` на `/`).
- **API:** `POST /api/docs/ask` — `src/helm/interface/api/routers/docs.py:1-120`.
  Публичный, без idP-гейта (решение владельца 2026-07-15, `src/helm/main.py:136-139`).
- **Внутреннее имя:** RAG#1 (ассистент документации ИВА).
- **Конвейер:** `DocsSearch` (dense Qdrant + fulltext Meili + RRF) → reranker →
  генерация внутренним vLLM `triva`. Сборка контекста —
  `src/helm/infrastructure/docs_assistant/service.py:50-120`.
- **Что индексирует сейчас:** ТОЛЬКО публичная документация `iva.ru/docs` — 2145 страниц
  9 продуктов (MCU, IVA One, CS, Terra, SBC, Room, Mail, Updates, Portal). Корпус лежит
  локально: `/Users/bubblemac/tacticum/iva-rag1-docs/` (`manifest.json` + `md/**`).
- **Как отдаёт ссылки:** payload чанка самодостаточен (`url`, `title`, `slug`,
  `heading_path`, `product`, `version`, `text`) — `docs_assistant/vectorstore.py:1-9`,
  `ingest/docs_index.py:116-131`. В ответе цитаты `[n]` строит `domain/docs.build_citations`,
  схема — `routers/docs.py:86-96` (`DocCitationOut`). Живая проверка ниже даёт реальные
  URL на `iva.ru/docs/...`.

## 2. Ingestion — как корпус попадает в индекс

Главный файл: **`src/helm/ingest/docs_index.py`** (257 строк, весь пайплайн).

Поток (`docs_index.py:1-16` — докстрока описывает его точно):
`manifest.json` (срез `version=latest` + `kind=doc`, строка 54-57) → `parse_frontmatter`
→ `structural_chunks` (чанк по heading-пути) → near-dup разметка → богатый payload
(строки 116-131) → эмбеддинги bge-m3 через Gateway батчами по 4 (строка 39, 160-168) →
upsert Qdrant + best-effort Meili (строки 179-193).

CLI: `uv run python -m helm.ingest.docs_index --docs DIR [--limit N] [--dry-run]`
(`docs_index.py:234-253`). Требует `HELM_GATEWAY_BASE_URL`, `HELM_GATEWAY_API_KEY`,
`HELM_QDRANT_URL`, каталог корпуса.

**Формат входа — жёсткий:** каталог с `manifest.json` (поля `slug,title,url,product,
section,version,word_count,kind,md_path`) + `.md` с YAML-frontmatter. Не «папка с
файлами».

**Форматы файлов.** Отдельного docproc в helm НЕТ — `docs_index.py:1` прямо пишет
«минуя docproc». Единственный экстрактор — `src/helm/infrastructure/rag2/extractors.py`,
и он написан ТОЛЬКО под вложения Confluence в RAG#2:
- умеет: `xlsx/xlsm` (openpyxl), `pdf` (pypdf, **без OCR — сканы не читаются**),
  `docx` (python-docx: параграфы + таблицы), `txt/csv/md/text/log`, `html`
  (`extractors.py:1-15`, диспетчер строки 155-170);
- НЕ умеет: `.doc`, `.pptx`, `.ppt`, `.rtf` — `TODO_SUFFIXES`, `extractors.py:27`.
Из пайплайна RAG#1 он **не вызывается** (единственный call-site —
`rag2/confluence.py:53` → `_attachment_documents`, строки 385-420).

**Пайплайна загрузки файлов пользователем нет.** Ни HTTP-upload, ни watch-папки.
Ингест — ручной CLI на хосте. Как HTML превращали в md — пример конвертера:
`/Users/bubblemac/tacticum/iva-rag1-docs/convert.py` (BeautifulSoup + markdownify +
DIRMAP продукт→каталог). Под чужой корпус нужен свой такой конвертер.

**Насколько дорого добавить новый корпус.**
- Ингест — **чисто конфиг**, кода не трогать: `Settings` читает env с префиксом `HELM_`
  (`config.py:15`), а `ingest_corpus` берёт tenant/коллекцию из настроек
  (`docs_index.py:204-219`). То есть
  `HELM_IVA_DOCS_TENANT=gost HELM_IVA_DOCS_QDRANT_COLLECTION=gost_docs__bge_m3_1024
   HELM_MEILI_INDEX=gost_docs uv run python -m helm.ingest.docs_index --docs DIR`
  кладёт чужой корпус в отдельную коллекцию/тенант **без единой строки кода**.
- Приведение корпуса к формату (файлы → md + `manifest.json`) — **пишется коннектор**,
  объём зависит от источника; для HTML прецедент есть (`convert.py`), для pdf/docx
  экстракторы есть, но лежат в rag2 и их надо переиспользовать.

## 3. Изоляция корпусов

**Механизм есть и он fail-closed.** `DocsVectorStore.search` требует непустой
`tenant_id`, иначе возвращает пусто; фильтр `tenant_id` — обязательный `must`, фильтры
фронта (product/section/version) идут ДОПОЛНИТЕЛЬНО, не вместо
(`docs_assistant/vectorstore.py:71-105`, ADR D7 — комментарий строки 7-8). Keyword-индекс
по `tenant_id` создаётся при `ensure_collection` (строка 19, 48-54).

Плюс возможна изоляция коллекцией: `iva_docs_qdrant_collection` — отдельная настройка
(`config.py:107`). В проде так и сделано: `iva_docs__bge_m3_1024`,
`iva_jira__bge_m3_1024`, `iva_confluence__bge_m3_1024`, `helm_mgmt__bge_m3_1024` — свои
tenant у каждого (`config.py:109, 227, 283, 312`).

**Но на ЧТЕНИЕ tenant прибит к процессу.** `build_docs_assistant_context` берёт tenant и
коллекцию один раз из `Settings` (`service.py:67, 83`); ВСЕ call-sites настройки — только
эти два места плюс ингест (grep по `iva_docs_tenant|iva_docs_qdrant_collection`: 5
попаданий, все перечислены). В запросе тенанта нет: `DocsAskIn` — только `question`,
`filters`, `conversation_id`, а `DocsFilters` — только product/section/version
(`routers/docs.py:52-82`). Комментарий в конфиге честен: **«Один корпус = один tenant»**
(`config.py:108`).

Отсюда два пути обслуживать два корпуса:
- **(A) второй инстанс helm** со своим `.env` (свой домен/путь) — 0 строк кода;
- **(B) параметр корпуса в запросе** — правка `routers/docs.py` + `service.py` (кэш
  контекстов по корпусу) + фронт. По весу — средняя задача, ~2-3 файла.

Альтернатива с настоящей мультиарендностью — платформенный
`/Users/bubblemac/tacticum/platform/services/data/knowledge_rag/`: tenant берётся из
токена project-hub / заголовка `X-Tenant-Id`, никогда из аргументов тула
(`interface/tools.py:3-5, 47`, `interface/gateway.py:23-26`). Коллекция
`knowledge__bge_m3_1024` в Qdrant уже существует. Минус известен и записан в концепте:
**чанкинг там заглушка** (1 запись = 1 item), живого smoke не было
(`helm/docs/iva-knowledge-rag-concept.md:242-244`).

## 4. MCP `docs_ask` и `helm-analyst`

- **Код:** `src/helm/interface/mcp/analyst_server.py`, сервер `analyst`, монтируется в
  `src/helm/main.py:186` (`app.mount("/mcp", _analyst_mcp_app)`), рядом `/mcp/process`
  и `/mcp/hrd` (строки 184-185).
- **Адрес (проверен живьём):** `https://helm.tacticum.ru/mcp/analyst` — POST initialize
  отвечает, `serverInfo: {"name":"analyst","version":"1.28.1"}`. Именно этот URL стоит в
  `/Users/bubblemac/tacticum/.mcp.json`. Голый `/mcp` и `/mcp/sse` → 404 (проверял).
- **`docs_ask`** — `analyst_server.py:429-441`: тонкая обёртка над тем же REST
  `routers/docs.ask` (строка 437-438). То есть **источник у него ровно один и тот же —
  публичная дока ИВА, коллекция `iva_docs__bge_m3_1024`, tenant `iva`**. Отдельного
  индекса у MCP нет.
- **Всего тулов 19** (`grep -c "@mcp.tool()"`). Кроме `docs_ask` — `analyst_search`,
  `analyst_context`, `related_tasks`, `api_registry_check`, `contract_check`,
  `requirement_tests`, `arch_map`, `arch_container`, `arch_drift`, `affected_systems`,
  `requirement_coverage`, `nearest_spec` и др. Все, кроме `docs_ask`, отдают
  структурированные данные без генерации прозы (докстрока сервера, строки 78-92).
- **Федерация корпусов уже есть:** RAG#2 может подмешивать корпус доков —
  `rag2_docs_corpus_enabled` (`config.py:286-298`), в проде `HELM_RAG2_DOCS_CORPUS_ENABLED=true`.
  Контур односторонний: публичное внутрь можно, внутреннее наружу — нет (ADR-0003).

## 5. Состояние на проде

Контейнеры на `helm` (`docker ps`): `helm-helm-1` (Up 8h), `helm-traefik-1` (Up 2d),
`helm-mcp-atlassian-1` (Up 6d), `helm-postgres-1` (Up 7d, healthy). Код — `/opt/helm`.

Qdrant и Meili **не на helm**, а на платформе `10.16.0.19` (`HELM_QDRANT_URL=http://10.16.0.19:6333`,
`HELM_MEILI_URL=http://10.16.0.19:7700`). Коллекции: `iva_docs__bge_m3_1024`,
`iva_jira__bge_m3_1024`, `iva_confluence__bge_m3_1024`, `helm_requirements__bge_m3_1024`,
`helm_mgmt__bge_m3_1024`, `knowledge__bge_m3_1024`.

Реранк включён в проде: `HELM_DOCS_RERANK_ENABLED=1`, `HELM_RAG2_RERANK_ENABLED=1`.
Генерация — внутренний vLLM `triva_llm_instruct` через туннель `172.18.0.1:8790`, фолбэк
`tacticum/cheap` на `llm.cifragen.ru`.

Живой ответ `/api/docs/ask` работает и даёт настоящие ссылки (полный вывод в
«Подтверждение»).

---

ВЕРДИКТ: **да, положить можно, и дешевле, чем кажется — но «дёшево» только при одном
условии: чужой корпус приведён к формату `manifest.json` + md с frontmatter.** Ядро
(чанкинг, эмбеддинги, hybrid-ретрив, реранк, генерация, цитаты со ссылками, изоляция по
tenant) — переиспользуется как есть, кода не трогая: ингест параметризуется env
(`HELM_IVA_DOCS_TENANT`/`HELM_IVA_DOCS_QDRANT_COLLECTION`/`HELM_MEILI_INDEX`), изоляция
fail-closed уже реализована и проверена. Платить придётся за две вещи. **(1) Коннектор
«их база знаний → md+manifest»** — его нет и он неизбежен: HTTP-загрузки файлов в системе
нет вовсе, а экстракторы pdf/docx/xlsx существуют только внутри RAG#2 под вложения
Confluence, без OCR и без .doc/.ppt/.rtf. Для «задач и описаний в их системах» нужен ещё
и адаптер трекера — прецеденты есть (`ingest/jira_adapter.py`, `rag2/confluence.py`), но
это отдельная работа под каждую систему. **(2) Отдача двух корпусов одной мордой:** ответ
сейчас жёстко однокорпусный (tenant читается из настроек процесса, в запросе его нет), и
надо выбрать — второй инстанс с другим `.env` (0 строк кода, но своя выкатка) или
параметр корпуса в API (правка `routers/docs.py` + `service.py` + фронт, средняя задача).
Смешивания корпусов при этом можно не бояться: `tenant_id` — обязательный фильтр, пустой
tenant возвращает пусто. Отдельно к вопросу «а сам ГОСТ-документ собрать» — этот механизм
отвечает по корпусу с цитатами, генерации документа по шаблону ГОСТ в нём нет; это второй
кирпич, поверх.

Проверено: `helm/src/helm/main.py`, `interface/api/routers/docs.py`,
`interface/mcp/analyst_server.py`, `infrastructure/docs_assistant/*`,
`infrastructure/rag2/{extractors,confluence,ingest}.py`, `ingest/docs_index.py`,
`config.py`; `helm/docs/iva-knowledge-rag-concept.md`; корпус `iva-rag1-docs/`
(manifest + convert.py + _stats.json); `platform/services/data/knowledge_rag/`;
заметки vault `20-Architecture/RAG-2 — внешние корпуса…`; прод-сервер `helm` (docker ps,
.env без секретов, Qdrant/Meili REST, живые HTTP-пробы).

Данные:
- корпус RAG#1: **2145 страниц**, 1 588 847 слов, 24 726 картинок, 10 505 строк таблиц;
  9 продуктов (MCU 1286 стр., IVA One 244, CS 201, Terra 191, SBC 131, Room 54, Mail 23,
  Updates 14, Portal 1); версий в манифесте 20 (latest 797);
- индекс Qdrant: `iva_docs__bge_m3_1024` — **8272 точки**, tenant единственный: `iva`
  (facet: `iva` = 8272); по продуктам: MCU 4101, IVA One 1659, CS 856, SBC 703, Terra 436,
  Mail 327, Room 190;
- соседние корпуса: `iva_jira__bge_m3_1024` — **329 792** точки,
  `iva_confluence__bge_m3_1024` — **104 721**;
- Meili-индексы (7): `iva_docs`, `iva_jira`, `iva_confluence`, `helm_company`,
  `helm_identity`, `knowledge__bge_m3_1024`, `reqs_eval`;
- форматы на входе: нативно `.md` (+frontmatter); через rag2-экстракторы —
  pdf (без OCR), docx, xlsx/xlsm, txt/csv/md/log, html; НЕ поддержаны doc/pptx/ppt/rtf;
- эмбеддер bge-m3, dim **1024**, cosine; батч эмбеддинга 4, upsert 256;
- MCP `analyst`: **19 тулов**, версия сервера 1.28.1.

Подтверждение (команды и вывод):
- `docker ps` на helm → `helm-helm-1 Up 8 hours`, `helm-traefik-1 Up 2 days`,
  `helm-mcp-atlassian-1 Up 6 days`, `helm-postgres-1 Up 7 days (healthy)`.
- `curl -o /dev/null -w %{http_code} https://helm.tacticum.ru{/iva-docs,/docs,/analyst,/api/docs/products}`
  → `200, 404, 200, 200`.
- `POST /api/docs/ask {"question":"как настроить запись конференции в IVA MCU?"}` →
  `not_found=False`, `reason="есть доказательства"`, `citations=2`:
  `[1] Настройки записи мероприятия | https://iva.ru/docs/mcu-server/latest/ag/system_settings/setting_event_recording.html | MCU latest`,
  `[2] Документация IVA Room | https://iva.ru/docs/room/latest/index.html | Room latest`.
- `curl http://10.16.0.19:6333/collections` → 6 коллекций (список выше);
  `points/count {"exact":true}` → docs 8272 / jira 329792 / confluence 104721.
- `POST /collections/iva_docs__bge_m3_1024/facet {"key":"tenant_id"}` → `[{"value":"iva","count":8272}]`.
- `POST https://helm.tacticum.ru/mcp/analyst` initialize → `{"name":"analyst","version":"1.28.1"}`.
- `grep -c "@mcp.tool()" analyst_server.py` → `19`.
- `grep -rn "iva_docs_tenant|iva_docs_qdrant_collection|iva_docs_dir" src/ scripts/ tests/` →
  5 строк: `config.py:107,109,217`, `ingest/docs_index.py:204,205,219,249`,
  `docs_assistant/service.py:67,83` — других call-sites нет.

НЕ проверено:
- **какая именно «их база знаний»** у смежного отдела — форматы файлов, где лежат
  (сетевой диск? облако?), какая система задач (Jira/EVA/своя). Без этого нельзя оценить
  коннектор точнее, чем «неизбежен». Это вопрос, который стоит задать до планирования.
- контур данных: материалы смежного отдела публичные или чувствительные. От этого
  зависит, можно ли их держать у нас или нужен on-prem (ADR-0003).
- не запускал ингест на тестовом корпусе — гипотеза «env-only, без кода» проверена
  чтением кода (`docs_index.py:204-219` + `config.py:15`), не прогоном.
- Serena использована частично: `activate_project` для helm сработал, но после первого
  bash-вызова рабочий каталог агента сбрасывается на `doc_translator` и
  `find_referencing_symbols` упал с `FileNotFoundError` на `helm/src/helm/config.py`.
  Call-sites настроек корпуса добраны grep'ом — для полей pydantic-модели это полный
  список (обращение всегда `settings.<имя>`), но говорю прямо: это текстовый поиск, а не
  символьный.
- eval/качество на чужом корпусе — не мерил; golden-set есть
  (`helm/data/competency-questions.md`, `iva-rag1-docs/golden`), прогона не делал.
- бот «Поддержка» (`routers/bot_support.py`) — тот же RAG#1 в чат ИВА, живость не проверял.