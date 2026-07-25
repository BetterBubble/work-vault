---
title: RAG#2 pipeline выгрузки+ингеста — канон (для реализации)
type: note
permalink: tacticum/20-architecture/rag-2-pipeline-vygruzki-ingesta-kanon-dlia-realizatsii
tags:
- rag2
- ingest
- pipeline
- extractor
- validation
- freshness
- canon
- helm
---

Канонический дизайн пайплайна выгрузки Jira/Confluence → ингест RAG#2. Решения приняты 2026-07-15 (с руководителем). Пилот доказан — см. [[RAG#2 выгрузка данных — пилот пройден (Confluence) + план полной]].

## Размещение (adp_emb чистый!)
- **adp_emb — ноль кода, ничего не оставляем.** Только SSH-форвард (как `iva-triva-tunnel`): `helm → adp_emb → 10.22.0.10:443` (jira.iva.ru и wiki.iva.ru — один IP, выбор по Host/SNI).
- **Весь код — у нас в git**, гоняется на нашей стороне (helm тянет; helm мал 2CPU/3.8ГБ/17ГБ свободно → стримить/батчить, не хранить весь сырой дамп).
- Доступ: Confluence — Bearer из `/var/lib/tacticum/env/confluence.env` на adp_emb; **Jira — Basic-auth `monakhov-tech:<EVA-пароль>`** (группа browse-allprojects, видит всё). Токен/пароль — только через туннель/env, НЕ в git.

## Скорости (замерено на пилоте)
Confluence ~1.42 стр/с; Jira ~2.75 задачи/с (полные: desc+комменты+changelog+связи+вложения). Полная выгрузка ~8–9 ч аккуратно, resumable, ночью.

