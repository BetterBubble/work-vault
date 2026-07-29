---
title: 'Реализация: формат наряда встроен в тело lite-task-workflow (прод-инцидент
  с залипанием в плановом режиме)'
type: note
status: current
created: 2026-07-28
updated: 2026-07-28
tags:
- board
- lite-base
- fix
permalink: tacticum/00-board/impl-lite-inline-order-2026-07-28-1
---

# Реализация: lite-base — наряд встроен в тело навыка

## Координаты

| | |
|---|---|
| Репозиторий | `/Users/bubblemac/tacticum/tacticum-dev` |
| Worktree | `/Users/bubblemac/tacticum-worktrees/lite-inline` |
| Ветка | `fix/lite-base-inline-work-order` (от `main` = `9be499d`) |
| Коммит | `769a852` — `fix(lite-base): встроить формат наряда в тело навыка, снять недоставляемый assets` |
| Push | НЕ делался (распоряжение: наружу сегодня ничего не отправляем) |

Путь worktree — под `$AGENT_WORKTREE_ROOT` (`/Users/bubblemac/tacticum-worktrees/`), а не
`../tacticum-dev-lite-inline`, как было в задании: канон роли требует держать worktree вне
кодового корня. Имя ветки — ровно из задания.

## Что сделано

1. **`templates/tacticum-lite-base/ingredients/skills/lite-task-workflow/SKILL.md`**
   - Первое предложение шага 2 заменено: ордер собирается «по формату ниже», всё нужное — в
     теле, внешнего файла для чтения нет; отсутствие вспомогательного файла явно объявлено
     НЕ поводом останавливаться и НЕ поводом советовать пользователю новую сессию.
     Остаток абзаца и таблица toolset/gate сохранены без изменений.
   - После таблицы toolset/gate и до абзаца «Gate words: …» вставлена подсекция
     `### Order format`: fenced-шаблон наряда (Type / Escalation / Diagnosis-Goal / Plan /
     Test-plan / Risks-zones / Platforms / Forks / «Ok?») + построчные правила заполнения по
     каждому полю.
   - Ссылок на `references/` в теле не осталось (проверено грепом).
2. **`manifest.yaml`** — снят блок `assets:` у ингредиента `lite-task-workflow`; версия
   `0.1.3` → `0.1.4`; над `ingredients:` добавлен комментарий, что вспомогательных файлов у
   скилла нет и объявлять их нельзя, пока в платформе нет механики доставки (со ссылкой на
   `skill_spec.py` / `SkillMetadata`).
3. **Удалён** `ingredients/skills/lite-task-workflow/references/work-order-template.md`
   (`git rm`) вместе с опустевшим каталогом `references/` — 116 строк.
4. **CHANGELOG.md** — секция `## [0.1.4] — 2026-07-28`, раздел `### Fixed`, в стиле
   существующих секций (русский). **README.md** — строка 14 больше не упоминает reference.
   Строка 54 (историческая ссылка на драфты в `docs/proposals/`) не тронута.

## Сверх списка из задания: goldens iva-role-kmp

Изменение тела навыка сдвигает sha256 в e2e-golden'ах — это PR-гейт, и оставить его красным
было бы дефектом. Перегенерированы **только** goldens `iva-role-kmp`
(`E2E_INSTALL_REGEN_GOLDEN=1 pytest tests/e2e_install -k install_flow_role_kmp`) — это
единственная роль с golden-снимком лейна. Изменилась ровно одна строка в каждом файле:

```
- "repo/.claude/skills/lite-task-workflow/SKILL.md": "a6b10f63…"
+ "repo/.claude/skills/lite-task-workflow/SKILL.md": "70dc21d8…"
```

**Побочное подтверждение диагноза:** до регенерации падение показывало
`added: []`, `removed: []`, `changed: ['…/lite-task-workflow/SKILL.md']` — удаление
`work-order-template.md` не убрало ни одного файла из устанавливаемого дерева, то есть он
пользователю действительно никогда не доставлялся.

## Проверки

### 1. Схемная валидация манифеста

`templates/tacticum-lite-base/manifest.yaml` прогнан против `manifest.v2.schema.json` +
`ingredient.v1.schema.json` (Draft7Validator, тот же путь, что у CI-тестов):

```
profile: tacticum-lite-base version: 0.1.4
SCHEMA ERRORS: 0
```

Синтетический тест `test_skill_spec_with_assets_and_scripts_renders` — **зелёный**
(`1 passed`): он собирает манифест в коде и от правки реального лейна не зависит.

CI-гейт версионной дисциплины: `scripts/check_profile_version_discipline.py --diff-against main`
→ `OK — 48 profile(s) clean.`

