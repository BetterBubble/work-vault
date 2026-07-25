---
title: impl-prc-consolidation
type: report
permalink: tacticum/00-board/impl-prc-consolidation
status: draft
role: implementer
for: lead-ds
tz: figma-ds ТЗ#1, PR-C = ось-1, консолидация (iva-core manifest + G7 + G8 + версия/CHANGELOG)
worktree: ~/tacticum/tacticum-dev-web-axis1 (ветка feat/ds-web-axis1, стек на PR-B)
autonomy: false
tags:
- ds-web
- figma-ds
- pr-c
- axis1
- iva-core
- consolidation
- implement
- blocker
---

# impl-prc-consolidation — консолидация PR-C (ось-1): iva-core в манифест, G7-trigger, G8, версия/CHANGELOG

⚠️ **СТАТУС: НЕ закоммичено, СТОП по правилу тест-сета.** Все правки iva-web-brownfield сделаны и корректны (schema/mirror/version — зелёные), НО добавление скилла `iva-core-design-system` в профиль механически валит контент-тест **role-coverage parity** — роль `iva-role-web` больше не покрывает свой замещаемый профиль. Фикс требует правки **`iva-web-development-base`** (mirror-**owner**-лейн) — явно вне моих границ («owner/др.пакеты»). Отчёт лиду на решение (см. §Блокер).

## Что сделано (worktree, НЕ commit)

`git diff --stat` (tracked):
```
 templates/iva-web-brownfield/CHANGELOG.md          | 37 +++++++++++++++++++++
 .../skills/design-system-discovery/SKILL.md        | 27 +++++++++++----   (G7, уже был modified)
 templates/iva-web-brownfield/manifest.yaml         | 18 ++++++++--
 3 files changed, 72 insertions(+), 10 deletions(-)
```
Untracked (новые, авторены параллельными воркерами — подключены/оставлены):
```
?? templates/iva-web-brownfield/ingredients/skills/iva-core-design-system/SKILL.md   (iva-core воркер)
?? docs/user_manuals/iva-web-figma-mapping-quickstart.md                             (G8)
```

### 1. manifest.yaml — iva-core-design-system + G7-trigger + счётчик
- (а) Добавлена запись `kind: skill_spec` `iva-core-design-system` сразу после `design-token-usage`, по образцу соседних DS-навыков: `tier: trial`, `install_scope: user`, `supports: [claude-code, codex, copilot]`, три target-path (claude/copilot/codex), `body_path: ingredients/skills/iva-core-design-system/SKILL.md`, `metadata.description_trigger` — из отчёта impl-iva-core-skill (конференц/call/VCS-поверхность, `from 'iva-core'`, отличия от `@iva/design-system`, триггеры iva-core/iva-connect/VCSWEB/get-color/--primary-color/iva-datepicker/iva-grid/iva-seeker).
- (б) Обновлён `description_trigger` записи `design-system-discovery` (по предложению G7): добавлен surface-router — «main iva-one UI → @iva/design-system; conference / MCU / VCS / iva-connect → iva-core» + выбор ДС по platform/framework_hint.
- (в) Счётчик комментария `# --- skill_spec (28) ---` → `(29)`.

### 2. G8-quickstart — в manifest НЕ регистрирую (по образцу)
Проверен образец `docs/user_manuals/iva-kmp-figma-mapping-quickstart.md`: **НЕ** зарегистрирован ни в одном manifest (grep по `templates/*/manifest.yaml` — 0). Значит `iva-web-figma-mapping-quickstart.md` — обычный user-manual doc, в manifest НЕ регистрируется (просто файл в git). Manifest для G8 не трогал. ✅ по образцу.

### 3. Версия + CHANGELOG
- `version: "0.4.0"` → `"0.5.0"`.
- Секция CHANGELOG `[0.5.0] — 2026-07-24`: **Added** — iva-core thin skill (3 факта + иллюстративный каталог + deferred server-resolve), web figma-mapping quickstart (user-manual, не ингредиент); **Fixed** — design-system-discovery surface-split + framework_hint-selection (ось-1 G7) + обновление его manifest description_trigger.

## Верификация — ПОЛНЫЙ тест-сет

venv создан: `cd apps/backend && uv sync --all-extras --dev` (pytest 9.1.1).

