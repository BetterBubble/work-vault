---
status: draft
topic: US#4 Проход B2 — фикс консистентности (post-critic)
task: ТЗ#3 · US#4 B2
role: implementer
lead: lead-fr
worktree: /Users/bubblemac/tacticum-wt/us4-passB2
branch: feat/us4-passB2
date: 2026-07-24
permalink: tacticum/00-board/impl-us4-pass-b2-fix-1
archived-at: 2026-08-03 11:16
---

# impl · US#4 B2 fix — выравнивание статус-вокаба по эталону rn

Добавлен ОДИН фикс-коммит поверх 4 cherry-pick. Версии НЕ поднимал (тот же
bump'ы из cherry-pick: mail 0.7.3 / rn 0.5.3 / ios 0.1.3 / firebird 0.1.3),
это консистентность-твик. Push/PR/merge не делал (autonomy off).

## Фикс-коммит
`80deed5 fix(us4-B2): выровнять статус-вокаб pin/tests по эталону rn (К-2/К-4 консистентность)`

## Что поправил (3 профиля; rn — эталон, НЕ тронут)

### 1. firebird pin — русские статус-токены ТЗ §4 К-2
`templates/firebird-web-brownfield/ingredients/skills/pin-authoring/SKILL.md`
- Статус-токены Project sections status `implemented`/`discrepancy` → русские ТЗ
  `реализован`/`расхождение` (`blocked` оставлен как в mail/rn/ios и ТЗ §4 К-2).
- **Уточнение по объёму:** токен-значение встречается в 5 местах SKILL (не только
  в определении ~стр.155-156, но и в top-summary, К-3, К-4, anti-patterns). Выровнял
  ВСЕ 5 сайтов токен-значений — иначе профиль остался бы внутренне несогласованным
  (половина токенов рус., половина англ.), что противоречит цели консистентности.
  Английская **проза** (`record a discrepancy`, `critical discrepancy`, заголовок
  «discrepancy table») и KB-поле `coverage_status=implemented` (другой домен, NFR)
  НЕ тронуты — только backtick-обёрнутые статус-значения. Семантика без изменений.

### 2. mail/ios/firebird tests — явное правило `расхождение` (трассировка К-4 → tests)
В coverage-таблицу проектных серий каждого профиля добавлена обработка члена,
помеченного PIN как `расхождение` (К-4): контрактный тест фиксирует ОЖИДАЕМЫЙ
контракт как failing/xfail (формулировка взята из эталона rn). Существующие
статус-токены каждого профиля сохранены — только ДОБАВлено `расхождение`:
- **mail** (`iva-brownfield-mail/…/tests-authoring/SKILL.md`): в vocab «Project
  series coverage» добавлен `расхождение` рядом с `covered`/`partial`/`blocked`
  (+ inline-списки в разделе «Input v2 FR» и в Rules).
- **ios** (`iva-ios-brownfield/…/tests-authoring/SKILL.md`): `расхождение` выделено
  из `blocked` в ОТДЕЛЬНЫЙ статус vocab «Project-series test status» — раньше
  складывалось в `blocked` (без теста), теперь член `расхождение` покрывается
  контрактным тестом на извлечённой чистой логике (xfail), `blocked` остаётся
  «теста нет». Согласовано с политикой iOS «не мокать fragile-границу» (тест на
  pure mapper). Правки в vocab, Rules и anti-pattern.
- **firebird** (`firebird-web-brownfield/…/tests-authoring/SKILL.md`): в vocab
  «Project-section coverage» добавлен `расхождение` (pure-decoder тест на fixture
  фиксирует ожидаемую схему/`Either` как failing/xfail), рядом с сохранёнными
  `covered`/`blocked`/`gap`.

### 3. mail tests — перенумерация Document structure
`iva-brownfield-mail/…/tests-authoring/SKILL.md`, «Document structure»:
- Убран дубль `5.` (Manual tests + Test environment setup).
- Contract/model/event-тесты подняты до Manual tests (item 5, как в rn).
- Итог: 5 = Contract/model/event tests → 6 = Manual tests → 7 = Test environment.
- Контент не удалялся — только перенумерация.

### CHANGELOG
Дописана строка-консистентность в СУЩЕСТВУЮЩИЕ записи версий (без повторного bump):
mail 0.7.3, ios 0.1.3, firebird 0.1.3. В firebird дополнительно синхронизированы
токены в тексте записи (`implemented`/`discrepancy` → рус.) + `расхождение` в vocab.
**rn (0.5.3) не тронут** — эталон.

## Подтверждение: rn НЕ тронут
`git diff --name-only` по фикс-коммиту — ни одного файла `iva-rn-brownfield`.
7 изменённых файлов, все в mail / ios / firebird.

## Проверки (свежий прогон)
Env: проектный `apps/backend` через `uv run --extra dev` (conftest тянет
alembic/sqlalchemy/pytest-asyncio; чистый `uv run --with pyyaml/pytest/jsonschema`
падал на импорте conftest — использовал полный dev-extra).

- **version-discipline** `--diff-against origin/main` → `OK — 48 profile(s) clean`.
- **pytest целевые** (schemas + role_presets + install_smoke + 3 профиля):
  `247 passed in 3.57s`, 0 failed.
  - test_manifest_schemas.py, test_iva_role_presets.py, test_role_install_smoke.py,
    test_iva_brownfield_mail_profile.py, test_iva_ios_brownfield_profile.py,
    test_firebird_web_brownfield_profile.py.
- Унаследованный red iva-role-web #149 ds — НЕ в нашем скоупе (эти тесты не гоняли),
  исключён по указанию.
- worktree clean.

## git log origin/main..HEAD (5 коммитов)
```
80deed5 fix(us4-B2): выровнять статус-вокаб pin/tests по эталону rn (К-2/К-4 консистентность)
01cb9ac feat(us4-B2): firebird pin/tests implement & cover CT-n/DM-n/EV-n project series
68e50d5 feat(iva-ios-brownfield): US#4 Проход B2 — pin/tests реализуют серии CT/DM/EV (К-2/К-4)
647be69 feat(us4-B2): pin/tests-authoring реализуют проектные серии FR v2 (К-2/К-4) для iva-rn-brownfield
f4848a3 feat(iva-brownfield-mail): pin/tests реализуют+покрывают проектные серии v2-FR (К-2/К-4)
```

## git diff --stat (фикс-коммит)
```
 templates/firebird-web-brownfield/CHANGELOG.md                          |  9 +++++++--
 templates/firebird-web-brownfield/ingredients/skills/pin-authoring/…    | 10 +++++-----
 templates/firebird-web-brownfield/ingredients/skills/tests-authoring/…  | 11 +++++++----
 templates/iva-brownfield-mail/CHANGELOG.md                             |  5 +++++
 templates/iva-brownfield-mail/ingredients/skills/tests-authoring/…      | 20 ++++++++++++--------
 templates/iva-ios-brownfield/CHANGELOG.md                              |  5 +++++
 templates/iva-ios-brownfield/ingredients/skills/tests-authoring/…       | 19 ++++++++++++-------
 7 files changed, 53 insertions(+), 26 deletions(-)
```

## На заметку тимлиду (1 момент за пределами буквального плана)
Фикс #1 был описан как «~стр.155-156», но токен-значение жило в 5 местах firebird
pin. Выровнял все 5 (иначе половинчатый фикс = внутренняя несогласованность).
Прозу и KB-поле не трогал. Если требуется откатить до узкого скоупа — скажи.