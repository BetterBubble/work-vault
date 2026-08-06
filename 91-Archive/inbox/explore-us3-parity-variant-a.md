---
title: explore-us3-parity-variant-a
type: explore
permalink: tacticum/00-board/explore-us3-parity-variant-a-1
tags:
- explore
- us3
- parity
- variant-a
- test-matrix
archived-at: 2026-08-03 11:16
---

# explore-us3-parity-variant-a

status: draft
роль: explorer (read-only) для lead-fr, ТЗ#3 US#3
репо: /Users/bubblemac/tacticum/tacticum-dev @ 20412ff (origin/main, свежий)
задача: под ВАРИАНТ А (DM-n/EV-n добавляются в ОБА профиля пары — владелец iva-analysis-base + зеркало iva-fr-analyst + _mirrors.yaml) перепроверить, нужна ли правка test-matrix (REPLACEMENTS allowlist / ROLE_LANES).

## Поверхность (файлы, что смотрел)
- Тест parity: `apps/backend/tests/catalog/test_role_replacement_parity.py`
- Тест presets: `apps/backend/tests/catalog/test_iva_role_presets.py`
- Декларация зеркал: `templates/_mirrors.yaml`
- CI-скрипт зеркал: `scripts/check_mirror_sync.py`
- Схема ингредиента: `templates/_schema/ingredient.v1.schema.json`
- Smoke: `apps/backend/tests/catalog/test_role_install_smoke.py`
- Манифест владельца: `templates/iva-analysis-base/manifest.yaml`
- Манифест зеркала: `templates/iva-fr-analyst/manifest.yaml`
- Роль: `templates/iva-role-analyst/manifest.yaml` (own ingredients — только паки; depends_on лейны)

Ключевой факт композиции: `iva-role-analyst` тянет лейны `["tacticum-core-base", "iva-analysis-base"]` (test_iva_role_presets.py:45; и `depends_on` в манифесте роли). Значит владелец iva-analysis-base ВХОДИТ в композицию роли.

---

## 1) test_role_covers_replaced_profile — под А ЗЕЛЁНЫЙ БЕЗ allowlist. ДА.

Логика дословно (test_role_replacement_parity.py):
- `_composed_ids(role_id)` (строки ~157-163): `composed = _ids(role_id)`, затем `for lane in depends_on: composed |= _ids(lane)`. Для iva-role-analyst это = паки роли ∪ _ids(tacticum-core-base) ∪ **_ids(iva-analysis-base)**.
- Тело теста (строки ~205-217): `old = {RENAMES.get(i,i) for i in _ids(replaced)}`; `lost = sorted(old - composed - renamed_allow)`; assert not lost.
- Пара `("iva-role-analyst","iva-fr-analyst"): {}` (строка 63), allowlist пустой.

Под ВАРИАНТ А DM/EV лежат и в зеркале (`_ids(iva-fr-analyst)` ⊇ {DM,EV}) И у владельца (`_ids(iva-analysis-base)` ⊇ {DM,EV}). Владелец в composed → composed ⊇ {DM,EV}. Тогда `old - composed` вычитает DM/EV → `lost` их НЕ содержит → тест ЗЕЛЁНЫЙ с пустым allowlist. **Правка allowlist НЕ требуется.**

Контраст (почему риск флагировался под Б): если DM/EV ТОЛЬКО в зеркале — old⊇{DM,EV}, но composed их НЕ несёт (владелец без них) → lost=[DM,EV] → падение → нужен allowlist. Вариант А этот разрыв закрывает по построению.

ВАЖНО (обратная сторона, парный тест `test_allowlist_entries_are_real_gaps`, строки ~230-246): под А ДОБАВЛЯТЬ DM/EV в allowlist НЕЛЬЗЯ — тест assert-ит `RENAMES.get(id,id) not in composed` с сообщением «уже покрыт ролью — разрыв закрыт, удали строку». Т.е. лишняя строка в allowlist под А сама ПОВАЛИТ тест. Вывод однозначный: allowlist трогать не нужно И нельзя.

## 2) test_single_owner_lanes_are_pairwise_disjoint — НЕ ломается. Подтверждаю.

test_iva_role_presets.py (строки ~199-207): для iva-role-analyst лейны = (tacticum-core-base, iva-analysis-base); проверка `set(_lane_ids(a)) & set(_lane_ids(b)) - allowed` пусто.
- Текущие id tacticum-core-base: getting-started, kb-navigation, tacticum-context, conventional-git, context7, tacticum-mcp.
- DM-n/EV-n — новые уникальные skill_spec id, которых НЕТ в tacticum-core-base. Пересечение лейнов не появляется → disjointness цела. **НЕ ломается** (при условии, что id DM/EV не совпадут с одним из шести id core-base — они новые/уникальные, коллизии нет).
- Парный `test_golden_parity_union_equals_sum_of_lanes` (строки ~210-222): union == сумма размеров лейнов. Добавление уникальных id в один лейн увеличивает и union, и сумму на столько же → инвариант держится. НЕ ломается.

## 3) check_mirror_sync / _mirrors.yaml — добавить DM/EV в ingredients пары + байт-идентичность. Дополнительно:

- Пара уже есть: `_mirrors.yaml` pairs[0] owner=iva-analysis-base, mirror=iva-fr-analyst, ingredients=[api-contracts-discovery, design-system-discovery, fr-authoring, mockup-authoring, start-feature, update-feature]. Чтобы DM/EV зеркалировались — их id надо ДОБАВИТЬ в этот список ingredients.
- check_mirror_sync.py требует (док-строка + код): (1) ингредиент существует в манифестах ОБЕИХ сторон (иначе «декларация устарела»); (2) все body-файлы (body_path + codex_body_path; для скиллов — вся папка `ingredients/skills/{id}/`) идентичны байт-в-байт; (3) зеркало не deprecated.
- Тот же список зеркалируется в pytest: `test_mirror_content_is_byte_identical` (test_role_replacement_parity.py, MIRROR_PAIRS из того же _mirrors.yaml) — параметризован по каждому ingredient_id пары, проверяет совпадение набора файлов и байтов.
- ГЕЙТ ПАПКИ СКИЛЛА: и скрипт, и тест содержат жёсткую эвристику — для body_path со «/skills/» имя папки-родителя ОБЯЗАНО совпадать с ingredient_id (иначе `raise RuntimeError`/assert). Значит новые скиллы должны лежать в `templates/<профиль>/ingredients/skills/<DM-id|EV-id>/SKILL.md`, имя папки = ingredient_id. Иное — громкое падение.
- Ещё: если DM/EV несут `codex_body_path` — он тоже входит в байт-сверку (оба ключа). Owner и mirror должны иметь одинаковый набор ключей/файлов.

## 4) Прочие тесты каталога при добавлении нового skill_spec

- **Схема ингредиента** (`templates/_schema/ingredient.v1.schema.json`): для `kind: skill_spec` требуется `metadata.description_trigger` (string, minLength 1) — строки ~73-80. Новый skill_spec без непустого description_trigger не пройдёт seed-валидацию/`test_manifest_schemas.py`. Образец блока — как у fr-authoring (kind: skill_spec, tier, supports:[claude-code,codex], install_scope, target_path_template `.claude/skills/{ingredient_id}/SKILL.md`, codex_target_path `.agents/skills/{ingredient_id}/SKILL.md`, body_path, metadata.description_trigger).
- **install_smoke** (`test_role_install_smoke.py`): iva-role-analyst в списке ROLES. `_composed` тянет ингредиенты владельца iva-analysis-base → новые DM/EV попадут под:
  - `test_all_body_files_exist_and_non_empty` (строки ~76-83): SKILL.md (и codex-body, если объявлен) должны существовать и быть непустыми. Реальное требование к контенту (создаёт implementer), не правка матрицы.
  - `test_declared_target_paths_are_unique` (строки ~86+): цели `.claude/skills/{id}/SKILL.md` / `.agents/skills/{id}/` уникальны, пока ingredient_id уникален — ок.
- **role composition union** (golden-parity, п.2) — держится.
- **seed_community / governance** (`test_seed_community_governance.py`, `test_seed_community_cli_bodies.py`): работают с skill_spec обобщённо; при валидной схеме и наличии body — падать не должны (в этих тестах хардкод-списков по iva-analysis-base/iva-fr-analyst не найдено).
- **Хардкод-счётчиков/точных списков ингредиентов** для iva-analysis-base или iva-fr-analyst в тестах НЕ найдено (grep по apps/backend/tests): профили упоминают только test_role_replacement_parity.py, test_iva_role_presets.py (через ROLE_LANES/REPLACEMENTS — там id профилей, не списки ингредиентов) и `test_installation_registry.py` (только строка «iva-fr-analyst» в матрице MIGRATION_TARGETS ADR-0059, к ингредиентам отношения не имеет). Отдельного `test_iva_fr_analyst_profile.py`/`test_iva_analysis_base_profile.py` НЕТ.

## Риски / внимание для implementer (не правки матрицы, а контент)
- Папки скиллов обязаны называться ровно по ingredient_id (гейт mirror-эвристики) — иначе громкое падение и скрипта, и pytest.
- Owner и mirror: одинаковый набор body-ключей (body_path и, если есть, codex_body_path) и байт-идентичность.
- description_trigger обязателен и непуст.
- Не добавлять DM/EV в REPLACEMENTS-allowlist — под А это повалит test_allowlist_entries_are_real_gaps.

---

## Вывод для лида
**Правка test-matrix (REPLACEMENTS allowlist / ROLE_LANES) под вариант А НЕ нужна.** Потому что:
1. DM/EV добавляются в владельца iva-analysis-base, который уже входит в композицию iva-role-analyst (depends_on) → `test_role_covers_replaced_profile` остаётся зелёным с пустым allowlist `{}`; более того, любая строка allowlist под А сама повалит парный test_allowlist_entries_are_real_gaps.
2. ROLE_LANES не меняется — состав лейнов роли тот же; disjointness и golden-parity держатся (DM/EV — новые уникальные id, коллизий с tacticum-core-base нет).

Что РЕАЛЬНО нужно (это не test-matrix, а данные/контент): (а) добавить DM/EV id в `ingredients` пары в `templates/_mirrors.yaml` (иначе зеркалирование не проверяется); (б) добавить идентичные skill_spec-блоки и body-файлы в ОБА профиля (папки скиллов = ingredient_id, description_trigger непуст, байт-в-байт owner↔mirror).

Развилка к ГД по ROLE_LANES с lead-modes: по факту разведки — НЕ нужна (правка ROLE_LANES/allowlist под вариант А не требуется).