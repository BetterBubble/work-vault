---
title: explore-mirror-depreciation-mechanism
type: explore
permalink: tacticum/00-board/explore-mirror-depreciation-mechanism
tags:
- explore
- mirrors
- adr-0059
- tz3
- fr-analyst
---

# explore: механика депрекации/разъединения mirror-пары (ТЗ#3)

status: draft
для: lead-fr
репо: /Users/bubblemac/tacticum/tacticum-dev (main)
задача: разъединить пару owner `iva-analysis-base` (frozen, НЕ трогаем) / mirror `iva-fr-analyst`, чтобы `fr-authoring`, `api-contracts-discovery` (и позже `update-feature`) могли расходиться, без правки owner-профиля, с зелёным CI.

---

## 1. Схема `templates/_mirrors.yaml`

Файл: `/Users/bubblemac/tacticum/tacticum-dev/templates/_mirrors.yaml` (93 строки).

Шапка-правила (строки 1-11), дословно ключевое:
- L3-5: «задекларированные здесь ингредиенты обязаны быть БАЙТ-В-БАЙТ одинаковыми у владельца и в зеркале. Правки — только у владельца; зеркало обновляется тем же PR.»
- L6-7: сверка — `scripts/check_mirror_sync.py` (CI: profile-version-discipline.yml) + `tests/catalog/test_role_replacement_parity.py`.
- **L9-11 (механика разъединения, дословно):** «Депрекированные профили зеркал не имеют (frozen). Закрыл пару (профиль депрекирован / ингредиент разошёлся ОСОЗНАННО решением владельца) — удали запись здесь и объясни в CHANGELOG владельца.»

Структура: корневой ключ `pairs:` (L12), список из **6 пар**. Поля пары — ровно три: `owner`, `mirror`, `ingredients` (список ingredient_id). Больше НИКАКИХ полей нет.

Наша пара — первая (L13-21):
```
- owner: iva-analysis-base        # L13
  mirror: iva-fr-analyst          # L14
  ingredients:                    # L15
  - api-contracts-discovery       # L16
  - design-system-discovery       # L17
  - fr-authoring                  # L18
  - mockup-authoring              # L19
  - start-feature                 # L20
  - update-feature                # L21
```
Остальные 5 пар: iva-kmp-development-base/iva-kmp-brownfield (L22-37), iva-web-development-base/iva-web-brownfield (L38-55), iva-mail-development-base/iva-brownfield-mail (L56-65), iva-ios-development-base/iva-ios-brownfield (L66-79), firebird-web-development-base/firebird-web-brownfield (L80-92).

**Флага `deprecated`/`status`/`frozen`/per-ingredient-исключений на уровне пары в YAML НЕТ.** «deprecated» — это концепт уровня ПРОФИЛЯ (поле `deprecated:` в manifest.yaml профиля), не пары. Отдельной JSON-схемы для `_mirrors.yaml` НЕТ (в `templates/_schema/` только ingredient/manifest схемы). Файл — свободный YAML, читают его ровно 2 потребителя (см. п.6).

---

## 2. `scripts/check_mirror_sync.py`

Файл: `/Users/bubblemac/tacticum/tacticum-dev/scripts/check_mirror_sync.py` (120 строк).

Как читает: `yaml.safe_load(MIRRORS...)` (L65), итерирует `decl.get("pairs", [])` (L68). Берёт `pair["owner"]`, `pair["mirror"]` (L69), строит словари ингредиентов из manifest.yaml ОБЕИХ сторон (L71-73).

Логика по порядку:
- **L77-82:** `if mm_manifest.get("deprecated"):` → failure «зеркальный профиль депрекирован — пара устарела, удали её из _mirrors.yaml», `continue` (пара пропускается целиком). Т.е. если пометить mirror-манифест `deprecated: true`, но пару не удалить — CI КРАСНЫЙ.
- **L83:** `for ingredient_id in pair["ingredients"]:` — обходит РОВНО список из YAML. Ингредиент не в списке → вообще не проверяется.
- **L85-90:** если ingredient_id отсутствует в манифесте owner ИЛИ mirror → failure «декларация устарела».
- **L91-96:** собирает body-файлы (`body_path` + `codex_body_path`; для скиллов — вся папка ingredient_id через os.walk, L54-58).
- **L97-101:** если набор имён файлов различается → failure.
- **L103-108:** побайтовое сравнение каждого файла; `fo[name] != fm[name]` → failure «зеркало разъехалось».

