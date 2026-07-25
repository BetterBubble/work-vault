---
title: 'ADR — Helm: PG-ядро + подключение к платформенному субстрату (Graphiti/Gateway/knowledge_rag)'
type: decision
permalink: tacticum/21-decisions/adr-helm-pg-iadro-podkliuchenie-k-platformennomu-substratu-graphiti-gateway-knowledge-rag-1
tags:
- helm
- control-tower
- adr
- architecture
- graphiti
- substrate
- tenant-isolation
---

ADR по субстрату Helm. Основа — канон [[control-tower-v02]] §8.2/§9.1 + разведка platform #24 + чеклист [[control-tower-v02 — чеклист выравнивания (канон = спека)]]. Решение оператора 2026-07-03.

## Контекст
Канон §8.2 предписывает субстрат Graphiti + LightRAG + knowledge_rag + LLM Gateway. Разведка #24 установила по факту: Graphiti — **connect** к `platform/services/memory` (MCP `memory.*`, либа `tacticum_graphiti` над `graphiti_core`), бэкенд **FalkorDB** (Redis+RediSearch), НЕ Neo4j/AGE/PG. Изоляция — **group_id `mem:{tenant}` fail-closed by construction** (структурная, не «слабый group_id»). Gateway = LiteLLM (`services/ai`), knowledge_rag = Qdrant+Meili (`services/data`, deploy-ready), RE = `tacticum_re` (зрелый).

## Решение
- [decision] **Детерминированный граф `Signal→Initiative` + расчёты + аннотации ЕОЛ остаются в Postgres** (операционное ядро, §9 «FastAPI+Postgres», §0.10 «аннотации Helm ведёт сам»). НЕ переносим их в Graphiti. #core
- [decision] **Подключаемся к платформенному субстрату (connect, НЕ self-host, НЕ Postgres+AGE):** Graphiti (`services/memory`, FalkorDB) — темпоральный/нечёткий слой (velocity, дрейф приоритетов, semantic-dedup, нарратив); knowledge_rag (`services/data`, Qdrant+Meili) — доки/CV/код; LLM Gateway (`services/ai`, LiteLLM) — инференс (lead_time-черновик, скоринг, dedup-judge, embed/rerank bge-m3). #connect
- [decision] **Тенант-изоляция — ЛОГИЧЕСКАЯ** (group_id/tenant_id fail-closed, дефолт платформы, симметрично memory/knowledge_rag). Физическая (выделенный инстанс на тенант, ADR-0005 D4 через тот же memory-контракт) — ops-опция, если ИВА потребует КИИ/on-prem. #isolation
- [decision] **Реконсиляция Шага 4:** «граф → Graphiti» читаем как «темпоральная/knowledge-проекция → Graphiti», детерминированное ядро — PG. Явное отступление от БУКВАЛЬНОГО Шага 4, обоснованное природой Graphiti (bi-temporal KG под LLM-факты, не транзакционная CRUD-операционка) + §9 (Postgres для веб-аппа). #reconciliation
- [decision] **Субстрат — АДДИТИВНО, по мере 1b (НЕ rewrite ядра):** Gateway (Шаг 5), Graphiti/knowledge_rag/RE (Шаги 3–4, потребляют git/код/CV = 1b-данные). Ядро 1a переиспользуется как есть. #additive

## Последствия
- [consequence] Helm ЗАВИСИТ от развёрнутой платформы (достижимость + project-hub ключ/тенант). Коннекторы строим за нашими интерфейсами, тестируем со стабом, реальные endpoint+ключ подключаются при доступности платформы. #dependency
- [consequence] Postgres+AGE ОТВЕРГНУТ (оправдан только если Helm обязан не зависеть от платформы). Neo4j-вопрос снят (платформа на FalkorDB). #rejected
- [need] Для реального подключения нужны: base_url Gateway + project-hub service-key/тенант + достижимость memory/knowledge_rag. → в список запросов к IVA. #need

