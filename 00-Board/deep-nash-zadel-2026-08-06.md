---
title: 'Разведка: наш собственный технологический задел под заявку ИНТЦ'
type: report
status: draft
tags:
- board
- инновации
- explore
permalink: tacticum/00-board/deep-nash-zadel-2026-08-06
---

# Наш задел: что реально работает и что ложится в заявку

Задача лида: инвентаризовать наш собственный технологический задел (не заказчика) под
заявку на льготу ИНТЦ по двум продуктам — AVES(АВИС)+Terra и IVA Largo+голосовой
ассистент+AI+Terra. Разведка read-only: код в `~/tacticum` + живые серверы по SSH.

## 0. Поправка к постановке (важно для дальнейшей работы)

Лид дал пути `src/helm/infrastructure/transcribe_runner.py` и т.д. — **в рабочей копии
`~/tacticum/helm` их нет**, потому что локальная ветка `main` отстала на неделю
(последний коммит 2026-07-29 02:55). В `origin/main` файлы есть и код смержен
(`git ls-tree origin/main` находит их; последний мерж origin/main — PR #116 от
2026-08-06 16:29). Всё ниже про helm читалось из `origin/main`, не из рабочей копии.

## 1. Цепочка transcribe → dialogue → summarize (helm) — САМОЕ СИЛЬНОЕ

Собственный паттерн Президента, **живёт в проде helm.tacticum.ru**.

**Архитектура** (`src/helm/infrastructure/transcribe_runner.py:1-24` — докстрока-канон):
три вида задач в общей очереди `bot_task`, которые берёт **отдельная полоса**
`helm-media-worker`, чтобы длинная медиа-работа не душила интерактивный интейк
PM-бота. Цепочка ставится сама: `transcribe` → `dialogue` → `summarize`.

| Шаг | Что делает | Ключевой код |
|---|---|---|
| `transcribe` | ffmpeg → чанки mp3 → Gateway (whisper) → склейка | `transcribe_runner.py:74-160` (`TranscribeRunner.run`) |
| `dialogue` | второй проход LLM: границы реплик + метки говорящих | `transcribe_runner.py:163-236` (`DialogueRunner`) |
| `summarize` | Codex-тред по промпту пресета → `summary` + `thread_id` | `transcribe_runner.py:239-287` (`SummarizeRunner`) |

**Инженерные решения, которые видно в коде** (это и есть «насколько хорошо»):

- **Чанкинг с перекрытием:** `CHUNK_SECONDS = 600`, `CHUNK_OVERLAP_SEC = 3`
  (`src/helm/domain/transcribe.py:43,46`), план нарезки — чистая функция
  `chunk_plan()` (`transcribe.py:131-148`), тестируемая без ffmpeg.
- **Зачем второй проход:** «у whisper через Gateway **диаризации нет**, а сплошной
  текст на 35 минут читать невозможно» (`transcribe_runner.py:11-13`). То есть
  разметка говорящих сделана **LLM-проходом поверх ASR** — это обход отсутствия
  диаризации у провайдера, и это переносимо на любой ASR.
- **Сквозная нумерация говорящих:** все куски диалога идут в **одном треде**
  (`transcribe_runner.py:206-208`), иначе «Спикер 1» во втором куске был бы другим
  человеком. Нарезка на куски — `split_for_dialogue`, `DIALOGUE_CHUNK_CHARS = 6000`
  (`transcribe.py:231-259`).
- **Разметка — необязательный шаг:** любая ошибка в `dialogue` не лишает
  пользователя саммари, задача `summarize` ставится **при любом исходе**
  (`transcribe_runner.py:224-229`, комментарий «Саммари ставим ВСЕГДА»). Дефект
  2026-08-01 «цепочка обрывалась на расшифровано» зафиксирован в комментарии
  `transcribe_runner.py:117-118`.
- **Саммари берёт размеченный диалог, если он есть:** «с границами реплик модель
  заметно точнее привязывает решения к людям» (`transcribe_runner.py:266-273`).
