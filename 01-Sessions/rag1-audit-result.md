---
title: rag1-audit-result
type: report
permalink: tacticum/01-sessions/rag1-audit-result
tags:
- rag1
- docs-assistant
- bot-support
- audit
- helm
- night
---

# RAG#1 (docs_assistant) — аудит и улучшения

Ночной автономный аудит RAG#1 (публичная дока ИВА, бот «Поддержка» → `docs_ask`). Воркер helm, коорд. тимлид. Дата 2026-07-17.

## Итог одной строкой
Тесты **зелёные (1606 passed / 31 skipped, RAG#1-срез 128/0)**, тестов-обманок нет. Корпус выверен (796 стр / 8272 чанка, golden 796/796 без рассинхрона). Найдено **1 главный дефект качества (нет score-floor)** + 2 неактивных рычага. Реализован безопасный фикс F1 (score-floor, default-off) в worktree `fix/rag1-rerank-floor` (`c6fea90`), не деплоен. **Полный прод-eval заблокирован classifier — делегирован лиду.**

## 1. Тесты RAG#1
- `pytest -k "docs or bot_support"` → **128 passed, 0 skip/xfail**. Полный suite → **1606 passed, 31 skipped** (skip'ы предсуществующие, интеграционные с живыми сервисами, не RAG#1).
- Тестов-обманок НЕ найдено: моки только на границах I/O (Gateway/Qdrant/Meili/LLM), доменная логика (чанкинг, RRF, near-dup, цитаты, guardrail, known-gaps) реальная и юнит-покрыта.

## 2. Реальный корпус (curl, read-only, без прод-секретов)
- Коллекция Qdrant `iva_docs__bge_m3_1024`: **8272 чанка, 796 уникальных страниц**, dim 1024 cosine, статус green. Индексы payload: tenant_id/product/section/version.
- Golden `/root/iva-docs/golden/golden_iva_rag1.json` = **1306 кейсов**, все с `relevant_doc_ids` (slug страниц), БЕЗ `ideal_answer`/`key_facts` → считаются только retrieval-метрики (recall/hit/mrr/ndcg), answer_in_context/judge недоступны.
- **Slug-выверка: golden таргетит 796 уникальных slug, все 796 присутствуют в корпусе (0 absent)** — eval меряет реальный ретрив, не артефакт формата ключей.
- Распределение: qtype how_to 363 / factual 313 / concept 226 / config_setup 206; product MCU 706 / IVA One 216 / CS 139.
- **Прод-флаги RAG#1:** hybrid (Qdrant+Meili+RRF) + **rerank ON** (`HELM_DOCS_RERANK_ENABLED=1`), Meili ON, бот ON. query_rewrite / near_dup_dedup — **OFF**. known_gaps ON.

### docs_eval — ЗАБЛОКИРОВАН (нужен лид/verifier)
Полный пайплайн (embed запроса через Gateway + rerank) требует `docker exec python` и прод-секретов (gateway/meili ключи) — **classifier блокирует** (защита прод-exec из session-state). Меня хватило на: Qdrant scroll без ключа (структура/slug-выверка/near-dup-подсчёт). Golden уже скопирован в контейнер `/tmp/golden_rag1.json`.
Команда для лида:
```
docker exec helm-helm-1 python -m helm.eval.docs_eval --golden /tmp/golden_rag1.json --k 5 --out /tmp/eval_rag1.json
# сравнение стадий: --compare --modes semantic,hybrid,hybrid_rerank  (тяжело, можно --limit 200)
```

## 3. Дефекты / улучшения (ранжировано)

### F1 (ГЛАВНОЕ) — нет score-floor на ретриве → риск галлюцинаций. РЕАЛИЗОВАН (default-off).
`DocsAssistant.ask`: guardrail (`decide_guardrail(len(chunks), text)`) отказывает только если чанков 0 (а dense+ft всегда возвращают k ближайших) ИЛИ LLM сам сказал «не нахожу». На вопросе вне корпуса, не пойманном known-gaps, модель может синтезировать правдоподобный ответ из нерелевантных фрагментов. Аналог был решён в RAG#2 (confidence-floor τ), в RAG#1 отсутствовал.
**Фикс:** `Settings.docs_rerank_floor` (default None=off). После реранка фрагменты со `score < floor` выбрасываются; контекст опустел → guardrail даёт честный отказ (LLM не зовём). Применяется только при активном реранке (шкала bge-reranker-v2-m3). Default → поведение не меняется. 4 юнит-теста. Ветка `fix/rag1-rerank-floor` `c6fea90`. **Включение/калибровка порога — за лидом (нужен negative-set + A/B на реальном корпусе).**

### F2 — near_dup_dedup OFF в проде (рычаг, предлагаю A/B). 
Ingest размечает канон/дубль (D5), но `docs_near_dup_dedup_enabled=False` → `dedup_near_dup` в поиске не применяется. Измерено: **39/796 стр (4.9%) — near-dup дубли, 283/8272 чанка**. `MAX_CHUNKS_PER_DOC=4` капит per-slug, но НЕ per-группу → дубль+канон могут занять до 8 из 10 слотов контекста near-идентичным текстом, вытесняя разнообразные релевантные страницы. Флаг есть, dedup реализован и покрыт тестами — риск низкий, но меняет ретрив → A/B-включить.

### F3 — query_rewrite (синонимы) OFF в проде (рычаг). 
`synonyms.json` (35 групп: МСУ↔MCU, ВКС↔видеоконференцсвязь, переговорка↔Room). Помогает Meili ловить аббревиатуры/названия продуктов. `docs_query_rewrite_enabled=False`. Не измерено для RAG#1 — предлагаю A/B на golden.

### F4 (минор, UX бота) — дедуп источников по title casefold может схлопнуть разные страницы. 
`application/bot_support.py::_collect_sources` дедуплицирует источники по url И по нормализованному title. Две реально разные страницы с одинаковым заголовком (напр. «Настройка» в MCU и CS, разные url) схлопнутся в один источник. Это НАМЕРЕННЫЙ трейд-офф (глоссарии дублируются per-product, иначе спам 8× «Создание мероприятия»). Оставлено как есть; при желании — ключ дедупа (title, product).

## Проверенные и снятые гипотезы (НЕ дефекты)
- **ft-only чанки в hybrid** — не баг: `_index_meili` кладёт `{"id":uid, **payload}` с полным `text`, гидратация `_chunk_from_payload` корректна.
- Guardrail/цитаты (`select_cited_chunks`) — множественные `[2,5]`, дедуп по документу, согласованная перенумерация — корректно, покрыто.
- fail-closed tenant (Qdrant+Meili), known-gaps (32 антипримера + триггеры), sanitize_markdown / bot `_process_markdown` — зрелые, тестированы.

## Что готово к деплою
- **F1** (`fix/rag1-rerank-floor`, default-off) — безопасно мержить/деплоить: нулевое изменение поведения без установки флага. Реальный эффект — после калибровки порога.
- F2/F3 — код уже в проде (флаги), деплой не нужен; нужен A/B-замер и решение о включении.

## Ссылки
- Код: `src/helm/application/docs_assistant.py`, `infrastructure/docs_assistant/*`, `interface/api/routers/{docs,bot_support}.py`, `ingest/docs_index.py`, `eval/docs_eval.py`.
- Бот: [[bot-podderzhka-webhook-rag-1-v-chat-iva-zadeploen]]
- Чекпойнт: [[session-state]]

## Апдейт: решения лида + финализация F1 (2026-07-17, ночь)
- **F1 одобрен лидом, финализирован** в worktree `fix/rag1-rerank-floor` (`7f57d23`). Приведён в **симметрию с RAG#2** (`domain.rag2.apply_noise_floor`/`normalize_confidence`): floor по **калиброванной confidence 0..1** (та же шкала, что `rag2_noise_floor`; Gateway `/rerank` отдаёт relevance_score в 0..1), `score=None` — **иммунен** (реранк деградировал → не режем), `floor<=0` → no-op. 5 юнит-тестов, весь suite зелёный (1606+), ruff+mypy чисто. Default None → нулевое изменение поведения. **Не пушен, не деплоен.** Калибровку τ + включение делает лид после eval.
- **docs_eval — лид запустит САМ, ПОЗЖЕ** (не сейчас): rerank×1306 грузит тот же Gateway/rerank, что занят verify-sweep'ом RAG#2 (4 прогона) + живые аналитики; параллелить = деградация. Отложено до конца sweep (~1-2ч), прогон с `--compare --modes semantic,hybrid,hybrid_rerank`. Golden готов в контейнере `/tmp/golden_rag1.json`.
- **F2 (near_dup_dedup) и F3 (query_rewrite) — НЕ включать.** Кандидаты на A/B после eval, решение по числам. Гипотезы:
  - F2: включение dedup поднимет разнообразие контекста (убирает дубль+канон из 10 слотов) → возможный ↑ answer_in_context/recall@k на страницах с print/qsg-вариантами. Риск: если дубль ранжируется выше канона и текст канона полнее — нейтрально/+.
  - F3: синонимы (МСУ↔MCU, ВКС↔…) → ↑ ft-recall на запросах с аббревиатурами/названиями продуктов; риск разбавления dense-эмбеддинга запроса хвостом синонимов (mизмерить hybrid vs hybrid+rewrite).
- **Деплой RAG#1-фиксов** — лид скоординирует одним rebuild после sweep (вместе с links-volume), не в пик замера.
