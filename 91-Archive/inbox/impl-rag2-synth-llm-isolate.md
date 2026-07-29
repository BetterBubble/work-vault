---
title: impl-rag2-synth-llm-isolate
type: note
permalink: tacticum/00-board/impl-rag2-synth-llm-isolate
tags:
- implementer
- rag2
- synth
- llm
- demo
- roscosmos
- draft
archived-at: 2026-07-29 18:12
---

# impl-rag2-synth-llm-isolate — отдельный LLM синтеза RAG#2 (обход падения triva)

status: draft · autonomy: правки в ветку, НЕ пуш/НЕ мерж/НЕ деплой (прод-env + деплой — лид)
branch: `feat/rag2-synth-gateway` (от main 1814834) · worktree `/Users/bubblemac/tacticum/helm-wt-rag2-synthllm`
commit: **62e4e0e**

## Проблема
vLLM `triva` (`10.0.196.12:8004`) лёг в контуре ИВА → синтез `/api/rag2/answer` (RAG#2) не
работает. Синтез RAG#2 и синтез RAG#1 (`docs_assistant`, `/docs/ask`) делят ОДИН общий
конфиг `iva_llm_*`. Менять общий env нельзя — затронет RAG#1.

## Что сделано
Добавлен ОТДЕЛЬНЫЙ конфиг LLM синтеза RAG#2 с фолбэком на общий `iva_llm_*`. Задан env
для RAG#2 → синтез (ТОЛЬКО он) уходит на gateway-модель `tacticum/smart`
(`https://llm.cifragen.ru/v1`, OpenAI-совместимый, у helm есть грант). Не задан → всё как
сейчас (triva). RAG#1 не тронут.

## Файлы
- `src/helm/config.py` — 3 новые настройки (дефолт None, пусто):
  `rag2_synth_llm_base_url`, `rag2_synth_llm_api_key`, `rag2_synth_llm_model`.
- `src/helm/infrastructure/rag2/service.py`:
  - `Rag2Config` — новые поля `synth_llm_base_url/api_key/model` + маппинг в `from_settings`.
  - `Rag2Config.synth_llm_resolved` (property) — резолв с фолбэком: заданы `base_url` +
    `api_key` → берём отдельный конфиг (`model` с фолбэком на `iva_llm_model`); иначе —
    полностью `iva_llm_*` (triva).
  - `build_rag2_context` — строит `TrivaLlm`/`GatewayClient` синтеза из
    `synth_llm_resolved` вместо прямого `iva_llm_*`. Синтез-guard/таймаут/кэш работают
    поверх без изменений (тот же `TrivaLlm.generate`).
- `tests/infrastructure/test_rag2_synth_llm.py` — новый (7 тестов).

## Изоляция RAG#1 (проверено)
`docs_assistant` (`src/helm/infrastructure/docs_assistant/service.py`) читает
`settings.iva_llm_base_url/api_key/model` НАПРЯМУЮ и не касается `rag2_synth_llm_*`.
Правка полностью в пути RAG#2-синтеза.

## Тесты (числа)
- Новый файл `tests/infrastructure/test_rag2_synth_llm.py`: **7 passed** — (а) фолбэк без
  env → `iva_llm_*`; (б) override с env → отдельный конфиг; частичный конфиг (только
  base_url без key) → фолбэк; model-фолбэк на `iva_llm_model`; (в) изоляция RAG#1.
- `tests/interface/test_rag2.py` + новый: **26 passed** (регресс синтеза не сломан).
- Весь rag2-набор (`-k rag2`): **296 passed**, 1506 deselected.
- Docs-регресс RAG#1 (`-k docs`): **159 passed** — не затронут.
- ruff: clean (изменённые файлы). mypy: 0 ошибок в моих файлах (config.py,
  rag2/service.py, тест). Остальные mypy-ошибки в репо — предсуществующий baseline
  (cio.py, test_req_*), не мои.

## Env для прода (лиду прописать) — ТОЧНЫЕ имена
Чтобы увести синтез RAG#2 на gateway (обход triva):

    HELM_RAG2_SYNTH_LLM_BASE_URL=https://llm.cifragen.ru/v1
    HELM_RAG2_SYNTH_LLM_API_KEY=<грант helm на gateway>
    HELM_RAG2_SYNTH_LLM_MODEL=tacticum/smart

Не задавать все три (или оставить пустыми) → синтез RAG#2 остаётся на общем `iva_llm_*`
(triva), поведение как сейчас. Для активации override нужны минимум `BASE_URL` + `API_KEY`
(без `API_KEY` override не включается — фолбэк). `MODEL` без своего значения → фолбэк на
`HELM_IVA_LLM_MODEL`, поэтому для gateway его задавать обязательно (`tacticum/smart`).

RAG#1 (`docs_assistant`) остаётся на `HELM_IVA_LLM_*` — НЕ трогать.

## Замечания лиду
- Синтез RAG#2 по данным заказчиков ИВА сейчас пойдёт через внешний gateway
  (`cifragen`), а не внутренний контур ИВА. Формально это отклонение от ADR-0003
  (генерация по данным требований — только внутренний LLM). Это осознанный временный
  обход на время недоступности triva; после восстановления triva env убрать → фолбэк
  вернёт синтез в контур. Решение об использовании обхода на демо — за пользователем/лидом.
- Гейтвей-модель уже OpenAI-совместима (не-стрим `chat`) → `TrivaLlm`/`GatewayClient`
  подходят без изменений.