- **Приватность как инвариант, а не обещание:** исходник удаляется в `finally` при
  любом исходе (`transcribe_runner.py:154-160`) **плюс** `sweep_orphans()` на старте
  воркера подчищает следы прежних падений (`transcribe_runner.py:57-71`,
  `ORPHAN_SWEEP_HOURS = 6`). Ретенция 180 дней (`transcribe.py:61`).
- **Продуктовая обвязка:** пресеты суммаризации с CRUD и владельцем
  (`routers/transcribe.py:189-318`), пикер участников из справочника людей
  (`routers/transcribe.py:320-360`), загрузка до **1.5 ГБ** (`transcribe.py:39`,
  `MAX_UPLOAD_BYTES = 1536*1024*1024`), веб-экран 705 строк
  (`web/src/screens/TranscribeApp.tsx`) с прогрессом и оценкой остатка времени.
- **Модель:** `TRANSCRIBE_MODEL = "tacticum/transcribe"` — алиас гейтвея на
  whisper-large-v3 у DeepInfra (`src/helm/infrastructure/media.py:9,28`).
- **Тесты:** 12 домен + 6 раннеры + 12 API = **30 тестов** (`tests/domain/test_transcribe.py`,
  `tests/infrastructure/test_transcribe_runner.py`, `tests/interface/test_transcribe_api.py`).

**Доказательство работы в проде** (SSH `helm`, `helm-postgres-1`, 2026-08-06):
6 обращений, 5 `done` + 1 `purged`, последнее — **сегодня 10:28**. Пример строки:
запись **66 минут** (`duration_sec = 3967.12`) → transcript **46 389** символов →
dialogue **46 908** → summary **4 225**. Т.е. полная цепочка отработала на часовой
записи, не на демо-файле.

## 2. RAG-стек

### RAG-1 (докс-бот по публичной документации ИВА) — В ПРОДЕ, С ЗАМЕРАМИ

