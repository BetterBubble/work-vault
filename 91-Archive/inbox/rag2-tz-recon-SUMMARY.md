---
title: rag2-tz-recon-SUMMARY
type: report
permalink: tacticum/91-archive/inbox/rag2-tz-recon-summary
tags:
- rag2
- analyst-mcp
- recon
- tz
- helm
status: archived
updated: 2026-07-18
---

# RAG#2 ТЗ (Р-1..Р-5) — сверка с кодом (read-only recon)

Репозиторий `/Users/bubblemac/tacticum/helm`. Разведка на ветке `feat/analyst-mcp-server` (HEAD `84dd30e`) + сравнение с `main`. ТЗ датировано 2026-07-16 — часть фактов устарела.

## ⚠️ ГЛАВНОЕ (влияет на весь план): рекон-таргет — не та ветка
- HEAD `84dd30e` (`feat/analyst-mcp-server`) и `main` разошлись на общем предке `992b064`. **`main` = HEAD+64 коммита; HEAD = main-обгон всего на 1 коммит.**
- `analyst_server.py`: на HEAD — **396 строк / 8 тулов**; на `main` — **1190 строк / 14 тулов**. Живой MCP `helm-analyst` (14 тулов) отвечает **кодом `main`, а не этой ветки**.
- Почти всё, что описывает ТЗ, уже сделано и лежит в `main` (или в ветках, уже влитых в `main`). Планировать/реализовывать на HEAD `84dd30e` = переделывать готовое и терять контекст.
- **Вывод для тимлида:** рекон и план надо переякорить на `main` (или на свежую ветку от `main`), а не на `84dd30e`. Ниже факты даю по `main` как по реальной базе, с пометками где HEAD отстаёт.

---

