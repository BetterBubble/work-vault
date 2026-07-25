---
title: impl-rag2-conv-ui
type: report
permalink: tacticum/00-board/impl-rag2-conv-ui
status: draft
---

# impl-rag2-conv-ui

Роль: implementer. Задача: лента-диалог на `/analyst` (RAG#2) с поддержкой follow-up и сохранением мгновенности сценарных пресетов. Демо Роскосмосу.

## Где
- Worktree: `/Users/bubblemac/tacticum/helm-wt-rag2-convui`
- Ветка: `feat/rag2-conv-ui` (от main e7f075e)
- Commit: **761ea14**
- Затронуто только `web/**` (4 файла), backend не трогал.

## Что сделано
Страница уже имела ленту (turns, свежий Q&A сверху, до 5 пар). Добавил диалоговую семантику поверх:

1. **conversation_id в state** — `useState<string|null>`. Перенимается из эхо-поля каждого ответа (`answer.conversation_id`), толерантно к старому бэку (`undefined → null`).
2. **Follow-up (ручной ввод)** — форма шлёт `ask(q, { conversational: true })` → в запрос кладётся текущий `conversation_id` → backend отвечает в контексте истории. Placeholder поля меняется на «Продолжить диалог — ответ учтёт контекст выше…».
3. **Кэш-обход пресетов** — пресеты и примеры шлют `ask(text, { conversational: false })`. При `conversational=false` `sendId` принудительно `null` И `setConversationId(null)` ДО отправки → запрос уходит БЕЗ `conversation_id` → попадание в тёплый кэш (сценарные акты мгновенны). После ответа id беседы всё равно перенимается из эха, поэтому ручной follow-up можно продолжить и после пресета.
4. **Бейдж «в контексте диалога»** — на turn, ушедшем с `conversation_id` (`Turn.contextual`).
5. **Полоска состояния диалога** под полем ввода: «Идёт диалог — вопросы копят контекст» / «Новый вопрос без истории» + кнопка **«Новый диалог»** (сброс `conversation_id` + очистка ленты + очистка поля). Видна, когда есть история или накоплены turns.

## Файлы
- `web/src/types.ts` — `Rag2AnswerOut.conversation_id?: string | null` (эхо-поле).
- `web/src/api.ts` — `rag2Answer(query, { …, conversationId })` → `conversation_id` в теле POST (только если задан).
- `web/src/screens/AnalystChat.tsx` — state `conversationId`, `ask(question, { conversational })`, `resetConversation()`, разведение пресеты/примеры (fresh) vs форма (follow-up), бейдж + полоска.
- `web/src/styles.css` — `.an-q-ctx`, `.an-conv-bar`, `.an-conv-state`, `.an-conv-glyph`, `.an-conv-reset` (бирюзовый акцент `--iva-accent`).

## Как реализован кэш-обход (суть для демо)
Кэш backend ключуется на «чистый» запрос без `conversation_id`. Пресет-клик = `conversational: false` → `conversation_id` НЕ уходит в теле → cache hit → мгновенный ответ. Живой диалог (напр. Q7) идёт через ручной ввод = `conversational: true` → `conversation_id` уходит → backend копит контекст (не кэшируется, но это ожидаемо для follow-up).

## Проверка
- `tsc --noEmit` — чисто (EXIT 0). Симлинк `web/node_modules → helm/web/node_modules` создавал временно, **удалён**; в `git status` только 4 целевых файла, артефактов нет.
- Не ломал: панель синтеза, фолбэк на context, чипы-фильтры источника, цитаты, дисклеймеры (рендер turn не менялся, кроме добавленного бейджа в блок запроса).

## Не сделано (по правилам)
Не пушил, не мержил, не деплоил.
