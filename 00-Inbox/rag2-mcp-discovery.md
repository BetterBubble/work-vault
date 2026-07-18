---
title: Разведка — RAG#2 как MCP + C4-топология для аналитиков
date: 2026-07-15
tags:
- explore
- rag2
- mcp
- c4
- helm
- analyst
status: draft
permalink: tacticum/00-inbox/rag2-mcp-discovery-1
---

# Разведка: «RAG#2 как MCP» + «C4-топология для аналитиков»

Репо: `~/tacticum/helm` (Serena активна), `~/tacticum/platform` (grep/Read), `~/tacticum/agents` (только grep, поверхностно). Read-only.

## TL;DR

- **«ИВА Read MCP» как готовый расширяемый сервер ОТСУТСТВУЕТ.** Сейчас есть только HTTP `/api/rag2/{search,context}` и `/api/docs/ask` в helm. MCP-обёртка над ними — **design-only документ**, написан сегодня (`docs/iva-analyst-mcp-design.md`, untracked, ещё не в git), код (`interface/mcp/analyst_server.py`) **не существует**.
- helm сегодня — только **MCP-клиент** (`substrate/mcp_client.py`), потребляет платформенные `memory`/`knowledge` MCP и live `iva-mcp` (Atlassian). Собственным MCP-сервером не является.
- C4-топология — **есть как реальные данные** (не только доки): таблицы `ArchNode`/`ArchEdge` в БД helm, REST `GET /api/cio/arch-map` отдаёт узлы+рёбра+owner+риски+компоненты. Это готовый источник для тула вида `arch_map`/`arch_container`.
- «Воркфлоу аналитика 5 документов» — **термин нигде не найден** (ни в helm docs, ни в Taiga-поиске по проекту `iva-control-tower`). Ближайшее найденное — «Ассистент требований» (Q&A + приём, US #24/#25, задачи #27-#35) — но это про Q&A/intake, не про «5 документов».

---

## 1. «ИВА Read MCP» — что это и как расширяется

**Кандидаты и что они на самом деле:**

| Кандидат | Что это | Роль | Расширяемо нами? |
|---|---|---|---|
| `iva-mcp` (Atlassian) | Внешний продукт ИВА, live Jira/Confluence. В этой сессии уже подключён как `mcp__iva-mcp__*` (jira_*, confluence_*, фиксированный набор тулов). В коде helm используется как **источник**: `infrastructure/rag2/live_mcp.py` мапит логические операции (`issue`→`jira_get_issue`, `conf_page`→`confluence_get_page` и т.д., см. `TOOLS` dict) через порт `MCPTransport`. | data-source для live-догрузки свежести RAG#2 | **Нет** — чужой сервер, только вызываем существующие тулы |
| `platform/services/runtime/mcp_runtime` | Edge-auth gateway + registry (ADR-0007), НЕ сам источник знаний. Traefik forwardAuth (Bearer → project-hub `/resolve` → `X-Tenant-Id`/`X-Scopes`), `registry.json` со списком зарегистрированных MCP (`memory`, `knowledge`, `arch`). Phase 1a готов (18/18 тестов), Phase 1b (прод-миграция за Traefik) — под подтверждение. | gateway/registry | Нет — это инфраструктура доступа, не хранит тулы сам |
| `platform/services/data/knowledge_rag` | Платформенный **generic** multi-tenant MCP `knowledge` (FastMCP, `mcp.server.fastmcp`). Тенант берётся из project-hub токена, никогда из аргументов (fail-closed). | универсальный RAG-движок (не ИВА-специфичный) | **Да, это и есть живой пример «как добавить тул»** (см. ниже) |
| helm `interface/mcp/analyst_server.py` | **НЕ СУЩЕСТВУЕТ.** Спроектирован в `docs/iva-analyst-mcp-design.md` (2026-07-14, untracked) как тонкая FastMCP-обёртка поверх уже существующих `Rag2Orchestrator.answer()` и `DocsAssistant.ask()`. | proposed — «ИВА Read MCP» для аналитиков | Это и есть то, что нужно **построить** |

**Вывод:** сущности «ИВА Read MCP» как готового расширяемого сервера в коде нет. Дизайн уже готов (см. §2) — экспонировать RAG#2/RAG#1 через новый `POST /mcp/analyst` в том же процессе FastAPI helm, рядом с `/api/*`.

**Как регистрируется новый тул — живой пример (`platform/services/data/knowledge_rag`):**
- `interface/server.py` — создаёт `mcp = FastMCP("knowledge", host=..., port=8090, lifespan=_lifespan, stateless_http=True, instructions="...")`.
- `interface/tools.py` — импортирует `mcp` из `server.py`, каждый тул — обычная async-функция с `@mcp.tool()`, первым параметром `ctx: Context`, дальше — типизированные аргументы (Pydantic-friendly), тело резолвит scope через `_resolve_scope(ctx, token, project_id)` (gateway-headers либо Bearer→project-hub), зовёт use-case, возвращает `dict[str, Any]`.
- Пример: `knowledge_search`, `knowledge_ingest`, `knowledge_list_collections` (файл `src/knowledge_rag/interface/tools.py`, 123 строки).
- Регистрация в реестре платформы — запись в `platform/services/runtime/mcp_runtime/registry.json` (`tool_surface: [...]`, `endpoint`, `auth_mode`).

Для helm design-документ (`docs/iva-analyst-mcp-design.md` §5.1) предлагает ту же схему: `interface/mcp/analyst_server.py` (новый тонкий слой) → тулы `analyst_search`/`analyst_context`/`docs_ask` мапятся 1:1 на существующие HTTP-контракты, application/domain/infrastructure не трогаются.

---

## 2. Что RAG#2 уже отдаёт как MCP/API

**Только HTTP, MCP-обёртки нет.** helm — FastAPI-приложение (`src/helm/main.py`), роутер `interface/api/routers/rag2.py` (234 строки), `prefix="/api"`, `tags=["rag2"]`:

- `POST /api/rag2/search` — `Rag2SearchIn{query,k,filters}` → `Rag2SearchOut{query,mode,structural,as_of,results[],disclaimer,no_answer}`. Компактный список задач (key/title/status/url/freshness/as_of), без генерации.
- `POST /api/rag2/context` — тот же вход → `Rag2ContextOut{query,mode,context,citations[],structural,as_of,degraded,needs_confluence_body,disclaimer,answerable}`. Готовый цитируемый контекст-блок `[n]` для LLM-клиента.
- Оба зовут `ctx.orchestrator.answer(query, k, filters)` (`application/rag2.py::Rag2Orchestrator`), контекст лениво собирается и кэшируется на `app.state.rag2_context` (`_get_context`, `infrastructure/rag2/service.py::build_rag2_context`). Не сконфигурирован → 503; сбой ретрива → 502; пустой запрос → 422.
- Соседний `/api/docs/ask` (RAG#1, `routers/docs.py`) — не читал детально (не в фокусе Block 2), но упомянут в дизайн-доке как второй кандидат на MCP-обёртку (`docs_ask`).
- `substrate/mcp_client.py` (178 строк) — это **клиентская** сторона: `McpClient` (streamable-http, JSON-RPC 2.0, httpx) для вызова платформенных `memory`/`knowledge` MCP (`config.py`: `memory_mcp_url`, `knowledge_mcp_url` = `https://mcp.tacticum.ru/{memory,knowledge}/mcp`) — никак не относится к отдаче RAG#2 наружу.

**Итог:** RAG#2 сейчас — чистый REST. MCP-транспорт для аналитиков нужно строить с нуля по дизайн-доку §4-5 (streamable-http, `POST /mcp/analyst`, ASGI sub-app в том же процессе).

---

## 3. C4 / архитектурная топология в helm

**Это данные, не только доки** — отвечает на вопрос лида напрямую.

- **Модель:** `ArchNode` + `ArchEdge` + `ArchNodeRepoOverride` (`infrastructure/db/models.py:1697-1753`). Иерархия L1 (системы) → L2 (контейнеры) → L3 (компоненты) через `parent_id`. `kind`: person|system|system_ext|container|db|queue|component. Поля узла: `id` (slug), `level`, `kind`, `title`, `tech`, `description`, `grp` (группа-boundary), `repos` (`;`-список), `owner_person_id`.
- **Сид:** `scripts/ingest_arch_c4.py` — идемпотентный source-replace из `scripts/data/arch_c4.json` (авторский каталог в git). Риски — отдельно, `scripts/ingest_arch_risk_cards.py`/`ingest_arch_risks.py` → `ArchRisk`.
- **REST API (готово, продакшн):**
  - `GET /api/cio/arch-map?parent=&focus=&window_days=90` → `ArchMapOut{parent, breadcrumb[], nodes[ArchNodeOut], edges[ArchEdgeOut]}`. Drill-down по уровням; на каждом узле — `repos`, `has_children`, `risk_count`/`risk_high`, `commits`/`effort`/`top_committers` за окно, `owner_name` (CTO/техлид, US #123/#125), `components[]` (выведенные продуктовые компоненты через `ComponentRepo`), `mono_repos[]`. `focus` — deep-link из карточки требования (`?node=`) без явного parent — отдаёт уровень, где виден сам узел.
  - `GET /api/cio/arch-owners`, `GET/PATCH /api/cio/arch-nodes/{id}/owner[-candidates]`, `POST/DELETE /api/cio/arch-nodes/{id}/repo-override` — управление владельцами контейнеров.
  - `GET /api/cio/arch-risks` — открытые арх-риски.
  - Роутер: `interface/api/routers/cio.py` (класс большой, ~5000+ строк, arch-часть — строки 2820-5200+).
  - Фронт: `web/src/screens/ArchRisks.tsx`, `web/src/api.ts` (`cioArchOwners`, `arch-map` и т.д.).

- **«Какие системы затрагивает требование» — НЕ прямое поле.** У `Requirement` нет `arch_node_id`. Связь **косвенная**: `Requirement` → `RequirementComponent` (M:N на `Component`) → `Component`/`ComponentRepo` (репо по треку kmp/rn/native) → сопоставление с `ArchNode.repos` (уже реализовано как «выведенные компоненты» в `arch_map` через `_repo_component_names`/`_container_rows`, и отдельно как `component_repo_candidates` эндпоинт на основе evidence). Т.е. связка «требование → контейнер C4» **выводима существующим кодом**, но не хранится как явный FK — если аналитику нужен прямой ответ «какие системы затронуты», это либо (а) переиспользовать существующий join-путь Requirement→Component→repo→ArchNode, либо (б) завести явное поле (нет в текущей схеме).
- Wiki «Архитектура Helm» — **не проверял** (не блокер, как договорились); C4 данные и так полностью в БД/API, вики может дублировать текстом.

---

## 4. Воркфлоу аналитика «5 документов»

**Термин не найден нигде** — ни в helm (`docs/`, `docs/superpowers/{specs,prd}/`), ни полнотекстовым поиском по Taiga-проекту `iva-control-tower` (id 31; пробовал «5 документов», «use case», «установка разработчику» — 0 результатов везде).

**Что нашлось близкое — это ДРУГОЙ поток**, «Ассистент требований» (RAG#3a, чат-бот в helm, не про C4/RAG#2):
- Дизайн: `docs/superpowers/specs/2026-07-13-requirements-assistant-design.md` (принят к реализации 2026-07-13).
- PRD: `docs/superpowers/prd/2026-07-13-requirements-assistant-prd.md`.
- Хендофф (статус на 2026-07-13, лиду Шульге): `docs/superpowers/specs/2026-07-13-requirements-assistant-handoff.md` — базовый Q&A **уже в проде** (embed bge-m3 → Qdrant → evidence из БД → генерация triva → ответ с [№]+вердиктами). Задачи: **#31** (приём требований: `structure`→`coverage`→`register`, домен готов, эндпоинты — нет), **#32** (reranker `tacticum/rerank`, ещё не сделан), #34 (ADR-0003+деплой), #30/#33 (фронт, на Лебедеве).
- Это **два действия аналитика/presale**, не пять документов: (1) Q&A «что уже есть в IVA» с доказательствами (№/статус/трек/квартал/компонент), (2) приём текста требования → структурирование (guided_json) → карта покрытия (met/partial/absent) → регистрация в реестре (`origin=helm, status=candidate`) через существующий `create_requirement`.
- Что RAG генерит vs аналитик решает: **RAG только предлагает черновик** (`RequirementDraft`) и карту покрытия — регистрация только по подтверждению человека (HITL, архитектурный запрет «прямая передача в разработку»).

**Гипотеза (не подтверждена):** «5 документов» может быть термином из внешней встречи/презентации (`docs/concept/iva-synergy/management/процессы/requirements/presentation.md`, слайды 8/10/11 — упомянута как источник дизайна ассистента требований, но **сам файл в репо helm отсутствует** — либо не выгружен, либо лежит вне репозитория/в вики). Не проверял `agents` репо глубоко (только поверхностный grep на `fastmcp` — ничего релевантного по «5 документов» не нашлось).

---

## Пробелы / вопросы к лиду

1. **«ИВА Read MCP» — строить с нуля.** Готового сервера для расширения нет; ближайший живой референс паттерна регистрации тулов — платформенный `knowledge_rag` (`@mcp.tool()` в `interface/tools.py`). helm-дизайн (`docs/iva-analyst-mcp-design.md`) уже расписан детально (auth, транспорт, размещение) — **но это черновик, не в git**, и явно помечен «Статус: дизайн-документ (не реализация)», с открытыми вопросами (§7.1: scope тулов RAG#1 вкл/выкл, семантика «phk / tacticum-deploy» токена, rate-limit). Нужно решение — превращать ли этот черновик в ADR/спеку и коммитить.
2. **Требование → системы (C4)** — нет прямого поля, только выводимая связь через компоненты/репо. Если аналитику нужен быстрый прямой ответ «что затронуто» — обсудить, заводить ли явную связь `requirement ↔ arch_node`, или опираться на существующий вывод через `Component`.
3. **«5 документов» — не нашёл.** Нужно либо (а) уточнить у источника термина (созвон/презентация?), либо (б) прислать ссылку/скрин — я поищу точнее по конкретным словам. Файл `docs/concept/iva-synergy/management/процессы/requirements/presentation.md`, на который ссылается спека ассистента требований, физически отсутствует в репо helm — возможно, там и есть искомое.
4. **docs/iva-knowledge-rag-concept*.md (v1 удалён, v2/v3 + vision — новые untracked файлы в git status этой ветки)** — не разбирал подробно построчно (не входило в 4 блока), но они явно источник контекста для этой задачи (в т.ч. упоминают «MCP для аналитиков» как «интерфейс 2», см. v2 строка 86). Может стоит прочитать их лиду целиком, если ещё не читал — они выглядят как рабочий черновик прямо под эту задачу.
5. Wiki «Архитектура Helm» не проверял (по договорённости не блокер) — если нужно свести с C4-данными из БД, дам отдельным заходом.