---
status: draft
type: impl-report
topic: qa-codex-rework — KEYSTONE (субагенты + spawn_agent во write)
worktree: ~/tacticum/tacticum-dev-qa-codex
branch: feat/qa-codex-rework
date: 2026-07-24
permalink: tacticum/00-board/impl-qa-codex-keystone
---

# impl: QA-codex KEYSTONE — субагенты под Codex + spawn_agent во write

Ветка `feat/qa-codex-rework`, worktree `~/tacticum/tacticum-dev-qa-codex`. 4 новых коммита
поверх чанка-1. НЕ пуш/НЕ PR. Все правки в `templates/iva-qa-autotest-base/`.

## ⚠️ Ключевое расхождение с формулировкой задачи (прочитать первым)

Задача велела hand-authored role-toml класть как `ingredients/repo-configs/codex/agents/<role>.toml`
+ добавлять блоки `[agents.<role>]` в `.codex/config.toml` (механизм из /openai/codex docs).
**В репо действует СВОЙ ратифицированный механизм** доставки codex-субагентов, который это
подменяет — я пошёл по нему (не угадывал, нашёл в доках репо):

- **ADR-0025** (`docs/adr/0025-codex-cli-compat-tier-table.md`): `agent_spec` → native →
  `.codex/agents/<id>.toml` (keys: `name`, `description`, `developer_instructions`; опц.
  `model`, `model_reasoning_effort`, `sandbox_mode`, `mcp_servers`, `skills.config`).
- **Рендерер** `apps/backend/src/backend/catalog/infrastructure/renderers/codex.py:84-103`
  (`_render_agent`) сам эмитит `.codex/agents/<id>.toml` из canonical `agent_spec`.
- **Дивергентное codex-тело** доставляется полем `codex_body_path` (+`codex_target_path`):
  `seed_community.py:75-91` кладёт его в `metadata.codex_body`; `renderer.py:287-304`
  (`_cli_body_passthrough`) отдаёт verbatim, минуя авто-рендер (BL-1: иначе ломается model-id).
- **Прецедент** — все brownfield-* / iva-*-brownfield / iva-analysis-base / iva-go-development-base
  профили: `agent_spec` + `codex_body_path: ingredients/agents-codex/<id>.toml` +
  `codex_target_path: ".codex/agents/<id>.toml"` (напр. `templates/brownfield-task-workflow/manifest.yaml`).

**Вывод:** hand-authored `repo-configs/codex/agents/` + `[agents.<role>]` в config.toml —
НЕ нужны и конфликтовали бы с рендерером (drift). Codex подхватывает роли авто-дискавери из
`.codex/agents/*.toml`; глобальный `[agents] max_threads=4 max_depth=1` уже в config.toml роли
`iva-role-qa`. Механизм ратифицирован → это НЕ R7-провизор (в отличие от того, что закладывала задача).

## Что создано / изменено

