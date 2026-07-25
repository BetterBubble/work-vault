---
title: impl-qa-autotest-base
type: note
permalink: tacticum/00-board/impl-qa-autotest-base
status: draft
role: implementer
track: B
tags:
- implementer
- qa
---

# impl-qa-autotest-base

Собран автотест-лейн `iva-qa-autotest-base` из 9 реальных QA-скиллов и пересобран `iva-role-qa` под Трек B. По решениям [[resheniia-po-qa-profiliu-trek-b-2026-07-21]] и разведке [[explore-qa-autotest-skills]]. Не провижнил/не пушил/не мержил.

## Worktree / ветка / коммит
- Worktree: `/Users/bubblemac/tacticum/tacticum-dev-iva-write`
- Ветка: `feat/iva-write-base`
- Коммит (отдельный, аддитивный): `5df061c` — `feat(qa): автотест-лейн iva-qa-autotest-base (9 QA-скиллов) + пересбор iva-role-qa под Трек B`
- 51 файл в коммите. Guardrail соблюдён: `iva-analysis-base/*`, `iva-role-analyst/*` НЕ тронуты.

## Лейн `templates/iva-qa-autotest-base/` (leaf, БЕЗ depends_on)
- `manifest.yaml` — schema v2 по образцу analysis-base. 9 `skill_spec`. `stack.required: [one-web]`, `ide_targets.codex: best-effort` (честно), `post_install_notes`/`non_goals` фиксируют привязку к one-web + отсутствие MCP. mcp_server_spec = 0.
- `README.md` + `CHANGELOG.md` (0.1.0).
- 9 скиллов скопированы из источника в `ingredients/skills/<id>/SKILL.md` + `references/` (35 файлов) как есть. Правка **только frontmatter**: снят `allowed-tools` (у 8; у batch-autotest его не было), триггер вынесен в `manifest.metadata.description_trigger`. Тела скиллов и references не переписаны.

Тиры: trial = write-autotest, playwright-cli, run-tests, fix-failed-test (ядро цикла); full = batch-autotest, jira-issue-autotest, prepare-mr-branch, rebuild-autocore, retro.

`supports` по-честному: `[claude-code]` для write/batch/fix/run/jira (Task-субагенты / Skill() / хуки), `[claude-code, codex]` для playwright-cli/prepare-mr-branch/rebuild-autocore/retro.

## Новая композиция `iva-role-qa`
- `depends_on: [tacticum-core-base, iva-qa-autotest-base, iva-write-base]` (analysis убран).
- `persona.role: qa` (без изменений), `ingredients: []`.
- version 0.1.0 → 0.2.0. Суть: QA = исполнитель/автоматизатор автотестов + заведение дефектов; авторинг TC и покрытие (requirement_tests) остаются у аналитика. Обновлены name/description/scope/target_tasks/profiles/post_install_notes/non_goals/README/CHANGELOG.

## Дифф теста `apps/backend/tests/catalog/test_iva_role_presets.py`
```
-    "iva-role-qa": ["tacticum-core-base", "iva-analysis-base", "iva-write-base"],
+    "iva-role-qa": ["tacticum-core-base", "iva-qa-autotest-base", "iva-write-base"],
```
`ROLE_PERSONA['iva-role-qa'] = 'qa'` — без изменений.

## Реальный прогон (uv run pytest)
- `tests/catalog/test_iva_role_presets.py` → **35 passed in 0.43s** (0 fail). Покрыто: schema-v2 валидация, `ingredients==[]` (pure-composition leaf), depth-1 (новый лейн без depends_on), single-owner disjointness (core 6 ∩ qa-autotest 9 ∩ write 1 = ∅), golden-parity (union==sum), ide_targets (claude+codex full, copilot unsupported).
- `tests/catalog/test_manifest_schemas.py` → passed (новый лейн-манифест валиден по схеме).
- Полный `tests/catalog/` — file-level тесты зелёные; seed/init-тесты (test_seed_*, test_tacticum_init*) падают в **ERROR на setup** из-за недоступного Docker/Postgres в песочнице (`docker run postgres:16-alpine` exit 125) — инфра-зависимость, к правкам отношения НЕ имеет.

## Флаги для ГД
- **(а) 3 субагента отсутствуют** — `codebase-analyst`, `dom-explorer`, `code-writer` зовутся из write-autotest/batch-autotest/fix-failed-test через Task, в источнике их НЕТ. **Запрос `agent_spec` к QA-команде** (follow-up). Без них 3 из 9 скиллов неполны. Отражено в manifest.post_install_notes и README лейна.
- **(б) Привязка к one-web** — лейн репо-специфичен (autocore/venv/tools.testops/glab/CI), НЕ агностичен. Явно зафиксировано в `stack.required: [one-web]`, non_goals, README (не выдаётся за агностичный).
- **(в) docs ≠ skills** — PoC-документы описывают тест-ДИЗАЙН (Qwen → IVA QA Agent → TestOps); 9 скиллов = автотест-КОД. Слой тест-дизайна в лейне отсутствует.
- **(доп) codex-расхождение:** роль заявляет `codex: full` (конвенция ролей + требование теста `test_ide_targets_claude_and_codex_full`), но лейн реально `codex: best-effort` (Claude-специфика). Тест не подгонял — оставил роль full по конвенции, расхождение вынес флагом в manifest/README/CHANGELOG роли.

## Связано
- [[explore-qa-autotest-skills]]
- [[resheniia-po-qa-profiliu-trek-b-2026-07-21]]