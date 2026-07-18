---
title: 01-Repos — карта проектов
type: index
status: current
created: 2026-07-18 22:35
updated: 2026-07-18 22:35
permalink: tacticum/01-repos/index
tags:
- repos
- projects
- map
- canon
---

# Карта проектов

Точка входа «какие у меня репозитории и направления». Модель: **репозиторий → направление → подпроект** (см. `~/claude-stack/memory-protocol.md` §7). Taiga-проект указан, ЕСЛИ есть — он опционален.

> ⚠️ Первый черновик (2026-07-18) — собран из разведки + Taiga. **Правь прямо здесь в Obsidian** под свою реальную структуру: направления/подпроекты знаешь только ты.

## Заказные сервисы для ИВА (клиент, tenant iva/tacticum)
- **[[helm]]** (`~/tacticum/helm`) — активный. «Монитор ГД» / Control Tower + три RAG для ИВА. Taiga: `tacticum-helm` (#32), `iva-control-tower` (#31). Направления:
  - **Control Tower / дашборд** — управленческие срезы, ServiceDesk, клиент↔требование
  - **RAG для ИВА** → RAG1 (`/docs`, ассистент документации) · RAG2 (`/analyst`, аналитики+поддержка) · MCP-аналитика (analyst-MCP) · бот поддержки
- **iva-rag / iva-rag1-engine / iva-rag1-docs / iva-rag2** — код/eval-baseline RAG для ИВА (уточнить: отдельно или внутри helm). Taiga: под `tacticum-helm`.
- **rag_eval_service** (codex-дубль) — боевой eval-сервис RAG. Taiga: `tacticum-codex` (#22).

## Внутренние продукты (платформа Tacticum)
- **platform** — Core-слой (консолидация кода, SDK, контракты, профили деплоя). Taiga: `tacticum-platform` (#20).
- **agents** (LangGraph Builder) — backend мульти-агентных ботов (Telegram/REST). Taiga: `tacticum-agents` (#18).
- **tacticum-dev** (catalog-mcp) — SaaS «virtual dev team» + каталог агентов/скиллов/MCP. Taiga: `tacticum-tacticum_dev` (#12).
- **graph-builder** (TactFlow) — визуальный конструктор графов/ботов + Rasa-симулятор. Taiga: — (уточнить).

## Инфраструктура / песочницы
- **tei_service** — сервис эмбеддингов (bge-m3, TEI). Taiga: под `tacticum-platform`.
- **local-stack** — локальная песочница retrieval + eval-harness (recall/mrr/ndcg + тест изоляции тенантов).
- **doc_translator** (DocTranslate) — перевод документов с сохранением вёрстки.
- прочие: `zu-deploy`, `KB-Brownfield-Bootstrap`.

## Клиент ЗУ (Залог Успеха, tenant cifragen)
- **ЗУ / knowledge-base** — Taiga: `cifragen-zus-codex` (#28). Направление LightRAG закрыто (граф проиграл вектору) → в [[99-Archive]].
