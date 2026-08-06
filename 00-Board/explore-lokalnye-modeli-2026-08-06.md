---
title: 'Разведка: локальные (не-LLM) модели и детерминированные инструменты на своём
  железе'
type: report
status: draft
tags:
- board
- gost-docs
- models
- explore
permalink: tacticum/00-board/explore-lokalnye-modeli-2026-08-06
---

# Разведка: что у нас крутится на своём железе

Задача от лида: инвентаризовать мелкие локальные модели и детерминированный код под
автоматизацию техписа (ГОСТ-документы, требование «данные только через наши модели»).
Разведка read-only: код в `~/tacticum` + чтение живых серверов по SSH.

## 0. Главный вывод сразу

**Наш «локальный» эмбеддер на проде — НЕ локальный.** `bge-m3`, на котором стоят все
коллекции Qdrant (`*__bge_m3_1024`), обслуживается внешним DeepInfra. Локальный TEI у нас
есть, но он крутит другую модель (`gte-multilingual-base`, dim 768). Реранкер и
транскрибация — тоже DeepInfra. Полностью локальны у нас OCR, перевод и весь
OOXML-конвейер.

## 1. Эмбеддеры

Репозиторий `~/tacticum/tei_service` — да, это обёртка над HuggingFace Text Embeddings
Inference (TEI) плюс LiteLLM-гейтвей на том же хосте.

Реестр моделей — `/Users/bubblemac/tacticum/tei_service/models.yaml`:

| Модель | provider | dim | Где считает | Активна |
|---|---|---|---|---|
| `gte-multilingual-base` (`Alibaba-NLP/gte-multilingual-base`) | `tei` | 768 | **локальный TEI, CPU** | да |
| `qwen3-embedding-0_6b` (`Qwen/Qwen3-Embedding-0.6B`) | `tei` | 1024 | локальный TEI, cpu_or_gpu | **нет**, контейнер `tei-qwen3` не поднят |
| `bge-m3` (`BAAI/bge-m3`) | `openai_compatible` | 1024 | **внешний `api.deepinfra.com`** | да, `default_model` |

- `models.yaml:8` — `default_model: bge-m3`; `models.yaml:33-43` — `provider:
  openai_compatible`, `deployment: external`, `base_url: https://api.deepinfra.com/v1/openai`.
- `models.yaml:11-20` — единственная реально локальная модель, `internal_url: http://tei-gte:80`.
- `tei_service/docker-compose.yml:58-89` — контейнер `tei-gte`,
  образ `ghcr.io/huggingface/text-embeddings-inference:cpu-1.6`, команда
  `--model-id ${TEI_DEFAULT_MODEL_ID} --max-batch-tokens 8192 --max-client-batch-size 8`.
  Комментарий на `:64` — «GTE-multilingual-base на CPU держит идлово ~2.2 GiB».
- Сервиса `tei-qwen3` в compose нет вовсе — запись в реестре есть, деплоя нет.

**Живая проверка (сервер `gateway`, 155.212.134.20):**

```
cifragen-tei-gte-1        ghcr.io/huggingface/text-embeddings-inference:cpu-1.6   Up 2 months (healthy)
cifragen-embedding-api-1  tei-service:dev                                        Up 2 weeks (healthy)
cifragen-litellm-1        ghcr.io/berriai/litellm:main-stable                    Up 13 hours (healthy)
```
Железо: **4 vCPU, 7 GiB RAM, GPU нет** (`nvidia-smi` → пусто).
Живой `models.yaml` внутри `embedding-api` совпадает с git, только `bge-m3.max_batch_size`
поднят 4 → 64 (под внешнего провайдера, который держит крупный батч).

**Кто и как это зовёт.** helm ходит не в TEI напрямую, а через гейтвей-алиас
`tacticum/embed`: `tei_service/litellm/config.yml:70-76` → `openai/bge-m3` на
`http://embedding-api:8000/v1` → а там `bge-m3` = DeepInfra. Цепочка выглядит локальной
(`embedding-api` — наш контейнер), но упирается наружу.

