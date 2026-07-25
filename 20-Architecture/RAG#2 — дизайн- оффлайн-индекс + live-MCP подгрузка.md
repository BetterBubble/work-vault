---
title: 'RAG#2 — дизайн: оффлайн-индекс + live-MCP подгрузка'
type: note
permalink: tacticum/20-architecture/rag-2-dizain-offlain-indeks-live-mcp-podgruzka
tags:
- iva
- rag
- rag2
- wiki
- jira
- confluence
- live-mcp
- temporal
- дизайн
---

## Суть
RAG#2 (Wiki/Jira/EVA для аналитиков) — данные **живые и меняются**, поэтому чистый статичный индекс неполон. Дизайн = **гибрид: оффлайн-индекс (broad recall) + точечная live-MCP подгрузка актуального по запросу** (требование пользователя 2026-07-13).

## Что уже выкачано (НЕ дублировать; свежесть 10.07, 3 дня)
- `helm/data/iva/tasks_rich/` — 55 JSONL-шардов, ~8119 задач rich (IVAONE+IVAONEHALF), поля summary/desc(89%)/status/assignee(99%)/components/epic/sprint/fixVersions/labels/parent/links(65%)/даты. ~1.55M ток на выгрузку.
- `tasks_changelog/` — история статусов+комменты по 379 in-progress.
- `wiki_reqs_10/15.json` — таблицы требований One 1.0/1.5 из Confluence (не тела страниц).
- `epics.json`(67), `req_realization.json`, `req_jira_status.json`, `signal_active_salvage.json` и др.
- helm-сервер `/opt/helm/data/real/jira/`: CSV-дампы, **`jira_issue_links.csv` = 5512 связей (граф зависимостей!)**. `confluence/` — всего 6 файлов.
- **ГЛАВНЫЙ ПРОБЕЛ: тела Confluence-страниц почти не выгружены** → добрать аккуратно тем же range-методом.

## Метод эффективной выгрузки (отлажен, `helm/scripts/wf_extract_jira_mcp.js`)
- MCP отдаёт данные только через контекст модели (1× на приём неустраним). Bulk «в лоб» = тупик.
- Экономия: явный `fields` (НИКОГДА `*all` ~8k ток/задача), range-sharding по стабильному `ORDER BY key` + непересекающиеся окна start_at (limit MCP=50), агент пишет компактный JSONL-шард 1 раз (в ответ — счётчики), дедуп по key на ингесте, бюджет-предохранитель с nextStartAt. Для Confluence — `confluence_get_space_page_tree`→пагинация→`confluence_get_page(convert_to_markdown=true)` шардами.

## Дизайн потока (роутер index↔live)
1. Всегда: эмбеддинг-поиск по оффлайн-индексу → кандидаты + их ID (issue key, page_id).
2. Роутер решает нужна ли свежесть: триггеры «сейчас/текущий статус/за неделю/последние изменения», конкретный ключ, темпоральные вопросы → live; аналитические «про что» → только индекс.
3. Live-fetch **точечно по ID из шага 1** (не широкий JQL-скан): Jira `jira_get_issue`/`jira_get_issue_dates`/`jira_batch_get_changelogs`/`jira_get_issue_links`; Confluence `confluence_get_page`/`_history`/`_diff`.
4. Merge: свежее из MCP перекрывает устаревшие поля индекса по key; бейдж `as_of` на каждый кусок.
5. Ответ + цитаты со ссылками на Jira/Confluence.

## Риски и гашение
- Токен-стоимость live: только точечно по ID, N≤~10, явные fields. Confluence тела — markdown+лимит страниц.
- Лаг/латентность MCP: индекс отвечает всегда; live с таймаутом + graceful degradation (не дождались → по индексу + дисклеймер as_of).
- Rate limit: кэш live на короткий TTL по key; батч changelog.
- Дрейф индекса: ночной инкремент (JQL `updated>=-1d`, Confluence lastModified) тем же range-скриптом.

## Варианты глубины (из концепта)
- **2a Hybrid** (dense+sparse+reranker) — ядро A, рекомендуется.
- **2b Структурный граф Jira** — поверх `jira_issue_links` (5512 связей уже есть): зависимости/блокеры/критпуть. Отдельный graph-retrieval, сливается с A.
- **2c LightRAG** — стенд на zu_demo; мерить на реляционных wiki/jira (вывод ЗУ про проигрыш графа НЕ переносить вслепую).

