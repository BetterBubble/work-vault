---
status: draft
role: implementer
topic: ТЗ#3 US#4 Проход E — kmp pin/tests К-2/К-4 (независимая часть E)
repo: /Users/bubblemac/tacticum/tacticum-dev
worktree: /Users/bubblemac/tacticum-worktrees/us4-passE-kmp
branch: feat/us4-passE-kmp-pintests
base: origin/main @ 5884bcd (ребейз на свежий main; было 8831a00)
head: 8b0e3c9
date: 2026-07-24
permalink: tacticum/00-board/impl-us4-passE-kmp-pintests
---

> **UPD 2026-07-24 (после controller PASS + critic):** фикс консистентности токена
> tests + ребейз на свежий main. Итоги — в разделе «UPD» внизу.

# US#4 Проход E — iva-kmp-brownfield pin/tests К-2/К-4

Готово. Ветка `feat/us4-passE-kmp-pintests` от свежего origin/main. НЕ push. Скоуп —
ТОЛЬКО kmp pin/tests+manifest+CHANGELOG (brd/start-task не трогал — отдельные части E).

## Что сделано (под KMP/Compose-стек)

**pin-authoring/SKILL.md (К-2):** новая секция «Проектные серии — реализация и
расхождения (К-2/К-4, только v2-FR)». На v2-FR PIN реализует серии по стабильному ID:
- `CT-n` (§1.6 контракт) → контракт в `commonMain` (`interface`/`expect`-`actual` или
  codegen), платформенные `actual` в `androidMain`/`iosMain`/`jvmMain`/`jsMain`;
- `DM-n` (§1.7 модель) → `data class` / `sealed interface` / `enum`, MVI-State + редьюсер;
- `EV-n` (§1.8 событие) → `Flow`/`Channel`/MVIKotlin `Label`, идемпотентность/порядок.
- Статус-таблица `| ID | раздел | что реализуется (KMP) | статус |` словами единого
  словаря ТЗ `реализован`/`расхождение`/`blocked`.
- **К-3 уважает:** раздел реализуется только при утверждённом `D-n` (гейт start-task,
  Проход C-канон); иначе честный `blocked` с форвардом на start-task-гейт, без имитации.
- v1-FR — серий нет → шаг пропускается.

**pin-authoring/SKILL.md (К-4):** таблица расхождений FR↔KB расширена на проектные
разделы. Проект контракта/модели/события сверяется с реальным кодом через
`kb_verify_api_exists` (+ `kb_get_code_context`, `kb_get_task_context`, codegraph);
конфликт = **критичное расхождение** строкой `| ID раздела | проект FR | KB/код |
критичность | предложение |`, форвард владельцу FR (аналитику), не молчаливая перезапись.
Надстроено над существующим механизмом расхождений, а не заменяет его.
+ правило в Rules и 3 анти-паттерна (не переномеровывать CT/DM/EV, не имитировать
blocked, не перезаписывать конфликт молча).

**tests-authoring/SKILL.md (К-2):** новая секция «Проектные серии — контрактные тесты».
Контрактные тесты `Covers: CT-n`/`DM-n`/`EV-n` под KMP (`kotlin.test`/assertk/Turbine
в `commonTest`, `TestStoreFactory`, Compose-test). Статус-таблица серий
`| ID | раздел | Covers | статус |`.

**Статус-вокаб по эталону rn** — словарь сопоставлен на исход теста:
| словарь | исход теста |
|---|---|
| `реализован` | pass |
| `расхождение` | **xfail** (`@Ignore` c причиной-ссылкой на строку расхождения FR↔KB, либо assert на зафиксированное фактическое поведение) |
| `blocked` | `@Ignore("blocked: D-n не утверждён, форвард на start-task-гейт")` (К-3) |

Явно: `расхождение` ≠ «тест сломан» — осознанный xfail с трассировкой.
+ правило и 2 анти-паттерна (стабильный ID в `Covers:`, не ронять молча).

**manifest.yaml:** 0.5.0 → 0.5.1. **CHANGELOG.md:** запись [0.5.1] с К-2/К-4 под KMP.

## Backward-safe

Аддитивно: v1-FR (нет проектных разделов) → pin/tests работают как раньше — все новые
секции гейтятся «только v2-FR». Стек-специфику существующего pin/tests (Verified/NEW-
маркеры, API Verification Gate, Testing Trophy, `:core:test-utils` fakes, Store/Compose
тесты) не тронул — К-2/К-4 надстроены сверху.

## Развилки (durable)

1. **kmp brd в main НЕ регистрирует серии CT-n/DM-n/EV-n** (grep пусто; kmp brd diverged
   ждёт ack ГД — вне моего скоупа). pin/tests написаны как надстройка «если серии есть в
   FR/BRD» — трассировка по стабильному ID из FR §1.6/1.7/1.8. Когда kmp brd-регистрация
   серий приземлится, конвейер сомкнётся без правок pin/tests. Backward-safe до тех пор.