### 2. Тесты каталога — `pytest tests/catalog`

| | Результат |
|---|---|
| Было (`main`, отдельный worktree) | **710 passed**, 51 warnings, 55.24s |
| Стало (ветка) | **710 passed**, 51 warnings, 28.50s |

Новых красных нет, состав не изменился.

### 3. e2e_install

`pytest tests/e2e_install -k kmp` на ветке после регенерации: **7 passed**, 85 deselected.

Отдельно зафиксировано: на `main` полный `tests/e2e_install` уже красный — **35 failed,
57 passed** ДО нашей правки. Причины предсуществующие и к лейну отношения не имеют
(дрейф goldens у `iva-web-brownfield`, `tacticum-dev-base`, ролей go/java/analyst/web/mail/
ios/internal/platform/qa — расходятся `brd-authoring`, `start-task`, `pin-authoring` и др.;
плюс `profile_not_found` у `iva-go-backend-brownfield`). `role_kmp` в этом списке НЕ было —
он был зелёным, наша правка его временно уронила, регенерация вернула. **Это отдельный
тех-долг репозитория, не наш.**

### 4. Грепы

`grep -rn "work-order-template" .` — в живых телах и манифестах не осталось. Остались только
исторические места:

- `docs/proposals/workflow-modes/*.draft.md` (исторические драфты);
- `templates/tacticum-lite-base/CHANGELOG.md` — строка 92 (историческая секция 0.1.0) и
  новые строки 11/28, где дефект описывается;
- `templates/tacticum-lite-base/README.md:54` — историческая ссылка на драфты (по заданию не
  трогали);
- `apps/backend/tests/catalog/test_manifest_schemas.py:375,381` — синтетический тест
  механики `assets`, зелёный.

`grep -rn "references/" templates/tacticum-lite-base/` — только в тексте CHANGELOG.

### 5. `git diff --stat main..HEAD`

```
 .../golden/iva-role-kmp/claude-code.json           |   2 +-
 .../e2e_install/golden/iva-role-kmp/codex.json     |   2 +-
 templates/tacticum-lite-base/CHANGELOG.md          |  30 ++++++
 templates/tacticum-lite-base/README.md             |   2 +-
 .../ingredients/skills/lite-task-workflow/SKILL.md |  63 ++++++++++-
 .../references/work-order-template.md              | 116 ---------------------
 templates/tacticum-lite-base/manifest.yaml         |  13 ++-
 7 files changed, 102 insertions(+), 126 deletions(-)
```

Полный дифф — в ветке: `git diff main..fix/lite-base-inline-work-order`.

## Что осталось тимлиду

- Доставка ветки наружу и текст PR — не мой шаг (и пуш сегодня запрещён распоряжением).
- Предсуществующий красный `tests/e2e_install` на `main` (35 failed) — отдельная задача.
- Механики доставки `metadata.assets` в платформе по-прежнему нет; поле остаётся
  объявительным. Если оно нужно как механика — это отдельная работа по платформе.
---

## Доработка по критику (28.07)

Критик показал, что правка в прежнем виде **не доезжает до пользователя**: рёбра
`depends_on` замораживаются на момент сида (`seed_profile.py:349-356`), а композиция
читает именно замороженные рёбра (`composition.py:43-58`). У `iva-role-go` 0.2.1 ребро
жёстко указывает на `tacticum-lite-base` **0.1.3** — то есть после сида лейна 0.1.4 его не
получила бы ни одна роль. Плюс три замечания по телу навыка.

Ветка та же — `fix/lite-base-inline-work-order` в
`/Users/bubblemac/tacticum-worktrees/lite-inline`. Два новых коммита поверх прежних двух.

### 1. Пере-freeze шести ролей (коммит `968460e`)

`chore(roles): patch-ре-релиз шести ролей ради пере-freeze ребра на lite-base 0.1.4`

Подняты patch-версии всех шести ролей, тянущих лейн (список сверен —
`grep -rl "tacticum-lite-base" templates/*/manifest.yaml` даёт ровно эти шесть + сам лейн):

| Роль | было | стало |
|---|---|---|
| `iva-role-go` | 0.2.1 | **0.2.2** |
| `iva-role-kmp` | 0.2.1 | **0.2.2** |
| `iva-role-java` | 0.1.1 | **0.1.2** |
| `iva-role-web` | 0.1.1 | **0.1.2** |
| `iva-role-mail` | 0.1.1 | **0.1.2** |
| `iva-role-ios` | 0.1.1 | **0.1.2** |

