---
status: current
date: 2026-07-28
author: impl-qa-gate
lead: lead-qa
branch: chore/qa-acceptance-tooling
worktree: ~/tacticum/tacticum-dev-qa-gate
base: main @ 9be499d
permalink: tacticum/00-board/impl-qa-acceptance-2026-07-28
---

# Приёмка QA: инструмент проверки ссылок + фиксация состояния golden

Два куска задачи. **A** — написан инструмент, которого не было. **B** — состояние
golden зафиксировано фактом: прогон состоялся, красный дифф снят, три факта
перепроверены (один уточнён).

## Коротко

- Docker **поднялся** (был лёг, поднят через `open -a Docker`).
- `pytest apps/backend/tests/e2e_install` **прогнан**: 92 теста, **35 failed / 57 passed**.
- Golden красный **не только у QA**: расхождение с golden у **всех 13 профилей × 2 CLI = 26 падений**.
- Новый скрипт находит **216 битых ссылок** в 7 ролях × 2 CLI (`iva-role-go` 59, `iva-role-qa` 44); наше из них — только QA.
- Три факта про golden подтверждены; **число «17» — артефакт грепа, настоящее значение 7** (см. Факт 3).
- Найден **живой баг доставки в `renderer.py:304`** — четыре codex-скилла склеиваются в один файл по литеральному пути `{ingredient_id}`. Описан как готовый тикет; чинит платформа, у нас обход.
- **Команда из README не запускает тесты вообще** — вероятное объяснение, почему красный golden никого не останавливал. Вынесено отдельной задачей.
- Добавлен **режим `paths`** — закрывает дыру, через которую баг и прошёл. По каталогу затронут **один профиль** (наш лейн), не 17: число 17 — пересечение на уровне профиля, а баг требует обоих полей на одном ингредиенте. `iva-analysis-base` **чист**, у аналитиков комплект полный.

---

## A. `scripts/check_install_links.py` — проверка ссылок по дереву установки

Коммит `b4eddfd` в ветке `chore/qa-acceptance-tooling`. 1 файл, ~250 строк, `templates/` не тронут.

### Куда положен и почему не туда, куда просили

Просили `apps/backend/scripts/` «рядом с `check_profile_version_discipline.py` и
`check_mirror_sync.py`». **Оба этих скрипта лежат не там** — они в `scripts/` в корне
репозитория, и именно оттуда их запускает CI
(`.github/workflows/profile-version-discipline.yml`: `python scripts/check_mirror_sync.py`).
В `apps/backend/scripts/` лежит другое — сидеры и dev-утилиты (`seed_community.py`,
`dev_bootstrap.py`). Положил в **`scripts/`**, к названным соседям и к CI. Если нужно
именно `apps/backend/scripts/` — скажи, перенесу, но тогда он разъедется с соседями.

### Зачем он нужен: почему греп по `templates/` этот класс дефектов не видит

Тело скилла ссылается на `references/conventions.md`. В `templates/` этот файл **лежит
рядом** с `SKILL.md` — грепом и глазами ссылка выглядит живой. Ломается она только после
установки, потому что доставки соседей нет:

- `seed_community._build_payload` читает **ровно один файл** на ингредиент (`body_path`,
  для codex — `codex_body_path`); обхода папки нет;
- рендер ставит только `body`;
- поля `metadata.assets` / `metadata.scripts` — **объявительные**. Это не моя догадка, это
  записано в docstring `SkillMetadata` (`apps/backend/src/backend/catalog/domain/ingredients/skill_spec.py`):
  «**Доставки этих файлов сейчас нет** — рендер ставит только `body`. Не выдавать наличие
  поля за наличие механики»;
- **ни один golden-снапшот не содержит ни одного `references/`-файла** — проверил по всем 18 каталогам golden.

Поэтому проверка обязана идти по дереву установки, а не по исходникам.

### Что делает

1. Собирает дерево установки для пары «профиль × CLI»: композиция depth-1 по `depends_on`
   (ADR-0056, позже — выигрывает), фильтр по `supports`, per-CLI тела и цели
   (`codex_body_path` / `codex_target_path`), префикс `user/` для `~/`-путей и `repo/` для
   остальных — тот же контракт, что у `oracles.apply_actions`.
2. Вытаскивает из **каждого доставленного тела** ссылки пяти семейств.
3. Резолвит относительно места установки владельца и сверяет с деревом.

Семейства ссылок:

| Семейство | Пример | Резолв |
|---|---|---|
| `references/…` | `references/conventions.md` | собственный каталог скилла |
| соседний скилл | `fix-failed-test/references/allure-raw-parser.md` | дом скиллов |
| `$CRAFT/…` | `$CRAFT/rules/invariants.md` | `<дом скиллов>/craft-stack/` |
| `${CLAUDE_PLUGIN_ROOT}/…` | `${CLAUDE_PLUGIN_ROOT}/stacks/x.md` | собственный каталог |
| `.agent-kit/…` | `.agent-kit/tools/y.py` | корень repo |

**Пятое семейство добавил сверх задания** — бэктик-ссылка на файл-сосед **без префикса**.
Без него проверка пропускала самый крупный разрыв: `craft-stack/SKILL.md` адресует свой
стек как `` `fix-playbooks.md` ``, `` `rules/invariants.md` ``, `` `shared/pipeline-gate.md` ``.
Точность держится не регуляркой, а условием «профиль **реально авторил** такой файл рядом
с телом владельца» — иначе в улов пошли бы файлы проекта пользователя (`conftest.py`,
`PROGRESS.md`), которые профиль ставить и не должен.

### Про `$TMS/…` и `.agent-kit/…` из задания

- **`$TMS/…` как путь не существует.** В `templates/` `$TMS` встречается один раз — в
  комментарии манифеста. Везде остальное «TMS» это Test Management System (Allure TestOps),
  предметное понятие, не переменная пути. Семейство не заводил: оно ловило бы прозу.
- **`.agent-kit/…` на `main` не встречается ни разу.** Семейство поддержано (проверено
  юнит-прогоном резолва), но сейчас срабатываний нет. Если это форма из нового контента —
  проверка его подхватит без правок.

### Как запускается

Из корня репозитория, как соседние проверки. Единственная зависимость — PyYAML.

```bash
python scripts/check_install_links.py                                    # все роли × оба CLI
python scripts/check_install_links.py --profile iva-role-qa
python scripts/check_install_links.py --profile iva-role-qa --cli codex
python scripts/check_install_links.py --profile iva-role-qa --show-tree  # диагностика
```

Прогонял через backend-окружение (`uv run --project apps/backend python scripts/…`),
потому что системный python3.14 без PyYAML. `ruff check` — чисто.

### Пример вывода (красный, exit 1)

