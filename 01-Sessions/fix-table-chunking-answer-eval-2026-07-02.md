---
title: fix-table-chunking-answer-eval-2026-07-02
type: report
permalink: tacticum/01-sessions/fix-table-chunking-answer-eval-2026-07-02-1
tags:
- rag
- chunking
- eval
- answer-in-context
- table
- zu_demo
- fix
---

# Fix табличного чанкинга + вывод про метрику (answer-level, не doc-recall) — 2026-07-02

## Проблема (баг)
Кейс «Адрес отделения Перово»: retrieval находит документ `adresa-filialov-i-telefony` (ранг 1), но /ask отвечает «адрес не указан». Диагноз: **звено (b) — ретрив-гранулярность**. XLSX чанковался слепым char-slicing 900/150 (`rag/chunker.py`), извлечение теряло `row/col` в `rag/docproc_port.py`, `rag/ingest.py._group_segments` сплющивал лист в блоб. Итог: одна запись (Перово→адрес) тонула среди 6-8 отделений в чанке (ранг 21), а ранг 1 занимал 15-символьный чанк-заголовок «Адреса филиалов». Адрес НЕ попадал в top-5 чанков /ask. Извлечение при этом корректное (Перово+адрес в одной строке), таблица не рвётся.

## Фикс (ветка `fix/table-chunking`, origin, HEAD `fc23a6a`)
Запись-ориентированный табличный чанкинг в RAG-слое (ADR D4), проза 900/150 не тронута:
- `docproc_port.py`: пробрасывает `row/col/type` сегментов.
- `chunker.py`: `chunk_table_rows` + `rows_from_cells`/`table_prefix`/`is_tabular`/`doc_label`. Одна запись(и) на чанк (`TABLE_CHUNK_SIZE=160`) + **префикс заголовком листа/документа** = семантический мост (в ячейках нет слов «адрес/отделение»).
- `ingest.py`: табличный путь в `_build_chunks`.
- `config.py`: `TABLE_AWARE`/`TABLE_CHUNK_SIZE`/`TABLE_PREFIX_TITLE`.
- Тесты `rag/tests/test_table_chunking.py` (11, зелёные).
- Коммиты: `085fd0a` (чанкинг), `e63bbf7` (doc-диверсификация), `fc23a6a` (diversity дефолт OFF).

## ГЛАВНЫЙ ВЫВОД: doc-recall слеп к этому классу фиксов
- **Doc-level recall@k нельзя гейтить этот фикс.** Документ `adresa-filialov` и в baseline всегда ранг 1 (через чанк-заголовок), поэтому doc-recall=1.0 в обоих случаях — метрика не видит, дошёл ли адрес. Она лишь штрафует фикс за краудинг.
- **Правильная метрика — answer-in-context@k**: есть ли строка-ответ (реальный адрес/телефон) в top-5 чанках /ask (diversity OFF).
- Замер на 16 авто-сгенерированных адресных лукапах (ответ = улица+дом из xlsx): **baseline 5/16=0.31 → фикс 16/16=1.00**. Фикс решает целевой класс запросов.
- Скрипты замера на сервере: `/tmp/tablefix-src/answer_eval2.py` (авто-генерит адресные кейсы + answer-in-context), `answer_eval.py`.

## Числа (изолированное окружение, temp Qdrant-коллекции, боевое не тронуто)
- **answer-in-context@5 (то, что важно): baseline 5/16 → фикс 16/16** ✅
- doc-recall@5 (общий golden 35): baseline 0.9667 → фикс 0.9048 (+diversity 0.9476) — **метрика не про это**; регрессия = 4 кейса (q7,c3,c19,c30), где мелкие табличные чанки крауд-давят top-k, вытесняя НЕтабличные ответы (прайсы/svodnaya).
- Изоляция: старый чанкинг + новый algoritm-файл = 0.9667 → виноват **чанкинг**, не файл.
- **diversity=1 (дедуп до уник. доков) НЕ годится**: чинит doc-recall (0.948), но схлопывает `adresa` до чанка-заголовка → ломает answer-цель. Дефолт OFF, опция в env.

## Остаточный риск и следующий шаг
- Мелкие табличные чанки могут выталкивать НЕтабличные ответы (крауд). Правильное лечение — **cap чанков-на-документ (2-3)** в ретриве (не diversity=1): адресный док отдаёт 2-3 чанка (заголовок+адрес), крупный воркбук (algoritm 139 чанков) ограничен. Нужно доизмерить answer-level на НЕтабличных кейсах (прайсы, тарифы) + расширить answer-golden.
- **Прод-раскатка НЕ делалась.** Требует: cap-чанков, полный answer-golden (табличные+нетабличные), окно без клиентов, переиндекс боевой `knowledge__bge-m3_1024`.

