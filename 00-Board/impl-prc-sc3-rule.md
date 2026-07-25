---
title: impl-prc-sc3-rule
type: report
permalink: tacticum/00-board/impl-prc-sc3-rule
status: draft
role: implementer
for: lead-ds
repo: ~/tacticum/tacticum-dev-web-axis1 (worktree, ветка feat/ds-web-axis1)
commit: 0c5d857
tz: scratchpad/ds-scan/figma-ds-scenario-3-migration.md
context: 00-Board/map-sc3-and-remainder
date: 2026-07-24
tags:
- figma-ds
- impl
- sc3
- migration
- pr-c
- axis1
- lead-ds
---

# PR-C (добавка) — тонкое правило Сц.3 (миграция) в angular-ds-component-authoring

Задача: добавить ТОНКУЮ секцию-дисциплину миграции (Сц.3) в тело навыка `angular-ds-component-authoring`, БАЙТ-ИДЕНТИЧНО в обе копии. Строго по ТЗ Сц.3, без механик сверх (codemod/coverage-тулинг = RE, не выдумывал). НЕ пушено.

## Что сделано

Добавлена секция `## Migration (Scenario 3)` (20 строк, перед `## Related skills`) в ОБЕ копии навыка:
- `templates/iva-web-brownfield/ingredients/skills/angular-ds-component-authoring/SKILL.md`
- `templates/iva-web-development-base/ingredients/skills/angular-ds-component-authoring/SKILL.md`

### Текст добавленной секции

```
## Migration (Scenario 3)

Migrating a legacy surface (frozen `ui-kit` / `dsGetColor`) onto
`@iva/design-system` / `ivaGetColor` runs in **two layers, in batches**: tokens
first (legacy values → new tokens by the transition table), then components
(legacy component → new one from the dictionary). Each batch is verified like
**Scenario 2** (`ui-mockup-match` numeric compare, or design review when there is
no mockup).

- **Do not mix the two design systems in one component** — a batch migrates it
  whole.
- No new-DS analog for a component → **author it first** (this skill / Scenario
  1), then migrate the usage; do not substitute a hand-made copy.
- Removing the legacy layer is a **separate task**, run when its usage count in
  the coverage report reaches zero — not part of this skill.

The actual migration run of a repo (e.g. iva-one) and the coverage report itself
are **outside this skill** (iva-one team / RE pipeline); this skill only provides
the migration discipline, not codemod or coverage tooling.
```

Покрытие ТЗ Сц.3 (все 5 пунктов задания): (1) 2 слоя батчами, токены→компоненты, батч верифицируется как Сц.2; (2) не смешивать две ДС в одном компоненте; (3) нет аналога → сначала авторить (этот навык/Сц.1); (4) удаление легаси = отдельная задача по нулю в отчёте покрытия; (5) явно — прогон миграции iva-one + отчёт покрытия ВНЕ навыка (iva-one/RE), навык даёт только правило-дисциплину. Никаких codemod/coverage-механик не добавлял.

## Верификация

- **cmp двух копий:** IDENTICAL (identical до и после). Общий хэш после правки `git hash-object` = `fcd1e881698b0e25b4b75b0a0cbe319d304afd9e` (обе). До правки был `70f50ec1…` (обе).
- **check_mirror_sync.py:** `OK — 64 зеркальных ингредиентов в 6 парах синхронны.` (rc=0)
- **version-discipline (static):** `OK — 48 profile(s) clean.` (rc=0)
- **version-discipline --diff-against origin/main:** `OK — 48 profile(s) clean.` (rc=0) → **бамп версий НЕ требуется** (правка в теле навыка, не новый ингредиент; валидатор чист). CHANGELOG не трогал.
- **pytest apps/backend/tests/catalog/ -q** (venv `apps/backend/.venv/bin/python`): **549 passed**, 2 failed, 120 errors. Все 122 не-прошедших = DB/docker-инфра (Postgres на :5432 не поднят / `docker run postgres:16-alpine` exit 125 — нет демона). Coverage/не-DB тесты зелёные; правка markdown-only в шаблонах на них влиять не может (проверил: 2 failed — `test_admin_catalog_authoring` patch_profile → DB-коннект).

## Границы (соблюдены)

- Тронуты РОВНО 2 файла (обе копии навыка), diff `2 files changed, 40 insertions(+)`. Другие навыки/manifest/версии/CHANGELOG/`_mirrors.yaml` НЕ трогал.
- Правка крошечная, строго по ТЗ Сц.3. Механик сверх ТЗ (codemod, coverage-тулинг) не добавлял — они RE-зона.
- Коммит `0c5d857`, без AI-подписей. НЕ пушено (ждёт ревью лида → решение пользователя).

## Самопроверка

- Обе копии байт-идентичны (cmp identical, одинаковый hash-object). ✅
- Правка тонкая, по ТЗ. ✅
- Версии согласованы (бамп не понадобился — валидатор зелёный). ✅