```
$ python scripts/check_install_links.py --profile iva-role-qa --cli claude-code
INSTALL LINKS FAILED — 44 битых ссылок:
  ✗ iva-role-qa [claude-code] write-autotest: 'references/csv-parsing.md' → repo/.claude/skills/write-autotest/references/csv-parsing.md не доставлен (владелец: repo/.claude/skills/write-autotest/SKILL.md)
  ✗ iva-role-qa [claude-code] run-tests: 'fix-failed-test/references/allure-raw-parser.md' → repo/.claude/skills/fix-failed-test/references/allure-raw-parser.md не доставлен (владелец: repo/.claude/skills/run-tests/SKILL.md)
  ✗ iva-role-qa [claude-code] craft-stack: 'rules/invariants.md' → repo/.claude/skills/craft-stack/rules/invariants.md не доставлен (владелец: repo/.claude/skills/craft-stack/SKILL.md)
  ✗ iva-role-qa [claude-code] craft-stack: 'shared/pipeline-gate.md' → repo/.claude/skills/craft-stack/shared/pipeline-gate.md не доставлен (владелец: repo/.claude/skills/craft-stack/SKILL.md)
  …
```

Зелёный (exit 0): `python scripts/check_install_links.py --profile tacticum-role-internal --cli codex`
→ `OK — ссылки целы в 1 парах «профиль × CLI».`

### Что нашёл: 216 битых ссылок

| Профиль | claude-code | codex |
|---|---|---|
| iva-role-go | 59 | 59 |
| **iva-role-qa** | **44** | **44** |
| iva-role-ios / java / kmp / mail / web | по 1 | по 1 |

**Наше — только `iva-role-qa`.** Остальные 128 битых ссылок (`iva-role-go` 118 + пять ролей
по 2) — чужие профили: фиксируем цифрой и отдаём ГД, сами не трогаем. Чинить будем
**доставкой, а не текстом**: 45 файлов канона объявляются ингредиентами и начинают
доезжать. Переписать ссылки в телах под «нечего доставлять» — значит закрепить дефект.

У QA 44 = 29 `references/…` (write-autotest, batch-autotest, fix-failed-test, playwright-cli,
prepare-mr-branch, run-tests) + **15 файлов стека craft-stack** — доставляется только
`craft-stack/SKILL.md`, а весь стек, на который он ссылается (`rules/invariants.md`,
`rules/tests.md`, `rules/page-objects.md`, `shared/pipeline-gate.md`,
`shared/ledger-and-deviations.md`, `shared/coverage-ledger.template.md`,
`shared/jira-candidates.md`, `fix-playbooks.md`, `failure-taxonomy.md`,
`allure-raw-parser.md`, `pytest-runner.md`, `batch-conventions.md`, `recon.md`,
`test-templates.md`, `craft-answers.example.toml`), не доставляется вообще.

### Честная граница инструмента

- Скрипт моделирует **заявленное манифестами** дерево (контракт авторинга), а не текущий
  вывод рендера. Баги самого рендера — предмет e2e-оракулов, не этой проверки.
- В дерево не входят два файла-приёмника, которые **синтезирует рендер**, а не манифест:
  `.mcp.json` (claude, `merge_json` по `mcp_server_spec`) и `~/.codex/config.toml`
  (codex, `renderers/codex.py`). Тела и ссылок у них нет.
- **Сверено с реальностью:** для `iva-role-qa` × claude-code собранное дерево — точное
  подмножество golden (21 из 22 путей, разница ровно `.mcp.json`). То есть реконструкция
  не выдумана.

---

## B. Состояние golden — фактом

### B1. Docker

Демон **был не запущен** (`Cannot connect to the Docker daemon at
unix:///Users/bubblemac/.docker/run/docker.sock`), клиент 28.2.2 при этом отвечал.
Поднял `open -a Docker`, через ~2 минуты `docker info --format '{{.ServerVersion}}'` → `28.2.2`.
Дальше прогон шёл с рабочим демоном.

### B2. Прогон e2e — ссылка на конкретный прогон

**Команда:** `cd apps/backend && uv run --extra dev pytest tests/e2e_install -q -p no:cacheprovider`
**Дата:** 2026-07-28, 13:57:02–13:57:40 UTC
**Репо:** `~/tacticum/tacticum-dev-qa-gate`, ветка `chore/qa-acceptance-tooling` от `main @ 9be499d`
**Результат:** `EXIT=1` — **92 теста: 35 failed, 57 passed** (xfail/skip — 0)
**Полный лог:** `/private/tmp/claude-501/-Users-bubblemac-tacticum-vault/598cba87-f04d-46da-9ae8-a23ed99f697e/scratchpad/e2e_run2.log`

**Отдельно про «pytest не запускается».** Подтверждаю, и причина конкретная. Команда из
`tests/e2e_install/README.md` — `uv run pytest tests/e2e_install -q` — **не работает**:
`pytest` не в зависимостях проекта (он в extra `dev`), uv не находит его в окружении и
падает на **системный** `/Library/Frameworks/Python.framework/Versions/3.12/bin/pytest`, а
там в `site-packages` лежит плагин `langsmith`, который валит **сбор тестов** на
`ModuleNotFoundError: No module named 'pydantic'`. Ни один тест не стартует. Рабочая
команда — **`uv run --extra dev pytest`**. Это правка в README на одно слово, и без неё
любой, кто идёт по инструкции, видит стену трейсбека вместо прогона.

**Разбор 35 падений:**

| Причина | Падений |
|---|---|
| дерево не совпало с golden | **26** (13 профилей × 2 CLI) |
| `profile_not_found` (`iva-go-backend-brownfield` не сидится) | 5 |
| `depends_on_missing_ref`: `iva-role-go` → `tacticum-lite-base` не существует | 2 |
| прочее (`test_start_task_no_subagent`) | 2 |

Главное здесь: **golden красный не у QA, а у всех**. Профили с расхождением: `iva-role-qa`,
`iva-role-web`, `iva-role-analyst`, `iva-role-mail`, `iva-role-ios`, `tacticum-role-internal`,
`tacticum-role-platform`, `firebird-role-web`, `iva-web-brownfield`, `iva-ios-brownfield`,
`firebird-web-brownfield`, `tacticum-dev-base`, `e2e-two-base-dependent`.

Ещё: `xfail` в прогоне **ноль**, хотя `tests/e2e_install/README.md` в разделе «Known defect»
утверждает, что claude-code-ячейки помечены `xfail(strict=True)`. Метки сняты, README про
это не знает — раздел врёт.

### B3. Красный дифф golden для `iva-role-qa`

`claude-code` — состав совпадает, поехало **содержимое 11 файлов**:

```
added (not in golden):    []
removed (missing files):  []
changed (content drift):  ['repo/.claude/agents/code-writer.md', 'repo/.claude/agents/dom-explorer.md',
  'repo/.claude/skills/batch-autotest/SKILL.md', 'repo/.claude/skills/craft-stack/SKILL.md',
  'repo/.claude/skills/fix-failed-test/SKILL.md', 'repo/.claude/skills/playwright-cli/SKILL.md',
  'repo/.claude/skills/prepare-mr-branch/SKILL.md', 'repo/.claude/skills/rebuild-autocore/SKILL.md',
  'repo/.claude/skills/retro/SKILL.md', 'repo/.claude/skills/run-tests/SKILL.md',
  'repo/.claude/skills/write-autotest/SKILL.md']
```