Это зафиксировано в самом helm прямым текстом —
`/Users/bubblemac/tacticum/helm/docker-compose.prod.yml:55-57`:

> «bge-m3 обслуживается внешним DeepInfra (проверено 2026-07-18 — в embedding-app нет
> локальной ML-модели, только httpx→api.deepinfra.com)»

Там же на `:61` — «при свапе на локальный TEI (кап 4) правок helm не нужно»: клиент
адаптивен по батчу, то есть **переезд на локальный эмбеддер со стороны helm бесплатен**.
Цена переезда — не код, а пересчёт коллекций (dim 768 против 1024) и железо: bge-m3 на
4 vCPU без GPU поедет, но медленно.

Что это значит для задачи техписа: индексация и поиск по чужим документам **сегодня**
внешние вызовы требуют. Локально они возможны — но либо на `gte-multilingual-base` (уже
крутится, dim 768, качество на RU ниже), либо подняв `bge-m3`/`qwen3-embedding-0.6B` у
себя (реестр готов, контейнера нет).

## 2. OCR — полностью локальный

`~/tacticum/doc_translator`, два движка, оба на своём железе, без сети:

1. **Tesseract (основной)** — `backend/app/domains/processing/extractors/pdf_extractor.py:2563-2604`,
   вызов бинаря через `subprocess`, языки `rus+eng` (`:2581-2582`, `:2616-2617`), вывод TSV
   с координатами. Есть и режим native-guided (`:2707`).
2. **PaddleOCR (fallback)** — `backend/app/domains/processing/ocr_engine.py:11-34`,
   синглтон `PaddleOCR(use_angle_cls=True, lang="ru")`. Вызывается из
   `pdf_extractor.py:2508-2519`, **за флагом** `PDF_PADDLE_OCR_FALLBACK_ENABLED`
   (дефолт `False`, `backend/app/core/settings.py:47`). Зависимости — `paddleocr>=2.7`,
   `paddlepaddle>=2.6` (`backend/pyproject.toml:28-29`), то есть модель едет в образе.

Классификатор native/scan (какую страницу гнать в OCR) — детерминированный скоринг,
`pdf_extractor.py:353-420`, плюс детект mojibake в нативном слое (`:444`). Порог
`MIN_OCR_TRANSLATION_CONFIDENCE = 0.70` (`settings.py:58`). Бенчмарк маршрутизации —
`doc_translator/docs/specs/pdf-ocr-routing-benchmark-report-2026-05-22.md`.

**Второй OCR** — `~/tacticum/rag_eval_service/document_processing`: Tesseract `rus+eng`
через subprocess (`service.py:98-131`, `app.py:40` отдаёт `"ocr_engine": "tesseract"`).
Экстракторы туда вендорены из doc_translator, но OCR при вендоринге переставили с Paddle
на Tesseract. Сервис отдаёт `list[TextSegment]` с `extraction_source` (`native`/`ocr`),
страницей и bbox. Чанкинга и эмбеддингов внутри нет намеренно.

## 3. Перевод

**LibreTranslate — локальный контейнер.** `doc_translator/docker-compose.yml:73-84`:
образ `libretranslate/libretranslate:latest`, `cpus: 2.0`, `mem_limit: 2g`, `LT_THREADS: 4`,
модели в volume `lt_data:/home/libretranslate/.local`. Клиент —
`backend/app/infrastructure/translation/libretranslate_client.py`, пары
`SUPPORTED_PAIRS = {("ru","en"), ("ru","es"), ("ru","pt")}` (`:13`), кэш переводов в Redis
на 24 ч (`:16`), батчинг склейкой с маркерами `DTSEG` (`:19`).

Каталог `backend/app/infrastructure/translation/` содержит **только** этот клиент — то есть
в doc_translator **нет LLM-пути перевода вообще**, ни запасного. Наружу не ходит ничего.