Exit: 1 при любом failure (L109-113), иначе 0 (L114).

**Вывод по п.2:** per-ingredient «skip» реализован ЕДИНСТВЕННЫМ способом — не включать ингредиент в `pair["ingredients"]`. Отдельной обработки deprecated-пары нет; есть только реакция на `deprecated`-флаг mirror-ПРОФИЛЯ (требует удалить пару). Падает при первом байтовом расхождении задекларированного ингредиента.

---

## 3. `apps/backend/tests/catalog/test_role_replacement_parity.py`

Файл: `/Users/bubblemac/tacticum/tacticum-dev/apps/backend/tests/catalog/test_role_replacement_parity.py` (212 строк). Два независимых блока:

**Блок A — зеркала (дублирует check_mirror_sync в pytest):**
- L102-107: `_MIRRORS_DECL = yaml.safe_load(_mirrors.yaml)`; `MIRROR_PAIRS = [(owner, mirror, list(ingredients)) ...]`.
- L193-211: `test_mirror_content_is_byte_identical` — параметризован ПО КАЖДОМУ ингредиенту из `MIRROR_PAIRS` (L194-196). Байтовое сравнение body-файлов owner vs mirror (L207-211).
- Поведение: **убрать ингредиент из `pair["ingredients"]`** → этот параметр исчезает, байт-чек по нему не выполняется. Никаких других эффектов. Обработки `deprecated` в этом блоке НЕТ (в отличие от скрипта) — читает только owner/mirror/ingredients.

**Блок B — replacement-parity (НЕ читает `_mirrors.yaml`!):**
- L62-97: `REPLACEMENTS` — хардкод-словарь пар «роль ← замещаемый профиль». Наша строка: **`("iva-role-analyst", "iva-fr-analyst"): {}`** (L63) — пустой allowlist.
- L118-125: `_composed_ids(role)` = собственные ingredient_id роли ∪ union ingredient_id всех лейнов из `depends_on`.
- L157-169: `test_role_covers_replaced_profile` — `lost = _ids(iva-fr-analyst) − composed − allowlist`; `assert not lost`. Т.е. КАЖДЫЙ ingredient_id профиля `iva-fr-analyst` обязан присутствовать в композиции роли `iva-role-analyst`, иначе тест КРАСНЫЙ.
- L177-190: `test_allowlist_entries_are_real_gaps` — allowlist не должен копить закрытые/несуществующие разрывы.

`iva-role-analyst` `depends_on: [tacticum-core-base, iva-analysis-base]` (manifest L20-22), своих skill-ингредиентов не несёт (только пак-kinds). Значит композиция роли включает все ингредиенты owner-лейна `iva-analysis-base`.

**Вывод по п.3:** пометка пары deprecated этим тестом не поддержана (нет такого поля). Удаление ингредиента из списка пары гасит только байт-чек блока A. Блок B про `_mirrors.yaml` НЕ знает и от него не зависит — он смотрит manifest `iva-fr-analyst` напрямую.

---

## 4. Варианты разъединения (факты)

