---
title: explore-r5-changelog-recon-raw
type: note
permalink: tacticum/91-archive/inbox/explore-r5-changelog-recon-raw
tags:
- helm
- rag2
- explore
- raw
status: archived
updated: 2026-07-18
---

# explore: Р-5 changelog реконструкция — сырьё

Черновик разведки (2026-07-17). Каноническая заметка: [[rag2-r5-rebuild-and-datamap]].

## Raw evidence
- `iva_jira__bge_m3_1024` payload keys: tenant_id, source_doc_id, **key**, title, project, epic, ..., has_changelog, **text**, **chunk_idx**. `key`==`source_doc_id`.
- Перекрытие соседних чанков = **150 симв** (константа) → дедуп по (ts,from,to) обязателен и достаточен.
- Прогон `_status_durations` на реконструкции:
  - IVAONE-1175: chunks=3, uniq_transitions=21 → active_days=20.9, closed_at=2024-04-16T17:03:29.877+0300
  - IVAONE-3803: chunks=11, uniq=22 → active_days=7.3, closed_at=2025-02-24T16:53:12.994+0300
  - IVAONE-1: chunks=22, uniq=3 → active_days=0.0 (открыта)
- БД prod `epic_task`: count=8073, non-empty changelog=**377**, max as_of=2026-07-10. Формат: `[{"f":"Open","t":"Planned","d":"2023-10-16"}]` — **date-only**.
- Qdrant counts: iva_jira 319303 · iva_confluence 92374 · knowledge 80274 · iva_docs 8272 · helm_requirements 1465 · helm_mgmt 400.

## Скрипт реконструкции (проверенный, запускать на helm)
Логика: scroll по key → sort chunk_idx → join `"\n"` → regex `\[(ts)\]\s*status:\s*(.+?)\s*[→>]\s*([^\n\[]+)` → dedup (ts,from,to) → `{k, chlog:[{f,t,d=full_ts}]}` в tasks_changelog_v2/. Бакеты/маппинг статусов — из `helm.ingest.velocity` (_DEVELOPMENT/_CLOSED/_TESTING, canonical_status). Полная версия отработала как /tmp/r5_recon.py (удалён после прогона).

## Ключевые файлы кода
- `src/helm/interface/mcp/analyst_server.py`: `_status_durations`:681, `_parse_ts`:659, `_normalize_epic_changelog`:736, `effort_hint`:943.
- `src/helm/ingest/epic_tasks.py`: `load_epic_tasks`:51 (пишет ТОЛЬКО epic_task).
- `scripts/ingest_epic_tasks.py`: `_read_dirs` мёрж по k; `chlog`→`changelog`.
- `src/helm/ingest/velocity.py`: `status_bucket`:75, `canonical_status`:65, корзины :55-58.
