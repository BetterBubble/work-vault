---
title: RAG#1 морда собрана по коду (iva-rag)
type: note
permalink: tacticum/01-sessions/rag-1-morda-sobrana-po-kodu-iva-rag
tags:
- iva
- rag
- rag1
- checkpoint
- backend
- frontend
- iva-rag
---

## Итог (2026-07-13)
RAG#1 (чат по документации ИВА) собран **целиком по коду** — форк codex + улучшенный фронт под бренд ИВА. Крутиться будет на сервере (пока не выбран); локально не запускался. Всё вне git, в `/Users/bubblemac/tacticum/iva-rag/`.

## Backend (`iva-rag/`, форк rag_eval_service)
- Ingest .md минуя docproc (`rag/md_ingest.py`), структурный чанкер heading-path (`rag/md_chunker.py`), богатые метаданные (весь frontmatter в payload). Dry-run: 796 стр → 8272 чанка.
- **Reranker включён** (`rag/reranker.py`, порт gateway `tacticum/rerank` | local bge-reranker | none), candidate_k 20→60. Hybrid+reranker дефолтом. fail-closed изоляция сохранена.
- Grounding (`bff/grounding.py`): «только из контекста, иначе не нашёл» + гейт known_gaps.
- Фильтры {product,section,version} в Qdrant+Meili + `GET /facets`. Cap чанков/документ (MAX_CHUNKS_PER_DOC=2).
- BFF `POST /ask` (NDJSON, deep-link цитаты url=iva.ru/docs, S3 выпилен), история+фидбэк, Langfuse env-gated (проект iva-rag).
- **47 тестов зелёные** (моки): chunker/RRF/citations/metrics/reranker/cap/filters/fail-closed изоляция/grounding/контракт. README_RUN.md для сервера.

## Frontend (`iva-rag/frontend/`, Next.js 15)
- Бренд ИВА (primary #21b175, cyan-glow, Inter, лепестки-глиф, светлая/тёмная тема). Токены в globals.css.
- UX vs ЗУ: цитаты сниппет+подсветка+якорь-URL+inline [1][2], фильтры из /facets, мобильная история (drawer), фидбэк, стриминг. OIDC-прокси (env).
- **`preview.html`** — статичное превью для ревью стиля (отправлено пользователю). tsc/next build зелёные. MOCK_MODE для демо без бэка.
- API-контракт согласован с backend (POST /ask, GET /facets, /history, feedback).

## Осталось (этап сервера)
- Поднять на сервере: Qdrant+Meili+эмбеддер (gateway/local) → `python -m rag.md_ingest` → `uvicorn bff.app:app`.
- Проверить формат gateway `/rerank` вживую (правится только rerank.py; при сбое деградация без падения).
- Откалибровать пороги цитат на golden после боевого прогона; прогнать eval (golden 1306 + known_gaps 32).
- Подключить eval-harness под ИВА (порт готов).
- Решить: репа (выделенная vs helm) + git, шрифт Inter self-hosted, ревью-правки фронта по фидбэку пользователя.

## Отношения
- part_of [[Концепт: три RAG для ИВА на общем движке]]
- relates_to [[RAG-1 ИВА — корпус iva.ru-docs + golden-set готовы]]
- relates_to [[RAG-1 — чеклист улучшений vs ЗУ (нюансы codex)]]
- relates_to [[Бренд-гайд ИВА для фронта RAG-1]]
- relates_to [[Codex (rag_eval_service) — слои поверх retrieval для RAG-1]]
