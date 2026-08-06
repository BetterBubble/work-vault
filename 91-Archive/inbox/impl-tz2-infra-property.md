---
title: impl-tz2-infra-property
type: note
permalink: tacticum/00-board/impl-tz2-infra-property-1
tags:
- draft
archived-at: 2026-08-03 11:16
---

# impl-tz2-infra-property

status: draft
Ветка: feat/workflow-modes-infra (worktree /Users/bubblemac/tacticum-worktrees/modes-infra)
Коммит: 1e66102
Задача: решение ГД B — микро-добор ТЗ#2, §1.2 «ИНФРА — свойство, не режим». Строго аддитивно, по §1.2 proposal + §4 Шаг3.

## Что сделано

Единственный файл-скилл: `templates/tacticum-lite-base/ingredients/skills/lite-task-workflow/SKILL.md`.
Инфра во всём скилле фигурировала РОВНО в одном месте (Step 0, было :105) — там и была неверная трактовка как повод эскалации. В теле команды `ingredients/commands/lite-task.md` инфра не упоминается — несогласованности правка не создаёт (проверено grep).

### 1. Убрано из эскалации (Step 0)
- **Было** (:104-105): `editing a *place* → lite; editing a *subsystem / lifecycle / infrastructure* → escalation.`
- **Стало**: `editing a *place* → lite; editing a *subsystem / lifecycle* → escalation. **"Infra" is NOT an escalation trigger** — it is a trait handled inside this lane (see step 3, "Infra trait").`
- Т.е. `/ infrastructure` удалено из списка поводов эскалации + добавлена явная дизамбигуация.

### 2. Добавлено как СВОЙСТВО внутри режима (Step 3, новый блок «Infra trait» после общего «frozen»-абзаца, ~:178)
Формулировка близко к §4 Шаг3 proposal, стек-агностично:
- (а) **warn about the blast radius** — возможный хвост правок на соседних платформах/модулях;
- (б) **toolchain error = subject of the work, not a blocker** — не срабатывают стоп-правила лейна; отдельно оговорено, что стоп «verify fails a 2nd time in the same place» (из секции Escalation mid-flight) на намеренно прорабатываемую toolchain-ошибку НЕ распространяется;
- явно: инфра обрабатывается ВНУТРИ lite, НЕ становится отдельным режимом и сама по себе не повод эскалации;
- ремарка про [стек/repo-слой] для конкретных платформ/модулей.

## Доказательство «инфра НЕ режим, настоящая эскалация цела»
- Новый режим НЕ создан: инфра — блок-trait внутри Step 3, отдельной таблицы типов (refactoring-S/feature-S) не трогал; frontmatter/триггеры не менял.
- **Настоящая эскалация вверх сохранена без изменений**: список Forced escalation (Step 0: новый экран/диалог, новый flow, новый модуль, зависимость вне version catalog, серверный контракт, >~10 файлов / >3 модулей) не тронут; секция «Escalation mid-flight» (вверх в полный цикл + вбок/вниз в research) не тронута. Убрана ТОЛЬКО неверная трактовка инфры.
- «Серию» тикетов отдельно НЕ реализовывал (по §1.2 это просто несколько /lite-task) — ничего сверх.

## Дисциплина версий (§4a)
- `manifest.yaml`: version `0.1.2` → `0.1.3` (в том же коммите).
- `CHANGELOG.md`: новая секция `[0.1.3] — 2026-07-24`, стиль файла (Keep a Changelog, раздел Changed).
- `check_profile_version_discipline.py --diff-against origin/main` → **OK — 48 profile(s) clean. 0 violations.**

## Тесты
`pytest tests/catalog/test_manifest_schemas.py test_iva_role_presets.py test_role_replacement_parity.py` → **212 passed, 1 failed**.
- Единственный failed: `test_role_covers_replaced_profile[iva-role-web<-iva-web-brownfield]` (потеряны angular-ds-component-authoring/usage).
- **Падение ПРЕДСУЩЕСТВУЮЩЕЕ и ВНЕ скоупа**: проверено через `git stash` на базовом коммите 5552118 (merge #149 ds-web-sc12) — падает идентично БЕЗ моих правок. Относится к лейну iva-role-web/angular-ds, который мне трогать запрещено. Мои 3 файла (все в tacticum-lite-base) на этот тест не влияют; все тесты, касающиеся lite-base, зелёные.
- Ожидалось 211 — фактически 213 тестов в наборе (ветка добавила DS-web SC12). Диффа тест-файлов от моих правок нет.

## Затронутые файлы (git diff --stat origin/main, 3 файла, +30 -2)
- `templates/tacticum-lite-base/ingredients/skills/lite-task-workflow/SKILL.md` (+14 -1... по факту +13/-3)
- `templates/tacticum-lite-base/manifest.yaml` (version)
- `templates/tacticum-lite-base/CHANGELOG.md` (+16)

iva-analysis-base / tacticum-bugfix-base / роли / ROLE_LANES / тесты / _mirrors / другие лейны — НЕ тронуты. autonomy off — НЕ пушил.

## На ревью тимлиду
- Пре-существующий фейл iva-role-web/iva-web-brownfield — не мой, но подсвечиваю: набор из 213 тестов на ветке НЕ полностью зелёный до моих правок (нужен отдельный добор в web-лейне, вне ТЗ#2 инфра-задачи).