---
title: impl-ds-source-reference-skill
type: report
permalink: tacticum/00-board/impl-ds-source-reference-skill
status: draft
task: ТЗ#1 Сц.4 Фаза 3 ось-2 п.3 — reference-скилл «читать репо A read-only, писать
  репо B»
worktree: /Users/bubblemac/tacticum/tacticum-dev-ds-web-to-kmp
branch: feat/ds-web-to-kmp
head: d715669 (base 62bc27c)
autonomy: off (не мержено/не пушено/не деплоено)
tags:
- ds
- kmp
- tz1-sc4
- os-2
- implementer
---

# impl: web-to-kmp-source-reference (ось-2, два дерева)

Создан новый reference-скилл двух-древесного доступа для переноса iva-one → KMP и подключён repo-native в пакет `iva-kmp-development-base`. Всё в worktree на ветке `feat/ds-web-to-kmp`, закоммичено (`d715669`), не мержено/не пушено.

## Имя навыка
Выбрал **`web-to-kmp-source-reference`** (из двух предложенных). Парно к главному навыку `web-to-kmp-screen-port`: тот владеет *процедурой переноса*, этот — *моделью доступа к двум деревьям* (где источник, что с ним можно). Явная связка по имени лучше, чем нейтральное `source-repo-reference-context`.

## Созданные / изменённые файлы
- **NEW** `templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-source-reference/SKILL.md` — тело навыка (по образцу `angular-legacy-web-context`: тон, жёсткое read-only-правило, out-of-scope секция).
- **MOD** `templates/iva-kmp-development-base/manifest.yaml` — запись ingredient (skill_spec, repo-native Вариант 1) + bump `version: 0.4.0 → 0.5.0`.
- **MOD** `templates/iva-kmp-development-base/CHANGELOG.md` — запись `## [0.5.0]` (Added новый навык + Changed сшивка).
- **MOD** `templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-screen-port/SKILL.md` — минимальная сшивка: ссылка на новый навык в §1 (read-order intro) и строка в реестре §8. Доктрина переноса не менялась.

НЕ трогал: `iva-role-kmp`, зеркало `iva-kmp-brownfield`, `_mirrors.yaml` (навык owner-only, как и `web-to-kmp-screen-port` — в зеркало не входит), `brownfield-task-workflow`/`start task`/workflow-гейты (шаренный лейн lead-modes).

## git diff --stat
```
 templates/iva-kmp-development-base/CHANGELOG.md    | 25 ++++++++++++++++++++++
 .../skills/web-to-kmp-screen-port/SKILL.md         | 10 ++++++---
 templates/iva-kmp-development-base/manifest.yaml   | 19 +++++++++++++++-
 3 files changed, 50 insertions(+), 4 deletions(-)
 + NEW web-to-kmp-source-reference/SKILL.md (untracked→committed)
```

## Содержание навыка (что покрыто по ТЗ)
- **Роль:** два дерева в одной задаче — источник **iva-one READ-ONLY** (образец) + цель **su.ivcs.messenger WRITE**. Таблица «две ветки, не путать» + явное отличие от `ivcs-libs-contract` (там оба репо меняются через артефакт — здесь только читаем A, пишем B).
- **Доступ к источнику:** (1) sibling-клон iva-one, пиннутый на ref образца, не мутировать; (2) отдельный прогон `kb_discover` исходного репо (своя KB, отличная от целевой). Источник привязан к **своему** `.tacticum/context.yaml`/installation. Нет ни клона, ни KB-прогона → стоп + флаг гейту готовности (не выдумывать источник по памяти).
- **ЖЁСТКОЕ правило:** никогда не писать в источник (нет edit/commit/pull/branch); не автор Angular «чтобы было проще»; require-изменение iva-one = out-of-scope → стоп+эскалация (как `angular-legacy-web-context`). Все записи — в цель; **ДС письма определяет ЦЕЛЕВОЙ репо** (KMP `Iva*`), не источник `@iva/design-system`.
- **Что извлекать:** указатель на `web-to-kmp-screen-port` §1 «Read the source (iva-one)» без дублирования (структура/состояние/состав/REST/Transloco).
- **Разграничение поверхностей:** основной UI iva-one = `@iva/design-system` (в scope текущего прохода); конференц/MCU = `iva-core`/VCSWEB (отдельная ДС, **вне прохода**, router-note к будущему iva-core-заходу). Если образец на конференц-поверхности → стоп+роутинг.
- **Companion:** явная связка с `web-to-kmp-screen-port` (кто чем владеет).

