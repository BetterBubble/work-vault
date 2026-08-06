---
title: critic-fidelity-tz2-addon
type: note
permalink: tacticum/00-board/critic-fidelity-tz2-addon-1
status: draft
tags:
- draft
- critic
- fidelity
- lead-modes
- tz2
- addon
archived-at: 2026-08-03 11:16
---

# critic/fidelity — добор полноты ТЗ#2 @ f66be51

> Отчёт critic (записан тимлидом — нет write-инструмента). Ветка feat/workflow-modes-addon.

## Вердикт
- **Пробел 1 (ADR-first): ДА** — контракт research→build замкнут с обеих сторон (research/SKILL.md:173 + start-research.md:56,59-60 ↔ start-task.md $3 + Phase 1 врезка в обоих телах, соответствует scenario Step 2).
- **Пробел 2 (2-й слой Фаза-1): ДА** — гейт пересмотра после Фазы1 в обоих телах (3 триггера PIN→lite/KB молчит→research/независимые→split, «не молча», handoff), Gate summary цел, аддитивно.
- **Пробел 3 (lite→research): ЧАСТИЧНО** — SKILL закрыт (:246-253 ветка+3 триггера, up-эскалация сохранена), НО командное тело lite-task.md НЕ отражает ветку → рассинхрон.
- Аддитивно: ДА. 2-CLI синк: ДА. Без сверх-ТЗ: в целом да (искл. report.md).

## Находки
- **MAJOR (обязательно до бандла)** — `tacticum-lite-base/…/commands/lite-task.md:61-64`: блок Escalate mid-flight покрывает только up-эскалацию; research-ветка (из SKILL:246-253) не перенесена. Команда инлайнит эскалацию → обязана отражать обе ветки. Фикс: добавить строку про STOP→/start-research (мини) при неподтверждённом корне/файлах вне ордера/verify 2-й раз, тот же handoff.
- **MINOR** — phantom `report.md`: `agents/tacticum-workflow.md:141` + `agents-codex/tacticum-workflow.toml:109` ссылаются на report.md, которого design-агент не производит (артефакты brd/adr/pin/tests + handoff.md; report.md — у run-implementation, другой агент). Фикс: убрать «зафиксируй в report.md», причину оставить в handoff.md; синхронно в оба тела.
- **MINOR** — `$3` (ADR) × гейт классификации не задан: при поданном ADR (решение принято) гейт может предложить research (:32-33) — противоречие. Фикс: в Шаг 1 start-task.md строка «если $3 задан (готовый ADR) — research исключён; классифицируй ПОЛНЫЙ vs ЛАЙТ».
- **NIT (не трогать в этом PR)** — предсущ. тройная дупликация критериев полного дизайна в bug-fix/SKILL.md (33-34/39-41/156-158). На будущее.

## ✓ Сверка аддитивности
1-й гейт классиф. (#142) цел, Phase 1-2 целы, scope-tripwire восстановления в bugfix сохранён (research лишь дополнил). Опц. мелочь (bugfix→research; refactor→полный) корректна.

## Итог
Blocker'ов нет. До бандла обязательна MAJOR (lite-task.md research-ветка); желательны 2 MINOR (report.md + $3-research-exclusion) — синхронно. После — готов к бандлу.