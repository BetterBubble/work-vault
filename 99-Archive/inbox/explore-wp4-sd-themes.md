---
title: explore-wp4-sd-themes
type: report
permalink: tacticum/00-inbox/explore-wp4-sd-themes
tags:
- helm
- servicedesk
- wp4
- scout
- embeddings
status: archived
updated: 2026-07-18
---

# Разведка WP4 — пайплайн ТЕМ застрявших SD (helm@feature/incidents-phase1)

Черновик Scout. Карта строительных блоков для переиспользования. Только чтение.

## 1. LLM / эмбеддинги через Gateway — **embeddings endpoint УЖЕ ЕСТЬ**
- `src/helm/llm/gateway.py:98` — `GatewayClient.embed(texts) -> list[list[float]]`, реальный `/embeddings`, батч (input=list), порядок сохраняется, пустой вход → [].
- Модель: `DEFAULT_EMBED_MODEL = "tacticum/embed"` (gateway.py:52) = bge-m3/1024 (ADR-0005 D6). Chat = `tacticum/chat`. Rerank-тир `tacticum/rerank` объявлен как задел (не реализован вызов).
- Ключи/креда: `src/helm/config.py:23-27` — `HELM_GATEWAY_BASE_URL` + `HELM_GATEWAY_API_KEY` (project-hub, несёт тенант, `Authorization: Bearer`). Нет обоих → fallback на Mock/Lexical.
- Клиент через `openai` SDK поверх base_url (LiteLLM OpenAI-совм.). timeout=30 дефолт. Лимитов/ретраев батча в коде нет — вызов «как есть».
- Потребители chat: GatewayLeadTimeEstimator, GatewayCvParser, GatewayDedupJudge, GatewayTaskGoalJudge (все в gateway.py), meeting summary (`meetings.py:346`, `application/meeting_report.py`).
- **Потребитель embed сейчас ОДИН**: `src/helm/ingest/re_enrich.py:392` (GatewayEmbeddingBackend) — semantic dedup блоков кода. RAG инцидент-прототип embed НЕ использует.
- Вывод: эмбеддим САМИ через Gateway.embed, LLM-метки (chat) — только на нейминг кластера.

## 2. Семантика/кластеризация — **инфра похожести есть, clustering-либ НЕТ**
- `src/helm/ingest/re_enrich.py` — готовый скелет: `EmbeddingBackend` Protocol (:300), `LexicalEmbeddingBackend` офлайн TF-baseline (:356), `GatewayEmbeddingBackend` (:384), `_cosine` (:402), `_l2_normalize` (:395), `find_duplicate_blocks` порог+пары (:417), `make_embedding_backend(settings)` своп-фабрика (:478), `DEFAULT_DEDUP_THRESHOLD=0.6`.
- pyproject/uv.lock: только `openai>=1.40` + `httpx`. **НЕТ** numpy/sklearn/hdbscan/scipy/umap/qdrant-client/networkx (0 в lock). → кластеризацию писать руками (граф косинуса + connected-components/greedy на чистом python), либо делегировать в knowledge_rag.
- knowledge_rag коннектор из Helm ЕСТЬ: `src/helm/substrate/knowledge.py` (`KnowledgeConnector`/`McpKnowledgeConnector`, `SearchHit`, `IngestItem`, `make_knowledge_connector`) + `mcp_client.py`; endpoint `knowledge_mcp_url` (config.py:35), активен только с `projecthub_token`. Платформенный Qdrant+Meili — потенциальный store векторов/semantic search.

