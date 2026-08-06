---
title: critic-modes-lanes
type: note
permalink: tacticum/00-board/critic-modes-lanes-1
status: draft
tags:
- draft
- critic
- lead-modes
- tz2
archived-at: 2026-07-31 17:27
---

# critic-modes-lanes — ревью дизайна лейнов lite/research + маршрутизация (ТЗ#2)

> Отчёт critic-агента (записан тимлидом — у агента не было write-инструмента). Read-only, ветка feat/workflow-modes. Эталон: workflow-modes-proposal.md + research-task-flow-scenario.md + kmp-original lite-task-workflow.SKILL.md.

## Вердикт
**Нужны правки до пуша.** Один BLOCKER архитектуры (двойной bugfix в /fix-bug и /lite-task) + связанный BLOCKER авто-активации скиллов. Fidelity lite→kmp и research→сценарий высокая, суть не выхолощена. Но правило маршрутизации /fix-bug ↔ /lite-task **неоднозначно и асимметрично** — багфикс на 2-9 файлов попадает в ОБА лейна. research чист по содержанию, но не определил, где живут артефакты при «без клонирования».

## BLOCKER

**B1. Двойной bugfix, критерий разведения асимметричен.**
- `lite/SKILL.md` §"When this lane" (29-35, 52-53): `/fix-bug` = «NARROW, restore-only … in one place», `/lite-task` = «bugfix broader than one spot → lite». Критерий «one spot vs broader».
- `bugfix/bug-fix/SKILL.md` §"When NOT" + rule of thumb (26-41): `/fix-bug` = «restore … route by size … scope >~10 files/>3 modules → /start-task». Про «one spot» — ни слова; многофайловая реставрация до ~10 файлов остаётся в /fix-bug.
- Итог: багфикс-реставрация на 2-9 файлов подпадает под ОБА. Тай-брейка нет.
- Расхождение с proposal §1: там «Багфикс(лайт)→/fix-bug», «Лайт-доработка→refactoring-S/feature-S» (БЕЗ bugfix в lite). §1.2 «Принятые упрощения»: bugfix остаётся в bugfix-лейне, lite = refactoring-S/feature-S. Реализованный lite Step 0 вернул bugfix первым типом — вопреки §1/§1.2 (наследие kmp-оригинала, где bugfix был в скилле).
- **Рекомендация critic:** bugfix → только /fix-bug; `/lite-task` = refactoring + feature-S; убрать bugfix из lite Step 0 и «wide entry». Три непересекающихся интента: restore→fix-bug, change-structure→lite/refactoring, add-behaviour→lite/feature-S, new screen/flow/module/ADR→start-task. Совпадает с proposal §1. Альтернатива: если owner хочет lite-грейд для багфиксов — /fix-bug мёржится в lite (не параллельно). Если bugfix остаётся в lite — критерий обязан быть дословно идентичным в обоих скиллах.

**B2. Коллизия триггеров → недетерминированная авто-активация.**
- bug-fix description и lite description/manifest пересекаются: «почини», «исправь», «падает», «не работает», «баг»/«bug», «fix», «crash»/«крашится». Два скилла с одинаковыми триггерами → авто-активация недетерминирована. Следствие B1.
- Правка: после B1 убрать из lite чисто-багфиксные фразы, оставить «отрефактори», «вынеси», «убери дубль», «мелкая доработка», «добавь небольшой…», «по готовому макету», «refactor», «extract», «small change».

## MAJOR
**M1. У /fix-bug нет off-ramp в /start-research при невоспроизводимом дефекте** (риск «агент сдался»). proposal §3 предусматривает lite→research/полный→research — ни один лейн не реализует. 2-й слой отложен (§6), для пуша допустимо отложить; минимум — в bug-fix Phase 2 и lite Step 1 добавить «не воспроизводится/механизм неизвестен → предложи /start-research».
**M2. research пишет в `Tasks/<N>/`, но лейн «без клонирования»** — где артефакты, как доезжают до /start-task (который клонирует репо). Разрыв continuity. Правка: явно указать локацию research-артефактов при no-clone (workspace роли, не repo-`Tasks/`) + что перенос ADR в /start-task ручной.

## MINOR
- **m1.** lite-триггер «по готовому макету» конфликтует с эскалацией «new screen → mockups». Оговорить: заглушка/элемент на существующем экране, не новый экран.
- **m2.** research description не перечисляет «плавающий дефект без причины» (proposal §2.2) — зона разведения с bugfix. Добавить.
- **m3.** lite Escalation «preserved as input (handoff)» vs «No artifact files»: lite не может записать `handoff.md` (proposal §3 — единый формат). Пока §3 отложен — ок; при внедрении разрешить lite исключение. (= D3 из fidelity.)

## NIT
- **n1.** lite «Completion marker» дублирует Step 4.3 mini-report; аналогично research RESEARCH_COMPLETE. Свернуть.
- **n2.** proposal §4 gate ссылается на `/start-refactor` (лейн отложен, покрыт lite) — при реализации gate резолвить в lite (тип refactoring).

## Проверено и ЧИСТО (не трогать)
- **lite→kmp:** ок-гейт без AFK (жёстче bug-fix — намеренно), test-first+FROZEN снапшот+дифф, диагностика 3 типов mandatory, эскалация посреди работы, отчёт в чат no-files, smoke в fragile «green build ≠ working subsystem», «режь артефакты не проверку». Генерализация [REPO-СЛОЙ] обязательность проверок не потеряла.
- **research→сценарий:** без клонирования; верификация каждого утверждения по коду + честное «not found»; report+ADR-draft; «no code needed» валиден; handoff research→lite/start-task ADR-first; факт-инструменты best-effort не блокируют.
- **feature-S/refactoring** уникально в lite, перекрытий нет; server-contract/new-module/>10 файлов эскалация консистентна; refactoring-кампании отложены.

## Итог
Границы refactoring/feature-S/research/start-task чёткие; **стык bugfix между /fix-bug и /lite-task не готов (B1+B2)**. До пуша: развести bugfix (реком.: bugfix→только /fix-bug, lite=refactoring+feature-S), синхронизировать критерий и триггеры, определить локацию research-артефактов (M2). M1 допустимо отложить со 2-м слоем.