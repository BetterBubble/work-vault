---
title: explore-26-memory-knowledge-contracts-for-helm-connectors
type: report
permalink: tacticum/00-board/explore-26-memory-knowledge-contracts-for-helm-connectors
tags:
- helm
- explore
- platform
- memory
- knowledge-rag
- contracts
status: archived
updated: 2026-07-18
---

# Разведка #26 — контракты memory/Graphiti + knowledge_rag для коннекторов Helm

Черновик explorer (продолжение #24). Факты из кода `platform/services/{memory,data/knowledge_rag}`, `platform/{contracts,sdk}`.

## 1. memory / Graphiti — точные сигнатуры MCP

Транспорт: FastMCP `memory`, streamable-http `:8080/mcp` (или stdio). Три тула:

**`memory_write`** (`tools.py`):
```
memory_write(type: "user_profile"|"feedback"|"project_state"|"reference",
             content: str, title?: str, scope: "user"|"project"|"tenant"="user",
             project_id?: str, tags?: list[str], source_ref?: str, token?: str)
→ {status:"written", type, scope, title, group_id, record_id}
```
Пишет 1 episode в Graphiti (entities+facts извлекает LLM) + 1 строку в Postgres record-index. reference_time на тул НЕ проброшен → всегда «сейчас» (см. ограничения).

**`memory_read`**:
```
memory_read(query?: str, path?: str, project_id?: str,
            disclosure: "L0"|"L1"|"L2"="L0", limit=10, token?)
```
- `path`=`<type>/<title>` → `{path, record}` из индекса.
- `query` L0 → `{disclosure, nodes:[{id,name,summary,labels}]}`.
- L1 → + `facts:[{fact,relation,valid_at,invalid_at}]`.
- L2 → + `episode_ids` (провенанс) на каждый факт и общий список.
(В `records.py` L1/L2 РЕАЛИЗОВАНЫ через `search_facts`; докстринг в `tools.py` «L1/L2 stub» устарел. Но **кастомные entity_types (D4) НЕ проброшены** — извлечение генерическое, своей онтологии Initiative/Signal через тул задать нельзя.)

**`memory_list_by_type`**:
```
memory_list_by_type(type, project_id?, limit=25, token?)
→ {type, count, records:[{id,type,title,summary,scope,episode_id}]}
```

**Модель узлов/рёбер** (`tacticum_graphiti/client.py`):
- NodeSummary (L0): id, name, summary, labels[], group_id.
- FactSummary (L1/L2): fact, name(relation), valid_at, invalid_at (**bi-temporal**), episode_ids[].
- Типы узлов/рёбер — генерические graphiti (extraction по умолчанию); типизированная онтология — задел D4, не wired.

**tenant/group_id — НЕ аргумент**. Выводится сервером: identity из project-hub `resolve(token)` → `tenant_id`+`user_id` → `scope.py` строит `group_id`:
- write: `mem:{tenant}:u:{user}` (user) / `:p:{project}` (project) / `:shared` (tenant).
- read: fan-out [user, project, shared] под tenant-префиксом (кросс-тенант невозможен).
Клиент передаёт только `scope`-уровень + `project_id` (сужает). За MCP Runtime gateway — identity из trusted-headers X-Tenant-Id/X-Auth-User-Id (`trust_gateway_headers=true`), тогда token не нужен.

**Как Helm пишет Signal→Initiative темпораль и читает velocity/дрейф**:
- Механика: писать периодические episode-снимки (content = проекция инициативы/сигнала, JSON/текст) с reference_time = реальный момент; graphiti датирует факты (valid_at/invalid_at). Читать `memory_read(query, L1/L2)` → факты с временны́ми границами → дрейф/velocity по изменению valid диапазонов.
- **Ограничения для этого юзкейса**:
  1. reference_time на MCP-тул не проброшен → бэкфилл исторической ленты через тул невозможен (только «сейчас»). Нужна доработка тула ИЛИ прямой lib-вызов `add_episode`.
  2. Типы записей — фикс-enum (user_profile/feedback/project_state/reference); «initiative»/«signal» не предусмотрены → лягут как `project_state`/`reference` эпизоды, извлечение генерическое.
  3. Вывод: memory заточен под «корпоративную память», не под произвольный доменный граф. Подтверждает рекомендацию #24 — детерминированный Signal→Initiative держать в Postgres Helm; memory использовать для нарратива/суждений/дрейфа как вторичной проекции.

## 2. knowledge_rag — сигнатуры MCP

Транспорт: FastMCP `knowledge`, http `:8090/mcp`. Три тула (`interface/tools.py`):
```
knowledge_search(query, mode:"hybrid"|"semantic"|"fulltext"="hybrid", top_k=10,
                 project_id?, filters?: dict, rerank=False, token?)
→ {hits:[{id, score, content, ref, metadata}]}

knowledge_ingest(source_type: str, mode:"semantic"|"fulltext"|"passthrough",
                 items:[{id, content?, ref?, metadata?}], project_id?, token?)
→ {indexed: n}          # passthrough = только ref+metadata (blob не хранит)

knowledge_list_collections(token?) → collections (MVP: одна активная)
```
Домен-VO (`domain/models.py`): SearchQuery/SearchHit/IngestItem/IngestRequest/Collection (frozen dataclasses).

**tenant scoping** (`domain/scope.py`): `Scope(tenant_id, project_id?, grants)`; `scope_filter` — обязательный `tenant_id` payload-фильтр, fail-closed (нет tenant → raise). `project_id`/`filters` только сужают, `tenant_id` не перекрыть. Идентичность — из project-hub (token) или X-Tenant-Id header. Идентична модели memory.

**Модель коллекций**: MVP — ОДНА активная коллекция, имя по embedding-space (`knowledge__bge-m3_1024`). Раскладка код/CV/Confluence в MVP — НЕ отдельные коллекции, а через `source_type` + `metadata` + `filters` внутри общей коллекции (логическая сегрегация). Отдельные коллекции/сторы = опция физ-изоляции под отдельный ADR (D4).

## 3. Точки интеграции для Helm

**contracts/sdk — ПУСТЫ**: `platform/contracts` и `platform/sdk/{python,typescript}` содержат ТОЛЬКО README (план: OpenAPI+JSON-Schema+MCP tool-schema → генерят типы SDK; `ScopedRepository`). Кода контрактов/типов ещё нет.
→ **Импортировать SDK сегодня нельзя.** Варианты: (а) вендорить у себя тонкие shape-типы тулов (я привёл точные сигнатуры — они стабильны, ADR-0005 D3), (б) звать как MCP-клиент по tool-schema. Рекомендация: коннектор за Port-интерфейсом Helm + вендоренные dataclass-shape'ы; когда platform-SDK появится — заменить адаптер.

**local-dev**: docker-compose есть у обоих:
- memory: `falkordb + postgres + memory-service (:8080/mcp)`.
- knowledge_rag: `qdrant + meili + service (:8090/mcp)`.
- НО оба в полном режиме требуют **Gateway (LiteLLM) ключ + project-hub service-key**. Без Gateway → degraded (memory: NullGraphitiClient — записи молча дропаются, чтение пусто).
→ **Рекомендация**: коннекторы за интерфейсом (Port) + **стаб-транспорт** для unit/интеграции (тестим на стабах, детерминировано). Полный e2e-смоук — опционально через compose, когда есть Gateway+hub креды. `trust_gateway_headers=true` + X-Tenant-Id header позволяет обойти project-hub локально (упрощает стаб/смоук без IdP).

## Итог для implementer
- Строить `MemoryConnector`/`KnowledgeConnector` за Helm-Port, вендорить shape-типы (SDK нет).
- tenant/group_id не передавать — прокидывать project-hub user-token ИЛИ X-Tenant-Id (gateway-режим); tenant авторитетен из hub.
- Тесты — на стаб-транспорте; e2e-смоук за флагом (нужны Gateway+hub).
- Для Signal→Initiative-темпорали через memory: заложить проброс reference_time (доработка тула/прямой lib) — иначе только live-снимки.
