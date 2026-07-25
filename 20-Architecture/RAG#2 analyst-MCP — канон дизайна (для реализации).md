---
title: RAG#2 analyst-MCP — канон дизайна (для реализации)
type: note
permalink: tacticum/20-architecture/rag-2-analyst-mcp-kanon-dizaina-dlia-realizatsii-1
tags:
- rag2
- mcp
- analyst
- c4
- helm
- canon
- design
---

Канон дизайна «RAG#2 как MCP для аналитиков». Решения приняты 2026-07-15 (с руководителем). Основано на разведке (`00-Board/rag2-mcp-discovery.md`), распаковке наброска (`rag2-mcp-design-distill.md`, авторский `docs/iva-analyst-mcp-design.md`), ресёрче (`analyst-docs-research.md`).

## Принцип
MCP **даёт агенту (Claude Code) всю правильную и всеобъемлющую ФАКТУРУ**, тяжёлую генерацию НЕ делает — 5 документов собирает сам агент. Data-first: сначала полнота данных в RAG#2, MCP — поверх.

## Документы аналитика (требование → задача разработчику)
1. Разбор требования; 2. **Затрагиваемые системы / impact (C4)** — где RAG даёт максимум; 3. Пользовательский сценарий / use case; 4. Функц+нефункц требования; 5. Постановка задачи разработчику (AC Given/When/Then). (+ вход «карта покрытия — что уже реализовано», против дублей). Можно расширять — критерий: реально упрощает аналитика.

## Контур
Confluence/Jira/архитектура через Claude Code (облако) — можно. Стоп только чаты/почта.

## Данные — бо́льшая часть УЖЕ в helm
- Знания Confluence/Jira → RAG#2 индекс (`/api/rag2/search,context`) — догружаем экстрактором.
- **C4-топология — реальные данные:** `ArchNode/ArchEdge` + `GET /api/cio/arch-map` (drill L1→L3: контейнер, `tech`, `description`, `repos`, owner, риски, компоненты). ✅
- Требование→системы — **выводимо** (Requirement→Component→repo→ArchNode, код есть; прямого FK нет — оставляем вывод). ✅
- Требования/realization/coverage, владельцы контейнеров — ✅ в БД.

## MCP — строим с нуля (готового «ИВА Read MCP» нет)
- helm сейчас — только REST + MCP-клиент; своего MCP-сервера нет. Строим новый **FastMCP** по паттерну платформенного `knowledge_rag` (`@mcp.tool()` в `interface/tools.py`).
- Транспорт/размещение (из наброска): streamable-http JSON-RPC, `POST /mcp/analyst` — ASGI sub-app в том же процессе helm, рядом с `/api/*`. application/domain/infrastructure не трогаем — тулы 1:1 на существующие контракты.
- Auth: Bearer→project-hub `/resolve`→tenant-gate `iva`, fail-closed. (Открытый вопрос §7.1: семантика phk/tacticum-deploy токена, rate-limit — к встрече.)
- **Путь: пока просто отдельный MCP.** Федерацию в Единый ИВА MCP — НЕ сейчас (через несколько дней, отдельно). Не проектировать под федерацию заранее.

## Тулы (всё — ДАННЫЕ)
- `analyst_search(query,k,filters)` / `analyst_context(query,...)` — знания Confluence/Jira с цитатами (обёртка `/api/rag2/*`).
- `docs_ask(query)` — публичные доки RAG#1.
- `arch_map(parent?,focus?)` / `arch_container(id)` — C4: контейнер, за что отвечает, связи, владелец (обёртка `/api/cio/arch-map`).
- `affected_systems(requirement|text)` — вывести затронутые системы (req→компонент→ArchNode).
- `requirement_coverage(query)` — что уже реализовано и где.
- `related_tasks(query)` — похожие задачи/история Jira.

## План реализации (раздача)
1. **Данные:** экстрактор Фаза 1 (идёт, `feature/rag2-extract-ph1`) → полный прогон ингеста → RAG#2 корпус. Отдельный трек.
2. **MCP-сервер:** новый FastMCP `interface/mcp/analyst_server.py` + тулы — можно строить ПАРАЛЛЕЛЬНО (arch/coverage данные уже есть; knowledge-тулы обёртывают существующий REST). Ветка, тесты, хендофф-файл.
3. Оба — через git, воркерами. Лид: канон + проверка сводок + решения.

Связано: [[RAG#2 pipeline выгрузки+ингеста — канон (для реализации)]], [[Единый MCP ИВА — видение и что можно сделать (заготовка, не сегодня)]].