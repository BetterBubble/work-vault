---
title: impl-modes-critic-verify-fix
type: note
permalink: tacticum/00-board/impl-modes-critic-verify-fix
tags:
- draft
---

# impl-modes-critic-verify-fix

status: draft
worktree: /Users/bubblemac/tacticum-worktrees/modes-workflow (ветка feat/workflow-modes)
коммит: `7f0451e8d8727bb130bf4809cfd9f6f20c426c81`
тесты: **211 passed** in 3.53s (test_manifest_schemas.py + test_iva_role_presets.py + test_role_replacement_parity.py)

3 точечные правки по вердикту critic-verify. Строго по ТЗ (proposal §3), аддитивно, минимальный дифф. NIT n1/n2 не трогал. Роли/ROLE_LANES/_mirrors/другие лейны не трогал.

## MAJOR — инверсия направления в handoff-маппинге
Путь: `templates/tacticum-development-core/ingredients/commands/run-implementation.md:73`

ДО:
> full→`/lite-task` — the mini-plan and code become the PIN input (stage 0 "already done");

ПОСЛЕ:
> full→`/lite-task` — the existing (degenerated) PIN and the already-written code become the lite work-order (lite is chat-only and has no PIN of its own);

Фикс: для даунгрейда full→lite убрана формулировка «become the PIN input (stage 0)» (у цели /lite-task PIN нет), вместо неё — «выродившийся PIN + написанный код → рабочий ордер lite». Формулировка про PIN input в этом маппинге относилась только к full→lite и была ошибочной; направление lite→full в этом списке не перечислено (оно корректно описано в lite/SKILL.md:~240), поэтому отдельной строки lite→full не добавлял. Сверено с §3 п.2 и lite/SKILL.md:~240.

## MINOR-1 — no-files-исключение одностороннее (плечо lite→вверх)
Путь: `templates/tacticum-lite-base/ingredients/skills/lite-task-workflow/SKILL.md:239-241` (секция «Escalation mid-flight»)

ДО (хвост абзаца эскалации):
> Escalation is a proposal to switch mode (not a "failed" ending): the mini-plan and code are preserved as the input into the full cycle (handoff).

ПОСЛЕ:
> Escalation is a proposal to switch mode (not a "failed" ending): the mini-plan and code are preserved as the input into the full cycle (handoff). When you escalate, write a `handoff.md` in the single cross-mode format — this is the ONE permitted file exception to the "no artifact files" rule above (consistent with the second-layer handoff format); the rule stays fully in force for ordinary lite work.

Фикс: при эскалации вверх (через /start-task, где нет секции записи handoff) теперь явно определено, что handoff.md — единственное разрешённое файловое исключение к правилу «no artifact files». Консистентно с run-implementation.md:~77 и §3 «единый формат для всех переходов». Правило «no files» для обычной работы lite не ослаблено (явно оговорено «stays fully in force for ordinary lite work»).

## MINOR-2 — «в одном конкретном месте» сужало bugfix-корзину гейта
Путь: `templates/iva-analysis-base/ingredients/commands/start-task.md:38-40`

ДО:
> - БАГФИКС: наблюдаемый дефект-реставрация в одном конкретном месте («почини», «исправь», «падает», «крашится», «не работает», один видимый дефект) → предложи `/fix-bug`. НЕ клади багфикс в `/lite-task`.

ПОСЛЕ:
> - БАГФИКС: наблюдаемый дефект-реставрация — один дефект любого охвата в пределах лимитов, НЕ составное («почини», «исправь», «падает», «крашится», «не работает», один видимый дефект) → предложи `/fix-bug`. НЕ клади багфикс в `/lite-task`.

Фикс: снято «в одном конкретном месте» (уводило широкую-но-единичную реставрацию — один дефект, много файлов — мимо /fix-bug), заменено на «один дефект любого охвата в пределах лимитов, НЕ составное». Смысл «один дефект, не составное» сохранён, гейт не деградирован. Консистентно с общим правилом лейнов «restore любого размера в пределах лимитов → /fix-bug».

## Наблюдения
- autonomy off — НЕ пушил. Коммит только локальный в feat/workflow-modes.
- Затронуты ровно 3 файла из ТЗ, ничего сверх.
