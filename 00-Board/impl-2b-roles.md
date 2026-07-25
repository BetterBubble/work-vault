---
title: impl-2b-roles
type: note
permalink: tacticum/00-board/impl-2b-roles
status: draft
tags:
- implementer
- role-presets
---

# impl-2b-roles

Реализация трёх ролей-пресетов 2B по разведке [[explore-2b-roles]] и плану [[plan-tri-deliveravla-iva-iva-write-3-roli-my-todo-priviazka-k-sisteme]]. Работа строго аддитивная; guardrail соблюдён (analysis-лейны/analyst-роль/скиллы НЕ тронуты).

## Worktree / ветка / коммит

- Worktree: `/Users/bubblemac/tacticum/tacticum-dev-iva-write`
- Ветка: `feat/iva-write-base`
- Новый коммит (отдельный, поверх скелета write-base `cf8dd61`): **`69eb440`** — `feat(roles): три роли-пресета 2B — architect, qa, techwriter (ADR-0057)`
- НЕ провижнено, НЕ запушено/смержено/задеплоено.

## Созданные файлы (9 новых, по 3 на роль)

`templates/iva-role-architect/` — `manifest.yaml` + `README.md` + `CHANGELOG.md`
`templates/iva-role-qa/` — `manifest.yaml` + `README.md` + `CHANGELOG.md`
`templates/tacticum-role-techwriter/` — `manifest.yaml` + `README.md` + `CHANGELOG.md`

Все manifest'ы: `schema_version: "2"`, `ingredients: []` (чистый composition-leaf), `version: 0.1.0`, `maintainer: mr.diaret@ya.ru`, `license: MIT`, `deprecated: false`, `persona.role`+scope, `target_tasks`, `stack.required/optional: []`, `ide_targets` (claude-code+codex=full, copilot/opencode/gemini=unsupported), `profiles.trial/full`, `post_install_notes`, `non_goals`. README — квикстарт (композиция/что умеет/провенанс/CLI/связано), CHANGELOG — Keep-a-Changelog 0.1.0.

**Композиции (depends_on, порядок сохранён):**
- `iva-role-architect` → `[tacticum-core-base, iva-analysis-base, iva-write-base]`, persona.role `architect`
- `iva-role-qa` → `[tacticum-core-base, iva-analysis-base, iva-write-base]`, persona.role `qa`
- `tacticum-role-techwriter` → `[tacticum-core-base, tacticum-documentation-base, iva-write-base]`, persona.role `techwriter`

techwriter: скилл-мост «документ→публикация» НЕ добавлял (content-follow-up вне 2B-скелета) — роль осталась чистой композицией. Отмечено в README и CHANGELOG.

## Дифф теста `apps/backend/tests/catalog/test_iva_role_presets.py`

Единственная правка — +3 записи в `ROLE_LANES` и +3 в `ROLE_PERSONA` (больше нигде роли не перечисляются):

```python
    "iva-role-architect": ["tacticum-core-base", "iva-analysis-base", "iva-write-base"],
    "iva-role-qa": ["tacticum-core-base", "iva-analysis-base", "iva-write-base"],
    "tacticum-role-techwriter": [
        "tacticum-core-base",
        "tacticum-documentation-base",
        "iva-write-base",
    ],
...
ROLE_PERSONA = {
    "iva-role-analyst": "analyst",
    "iva-role-go": "developer",
    "iva-role-architect": "architect",
    "iva-role-qa": "qa",
    "tacticum-role-techwriter": "techwriter",
}
```

## Реальный результат прогона

Команда (окружение проекта, dev — optional-extra не dependency-group):
`cd apps/backend && uv run --extra dev python -m pytest tests/catalog/test_iva_role_presets.py -v`

**Результат: 35 passed, 0 failed** (0.95s). Было 14 (2 роли × 7 проверок) → стало 35 (5 ролей × 7). Все параметризованные проверки для новых ролей зелёные:
- `test_manifest_validates_against_v2_schema` — schema v2 OK
- `test_role_is_pure_composition_leaf` — profile_id/schema_version=="2"/deprecated is False/ingredients==[]/persona.role OK
- `test_depends_on_is_the_declared_lanes_in_order` — depends_on точь-в-точь, все лейны на диске (в т.ч. iva-write-base из этой ветки)
- `test_lanes_are_depth1_bases` — depth-1 OK
- `test_single_owner_lanes_are_pairwise_disjoint` — single-owner дизъюнктность OK (в т.ч. analysis∩write=∅)
- `test_golden_parity_union_equals_sum_of_lanes` — golden-parity OK
- `test_ide_targets_claude_and_codex_full` — ide_targets OK

Тест под зелёный не подгонялся — проверки сработали на честных манифестах.

## Флаги / замечания

- **Секвенс-блокер снят:** роли собраны в worktree, где `iva-write-base` присутствует (ветка `feat/iva-write-base`); в `main` write-base ещё нет — при вливании нужен порядок «write-base → роли».
- **architect и qa композиционно идентичны** (обе = core+analysis+write, ingredient-идентичны), различаются только `persona.role`/`name`/README/CHANGELOG. Соответствует плану; тесты проверяют роли независимо. Если ожидалось иное разведение — за владельцем.
- **Запуск pytest:** dev-зависимости в `pyproject.toml` — это `[project.optional-dependencies].dev`, не `dependency-groups`; поэтому `uv run --extra dev python -m pytest`, а не `--group dev`. `uv run pytest` без `python -m` подхватывает системный pytest без pydantic — не использовать.

## Связано

- [[explore-2b-roles]]
- [[plan-tri-deliveravla-iva-iva-write-3-roli-my-todo-priviazka-k-sisteme]]