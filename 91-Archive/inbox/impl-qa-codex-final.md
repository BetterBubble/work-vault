---
title: impl-qa-codex-final
type: impl-report
permalink: tacticum/00-board/impl-qa-codex-final-1
status: draft
topic: qa-codex-rework — FINAL (ядро fix/batch/jira + rebuild-doc + path-fix + iva-qa-mcp)
worktree: ~/tacticum/tacticum-dev-qa-codex
branch: feat/qa-codex-rework
date: 2026-07-24
tags:
- qa-codex
- implementer
- draft
archived-at: 2026-07-31 17:27
---

# impl: QA-codex FINAL — доделка ядра (fix/batch/jira) + rebuild-триггер + path-fix

Продолжение поверх KEYSTONE ([[impl-qa-codex-keystone]]). Ветка `feat/qa-codex-rework`,
worktree `~/tacticum/tacticum-dev-qa-codex`. 4 новых коммита поверх `0721ed2`. НЕ пуш/НЕ PR.
Механизм тот же ратифицированный: `codex_body_path` passthrough (ADR-0025 + прецедент
brownfield-*), НЕ угадан. Следовал эталону write-autotest.

## Коммиты (ветка feat/qa-codex-rework)
- `d65821d` feat(qa-codex): дивергентные codex-тела fix/batch/jira + path-нейтрализация write
- `0e6d067` feat(qa-codex): manifest — codex-доставка fix/batch/jira + уточнение комментариев
- `0b49adb` docs(qa-codex): rebuild-autocore — авто-триггер на Codex недоступен (R3)
- `c8b6d7f` docs(qa-codex): CHANGELOG [0.2.0] FINAL + README

## Что переделано по пунктам

### 1. fix-failed-test — codex-тело
`ingredients/skills-codex/fix-failed-test/SKILL.md` (новый, via `codex_body_path`).
- **Phase 4 делегирование** (было: «Делегируем: 1. codebase-analyst … 4. dom-explorer …
  5. code-writer через Edit») → три явных блока `spawn_agent(agent_type=…, task_name=…,
  message="… Сам субагентов НЕ спавни.", model="gpt-5.4")` → `wait_agent` → `close_agent`
  (SKILL.md:~215-265). codebase-analyst возвращает контент текстом (персист оркестратором),
  dom-explorer пишет locators сам, code-writer правит проект сам.
- **code-writer Edit → Codex apply_patch** — упомянуто: в шапке-комментарии (SKILL.md:11-18),
  в интро-списке ролей («правит … механизмом Codex `apply_patch`», ~85), в блоке spawn_agent
  (`apply_patch`), и в пороге тривиальности «≤3 строк» (оркестратор вносит сам `apply_patch`).
- Claude-тело сохранено (`ingredients/skills/fix-failed-test/SKILL.md` не тронут).
- manifest: `supports: [claude-code, codex]` + `codex_body_path`/`codex_target_path`
  (`.agents/skills/{ingredient_id}/SKILL.md`), manifest.yaml:145.

### 2. batch-autotest — codex-тело
`ingredients/skills-codex/batch-autotest/SKILL.md` (новый).
- **Веер разведки Phase 0.2** (было: «дефолт — 3 агента: 1 general-purpose + 2 Explore/
  codebase-analyst») → веер `spawn_agent` (дефолт 3 потока, потолок `max_threads=4`, пятый+
  в очередь) → `wait_agent`/`close_agent` (SKILL.md:~57).
- **Phase 2 делегирование** (steps 4/5/6 codebase-analyst/dom-explorer/code-writer) помечено
  `spawn_agent`, code-writer — `apply_patch` (SKILL.md:~72-74).
- **Сигналы компактации Claude сняты** из Handoff-чека Phase 2.0 — заменены на явное «на Codex
  механизма компактации нет; здесь их не проверяем», остались провайдер-нейтральные пороги
  (>2 циклов fix, >5 TC, запрос юзера, большой возврат субагента) (SKILL.md:~71).
- **`~/.claude/plans/` → `.tasks/`** в «Восстановление контекста» (SKILL.md:~112-115),
  формулировка «Если открыта новая сессия» (убрано «контекст компактирован»).
- Claude-тело сохранено. manifest: supports codex + codex-поля, manifest.yaml:158.

### 3. jira-issue-autotest — codex-тело
`ingredients/skills-codex/jira-issue-autotest/SKILL.md` (новый).
- **AskUserQuestion ×2 → свободный текст**: развилка ISSUE/VCSAT во входе (SKILL.md:~42) и
  «Разобрать падения ланча?» в Фазе 6 (SKILL.md:~96).
