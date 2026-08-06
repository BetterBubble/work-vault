---
title: impl-fix-role-web-coverage
type: report
permalink: tacticum/00-board/impl-fix-role-web-coverage-1
status: draft
archived-at: 2026-08-03 11:16
---

# impl-fix-role-web-coverage

**Задача:** починить красный main CI после PR-A — `test_role_covers_replaced_profile[iva-role-web<-iva-web-brownfield]` FAILED: роль `iva-role-web` не покрывала 2 новых навыка PR-A (`angular-ds-component-authoring`, `angular-ds-component-usage`, brownfield-only).

**Worktree/ветка:** `~/tacticum/tacticum-dev-fix-rolecov` на `fix/role-web-coverage` (от свежего `origin/main` c6be10a). autonomy off, НЕ пушил.

## Что прочитал в тесте (механика + allowlist)

Тест `test_role_covers_replaced_profile` живёт в `apps/backend/tests/catalog/test_role_replacement_parity.py` (НЕ в `test_iva_role_presets.py`, как было в исходной формулировке — уточняю для протокола).

- **Механика подтверждает диагноз лида.** `composed = _composed_ids(role)` = собственные пак-ингредиенты роли + union `_ids(lane)` по всем `depends_on`. `old = ids(brownfield)` (с применением RENAMES). `lost = old − composed − allowlist` → должно быть пусто.
- **Allowlist ЕСТЬ** — `REPLACEMENTS[(role, profile)]`. Для web = `_implement_only_allowlist({helm-analyst, iva-read})`, т.е. только analysis-навыки (ADR-0059 Р.6 — постановка не входит в implement-only dev-роль). Мои 2 DS-навыка — dev-навыки, НЕ analysis → в allowlist входить не должны, обязаны покрываться композицией. Значит вариант «добавить в allowlist» неверен; правильный — врезать в лейн. План лида верен, STOP не потребовался.
- Сопутствующие: `test_allowlist_entries_are_real_gaps` (каждая строка allowlist всё ещё реальный gap) и `test_mirror_content_is_byte_identical` (только по парам из `_mirrors.yaml`).

## Решение (как в замысле лида)

Врезал 2 навыка в `iva-web-development-base` (composed только `iva-role-web` → нет протечки в kmp/ios/mail; не в `tacticum-ui-base` — Angular-специфика протекла бы в не-веб роли).

1. Скопировал 2 `SKILL.md` из `templates/iva-web-brownfield/ingredients/skills/<skill>/` в `templates/iva-web-development-base/ingredients/skills/<skill>/` — **байт-в-байт** идентичны (diff чист).
2. Добавил 2 записи `skill_spec` в `templates/iva-web-development-base/manifest.yaml`. **Формат base ОТЛИЧАЕТСЯ от brownfield:** base не несёт copilot (`ide_targets.copilot: unsupported`) → `supports: [claude-code, codex]`, без `copilot_target_path`. `tier: trial`, `install_scope: user` — как в base-образцах. description_trigger скопированы из brownfield дословно.
3. Bump `iva-web-development-base` 0.1.0 → 0.1.1 + CHANGELOG (Fixed).

## Проверка mirror (step 4)

Пара `iva-web-development-base(owner)→iva-web-brownfield(mirror)` в `_mirrors.yaml` эти 2 навыка в списке `ingredients` НЕ содержит (как и существующие DS-навыки brownfield — они не зеркалятся). В пару НЕ добавлял. `check_mirror_sync.py` зелёный без добавления → ок, STOP не потребовался.

## Изменённые файлы (git diff --stat)

```
 templates/iva-web-development-base/CHANGELOG.md    |  10 ++
 .../skills/angular-ds-component-authoring/SKILL.md | 169 ++++++++++++++++++++
 .../skills/angular-ds-component-usage/SKILL.md     | 177 +++++++++++++++++++++
 templates/iva-web-development-base/manifest.yaml   |  26 ++-
 4 files changed, 381 insertions(+), 1 deletion(-)
```

## Верификация

**Целевая параметризация:**
```
test_role_covers_replaced_profile[iva-role-web<-iva-web-brownfield]  PASSED
test_allowlist_entries_are_real_gaps[iva-role-web<-iva-web-brownfield]  PASSED
2 passed in 0.12s
```

**Статические каталог-тесты (parity + presets + smoke, DB-free):**
```
252 passed in 5.41s
```

**Валидаторы:**
```
check_mirror_sync.py:                          OK — 64 зеркальных ингредиентов в 6 парах синхронны. (exit 0)
check_profile_version_discipline.py (static):  OK — 48 profile(s) clean. (exit 0)
  ... --diff-against origin/main:              OK — 48 profile(s) clean. (exit 0)
```

**Полный `pytest tests/catalog/`:** `2 failed, 549 passed, 4 warnings, 120 errors`.
⚠️ **Все 2 failed + 120 errors — инфраструктурные, НЕ регрессия.** Причина: Postgres на :5432 не поднят (Docker daemon не запущен) — `OSError: Connect call failed … 5432` / `getaddrinfo() returned empty list`. Это seed/tenant/init/admin DB-тесты, требующие живую БД. Правка статичных template YAML/md логически не может вызвать ошибку подключения к БД. Поднять БД локально не смог — docker daemon down. Все DB-free тесты (в т.ч. вся parity/coverage/overlap/mirror-логика) зелёные.

## Коммит

`e1f8394f73a5525176bd85b6a082fdf7f9d745a9` — `fix(roles): cover angular-ds-component skills for iva-role-web (main CI green)` (без AI-подписей, НЕ запушен).

## Самопроверка

- Целевой `test_role_covers_replaced_profile[iva-role-web<-iva-web-brownfield]` теперь PASSED. ✅
- Ничего другого не сломано: все DB-free каталог-тесты зелёные, оба валидатора зелёные, mirror byte-identical держится. ✅
- Открытый вопрос для лида (не блокер): DB-зависимые каталог-тесты локально не прогнаны (Docker down) — CI их прогонит с БД; правка их не касается.