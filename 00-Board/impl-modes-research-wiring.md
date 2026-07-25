---
title: impl-modes-research-wiring
type: note
permalink: tacticum/00-board/impl-modes-research-wiring
tags:
- draft
---

# impl-modes-research-wiring

status: draft
Автор: implementer (lead-modes, ТЗ#2). Worktree `/Users/bubblemac/tacticum-worktrees/modes-workflow`, ветка `feat/workflow-modes`. autonomy off — НЕ запушено, НЕ смержено.

## Что сделано
Врезан готовый лейн `tacticum-research-base` в `depends_on` целевых ролей (композиция ADR-0056/0057) + обновлена тест-матрица `ROLE_LANES` (и её не потребовалось трогать `ROLE_PERSONA` — persona ролей не меняется). Место врезки — рядом с процессным `tacticum-bugfix-base` (порядок=приоритет; т.к. лейны single-owner и research disjoint, порядок влияет только на детерминизм).

**Коммит (локально, без AI-подписей):** `65d4984` — `feat(roles): compose tacticum-research-base into dev + analyst role presets`

## Затронутые роли — depends_on ДО / ПОСЛЕ

| Роль | ДО | ПОСЛЕ |
|---|---|---|
| iva-role-go | core, development-core, go-dev-base, bugfix | core, development-core, go-dev-base, bugfix, **research** |
| iva-role-java | core, development-core, java-dev-base, bugfix | core, development-core, java-dev-base, bugfix, **research** |
| iva-role-ios | core, development-core, ios-dev-base, bugfix, ui | core, development-core, ios-dev-base, bugfix, **research**, ui |
| iva-role-kmp | core, development-core, kmp-dev-base, bugfix, ui | core, development-core, kmp-dev-base, bugfix, **research**, ui |
| iva-role-mail | core, development-core, mail-dev-base, bugfix, ui | core, development-core, mail-dev-base, bugfix, **research**, ui |
| iva-role-web | core, development-core, web-dev-base, bugfix, ui | core, development-core, web-dev-base, bugfix, **research**, ui |
| iva-role-analyst | core, iva-analysis-base | core, iva-analysis-base, **research** |

Итого 7 ролей: 6 dev + аналитик. Совпадает с целевым списком ГД.

**НЕ тронуты** (по ГД-скоупу «dev-роли + аналитик, стек-агностик»): `tacticum-role-internal`, `tacticum-role-platform`, `firebird-role-web`, `iva-role-architect`, `iva-role-qa`, `tacticum-role-techwriter`. См. «спорные роли» ниже.

## ПОЛНЫЙ ДИФФ ROLE_LANES (дословно — для отправки ГД, разводка с lead-fr US#3)

```diff
diff --git a/apps/backend/tests/catalog/test_iva_role_presets.py b/apps/backend/tests/catalog/test_iva_role_presets.py
@@ ROLE_LANES: dict[str, list[str]] = {
-    "iva-role-analyst": ["tacticum-core-base", "iva-analysis-base"],
+    "iva-role-analyst": ["tacticum-core-base", "iva-analysis-base", "tacticum-research-base"],
     # Implement-only dev-роли (ADR-0059 Решение 6): без analysis и без documentation.
+    # + research-lane (стэк-агностик investigation, ТЗ#2 §6: research нужен dev-ролям —
+    # обе жалобы от разработчиков — рядом с процессным bugfix-лейном).
     "iva-role-go": [
         "tacticum-core-base",
         "tacticum-development-core",
         "iva-go-development-base",
         "tacticum-bugfix-base",
+        "tacticum-research-base",
     ],
     "iva-role-kmp": [
         "tacticum-core-base",
         "tacticum-development-core",
         "iva-kmp-development-base",
         "tacticum-bugfix-base",
+        "tacticum-research-base",
         "tacticum-ui-base",
     ],
     "iva-role-web": [
         "tacticum-core-base",
         "tacticum-development-core",
         "iva-web-development-base",
         "tacticum-bugfix-base",
+        "tacticum-research-base",
         "tacticum-ui-base",
     ],
     "iva-role-mail": [
         "tacticum-core-base",
         "tacticum-development-core",
         "iva-mail-development-base",
         "tacticum-bugfix-base",
+        "tacticum-research-base",
         "tacticum-ui-base",
     ],
     "iva-role-ios": [
         "tacticum-core-base",
         "tacticum-development-core",
         "iva-ios-development-base",
         "tacticum-bugfix-base",
+        "tacticum-research-base",
         "tacticum-ui-base",
     ],
     "iva-role-java": [
         "tacticum-core-base",
         "tacticum-development-core",
         "iva-java-development-base",
         "tacticum-bugfix-base",
+        "tacticum-research-base",
     ],
```

## Тесты (без docker, .venv)
`pytest test_iva_role_presets.py test_manifest_schemas.py test_role_replacement_parity.py`
**211 passed, 0 failed** (4.10s). Красных нет — чинить ничего не пришлось.

## Находки / риски
- **Parity/single-owner держатся.** `tacticum-research-base` несёт ровно 2 ingredient_id: `research`, `start-research`. Ни один не встречается ни в одном из композируемых лейнов (core/development-core/bugfix/ui/analysis/*-development-base) → тесты `test_single_owner_lanes_are_pairwise_disjoint` и `test_golden_parity_union_equals_sum_of_lanes` зелёные без правок KNOWN_OVERRIDES.
- **depth-1 сохранён.** research-base — база без своего depends_on (`test_lanes_are_depth1_bases` зелёный).
- **replacement-parity не задет.** Врезка только ДОБАВЛЯет ингредиенты в union роли; allowlist старых профилей описывает то, чего роль НЕ несёт — новые id его не меняют. Зелёный.
- **Зеркала (_mirrors.yaml) не тронуты.** research-base новый, в декларации зеркал не участвует; body-файлы ролей/лейнов не менялись → byte-parity инвариант зелёный. Синхронизация зеркал НЕ требуется.
- **iva-analysis-base НЕ тронут** (guardrail Diaret): у аналитика правился только role-манифест `iva-role-analyst` (добавлен лейн в depends_on), сам лейн analysis не менялся. STOP-условие не наступило.
- **Спорные роли (для сверки ГД до пуша).** Под скоуп «dev-роль» формально попадают ещё `tacticum-role-internal`, `tacticum-role-platform` (persona=developer) и `firebird-role-web` (dev, ui). Я их НЕ включил, т.к. ГД дал явный целевой список из 6 iva-dev-ролей + аналитик, и internal/platform — служебные контуры (по ТЗ Солонко жалобы шли от продуктовых разработчиков). Если ГД захочет расширить на internal/platform/firebird — врезка тривиальна и тем же паттерном (после bugfix-base). Пометка для тимлида: включить в вопрос к ГД.

## Файлы (абсолютные пути)
- Манифесты: `/Users/bubblemac/tacticum-worktrees/modes-workflow/templates/iva-role-{go,ios,java,kmp,mail,web,analyst}/manifest.yaml`
- Тест-матрица: `/Users/bubblemac/tacticum-worktrees/modes-workflow/apps/backend/tests/catalog/test_iva_role_presets.py`
