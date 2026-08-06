---
title: impl-modes-b1b2m2-fix
type: note
permalink: tacticum/00-board/impl-modes-b1b2m2-fix-1
tags:
- draft
archived-at: 2026-07-31 17:27
---

# impl-modes-b1b2m2-fix

status: draft
Роль: implementer (lead-modes, ТЗ#2). Worktree: `/Users/bubblemac/tacticum-worktrees/modes-workflow`, ветка `feat/workflow-modes`. autonomy off — НЕ пушил.

Задача (GO ГД по вердикту critic): выровнять lite/research/companion-bugfix с proposal §1.2. Сделано B1, B2, M2 + companion. Тесты зелёные, 2 локальных коммита.

## Коммиты (локально, без пуша, без AI-подписей)
- `f7c3b72` feat(lite-base): align lite lane with proposal §1.2 — two types (refactoring-S / feature-S) — B1 + B2
- `7ece71f` feat(research,bugfix-base): research no-clone artifact location + bugfix restore/change split — M2 + companion

## B1 — LITE = refactoring-S + feature-S ТОЛЬКО, bugfix ВОН
Файлы `templates/tacticum-lite-base/`:
- `ingredients/skills/lite-task-workflow/SKILL.md` — из Step 0 убран тип **bugfix** (таблица типов теперь refactoring-S + feature-S). Раздел «When this lane / When NOT» переписан на новую маршрутизацию: восстановление наблюдаемого дефекта (любого размера в лимитах) → `/fix-bug` (НЕ lite); изменение структуры без смены поведения → `/lite-task` (refactoring-S); новое поведение без нового экрана/сценария → `/lite-task` (feature-S); новый экран/сценарий/модуль/ADR/зависимость вне version catalog/серверный контракт/>~10 файлов/>3 модулей → `/start-task`. Явно: «broad restore ≠ lite, остаётся /fix-bug». Убрана формулировка «bugfix broader than one spot → lite». Step 1 диагностика — «for both types» (refactoring-S→blast-radius, feature-S→API+upstream), bugfix-буллет удалён. Step 3 — удалена вся `### bugfix`-ветка вместе с её test-first; freeze-момент переписан (feature-S→после RED, refactoring-S→после green characterization). Заголовок лейна и «Why this loop exists» (пример «fix an NPE» заменён на extract/small field).
- `ingredients/commands/lite-task.md` — frontmatter + тело: «два типа», classify-шаг и execute-шаг без bugfix-ветки; добавлено «restore → /fix-bug, не lite».
- `manifest.yaml` — header-комментарии, `description` block, persona.scope, target_tasks, post_install приведены к двум типам; version `0.1.0`→`0.1.1`.
- `README.md` — секция «Маршрутизация» переписана (restore→/fix-bug любого размера; lite = refactoring-S/feature-S); genealogy «диагностика на 3 типа»→«на 2 типа»; companion-примечание обновлено под split.
- `CHANGELOG.md` — добавлена секция `[0.1.1]` (Changed: два типа + чистка триггеров), `[0.1.0]` подчищена от «bugfix/…/3 типа».

## B2 — Триггеры
- `manifest.yaml` `description_trigger` И `SKILL.md` frontmatter-description: убраны чисто-багфиксные фразы (почини/исправь/падает/крашится/не работает/NPE/баг/fix/bug/crash/broken). Оставлены: отрефактори/вынеси/убери дубль/мелкая доработка/добавь небольшой…/по готовому макету/refactor/extract/small change + добавлен **dedupe**. Проверено grep-ом: пересечения с багфикс-триггерами `bug-fix` (почини баг/regression/не работает/…) в lite больше НЕТ (оставшиеся вхождения «NPE/fix» — это routing-проза «→ /fix-bug» и распознавание verb-vs-object, не триггеры).

## M2 — RESEARCH локация артефактов при no-clone
Файлы `templates/tacticum-research-base/`:
- `ingredients/skills/research/SKILL.md` — Phase 3: явно «lane does NOT clone the target repo → артефакты (research-report + adr-draft) живут в РАБОЧЕМ ПРОСТРАНСТВЕ роли, а не в repo-`Tasks/`». Phase 4 hand-off: перенос ADR в последующий `/start-task` — **РУЧНОЙ** шаг (start-task клонирует целевой репо и принимает ADR на вход, как scenario Step 2).
- `ingredients/commands/start-research.md` — арг `$2` и `${2:-Tasks/}` аннотированы как «role working space, NO repo cloned»; шаг «Produce TWO documents» и outcome/hand-off дополнены про ручной перенос ADR.

## COMPANION bugfix-base — консистентность с B1 (минимальный дифф)
`templates/tacticum-bugfix-base/ingredients/skills/bug-fix/SKILL.md` — правка rule-of-thumb: явно «restore любого размера в пределах лимитов → /fix-bug, broad restore НЕ становится /lite-task»; change-семантика разведена по размеру (мелкое refactoring-S/feature-S → /lite-task, крупное → /start-task). Остальной lite/scope-tripwire уже был консистентен (все `/lite-task`-ссылки — про CHANGE поведения, не restore; крупный/расползающийся restore и так шёл в /start-task). fix-bug.md не менял — уже корректен.

## ⚠️ Вышел за явный список файлов B1 (сообщаю, не молча)
Тимлид в B1 перечислил SKILL/command/README/CHANGELOG. Я также правил **`references/work-order-template.md`** (asset lite-лейна, читается из Step 2 SKILL): там был Type-строка «bugfix | refactoring | feature-S», fill-rules «ALL three types» + bugfix-буллет, и **целиком bugfix-пример заполненного ордера**. Оставить = лейн сам себе противоречит (агент решил бы, что bugfix ещё валиден). Привёл к двум типам, worked-example переписал на **feature-S** (тот же домен ChatItemMapper, чтобы сохранить демонстрацию RED-потока). Если тимлид считает, что этот файл трогать не следовало — откатить точечно.

## Проверка
`pytest tests/catalog/{test_manifest_schemas,test_iva_role_presets,test_role_replacement_parity}.py` — **211 passed** (до и после правок template). Дерево чистое, 9 файлов в 2 коммитах (+171/−162).

## Не трогал (отложено ГД)
M1 (off-ramp в research), minor m1/m2, nit n1/n2. Не касался: iva-analysis-base, run-implementation, роли/ROLE_LANES, _mirrors.yaml. Push не делал (autonomy off).

## Заметки для тимлида / открытые вопросы
- Наименование типа: тимлид в ТЗ писал «refactoring-S», оригинал kmp-скилла (r.yarullin) называл средний тип просто `refactoring` (см. `docs/proposals/workflow-modes/ingredient-mapping.md` §3 CORRECTED). Я следовал ТЗ и переименовал в **refactoring-S** консистентно по всему lite-лейну (симметрично feature-S). Если истина — `refactoring` без суффикса, дай знать: rename тривиальный.
- Bump версии lite `0.1.0→0.1.1` + новая CHANGELOG-секция — сделал по образцу bugfix-base (0.1.1). Если предпочитаешь амендить 0.1.0 (лейн не релизился) — скажу как поправить.