## Стадии (в git, каждая с проверкой)
1. **Catalog** — все пространства/проекты → manifest (in/out + счётчики источника). Основа reconcile и дельт.
2. **Extract + schema-validate** — каждая запись через строгую pydantic-схему; кривые → reject/flag; обязательные поля enforce. Формат под helm-ингест: Jira `{k,sum,desc,com,ch,links,att,+payload}`; Confluence сырой REST `content` (`body.storage.value`, space, version, ancestors, labels, attachments).
3. **Reconcile** — `extracted==source` по каждому контейнеру; расхождение → жёсткий фейл.
4. **Вложения** — helm УЖЕ умеет `xlsx/csv/html/txt` (`rag2/extractors.py`). Добавить **только pdf/docx** (2 либы `pypdf`+`python-docx`, там TODO). **OCR/картинки/pptx/rtf — НЕ нужны** (низкая ценность, тяжело). `document_processing` НЕ форкаем.
5. **Quality/freshness-фильтр** — hard-exclude: personal/HR/🔴/archive, пустые/заглушки, deprecated/draft. Свежесть: не менялось >X лет → exclude или мягкий тег `stale`. Dedup near-dup (свежую версию). Обогащаем payload (`updated/stale/importance/status/views`) → реранкер топит актуальное. Jira старые closed НЕ выкидываем (история для RAG#3), тегируем по свежести.
6. **Chunk+embed+upsert → STAGING-коллекция** (не прод). Детерминир. UID (`tenant:key:idx`) → идемпотентно/resumable.
7. **Validate staging** — счётчики точек; sample-retrieval; **golden-eval гейт** (recall@k/MRR/nDCG не хуже baseline).
8. **Promote** — swap alias staging→prod только при прошедшем eval. В живой индекс напрямую — никогда.
9. **Manifest+дельта** — items.jsonl (id/version/updated/hash) → следующий прогон diff new/changed/removed; инкрементальный синк по `updated>last_run`.

## Фазы
- **Ф1:** тексты страниц/задач + уже поддержанные xlsx/csv/html вложения. helm готов, doc-processing не нужен.
- **Ф2:** pdf/docx (2 либы в `extractors.py`).
- Каждая фаза → staging → eval-гейт → promote.

## Полнота (максимальная)
Jira: desc+все комменты+весь changelog+все связи+вложения+метадата. Confluence: полное тело+иерархия+labels+вложения. Пилот: helm `build_document`/`load_pages` собирают всё (Комментарии/История/Связи в тексте; 51/51 стр→86 чанков).

Реализация — воркером, ветка в helm-репо, чистые функции+тесты, пилот-данные как фикстуры. ADR не пишем (решение руководителя).

## Финальный список исключений Confluence (решено 2026-07-15)
**Исключаем:** все personal (111); HR/люди (HRHITECH, HR1, birthday, zhuravlev); чувствительное/коммерция/право (IS, Legal, LEGALIVA, IVACD, KEYCLT, KEYCLT1, integrator, SSFYL); архив (ArchiveTPU); админ/хозчасть/закупки (IVAAHO, IVADELO, PUR/Purchasing, STCK/Stock, SRVRSKLAD, MMNTC, NTCPROD, Licenses, DOCFLOW).
**Берём (продуктовые/процессные/оргструктура):** IVACORE, IVAADP, IMP, IM, IVATerra, IVAUC, SBC, SS, PHON, IVAMP, IOA, IVAQA, IVACODEC, IVADS, IVAEDU, PMO, IVAPROJECT, пресейл (IPS/PSL/PI/PRSLHT), разработка (NTD/TECHDEP), FAQ, SD, **+ оргструктура (org, TPUORG, IVADIGIDAL, ITPAO, ITIVA — руководитель: нужна)**.
Список — стартовый фильтр; на несомненные HR/личное/чувствительное — hard-exclude; спорное можно тегировать и решать по eval.

## Грант triva (решено)
`tacticum/triva` helm-ключу (`control-tower`) **НЕ нужен** — оставляем грант только у `tacticum-agents`. rerank раздан 8 ключам (вкл. helm) — ок.

## Правка формата changelog (2026-07-15, по вопросу воркера)
Прод-`_render_changelog` в `infrastructure/rag2/ingest.py` читает **плоские** записи, а не вложенные `items[]`. Поэтому экстрактор выдаёт `ch` как плоский список **`{field, from, to, author, created}`** (одна запись на смену поля, author/created денормализованы) — чтобы `build_document` реально рендерил значения changelog. НЕ `{author,created,items:[...]}`. Обязателен тест: `ch` через `_render_changelog`/`build_document` → в тексте есть сами значения (не только заголовок). ingest.py не трогаем.

## Многократная валидация ПОЛНОТЫ данных (жёсткий гейт — приоритет руководителя 2026-07-15)
Весь RAG стоит на полноте данных → валидируем полноту НЕСКОЛЬКО РАЗ, на каждом слое; расхождение → hard-fail/алерт, НЕ молча:
1. **Каталог vs источник:** source count (страниц/задач по каждому space/project) == manifest — ДО выгрузки.
2. **Извлечено vs источник:** extracted == source по каждому контейнеру (0 потерь) — ПОСЛЕ выгрузки.
3. **Проиндексировано vs извлечено:** каждый item → ≥1 точка в Qdrant; сверка счётчиков (staging) — ПОСЛЕ ингеста.
4. **Дельта-повтор:** повторный diff источника → 0 new/removed (стабильно) — подтверждает, что ничего не пропущено.
5. **Сэмпл-аудит + известные важные:** случайная выборка + список заведомо ключевых страниц/задач → присутствуют в индексе И достаются ретривом.
6. **Вложения:** каждое извлечено ИЛИ явно помечено (skip/ошибка) — ничего молча не потеряно.
Каждый слой логируется числами; promote в прод — только когда все 6 сходятся + golden-eval прошёл. Пробелы фиксируем (что отложено), не выдаём частичное за полное.

## Порционный/инкрементальный налив (требование руководителя 2026-07-15)
Данные должны становиться доступны ПОРЦИЯМИ — часть usable уже через час-два, не после полной выгрузки.
- Обработка **контейнер-за-контейнером** по **приоритету** (`--priority-file`): сначала продуктовые спейсы (IVACORE/IM/IMP/IVATerra/IVAUC/SBC/IVAQA/IVADS/IVAADP/IOA/FAQ/SD) + ключевые Jira (IVAONE/VCSWEB/VCSMOB/IMP/IVATR/IVAUC2). Каждый: extract→валидация→**upsert-ингест**→валидация→сразу в чате/MCP.
- **Первичный налив — напрямую в целевую коллекцию** (идемпотентные UID, аддитивно). Дисциплина staging→eval→promote — для будущих полных ре-ингестов/дельт, НЕ для первичного порционного налива.
- Валидация полноты — **per-контейнер + накопительно**, финальная полная — после всех.
- Resume по контейнерам. C4/арх и так в БД → impact-анализ доступен сразу, знания доливаются порциями.