`codex` — поехал **состав**:

```
added (not in golden):    ['repo/.agents/skills/craft-stack/SKILL.md', 'repo/.agents/skills/run-tests/SKILL.md',
  'repo/.agents/skills/{ingredient_id}/SKILL.md', 'repo/.codex/agents/code-writer.toml',
  'repo/.codex/agents/codebase-analyst.toml', 'repo/.codex/agents/dom-explorer.toml',
  'repo/.gitignore', 'repo/secrets.yaml.example']
removed (missing files):  []
changed (content drift):  ['repo/.agents/skills/playwright-cli/SKILL.md', 'repo/.agents/skills/prepare-mr-branch/SKILL.md',
  'repo/.agents/skills/rebuild-autocore/SKILL.md', 'repo/.agents/skills/retro/SKILL.md']
```

> В списке добавленных — `repo/.agents/skills/{ingredient_id}/SKILL.md`, литеральный
> неотформатированный плейсхолдер. Это не расхождение снапшота, а **дефект доставки в
> рендере**; разобран отдельным разделом ниже как готовый тикет (см. «Баг рендера»).

---

## Баг рендера — готовый тикет для платформенной команды

**Скоуп не наш.** `renderer.py` — платформенный код, мы туда не лезем. Обход на нашей
стороне (замена `{ingredient_id}` литеральными именами в манифесте лейна) делает второй
воркер — это наш файл, и этого достаточно, чтобы у инженера появились все четыре скилла.
Ниже — описание для передачи как есть.

**Заголовок.** `render_for_codex`: `codex_target_path` доставляется без
`.format(ingredient_id=…)` — четыре skill_spec схлопываются в один файл

**Где.** `apps/backend/src/backend/catalog/domain/renderer.py`
- функция `_cli_body_passthrough`, строки **287–304**;
- дефектная строка — **304**:
  ```python
  return {"action": "write_file", "path": path, "content": body, "compat": "native"}
  ```
  `path` взят из `meta[<cli>_target_path]` (строка 301) и подставляется **дословно**;
  вызова `.format(ingredient_id=…)` в этой ветке нет вообще.
- вызывается из `_render_via_canonical`, строки **332–335** (passthrough уходит в
  `actions` до канонического рендера и мимо Pydantic-валидации);
- путь живой: `render_for_codex` (строка 382) → `_render_via_canonical` → это.

**Чем отличается от `:211` (где то же самое уже починено).** В `render_for_claude_code`
общая ветка `_WRITE_FILE_KINDS` шаблон форматирует — строки **210–211**:

```python
template = tpt or _WRITE_FILE_KINDS[kind]
path = template.format(ingredient_id=ingredient_id)
```

с комментарием про Issue #658 («the template must be formatted — assigning it verbatim
collapses every ingredient onto one literal `{ingredient_id}` path»). То есть **этот
класс дефекта уже был найден и закрыт — но только в claude-ветке**. Ветка
`_cli_body_passthrough` появилась позже (BL-1, per-CLI тела агентов) и починку не
получила: ровно тот же баг, второй экземпляр.

**Почему не всплывал до QA-лейна.** Docstring `_cli_body_passthrough` (строка 290)
прямо заявляет: *«Only agent ingredients carry a per-CLI body»*. У агентов
`codex_target_path` — литеральный путь (`.codex/agents/codebase-analyst.toml`),
плейсхолдера в нём нет, и отсутствие `.format()` ничего не ломало. QA-лейн первым
применил `codex_body_path` к `skill_spec` (в манифесте это помечено как **R7-FLAG**,
ожидающий ратификации), а у скиллов `codex_target_path` —
`.agents/skills/{ingredient_id}/SKILL.md`. Предпосылка функции нарушена → баг проявился.

**Механика потери — конкатенация, а не затирание.** Все четыре действия получают
одинаковый литеральный `path`, после чего `_dedupe_actions_by_path` (строки 344–373,
склейка на строке **366**) сливает их в одно действие, **склеивая содержимое через
`\n\n`**. Проверил прямым вызовом функции:

```
actions in : 4
actions out: 1
path       : .agents/skills/{ingredient_id}/SKILL.md
content    : 'BODY-write-autotest\n\nBODY-batch-autotest\n\nBODY-fix-failed-test\n\nBODY-jira-issue-autotest'
```

*(Поправка к моему первому отчёту лиду: я сказал «последний затирает предыдущих» — это
неверно. Содержимое склеивается.)*

**Последствие для пользователя.** Codex-инженер QA-роли вместо четырёх скиллов получает
**один файл по несуществующему пути** `.agents/skills/{ingredient_id}/SKILL.md`, внутри
которого четыре разных SKILL.md подряд. CLI такой путь скиллом не увидит, то есть
`write-autotest`, `batch-autotest`, `fix-failed-test` и `jira-issue-autotest` не работают
ни один — при том что в снапшоте формально «файл есть».

**Как воспроизвести.** `cd apps/backend && uv run --extra dev pytest
tests/e2e_install -q -k "roles_generic and codex and iva-role-qa"` → в дифже с golden
появляется путь с литеральным `{ingredient_id}`.

**Ожидаемое поведение.** Форматировать путь в passthrough так же, как на строке 211:
`path.format(ingredient_id=ing.ingredient_id)`. Дополнительно стоит проверить, не нужен ли
явный запрет литерального `{` в целевом пути — тогда дефект падал бы громко, а не
маскировался склейкой.

**Побочно.** `assert_unique_ingredients` этот случай не поймал: после склейки действие
одно, дубликатов нет. Оракул «пути не содержат неразвёрнутых плейсхолдеров» в наборе
отсутствует — это дыра в приёмке, а не только в рендере. **Дыра закрыта** режимом `paths`
в `check_install_links.py` — см. раздел ниже.

---

### B4. Три факта — перепроверка

**Факт 1 — подтверждён точно.** `golden/iva-role-qa/claude-code.json` держит для
`repo/.claude/skills/run-tests/SKILL.md` хеш
`00e763a0a50a73db4cb32e50dfeaf685b56ea77811aa9791c0f0721d5799963c`. Текущее тело
(`templates/iva-qa-autotest-base/ingredients/skills/run-tests/SKILL.md`) даёт
`0e955299f407e4a6d68e045c8fe24d9ede8dee8da1acf7fe90b43189e06354be`. Тело живёт в лейне,
не в роли: `iva-role-qa` — роль-пресет, субстанция приходит через `depends_on`.

