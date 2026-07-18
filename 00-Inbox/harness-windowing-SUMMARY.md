---
title: rag2_harness — оконный (windowed) стриминговый ингест (worker-report)
tags:
- helm
- rag2
- harness
- worker-report
date: 2026-07-15
permalink: tacticum/00-inbox/harness-windowing-summary-1
---

# rag2_harness windowing — отчёт implementer'а

## Итог
Готово. Реализован оконный стриминговый ингест в `run_container_pipeline`, чтобы
полноглубинный проход без лимита не держал весь контейнер в памяти (устраняет OOM
на IVAONE 11 694 задачи / хост 3.8 ГБ). Тесты/ruff/mypy — зелёные.

- **Ветка:** `feat/rag2-harness-windowing`
- **Финальный SHA:** `62c01ad816500ccb2547e78939c0dda1c8bb7267`
- **Worktree:** `~/tacticum-worktrees/helm-harness-windowing`
- **База:** `origin/main` = `1c32786` (НЕ 0eadc50, см. «Отклонения»)
- **НЕ запушено, merge не делался** (по протоколу — решение пользователя).

## Что изменено

### `src/helm/ingest/rag2_harness.py`
- `ContainerProgress` (+3 строки): новое поле `partial: dict[str, int] = {}` —
  resume ВНУТРИ контейнера (контейнер → следующий offset докачки). Default `{}` →
  старые progress-файлы без поля читаются без ошибок (Pydantic).
- Новый хелпер `_seed_expected_from_windows(container_dir, *, source, before_offset)`
  — при resume собирает expected_ids из уже записанных `win_{offset}/items.jsonl`
  (offset < before_offset), чтобы итоговая cumulative-валидация после докачки была
  честной (эти окна уже в коллекции, но их id иначе потерялись бы из `expected`).
- `run_container_pipeline`: новый kw-параметр `window_size: int | None = None`.
  - `window_size is None` → **прежнее поведение дословно** (весь контейнер одним
    `extract_*_container` без start_at/start → общий `container_dir/items.jsonl` →
    один ингест каталога). Оставлен inline-вызовом (не через хелпер), чтобы жёсткий
    mock существующего теста не сломался.
  - `window_size` задан → цикл окнами: `extract_*(limit=window_size, start_at/start=offset)`
    → пустое окно (`win_count==0`) обрывает цикл без ингеста; иначе пишем
    `container_dir/win_{offset}/items.jsonl`, аккумулируем `expected_ids` (set),
    ингестим ТОЛЬКО этот подкаталог, двигаем `progress.partial[container]=offset+window_size`
    + `save_progress`, освобождаем окно (`del kept, rejected`); `win_count < window_size`
    → обрыв (конец контейнера), иначе `offset += window_size`.
  - Общий хвост обеих веток: `scroll_distinct_payload_values` (фильтр по project/space —
    cumulative) → `validate_indexed_vs_extracted(expected_ids, indexed_ids)` → тот же лог
    «контейнер X: извлечено N / проиндексировано M ✓/⚠» → `done.append` +
    `partial.pop(container, None)` + save.
  - `ingest_result` в оконном режиме = `{"windows": N, "results": [...]}`; в прежнем —
    как раньше (dict одного ингеста). `kept`/`rejected`/`extracted_total` — суммы по окнам.
- Внутренние хелперы `_extract`/`_ingest` (замыкаются на стабильные параметры, `container`
  передаётся аргументом — B023-safe). `_extract` используется только оконным путём.
- `_cmd_run` + argparse: новый флаг `--window-size` (type=int, default=None) и проброс
  `window_size=args.window_size`.

### `tests/ingest/test_rag2_harness.py` (+5 тестов, ~233 строки)
- `test_run_container_pipeline_windowed_streams_and_stops_on_partial` — window_size=2,
  окна [2],[2],[1]: (a) ингест на КАЖДОМ окне (3 раза, подкаталоги win_0/win_2/win_4),
  (b) extract с offset 0,2,4 и стоп на неполном, (c) expected=5 аккумулированы, ok,
  (d) partial продвигался, контейнер в done, partial очищен.
- `test_run_container_pipeline_windowed_stops_on_empty_window` — точное кратное: последнее
  окно возвращает 0 → 1 ингест (пустое окно не ингестится).
- `test_run_container_pipeline_windowed_resumes_mid_container` — partial={IVAONE:2},
  win_0 предзаписан на диск → extract стартует с offset=2 (окно 0 не переизвлекается),
  expected сидируется из win_0 → cumulative=3, ok, done, partial очищен.
- `test_run_container_pipeline_window_none_is_single_pass` — регресс: один extract,
  один ингест корневого каталога, общий items.jsonl, никаких win_*.
- `test_container_progress_partial_defaults_backcompat` — старый JSON без `partial` читается.

## Результаты проверок
- `uv run pytest tests/ingest/test_rag2_harness.py -q` → **33 passed** (было 28, +5 новых).
- `uv run ruff check` на обоих файлах → **All checks passed**.
- `uv run mypy` на обоих файлах → **Success: no issues found in 2 source files**.

## Отклонения от спеки / решения
1. **База origin/main сдвинулась:** лид указал `0eadc50`, но origin/main уже =
   `1c32786` (влит PR #50 fix/rerank-client-sort). `git diff 0eadc50..1c32786` по
   `rag2_harness.py` и его тесту — ПУСТ (PR#50 их не трогает). Базировался на текущем
   origin/main `1c32786`. Если нужна база строго от 0eadc50 — скажи, переберу.
2. **Формат `ingest_result` в оконном режиме** = `{"windows": N, "results": [...]}`
   (в спеке формат не задан; тесты/‎`_cmd_run` этот ключ не ассертят — проверяется только
   `validate_indexed_vs_extracted.ok`). Если нужен другой агрегат — правится тривиально.
3. **Сид expected при resume (`_seed_expected_from_windows`)** — сверх минимума спеки:
   без него после докачки `expected` в отчёте недосчитывал бы id уже залитых окон
   (валидация не падала бы — scroll cumulative, missing пуст — но число «извлечено» врало
   бы). Читаю id из уже записанных `win_*/items.jsonl`. Если считаешь лишним — уберу.
4. **Прежняя вет\ка вызывает extract inline** (не через `_extract`), т.к. существующий
   тест `test_run_container_pipeline_jira_two_containers` мокает extract жёсткой сигнатурой
   без `start_at`. Так поведение прежнего пути байт-в-байт совпадает со старым.

## Открытые вопросы
- Подтверди базу (п.1) и формат ingest_result (п.2) — если ок, вопросов нет.