**MarianMT в Terra — не нашёл, и, судя по всему, его нет.** Греп по всему `~/tacticum` на
`marian|MarianMT|opus-mt` даёт ноль совпадений. Все хиты на «terra» — это `gpt-5.6-terra`,
имя внешней модели OpenAI в шаблонах QA-профилей
(`tacticum-dev-qa-kit/templates/iva-web-brownfield/CHANGELOG.md:183` и его копии в
worktree-ветках). Локального Marian-переводчика у нас нет. Если у лида «Terra» — это что-то
другое (продукт заказчика?), нужен уточняющий ориентир: репозиторий или хост.

## 4. Реранкеры

**Внешний.** Живой конфиг гейтвея (`docker exec cifragen-litellm-1`, строки 85-90):

```yaml
- model_name: tacticum/rerank
  litellm_params:
    model: deepinfra/Qwen/Qwen3-Reranker-4B
  model_info:
    mode: rerank
```

Осторожно: **комментарии в коде врут про модель**. Везде в helm написано
«bge-reranker-v2-m3» — `helm/src/helm/config.py:207`, `helm/src/helm/llm/gateway.py:57-58`,
`helm/src/helm/infrastructure/assistant/reranker.py:3`,
`helm/src/helm/infrastructure/rag2/reranker.py:3`,
`helm/src/helm/infrastructure/docs_assistant/reranker.py:3`. Живой тир —
`Qwen/Qwen3-Reranker-4B`. В git-версии `tei_service/litellm/config.yml` тира `rerank` нет
совсем: конфиг на сервере ушёл вперёд репозитория.

Интерфейс — Cohere-совместимый `/v1/rerank`, `helm/src/helm/infrastructure/assistant/reranker.py:31-42`.
Включён на проде для всех трёх контуров: `helm/docker-compose.prod.yml:45-47`
(`HELM_DOCS_RERANK_ENABLED`, `HELM_RAG2_RERANK_ENABLED`, `HELM_ASSISTANT_RERANK_ENABLED` = 1).

Остальное в RAG-пайплайне helm — **детерминированное и локальное**: RRF-федерация,
шумовая отсечка по калиброванной confidence (`HELM_RAG2_NOISE_FLOOR: "0.5"`,
`HELM_RAG2_NOISE_ACTION: "drop"`, `docker-compose.prod.yml:50-53`), Qdrant. Из внешних
моделей в цепочке ровно два звена: эмбеддер и реранкер.

**Есть готовый локальный образец.** `~/tacticum/local-stack` — стенд под задачу #32:
`app/embeddings.py` — bge-m3 через `sentence-transformers`, `app/reranker.py` —
bge-reranker-v2-m3 через `FlagEmbedding`, оба в процессе, плюс Qdrant + Meilisearch + RRF
(`local-stack/README.md:1-25`, `pyproject.toml:17-18`, экстра `models`). Это песочница, не
деплой, но она доказывает: связка эмбеддер+реранкер целиком локально у нас уже собиралась.

## 5. Речь и зрение — локального нет

- **Vision:** греп по `~/tacticum` на `llava|qwen-vl|qwen2-vl|florence|donut|layoutlm|clip-vit|open_clip`
  → ни одного совпадения в коде (все хиты — слово «clip» в figma-отчётах
  `tacticum-dev*/figma-ds-process/iva-core-mapping/*.md`). Тир `tacticum/vision` в гейтвее
  (`tei_service/litellm/config.yml:47-50`) — это `gemini/gemini-2.5-flash`, внешний.
  **Локальной модели для описания скриншотов интерфейса у нас нет.**
- **Речь:** `project-hub/transcription-mcp` работает через **fal.ai Whisper** —
  `src/transcription_mcp/engines/fal_whisper.py`, конфиг из env `FAL_WHISPER_MODEL_ID`
  (`tests/test_fal_whisper_engine.py:131-137`). Плюс тир `tacticum/transcribe` в гейтвее =
  `deepinfra/openai/whisper-large-v3` (живой конфиг, строки 79-84; комментарий: «решение
  владельца 2026-07-31»). Оба пути внешние. Локального whisper.cpp/Vosk нет.