**Факт 2 — подтверждён точно.** `codex.json` — 11 ключей, `claude-code.json` — 22. В codex
нет ни одного из трёх субагентов (`code-writer`, `codebase-analyst`, `dom-explorer`) и нет
`write-autotest`, `batch-autotest`, `fix-failed-test`, `jira-issue-autotest`, `craft-stack`,
`run-tests`.

**Факт 3 — подтверждён по сути, число уточнено.** Причина названа верно и доказывается
git-историей:

- golden создан в **PR #133**, коммит `20ff9b8` от 2026-07-24, и **с тех пор не менялся ни разу**
  (`git log -- golden/iva-role-qa/` — ровно один коммит);
- хеш `run-tests` **на момент `20ff9b8`** = `00e763a0…`, то есть golden тогда содержанию соответствовал;
- на `20ff9b8` у `run-tests` было `supports: [claude-code]` — потому его в codex-снапшоте и нет;
  сейчас `supports: [claude-code, codex]` + `codex_target_path`;
- `codex_body_path` в `templates/iva-qa-autotest-base/manifest.yaml` на `20ff9b8` — **0**, сейчас — **7**.

**Уточнение: не 17, а 7.** «17» — это `grep -c`, то есть число *строк*, где встречается
подстрока; 10 из них — комментарии (`# codex_body_path passthrough`, R7-FLAG и пр.).
Реальных полей `codex_body_path:` в манифесте — **7** (4 скилла + 3 агента). Направление
факта (было 0 → стало не-ноль, отсюда стухший codex-снапшот) верное, число — артефакт
инструмента. Проверять такое надо парсером yaml, не грепом.

### B5. Golden не перегенерирован

Как договорились — не трогал. `E2E_INSTALL_REGEN_GOLDEN=1` не запускал.

**Процедура перегенерации, когда контент устаканится:**

```bash
cd apps/backend
E2E_INSTALL_REGEN_GOLDEN=1 uv run --extra dev pytest tests/e2e_install -q   # перезапись
uv run --extra dev pytest tests/e2e_install -q                             # проверка: зелено
git diff --stat apps/backend/tests/e2e_install/golden/
```

Условия, без которых перегенерация закрепит дефекты вместо того, чтобы их снять:

1. контентные ветки QA **влиты**, иначе снапшот придётся выбрасывать;
2. баг `{ingredient_id}` в codex-passthrough **починен** — иначе литеральный плейсхолдер
   и потеря трёх скиллов законсервируются в golden как «норма»;
3. `check_install_links.py` **зелёный** — иначе консервируются битые ссылки;
4. дифф golden **прочитан глазами**: перегенерация обязана быть объяснимой построчно.

---

## Режим `paths` — закрыта дыра, из-за которой баг прожил незамеченным

Коммит `30d123e`. Добавлен вторым режимом в тот же скрипт, **не** в общие оракулы
`tests/e2e_install` — там он покраснел бы на чужом долге и его сняли бы, ровно как с
CI-гейтом.

### Почему старая приёмка это пропускала

`assert_unique_ingredients` ищет дубликаты действий. Но четыре действия с одинаковым
путём **склеиваются** в `_dedupe_actions_by_path` в одно — дубликатов не остаётся, оракул
доволен. Проверки «в пути нет неразвёрнутого плейсхолдера» в наборе не было вообще.
Дефект проходил обе.

### Что проверяет

Режим моделирует **фактическую** доставку (`build_actions`), а не заявленную:
воспроизводит ветку `_cli_body_passthrough`, отдающую `<cli>_target_path` дословно.

1. **Неразвёрнутый плейсхолдер** — `{…}` в пути действия. Гарантированный дефект: такого
   пути CLI не увидит.
2. **Схлопывание** — несколько **разных** ингредиентов в одном пути.

Две поправки на ложные срабатывания, обе существенные:

- **Один и тот же `ingredient_id` из двух профилей композиции — не схлопывание.** Лейн-
  владелец и зеркало старого профиля (ADR-0059) объявляют один ингредиент дважды; это
  штатная коллизия «позже выигрывает». Без этой поправки режим давал 18 находок, из них
  16 ложных (`pin-authoring ← pin-authoring` в `firebird-web-brownfield`, `iva-ios-brownfield`,
  `iva-role-kmp`). Сначала сворачиваю по `ingredient_id`, потом группирую по пути.
- **Слияние в общий файл — штатное.** `instruction_pack` в `CLAUDE.md`/`AGENTS.md` и
  `mcp_server_spec` в `.codex/config.toml` сливаются по замыслу (`_dedupe_actions_by_path`,
  E26 S5 Finding-4). Схлопывание считаю только для kind'ов «один ингредиент = один файл»:
  `skill_spec`, `agent_spec`, `command_spec`, `rule_set`.

### Запуск

```bash
python scripts/check_install_links.py --mode paths --all-profiles
python scripts/check_install_links.py --profile iva-role-qa --mode paths
```

`--mode` — `links` | `paths` | `all` (по умолчанию `all`). Добавлен `--all-profiles`:
весь каталог, а не только роли с `depends_on`. Ненулевой код возврата при находках.
`ruff check` чисто.

### Результат по каталогу

```
$ python scripts/check_install_links.py --all-profiles --mode paths
INSTALL PATHS FAILED — 2 дефектных путей:
  ✗ iva-qa-autotest-base [codex] repo/.agents/skills/{ingredient_id}/SKILL.md: неразвёрнутый плейсхолдер в пути; схлопывание 4 ингредиентов в один путь ← batch-autotest, fix-failed-test, jira-issue-autotest, write-autotest
  ✗ iva-role-qa [codex] repo/.agents/skills/{ingredient_id}/SKILL.md: неразвёрнутый плейсхолдер в пути; схлопывание 4 ингредиентов в один путь ← batch-autotest, fix-failed-test, jira-issue-autotest, write-autotest
```

**Затронут один профиль — наш лейн `iva-qa-autotest-base`. Теряются 4 `skill_spec`:**
`write-autotest`, `batch-autotest`, `fix-failed-test`, `jira-issue-autotest`. Две строки —
это один дефект: лейн сам по себе и роль `iva-role-qa`, которая его композит.

### Сверка с числом 17 — оно завышено, реальное 1

Твои 29 подтверждаю точно: **29 профилей** имеют шаблонный `codex_target_path`.
Число 17 воспроизвёл и понял, откуда оно: это пересечение **на уровне профиля** —
где-то в манифесте есть шаблонный путь И где-то есть `codex_body_path`.

Но баг требует **обоих полей на ОДНОМ ингредиенте**, и вот почему:

- сидер (`seed_community`, строки 79–91) кладёт `codex_target_path` в metadata **только
  внутри** ветки «есть `codex_body_path`» — без per-CLI тела путь из манифеста в metadata
  вообще не попадает;
- без `codex_body` не срабатывает `_cli_body_passthrough` (нужны оба, строка 302), и
  работает канонический рендер;
