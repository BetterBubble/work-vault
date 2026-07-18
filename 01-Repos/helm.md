---
title: helm — репозиторий
type: architecture
status: current
created: 2026-07-18 22:35
updated: 2026-07-18 22:35
permalink: tacticum/01-repos/helm
repo: helm
tags:
- repo
- helm
- iva
- rag
---

# helm

**Путь:** `~/tacticum/helm` · **Прод:** `helm.tacticum.ru` (сервер ssh-manager `helm`, `/opt/helm`).
**Назначение:** заказной сервис для ИВА — «Монитор ГД» / Control Tower (управленческий overlay) + три RAG для ИВА на общем движке.
**Стек:** Python, FastAPI, SQLAlchemy async, Alembic, Postgres, uv. LLM через Gateway/vLLM в контуре ИВА.
**Taiga:** `tacticum-helm` (#32, основной, активный), `iva-control-tower` (#31).
**Деплой:** bundle → ff-merge → `SEED=0 bash scripts/deploy.sh` (rebuild; volume-mount ненадёжен) → verify `getsource`. Подробности — [[git-deploy-hygiene-helm]].

## Направления / подпроекты
- **Control Tower / дашборд** — управленческие срезы для C-level: ServiceDesk (типы обращений, клиенты×деньги, KPI), клиент↔требование, Sales/Pipeline, Бэклог/Потребности, риск-панель сроков.
- **RAG для ИВА** (концепт: [[Концепт- три RAG для ИВА на общем движне]]):
  - **RAG1** — `/docs`, ассистент документации (бот наружу). В проде.
  - **RAG2** — `/analyst`, для аналитиков+поддержки: гибридный ретрив + live-MCP (Jira/Confluence) + граф. В проде.
  - **MCP-аналитика** (analyst-MCP) — 14 тулов, детерминированные реестры (Р-1 api_registry, Р-4 contract, Р-5 effort_hint).
  - **бот поддержки** — webhook IVA → docs_ask (RAG1) → чат.

## Текущее состояние
См. живой чекпойнт [[session-state]]. Кратко (2026-07-18): RAG#2-доработка (Р-1/Р-2/Р-4/Р-5 + дотяжка) 🟢 в проде; ждут пользователя IVAONEHALF, Eva-wiki, Р-3.