2. **passB2 pin/tests НЕ в main** (rn всё ещё 0.5.2, эталонного статус-вокаба в rn tests
   пока нет). Статус-вокаб авторил по ТЗ-токенам + указанному маппингу (`расхождение`=xfail),
   консистентно с sibling-намерением rn. Если rn финализирует иную формулировку — свериться
   при сборке батареи.
3. **Вокаб серий** взял канонически из mail brd-authoring (§1.6 CT-n / §1.7 DM-n / §1.8 EV-n),
   т.к. mail — единственный профиль в main с зарегистрированными сериями (эталон именования).

## Проверки

- `check_profile_version_discipline.py --diff-against origin/main` → **OK — 48 profile(s) clean.**
- pytest (PYTHONPATH=apps/backend, --noconftest): `test_manifest_schemas.py` +
  `test_iva_role_presets.py` + `test_role_install_smoke.py` → **206 passed, 0 failed**
  (все точки, 100%). Профильного `test_iva_kmp` в catalog нет (есть только golden
  `iva-role-kmp` для e2e_install — покрыт install_smoke).
- Предсущ. red iva-role-web (#149) в моих целевых тестах не всплыл — они зелёные.

## git

```
git diff --stat origin/main..HEAD
 templates/iva-kmp-brownfield/CHANGELOG.md                          | 23 +++++++++
 .../ingredients/skills/pin-authoring/SKILL.md                      | 56 ++++++++++
 .../ingredients/skills/tests-authoring/SKILL.md                    | 41 ++++++++++
 templates/iva-kmp-brownfield/manifest.yaml                         |  2 +-
 4 files changed, 121 insertions(+), 1 deletion(-)

git log origin/main..HEAD --oneline
 0b9ca13 feat(iva-kmp-brownfield): pin/tests проектные серии CT-n/DM-n/EV-n (К-2/К-4)

git status → clean (всё закоммичено)
```

НЕ push / НЕ merge / без AI-подписей. Готово к ревью лида и сборке в общую ветку Прохода E.

## UPD — фикс консистентности (critic) + ребейз на свежий main

### 1. Фикс токена tests (critic-дефект)
В `tests-authoring/SKILL.md` статус-токен покрытия тестом `реализован` → **`covered`**
(эталон — 4 B2-профиля в main mail/rn/ios/firebird: `covered` = «покрыто тестом»;
`реализован` — концепт PIN, не TESTS). Затронуто: строка таблицы статусов (`covered`=pass),
словарь в Rules, и tests-маппинг в CHANGELOG. Проза «раздел не реализован» (причина
`blocked`) — глагол, не статус-токен, оставлена как есть.
- **pin-authoring НЕ тронут:** `реализован` там корректен (pin-концепт) — подтверждено
  grep: pin = 3× `реализован`, 0× `covered`.
- **tests:** grep `реализован` как статус-токена больше нет; словарь = `covered`/`расхождение`/`blocked`.

### 2. Ребейз на свежий main
`git fetch origin main` → `git rebase origin/main`. Ветка отставала (main ушёл вперёд).
Реплеился ровно 1 мой коммит, **конфликтов не было** (main-коммиты трогали не
pin/tests-конвейер kmp).

**Версия kmp примирена:** main kmp-manifest = **0.5.0** (axis-2/#145 в этот manifest не
поднял версию) → моя **0.5.1 остаётся** валидным следующим шагом. Примирение версии не
потребовалось.

### Проверки (после фикса+ребейза)
- HEAD = **8b0e3c9**; base = 5884bcd (свежий origin/main).
- `git log origin/main..HEAD --oneline` → **1 коммит** (мой, kmp pin/tests).
- `git diff --stat origin/main..HEAD` → **ровно 4 файла kmp**, чисто (без чужих коммитов):
  ```
  CHANGELOG.md                                   | 23 +++++++++
  ingredients/skills/pin-authoring/SKILL.md      | 56 ++++++++++++++++++++++
  ingredients/skills/tests-authoring/SKILL.md    | 41 ++++++++++++++++
  manifest.yaml                                  |  2 +-
  4 files changed, 121 insertions(+), 1 deletion(-)
  ```
- version-discipline `--diff-against origin/main` → **OK — 48 profile(s) clean.**
- pytest целевые (schemas+role_presets+install_smoke, PYTHONPATH=apps/backend, --noconftest)
  → **206 passed**. (Полный `catalog/` даёт падения только в DB-тестах — Postgres :5432 не
  поднят локально — и авторинг-тестах: инфра, вне скоупа; целевые схемные — зелёные.)
- kmp `реализован` в tests как статус-токена нет; pin `реализован` не тронут.

НЕ push. Без AI-подписей. Готово к ре-ревью.

## Связано
[[spec-us4-passB2-pintests]]
</content>
</invoke>
