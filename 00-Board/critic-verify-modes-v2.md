---
title: critic-verify-modes-v2
type: note
permalink: tacticum/00-board/critic-verify-modes-v2
status: draft
tags:
- draft
- critic
- lead-modes
- tz2
- verify
---

# critic-verify (v2) — закрытие B1/B2/M2 + новые гейты

> Отчёт critic-verify (записан тимлидом — у агента нет write-инструмента). Ветка feat/workflow-modes.

## Вердикт
- **B1 / B2 / M2 закрыты: ДА (все три, чисто).**
- **1-й гейт (start-task): верен.** 2-й слой (run-implementation): почти — 1 MAJOR + 2 MINOR.

## Закрытие
- **B1** ✓ — lite = refactoring-S + feature-S только; bugfix убран из Step 0/маршрутизации; restore любого размера → /fix-bug во всех местах (lite SKILL, bugfix companion, README, manifest, gate start-task). Три интента непересекающиеся, = proposal §1/§1.2.
- **B2** ✓ — багфикс-фразы удалены из lite триггеров; пересечения с bug-fix нет; авто-активация детерминирована.
- **M2** ✓ — research: артефакты в рабочем пространстве роли (не repo-Tasks/), перенос ADR в /start-task — ручной; продублировано в SKILL + start-research.

## 1-й гейт (start-task.md) — верен ✓
Шаг 0 достаточность→3 вопроса; Шаг 1 классификация по приоритету (research→composite→refactoring-S→lite/feature-S→bugfix→полный); Шаг 2 предложение+подтверждение (не молча); «чему не доверять»; проверка состава роли. Маршрут = §1.2. Аддитивно (полный конвейер сохранён). Резолвит рефакторинг в /lite-task (не отложенный /start-refactor) — закрывает прошлый n2.

## 2-й слой (run-implementation.md) — правки
- **MAJOR** (стр. 73): маппинг **full→/lite-task** ошибочно «code → PIN input (stage 0)» — это направление lite→full; у lite нет PIN. → full→lite: «выродившийся PIN + код → рабочий ордер lite»; «вход PIN» оставить для lite→full.
- **MINOR-1** (стр. 77): no-files-исключение одностороннее — на плече lite→вверх не определён, кто пишет handoff.md. → карвнуть исключение в lite/SKILL.md (handoff.md — единственное файловое исключение при эскалации).
- **MINOR-2** (start-task.md:38-40): «в одном конкретном месте» сужает bugfix-корзину → «один наблюдаемый дефект (любого охвата в лимитах) → /fix-bug».

## NIT (не блок)
- n1: доп. split-триггер «независимая вторая задача» для Phase 1 — опц.
- n2: proposal §4 черновик устарел vs реализация (bugfix в lite-корзине, /start-refactor) — это документ Солонко (эталон), НЕ наш репо; на обновление ТЗ owner'ом, не правим.

## Итог
Прошлые BLOCKER'ы B1/B2/M2 закрыты чисто, 1-й гейт корректен, рассинхронов имён нет. **До бандла: 1 правка MAJOR** (+ 2 MINOR тем же заходом). После — готов к бандлу.
