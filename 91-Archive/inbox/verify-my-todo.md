---
title: verify-my-todo
type: note
permalink: tacticum/00-board/verify-my-todo-1
tags:
- verifier
- my-todo
- helm
archived-at: 2026-08-03 11:16
---

# verify-my-todo — прогон тула на реальном прод-срезе

status: draft
role: verifier
date: 2026-07-21
baseline: worktree `/Users/bubblemac/tacticum-worktrees/helm-my-todo`, ветка `feat/my-todo`
данные: реальный прод-срез `scratchpad/mytodo_seed.json` (as_of=2026-07-10, 2 persons / 2 emails / 36 tasks — реальные строки Фадина и Васильева)
метод: засеял эфемерную SQLite (тот же паттерн `_sqlite_factory`) EpicTask+Person+PersonEmail из сида, вызвал реальный `_my_todo(session, email)` из `src/helm/interface/api/routers/task_mgmt.py`. requirement/approval/clientbundle пустые (ветка tasks от них не зависит). Продовый код не менялся.

## Вердикт по подопытным

### Подопытный 1 — `n.vasiliev@iva.ru` (Васильев Никита) — PASS
- open: OK — got=9 == exp=9 (полное совпадение множества ключей)
- blocked_by непустой: OK — got=3 == exp=3 {IVAONE-12228, IVAONE-9602, IVAONE-9769}
- high_priority=true: OK — got=1 == exp=1 {IVAONE-12549}
- matched=true: OK
- actor_names=['Васильев Никита'], as_of=2026-07-10

### Подопытный 2 — `e.fadin@iva.ru` (Фадин Евгений) — PASS
- open: OK — got=27 == exp=27 (полное совпадение множества ключей)
- blocked_by непустой: OK — got=1 == exp=1 {IVAONE-6521}
- high_priority=true: OK — got=14 == exp=14 (эталонное high-множество)
- matched=true: OK
- actor_names=['Фадин Евгений'], as_of=2026-07-10

Дельт нет ни у одного подопытного (нет лишних, нет недостающих ключей ни по одной оси).

## Штатные тесты (регресс-контроль)
`uv run pytest tests/interface/test_analyst_mcp.py tests/interface/test_task_mgmt.py -q`
- Результат: 86 passed, 0 failed.
- 1 warning — косметический ResourceWarning (`Event loop is closed`) от teardown aiosqlite, не падение теста.

## Итог
Данные достоверны — расхождений с золотым эталоном нет. Тул `my_todo` (ветка tasks) на реальном прод-срезе воспроизводит эталон точь-в-точь по всем осям (open / blocked / high / matched) для обоих подопытных. Acceptance выполнен. Штатные регресс-тесты зелёные.

## Связано
- [[Золотой эталон my_todo — прод-срез 2026-07-10 (для verifier)]]
- [[impl-my-todo]]