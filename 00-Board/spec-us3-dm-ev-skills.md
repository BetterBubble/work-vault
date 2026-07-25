---
status: draft
role: lead-fr
topic: ТЗ#3 US#3 — спека implementer (навыки DM-n / EV-n, вариант А владелец+зеркало)
repo: /Users/bubblemac/tacticum/tacticum-dev
date: 2026-07-24
permalink: tacticum/00-board/spec-us3-dm-ev-skills
---

# US#3 — спека implementer: навыки-анализаторы модели данных (DM-n) и событий (EV-n)

**Источник истины:** ТЗ Солонко §2.3 (слоты §1.7/§1.8), §2.4 (единый контракт навыка-анализатора), §2.5 (жанр ТЗ). Дословный текущий текст ТЗ-скелета (разделы 7 «Доменные модели»/9 «События EV-nn») — в [[verbatim-fr-authoring-contracts-blocks]] (стр.538-546).
**Модель доставки — вариант А:** 2 новых skill_spec в ОБА профиля (владелец `iva-analysis-base` + зеркало `iva-fr-analyst`) байт-в-байт + в mirror-список.
**Предусловие:** пара US#1+US#2 смержена в main (§1.7/§1.8 плейсхолдеры от US#1 на месте). Ветка US#3 — от обновлённого main.
**Parity:** правка test-matrix/ROLE_LANES НЕ нужна (см. `explore-us3-parity-variant-a`). US#3 — чистый контент.

## Что делает (сужено реконсиляцией — концепты уже есть в ТЗ-скелете)
Концепты домен-моделей и событий УЖЕ живут в жанре ТЗ fr-authoring (разделы 7/9, серия EV-nn). US#3 — вынести их в ОТДЕЛЬНЫЕ навыки-анализаторы + внести в жанр FR (§1.7/§1.8) + унифицировать серию.

## Правки

### 1. Новый skill_spec `data-model-analyzer` (серия DM-n) — оба профиля байт-в-байт
- Директория: `templates/iva-analysis-base/ingredients/skills/data-model-analyzer/SKILL.md` + `templates/iva-fr-analyst/ingredients/skills/data-model-analyzer/SKILL.md` (папка = ровно ingredient_id).
- Вход: фактура Части 2; выход: раздел §1.7 «Модель данных и состояния» Части 1 с серией DM-n (единый контракт анализатора ТЗ §2.4 — определён в US#1, потребляем).
- Содержимое по образцу ТЗ-скелета раздел 7 «Доменные модели» (ролевая матрица операции×роли + правила/ограничения) + раздел 8 «Состояния». Проектные имена/модели — под двухзонной §2 (в §1.7 to-be под 3 предохранителями; фактура as-is в П.F/П.E).
- frontmatter: непустой `metadata.description_trigger` (схема ingredient.v1).

### 2. Новый skill_spec `events-analyzer` (серия EV-n) — оба профиля байт-в-байт
- Директория `.../skills/events-analyzer/SKILL.md` (оба профиля).
- Выход: раздел §1.8 «События и консистентность» с серией EV-n (публикация, идемпотентность, сверка, порядок — по образцу ТЗ-скелета раздел 9).
- ⚠️ **Консолидация нумерации (ТЗ):** ТЗ-скелет жанра ТЗ уже несёт серию `EV-nn` (раздел 9, fr-authoring). Унифицировать: EV-n (жанр FR) = EV-nn (жанр ТЗ) — ОДНА серия/формат ID, чтобы не плодить параллельные. Рефакторить раздел 9 ТЗ-скелета так, чтобы его порождал events-analyzer (единый источник нумерации). НЕ дублировать.

### 3. `_mirrors.yaml` — добавить id обоих навыков в первую пару
В `pairs[0].ingredients` добавить `data-model-analyzer`, `events-analyzer` → check_mirror_sync будет энфорсить байт-идентичность owner↔mirror по ним.

### 4. manifest.yaml обоих профилей — добавить 2 skill_spec + version bump + CHANGELOG
Добавить оба навыка в `ingredients` (skill_spec) обоих manifest. Bump версий обоих (patch) + CHANGELOG-записи (человеческим языком: анализаторы модели данных/событий, серии DM-n/EV-n, §1.7/§1.8 FR).

### 5. fr-authoring §1.7/§1.8 — связать плейсхолдеры с навыками
Плейсхолдеры §1.7/§1.8 (созданы US#1) — указать, что их порождают data-model-analyzer/events-analyzer, серии DM-n/EV-n. (Правка fr-authoring в обоих профилях байт-в-байт — согласованно.)

## Прод-safe
Аддитивно: новые навыки не ломают существующий флоу; ТЗ-скелет раздел 9 рефакторится на единый источник без потери семантики EV. as-is дисциплина сохраняется.

## Приёмка US#3
- 2 новых навыка в обоих профилях, папка=ingredient_id, непустой description_trigger, непустой SKILL.md.
- check_mirror_sync OK (оба навыка identical owner↔mirror + добавлены в пару); diff -q identical.
- Серия EV единая (EV-n=EV-nn), без параллельных.
- DM/EV генерят §1.7/§1.8 под двухзонной §2.
- version-discipline clean; целевые catalog-тесты (schemas+parity+role_presets+install_smoke) зелёные — parity ЗЕЛЁНЫЙ БЕЗ allowlist (DM/EV покрыты через владельца).
- critic §2 (DM/EV под теми же предохранителями) + controller-гейт.

## Связано
[[verbatim-fr-authoring-contracts-blocks]] · [[explore-us3-parity-variant-a]] · [[plan-tz3-us-breakdown-detailed]] · [[spec-us1-fr-authoring-v2]]