**(а) Убрать конкретные ingredient_id из `pair["ingredients"]` первой пары** (fr-authoring, api-contracts-discovery, update-feature):
- check_mirror_sync (L83) и test_mirror_content_is_byte_identical (L194-196) перестают байт-проверять именно эти 3 → их контент в mirror может расходиться с owner свободно.
- Остальные 3 (design-system-discovery, mockup-authoring, start-feature) остаются в списке → продолжают байт-сверяться с owner.
- Owner-профиль `iva-analysis-base` НЕ трогается. Manifest `iva-fr-analyst` не обязателен к правке для самого разъединения (правки контента — отдельно по ТЗ#3).
- Блок B parity (test_role_covers_replaced_profile) — НЕ затрагивается: он не читает `_mirrors.yaml`.
- **Минимально-инвазивно, точечно, соответствует шапке L9-11 («ингредиент разошёлся осознанно — удали запись здесь»).**

**(б) Пометить пару deprecated / удалить пару целиком:**
- «deprecated на паре» — механики НЕТ (нет поля). Единственный deprecated-триггер — `deprecated: true` в manifest mirror-ПРОФИЛЯ; тогда check_mirror_sync L77-82 ТРЕБУЕТ удалить пару (красный, пока не удалишь) — но депрекировать сам `iva-fr-analyst` нельзя, у него живые пользователи (ADR-0059 §8: гейт — только после зелёного UAT) и в него же лягут правки ТЗ#3.
- Удалить всю пару (L13-21) → снимает байт-чек со ВСЕХ 6 ингредиентов, включая 3, которые надо держать синхронными → избыточно, теряем защиту от дрейфа design-system-discovery/mockup-authoring/start-feature.

**Чистейший — вариант (а).**

---

## 5. Прецедент

`git log -- templates/_mirrors.yaml` — всего 2 коммита:
- `fe17a7f` feat(ci): механика зеркал US #714 — _mirrors.yaml + байтовая автосверка (создание).
- `35e0cb0` fix(templates): пост-мерж синк после main #122–#131 — зеркала отработали штатно (1 insertion).

**Прецедента удаления ингредиента из пары / депрекации/разъединения пары в истории НЕТ.** `git log -p` не содержит удалений строк ingredients. Grep `deprecated` по templates находит только поле `deprecated: false` в manifest.yaml профилей (это профильный флаг, не про mirrors). Т.е. разъединение будет ПЕРВЫМ применением механики из шапки L9-11 — раньше её не исполняли, только декларировали.

---

## 6. Побочные проверки

**Кто читает `_mirrors.yaml` (точный список — только 2 потребителя + CI):**
- `scripts/check_mirror_sync.py` (L65).
- `apps/backend/tests/catalog/test_role_replacement_parity.py` (L102-104).
- `.github/workflows/profile-version-discipline.yml` — step «Mirror sync check (US #714 / ADR-0059 Решение 7)» запускает `python scripts/check_mirror_sync.py` (на PR и push в main). Триггерится в т.ч. на изменение `scripts/check_mirror_sync.py` (paths L16, L23).

Более широкий grep `mirror`/`parity` по apps/backend/tests даёт много файлов, но по строке `_mirrors.yaml`/`check_mirror_sync` — только эти. Другие тесты (`test_composition.py`, `test_iva_role_presets.py`, `test_manifest_schemas.py` и пр.) `_mirrors.yaml` НЕ читают; убирание ингредиента из пары их не касается.

**КРИТИЧНО — если в `iva-fr-analyst` ДОБАВИТЬ новые skill_spec (DM/EV), которых нет в owner и нет в паре:**
- `test_role_covers_replaced_profile` (L157-169, пара `iva-role-analyst ← iva-fr-analyst`, allowlist `{}`) вычислит `lost = _ids(iva-fr-analyst) − composed(iva-role-analyst) − {}`. Новые DM/EV попадут в `_ids(iva-fr-analyst)`, но их НЕТ в композиции роли (роль тянет только tacticum-core-base + iva-analysis-base, а owner заморожен и их не несёт) → `lost` непуст → **тест КРАСНЫЙ**.
- Значит добавление новых ингредиентов в mirror ломает replacement-parity. Чинится одним из двух (это уже зона решения лида/ответственной роли, не разведки): либо новый ингредиент кладётся в лейн композиции роли, либо добавляется строка в allowlist `REPLACEMENTS[("iva-role-analyst","iva-fr-analyst")]` с причиной. Байт-mirror-чек (check_mirror_sync/блок A) на новый ингредиент НЕ сработает, пока он не внесён в `pair["ingredients"]` — так что новые DM/EV в паре можно не декларировать.
- `test_allowlist_entries_are_real_gaps` (L177-190) при этом требует, чтобы каждая allowlist-строка реально отсутствовала в композиции роли (иначе тоже красный) — учесть при выборе способа.

Другие completeness-тесты, падающие от одного лишь удаления ингредиента из `_mirrors.yaml`, НЕ найдены.

---

## 7. ADR-0059

Файл: `/Users/bubblemac/tacticum/tacticum-dev/docs/adr/0059-single-axis-process-lanes-and-role-packs.md`.

Заголовок: «Одна ось разбиения профилей: процессы-лейны... **правило зеркал переходного периода**».

**Раздел «### 7. Переходный период — правило зеркал» (L63-65), дословно суть (L65):**
«Пока старые профили живы: у каждого ингредиента один владелец — лейн; копия в старом профиле — зеркало; правки только у владельца, зеркало обновляется тем же PR. Механика автосверки (US #714) реализована: декларация пар — `templates/_mirrors.yaml` (**6 пар, 61 ингредиент**), байтовая проверка `scripts/check_mirror_sync.py` в CI (profile-version-discipline.yml, PR и push) и в тестах каталога + replacement-parity "роль ⊇ замещаемый профиль" с честным allowlist... + install-smoke.»

Связанный раздел «### 8. Судьба старых профилей» (L67-69): гейт для `iva-fr-analyst` — депрекация **только после зелёного UAT «отзыв письма» на iva-role-analyst** (у профиля живые пользователи-аналитики). Т.е. депрекировать mirror сейчас нельзя — подтверждает, что вариант (б) через deprecated отпадает.

Последствие L77: «⚠️ До US #714 зеркала синхронизируются вручную тем же PR — окно держать коротким.»

**Точка для документирования правки протокола:** раздел 7, строка с «(6 пар, 61 ингредиент)» — при удалении 3 ингредиентов из первой пары счётчик станет 58; и сам факт «осознанного расхождения ингредиента» стоит зафиксировать здесь как первый прецедент применения правила из шапки `_mirrors.yaml` L9-11.

---

## Рекомендация по механике для лида

**Чистейший — вариант 4(а): точечно удалить строки ингредиентов из первой пары `_mirrors.yaml`.** Не трогает owner `iva-analysis-base`, оставляет под зеркалом 3 неизменяемых ингредиента, снимает байт-контроль ровно с расходящихся.

Набросок диффа (`templates/_mirrors.yaml`, пара L13-21) — под US#3 сейчас fr-authoring + api-contracts-discovery; update-feature — позже под US#5:
```
- owner: iva-analysis-base
  mirror: iva-fr-analyst
  ingredients:
- - api-contracts-discovery      # удалить (US#3, расходится)
  - design-system-discovery
- - fr-authoring                 # удалить (US#3, расходится)
  - mockup-authoring
  - start-feature
- - update-feature               # удалить позже под US#5
```
(остаётся: design-system-discovery, mockup-authoring, start-feature.)

Сопутствующее:
1. Обновить комментарий/CHANGELOG владельца по требованию шапки `_mirrors.yaml` L9-11 («объясни в CHANGELOG владельца») — но owner заморожен; вероятно CHANGELOG `iva-fr-analyst` (уточнить у ответственной роли, куда писать при frozen owner).
2. В ADR-0059 раздел 7 — поправить «6 пар, 61 ингредиент» → актуальный счётчик и добавить строку о первом осознанном расхождении ингредиентов пары iva-analysis-base↔iva-fr-analyst.
3. Проверять НЕ надо `deprecated` — mirror-профиль остаётся `deprecated: false` (ADR-0059 §8 гейт по UAT).
4. **Отдельно предупредить:** новые skill_spec DM/EV в `iva-fr-analyst` уронят `test_role_covers_replaced_profile` (allowlist `{}`), пока не будут либо покрыты композицией роли, либо внесены в allowlist REPLACEMENTS с причиной — это НЕ решается через `_mirrors.yaml`.

Ключевые файлы:
- /Users/bubblemac/tacticum/tacticum-dev/templates/_mirrors.yaml
- /Users/bubblemac/tacticum/tacticum-dev/scripts/check_mirror_sync.py
- /Users/bubblemac/tacticum/tacticum-dev/apps/backend/tests/catalog/test_role_replacement_parity.py
- /Users/bubblemac/tacticum/tacticum-dev/.github/workflows/profile-version-discipline.yml
- /Users/bubblemac/tacticum/tacticum-dev/docs/adr/0059-single-axis-process-lanes-and-role-packs.md
- /Users/bubblemac/tacticum/tacticum-dev/templates/iva-role-analyst/manifest.yaml (depends_on L20-22)
