---
title: '2026-07-04 — Helm: реорг data + план 1a/1b (backend-only, Agent Team)'
type: report
permalink: tacticum/01-sessions/2026-07-04-helm-reorg-data-plan-1a-1b-backend-only-agent-team-1
tags:
- helm
- control-tower
- data
- wave-1a
- wave-1b
- agent-team
- session
- plan
---

Тимлид-сессия 2026-07-04: приехал весь Трек B (реальные выгрузки ИВА), разведка Agent Team (2 explorer, Opus), реорг `data/`, зафиксирован scope backend-only и фазовый план 1a→1b. Продолжение [[Helm — план после деплоя 1a + handoff]], реализует [[control-tower-v02]].

## Ключевое решение — SCOPE: только backend, без фронта

- [decision] Пользователь: **делаем только backend, фронт не строим.** Следствие: вывод разведки «интерактивный веб-апп (DoD 1a) = 0%» нерелевантен. «Полное 1a по канону» переопределено как **backend + пайплайн + экспорт-артефакты** (Монитор ГД: статичный HTML + markdown-бриф + JSON API), НЕ интерактивный дэш. #scope
- [decision] Деливерабл по канону без UI = батч-пайплайн на реальных данных → граф → Монитор ГД HTML/бриф (экспорт) + JSON API. heatmap/scatter/матрица (1b) — как данные/статический экспорт, не интерактив. #scope

## Реорг data/ — ВЫПОЛНЕНО

- [outcome] `data/` разложен по источникам (скрипт только mv/mkdir, ничего не удалено, всё реверсивно). Новая раскладка: `example/`(🟢 синтетика @iva.example, 64K) · `real/`(🔒 реальные ИВА, 564M: jira · monitor-gd · git · crm · service · identity · confluence+bodies · data-room · sensitive🔴) · `_superseded/`(байт-в-байт дубли: jira-loose, repos-shallow, repos-csv-only, 4.2M). Полный инвентарь — [[explore-data-inventory]]. #data
- [reference] Канон git = `real/git` (бывш. repos-1, 24040 коммитов + repomix 9 + scip 8 + topology 8 + registry). Дубли-подтверждены md5: loose jira==real/jira; repos(436)⊂git; data(2)/repos == git CSV но без обогащения. #data

## Пробелы данных (касаются оператора)

- [followup] Реальной кураторской нарезки блоков/ЕОЛ/целей для 1a НЕ приехало (канон Q0). Кандидаты: `real/data-room/product_mgmt/product_owners.csv`, `track_owners.csv`, `real/monitor-gd/ceo_directives.csv`(15 поручений owner_email+block). Дефолт: собрать черновую нарезку из данных (канон разрешает), оператор валидирует. #wave-1a
- [followup] 🔴 Зарплата по ФИО для 1b НЕ приехала в чистом виде — в `real/sensitive/by_person.csv` только `ставка_fte`/трек/статус, рублёвого дохода нет; ФОТ агрегированно+pivot. Дефолт: скоринг стоимости 1b на FTE-прокси + объяснимость, реальную зп доснять. #wave-1b #sensitive
- [followup] Прочее к доснятию: тела Confluence (3/3154), GitLab PAT (полный MR API), полная история Jira (≤300/проект), SCIP Kotlin+Java (kmp+ivcs непокрыты), хвост идентичности (25 team+122 git-кластера). #data

## Состояние кода vs канон (карта @explorer-code)

- [reference] 1a-backend ГОТОВ 100% (204 теста): модели Signal/Initiative/Block/Sales §3, генезис (`domain/genesis.py`), расчёты §4/§5.2 (`domain/calc.py`), 5/6 разрывов §6.1, снапшот+diff (`application/snapshot.py`), бриф+markdown (`application/brief.py`), портфель (`application/portfolio.py`), JSON API (`interface/api/routers/*`), FK-фиксы (`repository.py`). #wave-1a
- [reference] 1a backend-долги до «полного по канону»: A1 Signal.text; серверные фильтры portfolio/gantt; PATCH override plan_finish/lead_time + аудит (нет CRUD инициативы); PUT goals/blocks/teams + annotations-эндпоинт; продуктовый экспорт HTML/бриф (T0 был scratchpad-раннер). OIDC — P1, external-blocker (service-key), для backend-деливерабла не критичен без фронта. #wave-1a
- [reference] 1b backend: git-ingestion ✅скелет (`ingest/git_source.py`), идентичность ✅скелет (`ingest/identity.py`), M8 Gateway ✅ но только lead_time (`llm/gateway.py`); ОТСУТСТВУЮТ: M3 RE-обёртка (`ingest/re_enrich.py` — topology/dedup/scip→work→product/поколение, +6-й разрыв дубли), M5 скоринг (`application/scoring.py` — вклад/ценность/стоимость/квадранты), M8 CV-парс+dedup-judge, M9 governance (2 контура на уровне данных). #wave-1b