## A. Реранк (Р-2)
- **Дефолт `rag2_rerank_enabled = False`** — верно и на HEAD, и на main: `config.py:190`. Флаг читается pydantic-settings, `env_prefix="HELM_"` (`config.py:15`) → env-имена `HELM_RAG2_RERANK_ENABLED`, `HELM_DOCS_RERANK_ENABLED`, `HELM_ASSISTANT_RERANK_ENABLED`. Дефолт в коде — хардкод `False`; фактическое включение — только через env на деплое (в репо `.env` ключей RERANK нет).
- **ТЗ «rerank off» — формально верно по дефолту кода, но по сути устарело:** ветка `feat/enable-rerank` (PR #49, `f18f0da` «включить reranker tacticum/rerank через Gateway») **влита в main** (ahead-of-main=0). Т.е. реранк-конвейер полностью разведён и «включён» операционно. Подозрение тимлида про env-включение на проде — согласуется с кодом.
- **Модель:** `bge-reranker-v2-m3` через Gateway-тир `tacticum/rerank` (`config.py:148-151`, `reranker.py:1-5`). Общая для доков/требований/RAG#2.
- **Per-corpus реранк** — реализован: один `JiraReranker` инстанс инъектится в КАЖДЫЙ `JiraIndexSearch` (Jira/Confluence/helm) и применяется ВНУТРИ корпуса до слияния (`search.py:162-163`; сборка — `service.py:166-217`). Реранк-скор пишется сырым в `JiraDoc.score` (`reranker.py:39` `replace(..., score=score)`). **Нормированного confidence НЕТ** — тулы отдают сырой скор (rerank ~0.0x или RRF fused). `confidence` в коде есть только в `review.py` (чистота кластера needs) — не связано.
- **Cross-corpus реранк — тоже уже реализован в main:** новый флаг `rag2_cross_rerank_enabled = False` (`config.py:193`, только на main), `service.py` разводит `corpus_reranker` (per-corpus) и `cross_reranker` (слитого списка). Ветка `feat/rag2-cross-rerank` (PR #53) влита в main. `fix/rag2-cross-rerank-helm-tune` — **на 7 коммитов впереди main, НЕ влита** = тот самый провалившийся тюнинг кросс-реранка (подтверждает заметку тимлида).
- **Порога отсечения «лексического шума» по скору — НЕТ.** В `search.py` нет min_score/floor; хиты ниже любого порога отдаются как есть. Кросс-корпусное слияние `federate` (`domain/rag2.py:324`) сливает РАНГИ через RRF (не сырые скоры — шкалы cosine/BM25 несопоставимы), идентичность = (source, key).
- Статус веток относительно main (ahead/behind): enable-rerank 0/27 (влита), rerank-client-sort 0/25 (влита), rag2-cross-rerank 0/21 (влита), cross-rerank-helm-tune **7/0 (не влита, failed tune)**.

## B. Полнотекст / Meilisearch (Р-3)
- `fulltext.py`: индекс задаётся конфигом, дефолт Jira = `iva_jira` (`config.py:171`, `fulltext.py:33`). primaryKey=id (= uid Qdrant для RRF).
- **Confluence НЕ «только векторно» (ТЗ устарело):** отдельный индекс `iva_confluence` уже есть (`rag2_confluence_meili_index`, `service.py:105`). В `service.py:188-204` Confluence-корпус собирается с `conf_fulltext` и `conf_mode="hybrid"`, если задан `meili_url`. Т.е. второй индекс для Confluence уже добавлен, отдельным индексом (не через `source_type` в едином).
- **RRF-слияние двухуровневое:** (1) внутри корпуса dense(Qdrant)+fulltext(Meili) по uid — `search.py:_hybrid` + `docs_assistant.fusion.rrf_fuse`, RRF_K=60, CANDIDATE_K=60; (2) межкорпусное `federate` по рангам. Фильтруемые поля Meili: tenant_id/source_doc_id/key/project/epic/component/status/assignee/type (`fulltext.py:17-24`), tenant fail-closed.
- **Токенизация camelCase/snake_case** — в `fulltext.py` НЕ найдена (полагаемся на дефолт Meili). Уточнить, есть ли на уровне ingest/analyzer-настроек индекса.

## C. Ингест / корпусы / экстракторы (Р-1, Р-4)
- **3 корпуса RAG#2**, каждый = свой Qdrant-collection + tenant/scope (fail-closed, ADR D7): Jira `iva_jira__bge_m3_1024`/tenant `iva`; Confluence `iva_confluence__bge_m3_1024`; helm-mgmt `helm_mgmt__bge_m3_1024`/tenant `helm` (`config.py:166-208`). Тег `source` в payload → морда помечает происхождение (`search.py:70`).
- **`source_type` = api/contract — НЕ существует** нигде в src (найденные «contract» — это стадии CRM-сделок, не RAG). Т.е. корпус контрактов/реестра API для Р-1 — с нуля.
- **Экстракторы — ТЗ «нет HTML» устарело для main:** `extractors.py` на **main** умеет pdf (pypdf, без OCR), docx (python-docx), xlsx (openpyxl), txt, **html** (`confluence.html_to_markdown`). На **HEAD** экстрактор беднее: только xlsx/txt/html, pdf/docx = TODO (возвращает None). Ещё один аргумент переякориться на main.
- **Eva-коннектор — «коннектора пока нет» ПОДТВЕРЖДЕНО:** `domain/rag2.py:369` (`# EVA … коннектора пока нет`), роутер: EVA → только `live` (`domain/rag2.py:424,524`). `application/tasks.py:20,202` — «EVA-трекер не загружается — единый SoR пока только по Jira».
- **`distrohost`/`orionpro`/`eva-wiki` — в src НЕ упоминаются** (ни на HEAD, ни в grep). Есть только `eva`/`eva.iva.ru` как live-намерение роутера.

## D. Регистрация тулов (Р-1)
- Паттерн: FastMCP, `mcp: FastMCP = FastMCP(...)` (`analyst_server.py:53`), тул = `@mcp.tool()` над `async def name(ctx, ...) -> dict` (`:217+`). Тул зовёт application/REST через хелперы `_call_rest`/`_with_session`, principal через `_require_principal` (`:165`).
- **14 тулов на main:** analyst_search, analyst_context, related_tasks, docs_ask, arch_map, arch_container, affected_systems, requirement_coverage, nearest_spec, who_to_involve, effort_hint, gap_questions, constraints, contradiction_check.
- **На HEAD только 8** (нет nearest_spec, who_to_involve, effort_hint, gap_questions, constraints, contradiction_check).
- `api_registry_check` — не существует ни на одной ветке → новый тул. Точка добавления: рядом с остальными `@mcp.tool()` в `analyst_server.py` (на main).

## E. effort_hint (Р-5)
- Тул есть на main: `analyst_server.py:856` `async def effort_hint(ctx, requirement_text=None, system_id=None)`.
- **Причина null-медиан — by design (подтверждено докстрингом тула):** `active_days`/`lead_time_days` считаются из changelog с таймстемпами; если у похожей задачи нет валидного changelog → значение `null` (не 0.0) и она НЕ входит в медиану; если ни у одной нет длительностей → медианы `null` + `note`. Читает `t.changelog` через `_normalize_epic_changelog`/`_status_durations`.
- Т.е. проблема — не в query-time, а в **полноте changelog в корпусе EpicTask** (ingest). Доступ к changelog есть (объекты задач несут `.changelog`), но у части задач он пустой/без таймстемпов. Направление Р-5 — обогатить ингест changelog'ом Jira.

## F. Golden eval (Р-2 приёмка)
- Файлы есть и на HEAD, и на main: `eval/rag2_golden.py`, `rag2_runner.py`, `rag2_eval.py`, `rag2_adapters.py`, `golden.py`, `metrics.py`, `judge.py`; данные — `tests/data/rag2_golden.json`.
- **68 кейсов** (одинаково HEAD и main). Формат: `meta` (name/purpose/as_of_data/modes/flags) + `cases[]` с `id`, `query`, `expected_sources[]`, флаги temporal/needs_confluence_body. Валидирует роутер index/live/hybrid + полноту домена.
- **Метрики:** `recall@k`, `hit@k`, `MRR`, `nDCG@k`, `answer_in_context@k` — чистые функции `eval/metrics.py:69-85+`, харнесс `rag2_runner.py` их переиспользует.
- Ветки `feat/rag2-golden-eval` (0 ahead — влита в main) и `feat/routing-eval-golden` (0 ahead — влита). Т.е. golden-харнесс в main актуален.

---

## Блокеры / зависимости
- **[КРИТ] База ветки.** Нужно решение тимлида: рекон/имплемент вести на `main` (14 тулов, rerank, cross-rerank, полные экстракторы), а не на `84dd30e`. Иначе дублирование готового.
- **Р-4 (api/contract корпус):** нужен доступ к внутреннему контуру-источнику API-реестра/контрактов + определение `source_type`. Сейчас нет ни источника, ни коннектора.
- **Р-1 EVA / live:** реальный tool-capable endpoint iva-mcp (`rag2_live_mcp_url`/token пусты → заглушка). EVA-коннектора нет by design.
- **Р-5:** полнота Jira changelog в ингесте EpicTask (иначе медианы остаются null честно).

## Открытые вопросы к тимлиду
1. Переякориваем рекон/план на `main`? (сильно рекомендую — HEAD `84dd30e` стар на 64 коммита).
2. Р-2 «включить rerank» — речь про env-включение на проде (`HELM_RAG2_RERANK_ENABLED=1`, уже возможно) или про изменение дефолта в коде? И кросс-реранк: добивать `fix/rag2-cross-rerank-helm-tune` (7 коммитов, провал тюнинга) или оставить per-corpus?
3. Р-2 «нормированный confidence + порог отсечения шума» — это новая работа (сейчас отдаётся сырой скор, порога нет). Подтвердить, что это в скоупе.
4. Р-3 — Confluence-fulltext уже есть (`iva_confluence`); что именно ещё требуется (camelCase/snake токенизация? единый индекс с source_type вместо раздельных?).
5. Р-4 — откуда берём реестр API/контрактов (источник, формат, доступ)?


---

# ДОПОЛНЕНИЕ (углубление A-реранк и C-Eva, по запросу тимлида)

## A+. Реранк — детальный пайплайн и почему cross-rerank «не закрыт»

### Как включён и где читается
- Дефолты хардкодом в `config.py` (`Settings`, pydantic-settings, `env_prefix="HELM_"`): `rag2_rerank_enabled=False` (:190), `rag2_cross_rerank_enabled=False` (:193, только main), `docs_rerank_enabled=False` (:117), `assistant_rerank_enabled=False` (:154). Env-override: `HELM_RAG2_RERANK_ENABLED`, `HELM_RAG2_CROSS_RERANK_ENABLED`, `HELM_DOCS_RERANK_ENABLED`, `HELM_ASSISTANT_RERANK_ENABLED`.
- Фактическое значение на проде — из env контейнера (в репо `.env` пуст по RERANK). Vault подтверждает: per-corpus rerank ВКЛючали на проде (сессия 2026-07-15 «Rerank подключён к RAG#1 и RAG#2, проверено интроспекцией живого контейнера», эндпоинт `llm.cifragen.ru/v1/rerank` = `tacticum/rerank`). Значит ТЗ «rerank off» — про дефолт кода, а операционно per-corpus включён.

### Порядок применения (per-corpus)
- Пайплайн одного корпуса (`search.py:JiraIndexSearch.search`): retrieve (dense Qdrant + fulltext Meili, `want=max(limit,CANDIDATE_K=60)`) → RRF-слияние по uid → **rerank всего списка** (`:162-163`, `top_n=len(docs)`) → `_cap_per_doc` (MAX_CHUNKS_PER_DOC=2) → `docs[:limit]`. Т.е. rerank ДО cap и обрезки. Реранк-скор пишется сырым в `JiraDoc.score` (`reranker.py:39`), НЕ нормируется.
- `fix/rerank-client-sort` (2e23dac, PR #50, влита в main) — чинила баг: reranker-клиент возвращал результаты не отсортированными → чинит «сортировать выдачу reranker по убыванию score». Т.е. был реальный баг «реранкнули, но порядок не по score» — уже исправлен на main. (На ветке HEAD `84dd30e` этого фикса нет.)
- Единообразие: один `JiraReranker` инстанс инъектится во все 3 корпуса (`service.py`), поведение идентично. НО helm-корпус реранкается по обеднённому тексту (см. ниже) — источник регресса.
- Клиенту отдаётся сырой скор (RRF fused ~малые дробные ИЛИ rerank-score). Нормированного confidence/порога нет (подтверждено в осн. части).

### Cross-rerank — механизм и ПОЧЕМУ «не закрыт» (НЕ «провалили тюнинг»)
- **Механизм** (`feat/rag2-cross-rerank` 2209f98, PR #53, влита в main): после `federate()` (RRF рангов 3 корпусов) единый кросс-энкодер `tacticum/rerank` реранкает ВЕСЬ слитый пул глобально (`application/rag2.py:answer` ~:182). Развилка режимов в `service.py:170-181`: `corpus_reranker = reranker if rerank_enabled else None` (per-corpus), `cross_reranker = reranker if cross_rerank_enabled else None`. Т.е. в cross-only режиме per-corpus rerank выключается — реранк один раз по слитому списку. **Source-aware развилки «helm мимо кросс-энкодера» в смердженном коде НЕТ** — все корпуса идут в один кросс-энкодер одинаково.
- **A/B результат** (vault `cross-rerank-ab-SUMMARY`, набор B=16 запросов): off nDCG@10=**0.608**, top-1=5/16 → on nDCG@10=**0.712 (+17%)**, top-1=11/16, латентность без надбавки. НО **регресс на 4/16** (Q2/Q6/Q7/Q8): helm-хиты `1.0-xxx` проваливаются из топа. Т.е. cross-rerank даёт общий выигрыш, но роняет часть helm-ответов.
- **Диагноз регресса (root cause):** helm-требование матчит запрос по ОБОГАЩЁННОМУ embed-тексту (title+desc + управленческая **сводка покрытия по трекам**), а кросс-энкодер судит его по обеднённому `payload["text"]` (title+desc БЕЗ сводки) → систематически недооценивает helm → роняет. jira/confluence не страдают (у них `payload["text"]` = полное тело).
- **Что сделано в `fix/rag2-cross-rerank-helm-tune`** (worktree `~/tacticum-worktrees/helm-cross-rerank-tune`, база origin/main a0435f6): **(г)** source-aware `helm_rerank_text(doc)` в `domain/rag2.py:135` (зеркалит ingest-контекст helm: поколение/компоненты/заказчики/покрытие по трекам) + branch в `reranker.py:35` (`helm_rerank_text if source=="helm" else jira_rerank_text`) — чинит ОБА реранка; домап `generation/client/verdicts` в `search.py:_doc_from_payload` (реингест НЕ нужен, поля в Qdrant). **(б)** rank-floor `_apply_rank_floor` в `application/rag2.py:40` + `Rag2Policy.rerank_rank_floor` (:122) + env `HELM_RAG2_RERANK_RANK_FLOOR` (дефолт safety-net 5) — не даёт RRF-топ-10 провалиться. **Код готов, юнит-тесты зелёные (+6 тестов, 61/61 профильных), ruff/mypy чисто. НЕ смержено, НЕ запушено, изменения UNCOMMITTED в worktree** (`git status`: M на 6 src-файлах + ab-kit/).
- **Почему «не закрыт»:** живой A/B (доказать, что регресс ушёл и +17% цел) **ЗАБЛОКИРОВАН прод-гейтом** — прогон нужен против Qdrant/gateway из контейнера `helm-helm-1`, авто-классификатор блокирует exec без авторизации ЖИВОГО человека. Готов ready-to-run кит (patch + `rerank_ab_driver.py` + `rerank_ab_analyze.py` + RUNBOOK). tolerance rank-floor=5 — стартовое, эмпирически НЕ подтверждено.
- **Осталось непонятым / что попробовать:** (1) реальный вклад (г) vs (б) по отдельности — нужна абляция без floor; (2) финальный tolerance floor (sweep 0/3/5/8); (3) не появится ли регресс на не-B запросах / других корпусах; (4) альтернатива — вообще НЕ гонять helm через кросс-энкодер (source-aware обход), в коде не реализовано.
- **Вывод:** формулировка «провалили тюнинг» неточна. Точнее: **cross-rerank даёт +17%, но регрессит helm; фикс написан и юнит-протестирован, но не провалидирован A/B из-за прод-доступа.** Разблокировка = человек авторизует прогон на `helm-helm-1` ЛИБО сам гоняет RUNBOOK и отдаёт JSON.

## C+. Три сущности «Eva» — раздельно, и КАК добавить

ТЗ смешивает три разные вещи. Развожу с фактами по коду:

### 1. EVA-трекер `eva.iva.ru` (таск-трекер, аналог Jira)
- **Заготовка в коде ЕСТЬ, но двумя разными путями, не путать:**
  - **Ingest-путь СУЩЕСТВУЕТ:** `src/helm/ingest/eva_source.py` (main) — читает выгрузку EVA из `data/real/eva/` → сводит к `RawEpic/RawJiraIssue/RawIssueLink`, подключён в `real_source.py` как ВТОРОЙ трекер РЯДОМ с Jira, сигналы несут `source="eva"`, идут в genesis-эпики/граф (`loader.py:565`, `real_source.py:87-89,280`). Маппинг проектов EVA→продукт зашит. **Это управленческий/helm-контур (control-tower), НЕ векторный корпус RAG#2.**
  - **RAG#2-путь НЕТ:** RAG#2 Jira-ингест (`infrastructure/rag2/ingest.py`) хардкодит `source:"jira"` — корпус `iva_jira__bge_m3_1024` только Jira. EVA-задачи в векторный корпус аналитиков НЕ индексируются.
  - **Runtime:** `application/tasks.py:20,202` «EVA-трекер не загружается — единый SoR пока только по Jira» (данных выгрузки нет/не подключены). `domain/rag2.py:369` — EVA-метрические запросы (активные пользователи/дашборды, `_EVA` :371) роутятся в `live`, но live-коннектора к eva.iva.ru нет (транспорт-заглушка).
- **Интент-роутинг под неё:** `domain/rag2.py:369-373` (`_EVA` триггеры), `:424` (комментарий про EVA-метрики), `:465` (`eva = _has(ql, _EVA)`), `:524` («EVA — только live»). Т.е. роутер уже знает про EVA и адресует её live.
- **Как добавить:** (а) для аналитиков как корпус задач — либо положить выгрузку EVA в `data/real/eva/` и прогнать ingest (код `eva_source.py` готов), либо расширить RAG#2-ингест, чтобы индексировать EVA-задачи с `source="eva"` в векторный корпус; (б) для live-метрик eva.iva.ru — нужен MCP-инструмент/коннектор за `HttpMcpTransport` (`live_mcp.py:89`, JSON-RPC tools/call), которого нет. **Нужен доступ:** формат выгрузки EVA / API eva.iva.ru + токен.

### 2. Eva-wiki `eva-wiki.orionpro.org` (Eva DOC-000245, документация/канон)
- **Заготовки в коде НЕТ вообще.** `orionpro`, `DOC-000`, `eva-wiki` в `src/` не встречаются ни на одной ветке (единственное упоминание `orionpro`/`distrohost` — в сгенерённом дата-файле `scripts/data/ceo_code_risks_2026-07-12.json`, это анализ чужого фронта ИВА, не наш код).
- **Как добавить:** greenfield-коннектор к eva-wiki (аналог Confluence). HTML-экстрактор уже есть (`extractors.py:_extract_html` на main). Целевой корпус — либо `iva_confluence`, либо новый doc-корпус. **Нужен доступ:** URL/экспорт eva-wiki + права.

### 3. distrohost.msk/Docs/*.html (серверные контракты JUMP)
- **Заготовок НЕТ.** `distrohost` только в том же CEO-risk JSON как «мёртвый внутренний URL» (`http://distrohost.msk/Docs/Sessions.html`) в чужом коде ИВА (`libs/jump/data-access/.../jump-api-rule.interfaces.ts`, комментарий). Нашего интеграционного кода нет.
- **Как добавить (пересекается с Р-1/Р-4 `api`/`contract`):** greenfield. Нужен `source_type=api/contract` (его нет — см. осн. часть), доступ к distrohost.msk, HTML-экстрактор (есть), новый корпус/скоуп. **Нужен доступ:** сетевой доступ к distrohost.msk + структура контрактов JUMP.

### Итог по объёму «добавить Еву»
- EVA-трекер: **частично есть** (ingest-код готов, нужны данные/доступ + решение — в helm-корпус или в RAG#2-вектор).
- Eva-wiki + distrohost/JUMP: **с нуля** (нет ни кода, ни доступов). distrohost = тот же трек, что Р-4 (корпус контрактов API).
- От пользователя нужны: доступы (выгрузка/API EVA, eva-wiki, distrohost.msk) и решение, в какой корпус класть каждый источник.
