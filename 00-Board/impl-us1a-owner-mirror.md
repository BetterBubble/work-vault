---
status: draft
role: implementer
for: lead-fr
topic: US#1-А — fr-authoring FR v2 в владельце iva-analysis-base + зеркало iva-fr-analyst (фиксы critic A/B/C/E)
repo: /Users/bubblemac/tacticum/tacticum-dev
worktree: /Users/bubblemac/tacticum/tacticum-dev-us1-owner
branch: feat/us1-fr-authoring
base: origin/main @ abed800 (revert US#0 внутри)
commit: c028882 (amend поверх 19813f6 — фикс A протянут в стоп-гейт шага 9 + п.(iii))
date: 2026-07-24
permalink: tacticum/00-board/impl-us1a-owner-mirror
---

# US#1-А — реализация (модель А: правим владельца, зеркало байт-в-байт)

## Worktree / ветка
- **worktree:** `/Users/bubblemac/tacticum/tacticum-dev-us1-owner`
- **ветка:** `feat/us1-fr-authoring` (от origin/main @ abed800)
- **коммит:** `19813f6`
- **autonomy off:** локально, НЕ push / НЕ merge / НЕ PR.

## Что сделано (человеческим языком)
База — контент fr-authoring/SKILL.md из edc5cc2 (worktree tacticum-dev-fr-us1, 730 строк). Применил к нему 4 фикса critic (A/B/C/E) и записал итог **байт-в-байт в оба файла**:
- ВЛАДЕЛЕЦ: `templates/iva-analysis-base/ingredients/skills/fr-authoring/SKILL.md`
- ЗЕРКАЛО: `templates/iva-fr-analyst/ingredients/skills/fr-authoring/SKILL.md`

Содержательно скилл теперь несёт FR v2: FT-n/UC-n живут в Части 1 (§1.4/§1.5), UI в §1.9, §2 двухзонная (Приложение строгое as-is; проектные разделы Части 1 допускают проектные wire-имена под тремя предохранителями), валидатор границы как чек-лист pass/fail, маркер `fr_skeleton: 2`, плейсхолдеры §1.7/§1.8. Форматы и нумерация FT-n/UC-n не тронуты (сквозная трассировка конвейера сохранена). Изменение строго аддитивное — существующий as-is флоу аналитика не ослаблен.

### Применённые фиксы critic
- **A (критично, дыра Q-n-байпаса):** валидатор п.(ii) переформулирован — основание обязательно ВСЕГДА, `Q-n` его не заменяет. Плашка/Q-n вынесены в п.(iii). Дыра «имя с Q-n, но без основания проходит» закрыта.
- **B (мелочь):** §2 непроектные разделы — «**проектные** wire-имена не вводятся» (подтверждённые код-verify/каноном имена — факт, их допускает валидатор (iv)).
- **C (align к ТЗ инв.3):** в §2 добавлена фраза про проектный UI под той же дисциплиной через существующие UI-навыки; §1.9 держит as-is. Новых US/навыков НЕ создавал, UI вне скоупа НЕ выносил.
- **E:** валидатор остался «чек-лист pass/fail»; regex в контенте нет (проверено grep — 0 совпадений), обещаний regex не добавлял.

### Version bump + CHANGELOG
- `iva-analysis-base`: 0.1.3 → **0.1.4** + запись CHANGELOG (FR v2, человеческим языком).
- `iva-fr-analyst`: 0.1.10 → **0.1.11** + запись CHANGELOG (+ отметка о зеркале владельца).

## ПОДТВЕРЖДЕНИЕ: owner ↔ mirror identical
```
diff -q templates/iva-analysis-base/.../fr-authoring/SKILL.md \
        templates/iva-fr-analyst/.../fr-authoring/SKILL.md
→ identical
```

## ИТОГОВЫЙ текст валидатора п.(ii) после фикса A (проверьте, что дыра закрыта)
> - (ii) **Каждое wire-имя проектных разделов Части 1 (§1.6/§1.7/§1.8) обосновано.**
>   Имя трассируется к основанию в Приложении (→ П.F проба / П.E код-verify):
>   конвенция ближайшего реестра / аналог / negative evidence («операции нет →
>   проектируем»). → *fail*, если wire-имя проектного раздела без основания.
>   (Основание обязательно ВСЕГДА — `Q-n` его не заменяет; плашку и `Q-n`
>   проверяет п.(iii).)

Fail-условие теперь безусловное: «без основания» → fail, точка. `Q-n` больше не открывает байпас.

## Фраза про UI (фикс C, в §2 «Правила честности»)
> Проектный UI (при необходимости, инв.3) подчиняется ТОЙ ЖЕ зонной дисциплине
> (основание + плашка + валидатор) и производится СУЩЕСТВУЮЩИМИ UI-навыками
> (`mockup-authoring` / `design-system-discovery`); §1.9 в fr-authoring держит
> только as-is описание UI из фактов Приложения.

## Результаты проверок
1. **check_mirror_sync.py** → `OK — 62 зеркальных ингредиентов в 6 парах синхронны.` (fr-authoring снова в зеркале, owner↔mirror байт-идентичны).
2. **diff -q owner vs mirror** → `identical`.
3. **check_profile_version_discipline.py --diff-against origin/main** → `OK — 46 profile(s) clean.`
4. **pytest apps/backend/tests/catalog/ --noconftest** (venv apps/backend, PYTHONPATH=src):
   - целевой набор (test_manifest_schemas + test_role_replacement_parity + test_iva_role_presets) → **211 passed**;
   - весь каталог → **547 passed, 120 errors, 2 failed**. Все 120 errors и 2 failed — Postgres-зависимые тесты (connection refused 127.0.0.1:5432 / SQLAlchemy); Postgres в среде не поднят. К правке markdown-контента отношения не имеют (подтверждено: FAILED — SQLAlchemy `_connection_for_bind`). Не-DB content-тесты все зелёные.

## git
- `git diff --stat origin/main..HEAD`: 6 files, +458 −178 (SKILL.md ×2 по +291/−, manifest ×2, CHANGELOG ×2).
- `git status`: clean (всё закоммичено, лишних/мусорных файлов нет).
- `git log origin/main..HEAD --oneline`: `19813f6 feat(fr-authoring): FR v2 двухзонная модель + валидатор границы (US#1-А, owner+mirror)`.

## Довесок (повторный critic — фикс A протянут в стоп-гейт)
Правки поверх 19813f6 (amend, версии не менялись, diff --stat тот же):
- **Стоп-гейт шага 9 (строка 283):** было «wire-имя проектного раздела без основания **и без `Q-n`**» → стало «wire-имя проектного раздела **без основания**; отсутствие плашки…». Конъюнкция, реоткрывавшая дыру (имя с Q-n но без основания), убрана — гейт согласован с валидатором п.(ii).
- **Валидатор п.(iii) (строки 183–185):** в fail-условие дописан открытый `Q-n`: «несёт плашку … **и открытый `Q-n`**. → *fail*, если имена есть, а плашки и/или открытого `Q-n` нет.» Приведено в соответствие с примечанием п.(ii) и §2(б).
- HEAD после amend: **c028882**. check_mirror_sync OK (62), diff -q owner↔mirror identical, version-discipline clean.

## Развилки / флаги
- **Фикс C — существующие UI-навыки:** реализовал как отсылку §2 к `mockup-authoring`/`design-system-discovery` под той же дисциплиной (основание+плашка+валидатор), §1.9 держит as-is. НЕ проверял по коду навыков, реально ли они физически производят «дисциплинированный проектный UI» (это US#3-зона/сверх-US#1). Если лид/critic сочтёт, что это надо верифицировать — отдельный флаг ГД (интент Солонко), сам навык не выдумывал.
- **Куплинг US#1↔US#4** (переезд FT/UC ↔ чтение конвейером) — как в спеке: держу переходную совместимость по `fr_skeleton: 2`, форматы/нумерацию не менял. Окно с US#4 — на ГД, вне моего скоупа.
- Контент сверх фиксов A/B/C/E не трогал (edc5cc2 апрувлен). Другие скиллы (api-contracts-discovery / design-system-discovery / mockup-authoring) не трогал.