## Запись манифеста нового навыка (целиком)
```yaml
- ingredient_id: web-to-kmp-source-reference
  kind: skill_spec
  tier: full
  supports:
  - claude-code
  - codex
  install_scope: repo
  target_path_template: 'AI common/skills/{ingredient_id}/SKILL.md'
  codex_target_path: 'AI common/skills/{ingredient_id}/SKILL.md'
  body_path: ingredients/skills/web-to-kmp-source-reference/SKILL.md
  metadata:
    description_trigger: Two-tree access model for a web→KMP port — read the iva-one Angular
      source (sibling clone or its own kb_discover run, bound to its own .tacticum/context.yaml
      installation) while writing into su.ivcs.messenger; HARD rule source tree is read-only,
      target repo owns the write-side design system; surface split main UI @iva/design-system
      (in scope) vs conference iva-core/VCSWEB (router-note, out of scope); companion to
      web-to-kmp-screen-port which owns the port procedure
```
Repo-native Вариант 1 — байт-в-байт по образцу `web-to-kmp-screen-port`: `install_scope: repo`, `target_path_template` = `codex_target_path` = `'AI common/skills/{ingredient_id}/SKILL.md'` (KMP-репо потребляет через opencode `ai-skills` path=`AI common/skills`).

## Вывод валидаторов (venv репо: apps/backend/.venv/bin/python)
```
1) SCHEMA (manifest.v2 + ingredient.v1, jsonschema Draft7):
   manifest.v2: OK — version=0.5.0, 16 ingredients
   ingredient.v1: OK
   new ingredient found: True | install_scope: repo | body_path exists: True

2) check_mirror_sync.py:
   OK — 62 зеркальных ингредиентов в 6 парах синхронны.

3) check_profile_version_discipline.py (static):
   OK — 46 profile(s) clean.

4) check_profile_version_discipline.py --diff-against 62bc27c:
   OK — 46 profile(s) clean.

5) pytest apps/backend/tests/catalog/test_manifest_schemas.py:
   38 passed.
```

## Самопроверка
- Оба SKILL.md читаются, YAML-frontmatter валиден (name/description/license).
- Манифест консистентен: 16 ingredients, версия 0.5.0 ↔ CHANGELOG `## [0.5.0]` присутствует.
- Мёртвых ссылок нет: новый навык ссылается на существующие навыки по имени (`web-to-kmp-screen-port`, `angular-legacy-web-context`, `ivcs-libs-contract`) + on-repo концепты (iva-core, `@iva/design-system`); `web-to-kmp-screen-port` ссылается на новый навык (§1, §8) — файл существует.
- Mirror sync зелёный — навык owner-only, `_mirrors.yaml` не трогал.

## Открытые вопросы / для тимлида
- Version-discipline diff-check против `62bc27c` тривиально пуст (HEAD был == base до правок; мои правки теперь в коммите `d715669`). Static-check и pytest подтверждают инвариант version↔CHANGELOG. Против `origin/main` diff-check прогонит CI при PR.
- Навык остаётся в owner-пакете `iva-kmp-development-base`; дом доставки (провижн в `AI common/skills/` целевого репо + AGENTS.md-роутер) — этап доставки, вне этого пакета, как и у `web-to-kmp-screen-port`.