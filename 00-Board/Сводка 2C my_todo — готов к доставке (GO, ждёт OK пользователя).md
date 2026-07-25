---
title: Сводка 2C my_todo — готов к доставке (GO, ждёт OK пользователя)
type: report
permalink: tacticum/00-board/svodka-2-c-my-todo-gotov-k-dostavke-go-zhdiot-ok-polzovatelia
status: merged
role: lead (тимлид)
date: 2026-07-21
autonomy: 'off'
tags:
- summary
- my-todo
- helm
- delivery
---

# 2C my_todo — сделано, гейт пройден, СМЕРДЖЕН в main

## Статус
- ✅ Реализовано, верифицировано на реальных прод-данных (PASS оба подопытных), гейт controller GO.
- ✅ Запушено `feat/my-todo`, PR смерджен пользователем в main. Worktree убран.
- ⏸ **Прод:** тул появится на `helm.tacticum.ru` только после редеплоя helm-сервиса. По autonomy:off — ждёт OK пользователя на деплой.

## Числа (реальные)
- Верификация (срез 2026-07-10): Васильев open 9/blocked 3/high 1; Фадин open 27/blocked 1/high 14 — совпало с независимым SQL-эталоном, matched=true.
- Тесты: 86 passed, 0 failed. ruff clean.

## Осталось
- Деплой на прод по runbook (бэкап → deploy → smoke на реальных данных → сверка) — ждёт OK.
- Follow-up (не блокер): CSV-фолбэк IdentityDirectory для точности имён.

## Связано
- [[verify-my-todo]] · [[gate-my-todo]] · [[impl-my-todo]]