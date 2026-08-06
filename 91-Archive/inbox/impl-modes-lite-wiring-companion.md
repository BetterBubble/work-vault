---
title: impl-modes-lite-wiring-companion
type: note
permalink: tacticum/00-board/impl-modes-lite-wiring-companion-1
tags:
- draft
archived-at: 2026-07-31 17:27
---

# impl-modes-lite-wiring-companion

status: draft
worktree: /Users/bubblemac/tacticum-worktrees/modes-workflow · ветка feat/workflow-modes
autonomy off — НЕ пушено. Два отдельных коммита, как заказано.

## Коммиты
- ШАГ 1 (companion bugfix-base scoped): `1102198716d058233e641fc566569eb2a13f4e5f`
- ШАГ 2 (врезка lite-base в 6 dev-ролей + ROLE_LANES): `3f6aa5af641d0f3d154b78affa2710f63f6ad2ba`
- (родитель: 2ab690f — сам лейн tacticum-lite-base; research уже был в ветке)

---

## (1) ШАГ 1 — дифф bugfix-base (дословно, до/после каждого места)

Инвариант **restore→/fix-bug НЕ тронут**. Разведён ТОЛЬКО маршрут «change intended behaviour». Процедура bugfix (diagnose→repro→fix→verify) не трогалась. Fragile-zone-триаж не трогался.

### 1a. SKILL.md — rule of thumb (было строки 33-35)

ДО:
```
> Rule of thumb: **restore** intended behaviour → `/fix-bug`. **Change** intended
> behaviour → `/start-task`. If you cannot state the one-line "correct behaviour it
> should have had", it's probably not a bug — reconsider the lane.
```
ПОСЛЕ:
```
> Rule of thumb: **restore** intended behaviour → `/fix-bug`. A **small change** of
> behaviour (no new screen/flow/module, within lite limits) → `/lite-task`; a change
> that needs the full design (new screen/flow/module, an ADR-level decision, a
> dependency outside the version catalog, a server contract, or scope > ~10 files /
> > 3 modules) → `/start-task`. If you cannot state the one-line "correct behaviour it
> should have had", it's probably not a bug — reconsider the lane.
```

### 1b. SKILL.md — Scope tripwire (было строки 142-152)

ДО:
```
## Scope tripwire — hand back to /start-task

Stop and **pause** the moment the fix reveals it is not a fix:

- it would **change intended/public behaviour** (feature, not restoration),
- it needs an **ADR-level** decision or a new pattern,
- it touches a **fragile-zone guard** or spreads across **many modules**.

On trip: do **not** silently continue. Tell the developer what you found and offer to
switch to `/start-task` (the full design lane). Respect their intent — don't surprise
them with heavy docs.
```
ПОСЛЕ:
```
## Scope tripwire — hand back to /lite-task or /start-task

Stop and **pause** the moment the fix reveals it is not a fix:

- it would **change intended/public behaviour** (a feature, not restoration) — offer
  **/lite-task** for a small change (no new screen/flow/module, within lite limits), or
  **/start-task** if it needs the full design (new screen / new user flow / new module,
  an ADR-level decision, a dependency outside the version catalog, a server contract, or
  scope > ~10 files / > 3 modules),
- it needs an **ADR-level** decision or a new pattern → `/start-task`,
- it touches a **fragile-zone guard** or spreads across **many modules** → `/start-task`.

On trip: do **not** silently continue. Tell the developer what you found and offer to
switch to `/lite-task` (a small change) or `/start-task` (the full design lane). Respect
their intent — don't surprise them with heavy docs, and don't silently turn a bug-fix
into a feature.
```
Примечание: два escalation-буллета (ADR / fragile-zone|many-modules) сохранены как есть, только явно проставлен их таргет `/start-task`. Split применён строго к буллету «change intended behaviour».

### 1c. fix-bug.md — Scope tripwire (было строки 51-52)

ДО:
```
> **Scope tripwire:** if the fix would *change* intended behaviour, needs an ADR, or
> touches a fragile zone / many modules — STOP, tell me, and offer `/start-task`.
> Do not silently turn a bug-fix into a feature.
```
ПОСЛЕ:
```
> **Scope tripwire:** if the fix would *change* intended behaviour, needs an ADR, or
> touches a fragile zone / many modules — STOP, tell me, and offer `/lite-task` (a small
> change: no new screen/flow/module, within lite limits) or `/start-task` (if it needs
> the full design). Do not silently turn a bug-fix into a feature.
```

Формулировки лайт-лимитов взяты дословно консистентно с tacticum-lite-base/SKILL.md (escalation-список: new screen/dialog, new user flow, new module, dep outside version catalog, server-contract change, scope > ~10 files / > 3 modules).

---

## (2) ШАГ 2 — врезка lite-base

### ПОЛНЫЙ дословный дифф ROLE_LANES (apps/backend/tests/catalog/test_iva_role_presets.py)

Во всех 6 dev-ролях вставлена строка `"tacticum-lite-base",` РОВНО между `"tacticum-bugfix-base",` и `"tacticum-research-base",` (order-sensitive — тест `test_depends_on_is_the_declared_lanes_in_order` требует depends_on == lanes точно по порядку).