- канонический рендер `codex_target_path` **игнорирует** и зашивает свой путь настоящей
  f-строкой (`renderers/codex.py:79`: `f".agents/skills/{s.ingredient_id}/SKILL.md"`).

Пересечение по этому — настоящему — условию: **1 профиль**, 4 ингредиента.

| Замер | Число |
|---|---|
| профилей с шаблонным `codex_target_path` | 29 |
| профилей с `codex_body_path` где-либо | 18 |
| пересечение **на уровне профиля** (даёт «17») | 17 |
| пересечение **на одном ингредиенте** (условие бага) | **1** |

В остальных 16 из 17 сочетание безопасное: `codex_body_path` стоит на `agent_spec`, а у
агентов `codex_target_path` литеральный (`.codex/agents/<id>.toml`, без `{…}`) — дословная
подстановка ничего не ломает. Шаблонные же пути стоят на скиллах **без** `codex_body_path`,
которые идут каноническим рендером. Именно поэтому баг и не всплывал до QA-лейна: тот
первым применил `codex_body_path` к `skill_spec` (в манифесте помечено как **R7-FLAG**,
ждущий ратификации).

**Эмпирическая сверка.** В логе прогона e2e литеральный `{ingredient_id}` встречается
**ровно один раз**, в дифже `golden/iva-role-qa/codex.json`. Статический замер и
фактический прогон сошлись.

### `iva-analysis-base` — чисто, комплект полный

```
$ python scripts/check_install_links.py --profile iva-analysis-base --mode paths
OK — 2 пар «профиль × CLI», режим paths: чисто.
```

Профиль аналитика (v0.1.8, 25 ингредиентов) под этот дефект **не попадает**, хотя в списке
«17» он есть. Разбор по составу:

- **14 `skill_spec`** (`fr-authoring`, `tests-authoring`, `adr-authoring`, `brd-authoring`,
  `api-contracts-discovery`, `data-model-analyzer`, `events-analyzer`,
  `multi-container-pin-authoring`, `pin-api-verification`, `pin-upstream-dependency-check`,
  `process-analysis-stage`, `process-arch-signoff`, `design-system-discovery`,
  `mockup-authoring`) — шаблонный `codex_target_path`, но `codex_body_path` **нет ни у
  одного**. Идут каноническим рендером, путь зашит правильно.
- **2 `agent_spec`** (`system-analyst`, `tacticum-workflow`) — `codex_body_path` есть, но
  путь литеральный `.codex/agents/<id>.toml`, плейсхолдера нет.
- 5 `command_spec` и 4 `mcp_server_spec` — вне механики.

**Вывод: люди, которые сейчас работают с профилем аналитика, получают полный комплект
скиллов.** Схлопывания у них нет. Чинить там нечего, и мы туда не лезем.

*(Оговорка о границе: это про данный дефект — плейсхолдеры и схлопывание путей. По
битым ссылкам `iva-analysis-base` я не мерил: он не роль, в прогон по ролям не входил.
Если нужно — замер одной командой, но это уже чужой скоуп.)*

---

## Golden красный по всему каталогу — самостоятельная находка

Это шире QA. Из 35 падений прогона **26 — это расхождение дерева с golden**, и они
покрывают **все 13 профилей, у которых есть golden-снапшот, оба CLI без исключения**.
Не «сломался QA-профиль», а **приёмка каталога не держит ни одного профиля**.

| Профиль | CLI | added | removed | changed |
|---|---|---:|---:|---:|
| iva-role-qa | claude-code | 0 | 0 | **11** |
| iva-role-qa | codex | **8** | 0 | 4 |
| iva-role-analyst | оба | 6 | 0 | 6 |
| iva-role-web | оба | 7 | 0 | 3 |
| iva-role-ios | оба | 4 | 0 | 3 |
| iva-role-mail | оба | 4 | 0 | 3 |
| iva-web-brownfield | оба | 3 | 0 | 6 |
| iva-ios-brownfield | оба | 0 | 0 | 4 |
| firebird-web-brownfield | оба | 0 | 0 | 4 |
| firebird-role-web | оба | 0 | 0 | 3 |
| tacticum-dev-base | оба | 0 | 0 | 2 |
| tacticum-role-internal | оба | 0 | 0 | 2 |
| tacticum-role-platform | оба | 0 | 0 | 2 |
| e2e-two-base-dependent | оба | 0 | 0 | 2 |

Три вещи, которые из этой таблицы читаются и которые стоит донести:

1. **`removed` равен нулю везде.** Ни один профиль ничего не потерял. Расхождение
   одностороннее: контент ушёл вперёд, снапшоты остались. Это не деградация доставки —
   это заброшенная приёмка.
2. **У шести профилей `added` > 0** — в дереве есть ингредиенты, о которых golden не
   знает вообще. То есть снапшоты отстали не на правку текста, а на состав.
3. **Красный статус давно перестал что-либо значить.** Раз золото красное у всех и по
   любому поводу, сигнал «дифф с golden» неотличим от шума — и дефект QA-профиля проехал
   ровно через эту слепую зону. Механика приёмки цела (оракулы отработали и всё поймали),
   отсутствует привычка держать её зелёной.

Плюс к этому — два независимых симптома того же запустения, оба вне QA:

- `iva-role-go` **не сидится вообще**: `depends_on_missing_ref` — зависит от
  `tacticum-lite-base`, которого не существует (2 падения);
- `iva-go-backend-brownfield` не сидится → `profile_not_found` (5 падений).

То есть на `main` два профиля физически не устанавливаются, и это не ловится ничем, кроме
этого же прогона, который никто не запускал (см. следующий раздел — команда из README
тесты не запускает в принципе).

---

## README `tests/e2e_install` — команда не запускает тесты (отдельная мелкая задача)

Отдельным пунктом, как договорились. Правку не делал.

**Где:** `apps/backend/tests/e2e_install/README.md`, раздел «How to run».

**Что написано (не работает):**

```bash
cd apps/backend
uv run pytest tests/e2e_install -q
```

**Что происходит.** `pytest` не в зависимостях проекта, он в extra `dev`. uv не находит его
в окружении проекта и падает на **системный**
`/Library/Frameworks/Python.framework/Versions/3.12/bin/pytest`; там в `site-packages`
лежит плагин `langsmith`, который валит **сбор тестов**:
`ModuleNotFoundError: No module named 'pydantic'`. Ни один тест не стартует — на экране
стена трейсбека вместо прогона.

**Рабочая команда:**

```bash
cd apps/backend
uv run --extra dev pytest tests/e2e_install -q
```

**Заодно поправить в том же файле:** раздел «Known defect (claude-code cases xfail)»
утверждает, что claude-code-ячейки помечены `xfail(strict=True)`. В прогоне **xfail = 0**,
метки сняты — раздел описывает несуществующее состояние.

**Почему это важнее, чем выглядит.** Пока команда неверна, любой гейт поверх e2e либо не
запустится, либо «пройдёт», не проверив ничего. Это правдоподобное объяснение, почему
красный golden никого не останавливал.

