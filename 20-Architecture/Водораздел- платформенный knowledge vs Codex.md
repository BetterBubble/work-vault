---
title: 'Водораздел: платформенный knowledge vs Codex'
type: report
permalink: tacticum/20-architecture/vodorazdel-platformennyi-knowledge-vs-codex-1
tags:
- architecture
- knowledge
- codex
- watershed
- platform
---

# Платформенный knowledge_rag vs мой Codex (rag_eval_service) — водораздел

**Источник на диске:** `~/tacticum/_analysis/knowledge_vs_codex_2026-06-29.md`
**Тип:** разведка (read-only, 2026-06-29). Принцип: **не плодить дубли** — для каждого «завести» сначала проверка, нет ли уже на платформе. Водораздел: переиспользуемое → платформа, специфика ЗУ → Codex.

## Ключевое
- Обновлены репы (main): platform → ADR-0008 (Compass); tacticum-dev; agents; KB-Brownfield-Bootstrap; graph-builder/tei_service без изменений. `rag_eval_service` (feature/m2) и `doc_translator` не трогались.
- MCP/Taiga восстановлены: причина прежней 401 — истёк статический `phk_`-токен project-hub в `~/.claude.json`; после перевыпуска — ОК. Скоуп токена «тактикум»: tacticum-platform (id 20), agents (18), codex (22), tacticum_dev (12), zus-codex (28).
- Архитектурный канон — из ADR platform-репы (истина): ADR-0001 (водораздел), **ADR-0005** (knowledge-rag boundaries D1–D8), ADR-0004 (arch-mcp), ADR-0007, ADR-0008.
- Платформенная сторона подтверждена живой Taiga: эпик **#13 «Knowledge/RAG & Document Processing»** (In progress, assignee Шульга, 9 историй, progress 0) — консолидация knowledge_rag (#32 hybrid / #33 docproc). Соседние эпики: #11 LLM Gateway, #14 Memory, #20 Tech Debt.

## Relations
- part_of [[20-Architecture]]
- relates_to [[TARGET — целевая архитектура knowledge v2.3]]
- relates_to [[ADR-0001 — Демо-стенд RAG ЗУ (Codex)]]
- relates_to [[Аудит текущего RAG по флоту]]
- relates_to [[glossary]]