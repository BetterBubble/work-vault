---
title: impl-rag2-chips-v2
type: report
permalink: tacticum/00-board/impl-rag2-chips-v2
status: draft
role: implementer
task: rag2-chips-v2
branch: feat/rag2-chips-v2
worktree: /Users/bubblemac/tacticum/helm-wt-rag2-chipsv2
commit: e3cbcfc
tags:
- rag2
- analyst
- demo
- implementer
---

# impl-rag2-chips-v2 — чипы-пресеты /analyst на эталонные вопросы

## Что сделано
Заменил `DEMO_PRESETS` в `web/src/screens/AnalystChat.tsx` (6 старых смешанных вопросов → 10 эталонных из run-of-show). Тексты дословно синхронны с прогрев-кэшем (источник: `tacticum/00-board/demo-runbook-rag2-analyst`). Убрал старый «Календарь + Диск», который ронял Диск.

10 пресетов (label → question):
1. SSO / IVA ID
2. SSO killer
3. Федерация
4. ВКС / MCU
5. Звонки
6. Звонки: тесты
7. Диск
8. Календарь
9. Почта
10. Terra (AI)

## Вёрстка
Правок разметки/CSS не потребовалось. `.an-presets` уже `display:flex; flex-wrap:wrap; gap:8px` — 10 кнопок компактно переносятся в ~2 ряда, layout не ломается. Кнопка = короткий `label`, клик → `setQ(question)` + `ask(question)` через тот же путь `rag2Answer` → `/api/rag2/answer`. Стиль `.an-preset-chip` без изменений.

## Границы соблюдены
Только `web/src/screens/AnalystChat.tsx`. Панель синтеза, фолбэк, чипы-фильтры источника, цитаты — не тронуты. Backend не трогал.

## Проверка
- `tsc --noEmit` → EXIT=0 (чисто). Временный симлинк `web/node_modules` → `helm/web/node_modules` создан для проверки и удалён; в коммит не попал.
- `git status` перед коммитом: только `M web/src/screens/AnalystChat.tsx`.

## Коммит
`e3cbcfc` в ветке `feat/rag2-chips-v2`. Не пушил, не мержил, не деплоил.