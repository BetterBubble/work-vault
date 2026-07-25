---
title: impl-modes-lite-base-lane
type: note
permalink: tacticum/00-board/impl-modes-lite-base-lane
tags:
- draft
- lead-modes
- adr-0057
- implementer
---

# impl-modes-lite-base-lane

status: draft
Реализация implementer'а под ТЗ#2 (lead-modes): создан НОВЫЙ процесс-лейн `tacticum-lite-base` (режим лайт). Развилка A/B = **Б** (отдельный лейн, НЕ расширение fix-bug).
Worktree: `/Users/bubblemac/tacticum-worktrees/modes-workflow`, ветка `feat/workflow-modes`. autonomy off — НЕ пушено.

## Коммит
`2ab690f` — `feat(lanes): add tacticum-lite-base process-lane (lite mode)` (локально, без пуша, без AI-подписей). 6 файлов, +678 строк.

## Пути созданного (все — под `templates/tacticum-lite-base/`)
- `manifest.yaml` — base (NO depends_on), schema_version "2", version 0.1.0, ide_targets claude-code/codex=full, 2 ингредиента.
- `ingredients/commands/lite-task.md` — тонкая команда `/lite-task` (запуск лайт-цикла в top-level сессии, без спавна суб-агента; маркер `LITE_COMPLETE`).
- `ingredients/skills/lite-task-workflow/SKILL.md` — тело скилла (перенос из драфта).
- `ingredients/skills/lite-task-workflow/references/work-order-template.md` — шаблон рабочего ордера (перенос из драфта).
- `README.md` + `CHANGELOG.md` — provenance/маршрутизация/companion.

Абсолютные пути: `/Users/bubblemac/tacticum-worktrees/modes-workflow/templates/tacticum-lite-base/…`

## Манифест (ключевое)
- 2 ингредиента, distinct id (skill `lite-task-workflow` ≠ command `lite-task`), как bug-fix≠fix-bug.
- skill_spec `lite-task-workflow`: `metadata.description_trigger` — из frontmatter сверенного драфта (триггер-слова: почини/исправь/падает/крашится/не работает/NPE/баг/отрефактори/вынеси/убери дубль/мелкая доработка/добавь небольшой/по готовому макету + fix/bug/crash/broken/refactor/extract/small change; NOT-триггеры: чистые вопросы, крупные фичи→/start-task). Добавлен `metadata.assets: [references/work-order-template.md]` — скилл ссылается на reference-файл (в bugfix/research references нет, но здесь скилл его читает — задекларировал для корректной раскладки).
- command_spec `lite-task`: `metadata.name` "lite-task".
- Owner-маркер по образцу: комментарий-шапка манифеста (как у research/bugfix) + provenance-таблица в README (в research/bugfix-base отдельного `tacticum:`-маркера нет — они используют шапку-комментарий + README-provenance; воспроизведено 1:1).

## Как перенесено содержание из драфта
Источник — сверенный с оригиналом kmp-скилл (r.yarullin, коммит 86bb469): `lite-task-workflow.SKILL.draft.md` + `work-order-template.draft.md` + `ingredient-mapping.md`. Логику НЕ переписывал — перенёс.
Механика сохранена вся:
- **ок-гейт без AFK-обхода** — правки только после явного гейт-слова (ок/approved/го/делай/утверждаю/принято/go/proceed); молчание/«интересно»/уточняющий вопрос = НЕ аппрув; «сгенерил ордер и сразу начал» = запрещённый паттерн.
- **диагностика read-only на 3 типа** (bugfix→корень; refactoring→blast-radius; feature-S→проверка каждого API + апстрим до эмит-сайта).
- **test-first + FROZEN** как проверяемый факт (снапшот тест-файлов + дифф против снапшота в финале); RED-показ + «строчка-на-тест» для bugfix/feature-S; characterization зелёные до преобразования для refactoring.
- **гейт по тулсету** (plan mode+AskUserQuestion → ордер=план в ExitPlanMode; иначе печать в чат + wait) с обеими ветками; блокирующая vs неблокирующая развилка.
- **эскалация посреди работы** («ТРЕБУЕТСЯ эскалация» → новый «ок», handoff в /start-task).
- **отчёт в чат без файлов** (нет Tasks/<N>/, нет fix.md); платформенное само-ревью как обязательный принцип; «цикл сокращает процесс, а не знания зон».

