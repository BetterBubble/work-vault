---
title: impl-rag2-analyst-chips
type: report
permalink: tacticum/00-board/impl-rag2-analyst-chips
tags:
- rag2
- analyst
- frontend
- demo
- roskosmos
---

# impl-rag2-analyst-chips

status: draft
role: implementer (frontend)
worktree: /Users/bubblemac/tacticum/helm-wt-rag2-chips
branch: feat/rag2-analyst-chips (от main 9d41fdb)
commit: e996c38

## Задача
Чипы-пресеты демо-вопросов в `/analyst` (RAG#2) для демо Роскосмосу 16:00 — wow + гарантия кэш-хита.

## Что сделано
- В `AnalystChat.tsx` добавлена константа `DEMO_PRESETS` — 6 пресетов: короткая подпись + ДОСЛОВНЫЙ текст вопроса (синхронизирован с прогретым кэшем, тексты из `demo-runbook-rag2-analyst`, не менял).
- Ряд кнопок-пресетов «Демо-вопросы» отдельной группой (`.an-presets`) над чипами-фильтрами источника. Клик: `setQ(вопрос)` + `void ask(вопрос)` — тот же путь, что кнопка «Собрать мозг-план» (`api.rag2Answer` → `/api/rag2/answer`). Кнопки disabled при `busy`. `title` = полный текст вопроса.
- CSS `.an-presets` / `.an-presets-h` / `.an-preset-chip` в `styles.css` в стиле существующих `.an-*`. Визуальное отличие от чипов-фильтров: зелёный вторичный акцент `--iva-accent` (фильтры — бирюзовый `--iva-primary`) + dashed-контейнер с заголовком «Демо-вопросы».

### Тексты пресетов (label → question)
1. SSO / IVA ID → Что по SSO / IVA ID сделано, что в работе, что не начато?
2. Федерация → Что по федерации мессенджера сделано, что в работе, что не начато?
3. ВКС / MCU → Что по ВКС / MCU (видеоконференции) сделано, что в работе, что не начато?
4. Звонки → Что по звонкам / IVA CS (SIP-телефония) сделано, что в работе, что не начато?
5. Календарь + Диск → Что по Календарю и Диску сделано, что в работе, что не начато?
6. Почта + Terra → Что по Почте (Phoenix / JUMP) и Terra сделано, что в работе, что не начато?

## Файлы
- /Users/bubblemac/tacticum/helm-wt-rag2-chips/web/src/screens/AnalystChat.tsx (+34)
- /Users/bubblemac/tacticum/helm-wt-rag2-chips/web/src/styles.css (+38)

## Проверка
- tsc --noEmit → exit 0 (временный симлинк на helm/web/node_modules, удалён, в коммит не попал).
- git diff --stat: только 2 целевых файла, node_modules не в индексе.
- Границы соблюдены: только `web/**`, backend/`/context`/панель синтеза/фолбэк/чипы-фильтры/цитаты/дисклеймеры не тронуты.

## Не сделано / примечания
- Не пушил, не мержил, не деплоил (по инструкции).
- Пресеты работают в общем потоке `ask` → уважают активный фильтр источника (`source`). Для демо фильтр по умолчанию «Все источники» — кэш прогревался, вероятно, без фильтра; если warmup был с конкретным source, тимлиду стоит свериться (кандидат на промах кэша). В остальном — дословный текст гарантирует хит.