## Состояние окружения
- Ветка `fix/table-chunking` @ `fc23a6a` в origin (TacticumApps/codex).
- Сервер: заменён `/root/codex/staging/algoritm-deistvii-ks.xlsx` (пользователем, 66346 б); дерево ветки в `/tmp/tablefix-src`; temp-коллекции Qdrant `knowledge_tablefix_tmp` (784) и `knowledge_noaware_tmp` (575); контейнеры `tablefix-ingest`/`tablefix-ingest2`; Meili-индексы `tablefix_tmp`/`noaware_tmp`. Боевая `knowledge__bge-m3_1024` (577) НЕ тронута, codex-1 не пересобирался.
- **Teardown temp-артефактов** (когда не нужны): снести temp Qdrant-коллекции + Meili-индексы + контейнеры + `/tmp/tablefix*`.

## Relations
- part_of [[01-Sessions]]
- relates_to [[server-zu-demo]]
- relates_to [[Runbook: прогон eval на сервере]]
- relates_to [[lightrag-server-eval-2026-07-02]]

## ПРОД-РАСКАТКА ВЫПОЛНЕНА (итог, 2026-07-02)
Раскатано на боевую (окно без клиентов, по решению пользователя):
- **Бэкап**: Qdrant-снапшот `knowledge__bge-m3_1024-7181624656111461-2026-07-02-11-27-05.snapshot` (оставлен для отката).
- **Переиндексация боевой `knowledge__bge-m3_1024`**: удалены cifragen-точки + Meili очищен → переиндекс всех 70 staging-доков (вкл. новый algoritm) fix-кодом (table_aware, size=160). Итог: **784 точки, tenant cifragen** (единственный, fail-closed).
- **codex-1 НЕ пересобирался** — фикс на стороне данных; поиск тот же. Раскатка = только данные.
- **Верификация retrieval** (та же `rag.search`, что в codex-1, diversity=false как в старом образе): Перово-чанк **ранг 3 (0.5934), адрес в top-5**; answer-in-context@5 **20/22 = 0.91** (адреса 15/16, проза 5/6). Baseline был 9/22.
- **/ask через codex-1**: использует РЕАЛЬНЫЙ project-hub auth (SCOPE_RESOLVER=projecthub), нужен operator-токен `phk_` → живой /ask проверяет пользователь из демо-UI. Retrieval-путь идентичен проверенному.
- **ВАЖНЫЙ КАВЕАТ**: diversity дефолт выставлен OFF (`fc23a6a`) — diversity=1 схлопывает адресный док до заголовка и ЛОМАЕТ фикс. Если ветку смёржат и codex-1 пересоберут — дефолт OFF сохраняет корректность. НЕ включать SEARCH_DOC_DIVERSITY на этих данных.
- **Teardown temp**: снесены temp Qdrant-коллекции (knowledge_tablefix_tmp/noaware_tmp), temp Meili, контейнеры (age-builder/tablefix-ingest*/prod), /tmp/tablefix*. Бэкап-снапшот оставлен.

## Осталось
- Живой /ask из демо-UI (токен у пользователя).
- Отдельно: **LightRAG-teardown НЕ сделан** (lightrag_vdb_* коллекции + AGE-граф в codex + LIGHTRAG_* таблицы + lightrag-eval контейнер + age extension) — по прежнему решению отложен, ранбук в [[lightrag-server-eval-2026-07-02]].
- Мердж ветки `fix/table-chunking` в main — осознанный шаг пользователя (PR).

## Пост-фикс: русские title в цитатах (2026-07-02)
Симптом: в UI источники показывали слаг (`adresa-filialov-i-telefony`) вместо русского имени. Причина: driver `ingest_all.py` НЕ передавал `title` → title дефолтил в имя staging-файла (транслит-слаг), а раньше был русский оригинал.
Исправлено: обновил payload `title` во всех 70 доках боевой `knowledge__bge-m3_1024` по карте **`/root/codex/staging/mapping.tsv`** (col1=слаг-файл, col2=русское имя). storage_uri оставлен слагом (ключ S3). Проверка: 784/784 чанка с кириллицей в title.
**КАВЕАТ на будущее:** при любой переиндексации staging передавать `title` из mapping.tsv (col2), иначе слаги вернутся. staging-файлы названы транслитом, mapping.tsv — источник русских имён.