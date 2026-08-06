---
title: gate-ds-source-reference-skill
type: report
permalink: tacticum/00-board/gate-ds-source-reference-skill-1
task: ТЗ#1 Сц.4 Фаза 3 ось-2 п.3 — reference-скилл web-to-kmp-source-reference
worktree: /Users/bubblemac/tacticum/tacticum-dev-ds-web-to-kmp
branch: feat/ds-web-to-kmp
head: d715669 (base 62bc27c)
verdict: PASS
autonomy: off (не мержено/не пушено/не деплоено)
tags:
- ds
- kmp
- tz1-sc4
- os-2
- controller
- gate
archived-at: 2026-07-31 17:27
---

# Гейт: web-to-kmp-source-reference (ось-2, два дерева)

**ВЕРДИКТ: PASS.** Все 5 пунктов чеклиста пройдены. Прогнал валидаторы сам — зелёные. Скоуп строго по плану, шаренный лейн lead-modes не тронут, конформность и достоверность подтверждены. Одно косметическое замечание (не блокирует).

## 1. Гит / скоуп — ПРОШЛО
- Дельта `62bc27c..d715669` = **ровно 4 файла**, все в `templates/iva-kmp-development-base/`: NEW `skills/web-to-kmp-source-reference/SKILL.md`, MOD `manifest.yaml`, MOD `CHANGELOG.md`, MOD `skills/web-to-kmp-screen-port/SKILL.md`. `git status` чист.
- **Шаренное НЕ тронуто** (проверено грепом путей в диффе — пусто): `brownfield-task-workflow`, `start task`, workflow-гейты, `iva-role-kmp`, зеркало `iva-kmp-brownfield`, `_mirrors.yaml`, ROLE_LANES/тест-матрица, `iva-web-brownfield`. Отсутствие шаренного workflow в диффе — **корректно** (лейн lead-modes, координируется отдельно), не дефект.
- Секретов/ключей/`.env`/мусора нет. **AI-подписей нет** (греп Co-Authored-By/Generated with/claude.ai — пусто). Ветка `feat/ds-web-to-kmp`, не main.

## 2. Конформность — ПРОШЛО
- Запись нового `skill_spec` в manifest — **Вариант-1, байт-в-байт по образцу** `web-to-kmp-screen-port` (сверено, строки 144-153): `install_scope: repo`, `target_path_template` == `codex_target_path` == `'AI common/skills/{ingredient_id}/SKILL.md'`, `supports: [claude-code, codex]`, `tier: full`, `body_path` существует.
- **Schema (прогнал сам, venv репо, jsonschema Draft7):** `manifest.v2` OK; `ingredient.v1` OK для всех 16 ингредиентов; новый ингредиент найден, body_exists=True.
- **Version 0.4.0 → 0.5.0 == CHANGELOG** `## [0.5.0] — 2026-07-24` (Added навык + Changed сшивка). n_ingredients=16.
- frontmatter SKILL.md валиден: `name: web-to-kmp-source-reference` + `description` (с триггерами) + `license`.

## 3. Достоверность / ТЗ — ПРОШЛО
Покрытие п.3 спека (ось-2) полное:
- **Два дерева:** таблица источник iva-one **READ-ONLY** (образец) + цель su.ivcs.messenger **WRITE**; явное отличие от `ivcs-libs-contract` (там оба репо через артефакт).
- **Доступ к источнику:** sibling-клон ИЛИ отдельный прогон `kb_discover` исходного репо; источник привязан к **своему** `.tacticum/context.yaml`/installation. Нет ни того ни другого → стоп+флаг (не выдумывать по памяти).
- **ЖЁСТКОЕ «в источник не писать»:** нет edit/commit/pull/branch/reset; require-изменение iva-one = out-of-scope → стоп+эскалация (по образцу `angular-legacy-web-context`).
- **ДС письма = ЦЕЛЕВОЙ репо** (KMP `Iva*` в `core/design-system`), не источник `@iva/design-system`.
- **Ссылка на §1 главного навыка** «Read the source (iva-one)» вместо дублирования процедуры — не дублирует.
- **Принцип президента (без over-scope):** ограничений СВЕРХ ТЗ не выдумано. Разграничение поверхностей (main UI в scope / конференц-`iva-core`/VCSWEB вне прохода, router-note) прямо опирается на ось-1 спека. Упоминание гейта готовности — как указатель на планируемый Phase-3.0-гейт (п.4 спека, лейн lead-modes), а не его реализация здесь. Скоуп корректен.
- **Сшивка в `web-to-kmp-screen-port` (§1 + реестр §8)** — только defer-указатель на нового владельца модели доступа; доктрина переноса не менялась. **Ссылка жива** (файл `web-to-kmp-source-reference/SKILL.md` существует, не мёртвая).

## 4. Валидаторы (прогнал сам) — ПРОШЛО
```
check_mirror_sync.py            → OK — 62 зеркальных ингредиента в 6 парах синхронны
check_profile_version_discipline.py (static)          → OK — 46 profile(s) clean
check_profile_version_discipline.py --diff-against 62bc27c → OK — 46 profile(s) clean
schema manifest.v2 + ingredient.v1 (16/16)            → OK
```
**Owner-only подтверждён:** в `_mirrors.yaml` пара `iva-kmp-development-base ↔ iva-kmp-brownfield` перечисляет конкретные ингредиенты; ни `web-to-kmp-screen-port`, ни новый `web-to-kmp-source-reference` в списке нет → навык корректно НЕ входит в зеркало, `_mirrors.yaml` правильно не тронут, mirror-sync зелёный.

## 5. Память — ПРОШЛО
- Отчёт implementer'а на доске (`00-Board/impl-ds-source-reference-skill`) — присутствует, полный (файлы, дифф, содержание, вывод валидаторов, открытые вопросы). Frontmatter корректен (task/worktree/branch/head/autonomy).
- Данный гейт-отчёт — эта карточка.

## Замечание (не блокирует)
- В отчёте implementer'а блок `git diff --stat` показывает «3 files changed» + NEW-файл отдельной строкой (untracked→committed), тогда как фактический коммит `d715669` = **4 файла**. Косметика форматирования отчёта, не расхождение содержания. Реальный дифф проверен напрямую — консистентен.

## Контекст
Скелет+указатели (без реальных Figma/пилот-путей) — норма для этой фазы. Шаренный workflow намеренно не трогается. Autonomy off — мерж/пуш не сейчас, отдельным осознанным шагом. Дальше: тимлид → OK Президента (через ГД).