### 1. Три субагента → двух-провайдерная доставка
- **Новые codex-тела** (hand-authored, verbatim body):
  - `ingredients/agents-codex/codebase-analyst.toml`
  - `ingredients/agents-codex/dom-explorer.toml`
  - `ingredients/agents-codex/code-writer.toml`
  - Формат: `name`/`description`(из manifest metadata)/`model="gpt-5.4"`/`model_reasoning_effort="medium"`/
    `sandbox_mode` + `developer_instructions = """<тело .claude/agents/<id>.md verbatim>"""`.
    Все 3 парсятся `tomllib` ✓. Тела без `\` и `"""` (проверено) → безопасно в TOML basic-multiline.
- **Claude-тела сохранены** как есть: `ingredients/agents/<id>.md`.
- **manifest** (`manifest.yaml`), каждый `agent_spec`: `supports: [claude-code, codex]` +
  `codex_body_path` + `codex_target_path: ".codex/agents/<id>.toml"`
  (строки ~226-235 codebase-analyst, ~240-249 dom-explorer, ~255-264 code-writer — после правок).

**Маппинг tools (best-effort, FLAG):** у Codex-субагентов НЕТ per-agent tools-allowlist
(ADR-0025 optional keys его не содержат). Claude `tools:` смаплен на `sandbox_mode`:
- codebase-analyst `[Read,Glob,Grep]` → `read-only`
- dom-explorer `[Bash,Read,Write,Glob,Grep]` → `workspace-write`
- code-writer `[Read,Edit,Write,Glob,Grep,Bash]` → `workspace-write`
Точность scope теряется (sandbox грубее) — FLAG в шапке каждого toml + в комментарии manifest.

### 2. write-autotest — эталон spawn_agent
- **Новое дивергентное codex-тело**: `ingredients/skills-codex/write-autotest/SKILL.md`.
  - Шаги 3/4/5: `Task(...)` → `spawn_agent(agent_type=..., task_name=..., message=..., model="gpt-5.4")`
    → `wait_agent` → `close_agent`. Субагентам в message явно «Сам субагентов НЕ спавни» (анти-рекурсия).
  - Report-file эвристика Claude снята: codebase-analyst возвращает analysis текстом →
    оркестратор персистит в `analysis.md` (файл нужен на диске для шага 5, где его читает
    code-writer); dom-explorer/code-writer пишут артефакты сами.
  - Остальное тело идентично Claude-версии.
  - Дивергенция задокументирована в HTML-шапке тела (`<!-- CODEX-ВАРИАНТ … -->`).
- **Claude-тело сохранено**: `ingredients/skills/write-autotest/SKILL.md` (Task).
- **manifest** write-autotest: `supports: [claude-code, codex]` +
  `codex_body_path: ingredients/skills-codex/write-autotest/SKILL.md` +
  `codex_target_path: ".agents/skills/{ingredient_id}/SKILL.md"`.

### 3. Доки
- `CHANGELOG.md` — под `[0.2.0]` добавлена секция **Added (KEYSTONE)** + блок «Провизорно / под R7».
- `README.md` — секция «Двух-провайдерная доставка субагентов» + «Дивергенция Claude↔Codex во write».

## Провизорно / ждёт lead-arch (R7)
1. **`codex_body_path` на `skill_spec`** (write-autotest) — применён ВПЕРВЫЕ. Исторически поле
   только на `agent_spec` (renderer.py:287 docstring: «Only agent ingredients carry a per-CLI body;
   skills/mcp/commands share one body»). Функционально работает СЕГОДНЯ: `seed_community` кладёт
   `codex_body` для любого ingredient, `_cli_body_passthrough` проверяет metadata kind-agnostic.
   Требует ратификации: благословить `codex_body_path` для skill_spec ИЛИ ввести отдельный
   механизм дивергентных skill-тел. (Помечено FLAG в manifest + CHANGELOG + README.)
2. **Path-рассинхрон в codex-телах** — ссылки на дом стека внутри codex-тел субагентов и
   write-codex-тела оставлены claude-формой `.claude/skills/craft-stack/` (минимальный diff;
   item 1 требовал instructions=body verbatim). На Codex дом доставляется в
   `.agents/skills/craft-stack/` ($CRAFT, чанк-1) → внутри тел путь неверен. Нейтрализация — отдельный проход.
3. **e2e_install golden `iva-role-qa/codex.json`** устареет (добавятся 3 `.codex/agents/*.toml`
   + `.agents/skills/write-autotest/SKILL.md`). Уже стар от чанка-1 (нет run-tests/craft-stack).
   Регенерация — вне двух лёгких тестов, отдельный harness (DB/seeder). Не форсил.

## Статус тестов
`test_manifest_schemas.py` + `test_iva_role_presets.py` (venv py3.12, `--noconftest`): **73 passed**.
Доп.: все 15 ingredients лейна валидны против `ingredient.v1.schema.json` (0 ошибок); manifest — валидный YAML, 15 ingredients, version 0.2.0. Дерево чистое.

## Коммиты (ветка feat/qa-codex-rework)
- `adc8d3e` feat(qa-codex): codex-тела 3 субагентов craft (agents-codex/*.toml)
- `e3a6f3f` feat(qa-codex): дивергентное codex-тело write-autotest (spawn_agent)
- `7982f62` feat(qa-codex): manifest — codex-доставка субагентов + write-autotest
- `0721ed2` docs(qa-codex): CHANGELOG [0.2.0] + README

## НЕ трогал (следующий проход, как в ТЗ)
fix-failed-test, batch-autotest, jira-issue-autotest, rebuild-триггер, iva-qa-mcp,
path-нейтрализация тел субагентов/write под $CRAFT.