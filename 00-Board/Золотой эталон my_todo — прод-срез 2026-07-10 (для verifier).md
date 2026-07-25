---
title: Золотой эталон my_todo — прод-срез 2026-07-10 (для verifier)
type: note
permalink: tacticum/00-board/zolotoi-etalon-my-todo-prod-srez-2026-07-10-dlia-verifier
status: reference
role: lead (тимлид)
date: 2026-07-21
source: prod helm (helm-postgres-1), срез as_of=2026-07-10, read-only
tags:
- verify
- my-todo
- golden
- helm
---

# Золотой эталон my_todo — снят независимо с прода

Числа посчитаны SQL напрямую по прод-БД, по тем же предикатам, что в коде (`status_category != 'done'`, последний `as_of`, incoming-block = `t~block & d=in`, high = priority ∈ {Критический,Высокий,Blocker,Highest,Critical,High}). Не тавтология: SQL независим от кода тула.

Реальные строки этих задач выгружены в сид: `scratchpad/mytodo_seed.json` (as_of + persons + emails + tasks с links/changelog).

## Подопытный 1 — Васильев Никита · `n.vasiliev@iva.ru`
- **open (9):** IVAONE-12228, IVAONE-9602, IVAONE-9769, IVAONE-12549, IVAONE-11145, IVAONE-11489, IVAONE-12479, IVAONE-12488, IVAONE-12500
- **blocked (3):** IVAONE-12228, IVAONE-9602, IVAONE-9769
- **high (1):** IVAONE-12549

## Подопытный 2 — Фадин Евгений · `e.fadin@iva.ru`
- **open (27):** IVAONE-6521, IVAONE-10534, IVAONE-10703, IVAONE-10763, IVAONEHALF-212, IVAONEHALF-213, IVAONEHALF-214, IVAONEHALF-215, IVAONEHALF-216, IVAONEHALF-219, IVAONEHALF-220, IVAONEHALF-222, IVAONEHALF-227, IVAONEHALF-372, IVAONE-10641, IVAONE-10642, IVAONE-10643, IVAONE-10704, IVAONE-10783, IVAONE-11031, IVAONE-11358, IVAONE-12085, IVAONE-12237, IVAONE-5916, IVAONE-6625, IVAONEHALF-224, IVAONEHALF-358
- **blocked (1):** IVAONE-6521
- **high (14):** IVAONE-6521, IVAONE-10534, IVAONE-10703, IVAONE-10763, IVAONEHALF-212, IVAONEHALF-213, IVAONEHALF-214, IVAONEHALF-215, IVAONEHALF-216, IVAONEHALF-219, IVAONEHALF-220, IVAONEHALF-222, IVAONEHALF-227, IVAONEHALF-372

## Acceptance
Тул `my_todo(email)` на этих реальных строках обязан вернуть ровно: множество open-ключей == эталону; `blocked_by` непустой ровно у blocked-ключей; `high_priority=true` ровно у high-ключей; `matched=true`. Отклонение — баг.

## Связано
- [[Спека 2C my_todo — для implementer (2026-07-21)]]
- [[impl-my-todo]]