## Golden RAG#2
`helm/data/competency-questions.md` (30+ вопросов A–E, режимы D/G/R) пригоден ЧАСТИЧНО (заточен под helm-дэш шире RAG#2). Нужен профильный подкорпус: вопросы на тела Confluence («почему решение X»), темпоральные (page_diff/changelog), EVA (после eva-mcp) + флаг каждого вопроса **index-only / live-MCP / hybrid** (валидирует роутер).

## Отношения
- part_of [[Концепт: три RAG для ИВА на общем движке]]
- relates_to [[dont-duplicate-agent-work]]

## Смета выгрузки RAG#2 (токены, разведка 2026-07-13)
Эталон: Jira rich ≈ **190 ток/задача**; Confluence ~2.5 симв/токен.

**Jira counts (полный корпус, JQL total):** IVAONE 11641 + IVAONEHALF 474 = 12115 (уже есть срез 8119). IVAUC2 847+UCP 179=1026. VCS-семейство (VCSMOB/WEB2/WEB/DESK/ROOM/MASH/ANDR/OP) = **39933**. Инфра+прочее ~6800.

**Confluence — ГЛАВНЫЙ ПРОБЕЛ (тела почти не выгружены):**
- ⚠️ `confluence_get_space_page_tree` СЛОМАН на больших spaces (баг сортировки position str/int) — точные counts недоступны, оценки диапазонами. Перед выгрузкой снять перечень обходно (get_page_children рекурсивно / REST на helm).
- Размеры бимодальны: гиганты (требования/release notes с таблицами) 130–198k симв = **50–80k ток/стр**; типовые ~760 ток; медиана ~1900 симв.
- Spaces: **IVAPROJECT** (требования One 1.0/1.5/3.0, миграция MCU→One, архитектура — золото) ~1.3–1.6M ток; IM/IVCS/techwriters/IVAADP суммарно ещё ~1–2M.

**Итог-смета:**
- (а) уже есть Jira 8119 → 0.
- (б) добрать ядро СЕЙЧАС: **Confluence IVAPROJECT + инкремент свежести флагманов + IVAUC2/UCP ≈ 1.5–1.8M ток**.
- (б+) расширенное: + Confluence IM/IVCS/techwriters/IVAADP → **~3–4M суммарно**.
- (в) EVA (6634 зад., после eva-mcp) ~1.26M.
- НЕ рекомендую: полный бэкфилл старых закрытых IVAONE (~760k) и всё VCS-семейство Jira (~7.6M) — пока scope не подтверждён.

**Рекомендация:** инкрементально, подмножеством. Приоритет №1 — тела Confluence IVAPROJECT (закрывает главный пробел).

**Открытый вопрос к руководителю:** scope RAG#2 = только IVA One или **весь продуктовый портфель** (VCS-семейство +7.6M ток)?

**Вложения Confluence (xlsx/docx/pdf с требованиями) — ОБРАБАТЫВАЕМЫ**, не за бортом: `iva-mcp` (confluence_get_attachments / download_attachment) → `document_processing` extract (docx/xlsx/pdf/pptx+OCR, есть в форке codex) → в индекс. Пайплайн Confluence = тело страницы (markdown) + вложения через docproc. Уточнить при выгрузке: механику download (файл на диск = дёшево по токенам vs через контекст) и объём вложений.

**Риски оценки:** баг tree (counts грубые), char→token ±30%, объём вложений пока не посчитан.

## Уточнение по контуру (2026-07-13, от пользователя)
Данные Jira/Confluence ИВА **НЕ строго on-prem** — их уже брали не on-prem для helm. Значит **RAG#2 индекс может жить у нас (общий сервер)**, как RAG#1; on-prem строго обязателен только для **RAG#3** (чаты/почты/требования заказчиков, ADR-0003). Упрощает RAG#2: не требует выделенного хоста в контуре ИВА для индекса; live-MCP всё равно ходит в ИВА через существующий iva-mcp-туннель.