---

## Предложение по CI-гейту

`.github/workflows` **не трогал** — как договаривались, сначала описываю.

**Предлагаю: добавить шаг в существующий `profile-version-discipline.yml`, а не новый workflow.**

Почему туда: workflow уже стоит на `pull_request` + `push:main` по путям `templates/**`,
уже ставит Python 3.12 и PyYAML — ровно то, что скрипту нужно, других зависимостей нет.
Новый workflow дублировал бы триггеры и чекаут ради одной команды.

Что добавить — три строчки после шага mirror-sync:

```yaml
      - name: Install links check (ссылки ведут на доставленные файлы)
        run: python scripts/check_install_links.py
```

и в оба блока `paths` — `"scripts/check_install_links.py"`, чтобы правка самого скрипта
прогоняла проверку.

**Почему это безопасно:** шаг read-only (только читает `templates/`, ничего не пишет и
никуда не ходит по сети), зависимость уже установлена шагом выше, время — доли секунды.
Риск ровно один и он не технический: **на `main` сейчас 216 битых ссылок, гейт станет
красным немедленно**. Поэтому:

**Порядок включения — гейт последним, не первым.** Сначала контентные ветки закрывают
разрыв доставкой недостающих файлов, потом включается шаг. Иначе первый же PR упрётся в
чужой долг, и гейт отключат — как это обычно и происходит.

**Статус: не включён.** Лид подтвердил и порядок, и площадку; держим как предложение до
возвращения Президента. `.github/workflows` в ветке не тронут.

**Важное уточнение после режима `paths`: два режима готовы к включению по-разному.**

| Режим | Состояние каталога | Когда можно вешать |
|---|---|---|
| `--mode paths` | **чисто везде, кроме нашего лейна** | сразу после правки второго воркера |
| `--mode links` | 216 битых, из них 128 чужих | только после вычистки контента |

То есть `paths` не упирается в чужой долг вообще: единственная его находка — наша
собственная, и она закрывается той же ветке. Его можно включать на весь каталог
(`--all-profiles --mode paths`) сразу, не дожидаясь ничего, и он с первого дня будет
держать ровно тот класс дефекта, который проехал мимо приёмки. `links` — по прежнему
плану, после вычистки.

**Промежуточный вариант, если гейт нужен раньше:** повесить шаг с `--profile` на те роли,
которые уже вычищены, и расширять список по мере закрытия. Список профилей в команде виден
в дифф-ревью — это честнее, чем `continue-on-error`, который делает шаг декоративным.

**Отдельно от гейта ссылок — то, что ломает приёмку сильнее:** `pytest` в CI по инструкции
из README не стартует вообще (B2). Пока команда не исправлена на `uv run --extra dev pytest`,
любой гейт поверх e2e будет либо не запускаться, либо «проходить» ничего не проверив.

---

## Что не делал

- `templates/` — не трогал (там параллельный воркер).
- Тела скиллов — не менял.
- `.github/workflows` — не менял, только предложение выше.
- Golden — не перегенерировал.
- Не пушил. Ветка `chore/qa-acceptance-tooling` лежит локально, коммит `b4eddfd`.

## Решения лида по четырём вопросам — закрыты

1. **Скрипт остаётся в `scripts/`.** Лид перепроверил сам: оба названных соседа лежат
   именно там. Задание было неточным, ничего не переносим.
2. **Баг рендера — заводить, но чинить не нам.** `renderer.py` платформенный, туда не
   лезем. Описан выше как готовый тикет (файл, строки, отличие от `:211`, механика
   склейки, воспроизведение). Обход на нашей стороне — замена `{ingredient_id}`
   литеральными именами в манифесте лейна — у второго воркера.
3. **Ссылки чиним доставкой, не текстом.** 45 файлов канона объявляются ингредиентами.
   Чужие профили (128 битых ссылок) не трогаем — цифра уходит ГД.
4. **README — отдельной задачей.** Выписан отдельным разделом с точной рабочей командой.

## Что осталось на стороне лида / ГД

- отдать платформенной команде тикет по `renderer.py:304`;
- отдать ГД две находки: golden красный по всему каталогу (26 падений, 13 профилей) и
  128 битых ссылок в чужих профилях;
- завести мелкую задачу на README `tests/e2e_install`;
- включить CI-шаг — после вычистки контента.
---

# Заход 2 (вечер 28.07): ложное срабатывание, CI-гейт, golden

Коммиты `6f55b4f` (инструмент) и `ed63b6a` (CI) в той же ветке
`chore/qa-acceptance-tooling`. Не пушил.

## Кусок 1 — шесть «битых ссылок» были артефактом macOS, а не рантайм-артефактом

Постановка предполагала, что чинить надо признак «рантайм-артефакт vs поставляемый файл».
Разбор показал другую причину, и она проще.

`_sibling_links` засчитывал соседа так: `(owner_dir / 'PROGRESS.md').is_file()`. Рядом с
телами лежит `references/progress.md` — **строчными**. APFS на macOS по умолчанию
регистронезависима и отвечала на этот вопрос `True`. На Linux (CI, ext4) тот же вызов
отвечает `False`.

То есть инструмент давал **разный вердикт локально и в CI** — это хуже шести находок:
на такой проверке нельзя строить гейт, она бы «чинилась» самим фактом переезда на runner.

**Починка:** `_authored_here()` сверяет имя с фактическими элементами каталога
(`Path.iterdir()`), покомпонентно, а не спрашивает ФС. Вердикт одинаков на любой ФС.

Признак **надёжен**: это не эвристика, а устранение вранья файловой системы.

### Фильтр «артефакты рантайма» написан и снят — осознанно

Сначала сделал ровно то, что просили: признак не по имени, а по смыслу — имя считается
рантайм-артефактом, если хоть где-то в дереве установки оно адресовано путём внутри дома
задач `.tasks/` (`.tasks/work/wave-<кампания>/PROGRESS.md`, `.tasks/metrics.jsonl`).
Признак берётся из самих тел, ничего не хардкодится.

Замер показал, что фильтр **нулевой и вредный**:

| | до (ФС-регистр) | только регистр | регистр + фильтр |
|---|---|---|---|
| templates ветки канона, весь каталог | 134 | **128** | **128** |
| templates этой ветки (= main) | 216 | 216 | 216 |

Фильтр не меняет НИЧЕГО нигде. И не может: сосед попадает в улов только когда профиль
**реально авторил** файл с таким именем рядом с телом — а это ровно случай «авторил, но
не доставил», то есть настоящий дефект. Фильтр глушил бы его на 18 именах, которые
дерево QA объявляет рантайм-артефактами: `README.md`, `known-issues.md`, `analysis.md`,
`state.md`, `input.md`, `diff.md`, `INDEX.md`, `coverage-ledger.md` и др. Это широкая
слепая зона ради гипотетической выгоды.