## Фазовый план (backend-only)

- [plan] Ф0 реорг data ✅. Ф1: 1a-backend доводка (A1·фильтры·PATCH-override·CRUD·экспорт), параллелится по непересекающимся файлам. Ф2: деплой 1a + тест на РЕАЛЬНЫХ (черновая нарезка блоков/ЕОЛ из данных + real/jira+monitor-gd+crm+service), verifier на живом Postgres, acceptance = Монитор ГД HTML+бриф на реальных ИВА. Ф3: 1b помодульно (git-ingest full→identity→M3 RE→M5 скоринг→M8 CV/dedup→M9 governance). Ф4: деплой 1b + экспорт-артефактов ценности + acceptance «кто на деньгах/на фигне» с объяснимостью. #plan
- [process] Agent Team: тимлид + @explorer×2 (Opus, разведка ✅) → далее @implementer в git-worktree + независимый @verifier на живом Postgres. Каноническую память пишет только лид; воркеры — черновики в 00-Board + SendMessage. #agent-team

## Отношения
- relates_to [[Helm — план после деплоя 1a + handoff]]
- relates_to [[agent-team-design]]
- implements [[control-tower-v02]]
- relates_to [[explore-data-inventory]]

## Апдейт 2 — Ф1 закрыта + правило сервера + платформенный субстрат

