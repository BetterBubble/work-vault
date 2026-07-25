---
title: report-Р-2 golden hit@3 контрактные кейсы
type: report
permalink: tacticum/00-board/report-r-2-golden-hit-3-kontraktnye-keisy
status: draft
role: verifier
repo: /Users/bubblemac/tacticum/helm @ main eaf10f8
date: 2026-07-20
tags:
- rag2
- verifier
- acceptance
- golden
- contracts
---

# Р-2: golden hit@3 на контрактных кейсах — проверка

## TL;DR
Приёмка распадается на два несовпадающих механизма. «Контрактный golden» в репо существует и проходит на РЕАЛЬНЫХ данных JUMP/REST — но это **детерминированный матчер реестра** (found + top-1), а НЕ retrieval-харнесс с hit@3/below_noise_floor. В retrieval-golden RAG#2 контрактных кейсов **0**. hit@3 локально снять нельзя — нужен бэкенд на helm.

## 1. Golden: числа
- `tests/data/rag2_golden.json` — 68 кейсов, категории роутинга (реализация-требования, статус-релиз, архитектура и т.д.). expected_keys=0, контрактных=0.
- `tests/data/rag2_golden_profile.json` (дефолт харнесса) — 25 кейсов, analyst/support. **expected_keys заполнены лишь у 9**, ideal_answer/facts=0, контрактных=0.
- Контрактные запросы живут в ДРУГОМ месте — детерминированные golden-тесты реестра:
  - `tests/domain/test_contract_registry_golden.py` — JUMP, 8 кейсов (6 позитивных / 2 негатива-шум). Данные: `tests/data/contracts/jump-subset.commands.json` (12 реальных команд).
  - `tests/domain/test_api_registry_golden.py` — REST, 11 кейсов (7 позитивных / 4 негатива-шум).
  - Итого контрактных/API запросов: **19 (>10)**. Примеры: «отзыв письма в JUMP»→messageRevoke; «удалить сообщение»→messageDelete; «создать конференцию»→createConference; «забронировать переговорку»→not_found (шум); «messageSync»→not_found в REST (шум).

## 2. Как гоняется
- retrieval-харнесс: `python -m helm.eval.rag2_eval [--golden PATH] [--k 5] [--live] [--out ...]`. Full pipeline требует `Rag2Orchestrator` = боевой ретрив (Qdrant 10.16.0.19:6333 / Meili). hit@k считается против `expected_keys`; below_noise_floor тегируется в `application/rag2.py::apply_noise_floor`, режется в `interface/mcp/analyst_server.py` (строки 1327-1333).
- контрактные golden: обычный pytest, бэкенд не нужен (subset в CI всегда; полный корпус 101 команда — auto-skip если нет `/data/real/contracts`).

## 3. Прогон (реально снятое)
- Детерминированные контрактные golden: `pytest tests/domain/test_contract_registry_golden.py tests/domain/test_api_registry_golden.py` → **22 passed, 19 skipped** (skip = полный корпус, /data отсутствует). Реальный JUMP-subset + REST-subset проходят, включая негативы (шум→not_found).
- rag2_eval route-only (роутер — чистая функция, без бэкенда): 25 кейсов, route_mode/structural/temporal/conf_body acc = **1.0** по всем срезам.
- rag2_eval full-run локально: авто-фолбэк в route-only — «RAG#2 не сконфигурирован». **hit@3 локально СНЯТЬ НЕЛЬЗЯ.**

## 4. Команда для helm (снять hit@3 на профильном golden)
```
docker exec helm-helm-1 python -m helm.eval.rag2_eval \
    --golden /app/rag2_golden_profile.json --k 5 --out /app/rag2_eval_run.json
```
Оговорка: даст hit@3 максимум по 9 кейсам с expected_keys; контрактных кейсов там нет.

## 5. Вердикт по Р-2
**Частично / зависит от трактовки — эскалация тимлиду.**
- Если «контрактный golden» = детерминированный реестр (found+top-1 на реальных JUMP/REST): **ЗАКРЫТА**. ≥10 запросов (19), реальные данные, шум (негативы) отсекается, top-1 строже hit@3.
- Если приёмка буквальна — «контрактные кейсы В retrieval-golden RAG#2 с hit@3 и тегом below_noise_floor»: **НЕ ЗАКРЫТА**. В profile/68-golden 0 контрактных кейсов; расширение на ≥10 контрактных запросов в retrieval-golden по факту НЕ сделано; hit@3 не снят (нужен helm).
- Механизм отсечения шума (below_noise_floor/confidence) присутствует в retrieval-пути, но НИ ОДНИМ контрактным кейсом не покрыт (контракты идут детерминированным путём, не через embedding-ретрив — их нет в RAG#2-индексе).

Нужно решение тимлида/ГД: какой из механизмов считается носителем приёмки Р-2.