**Класс «завтра появится другой артефакт» после починки регистра не воспроизводится:**
имя без авторенного файла рядом в улов не попадает вообще.

Проверил заодно смежное: манифестов с расхождением регистра в `body_path` /
`codex_body_path` нет (599 полей) — то есть профиля, который ставится на macOS и падает
на Linux, в каталоге нет.

## Кусок 2 — два шага в `profile-version-discipline.yml`

Новый workflow не заводил: этот уже стоит на `templates/**`, уже ставит Python 3.12 и
PyYAML. В `on.paths` добавлен `scripts/check_install_links.py`.

```yaml
- name: Install paths check (весь каталог)
  run: python scripts/check_install_links.py --mode paths --all-profiles

- name: Install links check (только вычищенные профили)
  run: python scripts/check_install_links.py --mode links --profile iva-role-qa
```

К шагу `links` — комментарий на 10 строк, почему ограничение есть и почему его нельзя
«починить», сняв: по каталогу режим даёт 128 находок чужого долга, первый же чужой PR
упрётся в то, чего его автор не создавал, и гейт снимут вместе с пользой.
`continue-on-error` там же назван негодным: молчащий гейт хуже отсутствующего. Правильный
путь расширения — вычистить профиль и дописать ещё один `--profile`; список растёт, не
снимается.

### ⚠️ Порядок мержа — шаг `paths` красный до мержа канона

На templates **текущего main** `--mode paths` находит дефект:

```
iva-role-qa [codex] repo/.agents/skills/{ingredient_id}/SKILL.md:
  неразвёрнутый плейсхолдер; схлопывание 4 ингредиентов
  ← batch-autotest, fix-failed-test, jira-issue-autotest, write-autotest
```

Он закрыт в ветке `feat/qa-canon-delivery`. Значит **`chore/qa-acceptance-tooling`
мержится строго после неё**, иначе main краснеет сразу.

## Замеры после правки (оба режима, свои и чужие профили)

Код в обоих прогонах один — тот, что в ветке. Отличаются только `templates/`.

**templates этой ветки (= main @ 9be499d):**

| прогон | exit | результат |
|---|---|---|
| `--mode paths` (роли) | 1 | 1 дефектный путь |
| `--mode paths --all-profiles` | 1 | 2 дефектных пути |
| `--mode links` (весь каталог) | 1 | 216 битых |
| `--mode links --profile iva-role-qa` | 1 | 88 |
| go / web / ios / java / kmp / mail | 1 | 118 / 2 / 2 / 2 / 2 / 2 |

**templates ветки `feat/qa-canon-delivery` (читал, не менял):**

| прогон | exit | результат |
|---|---|---|
| `--mode paths` (роли, 30 пар) | 0 | чисто |
| `--mode paths --all-profiles` (86 пар) | 0 | чисто |
| `--mode links` (весь каталог) | 1 | 128 (весь — чужой долг) |
| `--mode links --profile iva-role-qa` | 0 | **чисто** |
| go / web / ios / java / kmp / mail | 1 | 118 / 2 / 2 / 2 / 2 / 2 |

Чужие профили не сдвинулись ни на единицу — правка на них не влияет.

## Кусок 3 — golden: перегенерировать могу, но не отсюда

**Технически готов.** Проверено фактически, а не предположением:

- `docker info` работает; сессионный `postgres:16-alpine` поднимается;
- `uv run pytest tests/e2e_install -k "roles_generic and iva-role-qa"` доезжает до конца;
- обе ячейки падают **ровно на сверке с golden** (`oracles.py:169`), всё остальное —
  состав в БД против манифестов, уникальность ингредиентов, маркер, `bulk == manifest` —
  зелёное. То есть конвейер исправен, устарел только снимок;
- `xfail` на claude-code-ячейках в `test_install_flow.py` больше нет (README про них
  устарел), так что перегенерация не упрётся в `xpass strict`.