В каждой роли — новая секция CHANGELOG `## [<версия>] — 2026-07-28`, раздел `### Fixed`:
patch-ре-релиз ради пере-freeze ребра, собственный контент профиля и состав `depends_on`
не менялись, причина — фикс лейна 0.1.4. Прецедент назван явно: `iva-role-go` `[0.1.1] —
2026-07-23` (пере-freeze на `tacticum-bugfix-base` 0.1.1, US #724) — тот же приём, тоже без
изменения контента профиля. У `iva-role-go` ссылка внутренняя («секция ниже»), у остальных
— на роль go.

`depends_on` не тронут ни в одном манифесте — поднята только строка `version`.

### 2. Правки тела навыка (коммит `21d44cc`)

`fix(lite-base): сузить правило про отсутствующий файл, вернуть потерянные фрагменты`

**(а) Правило сужено.** Было «A missing **auxiliary** file is never a reason to stall» —
под это подпадал и отсутствующий файл РЕПОЗИТОРИЯ, а это законный повод остановиться.
Стало: «A missing file **that this skill itself would have shipped** is never a reason to
stall…» + явная оговорка в скобках: «(A missing file of the *repo* you are working on is a
different matter — that is a legitimate reason to stop and ask.)» Продолжение («do not
stop, and do not tell the user to open a new session») сохранено.

**(б) Возвращены два потерянных фрагмента.**
- В правило поля **Risks / zones** — маркер уровня стека: `*(Concrete guard names, ADR
  numbers, the fragile-zone boundary come from the stack/repo layer.)*` Это был
  единственный разрыв: во всём остальном теле маркер выдержан последовательно.
- В правило поля **Platforms** — связка с шагом 4: «If the forecast is "only source-set X"
  — say so: it is a promise the finalize step will check».

**(в) Дубль убран.** Буллет **Forks** пересказывал блок «Forks are two distinct cases, do
not mix them», который идёт десятью строками ниже в том же шаге. Сокращён до одной строки
со ссылкой на этот блок; различение resolved / open в самом шаблоне наряда осталось.

Плюс в CHANGELOG лейна 0.1.4 переписан подпункт про правило поведения — прежняя
формулировка «отсутствие вспомогательного файла» повторяла ровно ту широту, которую
критик и забраковал.

### 3. Проверки

**`pytest tests/catalog`** — **710 passed, 0 failed**. База на `main` (worktree
`/Users/bubblemac/tacticum/tacticum-dev`) — **710 passed, 0 failed**. Новых красных нет,
число тестов не изменилось.

**`pytest tests/e2e_install -k kmp`** — **7 passed, 85 deselected**, зелёный.

Правка тела дважды сдвинула sha; goldens `iva-role-kmp` перегенерированы тем же способом
(`E2E_INSTALL_REGEN_GOLDEN=1`). Изменились **только строки sha** одного файла:

```
-  "repo/.claude/skills/lite-task-workflow/SKILL.md": "a6b10f63edfbd48b…"
+  "repo/.claude/skills/lite-task-workflow/SKILL.md": "4a7c98f472c6f8c47…"
-  "repo/.agents/skills/lite-task-workflow/SKILL.md": "a6b10f63edfbd48b…"
+  "repo/.agents/skills/lite-task-workflow/SKILL.md": "4a7c98f472c6f8c47…"
```
(`a6b10f63…` — значение на `main`; `4a7c98f4…` — после всей ветки.)

`added: []`, `removed: []` — состав установленного дерева не менялся.

Полный `tests/e2e_install` — **35 failed, 57 passed** и на ветке, и на `main`;
`diff` списков упавших тестов **пустой**, множества совпадают тест в тест. Красное
предсуществует ветке (в CI это nightly-джоба `nightly-install-e2e.yml`).

**Версионная дисциплина:**
```
$ python scripts/check_profile_version_discipline.py --diff-against main
OK — 48 profile(s) clean.
$ python scripts/check_mirror_sync.py
OK — 68 зеркальных ингредиентов в 6 парах синхронны.
```

Оговорка: скрипт печатает только нарушения, при чистом прогоне — одну строку «OK». Списка
затронутых профилей он не выводит **по устройству**, так что «шесть ролей + лейн в выводе
скрипта» получить нельзя. Эквивалентная сверка — по диффу ветки:

```
iva-role-go:        0.2.1 -> 0.2.2
iva-role-ios:       0.1.1 -> 0.1.2
iva-role-java:      0.1.1 -> 0.1.2
iva-role-kmp:       0.2.1 -> 0.2.2
iva-role-mail:      0.1.1 -> 0.1.2
iva-role-web:       0.1.1 -> 0.1.2
tacticum-lite-base: 0.1.3 -> 0.1.4
```

Семь профилей — ровно шесть ролей + лейн, лишнего в дифф не попало.

### 4. `git diff --stat main..HEAD` (вся ветка)

```
 .../golden/iva-role-kmp/claude-code.json           |   2 +-
 .../e2e_install/golden/iva-role-kmp/codex.json     |   2 +-
 templates/iva-role-go/CHANGELOG.md                 |  19 ++++
 templates/iva-role-go/manifest.yaml                |   2 +-
 templates/iva-role-ios/CHANGELOG.md                |  19 ++++
 templates/iva-role-ios/manifest.yaml               |   2 +-
 templates/iva-role-java/CHANGELOG.md               |  19 ++++
 templates/iva-role-java/manifest.yaml              |   2 +-
 templates/iva-role-kmp/CHANGELOG.md                |  19 ++++
 templates/iva-role-kmp/manifest.yaml               |   2 +-
 templates/iva-role-mail/CHANGELOG.md               |  19 ++++
 templates/iva-role-mail/manifest.yaml              |   2 +-
 templates/iva-role-web/CHANGELOG.md                |  19 ++++
 templates/iva-role-web/manifest.yaml               |   2 +-
 templates/tacticum-lite-base/CHANGELOG.md          |  40 +++++++
 templates/tacticum-lite-base/README.md             |   2 +-
 .../ingredients/skills/lite-task-workflow/SKILL.md |  66 +++++++++++-
 .../references/work-order-template.md              | 116 ---------------------
 templates/tacticum-lite-base/manifest.yaml         |  13 ++-
 19 files changed, 233 insertions(+), 132 deletions(-)
```

Коммиты ветки: `769a852` → `61dafd3` → `968460e` → `21d44cc`.

### 5. Что осталось тимлиду (обновлено)

- Пуш и PR — не мой шаг, сегодня наружу ничего не отправлялось.
- Порядок доставки: лейн и роли — **один PR**, разделять нельзя. Смержить только лейн 0.1.4
  без ре-релиза ролей = вернуть ровно тот инертный фикс, который нашёл критик.
- Предсуществующий красный `tests/e2e_install` на `main` (35 failed) — отдельная задача,
  веткой не затронут.

---

## Пятая правка: формулировка про механику доставки (28.07)

Прежняя фраза «механики доставки вспомогательных файлов в платформе нет» **шире правды**.
Проверил по коду: механика есть — ингредиент `kind: repo_config` с обязательным
`metadata.target_file` пишет файл на диск во всех трёх рендерах
(`renderers/claude_code.py:159`, `codex.py:190`, `copilot.py:156`) и в диспетчеризации
`domain/renderer.py:178` (`action: write_file` по `meta["target_file"]`). Нет доставки
**именно у `metadata.assets`**: поле объявительное, рендер ставит только `body_path`.

**`templates/tacticum-lite-base/manifest.yaml`** — комментарий над `ingredients:`
переписан: разведены «у `assets` доставки нет» и «механика класть файл на диск ЕСТЬ, это
`repo_config`». Рядом — почему здесь выбрано НЕ `repo_config`: формат ордера есть ядро
ок-гейта, читается на каждом проходе цикла, и зависимость работоспособности гейта от
доставки второго ингредиента воспроизвела бы тот же класс риска, из-за которого лейн и
правился; для одного файла тело навыка надёжнее.

**`CHANGELOG.md`, секция `[0.1.4]`** — то же уточнение в описании дефекта и в подпункте
про снятый `assets`. Убрана фраза «вызовов `tacticum_init`/`tacticum_fetch_action` нет»
(получена запросом с некорректным фильтром); осталось «человек работал на содержимом роли
от 23.07, куда лейн не входил, — проверено по прод-каталогу».

Коммиты: `9f388a5` (правка формулировки) и `2d9ec9f` (переформатирование абзаца после
неё). Первый закоммичен из этого же worktree под сообщением тимлида — содержимое то же,
что я подготовил.

### Проверки после пятой правки

Правка не трогает тела ингредиентов, только комментарий манифеста и CHANGELOG, — sha в
goldens не сдвинулись, регенерация не понадобилась.

- `pytest tests/catalog` — **710 passed, 0 failed** (без изменений).
- `pytest tests/e2e_install -k kmp` — **зелёный**, 0 failed.
- `check_profile_version_discipline.py --diff-against main` → `OK — 48 profile(s) clean.`
- `check_mirror_sync.py` → `OK — 68 зеркальных ингредиентов в 6 парах синхронны.`

### Итоговый `git diff --stat main..HEAD`

19 файлов, **+246 / −135**. Шесть коммитов ветки:
`769a852` → `61dafd3` → `968460e` → `21d44cc` → `9f388a5` → `2d9ec9f`.