## Открытые (уточнить у автора канона / отложить на 1b)
- [open] **LightRAG** — как OSS-либы в платформе НЕТ; в каноне §8.2 это собирательный термин к прототипу `rag_eval_service`. Трактовать как knowledge_rag? #open
- [open] **semantic_dedup** — отдельным named-модулем в RE не подтверждён (dedup-логика размазана). Уточнить §8.2 / реализовать в 1b поверх Graphiti+embed. #open

## Отношения
- implements [[control-tower-v02]]
- relates_to [[control-tower-v02 — чеклист выравнивания (канон = спека)]]
- relates_to [[Helm — что запросить у IVA (данные-доступы-ресурсы для 1a-1b)]]

## Уточнение контрактов (разведка #26)

- [contract] **memory/Graphiti** MCP `:8080/mcp`: `memory_write(type, content, title?, scope, project_id?, token?)` — type = ФИКС-enum (`user_profile/feedback/project_state/reference`); `memory_read(query?/path?, disclosure L0/L1/L2)` — L0 nodes, L1 +facts (bi-temporal valid/invalid), L2 +episode provenance; `memory_list_by_type`. **tenant/group_id НЕ аргумент** — выводится сервером из project-hub `resolve(token)` → `mem:{tenant}`, либо X-Tenant-Id header за MCP Runtime (trust_gateway_headers). #contract
- [limitation] memory: кастомные entity_types НЕ проброшены (онтологию Initiative/Signal через тул не задать); reference_time НЕ проброшен (бэкфилл истории невозможен, только «сейчас»). → ЖЕЛЕЗНО подтверждает: детерминированный граф — в PG; memory использовать для нарратива/дрейфа (периодические project_state-эпизоды → факты с valid_at). Темпораль-бэкфилл потребует доработки тула/прямого lib-вызова. #limitation
- [contract] **knowledge_rag** MCP `:8090/mcp`: `knowledge_search(query, mode hybrid/semantic/fulltext, top_k, project_id?, filters?, rerank, token?)`, `knowledge_ingest(source_type, mode, items, project_id?, token?)`, `knowledge_list_collections`. tenant_id fail-closed (Scope), project_id/filters сужают. MVP — ОДНА коллекция `knowledge__bge-m3_1024`; код/CV/Confluence — через `source_type`+`metadata`+`filters` (не отдельные коллекции). #contract
- [integration] **platform/contracts + platform/sdk ПУСТЫ** (только README, кода типов нет) → SDK импортировать нельзя. Вендорим тонкие shape-dataclass за **Helm-Port** (сигнатуры стабильны, ADR-0005 D3), заменим на platform-SDK когда появится. #integration
- [integration] **local-dev**: docker-compose есть у memory (falkordb+pg+svc) и knowledge (qdrant+meili+svc), НО полный режим требует Gateway(LiteLLM)+project-hub key (без Gateway memory→NullGraphitiClient, записи дропаются). → Коннекторы строим за Port + стаб-транспорт, тесты на стабах; e2e-смоук за флагом через compose (X-Tenant-Id обходит IdP локально). #integration
- [plan] Приоритет коннекторов: **Gateway** (#25, есть потребитель — lead_time/скоринг) → **knowledge_rag** (1b: docs/CV/код, semantic_dedup) → **memory** (1b: нарратив/дрейф). memory/knowledge строим, когда готовы 1b-фичи-потребители, не спекулятивно. #plan

## Разведка «граф основной?» (2026-07-06) — ADR подтверждён, коррекция #26

- [decision] **Вердикт (Б): платформенный граф НЕ годится как SoR структурного графа Helm, годится как нарративный/онтологический слой.** ADR (PG-ядро + граф-надстройка) подтверждён разведкой, не опровергнут.
- [reference] Причины «не SoR»: (1) write-path Graphiti = **LLM-экстракция из текста**, не детерминированная запись — наши связи (обращение→продукт→команда) точные факты из CSV, нельзя доверять LLM; (2) **нет числовых агрегатов** через seam (SUM/COUNT) — числа в PG (канон §8.3); (3) прямого lib-доступа сегодня нет (инфра-долг); (4) сырой FalkorDB = self-host против ADR.
- [correction] **#26 уточнён:** «fixed enum» (user_profile/…) — артефакт MCP-тула `memory_write`, НЕ потолок движка. graphiti_core `add_episode` принимает `entity_types`/`edge_types`; `tacticum_graphiti/client.py` их пробрасывает; `reference_time` тоже проброшен в lib, обрыв только в сигнатуре MCP-тула. → своя онтология + бэкфилл истории достижимы ЧЕРЕЗ LIB (не через текущий MCP). Изоляция group_id `mem:{tenant}` — произвольная строка, per-project ДА.
- [plan] **Возможность на Фазу 2+ (не сейчас):** поднять роль Graphiti с «всё в project_state» до **типизированного онтологического нарратива** (свои entity_types через lib) — обогащает темпоральный слой, но НЕ делает граф SoR. Детерминированный граф + числа остаются в Postgres.
- [reference] Черновик разведки: `00-Board/explore-graph-primary-store-feasibility-helm`.

## Уточнённое прочтение §8 канона v03 (2026-07-06) — граф-backbone = Graphiti, не Postgres

Прочитан §8 v03 целиком. Прежняя формулировка «детерминированный граф в Postgres» была НЕТОЧНОЙ — канон §8.3 ставит иначе. Согласованная позиция:

- [decision] **Целевая архитектура (§8.3-A, три плоскости):** (1) **Граф связей — Graphiti (bi-temporal), BACKBONE** — узлы Person/Team/Block/ЕОЛ/Initiative/Product/SalesInitiative/Signal/Task/Repo-Commit/Goal, рёбра owns/assigned/works_on/blocks-depends_on/part_of/sells_to/causes; (2) **Числа — warehouse (DuckDB/Postgres поверх CSV)** — точные агрегаты (ФОТ/суммы/counts/маржа), text-to-SQL, НЕ вектор; (3) **Тексты — knowledge_rag** (Qdrant+Meili). **LightRAG (§8.2) — graph-RAG поверх (dual-level retrieval).**
- [decision] **Граф в центре = КАНОН** (задумка оператора верна): Graphiti — основной слой связей, LightRAG поверх. Postgres в каноне = warehouse для ЧИСЕЛ, а не хранилище графа. НЕ строить свою графовую СУБД (Neo4j/FalkorDB self-host) — это отклонение от канона (канон предписывает платформенный Graphiti).
- [decision] **Волновая раскладка §8.3-J примиряет «сейчас vs целевое»:** 1a — только структурная плоскость (Postgres) + **граф-lite**, без вектора/LLM в критпути (уже работает); 1b — вектор + identity + router + governance-фильтр; 2–3 — **Graphiti-темпоральность** разворачивается полностью. → Прежний «граф в Postgres» — это 1a-реализация (граф-lite сейчас), НЕ отмена целевого Graphiti-графа.
- [correction] Детерминизм графа: канон §8.3-B хочет **upsert в граф** (детерминированная запись узлов/рёбер из loaders). Платформенный Graphiti ЧЕРЕЗ MCP-тул = LLM-экстракция (не детерминизм) — но ЧЕРЕЗ lib (`graphiti_core`, свои `entity_types`) детерминированный upsert + онтология достижимы (разведка graph-scout). → Канонный граф в Graphiti реален через lib-доступ, не через текущий MCP-обёртку. Это инфра-долг на волну 2-3.
- [reference] Разведка граф-в-центре (Workflow, 2026-07-06): по ФАКТИЧЕСКОМУ профилю данных сейчас числовой аналитики+транзакций больше (2178 сделок, экономика, incidents/needs = сводные таблицы), настоящий граф-обход нагружен только DAG зависимостей (7844 ребра, тянет recursive-CTE); центр Initiative пуст (0), сшивки Т2/Т3/sales-мост не построены. → Граф-выигрыш сегодня мал+аспирационный; полноценно окупается на волне 2-3 когда центр материализуется. Подтверждает §8.3-J (сейчас структурная несёт, граф разворачивается).

- relates_to [[Helm — стратегия построения: вертикальный срез, не горизонталь; ручной прогон = проект пайплайна]]