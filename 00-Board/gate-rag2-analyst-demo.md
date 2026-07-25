---
title: gate-rag2-analyst-demo
type: note
permalink: tacticum/00-board/gate-rag2-analyst-demo
tags:
- controller
- gate
- rag2
- analyst
- demo
- roscosmos
- helm
---

# Controller-гейт: RAG#2 /analyst «мозг-план» — демо Роскосмос 16:00

**Роль:** controller (read-only). **Ветка:** `feat/rag2-analyst-sintez` (HEAD `b15e1de`), база `main` `9571b1f`. **autonomy OFF** — PR+деплой делает пользователь.

## ВЕРДИКТ: GO (условный) на предъявление пользователю для PR+деплоя
Все статические/скоуп/безопасность/целостность-гейты, которые контролёр может проверить локально — **PASS**. Единственное открытое условие: **live-acceptance на реальной triva+данных ИВА НЕ прогнан** (по дизайну — post-deploy). Это блокирующее условие для самого демо, не для деплоя. См. п.4.

## По пунктам

### 1. Гит-чистота — PASS
- Рабочее дерево чистое; ветка `feat/rag2-analyst-sintez` (не main); 4 коммита строго по задаче.
- `git diff --stat main..HEAD` = ровно 9 ожидаемых файлов, лишнего нет (нет `.env`/секретов/ключей/бинарников/`__pycache__`/`.serena`/`.DS_Store`/`.venv`/node_modules).
- AI-подписей в subject и телах коммитов НЕТ (проверено grep по claude/co-authored/generated/anthropic).