## 3. Витрина/данные — точки монтажа по образцу sd_request
- Модель: `src/helm/infrastructure/db/models.py:571` `SdRequest` (PK=(source,ext_id)), поля title/client/product/tariff/age_days/created/... Комментарий и test-флаг НЕ хранятся (см. п.5).
- Провенанс/идемпотентность: `IngestRun` (models.py:502, sha-skip per source). Образец витрины-таблицы: `RepoRow` (:537), `ConformanceRow` (:518).
- Ingest: `src/helm/ingest/servicedesk.py` — `load_servicedesk` (:178), `_load_source` (:134, DELETE WHERE source + bulk insert + ingest_run), мапперы `_naumen_mapper` (:79, title=«тема»), `_crm_open_mapper` (:107, title=«Заголовок»).
- Seed-скрипт: `scripts/refresh_servicedesk.py` (45 строк) — тонкая обёртка над load_servicedesk. Шаг тем добавлять ОТДЕЛЬНЫМ скриптом `refresh_sd_themes.py` (embed+cluster+label поверх готового sd_request) ИЛИ доп. фазой после load_servicedesk.
- Роутер: `src/helm/interface/api/routers/servicedesk.py` (`router`, эндпоинты overview/stuck/requests/timeline/facets/clients-money). Монтаж: `src/helm/main.py:75` `app.include_router(servicedesk.router, dependencies=_AUTH)`. Новый `/api/servicedesk/themes` — либо в этот router, либо новый + include в main.py.
- Web: `web/src/screens/ServiceDesk.tsx`; api-обёртки `web/src/api.ts:159-168` (serviceDeskOverview/Facets/Stuck/requests/timeline); навигация `web/src/screens/roles.tsx:152,189`. Добавить `serviceDeskThemes()` + панель.

## 4. Гейт оператора — **два готовых паттерна, писать с нуля не нужно**
- **Annotation** (models.py:456): append-only override, `kind`/`value`/`reason`(обязательно)/created_at, FK→initiative. API: `POST /api/annotations` (`inputs.py:171`), `GET /api/annotations` (:181). Канонический аудируемый override.
- **PATCH-override инициативы**: `initiatives.py:25` `PATCH /api/initiative/{id}` — body с изменениями + `reason`, `null` очищает override.
- **Инлайн per-row override**: `meetings.py:200` `PATCH /{meeting_id}/decisions/{decision_id}` c `override: red|amber|green|null` — простейший паттерн «оператор меняет поле строки».
- **Snapshot** (models.py:477) — фиксация недели (payload+committed_at), паттерн «зафиксированное аудируемое событие».
- Рекомендация под §5A (подтвердить/переименовать/слить темы): таблица `sd_theme` со `status` (proposed|confirmed) + operator-label + FK на audit по образцу Annotation; эндпоинт PATCH по образцу decisions/initiative override.

## 5. Текст и фильтры — поля + пробел ingest
- crm_open CSV `data/real/data-room/support/crm_open_requests.csv` (2 строки преамбулы): 782 строки. Колонки включают `Placeholder/test`, `Заголовок`, `Комментарий клиента`, `Решение`.
- **`Placeholder/test` = флаг мусора ПОДТВЕРЖДЁН**: значения True/False. True=152, False=630 (~19.4% мусора). Мэппер `_crm_open_mapper` (servicedesk.py:107) его НЕ читает → **в SdRequest не хранится (пробел ingest)**.
- **`Комментарий клиента`**: заполнен 782/782 (100%). Мэппер его тоже НЕ читает → **не хранится (пробел ingest)**. `Заголовок` 782/782 (100%) → хранится в `title`.
- naumen (`data/real/service/sd_tasks.csv`): «тема» → `title` (servicedesk.py:100). Отдельного комментария нет.
- Текст в эмбеддинг: crm_open «Заголовок» + «Комментарий клиента»; naumen «тема». Оба crm-поля заполнены на 100% — хорошая база.
- Фильтр застрявших: уже есть `age_days > stuck_days` (дефолт 90) в `servicedesk.py:173` (clients-money) и `stuck` эндпоинт (:158). Baseline-паритет = subset `age_days>90` И `Placeholder/test=False`.

## Итог для дизайна
- Эмбеддинги: переиспользовать Gateway.embed + re_enrich backends/cosine/factory. Кластеризацию — руками (нет sklearn).
- Ingest: расширить `_crm_open_mapper` двумя полями (comment, is_test) + миграция SdRequest — иначе нечем фильтровать мусор и нечего эмбеддить (только title).
- Гейт: клонировать Annotation/decisions-PATCH под sd_theme.
