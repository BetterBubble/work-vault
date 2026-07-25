---
title: profile-latency-rag2-answer
type: note
permalink: tacticum/00-board/profile-latency-rag2-answer
tags:
- rag2
- latency
- triva
- helm
- profiling
---

# profile-latency-rag2-answer

дата: 2026-07-22 10:30 UTC · repo=helm · измерял: latency-profiler
прод: `helm.tacticum.ru`, `/opt/helm` @ **main `1814834`** (PR #90 `feat/rag2-analyst-latency` УЖЕ смержен — не sintez-ветка из задачи!). Контейнер рестартнут 10:21 UTC.

## Дисклеймер по замерам
- **triva ЛЕЖИТ весь сеанс** — `10.0.196.12:8004` (туннель 8790) даёт `000 connection refused` за 0.05с, стабильно 6+ проб за 10 мин. Контроль: соседние туннели 8791/8081 отвечают → это процесс vLLM triva мёртв, а не туннель/adp-jump.
- Прямой бенч triva (TTFT / tok/s / input-tokens / max_tokens-scaling / cold-start) **снять не смог** — модель недоступна.
- Live-эндпоинт своим curl тоже не дёрнул: ADP dev-token → **401** (HTTP API валидирует Bearer через project-hub `/resolve`, dev-token принимает только MCP-путь). Traefik access-логи выключены.
- Поэтому числа синтеза — из логов прода + docstring warm-скрипта + кода, не из моего live-прогона.

## Разложение /answer по стадиям (deployed main)
Пайплайн: `gather(retrieval ‖ topology ‖ tests)` → потом **последовательно** синтез triva.
wall = max(retrieval, topology≤8с, tests) + synth(≤35с) + overhead.

1. **Retrieval — быстрый, НЕ виноват.** В логах все ретрив-вызовы 200 OK и суб-секундные: embeddings (llm.cifragen.ru), Qdrant search (`iva_confluence`, `helm_mgmt` @ 10.16.0.19:6333), Meili (10.16.0.19:7700), rerank, intent/structural chat/completions — самые долгие `chat/completions in 0.4–0.99с`. Оценка стадии ~2–4с.
2. **Топология — capped, НЕ доминирует.** `build_topology_summary`: 2 rid (`_MAX_REQUIREMENTS=2`) × `requirement_detail`+`_enrich_containers` **последовательным циклом** (`for rid in rids`), ≤5 контейнеров. Но: под-таймаут **8с** (`rag2_topology_timeout_s=8`) и идёт **параллельно** ретриву в `gather`. Т.е. добавляет к пред-синтез-фазе не сумму, а ≤8с и только если медленнее ретрива. Латентный PR уже снял гипотезу «+30с из-за последовательной топологии»: gather + cap её обезвредили.
3. **Синтез triva — ДОМИНАНТА.** Единственный последовательный LLM-вызов ПОСЛЕ gather. `max_tokens=550`, temp 0.2, **non-stream**. По docstring warm-скрипта: **~15–25с на живой triva** (cold-start дольше). ~500 ток @ типичные on-prem vLLM ~20–40 ток/с = 13–27с. Non-stream → пользователь ждёт ВЕСЬ ответ до первого байта. Это и есть «25с+». 58с (Звонки) = больше топологии в пред-фазе + длиннее генерация + возможный ретрай-блип triva.

## Что реально видно на проде СЕЙЧАС (triva down)
6 фейлов синтеза 10:22:19→10:25:38, интервал **~39с** (прогон warm-скрипта против мёртвой triva). Каждый ≈35с — упор в `asyncio.wait_for(synth_timeout_s=35)`.
**Механизм 35с при недоступной triva** (важно): refused мгновенный, но `GatewayClient._call_with_retry` = **6 попыток** с backoff **1+2+4+8+16 = 31с** сна (+ openai-client internal retries, timeout=30с/вызов) → набегает до 35с → TimeoutError → `synthesis_failed`, HTTP 200 фолбэк. Т.е. при любом блипе/простое triva каждый /answer **тупит ~35с ни за что**.

## Доминирующая причина
**Генерация triva** (non-stream, ~500 ток при низком tok/s на on-prem GPU) — единственный тяжёлый последовательный вызов, всё остальное (<8с) на его фоне шум. Вторично: при нездоровой triva ретрай-backoff (31с) добивает /answer до ~35с.

## What-if (по коду/замерам)
- **Стриминг синтеза** — total не падает, но TTFT ~1–2с вместо спиннера на все 15–25с. Максимальный выигрыш воспринимаемой латентности для демо. Сейчас non-stream — худший вариант для UX.
- **max_tokens↓ (550→200)** — генерация ~линейна по длине → синтез ~5–10с. Дешёвый выигрыш (PR уже срезал 900→550, добить дальше).
- **Кэш (уже в проде, TTL 6ч) + warm-скрипт** — канон-демо-вопросы отдаются ~0мс. Это и есть штатная защита демо; живая triva остаётся только на импровизации.
- **Ретрай-backoff на интерактивном пути** — 31с сна на мёртвой triva это чистые потери; для синтеза нужен fail-fast (health-check / 1–2 попытки) → ~2с вместо 35с при простое.
- **Параллельная топология** — уже сделано (gather + cap 8с), больше не бутылочное горло.
- **Квантизация / быстрее модель / больше GPU** — поднимает tok/s → бьёт прямо в доминанту 15–25с.

## Немедленное
triva (`10.0.196.12:8004`) **лежит прямо сейчас** — на проде синтез 100% фейлит (фолбэк на контекст, ~35с ожидания). Поднять vLLM triva до любого демо.

- [ ] to:director from:latency-profiler Доминанта — генерация triva (non-stream ~500 ток @ низкий tok/s = 15–25с, всё остальное <8с и параллельно); при простое triva ретрай-backoff добивает /answer до ~35с. NB: triva СЕЙЧАС ЛЕЖИТ (10.0.196.12:8004 refused).