Ядро `DocsAssistant.ask` в helm, поверхности — бот «Поддержка» + веб `/docs`.
Задеплоен на helm.tacticum.ru (PR #81). Источник:
`20-Architecture/RAG-1 чат-бот — улучшения задеплоены (итог, замеры, решения).md`.

- **Замеры ретрива:** recall@5 **0.98**, mrr **0.92**, ndcg **0.93**,
  answer_in_context@5 0.86–0.879.
- **Замеры качества ответа (LLM-judge, 60 кейсов на golden-184):** correctness
  **0.796**, faithfulness **0.963**, relevance **0.963**. Это база «до»; «после» не
  снят (judge-гейтвей ~20с/кейс) — честно помечено в заметке как несделанное.
- **Латентность разложена по стадиям:** embed 63мс / Qdrant 22 / Meili 21 ≈ 106мс
  ретрив, реранк ~1.1с. **Узкое место — реранкер.** Кап кандидатов реранка = 20 дал
  **−40% времени реранка (1086→643мс) при hit@5 = 1.000**; ниже 20 роняет recall
  (12→0.96, 10→0.92). Это замер, а не оценка.
- **Уточняющие вопросы** реализованы (стор `docs_clarify_pending`, оркестратор
  `interface/api/docs_clarify.py`), триггер — решение LLM в генерации, **не порог
  скора**: замер показал, что реранк-скоры бимодальны (реальные ~1.0 / мусор ~0.0),
  полоса для порогового клэрифая пустая. Флаг по умолчанию **OFF**.
- **Golden-набор с эталонами:** 184 кейса, факты verbatim-заземлены,
  `iva-rag1-docs/golden/golden_iva_rag1.eval200.json`.

### RAG-2 (аналитик-MCP `helm-analyst`) — РАБОТАЕТ, ЕСТЬ A/B

Замеры: `20-Architecture/rag2-ab-measurements.md`, golden 45 кейсов (14 positive
размечено, 12 negative) на живом Qdrant `10.16.0.19`.

- Baseline recall@1/3/5/10 = **0.643 / 0.857 / 0.929 / 0.929**, MRR 0.732.
- **Дотяжка (exact-key retrieval) — главный фикс:** recall@1/5/10 на orig-9
  **0.000/0.000/0.111 → 0.889/0.889/0.889**, MRR 0.019→0.889. В проде, дефолт ON.
- Свип ft-weight: снижение **роняет** recall (1.0 → 0.929; 0.5 и 0.0 → 0.857) —
  гипотеза «мёртвый ft-канал вредит» проверена и **опровергнута** на полном наборе.
- Cross-rerank: recall@10 не меняет, recall@1 **хуже** (0.643→0.571) — флаг off.
- Калибровка порога отсечки: τ=0.7 режет 17% шума без потери recall.
- Тенант-изоляция держится (чужой/пустой tenant → 0).
- Набор тулов (из описания MCP): `analyst_search`, `analyst_context`, `arch_map`,
  `affected_systems`, `requirement_coverage`, `related_tasks`, `nearest_spec`,
  `who_to_involve`, `effort_hint`, `gap_questions`, `constraints`,
  `contradiction_check`, `api_registry_check`, `requirement_tests`.

### Движок RAG (`rag_eval_service`, он же «codex») — ЖИВОЙ, ДОНОР ДВИЖКА

`~/tacticum/rag_eval_service`: модули `rag/` (chunker, embedder, fulltext, fusion,
ingest, routing, search, store), `eval/` (runner, metrics, golden_sets, validate),
`document_processing/` (doc_extract). Живёт на zu_demo: контейнеры
`zu-deploy-codex-1`, `zu-deploy-qdrant-1`, `zu-deploy-meilisearch-1`,
`zu-operator-kb`, `lightrag-eval` — **все Up 5–6 недель** (SSH zu_demo, 2026-08-06).
Концепт «три RAG на общем движке» (`20-Architecture/Концепт- три RAG…`) прямо
называет его донором движка. Корпус ЗУ — 784 чанка, реранкер там выключен.

### `local-stack` — референс-стенд гибридного поиска

`~/tacticum/local-stack`: Qdrant + Meilisearch + pgvector + bge-m3 + bge-reranker-v2-m3,
**hybrid через RRF → rerank**, всё scoped по tenant (fail-closed), плюс eval-harness
(recall@k / mrr / ndcg) и **тест изоляции тенантов** (`eval/test_isolation.py`).
Чанкер 900/150. Запускается на fake-моделях за минуту без клиентских данных — это
готовый демо-стенд. Свежесть: последняя правка каталога 25 июня.

### `tei_service` — эмбеддинги как сервис

Sync+async доступ к embeddings через два провайдера: локальный HuggingFace TEI
(`gte-multilingual-base`) и внешний OpenAI-совместимый (`BAAI/bge-m3` дефолт).
Реестр моделей — `models.yaml`, Traefik + Grafana в compose, healthcheck.
Проверить живьём не смог: сервер `adp_emb` **не ответил на SSH** (таймаут
handshake) — это НЕ проверено.

## 3. `doc_translator` — платформа обработки документов

Полноценная платформа перевода RU-документов с **сохранением вёрстки**, async
(Celery), поиск, translation memory, human review, quality gates, runbooks.

- **14 экстракторов**: pdf, docx, xlsx, xls, pptx, ppt, doc, odt, rtf, epub, md, srt
  (+ конвертеры) — `backend/app/domains/processing/extractors/`.
- **12 реконструкторов** с диспетчером — `.../reconstructors/dispatcher.py`,
  docx/pdf/pptx/odt/rtf/md/plain_text.
- **OCR двухуровневый:** Tesseract основной, PaddleOCR как fallback
  (`backend/app/domains/processing/ocr_engine.py:19`,
  `extractors/pdf_extractor.py:2491-2498`).
- **LibreTranslate** в compose (`docker-compose.yml:73-75`) — офлайн-перевод в контуре.
- **Фиделити-гейт вёрстки** как тест: `backend/tests/unit/test_pdf_extractor_fidelity_gate.py`;
  всего **60 файлов unit-тестов** + Playwright E2E на фронте.
- Зрелость: есть `docker-compose.prod.yml`, runbooks (деплой, бэкап/восстановление,
  релиз, безопасность, фиделити). **Но** git-история схлопнута: единственный коммит
  «initial commit» от 2026-06-18 — историю разработки по репозиторию не восстановить.
  Живьём не проверял (не знаю адреса стенда) — **не проверено**.

## 4. `project-hub` — флот MCP-сервисов, ВСЕ ЖИВЫЕ

SSH `project_hub`, `docker ps` 2026-08-06 — аптаймы реальные, не «поднял и померло»:

| Сервис | Что это | Статус |
|---|---|---|
| `transcription-mcp` | project-scoped транскрипция аудио/видео | Up **7 недель** (healthy) |
| `transcription-worker` | отдельный воркер транскрипции | Up **7 недель** |
| `word-mcp` | обёртка docx-mcp (SSE/stdio) | Up 7 недель (healthy) |
| `excel-mcp` | обёртка mcp-excel-server | Up 7 недель (healthy) |
| `arch-mcp` | архитектурная память: **темпоральный граф знаний** ADR/PRD/RFC (Graphiti + FalkorDB), поиск конфликтов и дублей | Up 5 недель (healthy) |
| `wiki-mcp` | FastMCP поверх Wiki.js v2 GraphQL | Up 5 недель (healthy) |
| `taiga-mcp` | ~35 тулов поверх Taiga API | Up 3 недели (healthy) |
| `wiki-digest` | ежедневный дайджест изменений вики по тенантам, SMTP, cron | Up 3 дня (healthy) |

**`transcription-mcp` подробно** (второй, независимый от helm, конвейер речи):
- MCP-тулы: `create_source_media_upload_session`, `finalize_source_media_upload`,
  `submit_transcription_job`, `get_transcription_job`, `read_transcript`
  (`src/transcription_mcp/tools.py:91-226`).
- **Архитектура взрослая:** отдельный воркер, забирающий джобы **через Postgres-lease**
  с восстановлением зависших (`worker.py:157 claim_once`, `worker.py:207
  recover_stale_leases`); ffmpeg/ffprobe извлекают speech track
  (`media.py:89-214 FfmpegMediaCommands`); медиа и транскрипты в **MinIO** с
  project-scoped префиксами; ретенция и cleanup (`retention.py`, `cleanup.py`);
  аудит-события (`audit.py`); **шов расширения движков** — `engines/fal_whisper.py`
  + `engines/fake.py`, интерфейс спроектирован под локальные движки
  (ADR-0004 `docs/adr/0004-transcription-engine-extension-seam.md`).
- Экспорт транскрипта в **JSON / TXT / SRT / VTT** (`transcript_exports.py`).
- Защита от SSRF при импорте по URL — отдельная user story (PRD п.4).
- **26 файлов тестов**, включая `test_e2e_smoke.py`, `test_deployment_contract.py`,
  `test_project_scope_auth.py`, `test_migrations.py`.
- 4 ADR (`docs/adr/0001-0004`) + PRD + нарезка на слайсы.
- Движок сейчас — fal.ai Whisper (`FAL_WHISPER_MODEL_ID`, дефолт `fal-ai/whisper`);
  локальные движки сознательно вне MVP, но шов под них есть.

## 5. Остальное — что живо и что нет

- **`tacticum-dev` — агентная платформа профилей.** FastMCP+FastAPI, 5 bounded
  contexts × DDD-4-layer, Postgres+Alembic, Qdrant под Knowledge, Clerk-auth,
  Next.js 15 фронт. Каталог профилей ролей в `templates/` (iva-analysis-fr,
  iva-architect-mcp, iva-brownfield-mail, firebird-role-web, brownfield-task-workflow
  и др.). Активна: последний коммит **2026-07-30**, PR #199. Прод есть — сервер
  `tacticum_prod` (catalog-mcp, каталог профилей).
- **`platform`** — консолидирующий слой поверх работающего флота (LiteLLM/Langfuse/
  Infisical/project-hub/TEI/Qdrant/RabbitMQ). **Статус в самом README: bootstrap.**
  Есть контракты (OpenAPI/JSON-Schema/MCP-schemas), SDK python+ts, каркас сервисов
  (`knowledge_rag`, `document_processing`, `llm_gateway`, `memory_service`, …).
  По концепту трёх RAG: `knowledge_rag` — 37 тестов зелёные, **живого smoke не было**.
- **`KB-Brownfield-Bootstrap`** — конвейер brownfield-KB: из репозитория строятся
  `components.md` (блоки, score, Mermaid-карта зависимостей), `topology-map.json`,
  `uc-map.json`, `behaviour/<block>.md`, ADR, NFR/security. Плюс **алгоритм
  планирования доработки в 3 уровня детализации** (репо→блок→модуль) с бюджетом
  контекста на шаг (`docs/agent-change-planning-algorithm.md`). Последний коммит
  2026-07-07. Живого стенда не проверял.
- **`graph-builder` (TactFlow, visual_graph)** — Rasa-бот + симулятор + web,
  Postgres/Alembic. **README пустой (одна строка заголовка), последний коммит
  16 июня.** Самый сырой и самый непрозрачный кусок задела; в заявку я бы его не
  тащил без отдельной разведки.

## 6. ЧТО ИЗ НАШЕГО ЗАДЕЛА ЛОЖИТСЯ В ЗАЯВКУ

Дешевизна: **дёшево** = переносим почти как есть · **средне** = нужна интеграция и
адаптация · **дорого** = каркас есть, продукта нет.

| Наш компонент | Какую продуктовую фичу позволяет сделать быстро | Продукт | Цена |
|---|---|---|---|
| Цепочка `transcribe→dialogue→summarize` (helm, прод) | Итоги совещания из записи: расшифровка → **реплики с говорящими без диаризации** → саммари по пресету + возможность дозадать вопрос в том же треде | **Largo+ассистент** (записи ВКС/звонков) | **дёшево** |
| `chunk_plan` + чанкинг с перекрытием + удаление исходника в `finally` + `sweep_orphans` | Длинные записи (проверено на 66 мин) и приватность как инвариант, а не политика — прямой аргумент для on-prem-контура | оба | **дёшево** |
| `transcription-mcp` (project-hub, 7 недель аптайма) | Речь как проектный артефакт: upload/URL-импорт, джобы, история, экспорт **SRT/VTT/TXT/JSON** (субтитры к записям ВКС), права по проектам, аудит | **Largo+ассистент**, AVES (протоколы) | **дёшево** |
| Шов движков ASR (ADR-0004) + `tacticum/transcribe` в гейтвее | Замена внешнего whisper на локальный движок в контуре без переписывания сервиса — закрывает требование «данные не уходят наружу» | оба | **средне** |
| RAG-1 (докс-бот в проде + golden-184 + judge-метрики) | Ассистент по документации продукта / базе знаний с **измеренным** качеством (recall@5 0.98, faithfulness 0.963) | AVES+Terra (докбот), Largo (справка) | **дёшево** |
| RAG-2 / `helm-analyst` (14 тулов + A/B) | Ассистент аналитика: похожие задачи, покрытие требований, с кем согласовать, ограничения и противоречия, покрытие автотестами | AVES+Terra | **средне** |
| Дотяжка exact-key retrieval (recall 0.111→0.889) | Точный поиск по идентификаторам (номер требования, ключ задачи, артикул) поверх векторного — типовая боль корпоративного поиска | оба | **дёшево** |
| Кап реранка = 20 (−40% латентности при hit@5=1.0) | Готовый рецепт «быстрый ответ без потери качества» + методика замера по стадиям | оба | **дёшево** |
| `local-stack` (hybrid RRF + rerank + tenant-изоляция + eval-harness) | Мультитенантный поиск с доказуемой изоляцией данных заказчиков + воспроизводимый замер качества | оба | **средне** |
| `tei_service` (bge-m3 / локальный TEI, реестр моделей) | Эмбеддинги как сервис с переключением локальный/внешний — основа импортозамещаемого контура | оба | **средне** |
| `doc_translator`: 14 экстракторов + 12 реконструкторов + фиделити-гейт | Загрузка **любого** офисного формата в базу знаний с сохранением структуры; сборка выходных документов с вёрсткой | AVES+Terra (документооборот) | **дёшево** |
| `doc_translator`: OCR Tesseract→PaddleOCR + LibreTranslate | Сканы и фото документов в текст; офлайн-перевод внутри контура (без облака) | AVES+Terra | **средне** |
| `word-mcp` / `excel-mcp` / `wiki-mcp` / `taiga-mcp` (все живые) | Агент, который **пишет** результат в реальные артефакты — docx/xlsx/вики/трекер, а не отдаёт текстом | оба | **дёшево** |
| `arch-mcp` (Graphiti+FalkorDB, темпоральный граф) | Память решений: конфликты и дубли между ADR/PRD/RFC во времени — фича, которой у конкурентов обычно нет | AVES+Terra | **средне** |
| `wiki-digest` | Дайджест изменений базы знаний по тенантам на почту | оба | **дёшево** |
| `tacticum-dev` (профили ролей, скиллы, MCP-скоупинг) | Ролевые AI-профили (аналитик/QA/техпис/архитектор) как продуктовая надстройка | оба | **средне** |
| `KB-Brownfield-Bootstrap` | Автопостроение карты legacy-системы (блоки, зависимости, UC, ADR) + план доработки по уровням | AVES+Terra | **средне** |
| `platform` (`knowledge_rag`, `document_processing`, `llm_gateway`) | Единый слой под все фичи — но это **инвестиция**, а не готовая фича | оба | **дорого** |
| `graph-builder` (TactFlow) | Непонятно; README пуст, репозиторий стоит с 16 июня | — | **не тащить** |

## ВЕРДИКТ

Наш задел — не набор прототипов, а **три работающих в проде вещи** (медиа-цепочка
helm, RAG-1 докс-бот, флот MCP на project-hub с аптаймами 3–7 недель) плюс два
крупных офлайн-актива (`doc_translator`, `rag_eval_service`/`local-stack`) с
измеренным качеством. Самый сильный и самый «наш» элемент для заявки — **цепочка
transcribe→dialogue→summarize**: она решает конкретную инженерную проблему
(разметка говорящих LLM-проходом там, где ASR не даёт диаризации), доказана на
66-минутной записи в проде и целиком переносится в голосовой ассистент Largo.
Второй по силе — **экстракторы/реконструкторы doc_translator**: 14+12 форматов с
фиделити-гейтом это то, что обычно пишут годами.

**Проверено** (код + живой сервер): медиа-цепочка helm, флот MCP project-hub,
zu_demo (codex/qdrant/meili/lightrag), состав doc_translator и transcription-mcp.
**Данные:** цифры RAG-1 и RAG-2 — из заметок с замерами в vault, не мои прогоны;
цифры прода helm — мой прямой SELECT 2026-08-06.
**Подтверждение:** `docker ps` на `helm` и `project_hub`; SELECT по `media_request`;
`git ls-tree origin/main`; счётчики тестов.
**НЕ проверено:** `tei_service` живьём — сервер `adp_emb` не ответил на SSH;
`doc_translator` живьём (стенд не найден); `platform/knowledge_rag` живьём (по
концепту smoke не было); `graph-builder` вообще (README пуст, репозиторий стоит);
качество RAG-1 «после» изменений (judge-прогон не снят — так и в исходной заметке).
