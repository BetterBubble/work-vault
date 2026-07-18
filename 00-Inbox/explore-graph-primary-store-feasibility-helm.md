---
title: explore-graph-primary-store-feasibility-helm
type: report
permalink: tacticum/00-inbox/explore-graph-primary-store-feasibility-helm
tags:
- helm
- explore
- platform
- graphiti
- falkordb
- memory
---

# Разведка — годен ли платформенный граф (Graphiti/FalkorDB) как ОСНОВНОЕ хранилище графа Helm

Черновик explorer. Продолжение #24/#26. Факты из кода `platform/lib/graphiti`, `platform/services/memory`, `helm/src/helm/substrate`, graphiti_core (Context7, getzep/graphiti).

Главный вывод-коррекция к #26: **fixed-enum — свойство MCP-обёртки, НЕ движка.** graphiti_core полностью поддерживает свою онтологию + reference_time. Но это НЕ делает граф пригодным как SoR детерминированного структурного графа — блокер в другом (LLM-экстракция на write-path + слабая агрегация чисел).

## Q1 Онтология — произвольные entity/edge types?
**Факт (движок):** graphiti_core `add_episode` (v>=0.29, Context7) принимает `entity_types: dict[str, type[BaseModel]]`, `edge_types: dict[str, type[BaseModel]]`, `edge_type_map`, `excluded_entity_types`, `custom_extraction_instructions`. Кастомные типы — pydantic-модели, их поля ложатся в `attributes` узла/ребра.
**Факт (обёртка):** `tacticum_graphiti/client.py::GraphitiClient.add_episode` УЖЕ пробрасывает `entity_types=entity_types` в движок (сигнатура строки 58-86). Т.е. lib-слой готов к своей онтологии.
**Факт (MCP):** `memory_service/tools.py::memory_write` — `type: Literal[user_profile|feedback|project_state|reference]`. Но это **record-kind индекса**, НЕ label узла графа. `records.py` (строки 74-97) вызывает `add_episode` БЕЗ `entity_types` (комментарий D4: «graphiti entity types classify extracted entities, not the record kind»; кастомная онтология — «lands in slice 4», застаблено).
**Вывод:** enum к онтологии графа отношения не имеет. Своя онтология (Обращение/Инициатива/Signal/Продукт + свои предикаты) на уровне graphiti_core/lib — ДА. Через текущий MCP-тул — нет (не проброшено).

## Q2 Прямой доступ мимо MCP?
**Факт:** Helm сегодня ходит в память ТОЛЬКО через MCP — `helm/substrate/memory.py::McpMemoryConnector` → `McpClient` (streamable-http JSON-RPC) → тул `memory_write`. Импорта `tacticum_graphiti` в Helm нет.
**Факт:** `tacticum-graphiti` — vendored editable lib (`platform/lib/graphiti`, pyproject `graphiti-core[falkordb]>=0.29.0`). `build_graphiti_client(falkordb_url, gateway_base_url, gateway_api_key, llm_model, embedder)` собирает клиента. Ничто не мешает Helm импортировать lib и собрать свой `GraphitiClient` со своей онтологией/группами.
**Вывод:** прямого пути СЕГОДНЯ нет (только MCP), но два обхода реальны: (а) импорт vendored-lib `tacticum_graphiti` и прямой `add_episode(entity_types=…, reference_time=…)`; (б) raw FalkorDB (см. Q6). Оба требуют инфры (Gateway для graphiti-экстракции; сеть до FalkorDB).

## Q3 Числа/свойства — агрегация/фильтр?
**Факт:** свойства узла/ребра — `attributes: dict[str, Any]` (Context7, nodes.py/edges.py), заполняются полями кастомных pydantic entity_types. Т.е. числовое поле объявить можно.
**КЛЮЧЕВОЙ нюанс:** attributes заполняет **LLM-экстракция из episode_body**, не детерминированная запись. Число в графе = то, что LLM вытащил из текста эпизода, а не то, что ты положил транзакционно.
**Факт (retrieval):** платформенный seam отдаёт только hybrid/semantic-поиск — `search_nodes` (NODE_HYBRID_RRF + `SearchFilters(node_labels)`), `search_facts` (edge-recipe). Агрегаций (SUM/COUNT/GROUP BY по attributes) через обёртку НЕТ. Сырой FalkorDB — OpenCypher, агрегации умеет, но мимо seam.
**Вывод:** граф — НЕ надёжное детерминированное числовое хранилище. Числа держать отдельно (Postgres). Через seam числа из графа не поагрегируешь.

