---
title: RAG#1 бот — план улучшений (краткость + уточняющие вопросы) — APPROVED
type: report
permalink: tacticum/20-architecture/rag-1-bot-plan-uluchshenii-kratkost-utochniaiushchie-voprosy-approved
status: current
autonomy: false
tags:
- rag1
- bot
- docs_ask
- plan
- approved
- helm
- clarify
- latency
---

# RAG#1 бот — план улучшений (APPROVED 2026-07-20)

Направление: чат-бот RAG#1 по ИВА (ассистент публичной документации). План согласован ГД по делегации пользователя (он в away, эскалация — пуш). Правки бьют по ядру `DocsAssistant.ask` → эффект и на бота «Поддержка», и на веб `/docs`. Репо helm `eaf10f8`. Разведка: [[explore-rag1-docs-ask-pipeline]].

## Финальный скоуп (утверждён пользователем)
**Делаем:** Ф0 замер латентности → Ф1 краткость (промпт + `max_tokens 1536→700`) + параллельный ретрив Qdrant∥Meili → Ф2 уточняющие вопросы (кап ≤2, всё на нашей стороне, стор pending-clarify по chatRoomId с TTL) → confidence-гейт (answer/clarify/not_found) → доп: `DocsQa`→eval.
**НЕ делаем (убрано пользователем):** кэш ❌ · стриминг ❌ (модель не поддерживает; в чате невозможно) · follow-up-подсказки ❌.
**Ограничения:** clarify 1 (макс 2); двухконтурный allowlist бота сохранить; clarify default OFF до калибровки τ; не мержить/деплоить без OK пользователя.

## Ключевые файлы (path:line из разведки)
- `src/helm/application/docs_assistant.py:61-79` SYSTEM_PROMPT · `:176-226` ask (clarify-развилка) · `:121-129` DocsAnswer
- `src/helm/config.py:144` docs_answer_max_tokens (+ новые τ/TTL)
- `src/helm/infrastructure/docs_assistant/search.py:157-187` _hybrid (параллелизация)
- `src/helm/domain/docs.py` новые `decide_retrieval_action` + `build_clarify_question` (образец `domain/rag2.py:460-500` normalize_confidence/apply_noise_floor)
- `src/helm/interface/api/routers/bot_support.py` + `.../docs.py` — clarify-петли
- `infrastructure/db/models.py:2005` + `repository.py:1896` + новая миграция по образцу `alembic/versions/f1a2b3c4d5e7_docs_qa_feedback.py` — стор `docs_clarify_pending`

## Стор состояния (обход stateless-бота — БЕЗ платформы)
Таблица `docs_clarify_pending(channel, conversation_key[chatRoomId|email], original_question, clarify_question, ask_count≤2, last_message_id[идемпотентность], expires_at[TTL~900s])`, UniqueConstraint(channel, conversation_key). Следующее сообщение из того же chatRoomId = ответ на уточнение → склейка. Кап 2 → дальше отвечаем что есть.

## К Медведеву ПОЗЖЕ (не блокер)
Жёсткая привязка «это ответ именно на наш вопрос» (нативный тред платформы) — сейчас эвристика «следующее сообщение из чата», работает без платформы. Группы бот игнорит.

## Порядок для тимлида
Параллельно: A(Ф0 замер) ∥ B(Ф1a промпт+max_tokens) ∥ C(Ф1b параллелизация). Затем D(Ф2.0 метрика+decide_retrieval_action) → E(Ф2.2 стор+миграция ∥ D) → F(Ф2.1 DocsAnswer/ask) → G(бот)∥H(web). Калибровка τ между D и включением clarify. I(DocsQa→eval) — в любой момент.

## Приёмка
Ф1: замер p50/p90 до/после (embed/Qdrant/Meili/LLM), длина ответа ↓, judge-score не просел, цитаты не сломаны. Ф2: сценарии «неоднозначный→1 вопрос→ответ», «кап 2», «TTL истёк», «уверенный не тронут», «дубль messageId». e2e: `docs_ask` через MCP/REST; бот — на `/docs` гарантированно, в чате если достижим.

## Связано
- [[bot-podderzhka-webhook-rag-1-v-chat-iva-zadeploen]] · [[explore-rag1-docs-ask-pipeline]] · [[session-state]]