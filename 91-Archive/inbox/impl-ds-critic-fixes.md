---
title: impl-ds-critic-fixes
type: report
permalink: tacticum/00-board/impl-ds-critic-fixes-1
tags:
- ds
- web-to-kmp
- critic
- bundle1
- implementer
archived-at: 2026-08-03 11:16
---

# impl: web-to-kmp-screen-port — обязательные правки критика (бандл #1)

status: draft
Отчёт implementer для lead-ds (ТЗ#1 Сц.4). Внесены обязательные правки critic-ревью в навык
`web-to-kmp-screen-port` перед PR бандла #1. НЕ пушено/не мержено (autonomy off).

## Объект
- worktree: `/Users/bubblemac/tacticum/tacticum-dev-ds-web-to-kmp`, ветка `feat/ds-web-to-kmp`
- новый коммит: `39ae642` (поверх `65e4fbf`)
- файл навыка: `templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-screen-port/SKILL.md`
- версия: `iva-kmp-development-base` 0.6.0 → 0.7.0 (+ CHANGELOG `[0.7.0]`)

## git diff --stat 65e4fbf..HEAD
```
 templates/iva-kmp-development-base/CHANGELOG.md    |  24 +++++
 .../skills/web-to-kmp-screen-port/SKILL.md         | 115 +++++++++++++--------
 templates/iva-kmp-development-base/manifest.yaml   |   2 +-
 3 files changed, 97 insertions(+), 44 deletions(-)
```

## Что сделано по пунктам

### ОБЯЗАТЕЛЬНЫЕ

**1. §7 «static gate + three-way parity» — устранён разнобой «три vs четыре».**
Заголовок → `## 7. Verification — static gate + three-way parity (acceptance criteria)`.
Структура переразбита: ОДИН **static gate — component level** (собран из `Iva*`, ноль сырых
`Color`/`dp`, hoisting, keyed-списки; с под-буллетами LazyColumn-vs-Column и «читать виджеты»)
+ **three-way parity** ровно из 3 нумерованных легов (web-sample · tokens · Compose-render).
Строка «three distinct legs» теперь ссылается ровно на 3 пункта — разнобой снят. Итоговая
строка «full three-way parity» охватывает эти же 3 лега, гейтится статикой и behavioural-легом.
Кросс-ссылки §7.1–§7.4 в §8-таблице и в TODO переведены на устойчивые именные якоря
(«§7 static gate / token leg / web-sample leg / Compose-render leg») — иначе после переразбивки
они указывали бы не туда.

**2. Behavioural/logic parity — отдельным легом.**
Критерий Сц.4 №2 (view-state enum + пайплайн список/выбор/деталь + action set залоченного
образца) вынесен из тела web-sample-лега в ОТДЕЛЬНЫЙ явный болд-блок «Behavioural / logic
parity — the main acceptance of the port»: ручная code-сверка source↔target (iva-one ↔
ported Decompose component/state holder), явно «NOT an instrumented `ui-mockup-match`» (тот лочит
веб-сторону, лег 1). Читается самостоятельно как главный acceptance переноса. Web-sample-лег
теперь чисто про «lock the sample».

**3. TODO-якорь словаря синхронизирован с resolved-состоянием.**
- (а) §TODO: буллет переименован `[Iva*↔web-component dictionary — RESOLVED on board, awaiting
  repo-native delivery]`; помечено RESOLVED на `00-Board/phase2-provisional-iva-web-dictionary`
  (**32 `figma_key` + 17 обоснованный `null`**, из code-bindings каталога `tokens.json`), ожидает
  repo-native доставки в `AI common/skills/`. Отмечено: таблица **reverse-keyed** (ключён
  `Iva*`→web, порт делает обратный lookup веб-тег→`Iva*`).
- (б) §1.6 и §5: добавлены ссылки на словарь как конкретный web→`Iva*` crosswalk, с той же
  пометкой reverse-keyed.

### ДЕШЁВЫЕ NICE-TO-HAVE

**4. §3 разрыв Level-1→Level-3.** В вводную строку про уровень state-holder добавлено «(levels
follow the `mvi-state-machine-su-ivcs-messenger-kmp` taxonomy; **Level-2 is not used in ports** —
the jump from Level-1 to Level-3 below is intentional)».

**5. §9 timeless.** Из тела нормы убраны атрибуция автора (`r.yarullin`) и жёсткие числа
(`~49`/`~41`). Различие live-shared-модуль vs stale-snapshot сохранено как норма; про числа —
мягкая сноска `[^inv]` («treat any composable count as indicative, not a fixed figure»).

**7. §0 scope flag/estimate.** В буллет «Structural edit» добавлено: «record the scope estimate
in a scope note / on the board **before committing** (autonomy off)».

НЕ делались: #6 (проверка source-reference ссылок) и #8 (ужатие description) — по указанию лида.
Второй навык `web-to-kmp-source-reference` не тронут.

## Валидаторы (venv репо, apps/backend/.venv)
- `scripts/check_profile_version_discipline.py` (static) → `OK — 46 profile(s) clean`
- `scripts/check_profile_version_discipline.py --diff-against HEAD` → `OK — 46 profile(s) clean`
- `scripts/check_mirror_sync.py` → `OK — 62 зеркальных ингредиента в 6 парах синхронны`
  (навык `web-to-kmp-screen-port` НЕ в списке зеркал iva-kmp-development-base — mirror не затронут)
- `pytest tests/catalog/test_manifest_schemas.py` → passed (38 профилей)

## Самопроверка
- §7 больше не «три vs четыре»: «three distinct legs» указывает ровно на 3 лега three-way; static
  gate и behavioural — отдельные блоки. ✓
- Behavioural parity читается самостоятельно как главный acceptance (свой болд-блок, «not
  `ui-mockup-match`»). ✓
- TODO-словаря синхронизирован (RESOLVED, 32+17, reverse-keyed, ссылки в §1.6/§5). ✓
- Секретов/мусора/AI-подписей в коммите нет; ветка `feat/ds-web-to-kmp`, не main; не пушено.

## Relations
- part_of [[gate-bundle1-git-final]]
- refines [[impl-ds-web-to-kmp-skill-skeleton]]
- resolves_from [[phase2-provisional-iva-web-dictionary]]