### 2. Скоуп — PASS
- `/api/rag2/context` НЕ тронут (тело функции не менялось; новый эндпоинт добавлен ПОСЛЕ неё).
- `/iva-docs`/docs_assistant (RAG#1) НЕ тронут; `application/rag2.py`, `domain/rag2.py`, `search.py` НЕ тронуты.
- **D осознанно отрезан:** `iva_docs__bge_m3_1024` НЕ подмешана в ретрив — в дифе слова `iva_docs` нет; единственное упоминание в коде — прежний комментарий в `mgmt_vectorstore.py` («helm_mgmt ИЗОЛИРОВАНА от iva_docs»), не изменён. Живой ретрив RAG#2 не тронут.
- Изменения строго в рамках A/B/C.

### 3. Достоверность/безопасность синтеза — PASS (плумбинг), но качество на реальных данных — см. п.4
- `SYNTH_PROMPT` анти-галлюцинационный: «СТРОГО ПО КОНТЕКСТУ», «нет данных в контексте» при отсутствии, цитаты [n] только на переданные источники, запрет LaTeX. (`routers/rag2.py`, конст. SYNTH_PROMPT).
- Guardrail-first: `if result.hits and not result.no_answer and ctx.llm is not None` — без hits / no_answer / несконфигуренного triva **synthesis НЕ зовётся** (`answer()`).
- Таймаут-guard: `asyncio.wait_for(run_in_threadpool(ctx.llm.generate...), timeout=ctx.config.synth_timeout_s)` (`_synthesize`), дефолт 25с (`config.py: rag2_synth_timeout_s=25.0`).
- Любая ошибка/таймаут/пустой ответ → `synthesis_failed=true`, `synthesis=null`, **HTTP 200** (retrieval валиден); ретрив-сбой → 502, несконфигурен → 503. Никогда 5xx на сбое синтеза, не зависает.
- Про тесты в промпте — только покрытие, без «зелёных %».
- **Юнит-тесты реальны и предметны** (не тавтологии) — `tests/interface/test_rag2.py`: успех-синтез (calls==1), guardrail-no-hits (calls==0), ошибка→failed+fallback+200, **таймаут через реальный sleep(0.3) vs timeout 0.05**→failed+200, llm=None→как /context, 503, 422. Независимо прогнано: **13 passed**; смежные 3 сюиты **80 passed**; ruff clean; компиляция OK.

### 4. Целостность кода — PASS
- Битых/полу-записанных символов НЕТ: `topology.py` полностью присутствует (158 стр.), компилируется; все референсы резолвятся — `TrivaLlm(client, temperature=, max_tokens=)`/`generate(system=,user=)`, `sanitize_markdown` (docs.py:170), `_assistant_context`/`_with_session`/`_fake_request`/`_unique`/`_enrich_containers`/`_allure_coverage`/`_requirement_test_bridge` (analyst_server.py), `cio.requirement_detail`, `retriever.retrieve_rids`, `for_requirement`. Сигнатуры совпадают с вызовами.
- `_answer`/`_require_context` (используемые новым эндпоинтом) существуют.
- `analyst_server.configure(app)` вызывается безусловно в lifespan (`main.py:68`) → B/C реально работоспособны на проде (при `assistant_enabled`+session_factory), иначе fail-soft None.
- **МИНОР (не блокер):** `_build_topology`/`_build_tests` вызываются до guardrail — при no_answer/без hits они всё равно дёргают retriever. Не безопасность (triva не зовётся, fail-soft), лишь мелкая неэффективность.

### 5. Frontend — PASS
- Фолбэк без регрессии: `hasSynthesis()`/`synthesisUnavailable()`/`nonEmptyStr()` толерантны — отсутствие `synthesis`/`topology`/`tests` (undefined, старый бэк) НЕ роняет рендер; при пустом/упавшем синтезе — тихий бейдж + рендер `context` как раньше (`AnalystChat.tsx`).
- `topology`/`tests` рендерятся как строки через `StructInsert`+`AnswerText`.
- `Rag2AnswerOut extends Rag2ContextOut` (types.ts) — поля синтеза опциональны/nullable. `Petal` определён локально (стр.55), не сломан. `rag2Context` в api.ts сохранён. CSS-классы (`an-synth*`,`an-struct*`,`an-synth-fallback`) присутствуют в styles.css.
- ⚠️ tsc/build локально НЕ прогнан (нет node_modules) — ручное ревью типов чистое, но формальный typecheck остаётся за сборкой перед деплоем.

### 6. Память/доска — PASS
- Отчёты обоих implementer'ов на доске: `impl-rag2-analyst-answer-backend`, `impl-rag2-analyst-ui`. Досье фичи: `12-features/ficha-rag-2-analyst-mozg-plan-demo-roskosmos-16-00`. Verifier-бриф-заготовка: `verifier-brief-rag2-analyst-demo`.

## Блокирующее условие ДО демо (не гейт деплоя, но гейт демо)
**Verifier acceptance на реальных данных НЕ выполнен.** На доске — только бриф-заготовка, отдельного отчёта с результатами нет. Юнит-тесты мокают triva (`_FakeLlm`) → проверяют плумбинг/безопасность, но НЕ качество синтеза/анти-галлюцинации на живой triva и не 6 демо-актов. Backend-implementer это сам флагнул. Live-смоук здесь не запустить (нет `iva_llm_*`/туннеля 127.0.0.1:8790 локально) — по дизайну verifier-брифа прогон post-deploy на дев/прод-данных.

**Пользователю/verifier после деплоя, ДО демо — обязательно (из критериев приёмки плана):**
1. 6 демо-вопросов (акты) на tenant=iva → синтез с [n], «как сделать» подтягивает топологию; проверить отсутствие галлюцинаций.
2. Тесты (C) — только счётчики, без «зелёных %» (кроме реально прогнанных Звонки 40% / Почта 3/3).
3. Q7 (живое отклонение) — синтез держится.
4. triva-нагрузка + искусственный таймаут → фолбэк на context, не 5xx, не зависание.
5. `/iva-docs` жив; `/context` не сломан; вход tenant=iva у презентующего.

Если live-прогон падает/синтез галлюцинирует — **NO-GO на демо** независимо от статических гейтов.

## Итог
Статические гейты контролёра — **GO**. Пользователь может идти в PR+деплой. Демо разрешать только после успешного live-acceptance (п. выше).
