---
title: RAG#1 ИВА — корпус iva.ru/docs + golden-set готовы
type: note
permalink: tacticum/04-sessions/rag-1-iva-korpus-iva.ru-docs-golden-set-gotovy
tags:
- iva
- rag
- rag1
- golden-set
- корпус
- документация
- checkpoint
---

## Итог (2026-07-13)
Для **RAG#1** (публичная документация ИВА для support/техписов) собраны и провалидированы **корпус + golden-set**. Делалось командой агентов под тимлидом (воркеры `crawler`, `golden`), финализация — детерминированно тимлидом.

**Каталог:** `/Users/bubblemac/tacticum/iva-rag1-docs/`

## Корпус
- Источник `https://iva.ru/docs/` (Antora, статический). Выкачан curl'ом, 0 пропусков sitemap.
- **2138 doc-страниц** (`kind=doc`) — 16 продуктов (MCU 1286, IVA One, CS, Terra, SBC, Room, Mail) + release-notes/changelog.
- **7 навигационных индексов** вынесены в `md/_peripheral/` (исключены). Scope чистый: только `/docs/`, без маркетинга.
- Каждый `.md` с YAML-frontmatter: title/url/product/section/breadcrumb/version/slug/word_count/kind.
- Всего 2145 стр; **актуальных `version=latest` = 796** (остальные 1348 — версионные дубли, отдельный слой/фильтр).
- **Индексировать: срез `kind=doc` + `version=latest` (≈796 стр).**
- Артефакты: `manifest.json`, `REPORT.md`, воспроизводимый пайплайн `download.py`/`convert.py`.

## Golden-set (`golden/`)
- **`golden_iva_rag1.json`** — **1306 кейсов, покрытие 100% (796/796)**, 1306 уник. id, 0 нарушений схемы.
- **`known_gaps.json`** — 32 антипримера (expected=no_answer: цены/лицензии/чего нет в доках).
- Схема кейса (обогащена под обновляемость): `id`=`<slug>#<n>` (стабилен при перекрауле), tenant_id, query, relevant_doc_ids=[slug], product, section, source_url, qtype, role, difficulty, corpus_version, source, tags.
- Распределение: роли admin 683/end_user 356/integrator 120/support 77/tech_writer 37/developer 33; сложность typical 773/hard 367/edge 166; qtype how_to/factual/concept/config_setup/troubleshooting/ui_navigation/integration_api.
- **`golden_iva_rag1.meta.json`** (версия/покрытие/статистика), **`GOLDEN_REPORT.md`**, **`README_GOLDEN.md`** (регламент обновления).
- Формат совместим с eval-harness codex (recall@k/MRR/nDCG, срезы по product/role/qtype/difficulty).

## Ограничения / что помнить
- Golden **синтетический** (LLM-генерация по контенту страниц). Спот-чек 10/10 хорош, но перед боевым eval нужна человеческая выверка выборки (100–200 кейсов). См. [[verify-data-credibility]].
- В корпусе есть **near-duplicate страницы** (`*-introduction` vs `*-introduction-pdf`, глоссарии iOS/WEB, `qsg.*` vs полные) — трудноразличимы для retrieval; свойство исходных данных.

## Дальше
Как поднимем движок RAG#1 (форк codex) — индексируем срез latest-doc и гоняем eval на этом golden.

## Отношения
- part_of [[Концепт: три RAG для ИВА на общем движке]]
- relates_to [[verify-data-credibility]]
- relates_to [[dont-duplicate-agent-work]]
