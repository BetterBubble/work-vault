---
title: impl-rag2-analyst-latency
type: note
permalink: tacticum/00-board/impl-rag2-analyst-latency
tags:
- implementer
- rag2
- latency
- demo
- roscosmos
- draft
---

# impl-rag2-analyst-latency — ускорение /api/rag2/answer (демо Роскосмос 16:00)

status: draft · autonomy: правки в ветку, НЕ пуш/НЕ мерж/НЕ деплой (деплой+замер прод — лид)
branch: `feat/rag2-analyst-latency` · worktree `/Users/bubblemac/tacticum/helm-wt-rag2-latency`
commits: **6b6010c** (perf-пакет) + **cff7299** (канон source-ключа). Между ними в ветку приехал
merge `913aa4b` (чипы-пресеты демо-вопросов, UI) — учтён.

## Цель
Стабильно <25с общего времени `POST /api/rag2/answer` без `synthesis_failed` на демо-вопросах.
Замерено лидом: SSO 24.8с (впритык), Звонки 58.5с→fail, поздний SSO-UI 32.4с→fail при коротком
`topology_len=158` — виноват ОБЪЁМ ВХОДА промпта + `max_tokens 900`, не сама топология.

## Что сделано (по рычагам)

### 1. Кэш синтез-ответов + прогрев (ГЛАВНЫЙ рычаг — снимает латентность из сценария)
- In-process кэш `/answer`. **Ключ = норм(query) + k + КАНОН(source) + прочие фильтры** (cff7299).
  norm = trim+схлопнуть пробелы+lower. source-канон: фронт при дефолтном чипе «Все источники»
  шлёт `filters=None`, warm-скрипт — тоже без filters → оба → токен `src=all` (дефолтный чип =
  кэш-хит прогретого). Конкретный источник (jira/confluence/helm) → ОТДЕЛЬНЫЙ ключ, разные
  источники НЕ схлопываются (чип-фильтр не получит чужую выдачу).
- TTL 6ч, флаг `HELM_RAG2_SYNTH_CACHE_ENABLED` (on). Кэшируем ТОЛЬКО успешный синтез (не
  `synthesis_failed`, не пустой) — демо не залипнет на фолбэке. Хит → мгновенно, triva не зовём.
  Файл: `interface/api/routers/rag2.py` (`_answer_cache_key/_get/_put`, `_SOURCE_ALL`).
- Прогрев `scripts/rag2_warm_demo.py`: cold-start warm-up + 6 канон-вопросов БЕЗ filters (=src=all,
  дефолт фронта). Формулировки ДОСЛОВНО = `DEMO_PRESETS` фронта (`web/src/screens/AnalystChat.tsx`)
  → клик демо-чипа = кэш-хит. Лид: `python3 scripts/rag2_warm_demo.py --base-url … --token <iva>`.

### 2. Компактный вход в промпт синтеза
- `render_rag2_context_compact` (`domain/rag2.py`): top-5 hits, тела ≤700 симв, ≤4500 суммарно.
  Нумерация [n] идентична полному рендеру → цитаты бьются с citations. В `context`/UI — полный
  рендер (НЕ тронут). Длина входа ~линейна времени генерации triva.

### 3. Короткая топология-саммари
- `infrastructure/rag2/topology.py`: `_MAX_CONTAINERS` 6→5; `_container_line` = «имя — владелец
  (+архриски)» (убраны tech/линии/релизы/компоненты); cap ≤800 симв (`_cap_summary`), env
  `HELM_RAG2_TOPOLOGY_MAX_CHARS`.

### 4. Параллелизация + под-таймаут топологии
- retrieval‖topology‖tests через `asyncio.gather(return_exceptions=True)` вместо цепочки.
- `_build_topology` в `asyncio.wait_for(topology_timeout_s)`, env `HELM_RAG2_TOPOLOGY_TIMEOUT_S`=8с →
  не успела → `topology=None`, синтез идёт без неё.

### 5. Параметры генерации
- `rag2_synth_max_tokens` 900→**550** (env override; поднять до 650 если режет рубрики).
- `rag2_synth_timeout_s` 25→**35с** (умеренный крайний guard).
- Новые поля config/Rag2Config: topology_timeout_s, synth_cache_enabled, synth_cache_ttl_s.

## Диагностика (БЕЗ слепых правок — прод-данных в worktree нет)
- **tests=null (Звонки, 96 ТК):** матч детерминирован. `by_jira` снапшота строится в
  `ingest/allure_snapshot.py` ТОЛЬКО из `RequirementJira.jira_key LIKE 'IVAONE-%'`; бридж берёт те
  же ключи. Кандидаты корня (проверить на проде): (а) Jira-ключи требования «Звонки/IVA CS» в
  ДРУГОМ проекте (не IVAONE-*) → фильтр ингеста их исключает → нет by_jira при наличии тестов; фикс
  = расширить фильтр + ПЕРЕ-ИНГЕСТ снапшота (территория лида). (б) component-name ≠ Feature-name в
  kmp-проекте → by_feature промах. Проверка: глянуть jira_keys/components требования Звонков и их
  наличие в `data/real/allure/allure_snapshot.json`. Правка требует пере-ингеста на проде.
- **as_of «2026-07-10»:** ЧЕСТНО — из `rag2_corpus_as_of` (env `HELM_RAG2_CORPUS_AS_OF`), реальная
  дата снапшота. Данные правда от 10.07. НЕ выдумывать 22.07. Кода менять не нужно.

## Ожидаемая латентность (лид перемерит на проде)
- **Демо-акты 1-6 (после прогрева, кэш-хит, src=all):** ~0с, triva не зовётся.
- **Живой Q7 (новый, cold path):** ret‖topo(≤8с)≈8с + синтез ~13-16с (вход ~вдвое короче + 550 ток)
  ≈ **~20-23с**, под guard 35с, без `synthesis_failed`.
- **Было→стало:** Звонки 58.5с (цепочка+30с топология+timeout) → параллель+короткий вход; SSO 32.4с
  fail (вход+900ток на грани 25с) → короткий вход+550ток укладывается.

## Тесты (числа)
- `tests/interface/test_rag2.py`: **19 passed** (было 13; +6: кэш-хит пропускает triva, source-ключ
  консистентность (all⇔None один, jira другой), провал НЕ кэшируется, флаг off, компактный промпт,
  под-таймаут топологии→None+синтез идёт).
- Широкий регресс (`rag2/topology/allure/config/service/analyst`): **410 passed**.
- `ruff check` изменённых: clean.

## Файлы
- `src/helm/interface/api/routers/rag2.py` — кэш+канон-ключ, параллель, под-таймаут, компактный вход.
- `src/helm/domain/rag2.py` — `render_rag2_context_compact`.
- `src/helm/infrastructure/rag2/topology.py` — короткая саммари + cap.
- `src/helm/config.py`, `src/helm/infrastructure/rag2/service.py` — новые поля/дефолты.
- `scripts/rag2_warm_demo.py` — прогрев кэша (новый, формулировки = пресеты фронта).
- `tests/interface/test_rag2.py` — 6 новых тестов.

## Для лида (после ревью)
1. Деплой ветки, env `HELM_RAG2_SYNTH_CACHE_ENABLED` on.
2. Прогреть: `python3 scripts/rag2_warm_demo.py --base-url https://helm.tacticum.ru --token <iva>`.
3. Перемерить (акты из кэша ≈0с; Q7 живой ~20-23с). Повторить прогрев за 5 мин до старта.
4. tests=null Звонков — решить по диагностике (пере-ингест снапшота, если нужен).