**Подтверждение устаревания:** оба файла в последний раз писались коммитом `20ff9b8`
(24.07, PR #133). `codex.json` — 11 ключей, в нём нет ни одного QA-скилла
(`batch-autotest`, `write-autotest`, `run-tests`, `fix-failed-test`,
`jira-issue-autotest`, `craft-stack`) и ни одного агента. `claude-code.json` — 22 ключа.

**Почему не сделал.** `E2E_INSTALL_REGEN_GOLDEN=1` снимает дерево с `templates/` **того
worktree, где запущен**. У меня это templates main'а — то есть я записал бы в golden
дефектный контент, который канон прямо сейчас заменяет. Это хуже устаревшего golden:
устаревший виден как красный тест, а свежеснятый с брака выглядит легитимным.

**Процедура, когда контент замрёт** (нужно ваше «замри»):

```bash
cd <worktree с финальным контентом>/apps/backend
E2E_INSTALL_REGEN_GOLDEN=1 uv run pytest tests/e2e_install -q \
  -k "roles_generic and iva-role-qa" -p no:randomly
git status --short apps/backend/tests/e2e_install/golden   # обязано быть ровно 2 файла
uv run pytest tests/e2e_install -q -k "roles_generic and iva-role-qa"  # зелено
```

`-k` держит границу «только iva-role-qa»: чужие golden красные по всему каталогу, это
отдельный общекаталожный долг, и трогать их этой задачей нельзя.

**Где запускать — решение за лидом.** Контент живёт в `feat/qa-canon-delivery`, и golden
физически обязан лечь туда же (или в main после мержа канона): снимок и контент,
которому он соответствует, должны ехать одним куском. В ветку 1 я не лезу — там второй
воркер. Варианты:

1. **тот, кто дорабатывает канон, регенерирует у себя** после заморозки — golden уезжает
   тем же PR, что и контент. Самый чистый;
2. **я делаю это после мержа канона в main** — тогда моей ветке нужен ребейз, и golden
   едет отдельным PR вслед. Дольше, зато без пересечения по веткам;
3. дать мне ветку 1 на короткое окно, когда второй воркер из неё вышел.

`~/tacticum/tacticum-dev-qa-canon/apps/backend/.venv` на месте — вариант 1 работает без
подготовки.

## Что не трогал

- ветка `feat/qa-canon-delivery` и её worktree — только читал `templates/` для замеров;
- `templates/` — ни одного байта;
- чужие профили и их golden;
- не пушил.

---

## Решения лида по заходу 2 (закрыты)

1. **Фильтр рантайм-артефактов не возвращаем.** Замер убедителен: нулевой везде, глушил бы
   настоящий класс «профиль авторил файл, но не доставил» на 18 именах. Код в истории
   (`6f55b4f` — снят в том же коммите, где описан).
2. **Golden — вариант 1:** регенерирует тот, кто дорабатывает канон, в своём worktree,
   после Т7 и Т10. Раннбук ниже.
3. **Порядок мержа зафиксирован:** `chore/qa-acceptance-tooling` — строго после
   `feat/qa-canon-delivery`.

---

# РАННБУК: перегенерация golden для `iva-role-qa`

Исполнителю: это самодостаточная инструкция, контекст задачи знать не нужно. Выполнять
**только когда контент профиля в ветке заморожен** — golden фиксирует байты тел, любая
правка после снятия делает его устаревшим.

## Что это и почему осторожно

`apps/backend/tests/e2e_install/golden/<profile_id>/<target_cli>.json` — снимок дерева
установки: `путь → sha256` доставленных файлов. Тест ставит профиль по-настоящему
(сид → provision → pull → apply) и сверяет результат с этим файлом.

В каталоге **18 профилей × 2 CLI = 36 golden-файлов**. Красных сейчас много — это
общекаталожный долг, не наша задача. Переменная `E2E_INSTALL_REGEN_GOLDEN=1` действует
на **каждый тест, который в этом прогоне доехал до сверки**. Запуск всей сюиты с ней
перепишет все 36 файлов и превратит чужой долг в «зелёный» молча. Отсюда `-k` во всех
командах ниже — он и есть граница.

## Предусловия

- Работать **в том worktree, где лежит финальный контент профиля**, — golden обязан
  ехать одним коммитом/PR с контентом, которому соответствует. Для этой задачи:
  `~/tacticum/tacticum-dev-qa-canon` (ветка `feat/qa-canon-delivery`).
- Docker запущен. Проверить: `docker info` отвечает без ошибки. Тесты поднимают
  сессионный `postgres:16-alpine` (см. `apps/backend/tests/conftest.py`); без docker
  прогон не стартует.
- `uv` в PATH (`which uv` → `/opt/homebrew/bin/uv`).

## Шаг 0 — убедиться, что golden сейчас не тронут

```bash
cd ~/tacticum/tacticum-dev-qa-canon
git status --short apps/backend/tests/e2e_install/golden
```

Вывод должен быть **пустым**. Если нет — разобраться, что уже изменено, и не продолжать:
после регенерации станет непонятно, что чьё.

## Шаг 1 — проверить, что отбор берёт ровно две ячейки

```bash
cd ~/tacticum/tacticum-dev-qa-canon/apps/backend
uv run --extra dev pytest tests/e2e_install --collect-only \
  -k "roles_generic and iva-role-qa"
```

Ожидается **ровно две** строки, других быть не должно:

```
tests/e2e_install/test_install_flow.py::test_install_flow_roles_generic[codex-iva-role-qa]
tests/e2e_install/test_install_flow.py::test_install_flow_roles_generic[claude-code-iva-role-qa]
```

Если собралось больше — **остановиться** и не запускать шаг 2: лишние ячейки перепишут
чужие golden.

## Шаг 2 — регенерация

```bash
cd ~/tacticum/tacticum-dev-qa-canon/apps/backend
E2E_INSTALL_REGEN_GOLDEN=1 uv run --extra dev pytest tests/e2e_install -q \
  -k "roles_generic and iva-role-qa" -p no:randomly
```

- `--extra dev` — ставит pytest и его плагины; работает и на холодном `.venv`.
- `-p no:randomly` — фиксированный порядок, чтобы прогон был воспроизводим.
- Первый прогон дольше остальных: тянется образ postgres.

**Зелёный результат этого прогона ничего не доказывает.** В режиме регенерации оракул
`assert_tree_matches_golden` (`tests/e2e_install/oracles.py:157`) пишет файл и выходит
досрочно, не сверяя. Доказательство — шаг 4.

## Шаг 3 — проверить, что затронуты ровно два файла

```bash
cd ~/tacticum/tacticum-dev-qa-canon
git status --short apps/backend/tests/e2e_install/golden
```

Обязано быть **ровно две строки, обе про `iva-role-qa`**:

```
 M apps/backend/tests/e2e_install/golden/iva-role-qa/claude-code.json
 M apps/backend/tests/e2e_install/golden/iva-role-qa/codex.json
```

Появился хоть один чужой профиль — **откатить именно его и не коммитить**:

```bash
git checkout -- apps/backend/tests/e2e_install/golden/<чужой-профиль>/
```

Посмотреть, что реально изменилось (ключи — пути установки, значения — sha256):

```bash
git diff apps/backend/tests/e2e_install/golden/iva-role-qa/
```

Здоровый диф: у `codex.json` появляются QA-скиллы (`batch-autotest`, `write-autotest`,
`run-tests`, `fix-failed-test`, `jira-issue-autotest`, `craft-stack`) — до этого их там
не было, файл держал 11 ключей с версии 0.2.0. Если ключей стало **меньше** или пропали
пути, которые профиль обязан ставить, — это не устаревший снимок, а дефект доставки:
остановиться и сообщить лиду, не коммитить.

## Шаг 4 — доказать, что зелено в обычном режиме

Переменную НЕ передавать:

```bash
cd ~/tacticum/tacticum-dev-qa-canon/apps/backend
uv run --extra dev pytest tests/e2e_install -q \
  -k "roles_generic and iva-role-qa" -p no:randomly
```

Ожидается `2 passed`. Вот это и есть доказательство.

## Шаг 5 — коммит

Только два файла, явно, без `git add .`:

```bash
cd ~/tacticum/tacticum-dev-qa-canon
git add apps/backend/tests/e2e_install/golden/iva-role-qa/claude-code.json \
        apps/backend/tests/e2e_install/golden/iva-role-qa/codex.json
git commit -m "test(qa): golden iva-role-qa — снимок с финального контента канона"
```

## Ожидаемые грабли

- **`docker info` не отвечает** — прогон не стартует; поднять Docker Desktop и повторить.
- **Собралось больше двух ячеек** — проверить, что `-k` набран точно
  `"roles_generic and iva-role-qa"`; без `roles_generic` подхватятся другие тесты.
- **`xpass strict` на claude-code** — не должно быть: `xfail`-маркеров в
  `test_install_flow.py` больше нет (README сюиты про них устарел, это отдельный мелкий
  долг на текст README, не блокер).
- **Тест падает НЕ на сверке с golden** — значит сломана доставка, а не снимок.
  Регенерацией это не лечится: сообщить лиду с текстом падения. Для справки, на состоянии
  до заморозки обе ячейки падали ровно на `oracles.py:169`
  (`installed tree does not match golden`), а состав в БД против манифестов, уникальность
  ингредиентов, единственность маркера и `bulk == manifest` были зелёные.

## Чего не делать

- Не запускать сюиту с `E2E_INSTALL_REGEN_GOLDEN=1` **без** `-k` — перепишет все 36 golden.
- Не регенерировать из worktree, где лежит не тот контент (например из main): снимок
  зафиксирует контент, который канон заменяет. Устаревший golden виден красным тестом,
  а свежеснятый с брака выглядит легитимным — это хуже.
- Не трогать golden чужих профилей — их краснота отдельный общекаталожный долг.