- [outcome] **Ф1 (1a-backend доводка) ПРИНЯТА и влита в `wave-1a-backend`** (@implementer-1, worktree). 5 WP: A1 Signal.text · серверные фильтры portfolio/gantt · PATCH override plan_finish/lead_time + аудит (каждое поле→Annotation, миграция `plan_finish_override`, грабли FK учтены — `_ensure_person_email`) · CRUD PUT goals/blocks/teams + аудит-трейл аннотаций · продуктовый экспорт HTML (`render_html`) + CLI `scripts/export.py`. 238 passed (+34), ruff+mypy strict чисто. OIDC не трогали (P1). #wave-1a #verified
- [decision] **Правило процесса (оператор): всё тестируем на dev-сервере `helm` (ssh-manager, 159.194.233.33, `/opt/helm`), @verifier гоняет на живом Postgres там же.** Локально — только быстрый прогон перед выкаткой. Сейчас на сервере крутится синтетический сид (`/api/gaps` отдаёт G1/G3/G4…). #process
- [followup] `scripts/seed_db.py` знает только `--source synthetic|csv` — **нет `--source real`**. Для Ф2 нужен seed-путь под реальные CSV (real-адаптеры RealJira/RealGit + реальные data/real/* + черновая нарезка блоков/ЕОЛ). #wave-2

### Платформенный субстрат «граф через платформу» (канон §8.2, ADR принят) — встроен в план как Ф3-подтрек

- [decision] Оператор напомнил: по канону граф идёт через платформу. Сверено с [[ADR — Helm: PG-ядро + подключение к платформенному субстрату (Graphiti/Gateway/knowledge_rag)]] и разведкой #26 [[explore-26-memory-knowledge-contracts-for-helm-connectors]]. **Это НЕ перенос ядра в Graphiti:** детерминированный Signal→Initiative остаётся в PG Helm; в платформу идёт темпоральная/knowledge-**проекция**. Шаг 4 канона реконсилирован. #substrate
- [plan] Субстрат — **аддитивно в Ф3 (1b), приоритет ADR: Gateway (готов, M8) → knowledge_rag → memory/Graphiti.** Коннекторы за Helm-Port + вендоренные shape-типы (platform-SDK пуст), тесты на стаб-транспорте, e2e-смоук за флагом. knowledge_rag MCP `:8090/mcp` (knowledge_search/ingest, коллекция `knowledge__bge-m3_1024`, tenant fail-closed) — код(repomix)/CV/Confluence + semantic_dedup (M3 дубли, D3/D4). memory/Graphiti MCP `:8080/mcp` (memory_write/read L0-L2, bi-temporal) — нарратив/velocity/дрейф (периодические project_state-эпизоды). Ограничение: reference_time на тул не проброшен → бэкфилл истории требует доработки тула/прямого lib-вызова (live-снимки без него). #substrate #wave-1b
- [need] Реальное подключение субстрата = **та же связка, что OIDC**: base_url Gateway + project-hub service-key/тенант + достижимость memory/knowledge_rag. До ключа — Port+стаб+локальный compose-смоук (X-Tenant-Id обходит IdP). Внесено в список запросов к IVA. #external-blocker

- relates_to [[ADR — Helm: PG-ядро + подключение к платформенному субстрату (Graphiti/Gateway/knowledge_rag)]]

## Апдейт 3 — 1a на реальных + project-hub ключ получен

- [outcome] **1a persisted на РЕАЛЬНЫХ данных ИВА, живой Postgres, 0 FK-ошибок.** Ф2.1 (derived-нарезка: 10 блоков ГД из ceo_directives, 15 директив-целей, генератор `scripts/derive_real_manual.py`) + Ф2.2 (`ingest/real_source.py`, `--source real`) приняты и влиты. Сид: 115 инициатив (15 директив + ~100 реальных Jira-эпиков), 9 блоков (ЦБ отвалился — `validate_blocks` требует ЕОЛ), 3016 сигналов, 61 назначение. #wave-1a #verified
- [followup] **Ф2.3 (в работе)** — честный Монитор ГД: due+override директив (осели в `directives_extra.csv`) пробросить в дедлайн + status_override инициативы (поля Ф1.3); `export.py` научить `--source real` (сейчас хардкодит синтетику). Ориентир светофора D03/D06=🔴, D14=🟡. #wave-1a
- [followup] **CRM→sales отложен (операторское решение):** в реальном `crm_deals` продукт пуст ~2161/2179, стадии — свободный текст без энума; `top_projects_pipeline` даёт продукт+бюджет без даты/стадии. Нужен выбор источника + словарь стадий/продуктов. На Монитор ГД 1a не влияет (дедлайны из директив), важность §4 ждёт этого. #wave-1b
- [resolved] **project-hub service-key (phk) ПОЛУЧЕН** — оператор положил в `.env`. На сервере `/opt/helm/.env`: `HELM_PROJECTHUB_TOKEN` + `HELM_TENANT` (+ `HELM_GATEWAY_BASE_URL`/`API_KEY`). Внешний блокер субстрата (#need выше) и OIDC СНЯТ — реальный e2e Graphiti/knowledge_rag возможен. Остаётся: `config.py` дописать чтение TOKEN/TENANT + добавить base-URL memory(:8080)/knowledge(:8090) при постройке коннекторов (Ф3). `.env` gitignored — в git не попадёт. #external-blocker #substrate
- [decision] Правило безопасности: `.env` не читать/не печатать/не коммитить (локальный read заблокирован deny by design); реализаторам запрещено. #security

## Апдейт 4 — Ф2 ЗАКРЫТА: 1a на сервере на реальных данных

- [outcome] **Ф2 ПРИНЯТА — 1a развёрнут на сервере `helm` и работает на реальных данных ИВА.** Доставка через SFTP (tar+ssh_upload, rsync не проходит — см. [[Helm — runbook работы с dev-сервером (SFTP-доставка, не rsync)]]). Код Ф1+Ф2 выкачен в `/opt/helm` (`.env` не тронут), реальные данные (manual+jira+monitor-gd+crm+identity, без 552М git) → контейнер. Обе миграции применились, real-seed: 115 инициатив/15 целей/9 блоков/3016 сигналов. `/api/portfolio` = дерево блок→ЕОЛ→инициативы со светофором; `/api/gaps` = реальные разрывы; экспорт Монитора ГД HTML+бриф: **One 1.5 🔴2 🟡12**. #wave-1a #verified #server
- [outcome] Ф2.3 (честный Монитор ГД: due+override→светофор, export --source real) принята и влита. Итого Ф1(5 WP)+Ф2(3 WP) = 8 work packages, 242 теста зелёные, ruff+mypy strict чисто. **Волна 1a по канону (backend) ЗАВЕРШЕНА и работает на реальных ИВА на сервере.** #wave-1a
- [followup] Мелочь на потом: 15 директив показываются как «цель без работы» (нет Т2-связки директива→Jira-эпик; директивы — управленческие поручения без эпиков под ними). Кандидат на уточнение — линковать директивы к Jira по продукту/теме. Не блокирует. #wave-1a
- [gotcha] macOS AppleDouble `._*` в tar ломают alembic (null bytes) — чистить `find -name '._*' -delete`; тарить `COPYFILE_DISABLE=1`. Записано в runbook. #gotcha

## Апдейт 5 — Эндпоинты субстрата найдены (реальный e2e разблокирован)

- [resolved] **Реальные адреса платформенного субстрата (сам вычислил из `platform/services/runtime/mcp_runtime/{registry.json, deploy/traefik-mcp-gateway.example.yml}` + проба с сервера):**
  - **memory (Graphiti):** `https://mcp.tacticum.ru/memory/mcp`
  - **knowledge_rag:** `https://mcp.tacticum.ru/knowledge/mcp`
  - Хост MCP-шлюза — **`mcp.tacticum.ru`** (НЕ cifragen; Gateway отдельно на `llm.cifragen.ru`). Traefik: `Host(mcp.tacticum.ru) && PathPrefix(/memory|/knowledge)` → forwardAuth(mcp-runtime→project-hub) → stripPrefix → backend `memory:8080`/`knowledge:8090`. Auth = `Bearer $HELM_PROJECTHUB_TOKEN`, tenant инжектится сервером (X-Tenant-Id). #substrate
- [outcome] **Достижимость с сервера `helm` подтверждена:** оба роута → HTTP 401 (не 000/404 — TLS+роутинг живы, нужен bearer). Сервер `helm` в приватной сети платформы `10.16.0.x` (достал `arch` 10.16.0.17:8780 из registry). Платформа доступна с нашего контура. #substrate #server
- [plan] Коннекторы Ф3: `config.py` += `HELM_MEMORY_MCP_URL=https://mcp.tacticum.ru/memory/mcp`, `HELM_KNOWLEDGE_MCP_URL=https://mcp.tacticum.ru/knowledge/mcp`, читать `HELM_PROJECTHUB_TOKEN`/`HELM_TENANT`. Транспорт — httpx (в контейнере есть; `curl` в контейнере НЕТ). MCP = JSON-RPC POST + Accept `application/json, text/event-stream`. #substrate

## Апдейт 6 — Ф3.A (git+identity) закрыта; Т1 ~0 = данные, M3 работает

- [outcome] **Ф3.A принята и влита.** git-ingestion на реальных 24039 коммитов: 24383 git-сигнала, идентичность 955 (57 сшито cross-system, 68 external, 18 alias-кандидатов, 5 bot, 3 machine-local), M3 repo→product покрыл все 24039 коммитов/8 репо. 248 тестов, ruff+mypy чисто. #wave-1b #verified
- [outcome] **Т1 commit→task привязано 0/24383 — это ДАННЫЕ, не баг.** Overlap-анализ: коммиты ссылаются на 5461 issue-ключ (VCSWEB 20129, IVAONE 3154, IVAMSG2 1706…), засеяно 300 свежих/проект (3071); пересечение commit-ключей ∩ seeded-issues = 148, ∩ epics = 0. git-история полная, Jira-срез свежий → ключи почти не перекрываются (gap #3 разведки). #wave-1b
- [decision] **Мост git→ценность для §6 = M3 repo→product (покрыл 24039 коммитов), НЕ точечный Т1** (структурно ~0 на этом снимке). M5-скоринг вклада строим на person→commit→product (M3) + person→jira (assignee), exact Т1 — опционально/спарс. Для плотного Т1 нужна полная история Jira (пагинация, dozapros у ИВА/DevOps). #wave-1b #decision
- [followup] Dozapros для плотного Т1 (позже): полная история Jira по проектам (VCSMOB 14345, VCSWEB2 9493…) через пагинацию — тогда commit issue-keys найдут задачи. Не блокирует value-аналитику (она на product-level). #wave-1b

## Апдейт 7 — Субстрат: реальный e2e ЖИВОЙ + коннекторы влиты

- [outcome] **Реальный e2e субстрата ПОДТВЕРЖДЁН с сервера `helm`:** прямой httpx-пробой из контейнера с `HELM_PROJECTHUB_TOKEN` → **memory (Graphiti) `initialize` → 200**, **knowledge_rag `initialize` → 200**, полный MCP-хендшейк (protocolVersion/capabilities). Платформа реально работает с нашим токеном. Ответ — **SSE** (`event: message\ndata: {json}`), не plain JSON. #substrate #verified
- [outcome] **Ф3.D коннекторы субстрата приняты и влиты** (@implementer-2): `src/helm/substrate/{mcp_client,knowledge,memory}.py` + config.py (HELM_MEMORY_MCP_URL/HELM_KNOWLEDGE_MCP_URL/токен). `mcp_client` корректно парсит SSE И JSON (`_parse_jsonrpc`/`_last_sse_json`), MCP-хендшейк, McpError, транспорт инъектируемый (MockTransport для стабов). KnowledgeConnector (search/ingest) + MemoryConnector (write project_state/read L0-L2) за Helm-Port. Тесты на стабах + e2e-смоук за флагом. 292 теста. #substrate #wave-1b
- [reference] Транспорт платформы: FastMCP streamable-http, POST `…/mcp`, `Accept: application/json, text/event-stream`, Bearer=token, ответ SSE. curl в контейнере helm НЕТ — только httpx. #substrate

## Апдейт 8 — M5 скоринг влит (value-аналитика на реальных)

- [outcome] **Ф3.C M5 скоринг принят и влит** (@implementer-1). На реальных: 196 человек, квадранты {star 101, valuable 1, overvalued 1, underloaded 93}, team_heatmap 6 треков (Трек Б BE mean_value 132 топ, Сквозное 24.9), scatter, CLI `scripts/score_value.py → value_report.json` (§6.2). **Объяснимость есть** (persons[].breakdown: jira/git-скор, product_scores, value_note, `cost_is_proxy=true` — FTE помечен). Новые `ingest/{economics,sensitive}.py`, `application/scoring.py`. 302 теста. #wave-1b #verified
- [followup] Калибровка квадрантов (перекос 101 star / 93 underloaded — медианный сплит нужен) + git-объём доминируют external/unassigned (159/196; идентичность сшила к трекам 57) → тюнинг идентичности (122 несшитых git-кластера) обострит трековый срез. Не баг — данные/тюнинг. #wave-1b
- [plan] Осталось до полной 1b: M8 (CV-парс + dedup-judge через Gateway) + M9 (governance 2 контура 🟢/🔴 на уровне данных). Запущены финальным батчем. #wave-1b

## Апдейт 9 — ФИНАЛ: 1a+1b собраны, влиты, развёрнуты на сервере

- [outcome] **M8 (CV-парс + dedup-judge) и M9 (governance 2 контура) приняты и влиты.** Волна 1a+1b (backend) по канону — СОБРАНА. `wave-1a-backend`: 331 тест, ruff+mypy strict чисто. Все модули: 1a (genesis/calc/gaps/snapshot/brief/portfolio/API/override/CRUD/filters/export) + 1b (re_enrich M3, scoring M5, dedup_judge+cv_profile M8, governance M9, substrate memory/knowledge, git-ingestion+identity). #wave-1b #verified
- [outcome] **Консолидированный деплой 1b на сервер `helm` (SFTP, COPYFILE_DISABLE=1 → без AppleDouble) — ПРОЙДЕН.** Код + git-lite (commits/merges/manifest/registry/topology, БЕЗ 552М repomix/scip) + data-room/economics + sensitive → контейнер. seed real+git: 27399 сигналов (24383 git), M3 24039 коммитов. Value-отчёт на сервере: 196 человек, квадранты, оба контура (🟢 green 0 persons/только агрегаты, 🔴 red 196 persons+scatter). 1a API живой (/api/portfolio 9 блоков). Субстрат e2e с сервера 200. #server #verified
- [outcome] **ИТОГ сессии 2026-07-04 (Agent Team, тимлид + 2 implementer, Opus):** Ф0 реорг data · Ф1 1a-backend доводка (5 WP) · Ф2 деплой+тест 1a на реальных · Ф3 1b (git+identity, M3, M5, M8, M9, субстрат real-e2e) · консолидированный деплой на сервер. Всё на реальных данных ИВА, всё по канону [[control-tower-v02]]. #done
- [followup] Тюнинг/dozapros (не блокеры): калибровка квадрантов M5 (медианный сплит); тюнинг идентичности (122 несшитых git-кластера → чище трековый срез); полная история Jira (плотный Т1); CRM→sales маппинг (важность §4); реальная зп по ФИО (сейчас FTE-прокси); тела Confluence; репомикс/scip на сервер для dedup-эмбеддингов; Taiga-синхронизация статусов M1-M10. #followup