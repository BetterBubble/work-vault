---
title: Журнал воркеров
type: note
status: draft
created: 2026-07-25 20:39
updated: 2026-07-25 20:39
permalink: tacticum/00-board/workers
tags:
- board
- workers
---

# Журнал воркеров

Пишется хуками SubagentStart/SubagentStop автоматически — руками не правь.
START без STOP = воркер не вернулся; повисших показывает `stack-doctor`.
Содержательный результат воркер кладёт отдельной заметкой на доску,
здесь только линия времени.

- 2026-07-25 20:39 STOP  ?            spawner=— agent=acb93811 sess=d2b374c0 ·  · ok
- 2026-07-25 20:41 STOP  general-purpose spawner=— agent=a7e943bc sess=d2b374c0 ·  · ok
- 2026-07-25 20:42 START general-purpose spawner=— agent=a7e943bc sess=d2b374c0 · 
- 2026-07-25 20:42 STOP  general-purpose spawner=— agent=a7e943bc sess=d2b374c0 · 0м · ok
- 2026-07-25 20:43 STOP  ?            spawner=— agent=a017331a sess=d2b374c0 ·  · ok
- 2026-07-25 20:43 STOP  ?            spawner=— agent=a2f72d54 sess=d2b374c0 ·  · ok
- 2026-07-25 20:45 STOP  ?            spawner=— agent=a8fa5c34 sess=d2b374c0 ·  · ok
- 2026-07-25 20:47 STOP  ?            spawner=— agent=adb281aa sess=d2b374c0 ·  · ok
