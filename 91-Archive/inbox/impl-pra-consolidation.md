---
title: impl-pra-consolidation
type: report
permalink: tacticum/00-board/impl-pra-consolidation-1
status: draft
role: implementer
for: lead-ds
task: ТЗ#1 figma-ds PR-A — консолидация 2 DS-навыков в манифест iva-web-brownfield
worktree: ~/tacticum/tacticum-dev-web-sc12 (branch feat/ds-web-sc12)
commit: b8e1b5ed2a40c8e9fcbef95a119d04dd5a0b5298
date: 2026-07-24
tags:
- figma-ds
- impl
- pra
- consolidation
- g1
- g4
- g6
- lead-ds
archived-at: 2026-08-03 11:16
---

# impl PR-A — консолидация angular-ds-component-authoring + -usage

Подключил два уже-заавторенных навыка в манифест `iva-web-brownfield`, версия + CHANGELOG, коммит, все валидаторы зелёные. НЕ пушено (autonomy off).

## Изменённые файлы (git diff --stat HEAD~1..HEAD)
```
 templates/iva-web-brownfield/CHANGELOG.md          |  25 ++++
 .../skills/angular-ds-component-authoring/SKILL.md | 160 ++++++++++++++++++++
 .../skills/angular-ds-component-usage/SKILL.md     | 165 +++++++++++++++++++++
 templates/iva-web-brownfield/manifest.yaml         |  28 +++-
 4 files changed, 376 insertions(+), 2 deletions(-)
```
(SKILL.md обоих навыков вошли как `A` — они были untracked от параллельных implementer'ов; я их закоммитил вместе с подключением.)

## Manifest — обе записи целиком (вставлены между design-token-usage и ui-mockup-match)

```yaml
  - ingredient_id: angular-ds-component-authoring
    kind: skill_spec
    tier: trial
    supports: [claude-code, codex, copilot]
    install_scope: user
    target_path_template: ".claude/skills/{ingredient_id}/SKILL.md"
    copilot_target_path: ".github/skills/{ingredient_id}/SKILL.md"
    codex_target_path: ".agents/skills/{ingredient_id}/SKILL.md"
    body_path: ingredients/skills/angular-ds-component-authoring/SKILL.md
    metadata:
      description_trigger: "Authoring a NEW @iva/design-system Angular web component from a Figma master (gap-report case): component folder anatomy, directive button[ivaX] vs element iva-*, IvaControlBase+CVA, ivaPrefix/ivaSuffix/ivaLabel slots, token-only styling (zero hex), full variant coverage + Storybook + .mdx + spec"

  - ingredient_id: angular-ds-component-usage
    kind: skill_spec
    tier: trial
    supports: [claude-code, codex, copilot]
    install_scope: user
    target_path_template: ".claude/skills/{ingredient_id}/SKILL.md"
    copilot_target_path: ".github/skills/{ingredient_id}/SKILL.md"
    codex_target_path: ".agents/skills/{ingredient_id}/SKILL.md"
    body_path: ingredients/skills/angular-ds-component-usage/SKILL.md
    metadata:
      description_trigger: "Building a new/reworked screen from a Figma frame by assembling ready @iva/design-system components — pre-flight the frame (auto layout + recognisable instances, else stop to the designer), resolve each instance via Code Connect / the code-bindings dictionary, stop per-element on anything not in the dictionary, compose by selector (directive button[ivaButton] vs element iva-*, form-field, menu trigger) reading the component .mdx first, tokens-only (zero hex)"
```

- authoring: `description_trigger` взят из отчёта impl-g1-authoring §«Предлагаемый metadata.description_trigger».
- usage: сформулирован по impl-g4-usage (pre-flight G6 → resolve Code Connect/словарь → не-в-словаре → per-element STOP → сборка по selector, читать .mdx, tokens-only).
- По образцу существующей `design-system-discovery`: user-scope, три target_path (claude/copilot/codex), supports три CLI, tier trial. Поставлены рядом с design-system-discovery/design-token-usage.
- Комментарий-счётчик `# --- skill_spec (26) ---` обновлён на `(28)`.

## Версия + CHANGELOG
- `version: "0.2.1"` → `"0.3.0"` (minor, +2 навыка).
- Добавлена секция `## [0.3.0] — 2026-07-24` (Added): оба навыка Сц.1/Сц.2, gap G1/G4/G6, помечено brownfield-only + не-в-mirror, числовой matcher G5 = отсылка на ui-mockup-match/PR-B.
- Инвариант version==CHANGELOG выдержан (валидатор version-discipline подтвердил).

## Зеркало / mirror (не тронуто)
Пара `iva-web-development-base → iva-web-brownfield` в `_mirrors.yaml` НЕ содержит DS-навыков (design-system-discovery/design-token-usage там нет) → оба новых навыка brownfield-only, зеркала не требуют. `_mirrors.yaml` и owner-пакет НЕ трогал. Подтверждено check_mirror_sync (см. ниже).

## Валидаторы (venv `apps/backend/.venv/bin/python`, создан через `uv sync`)
```
$ scripts/check_mirror_sync.py
OK — 62 зеркальных ингредиентов в 6 парах синхронны.                 exit=0

$ scripts/check_profile_version_discipline.py
OK — 48 profile(s) clean.                                            exit=0

$ scripts/check_profile_version_discipline.py --diff-against origin/main
OK — 48 profile(s) clean.                                            exit=0

$ pytest apps/backend/tests/catalog/test_manifest_schemas.py -q
38 passed                                                            exit=0
```

## Коммит
`b8e1b5ed2a40c8e9fcbef95a119d04dd5a0b5298` — `feat(ds-web): angular-ds-component-authoring + -usage skills (Сц.1/2 gap G1/G4/G6)`. Без AI-подписей. Стейджил только 4 файла по задаче (venv в .gitignore, не попал). НЕ пушено.

## Самопроверка
- 2 skill_spec валидны по схеме (test_manifest_schemas 38 passed). OK
- version (0.3.0) == CHANGELOG heading [0.3.0]. OK
- Зеркала/роль/owner не тронуты (check_mirror_sync зелёный, _mirrors.yaml без изменений). OK
- Оба SKILL.md на месте, закоммичены (git diff --stat: обе A). OK
- Границы не расширял: только manifest + CHANGELOG + подключение двух готовых SKILL.md.