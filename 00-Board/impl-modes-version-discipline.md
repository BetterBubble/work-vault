---
title: impl-modes-version-discipline
type: note
permalink: tacticum/00-board/impl-modes-version-discipline
tags:
- draft
---

# impl-modes-version-discipline

status: draft
Воркер: implementer (lead-modes ТЗ#2). Worktree: /Users/bubblemac/tacticum-worktrees/modes-workflow, ветка feat/workflow-modes. autonomy off — НЕ пушил.

## Задача
CI-чек «Profile version discipline» падал: 10 существующих пакетов изменили контент в ветке, но manifest.version не забамплен + нет CHANGELOG-записи (правило task_completion_checklist.md §4a). Добить 10 пакетов: patch-bump + CHANGELOG-запись. Дата записей = 2026-07-24.

## Как работает чек (для контекста)
- `scripts/check_profile_version_discipline.py`, workflow `.github/workflows/profile-version-discipline.yml`.
- На PR вызывается `python scripts/check_profile_version_discipline.py --diff-against "origin/${github.base_ref}"` (т.е. origin/main).
- «Content changed» = любой файл под `templates/<id>/` кроме CHANGELOG.md в diff `origin/main...HEAD` (закоммиченное, three-dot). Требует, чтобы `manifest.version` на диске отличался от версии в base-ref.
- CHANGELOG-требование: заголовок вида `## [<version>]` с точным вхождением новой версии (regex `^##\s+\[<version>\]`). Формат записей — Keep a Changelog: `## [X.Y.Z] — 2026-07-24` + `### Added/Changed/Fixed`.
- Важная деталь: пока bump manifest некоммичен, three-dot diff его не видит → срабатывает defensive-ветка «substantive files changed without touching manifest.yaml». Лечится коммитом (тогда manifest.yaml попадает в diff).

## Правки: пакет → версия было → стало
| Пакет | Было | Стало | Причина (CHANGELOG) |
|---|---|---|---|
| iva-analysis-base | 0.1.3 | 0.1.4 | 1-й гейт классификации режима в /start-task (аддитивно) |
| iva-role-analyst | 0.1.1 | 0.1.2 | depends_on += tacticum-research-base |
| iva-role-go | 0.2.0 | 0.2.1 | depends_on += tacticum-lite-base + tacticum-research-base |
| iva-role-ios | 0.1.0 | 0.1.1 | то же (lite+research) |
| iva-role-java | 0.1.0 | 0.1.1 | то же |
| iva-role-kmp | 0.1.0 | 0.1.1 | то же |
| iva-role-mail | 0.1.0 | 0.1.1 | то же |
| iva-role-web | 0.1.0 | 0.1.1 | то же |
| tacticum-bugfix-base | 0.1.1 | 0.1.2 | маршрутизация change→/lite-task vs /start-task (restore→/fix-bug цел) |
| tacticum-development-core | 0.1.0 | 0.1.1 | 2-й слой (гейт пересмотра режима + handoff) в run-implementation (аддитивно) |

Менял ТОЛЬКО manifest.yaml (version) + CHANGELOG.md. Контент лейнов не трогал. Примечание: у tacticum-development-core version без кавычек в файле (`version: 0.1.1`) — сохранил стиль файла; YAML парсит как строку, чек проходит.

## Проверки локально
Дисциплина-чек (та же команда, что CI, из apps/backend через uv для PyYAML):
```
uv run python ../../scripts/check_profile_version_discipline.py --diff-against origin/main
```
Результат ПОСЛЕ коммита: `OK — 48 profile(s) clean.` EXIT=0. **0 violations.**

Тесты каталога:
```
uv run pytest tests/catalog/test_manifest_schemas.py tests/catalog/test_iva_role_presets.py tests/catalog/test_role_replacement_parity.py
```
Результат: **211 passed in 3.64s.**

## Пакеты вне списка
Нет. Чек требовал ровно те 10 пакетов, что в ТЗ (исходный прогон дал 10 violations, поимённо совпали). Наши новые лейны lite/research чисты. После правок 0 violations на всех 48 профилях.

## Коммит (локально, БЕЗ пуша)
- Хеш: `f48e5e7f4e985d009dea9fdd0d75e22f7043691d`
- Сообщение: `chore: bump versions + CHANGELOG for modes-touched packages (§4a discipline)`
- Файлы: 10× manifest.yaml + 10× CHANGELOG.md (20 файлов, ничего лишнего в git status).

Тимлид: сверить и запушить ветку feat/workflow-modes.