**[REPO-СЛОЙ]-заглушки причёсаны** в чистые стек-агностичные формулировки в стиле bugfix-base: навигация/гарды/fragile-зона/команды сборки-теста/мультиплатформа описаны абстрактно в прозе, конкретика вынесена короткими курсивными парентезами «*(… из стек-/repo-слоя)*», не тяжёлыми блоками. Механику это не задело.

## Текст правила маршрутизации (в SKILL.md, раздел «When this lane — and when NOT», + продублировано в README)
- **`/fix-bug` — УЗКИЙ, restore-only:** наблюдаемый дефект в одном месте, поведение просто восстанавливается (краш/NPE/регресс), без изменения намеренного поведения.
- **`/lite-task` — ШИРОКИЙ вход:** bugfix + refactoring + feature-S, включая **изменение поведения** (feature-S) без нового экрана/сценария/модуля.
- **Неоднозначность fix-bug↔lite:** гейт классификации (шаг 0) разводит — чистый restore одного места→/fix-bug; изменение поведения/рефакторинг/разрастание→lite; при сомнении **спрашивает пользователя**, не выбирает молча.
- **Эскалация вверх в /start-task** по списку: новый экран/диалог, новый пользовательский сценарий, новый модуль, зависимость вне version catalog, серверный контракт, >~10 файлов / >3 модулей.
- Rule of thumb: restore одного места→/fix-bug; мелкое изменение (bugfix шире одного места / refactoring / feature-S)→/lite-task; новый экран/сценарий/модуль или архитектурное решение→/start-task. Смотреть на ОБЪЕКТ правки, не на глагол заголовка.

## ЧИСЛА ТЕСТОВ
`apps/backend/.venv/bin/pytest` (без docker), файлы: `test_manifest_schemas.py` + `test_iva_role_presets.py` + `test_role_replacement_parity.py`:
**211 passed, 0 failed** (3.41s).
Плюс прямая валидация нового манифеста и обоих ингредиентов против `manifest.v2.schema.json` / `ingredient.v1.schema.json`: **0 ошибок**.
Новый лейн — orphan (как research-base): не требует членства в роли, существующие тесты не ломает, чинить под схему ничего не пришлось.

## ⚠️ Companion-правка bugfix-base (для сигнала ГД → делает ТИМЛИД, я НЕ трогал)
Файлы `templates/tacticum-bugfix-base/ingredients/skills/bug-fix/SKILL.md` и `.../commands/fix-bug.md`.
Сейчас инвариант bug-fix: **«restore → /fix-bug, change → /start-task»** (rule of thumb SKILL.md:33-35 + scope-tripwire SKILL.md:142-152 + fix-bug.md:51-52). Нужно **сузить**: `change` (feature-S — изменение поведения без нового экрана/сценария/модуля) теперь идёт в **`/lite-task`**, а НЕ сразу в `/start-task`. Т.е. tripwire/rule-of-thumb должны развести три исхода: restore-only→остаёмся в fix-bug; small change/refactor/feature-S→предложить `/lite-task`; новый экран/сценарий/модуль/ADR→`/start-task`. Это COMPANION-правка ШАРЕНОГО лейна — по guardrail делается тимлидом ТОЛЬКО после сигнала ГД. bugfix зеркал в `_mirrors.yaml` не имеет — правка свободна от parity-теста.

## Что осталось (НЕ входит в этот лейн)
- **Врезка в роли:** добавить `tacticum-lite-base` в `depends_on` нужных dev-ролей (ROLE_LANES, тест-матрица) — отдельный шаг тимлида через ГД. Лейн создан orphan.
- **Companion-правка bugfix-base** (см. выше) — тимлид после сигнала ГД.
- **repo-слой:** заглушки [REPO-СЛОЙ] (инструмент навигации, гарды, граница fragile-зоны, enforced-механизмы, конкретные сорссеты/команды мультиплатформы, карта доменной маршрутизации) — наполняются в стек-/repo-специфике, не в генерализованном ингредиенте.
- НЕ тронуто (guardrail соблюдён): существующие лейны, `tacticum-bugfix-base`, `iva-analysis-base`, `_mirrors.yaml`, тест-матрица. Только новый каталог.
