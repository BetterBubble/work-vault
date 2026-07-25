---
title: План изменений по репозиториям (Phase 0-1-2)
type: note
permalink: tacticum/20-architecture/plan-izmenenii-po-repozitoriiam-phase-0-1-2-1
tags:
- architecture
- rag
- plan
- migration
- knowledge
---

# CHANGE_plan — что и где менять по репозиториям (Phase 0/1/2)

**Источник на диске:** `~/tacticum/_analysis/CHANGE_plan-2.md`
**Тип:** план/анализ. Приведён в соответствие с ADR-0005, ADR-0043 и TARGET v2.3.

## Фазы
- **Phase 0 — измеримость + фикс изоляции (разблокировка).** Снять состояние прода (dim коллекций `agents`, наличие `tenant_id` в payload); собрать golden-set + поднять локальный стенд (Qdrant+Meili+bge-m3); метрики recall@k/MRR/nDCG + baseline; параметризовать Meili-фильтр (баг инъекции); **падающий red-тест изоляции** (не добавляя ломающий фильтр вектора). **Гейт G0** = baseline зафиксирован + Meili-фикс + red-тест изоляции (прямой Acceptance #32).
- **Phase 1 — платформенный `knowledge` + контракт + cutover `agents`.** Каркас owned-сервиса `platform/services/data/knowledge_rag/` (domain/application/infrastructure/interface); контракт `knowledge.{search,ingest,list_collections}` в `platform/contracts/` (без `/doc/extract`); клиент в SDK; `doc.extract` — в `document_processing` (adapter). Развести с ADR-0004 (extract из agents vs adopt arch-mcp).
- **Phase 2** — governance (`pattern_search`, ADR-0043), отдельный гейт G-gov, не блокирует G1.

## Ключевое из ADR
ADR-0005: bge-m3/1024 везде (D6); tenant_id обязателен (D7); doc.extract вне knowledge (D2); контракт search/ingest/list (D3); SDK+AuthScope, API Hub не сейчас (D5); StoreRouter порт (D4); утечка вектора латентна (D7). ADR-0043: 4-я операция `pattern_search` (governance-retrieval); Q-9 (Codex/ADR) переопределён — это governance, не «4-й graph-источник».

## Relations
- part_of [[20-Architecture]]
- relates_to [[TARGET — целевая архитектура knowledge v2.3]]
- relates_to [[Аудит текущего RAG по флоту]]
- relates_to [[Водораздел: платформенный knowledge vs Codex]]
- relates_to [[glossary]]