- **Skill(fix-failed-test) → codex-скилл по имени**: Фаза 6 «Да» → «загрузи по имени из дома
  скиллов (`.agents/skills/fix-failed-test/SKILL.md`) и выполни инлайн» (тот же приём, что в
  run-tests). Оркестрируемые batch/prepare-mr — аналогично (в шапке-комментарии).
- Claude-тело сохранено. manifest: supports codex + codex-поля, manifest.yaml:168.

### 4. rebuild-autocore триггер (R3)
Тело единое (Bash-шаги провайдер-нейтральны), **дивергентного codex-тела НЕ делал** — правильно.
Задокументировал в теле (`ingredients/skills/rebuild-autocore/SKILL.md`, секция «Связь с другими
skill / hook», ~154) + в manifest ide_targets-комментарии + CHANGELOG + README: **PostToolUse-хука
на Codex НЕТ** → авто-детект коммита autocore не срабатывает → на Codex rebuild = **ручной вызов**
ИЛИ **git-hook самого репо one-web/autocore** (напр. post-commit, вне лейна — лейн его не
поставляет). Без выдумок.

### 5. Path-desync (флаг keystone #2)
Все codex-тела нейтрализованы: `.claude/skills/craft-stack/` → `.agents/skills/craft-stack/`
($CRAFT). write-codex — 10 ссылок (perl-replace), fix/batch/jira — при копировании (0 остатков).
Шапки codex-тел это фиксируют. Проверка: `grep -rc '.claude/skills/craft-stack/'
skills-codex/*/SKILL.md` = 0 (кроме 1 намеренной ссылки в write-шапке — «в Claude-теле тот же
дом .claude/skills/craft-stack/»).

### 6. iva-qa-mcp — allowed_tools (замечание Глеба)
**Правок НЕ потребовалось — сверено, уже сужено корректно** (сделано в чанке 1).
`templates/iva-qa-mcp/manifest.yaml`:
- **helm-analyst** (было=стало): `allowed_tools` = `requirement_tests` / `gap_questions` /
  `contradiction_check` / `nearest_spec` / `affected_systems` / `constraints` / `related_tasks`
  — ровно требуемый QA-read-срез (7 тулов), БЕЗ полной поверхности (нет arch_map/arch_container/
  requirement_coverage/analyst_search/docs_ask/who_to_involve/effort_hint/api_registry_check).
- **iva-atlassian-write** (было=стало): `allowed_tools` = только `jira_create_issue` /
  `jira_update_issue` / `jira_add_comment` / `jira_get_transitions` / `jira_transition_issue`
  (jira_* defect/status), `CONFLUENCE_*` env не выставляется.
Дрейфа/расширения нет → версия iva-qa-mcp (0.1.0) не менялась, свой CHANGELOG не трогал.

## Доки / версия
- CHANGELOG `[0.2.0]` — добавлена секция **Added (FINAL)** (fix/batch/jira codex-тела +
  rebuild-doc + path-fix + iva-qa-mcp сверка); провизорный пункт path-desync помечен РАЗРЕШЁН.
- README — секция «Дивергенция Claude ↔ Codex» расширена таблицей на 4 оркестратора; codex
  best-effort переформулирован (единственный пробел — авто-триггер rebuild).
- manifest — ide_targets/footer-комментарии обновлены (ядро на Codex; best-effort из-за rebuild).

## Статус тестов
`test_manifest_schemas.py` + `test_iva_role_presets.py` (venv py3.12, `--noconftest`): **73 passed**.
Все 4 `codex_body_path` + 3 `agent_spec codex_body_path` резолвятся (файлы на месте). YAML валиден.
Дерево чистое.

## Что осталось (следующий шаг / R7)
1. **`codex_body_path` на skill_spec** — теперь на 4 оркестраторах (не только write); тот же
   R7-FLAG, ждёт ратификации lead-arch (благословить поле для skill_spec ИЛИ отдельный механизм
   дивергентных skill-тел). Помечено в manifest + CHANGELOG + README.
2. **Golden `iva-role-qa/codex.json` регенерация — отдельный шаг под e2e.** В golden добавятся
   `.codex/agents/*.toml` (3) + `.agents/skills/{write,fix-failed,batch,jira}-*/SKILL.md` (4).
   Вне двух лёгких тестов — требует harness (DB/seeder/e2e_install). НЕ форсил.
3. **references-дир для codex-скиллов** — codex-тела ссылаются на `references/…` (как Claude);
   доставка references для codex_body_path-скиллов — наследованный вопрос keystone, не
   переоткрывал.