- Контейнеры `infra-transcription-mcp-1` и `infra-transcription-worker-1` на сервере
  `project_hub` подняты (Up 7 weeks) — но считают не они, они гоняют аудио к провайдеру.

## 6. Детерминированные инструменты — модели не участвуют вообще

Тут вопрос утечки не возникает: ни одного сетевого вызова к модели.

### `project-hub/word-mcp` (живёт: `infra-word-mcp-1`, Up 7 weeks, `project.cifragen.ru/word`)

Обёртка над `SecurityRonin/docx-mcp` (`word-mcp/README.md`). Всё — прямые операции над
OOXML через python-docx:

- жизненный цикл: `open_document`, `create_document`, `create_from_markdown`, `save_document`, `get_document_info`;
- чтение: `get_headings`, `search_text`, `get_paragraph`;
- **правки с track changes**: `insert_text`, `delete_text`, `accept_changes`, `reject_changes`, `set_formatting`;
- комментарии: `get_comments`, `add_comment`, `reply_to_comment`;
- таблицы: `get_tables`, `add_table`, `modify_cell`, `add_table_row`, `delete_table_row`; списки: `add_list`;
- сноски/концевые: `add_footnote`, `validate_footnotes`, `add_endnote`, `validate_endnotes`;
- колонтитулы и стили: `get_headers_footers`, `edit_header_footer`, `get_styles`;
- свойства и картинки: `get_properties`, `set_properties`, `get_images`, `insert_image`;
- разделы и перекрёстные ссылки: `add_page_break`, `add_section_break`, `set_section_properties`, `add_cross_reference`;
- защита и слияние: `set_document_protection`, `merge_documents`;
- валидация: `validate_paraids`, `remove_watermark`, `audit_document`.

Плюс наши четыре тула шаблонов — `word-mcp/src/word_mcp_wrapper/templates_tools.py:99-140`:
`upload_template`, `list_templates`, `delete_template`, `create_from_template`. И выгрузка:
`minio_tools.py:83-84` (`get_download_url`, presigned), HTTP-роуты `/files` и `/files/{name}`
(`__main__.py:172-200`).

Под ГОСТ это ровно то, что нужно: сборка из шаблона `.dotx`, правка по стилям,
нумерованные списки, сноски, колонтитулы, перекрёстные ссылки — и **правка с track
changes**, то есть техпис видит, что изменено.

### `doc_translator` — конвейер OOXML без единой модели

Каталог `backend/app/domains/processing/`:

- **Экстракторы** (`extractors/`): `docx`, `xlsx`, `pptx`, `pdf`, `odt`, `rtf`, `epub`,
  `md`, `srt`; конвертеры legacy `doc_converter.py`, `ppt_converter.py`, `xls_converter.py`.
- **Реконструкторы** (`reconstructors/`, 11 штук + `dispatcher.py`) — сборка документа
  обратно с сохранением разметки.
- **Рендер в PDF**: `ooxml_libreoffice_renderer.py:22-110`, `soffice`/`libreoffice` через
  `subprocess`; LibreOffice ставится в образ (`backend/Dockerfile:9`:
  `libreoffice-core libreoffice-writer libreoffice-impress libreoffice-calc`).
- **Диффы**: `ooxml_structural_diff.py` (структура), `ooxml_render_diff.py`,
  `ooxml_visual_diff.py` (визуальное сравнение рендеров).
- **Отчёты о точности**: `docx_fidelity_report.py`, `xlsx_fidelity_report.py`,
  `pptx_fidelity_report.py`, `pdf_fidelity_report.py`.
- **Тонкая работа с OOXML**: `docx_hyperlink_text.py`, `xlsx_rich_text.py`,
  `xlsx_comments.py`, `xlsx_workbook_metadata.py`, `pptx_chart_text.py`,
  `pptx_master_text.py`, `ooxml_namespace.py`, промежуточное представление `ir.py`.

