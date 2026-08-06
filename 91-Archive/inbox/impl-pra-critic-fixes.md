---
title: impl-pra-critic-fixes
type: note
permalink: tacticum/00-board/impl-pra-critic-fixes-1
status: draft
tags:
- board
- design-system
- lead-ds
- tz1
- sc12
- implementer
archived-at: 2026-08-03 11:16
---

# impl-pra-critic-fixes — критик-правки PR-A внесены

**Воркер:** implementer, 2026-07-24. **Worktree:** `/Users/bubblemac/tacticum/tacticum-dev-web-sc12` (ветка `feat/ds-web-sc12`). **Коммит:** `be8277a` (поверх `b8e1b5e`). **НЕ запушено** (autonomy off).

## Изменённые файлы
- `templates/iva-web-brownfield/ingredients/skills/angular-ds-component-usage/SKILL.md`
- `templates/iva-web-brownfield/ingredients/skills/angular-ds-component-authoring/SKILL.md`
- `templates/iva-web-brownfield/CHANGELOG.md`

`git diff --stat HEAD~1 HEAD`:
```
 templates/iva-web-brownfield/CHANGELOG.md          |  7 +++++
 .../skills/angular-ds-component-authoring/SKILL.md | 31 ++++++++++++++--------
 .../skills/angular-ds-component-usage/SKILL.md     | 24 ++++++++++++-----
 3 files changed, 45 insertions(+), 17 deletions(-)
```

## По пунктам

1. **usage — `installation_id` (критично).** Добавил оговорку после таблицы §Tools (формулировка выровнена под `design-system-discovery`/`design-token-usage`: «pass the value from `.tacticum/context.yaml` explicitly … team `phk_*` tokens … fail with `installation_id_required`; в sub-agent runs — из промпта»). В §Step 3 вызов теперь `design_get_tokens(design_system_id="iva-web", installation_id=<id from .tacticum/context.yaml>)`.

2. **Шов authoring↔usage по биндингу (критично).** (а) usage — интро больше НЕ приписывает создание биндинга навыку authoring: «authoring *writes the component*; its code-bindings entry is then registered by the dictionary / RE pipeline (outside both skills — G2/G3), and usage *consumes* that binding». То же в §Step 4 (hand-off): «author the component; its binding is registered by the dictionary/RE pipeline (G2/G3), re-run once present». (б) authoring — добавлен короткий pointer «Where the binding comes from» после §When to call: навык только пишет компонент, биндинг регистрирует словарный/RE-конвейер вне скоупа (G2/G3). Round-trip честный, старых противоречивых формулировок не осталось (проверено grep).

3. **Единое имя forward-ref (дёшево).** authoring приведён к формулировке usage: числовой Figma-compare = «future `ui-mockup-match` mode, not yet shipped (PR-B / gap G5)» — в §Acceptance и §Related skills. Разнобой «enhanced ui-mockup-match / delivered separately» убран. usage-формулировки НЕ трогал (они эталон).

4. **authoring — подрезка повторов (дёшево).** Три чистых дедупа без потери норм: (i) убран дублирующий «(scaffold via `angular-library`, do not hand-roll)» из заголовка §anatomy (полная версия остаётся в интро); (ii) §Acceptance — «Variant completeness» + «Green spec» свёрнуты в один пункт «All Completeness deliverables present» (variant/story/mdx/spec уже перечислены в §Completeness); (iii) §When to call — убрано переопределение «component X exists in Figma, not in code» (уже в интро), оставлены оба entry-point и норма «read master first → anatomy».

5. **authoring→usage симметрия (дёшево).** В §Related skills authoring добавлена именная ссылка на `angular-ds-component-usage` («the Scenario 2 counterpart …»).

## НЕ тронуто (по указанию лида)
`@iva/design-system`, поля биндинга (name/match/figma_key/selector/kind/source/storybook/inputs), правило нормализации «lowercase, strip spaces/dashes/underscores» — на месте (grep подтвердил). Числовой matcher G5 не реализован. Manifest-структура, зеркала, owner, роль — не тронуты.

## Версия
Version-discipline проходит **без бампа** — правки внутри навыков, добавленных на 0.3.0 (ещё не в main), покрыты существующей записью [0.3.0]. Оставил 0.3.0, добавил помету «Critic fix-round (PR-A)» в секцию [0.3.0] CHANGELOG.

## Валидаторы (venv `apps/backend/.venv/bin/python`)
- version-discipline static: `OK — 48 profile(s) clean.`
- version-discipline `--diff-against origin/main`: `OK — 48 profile(s) clean.`
- mirror-sync: `OK — 62 зеркальных ингредиентов в 6 парах синхронны.`
- pytest `-k manifest_schemas`: `38 passed, 1397 deselected`.

## Самопроверка
- usage везде с installation_id (§Tools note + §Step 3 вызов) ✅
- шов биндинга непротиворечив, старых формулировок «creates … binding / author it + its binding» нет ✅
- #3/#4 usage-эталон не тронут, защищённые элементы на месте ✅

## Связано
`00-Board/critic-pra` · `00-Board/gate-pra-fidelity` · `00-Board/gate-pra-git`