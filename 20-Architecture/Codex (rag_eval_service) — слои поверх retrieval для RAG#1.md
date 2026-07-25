---
title: Codex (rag_eval_service) — слои поверх retrieval для RAG#1
type: note
permalink: tacticum/20-architecture/codex-rag-eval-service-sloi-poverkh-retrieval-dlia-rag-1
tags:
- iva
- rag
- rag1
- codex
- rag-eval-service
- bff
- langfuse
- frontend
---

## Факт
В `rag_eval_service` (codex, RAG ЗУ) всё «поверх голого retrieval» уже собрано и **живёт в проде на zu_demo** с реальными адаптерами (не моки): `/health` → scope=ProjectHubScopeResolver, embed=gateway, langfuse=on, history_db=on, presigner=beget; фронт `zu-demo.cifragen.ru` = 200. Наш RAG#1-прототип (`iva-rag1-engine`) — пока только retrieval+eval; эти слои надо будет **прикрутить, переиспользуя codex как донор** (не писать с нуля). Заложено в концепте [[Концепт: три RAG для ИВА на общем движке]] §3.

## Компоненты (донор для RAG#1)
- **Ask-BFF** (`bff/app.py`, ~250 стр): FastAPI `/ask` scope→search→LLM(Gateway, стрим NDJSON)→{answer,citations}. Почти прямой донор.
- **Langfuse** (`bff/telemetry.py`, ~130 стр): трейс на /ask, токены+USD-cost, score-фидбэк. Langfuse — **внешний общий** `langfuse.cifragen.ru` (не контейнер), для RAG#1 = отдельный проект.
- **Фронт** (`frontend/`, Next.js): чат `AskPanel.tsx`, OIDC-сессия server-side, httpOnly `phk_`, прокси в codex. Бренд zu.ru → переверстать под ИВА.
- **OIDC/PKCE** (`frontend/lib/oidc.ts` + `bff/scope.py` ProjectHubScopeResolver): project-hub. Включается env; зависит от IdP-контура RAG#1.
- **История чатов** (`bff/history.py`): Postgres `ask_history`, scope по user_id. Самодостаточный.
- **Фидбэк** 👍/👎: в историю + Langfuse-score. Идёт с BFF+telemetry.
- **Presigned** (`bff/citations.py`): порт Presigner, beget-S3. Для ИВА — **адаптер под Confluence-ссылки** (не S3).
- **document_processing**: extract docx/xlsx/pdf+OCR (для вложений).
- **eval-harness** (`eval/`): recall@k/MRR/nDCG. Для RAG#1 golden-set **уже готов** (1306 кейсов, см. [[RAG#1 ИВА — корпус iva.ru/docs + golden-set готовы]]).

## Git-состояние (на 2026-07-13)
- remote `github.com/TacticumApps/rag_eval_service`. Локальный `main` **отстаёт от origin на 3 коммита** (нужен pull перед работой). Незапушенного своего нет. `.env` gitignored; `.env.example` (100 стр) документирует все подключения.

## Приоритет прикручивания к RAG#1
1. Ask-BFF (ответ+цитаты) → 2. Фронт (чат+OIDC) → 3. Langfuse → 4. История → 5. Фидбэк → 6. Presigned (адаптер Confluence) → 7. eval (golden уже есть).

## Отношения
- part_of [[Концепт: три RAG для ИВА на общем движке]]
- relates_to [[RAG#1 ИВА — корпус iva.ru/docs + golden-set готовы]]
