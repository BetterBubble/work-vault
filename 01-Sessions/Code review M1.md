---
title: Code review M1
type: note
permalink: tacticum/01-sessions/code-review-m1-1
tags:
- session
- code-review
- codex
- m1
---

# Code review M1 (document_processing + RAG-слой + Ask BFF)

**Источник на диске:** `~/tacticum/fast_task/code-review.md`
**Тип:** сессионное/ревью под переезд на сервер. Сделана только косметика комментариев; логика/архитектура не менялись. Ветка `chore/m1-review-comments` (push не делался).

## Архитектура / переиспользуемость (что хорошо)
- **A1.** `document_processing` полностью client-agnostic (grep по zu/Зудема/Залог/КЦ в движке — пусто; зависимость однонаправленная `rag → docproc` через порт).
- **A2.** Чистое разделение портов/адаптеров: Embedder (fake/gateway), StoreRouter→QdrantStore, docproc (inproc/http), ScopeResolver (mock→projecthub), Presigner (mock→Beget). Замена реального = новый адаптер за тем же портом.
- **A3.** Конфиг весь из env; переезд local→стенд = смена `.env`; секреты только из env.

## Relations
- part_of [[01-Sessions]]
- relates_to [[Сборка document_processing (шаги 0–7)]]
- relates_to [[ADR-0001 — Демо-стенд RAG ЗУ (Codex)]]
- relates_to [[glossary]]