Это, по сути, готовый бэкенд «разобрать документ → поправить → собрать → отрендерить →
сравнить с оригиналом», и он **весь** работает без моделей. Развёрнутым я его нигде не
нашёл: на серверах `gateway`, `project_hub` и `helm` контейнеров `doctranslate_*` нет.

Детерминированное и в helm: `api_registry_check` и `requirement_tests` в
analyst-MCP отдают данные без LLM (по описанию MCP-сервера).

---

## ВЕРДИКТ

**Полностью локально, без единого внешнего вызова, мы умеем сегодня:**

1. **Разобрать и собрать документ** — docx/xlsx/pptx/pdf/odt/rtf/epub/md/srt, с сохранением
   стилей, таблиц, сносок, колонтитулов, гиперссылок (`doc_translator`, 11 экстракторов +
   11 реконструкторов).
2. **Править Word по стилям и с track changes** — `word-mcp`, ~45 тулов, уже поднят на проде.
   Сборка из шаблона `.dotx` — есть.
3. **Распознать сканы и картинки** — Tesseract `rus+eng` (основной) + PaddleOCR `ru`
   (fallback), с детерминированным классификатором «нативная страница / скан».
4. **Перевести ru→en/es/pt** — LibreTranslate в своём контейнере, 2 CPU / 2 GiB.
5. **Отрендерить в PDF и сверить результат** — LibreOffice headless + структурный,
   render- и визуальный диффы + fidelity-отчёты по всем четырём форматам.
6. **Уложить в шаблон и проверить формальные требования** — поиск-замена, стили,
   нумерация, валидация `paraid`, `audit_document`.

**Наружу сегодня уходит:** эмбеддинги (DeepInfra bge-m3), реранк (DeepInfra
Qwen3-Reranker-4B), транскрибация (DeepInfra whisper-large-v3 / fal.ai), любое зрение
(Gemini). Из них **эмбеддинги и реранк переводятся в локальные без переписывания helm** —
модели известны, реестр готов, `local-stack` уже собирал эту связку локально; упирается в
железо (4 vCPU / 7 GiB / без GPU) и в пересчёт коллекций при смене размерности.

**Локального зрения нет** — описывать скриншоты интерфейса без внешней модели мы сейчас не
можем ничем.

---

**Проверено** — сервисы и модели, поднятые на `gateway` (155.212.134.20), `project_hub`
(45.141.79.157) и `helm` (159.194.233.33); маршрутизация тиров `tacticum/embed`,
`tacticum/rerank`, `tacticum/transcribe`, `tacticum/vision` в живом конфиге LiteLLM; живой
`models.yaml` внутри `embedding-api`; состав OCR, перевода и OOXML-конвейера по коду.

**Данные** — `docker ps`, `nproc`, `free -g`, `nvidia-smi` на трёх серверах;
`docker exec` чтения `config.yml` и `models.yaml`; исходники `~/tacticum/{tei_service,
doc_translator, helm, project-hub, local-stack, rag_eval_service}`.

**Подтверждение** — внешность `bge-m3` сходится в трёх независимых местах: реестр
(`provider: openai_compatible`), живой конфиг на сервере и комментарий helm с датой
проверки 2026-07-18. Локальность OCR и LibreTranslate — по коду (subprocess к бинарю,
образ в compose), сетевых вызовов в этих путях нет.

**НЕ проверено** —
(1) Развёрнут ли `doc_translator` хоть где-то: на трёх осмотренных серверах его нет,
    остальные четыре (`zu_demo`, `adp_emb`, `teststand`, `tacticum_prod`) не смотрел.
(2) Скорость `bge-m3` на нашем CPU без GPU — цифр нет, только оценка «поедет медленно».
(3) Реальное качество `gte-multilingual-base` против `bge-m3` на русских ГОСТ-текстах —
    eval не гонял.
(4) Что лид имел в виду под «MarianMT в Terra»: в коде этого нет, нужен ориентир.
(5) `tacticum/triva` (Gemma в контуре ИВА через SSH-туннель) — не наше железо, оставил
    другому разведчику.