---
title: report-controller-rag1-ph1-ph2-gate
type: note
permalink: tacticum/00-board/report-controller-rag1-ph1-ph2-gate-1
tags:
- rag1
- controller
- gate
- review
- helm
- clarify
- verdict
---

# Контролёр — гейт мержа RAG#1 Ф1+Ф2 (helm)

status: verdict · роль: controller (read-only, не правил/не мержил/не деплоил)
Ветки: `feat/rag1-ph1-brevity` (ca57a0d) · `feat/rag1-ph2-clarify` (2833fb8, стек поверх Ф1)
База: main = eaf10f8. Проверено на воркри `helm-wt-rag1-ph1` / `helm-wt-rag1-ph2`.

## ИТОГ: ГОТОВО К МЕРЖУ (с 2 некритичными нотами под калибровку/деплой)
Все 7 пунктов пройдены. Блокеров нет. Ноты (миграция multi-head baseline; upsert без ON CONFLICT) — не Ф2-регресс и не влияют при clarify OFF (дефолт).

## 1. Гит-чистота — OK
- ph1 от eaf10f8 (main head) ✓; ph2 от ca57a0d (ph1 head), ph1 — предок ph2 ✓. 2 коммита Ф1 + 3 Ф2.
- Мусора нет: ни .env/секретов/ключей, ни __pycache__/.DS_Store/.serena/worktree-артефактов. Воркри clean.
- `diff --stat main..ph2` — только файлы по задаче (17 файлов: src+tests+1 миграция).
- AI-подписей в коммитах НЕТ (grep clean). Сообщения содержательные, по задаче. Не в main.

## 2. Скоуп — OK
- Только утверждённое: промпт+max_tokens+параллелизация (Ф1); гейт+стор+миграция+clarify-петли (Ф2). Кэша/стриминга/follow-up НЕТ.
- Ф1a: критичные блоки промпта СОХРАНЕНЫ дословно — ЦИТАТЫ [n], anti-hallucination «не нахожу», ФОРМАТ/анти-LaTeX, PLAIN_SUFFIX (проверено чтением docs_assistant.py:61-95).
- max_tokens 1536→700 env-перекрываем (`HELM_DOCS_ANSWER_MAX_TOKENS`, env_prefix HELM_). Все τ/TTL/флаг Ф2 — env-перекрываемы.

## 3. Ф1b параллелизация — OK (1 микро-нота, не баг)
- `ThreadPoolExecutor` как контекст-менеджер → `__exit__`=shutdown(wait=True) гарантирует join потока. Утечки/незакрытого future НЕТ. Исключения Meili хранятся в future без предупреждений (concurrent.futures не варнит на неизвлечённые, в отличие от asyncio).
- Инвариант «сбой Meili → semantic-only» цел (try/except вокруг `ft_future.result()`); embed=None → []; сбой embed/Qdrant пробрасывается.
- Нота: `ft_future.cancel()` на уже стартовавшей задаче — no-op; в вырожденном embed=None случае Meili-вызов всё равно доработает (ждём на shutdown) → лишний запрос, но НЕ утечка/не некорректность. `PytestUnhandledThreadExceptionWarning` в сьюте — PRE-EXISTING baseline (`_connection_worker_thread`, есть и на main), НЕ от этого executor: `test_docs_search.py` под `-W error` чист.

## 4. optional_user на /docs/ask (отклонение от плана 2.4) — OK, приемлемо
- Контракт публичности СОХРАНЁН: аноним → principal=None → `clarify_on=False` → обычный ответ как раньше. Регресс-тест `test_docs_public_endpoints_open_without_token` зелёный (в сьюте).
- Дыры НЕТ: `optional_user` = обёртка `require_user` с проглатыванием HTTPException→None; НИКАКОГО повышения доступа не даёт, только опциональная идентичность.
- clarify включается лишь при `docs_clarify_enabled AND conversation_id AND principal AND session_factory` — при дефолтном OFF путь эндпоинта 1:1. Отклонение обосновано и раскрыто.

## 5. Миграция docs_clarify_pending — OK (1 нота)
- down_revision = `ab12cd34ef56` (requirement_commit) — РЕАЛЬНАЯ существующая ревизия, одна из голов main; Ф2 её продлевает (голов НЕ добавляет).
- Модель `DocsClarifyPending` ↔ миграция: колонки совпадают 1:1 (channel/conversation_key/original_question/clarify_question Text/ask_count/last_message_id/created_at tz/expires_at tz + UNIQUE(channel,conversation_key)=uq_docs_clarify_conv + index по expires_at).
- Postgres-валидность: `DateTime(timezone=True)`, `server_default now()`, именованный UniqueConstraint, отдельный индекс — всё корректно на PG. upgrade/downgrade симметричны (create_table+index / drop_index+drop_table).
- НОТА (baseline, не Ф2): в репо УЖЕ несколько alembic-голов на main (0a1c4e2b9d77, f9c0d1e2a3b4, ab12cd34ef56 и др.). Не введено Ф2, но деплой обязан катить через существующий процесс (`upgrade heads`/merge), не одиночный `head`.
- Гонки upsert БЕЗ ON CONFLICT — серьёзность НИЗКАЯ: UNIQUE не даёт дублей; редкий конкурентный первый insert из одного chatRoomId → IntegrityError проглатывается фоном (_process_message: лог, без ретрая, без краха), pending не дублируется и не теряется по существу. Актуально только при clarify ON (дефолт OFF). Приемлемо для MVP, раскрыто воркером. К калибровке/нагрузке — пересмотреть.

## 6. Guardrails — OK
- Двухконтурный allowlist бота НЕ тронут (в webhook diff только проброс session_factory; шаги 1-4 allowlist без изменений).
- clarify default OFF реально не меняет поведение: оба роутера гейтят по `docs_clarify_enabled`; в ask() ветка гейта под `if allow_clarify:` — при False не выполняется, ответ 1:1.
- Кап ≤2 в коде: `DOCS_CLARIFY_MAX_ASKS=2`; `allow_clarify = prior_asks < 2`; `ask_count` клампится 1..2 в upsert.

## 7. Тесты — ПРОВЕРЕНО запуском (не на доверии)
- Полный сьют ph2: **1703 passed, 31 skipped** (30.4s) — совпадает с отчётом.
- Baseline main: **1678 passed, 30 skipped** → дельта +25 passed/+1 skip = ровно новые тесты (домен-гейт 13 + clarify-ветка 6 + оркестратор 6).
- ruff на изменённых: чисто, КРОМЕ 1× E501 `models.py:1425` — PRE-EXISTING baseline (строка НЕ в диффе Ф2, подтверждено).
- mypy на 9 изменённых src-файлах: **Success, no issues**.
- Целевые Ф2-тесты (домен/clarify/docs/search/models): 75 passed.

## Рекомендация ГД
Мержить можно (ph1→main, затем ph2→main; ветки стекнуты). Перед деплоем: (1) катить миграции штатным процессом `upgrade heads`/merge (multi-head baseline); (2) clarify оставить OFF до калибровки τ на корпусе + reranker ON (иначе score=None → иммунитет, clarify не сработает). max_tokens=700 — держать под наблюдением на обрывы длинных инструкций (быстрый откат env).

## Связано
- [[report-rag1-ph1-brevity — Ф1 реализована (draft)]] · [[report-rag1-ph2-clarify]] · [[RAG-1 бот — план улучшений (краткость + уточняющие вопросы) — APPROVED]]