| Проверка | Итог |
|---|---|
| `pytest apps/backend/tests/catalog/ -q` | 120 ERROR = все требуют Postgres (`Connect call failed 127.0.0.1:5432`) — инфра, ожидаемо. |
| **test_manifest_schemas.py** (контент) | ✅ **passed** — новая запись iva-core skill_spec схемо-валидна. |
| **test_iva_role_presets.py** (контент) | ✅ **passed**. |
| **test_role_replacement_parity.py** (контент) | ❌ **1 FAILED** (см. Блокер) — mirror-content подтесты внутри файла passed. |
| `./scripts/check_mirror_sync.py` | ✅ `OK — 64 зеркальных ингредиентов в 6 парах синхронны` (iva-core/DS-навыки НЕ в mirror-списке → не затронуты). |
| `./scripts/check_profile_version_discipline.py` (static) | ✅ `OK — 48 profile(s) clean`. |
| `... --diff-against origin/main` | ✅ `OK — 48 profile(s) clean` (0.5.0 + CHANGELOG удовлетворяют дисциплину). |

## 🚧 Блокер — role-coverage parity (требует решения лида)

```
FAILED test_role_covers_replaced_profile[iva-role-web<-iva-web-brownfield]
AssertionError: iva-role-web не покрывает iva-web-brownfield:
потеряны ['iva-core-design-system'] — либо перенести в лейн, либо в allowlist
```

**Причина (не баг моей работы, а прямое следствие шага 1):** тест требует, чтобы всё из старого профиля `iva-web-brownfield` несла и роль `iva-role-web` (union её лейнов `depends_on`). Новый `iva-core-design-system` пока только в brownfield.

**Прецедент — как чинилось для DS-навыков:** коммит `e1f8394` "fix(roles): cover angular-ds-component skills for iva-role-web" добавил angular-ds-навыки в **`iva-web-development-base`** (лейн роли): body-файл(ы) SKILL.md + skill_spec в manifest + CHANGELOG + version bump лейна. DS-навыки **НЕ** входят в mirror-пару web (проверено `_mirrors.yaml`) — расходятся осознанно, mirror-sync не при чём.

**Точный фикс (для авторизации):** добавить `iva-core-design-system` в лейн `iva-web-development-base`:
1. `templates/iva-web-development-base/ingredients/skills/iva-core-design-system/SKILL.md` (копия body);
2. skill_spec в `templates/iva-web-development-base/manifest.yaml`;
3. CHANGELOG + version bump `iva-web-development-base`.

**Почему НЕ сделал сам:** ТЗ явно запрещает трогать «owner/др.пакеты»; `iva-web-development-base` = mirror-**owner** и отдельный пакет. Два явных сигнала (STOP-on-content-red + запрет owner-пакетов) → СТОП, не расширяю объём молча.

**Прошу лида:** (а) авторизовать правку `iva-web-development-base` (сделаю по прецеденту e1f8394, закоммичу обе части зелёными) — ЛИБО (б) взять покрытие роли на консолидацию. Альтернатива — allowlist в тесте с причиной, но по прецеденту правильнее перенести в лейн (роль реально должна нести конференц-ДС).

## Границы соблюдены
- ✅ НЕ трогал `tacticum-ui-base` / 6 копий discovery / `_mirrors.yaml` / owner-лейны / др.пакеты.
- ✅ НЕ трогал PR-B (`ui-mockup-match` — уже в базе ветки).
- ✅ G8 — manifest НЕ трогал (обычный user-manual, по образцу kmp).
- ⛔ НЕ commit, НЕ push (autonomy off + STOP по контент-тесту).

## Самопроверка
- ✅ iva-core skill_spec схемо-валиден (test_manifest_schemas passed).
- ✅ discovery description_trigger обновлён (surface-router).
- ✅ G8 по образцу — в manifest НЕ регистрируется (kmp-образец не зарегистрирован).
- ✅ версия 0.5.0 == секция CHANGELOG [0.5.0]; version-discipline diff-against origin/main зелёный.
- ✅ шаренное/PR-B/owner не тронуты.
- ⚠️ role-coverage красный → СТОП, ждёт решения лида (commit-hash будет после авторизации фикса).