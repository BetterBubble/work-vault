---
title: gate-pra-critic-fixes
type: note
permalink: tacticum/00-board/gate-pra-critic-fixes
status: resolved
verdict: PASS
tags:
- board
- design-system
- lead-ds
- tz1
- sc12
- controller
- gate
---

# gate-pra-critic-fixes — финальный re-гейт PR-A после критик-правок

**Контролёр:** controller-агент, 2026-07-24. **Объект:** worktree `/Users/bubblemac/tacticum/tacticum-dev-web-sc12`, ветка `feat/ds-web-sc12`, крайний коммит `be8277a` (критик-фиксы поверх `b8e1b5e`). Read-only. **Вход:** `00-Board/impl-pra-critic-fixes`, `00-Board/critic-pra`.

## ВЕРДИКТ: PASS — PR-A ready к вехе ГД

Все 5 пунктов зелёные. Один минорный не-блокирующий нюанс (pre-existing, вне дельты фикса) — ниже, для сведения лида. Не пушено (окно ведёт ГД).

---

## 1. Критик-правки корректны — ПРОШЛО

- **installation_id (крит. #1):** ✅ В usage добавлен явный мандат «`installation_id` on every `design_*` call» (§Tools, строки 48–51) — из `.tacticum/context.yaml`, иначе `installation_id_required`, в sub-agent runs из промпта; формулировка выровнена под `design-system-discovery`/`design-token-usage`. Реальный исполняемый вызов §Step 3 (стр. 94) теперь `design_get_tokens(design_system_id="iva-web", installation_id=<id from .tacticum/context.yaml>)`. Строка 46 (табличная сигнатура `design_get_tokens(design_system_id, ...)`) — это описание тула, не вызов; покрыта общим мандатом. Дыры исполнимости нет.
- **Шов authoring↔usage (крит. #2):** ✅ usage-интро больше НЕ приписывает создание биндинга authoring: «authoring *writes the component*; its code-bindings entry is then registered by the dictionary / RE pipeline (outside both skills — gaps G2/G3), and usage *consumes* that binding». То же в §Step 4 hand-off. authoring содержит короткий pointer «Where the binding comes from» (после §When to call): навык только пишет компонент, биндинг регистрирует словарный/RE-конвейер вне скоупа (G2/G3). Продюсер = словарный/RE-конвейер, не authoring. Round-trip непротиворечив; старых формулировок «creates … binding / author it + its binding» не осталось (grep чист).
- **#5 единое имя forward-ref:** ✅ В обоих навыках числовой Figma-compare = «future `ui-mockup-match` mode, not yet shipped (PR-B / gap G5)». authoring приведён к формулировке usage (§Acceptance + §Related skills); разнобой «enhanced / delivered separately» убран.
- **#6 подрезка повторов в authoring без потери норм:** ✅ Три чистых дедупа: (i) убран дубль «scaffold via angular-library» из заголовка §anatomy (полная версия в интро сохранена); (ii) §Acceptance «Variant completeness»+«Green spec» свёрнуты в «All Completeness deliverables present» (все дельверблы перечислены в §Completeness — variant/story/mdx/spec); (iii) §When to call убрано дублирующее «component X exists in Figma, not in code», оба entry-point и норма «read master first → anatomy» сохранены. Норм не потеряно.
- **authoring→usage именная ссылка:** ✅ §Related skills authoring — «`angular-ds-component-usage` — the Scenario 2 counterpart …». Симметрия есть.

## 2. Защищённое НЕ тронуто — ПРОШЛО

- `@iva/design-system` ✅ на месте (authoring стр. 5, 18).
- Поля биндинга ✅ все на месте (usage стр. 98–100): `name`, `match[]`, `figma_key`, `selector`, `kind`, `source`, `storybook`, `inputs{}`.
- Правило нормализации ✅ на месте (usage стр. 102), НЕ «поправлено» — критик-коммит `be8277a` эту строку не трогал (подтверждено `git show`; идентична base `b8e1b5e`).
- Числовой G5-matcher ✅ НЕ реализован — все упоминания pixel/ΔE только в контексте «do **not** implement» / «tracked as PR-B (gap G5), reference it». Только ссылка.

## 3. Гит/скоуп — ПРОШЛО

- `git diff --stat` от merge-base `origin/main` (`928fe37`): ровно **4 файла** — 2×SKILL.md + `manifest.yaml` + `CHANGELOG.md`, все строго в `templates/iva-web-brownfield`. (+404/−2.)
- owner / `_mirrors` / зеркала (`iva-web-development-base`) / роль / другие пакеты — НЕ тронуты (`git diff --name-only` = только 4 файла brownfield).
- Секретов/`.env`/ключей — нет. AI-подписей — нет: grep-хиты «claude» = легитимные платформенные `claude-code` в `supports:` и пути `.claude/skills/` манифеста, не футеры-атрибуции. Автор коммитов = Александр Шульга.
- Ветка `feat/ds-web-sc12` — НЕ main. Рабочее дерево чистое.

## 4. Версия/валидаторы — ПРОШЛО (прогнал сам)

Профиль `iva-web-brownfield` → `0.3.0`; правки внутри навыков (добавлены на 0.3.0, ещё не в main) покрыты записью [0.3.0] + помета «Critic fix-round (PR-A)». Прогон (`apps/backend/.venv/bin/python`):
- version-discipline **static:** `OK — 48 profile(s) clean.` ✅
- version-discipline **`--diff-against origin/main`:** `OK — 48 profile(s) clean.` ✅
- mirror-sync (brownfield-only, не в паре): `OK — 62 зеркальных ингредиентов в 6 парах синхронны.` ✅
- pytest `-k manifest_schemas`: `38 passed, 1397 deselected` ✅

## 5. 0 сверх-ТЗ — ПРОШЛО

Дельта фикс-раунда 1-в-1 ложится на критик-замечания #1–#6. Ничего сверх (никаких новых секций, тулов, реализации G5, правок защищённого). Принцип президента соблюдён.

---

## Минорное наблюдение (НЕ блокирует, pre-existing, вне дельты фикса)

Текст правила нормализации в навыке — «lowercase, strip spaces **and dashes**» (usage стр. 102). Отчёты `critic-pra`/`impl-pra-critic-fixes` описывают его как «lowercase, strip spaces/dashes/**underscores**» (с underscores). Это расхождение **описания в отчётах** vs текста навыка; строка НЕ менялась критик-коммитом (идентична base `b8e1b5e`), т.е. вне скоупа этого re-гейта и уже проходила fidelity-гейт. Рекомендация лиду: при случае сверить формулировку с реальным `tokens.json` (входит ли strip underscores) — если да, навык недоописывает норму. Для PR-A-ready вердикта не критично.

## Связано
`00-Board/critic-pra` · `00-Board/impl-pra-critic-fixes` · `00-Board/gate-pra-git` · `00-Board/gate-pra-fidelity`