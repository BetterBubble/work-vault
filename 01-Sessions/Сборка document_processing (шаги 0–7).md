---
title: Сборка document_processing (шаги 0–7)
type: note
permalink: tacticum/01-sessions/sborka-document-processing-shagi-0-7-1
tags:
- session
- document-processing
- codex
- build
- ocr
---

# Сборка сервиса document_processing (готово, шаги 0–7)

**Источник на диске:** `~/tacticum/fast_task/06-service-build-summary.md`
**Тип:** лог сборки. Репо `rag_eval_service`, ветка `feature/document-processing`, сервис в подпапке `document_processing/`. Донор-репы (doc_translator/agents) read-only, не тронуты.

## Что сделано
- `.gitignore` первым коммитом (секреты + данные клиента). 22 файла экстракторов doc_translator + schema/exceptions/ocr_engine, импорты переписаны на `doc_extract`. settings_shim (только флаги OCR, `PDF_PADDLE_OCR_FALLBACK_ENABLED=False`, Tesseract rus+eng).
- `service.extract()` — диспетчер по расширению + generic-фильтр входа + OCR изображений. Client-agnostic.
- FastAPI: `POST /extract` (multipart → list[TextSegment]), `GET /health`. Без БД/auth/RBAC. Dockerfile python:3.12-slim + tesseract (без paddle/LibreOffice).

## Решения по ходу
- **Аудио = вариант B:** сервис только документы+изображения; аудио/видео не маршрутизируются, транскрипция — внешний шаг (transcription MCP на `project`). Шов открыт: `TextSegment(extraction_source="transcript")`.
- **Client-agnostic:** никакой специфики ЗУ/КЦ в `extract()`.

## Relations
- part_of [[01-Sessions]]
- relates_to [[Code review M1]]
- relates_to [[ADR-0001 — Демо-стенд RAG ЗУ (Codex)]]
- relates_to [[glossary]]