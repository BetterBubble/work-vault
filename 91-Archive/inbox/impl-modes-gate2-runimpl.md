---
title: impl-modes-gate2-runimpl
type: note
permalink: tacticum/00-board/impl-modes-gate2-runimpl-1
tags:
- draft
archived-at: 2026-07-31 17:27
---

# impl-modes-gate2-runimpl

status: draft
Роль: implementer (lead-modes ТЗ#2, 2-й слой). Worktree: /Users/bubblemac/tacticum-worktrees/modes-workflow, ветка feat/workflow-modes, autonomy off (не пушено).

## Задача
Встроить 2-й СЛОЙ (гейт пересмотра режима по фактам работы + handoff) в run-implementation (лейн tacticum-development-core). Аддитивно, ничего не деградировать.

## Файл
- `/Users/bubblemac/tacticum-worktrees/modes-workflow/templates/tacticum-development-core/ingredients/commands/run-implementation.md` — единственный тронутый файл. +36 строк, 0 удалений.
- Правки run-coder/run-tester/run-test-runner НЕ потребовались: run-implementation — оркестратор всех контрольных точек (план, ack-маркеры фаз, блокер, повторный verify), гейт логически принадлежит ему. Атомарные run-* оставлены как есть, чтобы не дублировать.

## Что вставлено
Две новые blockquote-подсекции между «Mandatory ack-gate sequence» и «Failure handling» + один буллет в «Failure handling». На английском — в тон существующему файлу (proposal сам предписывает «адаптируем под поверхности CLI»); формула предложения оставлена дословно по-русски, как в ТЗ.

1. **## Mode-review gate — second layer.** Контрольные точки ровно по §3: план/PIN в руках; каждый ack-маркер (PHASE_3_CODER_COMPLETE, PHASE_3_TESTER_COMPLETE, PHASE_4_RUNNER_COMPLETE); блокер воркера; повторный (2-й) провал verify по одному месту/корню.
   Триггеры (релевантные dev-циклу, из таблицы §3):
   - coder упёрся в «сначала нужен рефакторинг/инфра» → split (префикс-задача);
   - test-runner исчерпал 3 итерации по одному корню → split ВМЕСТО капитуляции;
   - вскрылось, что задача не тянется как реализация (механизм неизвестен / нужна постановка/research) → эскалация в /start-task или /start-research;
   - PIN выродился в одну стадию без UI/новых классов → понижение в /lite-task.
   - ⚠️ «NEVER end with could not do it / failed without proposing a mode switch» — лечит жалобу «агент сдался».
   - Формула предложения дословно: «Похоже, режим выбран неверно. Факт: <что обнаружено>. Предлагаю перейти в <режим>; наработки сохраню и передам. Согласен?»
   - Явно оговорено: межкомандный вызов НЕ автоматика (разведка §4) — это текстовая инструкция оркестратору предлагать смену, юзер запускает целевую команду.

2. **## Handoff on an agreed switch.** По §3: Tasks/<N>-<slug>/handoff.md (причина; что сделано — пути; что выяснено + ссылки; что НЕ подтвердилось отдельным списком; открытые вопросы). Маппинг наработок: full→lite = код вход PIN (stage 0); full→research = открытые вопросы→вопросы исследования; Фазы3-4→split = дифф в ветку/стеш + дефект-лист в ТЗ префикс-задачи. Незавершённый код фиксируется как есть (ветка/git stash), не выбрасывать/не дочищать молча. Смена в report.md (было→стало→почему).

3. **Failure handling** — добавлен буллет: перед записью любого INCOMPLETE/PARTIAL/failed сначала прогнать Mode-review gate; блокер coder / тупик test-runner → switch proposal + handoff, не «could not do it».

## Конфликт lite no-files vs handoff (D3/m3 от critic)
Решён явной оговоркой в промпте гейта: «/lite-task declares no artifact files, but handoff.md is the SINGLE handoff format across all lanes (this second layer). Writing handoff.md is an explicit EXCEPTION to that no-files rule, permitted solely for the escalation mechanism — it is not a general artifact.» Противоречия с lite-скиллом больше нет.

## Аддитивность (доказательство)
- git show --stat HEAD: 1 file changed, 36 insertions(+), 0 deletions(-). Только вставки, ноль удалений.
- Существующая таблица ack-gate sequence, порядок фаз, параллельность 1∥2, процедуры coder/tester/test-runner — не тронуты.
- Старые буллеты Failure handling сохранены дословно, добавлен один новый.
- development-core зеркал НЕ имеет: проверено `grep development-core templates/_mirrors.yaml` → NO. run-implementation не в _mirrors.yaml. Правил свободно, iva-analysis-base/start-task/lite/research/bugfix/роли/_mirrors.yaml не трогал.

## Тесты
`.venv/bin/python -m pytest tests/catalog/test_manifest_schemas.py test_iva_role_presets.py test_role_replacement_parity.py` (apps/backend) → **211 passed in 3.75s**. Зелёные.

## Коммит
`a11c68e` (ветка feat/workflow-modes, локально, НЕ пушено). Только run-implementation.md. В рабочем дереве есть чужие изменения (start-task.md, lite-task.md, SKILL.md — другие воркеры на той же ветке) — их НЕ трогал и НЕ коммитил.

## Риски / замечания
- Язык: секция на английском (тон файла), формула предложения — русская дословно по ТЗ. Если лид хочет полностью русскую или полностью англ. формулу — тривиальная правка.
- Гейт добавлен только в run-implementation (Фазы 3-5). Точки «полный/Фаза 1» из §3 (PIN=1 стадия на этапе плана, KB не отвечает, вскрылись независимые работы) частично покрываются здесь на контрольной точке «план/PIN в руках», но первичный вход Фаз 1-2 живёт в iva-analysis-base/start-task (вне моего скоупа по ТЗ — там отдельный воркер/1-й слой).
- Handoff — текстовая инструкция агенту (не код), реального механизма записи handoff.md не тестировали автоматом (тесты каталога проверяют схемы, не поведение промпта).