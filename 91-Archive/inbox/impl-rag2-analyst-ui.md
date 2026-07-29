---
title: impl-rag2-analyst-ui
type: report
permalink: tacticum/00-board/impl-rag2-analyst-ui
tags:
- rag2
- analyst
- ui
- helm
- demo
- implementer
archived-at: 2026-07-29 18:12
---

# impl-rag2-analyst-ui — UI панель синтеза «мозг-план» (RAG#2 /analyst)

status: draft · demo Роскосмос 2026-07-22 16:00 · autonomy OFF (не пуш/не мерж/не деплой)

## Что сделано
Страница `/analyst` теперь запрашивает `POST /api/rag2/answer` и показывает синтезированный «мозг-план» СВЕРХУ, с честным фолбэком на цитируемый поиск (`context`) при пустом/упавшем синтезе — без регрессии. Реализовано строго под зафиксированный backend-контракт и уточнения координатора/техплана.

## Worktree / ветка / commit
- worktree: `/Users/bubblemac/tacticum/helm-wt-rag2-analyst-ui`
- ветка: `feat/rag2-analyst-sintez-ui` (от main 9571b1f)
- commit: **3805187** `feat(rag2/analyst): панель синтеза «мозг-план» над блоком цитат`

## Изменённые файлы (только web/**)
- `web/src/types.ts` — новый `Rag2AnswerOut extends Rag2ContextOut`: `synthesis: string|null`, `synthesis_failed?: boolean`, `topology?: string|null`, `tests?: string|null`. Опциональные поля → толерантный парсинг (отсутствие поля не роняет рендер).
- `web/src/api.ts` — метод `rag2Answer(query, {k?, filters?})` → `POST /api/rag2/answer` (импорт `Rag2AnswerOut`).
- `web/src/screens/AnalystChat.tsx` — основная работа: `Turn.answer: Rag2AnswerOut`; `ask()` зовёт `rag2Answer`; индикатор загрузки «Синтезирую план… (несколько секунд)»; панель синтеза над блоком цитат; врезки topology/tests; фолбэк-бейдж; кнопка «Собрать мозг-план».
- `web/src/styles.css` — стили `.an-synth` (акцентная врезка, лево-бордер `--iva-accent`), `.an-synth-h`/`.an-synth-glyph`, `.an-synth-fallback` (тихий жёлтый пунктирный бейдж), `.an-struct`/`.an-struct-h`/`.an-struct-body`.

## Поведение (как реализовано)
- **Синтез есть** (`synthesis` непустая строка И `synthesis_failed !== true`): панель `an-synth` с заголовком «Мозг-план» + `<AnswerText>` (реюз markdown+`[n]`-рендера из DocsChat) → цитаты кликабельны и ведут на те же карточки источников (`scrollToCite`). Под панелью — компактные текстовые врезки «Топология» и «Покрытие тестами» (тоже через `AnswerText`, только при непустой строке).
- **Фолбэк** (`synthesis == null` ИЛИ `synthesis_failed == true`): рендерим прежний блок `context` (`an-answer`) — как раньше, без регрессии. Если backend ЯВНО сигналит недоступность (`synthesis_failed===true` или `synthesis===null`) — тихий бейдж «синтез недоступен — показан поиск». Если поля `synthesis` нет вовсе (старый бэк, `undefined`) — бейдж не показываем, просто `context`.
- **Сохранено как есть**: блок цитат/карточки источников, чипы-фильтры источника, структурный граф (`an-graph`), плашки свежести/деградации/неполноты + дисклеймеры, «не нашлось контекста», порядок ленты.
- **Толерантность**: `topology`/`tests` рисуются только при непустой строке (`nonEmptyStr`); при `null`/отсутствии — не показываются, не падаем.

## Учтённые уточнения координатора
- `topology`/`tests` — **строки** (`string|null`), НЕ объекты. Первичную объектную версию заменил на текстовый рендер через `AnswerText`.
- Точный текст индикатора: «Синтезирую план… (несколько секунд)».

## Проверка сборки/типизации
`tsc --noEmit` — **без ошибок** (TSC_EXIT=0). node_modules в worktree нет; прогнал через временный симлинк на `helm/web/node_modules` (симлинк после проверки удалён, в коммит не попал). `vite build` не гонял (нет установленных зависимостей в worktree).

## Не делал (по autonomy OFF)
Не пушил, не мержил, не деплоил. Backend (`/api/rag2/answer`, synthesis triva) — параллельный worktree, не трогал.
