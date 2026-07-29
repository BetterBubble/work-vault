---
title: report-rag1-ph2-clarify
type: report
permalink: tacticum/00-board/report-rag1-ph2-clarify
status: draft
role: implementer
repo: /Users/bubblemac/tacticum/helm-wt-rag1-ph2 (feat/rag1-ph2-clarify)
tags:
- rag1
- bot
- docs_ask
- clarify
- helm
- report
- draft
archived-at: 2026-07-29 18:12
---

# RAG#1 Ф2 (уточняющие вопросы) — отчёт реализации

Реализована Фаза 2 плана (clarify). Ветка **`feat/rag1-ph2-clarify`** стекнута НА Ф1
(`feat/rag1-ph1-brevity`), worktree **`/Users/bubblemac/tacticum/helm-wt-rag1-ph2`**.
НЕ мержил, НЕ пушил, НЕ деплоил. 3 коммита поверх Ф1.

## Коммиты (ahead of Ф1)
- `545edbb` Ф2.0/2.1 — confidence-гейт + clarify-развилка в `DocsAssistant.ask`
- `52e8024` Ф2.2 — стор `docs_clarify_pending` (модель, миграция, репо)
- `2833fb8` Ф2.3/2.4/2.5 — clarify-петля бот+веб + оркестратор

## Изменённые/новые файлы
Новые:
- `src/helm/interface/api/docs_clarify.py` — оркестратор `resolve_with_clarify` (общий бот+веб)
- `alembic/versions/d2c3b4a5e6f7_docs_clarify_pending.py` — миграция (down_revision = head `ab12cd34ef56`)
- `tests/domain/test_docs_clarify_gate.py`, `tests/interface/test_docs_clarify.py`

Правки:
- `domain/docs.py` — `decide_retrieval_action` + `build_clarify_question` (детерминированно, без LLM)
- `config.py` — `docs_clarify_enabled=False`, `_tau_answer=0.55`, `_tau_floor=0.30`, `_ttl_seconds=900` (env-перекрываемы)
- `application/docs_assistant.py` — `DocsAnswer` +clarify/+clarify_question; `ask(..., allow_clarify=False)` развилка ПОСЛЕ rerank+cap, ДО LLM
- `infrastructure/db/models.py` — `DocsClarifyPending` + UNIQUE(channel, conversation_key)
- `infrastructure/db/repository.py` — get_active/upsert/clear/purge_expired_clarify + clarify_expires_at
- `infrastructure/docs_assistant/service.py` — пороги τ в контекст/фабрику
- `interface/api/auth.py` — `optional_user` (не бросает 401; для публичного /docs)
- `interface/api/routers/bot_support.py` — фон clarify-петля с короткой сессией из app.state
- `interface/api/routers/docs.py` — опц. `conversation_id`, DocsAskOut +clarify, web-петля

## Дизайн (кратко)
- **Гейт** по верхнему скору набора (0..1, симметрия с rag2): ≥τ_answer→answer; <τ_floor→not_found (без LLM); между→clarify (при allow_clarify и различимых кандидатах). `score=None` (реранк off) → иммунитет, "answer" (как confidence=None в rag2).
- **Уточнение** — дизъюнкция по различающимся product→section→heading→title (cap 3); неотличимы → "" → гейт отвечает.
- **Стор**: pending по (channel, conversation_key). Бот key=chatRoomId, веб key=email. TTL `expires_at`, кап `ask_count≤2`, идемпотентность `last_message_id`.
- **Петля**: purge TTL → get_active → дедуп по messageId (дубль→None, не отвечаем) → склейка ответа на уточнение → ask с allow_clarify=(ask_count<2) → clarify: upsert(inc); иначе clear.
- **Guardrails соблюдены**: clarify default OFF (каналы не прокидывают allow_clarify → гейт не вызывается, поведение уверенных ответов 1:1); кап ≤2 жёстко; двухконтурный allowlist бота не тронут; известные-gaps гейт остаётся первым; скоуп не расширял (без кэша/стриминга/follow-up/LLM-генерации вопроса).

## Тесты
- **pytest: 1703 passed, 31 skipped** (весь сьют). Новые: домен-гейт (13), clarify-ветка ask (6), оркестратор на sqlite (6) — сценарии приёмки: неоднозначный→1 вопрос→склеенный ответ, кап 2, TTL истёк, уверенный не тронут, дубль messageId, purge.
- **ruff/mypy на моих файлах — clean.** Остаточные ruff(E501)/mypy-ошибки в репо (cio.py, gateway.py, requirements.py, models.py:1425 и т.п.) — ПРЕДСУЩЕСТВУЮЩИЙ baseline Ф1, не мои файлы (проверено на ветке Ф1).
- **Миграция**: полный `alembic upgrade head` на sqlite невозможен (ранняя миграция цепочки использует ALTER constraint, не поддерж. sqlite; прод=Postgres, харнесс тестов = `create_all`). МОЯ миграция проверена изолированно (stamp prior → upgrade +1 → downgrade -1 на sqlite): create_table+index соответствует модели, downgrade чисто удаляет.

## Отклонение от плана (2.4) — требует внимания ревьюера
План 2.4 просил «добавить require_user+get_session» на `/docs/ask`. **Не сделал жёстко**: `/docs/ask` — ПУБЛИЧНЫЙ по решению владельца (2026-07-15), есть регрессионный тест `test_docs_public_endpoints_open_without_token` (ответ без токена). Жёсткий `require_user` сломал бы контракт и тест, а также нарушил бы guardrail «clarify не меняет поведения». Решение: `optional_user` (не бросает 401) + ленивая сессия из `app.state.session_factory` только на clarify-пути. При выключенном clarify (дефолт) путь эндпоинта 1:1 как раньше, публичность сохранена. Если ревью хочет иначе — обсудить.

## Риски / под калибровку
- **τ не откалиброваны** — `docs_clarify_enabled=False` до калибровки на корпусе. Гейт эффективен только при активном reranker'е (`docs_rerank_enabled`, тоже off); без реранка score=None → иммунитет, clarify не сработает даже при флаге. Для реального включения: reranker ON + калибровка τ_answer/τ_floor.
- **Веб conversation_key = email** (по плану): несколько параллельных вкладок одного пользователя делят одну pending-запись (MVP-упрощение). Веб дедуп по messageId не применяется (`message_id=""`), т.к. запрос/ответ без повторной доставки.
- **Гонки/конкурентность сессий**: upsert = select-then-update/insert (как `save_docs_feedback`), UNIQUE(channel, conversation_key) — страховка от гонки. Бот-фон открывает КОРОТКУЮ сессию на сообщение. Жёсткого ON CONFLICT (диалект-специфичного) не делал ради переносимости sqlite↔postgres; при высокой конкуренции одного chatRoomId возможна редкая IntegrityError на параллельном первом upsert — фон её проглотит (лог, без ретрая). При калибровке/нагрузке стоит перепроверить.
- **К Медведеву позже** (из плана, не блокер): жёсткая привязка «ответ именно на наш вопрос» через нативный тред платформы — сейчас эвристика «следующее сообщение из chatRoomId».

Не мержить. Worktree оставлен для ревью.