## Q4 Изоляция (tenant/project)?
**Факт:** `memory_service/scope.py` — `group_id = mem:{tenant}` (+ `:u:{user}` / `:p:{project}` / `:shared`). Tenant-префикс — жёсткая граница (fail-closed, ADR-0005 D4; кросс-тенант невозможен по построению). `sanitize_group_id` (group_id.py) схлопывает non-alnum → `_` (требование RediSearch).
**Факт:** group_id — произвольная строка. Helm может выкроить свой namespace (напр. `helm:{tenant}:{project}` под инцидент-срез project_id) — при прямом lib/FalkorDB-доступе.
**Вывод:** изоляция по group_id надёжная и достаточная; per-project инцидент-срез выражается отдельным group_id. ДА.

## Q5 Темпоральность / reference_time / бэкфилл?
**Факт:** `reference_time` — event-time би-темпорального движка. В движке (Context7) — обязательный параметр add_episode. В обёртке `client.py` — `reference_time: datetime | None = None` (default now), проброшен. В `records.py::write_memory` — тоже kwarg `reference_time` (строка 69) проброшен в add_episode. **Обрыв ровно один: MCP-тул `memory_write` (tools.py) его в сигнатуре НЕ имеет** → не пробрасывает.
**Вывод:** #26 верно, что через тул бэкфилла нет — но это дырка в 1 строке сигнатуры MCP-тула. Через lib/records reference_time работает → бэкфилл истории достижим. НЕ движковое ограничение.

## Q6 FalkorDB напрямую (Redis-протокол)?
**Факт:** `config.py` — `falkordb_url = redis://falkordb:6379` (shared substrate). `factory.py` — `FalkorDriver(host, port, username, password)` из urlparse. FalkorDB = Redis-протокол + OpenCypher.
**Факт:** достижимость — вопрос сети/деплоя: `falkordb` — внутренний docker-hostname сети платформенного compose. Из Helm-сети напрямую не виден без сетевой связки.
**Вывод:** технически FalkorDB доступен по Redis-протоколу и умеет Cypher-агрегации, НО: (1) нужна сетевая связка Helm↔платформенный compose; (2) сырой доступ обходит инварианты graphiti (temporal-модель, эмбеддинги, провенанс) и платформенное governance. Использовать сырой FalkorDB как SoR = по сути self-host графа на чужом инстансе — против ADR (connect, не self-host).

## Вердикт годности
**Основной: (Б) — граф платформы годится как темпоральный/нарративный слой; детерминированное ядро остаётся в Postgres.** С важной коррекцией #26: «fixed enum» — артефакт MCP-обёртки, не потолок движка. Своя онтология, reference_time-бэкфилл, per-project изоляция — ВСЕ достижимы (через vendored-lib или доработку MCP-тула). Значит граф может нести БОЛЕЕ богатый типизированный нарратив, чем «всё в project_state».

**Почему НЕ (А) — не годится как SoR структурного графа:**
1. write-path = **LLM-экстракция**, не детерминированная транзакция. Для точного Обращение→Инициатива→Signal с гарантией целостности — неприемлемо.
2. **числа/агрегации** через seam отсутствуют; attributes заполняет LLM.
3. прямого доступа СЕГОДНЯ нет — надо строить lib-коннектор либо сетевую связку до FalkorDB (инфра-долг).
4. сырой FalkorDB как ядро = self-host на чужом инстансе, против ADR (connect, не self-host).

**Что уточнить у лида (грань В):** требует ли «структурный граф Helm» детерминированных записей и числовых агрегаций. Если ДА → Postgres-ядро обязательно, граф = нарратив (Б). Если структурному графу достаточно семантики/типизированной онтологии + темпоральности и число всё равно живёт в PG → можно поднять роль графа до «типизированный онтологический слой» (не только project_state), но не до SoR.
