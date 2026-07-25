---
title: explore-golden-эталоны-rag1
type: note
permalink: tacticum/00-board/explore-golden-etalony-rag1
status: draft
role: explorer
tags:
- explore
- rag1
- golden
- eval
- answer_in_context
---

# explore-golden-эталоны-rag1

Дизайн подхода «golden с эталонами» для честного answer-eval RAG#1. Реализации нет — план для ГД.

## Главная находка: инфраструктура УЖЕ существует
В репозитории golden лежит готовый автономный пакет `~/tacticum/iva-rag1-docs/golden/answer_eval/` (stdlib-only), который ровно закрывает задачу:
- `gen_ideal_answers.py` — генерит `ideal_answer` + `ideal_answer_key_facts` из ТЕКСТА релевантной страницы (заземлённо).
- `corpus.py` — slug → `md_path` (из `manifest.json`) → текст страницы. **Тексты доков доступны ЛОКАЛЬНО** (`md/…`), Qdrant для извлечения эталонов НЕ нужен.
- `metrics.py`, `judge.py`, `gateway.py`, `answer_eval.py` (harness), `tests/`, `sample/`.
- `README.md` описывает всю схему полей и запуск.

Вывод для ГД: не строить с нуля — переиспользовать/допилить существующий пайплайн. Недостающее — только собственно ПРОГОН генерации на подвыборке + отбор сабсета.

## Как метрика реально считается (канонический путь docs_eval)
`docs_eval.py` → `runner.evaluate()` → `metrics.answer_in_context()`:
- `src/helm/eval/golden.py` `GoldenCase`: читает поля `ideal_answer: str|None`, `ideal_answer_key_facts: tuple[str,...]`. Незнакомые поля (`ideal_answer_meta`) игнорит — схема не ломается.
- `runner.evaluate` (runner.py:126-127): `key_facts = resolve_key_facts(case.ideal_answer, case.ideal_answer_key_facts)`; `aic = answer_in_context(key_facts, top_texts)` где `top_texts = [c.text for c in chunks[:k]]`.
- `metrics.resolve_key_facts` (metrics.py:165): если есть `ideal_answer_key_facts` — берёт их; иначе эвристика `extract_key_facts(ideal_answer)`; иначе `[]`.
- `metrics.answer_in_context` (metrics.py:202): `total==0 → undefined=True` (в агрегаты не идёт — **вот почему сейчас метрика undefined: оба поля пусты во всех 1306**). Иначе — доля key_facts, чья нормализованная форма является подстрокой объединённого текста top-k. Две нормализации: `normalize` (NFKC, lower, ё→е, без пунктуации) и `_norm_keep_symbols` (сохраняет URL/пути). `hit = score ≥ 0.5`.
- `aggregate` (runner.py:175-178): `answer_in_context@k` и `context_hit@k` считаются ТОЛЬКО по `not aic_undefined` кейсам (`n_scored`).
- LLM-judge (`judge.py`) — за флагом `--judge`; `correctness` сравнивает сгенерированный ответ с `ideal_answer` (нет эталона → null).

## Точный формат кейса с эталоном (добавляется, не ломая старое)
```jsonc
{
  "id": "cs-ag-admin-api#1",
  // ...все существующие поля без изменений...
  "ideal_answer": "Короткий фактический ответ по тексту страницы (2-4 предл.).",
  "ideal_answer_key_facts": ["/api/v1/swagger.yaml", "swagger.json", "Postman"],
  "ideal_answer_meta": {"model": "...", "prompt_version": "gen-ideal-v1",
                        "source_slug": "cs-ag-admin-api", "mode": "run"}
}
```
Для детерминированного `answer_in_context@k` достаточно `ideal_answer_key_facts`. `ideal_answer` нужен только для LLM-judge `correctness`.

## Метод извлечения key_facts (заземлённый, не циркулярный)
Источник фактов — **первоисточник (полный текст страницы `relevant_doc_ids[0]`), НЕ генерация RAG**.

Почему циркулярности нет:
- `answer_in_context@k` — чисто детерминированный substring-матч key_facts против текста извлечённых чанков. НИКАКОЙ генерации RAG в метрике нет. Факты из first-party дока, метрика меряет «достал ли retrieval чанк с ответом». Циркулярность отсутствует по построению.
- Единственная точка риска — LLM-judge `correctness` (ответ RAG vs `ideal_answer`). Развязка: модель-генератор эталона (`GEN_MODEL`, дефолт `tacticum/cheap` через Gateway) держать ОТЛИЧНОЙ от продуктового генератора ответа RAG#1 (`IvaAnswerGenerator`, vLLM-контур). Тогда общей систематической ошибки нет.

Детерминированно (выжимка предложений) vs LLM-суммаризация:
- Рекомендация: **гибрид** — LLM формулирует ideal_answer + предлагает атомарные факты (числа/URL/пути/имена параметров), НО добавить дешёвый детерминированный **guard**: каждый key_fact обязан быть verbatim-подстрокой текста исходной страницы (после `_norm_keep_symbols`), иначе — выбросить. Это режет парафраз/галлюцинацию LLM и гарантирует заземление. Чисто детерминированная выжимка предложений даёт шумные факты (см. mock-пример: тянет «API», «YAML» — общие термины), поэтому только фолбэк.