```diff
     "iva-role-go": [
         "tacticum-core-base",
         "tacticum-development-core",
         "iva-go-development-base",
         "tacticum-bugfix-base",
+        "tacticum-lite-base",
         "tacticum-research-base",
     ],
     "iva-role-kmp": [
         "tacticum-core-base",
         "tacticum-development-core",
         "iva-kmp-development-base",
         "tacticum-bugfix-base",
+        "tacticum-lite-base",
         "tacticum-research-base",
         "tacticum-ui-base",
     ],
     "iva-role-web": [
         "tacticum-core-base",
         "tacticum-development-core",
         "iva-web-development-base",
         "tacticum-bugfix-base",
+        "tacticum-lite-base",
         "tacticum-research-base",
         "tacticum-ui-base",
     ],
     "iva-role-mail": [
         "tacticum-core-base",
         "tacticum-development-core",
         "iva-mail-development-base",
         "tacticum-bugfix-base",
+        "tacticum-lite-base",
         "tacticum-research-base",
         "tacticum-ui-base",
     ],
     "iva-role-ios": [
         "tacticum-core-base",
         "tacticum-development-core",
         "iva-ios-development-base",
         "tacticum-bugfix-base",
+        "tacticum-lite-base",
         "tacticum-research-base",
         "tacticum-ui-base",
     ],
     "iva-role-java": [
         "tacticum-core-base",
         "tacticum-development-core",
         "iva-java-development-base",
         "tacticum-bugfix-base",
+        "tacticum-lite-base",
         "tacticum-research-base",
     ],
```

`iva-role-analyst` НЕ тронут (нет dev/bugfix-лейнов — lite=код-правки ей не положен; осознанная асимметрия с research). tacticum-role-internal/platform, firebird-role-web НЕ тронуты (нет пары bugfix+research). iva-analysis-base НЕ тронут (guardrail Diaret). _mirrors.yaml НЕ тронут.

### 6 ролей — depends_on до/после (манифесты)

Во всех 6 добавлена строка `  - tacticum-lite-base` сразу после `  - tacticum-bugfix-base`, перед `  - tacticum-research-base`. Пример (iva-role-go):

```diff
 depends_on:
   - tacticum-core-base
   - tacticum-development-core
   - iva-go-development-base
   - tacticum-bugfix-base
+  - tacticum-lite-base
   - tacticum-research-base
```

| Роль | depends_on ДО | depends_on ПОСЛЕ |
|---|---|---|
| iva-role-go | core, development-core, iva-go-development-base, bugfix-base, research-base | + lite-base между bugfix и research |
| iva-role-ios | core, development-core, iva-ios-development-base, bugfix-base, research-base, ui-base | + lite-base между bugfix и research |
| iva-role-java | core, development-core, iva-java-development-base, bugfix-base, research-base | + lite-base между bugfix и research |
| iva-role-kmp | core, development-core, iva-kmp-development-base, bugfix-base, research-base, ui-base | + lite-base между bugfix и research |
| iva-role-mail | core, development-core, iva-mail-development-base, bugfix-base, research-base, ui-base | + lite-base между bugfix и research |
| iva-role-web | core, development-core, iva-web-development-base, bugfix-base, research-base, ui-base | + lite-base между bugfix и research |

---

## (3) Числа тестов (без docker, .venv/bin/python -m pytest)

- test_iva_role_presets.py — **91 passed**
- test_manifest_schemas.py — **38 passed**
- test_role_replacement_parity.py — **82 passed**
- Вместе: **211 passed** (0 fail, 0 skip). Порядок depends_on↔ROLE_LANES сошёлся, single-owner disjoint и golden-parity зелёные (lite-base не пересекается по ingredient_id с bugfix/research/ui — иначе упал бы test_single_owner_lanes_are_pairwise_disjoint).

---

## (4) Хеши коммитов
- ШАГ 1: `1102198716d058233e641fc566569eb2a13f4e5f`
- ШАГ 2: `3f6aa5af641d0f3d154b78affa2710f63f6ad2ba`

---

## (5) Риски / находки

- **[Находка — на решение тимлида/ГД, вне scoped-правки] SKILL.md строки 26-31** (раздел «When this lane — and when NOT») по-прежнему содержит буллет-список «Do **not** use it — go to `/start-task` — when the task: introduces or changes **intended** behaviour (that's a feature, not a fix)». Это ТА ЖЕ маршрутная формулировка (change→/start-task), которую мы разводили в rule-of-thumb и tripwire, но она НЕ входила в названные тимлидом места правки (33-35 и 142-152). Я её НЕ трогал, чтобы не расширять объём молча. Итог: внутри файла осталась лёгкая несогласованность — этот список всё ещё гонит любое change→/start-task, тогда как rule-of-thumb/tripwire теперь мелкое change→/lite-task. Нужно решение: доводить ли split и здесь (тогда буллет «introduces or changes intended behaviour» → расщепить на lite/start-task) или оставить как «широкий дисклеймер». Правка тривиальна, могу дослать отдельным коммитом по отмашке.
- Инвариант restore→/fix-bug сохранён во всех трёх местах — bugfix остаётся про восстановление наблюдаемого поведения.
- lite-base врезан order-consistent (bugfix → lite → research → [ui]) во всех 6 ролях и в ROLE_LANES; парити-тесты это подтверждают.
- iva-role-analyst намеренно без lite (dev-лейнов нет) — зафиксировано в теле коммита ШАГ 2.
- Коммиты подписались автором «Александр Шульга» (git config ворктри); GPG-подпись отключена флагом на коммит. AI-футеров/Co-Authored-By нет.
- НЕ пушено (autonomy off). Merge/PR — за пользователем.