## Размер и отбор подвыборки
Не 1306. Рекомендация: **~200 кейсов** (основной ярус) + **~60** дымовой ярус.
- 200 достаточно для устойчивых агрегатов и парного сравнения verbose vs brief на ОДНОМ сабсете (paired → низкая дисперсия), при этом дёшево по LLM-judge.
- Отбор — **стратифицированный, детерминированный (фикс. seed)**:
  1. Первичная ось — `product` (главный источник гетерогенности корпуса). Пропорционально, но с полом ≥15 на продукт, чтобы Mail/Room/Terra (41/41/76) не утонули против MCU (706).
  2. Внутри продукта — пропорционально по `qtype` и `difficulty`.
  3. Брать только кейсы, где извлечение key_facts успешно (непустой grounded-набор) и страница найдена в manifest.
- Распределение генсовокупности (для стратегии): product MCU706/IVA One216/CS139/SBC87/Terra76/Mail41/Room41; qtype how_to363/factual313/concept226/config_setup206/troubleshooting94/ui_navigation69/integration_api35; difficulty typical773/hard367/edge166. У всех кейсов ровно 1 `relevant_doc_id`.

## Куда класть (не ломать существующее)
- Оригинал `golden/golden_iva_rag1.json` — не трогать.
- Новый файл рядом: `golden/golden_iva_rag1.eval200.json` — стратифицированный сабсет С эталонами. Это «файл честной меры».
- (Опц., дорого) полный `golden/golden_iva_rag1.with_ideal.json` — конвенция из README, если позже захотят эталоны на всех 1306.

## Пошаговый build-план
1. **Скрипт отбора сабсета** (детерминированный, локально, без сети): читает `golden_iva_rag1.json`, стратифицирует по product×qtype×difficulty с полом на продукт, пишет `golden_iva_rag1.eval200.json` (пока без эталонов). Можно расширить существующий `golden/review/_build_sample.py`.
2. **Генерация эталонов на сабсете** (нужен `GATEWAY_API_KEY`; тексты страниц — локальные md, Qdrant НЕ нужен):
   ```bash
   GATEWAY_API_KEY=... GEN_MODEL=tacticum/cheap \
   python3 golden/answer_eval/gen_ideal_answers.py \
     --golden golden/golden_iva_rag1.eval200.json --run --only-missing \
     --out golden/golden_iva_rag1.eval200.json --batch 50
   ```
   Локально без ключа — сначала `--mock --limit 8` для валидации формата.
3. **Guard-валидация фактов** (детерминированный, локально): для каждого кейса проверить, что каждый `ideal_answer_key_facts[i]` — verbatim-подстрока текста `relevant_doc_ids[0]` (через `corpus.page_text` + `_norm_keep_symbols`); выкинуть незаземлённые. (Нового модуля нет — добавить как post-step/скрипт.)
4. **Прогон метрики на проде** (там Qdrant `iva_docs__bge_m3_1024` + эмбеддинги; retrieved `payload.text` — это то, против чего матчатся факты):
   ```bash
   docker exec helm-helm-1 python -m helm.eval.docs_eval \
     --golden /app/golden_iva_rag1.eval200.json --k 5 --out /app/eval_run.json
   # + --judge для faithfulness/correctness (платно, развязать модель судьи vs RAG)
   ```
   Теперь `answer_in_context@k`/`context_hit@k` определены (`n_scored>0`), «слепые» кейсы подсвечиваются.
5. **Сравнение краткости**: прогнать шаг 4 для двух конфигов RAG (verbose vs brief) на ТОМ ЖЕ eval200; сравнить `answer_in_context@k`, `judge.correctness`, `judge.faithfulness` по срезам. Падение → краткость уронила качество.

## Риски
- **Качество извлечённых фактов**: LLM может дать парафраз, не совпадающий буквально с чанком → ложное «missing». Митигация — guard-валидация (шаг 3) + просить атомарные verbatim-сущности (URL/числа/имена), не фразы.
- **Chunk vs page**: факты извлекаются из ПОЛНОЙ страницы, а метрика матчит против top-k ЧАНКОВ. Факт из дальнего раздела может не попасть в top-k чанк даже при верном doc-level recall — это и есть «слепота», которую метрика ловит (не баг), но key_facts надо тянуть из части страницы, отвечающей на query.
- **Покрытие**: кейсы без md-страницы в manifest (`skipped_nopage`) выпадут — проверить, что сабсет их не содержит.
- **Стоимость LLM**: генерация ~200 (дёшево, `tacticum/cheap`); judge — платный, поэтому сабсет, а не 1306.
- **Циркулярность judge**: если GEN_MODEL == модель ответа RAG — общая систематическая ошибка. Держать модели разными.
- **Дрейф корпуса**: эталоны валидны против `corpus_version=latest`; при перекрауле iva.ru — перегенерить затронутые (id `<slug>#n` стабильны).

## Ключевые файлы (абсолютные пути)
- Метрика/раннер: `/Users/bubblemac/tacticum/helm/src/helm/eval/{metrics.py,runner.py,docs_eval.py,golden.py,judge.py,adapters.py}`
- Готовый пайплайн эталонов: `/Users/bubblemac/tacticum/iva-rag1-docs/golden/answer_eval/{gen_ideal_answers.py,corpus.py,metrics.py,judge.py,answer_eval.py,README.md}`
- Golden + корпус: `/Users/bubblemac/tacticum/iva-rag1-docs/golden/golden_iva_rag1.json`, `/Users/bubblemac/tacticum/iva-rag1-docs/manifest.json`, `/Users/bubblemac/tacticum/iva-rag1-docs/md/`