---
title: 'Разведка upstream: aqa-agent-toolkit + one-web-kmp против нашего лейна'
type: note
status: current
created: 2026-07-28
tags:
- board
- qa
- upstream
- toolkit
permalink: tacticum/00-board/explore-toolkit-kmp-2026-07-28-1
archived-at: 2026-08-05 15:19
---

# Разведка: aqa-agent-toolkit + one-web-kmp

Заказ lead-qa: карта двух выгруженных Евгением Байрамбековым репозиториев и сверка
с нашим лейном `iva-qa-autotest-base`. Только чтение, правок нет.

**Разобранные источники** (архивы — в `/Users/bubblemac/tacticum-vault/90-Materials/`,
распакованы в сессионный scratchpad; пути ниже — от корня распаковки):

| Обозначение | Путь распаковки | Дата файлов |
|---|---|---|
| `TK` = aqa-agent-toolkit | `…/scratchpad/repos/aqa-agent-toolkit-main` | 2026-07-15 |
| `KMP` = one-web-kmp | `…/scratchpad/repos/one-web-kmp-main` | 2026-07-27 |
| `KIT` = kit (наш снапшот 23.07) | `…/scratchpad/kit-cmp/kit-main` | 2026-07-22 |
| `LANE` = наш лейн | `/Users/bubblemac/tacticum/tacticum-dev/templates/iva-qa-autotest-base` | 2026-07-24 |

Полный префикс scratchpad: `/private/tmp/claude-501/-Users-bubblemac-tacticum-vault/598cba87-f04d-46da-9ae8-a23ed99f697e/scratchpad`.

---

## A. `aqa-agent-toolkit` — что это и как устроено

### A1. Структура и модель установки

117 файлов, три верхних каталога + `README.md` / `INSTALL.md` (406 строк) / `CHANGELOG.md`
(920 строк). Это **template-репозиторий**, не пакет и не плагин: `docs/DESIGN.md:10`
(«Форма дистрибутива → Template-репозиторий»).

Модель трёх слоёв — `TK/README.md:23-34`:

1. **`core/`** — generic-ядро, копируется в проект **как есть**: `core/claude/` (16 скиллов
   + 3 субагента + rules + hooks + evals + settings-шаблон), `core/scripts/` (7 скриптов),
   `core/tasks-schemas/` (схемы `.tasks/`), `core/CLAUDE.template.md`.
2. **`adapters/`** — 9 осей параметризации, у каждой `CONTRACT.md` (markdown-конвенция,
   не исполняемый слой — `TK/docs/DESIGN.md:114-119`) + `reference-<реализация>/`:
   `tms/` (Allure TestOps), `browser-driver/` (playwright-cli + selenoid), `product-api/`
   (IVA ONE REST/IdM), `test-stack/` (pytest+Selenium), `agent-platform/` (Claude Code
   референс + Codex-рецепт).
3. **product-specific** — шаблонизируется пустым при bootstrap.

**Установка** — раскладка по карте `TK/INSTALL.md:126-151` («источник → назначение»)
с подстановкой ~25 плейсхолдеров анкеты (`TK/INSTALL.md:36-58`) и вырезанием/конкретизацией
осевых блоков `[ось]` (~240 пометок в ~37 файлах, `TK/INSTALL.md:164`). Два сценария:
A — чистый репо (`§3`), B — поверх существующего (`§4`).

**Обновления** — не `git merge`: файлы ядра живут в проекте по рабочим путям с
подставленными плейсхолдерами (`TK/docs/DESIGN.md:53-61`). Проект фиксирует hash ядра в
`.claude/toolkit-manifest.md`, обновление = `git diff <hash>..HEAD -- core/` в клоне
тулкита → повторная точечная раскладка. Механизировано скиллом `/toolkit-update` +
`core/scripts/toolkit_update.py` (`TK/docs/DESIGN.md:62-68`).

Правило «core не правится» (`TK/docs/DESIGN.md:69-72`): вся параметризация — только через
плейсхолдеры, осевые блоки и адаптер-файлы; улучшение из проекта портируется обратно
(`docs/PORTING.md`).

### A2. Клиент TestOps — **ключевой ответ**

Расположение: `TK/adapters/tms/reference-allure-testops/testops/` — 5 файлов, 1019 строк.

**Что умеет** — 8 команд, не 5 (`TK/…/testops/__main__.py:116-157`):

| Команда | Что делает | Строка |
|---|---|---|
| `get <id>` | детали TC (overview + сценарий), `--json` | `__main__.py:123` |
| `cfv <id>` | custom fields + Owner | `__main__.py:126` |
| `search "<AQL>"` | поиск по AQL/rql, `--page/--size` | `__main__.py:129` |
| `cases <ISSUE>` | TC, связанные с Jira-issue (`--unautomated` / `--automated`) | `__main__.py:134` |
| `list` | листинг TC проекта | `__main__.py:140` |
| `launch <id\|url>` | не-зелёные результаты ланча, `--all`, `--attachments DIR` | `__main__.py:144` |
| `launches --name` | список/поиск ланчей (новые сверху) | `__main__.py:149` |
| `flake --name --limit` | сводка нестабильности по последним ланчам (вход fix-батча) | `__main__.py:153` |

**Как вызывается.** Пакет — **агентский инструмент**, живёт в `.claude/scripts/testops/`,
не в `tools/` (решение 2026-07-05, `TK/docs/DESIGN.md:17`). Форма вызова — плейсхолдер
`{{TMS_CLI}}` = `PYTHONPATH=.claude/scripts {{PYTHON}} -m testops`
(`TK/INSTALL.md:50`, `TK/adapters/tms/reference-allure-testops/README.md:12-17`).

**Env/секреты.** Литералов в коде нет (`TK/…/testops/client.py:8-10`). Резолв
`resolve_settings()` (`client.py:47-66`): env `TESTOPS_ENDPOINT` / `TESTOPS_TOKEN` /
`TESTOPS_PROJECT_ID` → секция `testops:` из `./secrets.yaml` (читается из cwd = корень
проекта, `client.py:37`) → `TestOpsError` с подсказкой. Авторизация: apitoken →
`POST /api/uaa/oauth/token` (`grant_type=apitoken`) → JWT в `Authorization: Bearer`,
ленивая, с одним авто-повтором при 401 (`client.py:91-129`). Всё read-only: единственный
POST — обмен токена (`client.py:4-6`).

**Самодостаточность.** Внешних зависимостей две: `requests`, `urllib3` (+ `yaml` лениво
внутри `_secrets_section`) — `client.py:12-16`; остальные импорты внутрипакетные
(`.client` / `.launch` / `.testcase`) и stdlib. README референса подтверждает:
`requests`, `pyyaml` — «оба обычно уже есть в окружении тест-стека»
(`TK/adapters/tms/reference-allure-testops/README.md:18`).

**Переносим ли как есть — да, с оговоркой на 3 доменных словаря.**
`TK/…/reference-allure-testops/README.md:26-36` называет точки адаптации:

| Точка | Где | Референс |
|---|---|---|
| тег «решено не автоматизировать» | `testcase.py: WONT_AUTOMATE_TAG` | `wont-automate` |
| пропускаемые вложения ланча | `launch.py: _SKIP_ATTACHMENTS` | `("Видео",)` |
| схема browser-маркера прогона | `launch.py: _BROWSER_MARKER_RE` | `browser_<os>_<browser>` |

Всё остальное копируется байт-в-байт. **Но:** в `KIT` тот же пакет живёт поколением
дальше — эти три словаря вынесены в `kit/tms/scripts/testops/params.py` (новый файл,
читает секцию `[tms]` из `.kit/answers.toml`), а вызов переведён на кроссплатформенный
шим `kit/tms/scripts/testops_cli.py` вместо `PYTHONPATH`-префикса. Детали — §C3.

Контракт `to_plain` (plain-выдача `get`) — интерфейс скиллов-парсеров, менять нельзя:
изменение формата = регресс всех читателей (`TK/…/README.md:38-40`).

### A3. Скиллы и субагенты — сверка с нашими

**Субагенты — совпадают по именам, 3 из 3:** `TK/core/claude/agents/{codebase-analyst,
dom-explorer,code-writer}.md` против `LANE/ingredients/agents/*.md`.
**Содержательно расходятся:** у `TK` dom-explorer — 0 упоминаний canvas/Compose/overlay
(это Selenium/DOM-референс one-web Angular-эры); у нашего — 5 (canvas + accessibility-overlay,
границы честности оверлея, реестр testTag). Наши тела пришли из `KIT/craft/agents/`
(тоже 5 упоминаний, тексты совпадают с точностью до `one-web` → `one-web-kmp`).

**Скиллы. У `TK` — 16, у нас — 9 (+ craft-stack).**

| Наш скилл | Аналог в `TK` | Статус |
|---|---|---|
| `write-autotest` | `core/claude/skills/write-autotest/` | есть; у нас тело из `KIT` (`craft:write`) |
| `batch-autotest` | `core/claude/skills/batch-autotest/` | есть; тело из `KIT` |
| `fix-failed-test` | `core/claude/skills/fix-failed-test/` | есть; тело из `KIT`. У `TK` 13 references, у нас 10 |
| `run-tests` | `core/claude/skills/run-tests/` | есть; **у нас старое тело** (см. §C2) |
| `jira-issue-autotest` | `core/claude/skills/jira-issue-autotest/` | есть; у нас старое тело |
| `prepare-mr-branch` | `core/claude/skills/prepare-mr-branch/` | есть, 3 references — совпадают |
| `retro` | `core/claude/skills/retro/` | есть |
| `playwright-cli` | **нет и не будет** | `TK` намеренно НЕ вендорит: инструмент сам ставит свой скилл (`playwright-cli install --skills`), снапшот форкается — `TK/adapters/browser-driver/reference-playwright-cli/README.md:5-14` |
| `rebuild-autocore` | **нет** | «не портировать as-is, 65% one-web-специфики»; generic-паттерн вынесен в `TK/docs/patterns/vendored-dep-rebuild.md` — `TK/docs/DESIGN.md:13` |
| `craft-stack` (наш общий дом стека) | рассыпан по `adapters/test-stack/reference-pytest-selenium/` + `core/claude/rules/` | другая раскладка |

**Чего у нас нет вовсе (7 скиллов `TK`)** — `TK/docs/skill-ecosystem.md:38-55`:

| Скилл | Роль | Тип |
|---|---|---|
| `update-autotest` | изменившийся TC (update-флаг) → ресинк существующего теста | И |
| `retest-issue` | ключ задачи → прогон уже-автоматизированных связанных тестов + ланч | И |
| `launch-parse` | URL красного ланча → read-only триаж до фикса | И (ось tms) |
| `worktree-land` | ворктри-задача → сведена в агентскую ветку + доставлена MR | Д |
| `refactor-codebase` | долг тест-кода → вычищен под зелёными тестами | АЭ |
| `aut-overview-bump-version` | новая сборка AUT → снимок карты заархивирован | АЭ (ось living-aut-map) |
| `backlog-dashboard` | беклог `.tasks/TODO/` → HTML-снимок | АЭ |
| `toolkit-update` + `feedback-deliver` | связь развёртки с ядром (обновления / фидбэк) | С |

Плюс общий слой `core/claude/skills/_shared/` (6 файлов: `delivery-tail`, `pipeline-run`,
`commit-hygiene`, `ledger-and-deviations`, `jira-candidates`, `subagent-fallback`) — у нас
из него есть только `jira-candidates` и `ledger-and-deviations` (+ наш собственный
`pipeline-gate.md` по ADR-0060, которого в апстриме нет).

Карта делегирования и различитель неоднозначных входов — `TK/docs/skill-ecosystem.md:148-172`
и `§5:229-248`. У нас различителя нет ни в каком виде.

### A4. Как toolkit решает запуск тестов

**Обе модели, разведены по путям** (`TK/core/claude/skills/run-tests/SKILL.md:24`):

- **Путь A** — раннер локально или на **гриде** (Selenoid-ферма win/mac/lin), словарь
  browser-маркеров `browser_win_chrome` … `browser_mac_safari` (`SKILL.md:67-77`), флаги
  `--browser-tag current` / `--local` (`SKILL.md:113-117`), двухфазная параллель с хвостом
  `parallel_unsafe` (`SKILL.md:96-101`).
- **Путь B** — CI-пайплайн → ланч TMS, переменные `TEST_SUITE`/`TEST_K`/`TEST_BROWSER`/
  `TEST_WORKERS`/`TEST_ENV` (`SKILL.md:170-181`), механика в `_shared/pipeline-run.md`.

**Критично для нас:** и словари, и путь B — **осевые блоки, вырезаемые при bootstrap**.
Шапка скилла прямо это фиксирует: «**Путь B целиком — осевой блок [tms]/[test-stack]**:
потребитель без CI/remote … вырезает его; … скилл честно деградирует до пути A»
(`SKILL.md:11-20`), «Словари ниже … — референс one-web …; при bootstrap заменить значения
на словарь проекта, **роли строк и механику разбора сохранить**» (`SKILL.md:34`).

Модель сетевых стендов есть (`{{DEFAULT_ENV}}`, `[dual-line]`-резолв по ветке —
`SKILL.md:79-85`), но она параметр, а не константа.

Selenoid — отдельный референс-адаптер `TK/adapters/browser-driver/reference-selenoid/`
(`README.md:1-20`): нужен, когда разведка/repro требуется **именно в целевом браузере
прогона**, для VNC-наблюдения и для среды, идентичной раннерной. Для одно-браузерного
стека неприменим по определению.

### A5. Зависимость от `.kit` — **нет, toolkit самостоятелен**

Грep по всему дереву `TK` на `\.kit\b`, `answers.toml`, `craft`, `tms:read` — **ноль
совпадений** (единственный хит по паттерну `schema\.md` — это `config-schema.md` адаптера
product-api, `TK/adapters/product-api/CONTRACT.md:23`). Toolkit не знает ни про `.kit/`,
ни про `craft`, ни про модульную раскладку. Зависимость односторонняя и обратная: `kit`
знает про toolkit и объявляет его прародителем (§C3).

### A6. Разрыв поколений по CHANGELOG

Наш лейн собран из среза ≈**2026-07-11/12** (см. §C2, обоснование). Что приехало в `TK`
после этого — по `TK/CHANGELOG.md`, сверху вниз:

| Дата | Что | Строка |
|---|---|---|
| 2026-07-12 | **`/run-tests` переработан целиком**: путь B (CI→ланч), `_shared/pipeline-run.md`, развилка-вопрос вместо автозапуска фикса, копия raw только у красных, пре-чек `--collect-only`, повторы для флак-проверки, вычистка мёртвых маркеров | `411-446` |
| 2026-07-12 | **`/launch-parse`** — новый скилл: read-only сводка ланча без фикс-пайплайна | `382` |
| 2026-07-12 | **`/refactor-codebase`** — санкционированная уборка тест-кодобазы | `350` |
| 2026-07-12 | **`/aut-overview-bump-version`** — бамп живой карты AUT | `314` |
| 2026-07-12 | `dom-explorer mode=adhoc` — произвольная разведка живого UI | `336` |
| 2026-07-12 | ledger-инвариант и гигиена коммита вынесены в `_shared/` | `286` |
| 2026-07-12 | карта экосистемы скиллов + различитель входов в `CLAUDE.md` | `252` |
| 2026-07-13 | **дефолт верификации в `fix-failed-test`: `browser_win_chrome` → без `-m`** (два адаптера ядра противоречили друг другу) | `188-216` |
| 2026-07-13 | `--local` нельзя даже для локальных win/mac grid-хостов | `199-201` |
| 2026-07-14/15 | Codex-слой стал проекцией живого CC (снят двойной ручной синк); гейты манифеста; `_shared/subagent-fallback.md`; Page/Chunk не материализуются внутри тестового модуля | `8-90` |

Более ранние, но уже после части нашего среза: групповой режим `fix-failed-test` и объём
cross-browser по классу правки (2026-07-11, `478-505`), таксономия `.tasks/` (2026-07-11,
`448`).

---

## B. `one-web-kmp` — факты под наши дефекты

37 файлов, весь корень: `config.yaml`, `conftest.py` (371 строка), `pytest.ini`,
`requirements.txt`, `secrets.yaml.example`, `README.md`, `.gitignore`, `tests/`, `tools/`.

### B1. `tools/` — **`tools/testops` НЕТ**

Содержимое `KMP/tools/`: `enums.py`, `test_tags.py` (196 строк, автогенерируемый реестр
локаторов), `generate_test_tags.py`, `i18n/` (`__init__.py`, `strings.py`), `data/`
(4 медиа-сэмпла), `helpers/` (`centrifugo.py`, `chat_setup.py` 619 строк, `media_chat.py`,
`session.py`, `steps.py`). Каталога `testops` нет, модуля с другим именем — тоже нет.

**Жалоба `No module named tools.testops` подтверждается, и причина не в venv.** Пакет там
никогда и не должен лежать: `KMP/secrets.yaml.example:6` прямо говорит — «Секция testops —
читает пакет testops (`.claude/scripts/testops/`; env-override: TESTOPS_ENDPOINT /
TESTOPS_TOKEN / TESTOPS_PROJECT_ID)». То есть репо несёт **тулкитовую** конвенцию: клиент
живёт в агентской зоне `.claude/scripts/`, а не в проектном дереве `tools/`.

Подтверждение первоисточником: `KMP/secrets.yaml.example` — это файл
`TK/adapters/tms/reference-allure-testops/secrets.yaml.example` **дословно** (diff даёт
только добавленные строки 15-23 — секция кредов стенда KMP). Репо one-web-kmp — развёртка
тулкита.

Переклассификация `tools/testops` → `.claude/scripts/testops/` состоялась **2026-07-05**
(`TK/docs/DESIGN.md:17`, `TK/CHANGELOG.md:892`; миграция существующих развёрток —
INSTALL §7.1). Наш лейн ссылается на путь, отменённый за три недели до его сборки.

**Оговорка.** В архиве нет ни `.claude/`, ни `.tasks/`, ни `AUT_OVERVIEW.md` — но
`KMP/.gitignore` ссылается на них как на существующие: `.tasks/**/*.png` (стр. 14-19),
`.tasks/_scratch/` (23), `.tasks/work/*/scratch/` (26), `/PROGRESS.md` (28),
`.claude/worktrees` (29), `.claude/scheduled_tasks.lock` (30), `.claude/settings.local.json`
(31), `CLAUDE.local.md` (32). Значит агентская среда в живом репо есть — выгрузка пришла
без неё (вероятно другая ветка: `KIT/CLAUDE.md:41-42` называет рабочей веткой полигона
`at-creation-main`). **Это стоит запросить отдельно** — см. §C.

### B2. `pytest.ini` — реальные маркеры и опции

`KMP/pytest.ini:10-21` — 11 маркеров:

`wip`, `requiredb`, `require_ldap_user`, `parallel_unsafe`, `qaauto_smoke`, `suite_smoke`,
`suite_regress_simple`, `suite_regress_advanced`, `suite_regress`, `lang_en`,
`multiple_browsers`.

**`browser_*` маркеров нет ни одного.** Фактически проставлены в тестах только два:
`suite_smoke` ×7 и `suite_regress` ×7 (все — в `KMP/tests/test_chat.py`, у 7 тестов с
`@allure.id`). `suite_regress_simple` / `suite_regress_advanced` / `qaauto_smoke` /
`lang_en` / `multiple_browsers` / `wip` / `requiredb` объявлены, но не носятся ни одним
тестом — это «мёртвые маркеры» в терминах `TK/…/run-tests/SKILL.md:61`: запуск по ним даст
`no tests collected`.

`addopts` (`pytest.ini:9`): `-v -p no:cacheprovider --tb=native --showlocals
--strict-markers --alluredir=./allure-raw`. Плюс `xfail_strict = true` (стр. 2).

**Опции (`KMP/conftest.py:366-370`) — ровно три:**
- `--env` (default `kmp-stage`),
- `--lang` (default `consts.LANG_RU`),
- `--headed` (store_true, отладка).

**`--browser-tag` и `--local` не существуют** — pytest ответит
`error: unrecognized arguments`. Прогон в грид/selenoid недоступен: браузер поднимается
внутри `conftest.py:189-194` (`sync_playwright().start()` → `playwright.chromium.launch`),
единственный, headless по умолчанию.

**Что из нашего `run-tests` невалидно** (`LANE/ingredients/skills/run-tests/SKILL.md`):

| Строка скилла | Что предписывает | Реальность KMP |
|---|---|---|
| `:35-45` | таблица из 6 browser-маркеров + alias | маркеров нет; `-m browser_win_ya` → `no tests collected` |
| `:49-56` | резолв `env` по ветке: `at-creation-crc` → `at-test-3-one`, иначе `at-test-1-one` | таких стендов нет — `KeyError` (§B3) |
| `:69` | база команды с `--browser-tag current \| --local` | обе опции не существуют → pytest падает на разборе аргументов |
| `:23-24` | `-m suite_regress_simple` / `suite_regress_advanced` | маркеры объявлены, но пустые |
| `:58-65` | параллель «только lin_chrome, один lin-selenoid» | selenoid нет; xdist есть и настроен (`conftest.py:19-25, 58-65`), ограничение по браузеру бессмысленно |
| `:79` | «autocore без `browser_*` в argv запускает lin_chrome на основном selenoid» | браузер поднимает conftest, не autocore |
| `:185-193` | все 11 примеров с `--env at-test-1-one` | все невалидны |

Валидны из скилла: тест-селекторы (`-k`, путь файла, `-m suite_smoke`/`suite_regress`,
резолв allure id через `Grep '@allure.id("<N>")'` — `@allure.id` реально стоит,
`test_chat.py:26,60,90,125,160,…`), чистка/копия `allure-raw`, подсчёт падений по
`*-result.json`.

### B3. `config.yaml` + `conftest.py` — почему `KeyError: 'at-test-1-one'`

`KMP/config.yaml` объявляет **ровно один стенд** — `kmp-stage` (стр. 7-13):
`web_url: https://stage-one.ivcs.su`, `admin_url: https://stage-id.ivcs.su`,
`browser_url: https://local.ivcs.su:8443`.

Резолв: `conftest.py:112` — `cfg = get_config()[option.env]`. `get_config()` —
`autocore.test.config` (`conftest.py:39`), модульный синглтон, читающий `config.yaml`
проекта. Индексация словаря по имени env напрямую → `--env at-test-1-one` даёт
**`KeyError: 'at-test-1-one'`** буквально на этой строке. Диагноз подтверждён.

Дефолт прокидывается в `sys.argv` до импорта autocore (`conftest.py:9-11`), потому что
autocore резолвит `--env` на момент импорта; выбранный env шарится xdist-воркерам через
`os.environ['ONE_WEB_ENV']`.

**Подтверждаю: запуск локальный, сетевые стенды в нашем понимании неприменимы.**
- Браузер — Playwright chromium в процессе прогона (`conftest.py:189-194`), Selenoid нет:
  `KMP/README.md:4-5` — «Тесты запускаются через `pytest` в headless Chromium (Playwright,
  **без Selenoid**)»; `README.md:74-75` — «Стенд и браузер указывать не нужно: дефолт —
  `kmp-stage`, единственный браузер — Chromium Playwright (**селеноидов и браузерных
  маркеров в KMP-линии нет**)».
- Гибрид, а не чистая локальность: браузер грузит UI с `browser_url` (локальный dev-server
  `local.ivcs.su:8443`, проксирующий API на стенд), REST-сетап ходит **напрямую** в
  `web_url` на stage (`conftest.py:124-127`, `config.yaml:4-6`). При отсутствии
  `browser_url` браузер берёт `web_url`. Итог: сеть нужна (stage-контур), но матрица
  стендов/браузеров — нет.

Секреты: `secrets.yaml` в корне (gitignored, `KMP/.gitignore:34`), секция с именем env
домёрживается в `config[env]` фикстурой `setup` (`conftest.py:90-113`, `_merge_secrets`).

### B4. Структура `tests/` — конвенции

```
tests/
├── __init__.py
├── test_login.py   (66 строк, 4 теста)
├── test_chat.py    (256 строк, 7 тестов)
├── flows/          chat.py (49), login.py (13)  — переиспользуемые сценарные шаги
└── pages/          base_page.py (274), chat.py (730), chat_card.py (537),
                    chat_creation.py (220), address_book.py (33),
                    login.py (37), media_viewer.py (38)
tools/helpers/      chat_setup.py (619), centrifugo.py (228), media_chat.py (206),
                    session.py (49), steps.py (53)   — REST/WS-сетап и ожидания
```

**Разметка теста** (`test_chat.py:26-32`) — канон, в который должен попадать генератор:

```python
@allure.id("54540")
@allure.title("Ответ на сообщение с цитированием")
@allure.tag("34", "Мессенджер")
@allure.feature("Мессенджер")
@pytest.mark.suite_smoke
@pytest.mark.suite_regress
def test_group_chat_reply_to_message(setup_aut, users_factory, teardown):
    """<предусловие текстом>"""
```

Плюс `log.tc('54540')` в теле после сетапа (`test_chat.py:44`) — граница «сетап / собственно
проверка». `@allure.id` привязан к каноническому ТК TestOps (project 37, домен Мессенджер);
`suite_smoke`/`suite_regress` ставятся **только** на тесты с такой привязкой
(`KMP/README.md:70-72`).

**Фикстуры** (`conftest.py`): `setup` (session, конфиг+секреты+`app_url`, стр. 108-128),
`setup_aut` (context+page+page_starter, 299-313), `setup_multiple_auts` (316-325),
`users_factory` (indirect-параметризуемая фабрика эфемерных юзеров с Keycloak-атрибутами,
168-185), `create_user` (131-165), `teardown` (список отложенных вызовов, 328-337),
`browser` (session, 188-194).

**Специфика canvas — главное для генератора** (`tests/pages/base_page.py:1-25`):
UI — один `<canvas>` (Skiko), семантика зеркалится в DOM-узлы accessibility-overlay внутри
open Shadow Root. Отсюда два правила: локация — по узлам overlay (`#test_id_*` из реестра
`tools/test_tags.py` и role/имя; CSS-id и `get_by_role` пробивают open shadow нативно,
**XPath — нет**); клик/ввод — **только координатные** `compose_click`/`compose_fill`
(`base_page.py:61-77`), потому что узлы overlay прозрачны для pointer-событий. Логин —
обычный Keycloak-DOM, водится штатным Playwright (`tests/pages/login.py`).
Границы честности overlay (`base_page.py:14-18`): межфичевая навигация и Compose Popup
ломают overlay безвозвратно (лечит только `goto`), скролл — не ломает.
`wait_for_timeout` разрешён ТОЛЬКО в примитивах `base_page.py` (`base_page.py:20-24`).

Evidence падений — `conftest.py:207-247`: скриншот+видео вложениями в Allure, trace файлом
в `tests/logs/pw-evidence/` (в отчёт не кладётся — десятки МБ), на зелёных удаляется.

### B5. Обязательные для нашего лейна артефакты — есть/нет

| Артефакт | В архиве | Комментарий |
|---|---|---|
| `.kit/` | **нет** | ни каталога, ни упоминаний в коде/доках |
| `.kit/answers.toml` | **нет** | наши 4 codex-скилла и 3 субагента читают параметры `[craft]` оттуда |
| `.kit/schema.md` | **нет** | контракт fallback ролей в наших скиллах ссылается туда |
| `.kit/craft/feature-mapping.md` | **нет** | роутинг теста по фиче в `write`/`batch`/`codebase-analyst` |
| `.tasks/` | **не в архиве**, но существует | `.gitignore:13-26` описывает подкаталоги детально |
| `AUT_OVERVIEW.md` | **не в архиве** | наш `playwright-cli` требует `.tasks/aut-research/AUT_OVERVIEW.md` |
| `autocore` | **есть**, как pip-зависимость | `requirements.txt:17` — `autocore` без версии, ставится с внутреннего PyPI `at-pypi.iva.ru` (`README.md:15,25`). Каталога-исходников в репо нет |
| `.claude/` | **не в архиве** | `.gitignore:29-31` его подразумевает |
| `.gitlab-ci.yml` | **нет** | CI-контура в выгрузке не видно; публикация описана ручным `allurectl upload` (`README.md:101-117`) |

### B6. Секреты и зависимости

**Секреты** — `KMP/secrets.yaml.example`: две части. Секция `testops:` (endpoint/token/
project_id, env-override `TESTOPS_*`) — дословно тулкитовая. Секция `<имя env>:`
(credentials.admin username/password Keycloak realm master, `iva-one-client-secret`) —
KMP-специфика, домёрживается фикстурой `setup`.

**Наш лейн несёт только первую половину.** `LANE/ingredients/config/secrets.yaml.example`
— 17 строк, только `testops:`; секции стенда нет. Модель резолва (env → secrets.yaml →
явная ошибка, apitoken→JWT) описана верно и совпадает с апстримом, но эталоном названа
`tools/testops/client.py` (стр. 5) — файла, которого не существует.
`LANE/ingredients/config/gitignore-secrets.txt` добавляет `secrets.yaml` в `.gitignore` —
в KMP это уже сделано (`.gitignore:34`), ингредиент избыточен, но безвреден.

**Зависимости** (`KMP/requirements.txt`, 17 строк): `allure-pytest==2.13.0`,
`allure-python-commons==2.13.0`, `attrs==22.2.0`, `pluggy==1.0.0`, `pytest`,
`pytest-xdist`, `numpy==2.3.3`, `PyYAML==6.0.2`, `playwright==1.61.0`,
`urllib3==1.26.15`, `paramiko==3.2.0`, `requests`, `websocket-client==1.9.0`,
`psycopg2-binary==2.9.10`, `opencv-python==4.11.0.86`, `faker`, `autocore`.
Python 3.13 (`README.md:14`).

Для клиента TestOps всё уже есть: `requests` и `PyYAML` в списке. **Дополнительных
зависимостей перенос клиента не требует.**

---

## C. Вывод для пересборки

### C1. Что берём как есть / адаптируем / чего нет

**Как есть (нулевая адаптация):**
- **Пакет `testops/`** — 5 файлов, 1019 строк, зависимости уже в `requirements.txt` KMP.
  Три доменных словаря под сверку (`WONT_AUTOMATE_TAG`, `_SKIP_ATTACHMENTS`,
  `_BROWSER_MARKER_RE`); в KMP browser-маркеров нет → `browser_marker()` вернёт `None`,
  это штатная деградация, не поломка. **Класть в `.claude/scripts/testops/`, не в `tools/`.**
- `secrets.yaml.example` — брать KMP-версию целиком (обе секции), она уже сверена с
  апстримом.
- Слой `_shared/` (`delivery-tail`, `pipeline-run`, `commit-hygiene`, `subagent-fallback`) —
  generic, у нас его нет.
- Схемы `.tasks/` (`core/tasks-schemas/`) — таксономия, TODO-шаблон, metrics-README,
  PROGRESS-template.

**Адаптируем под нашу модель лейнов/ролей:**
- **Модель установки несовместима с нашей.** У toolkit — клон + анкета плейсхолдеров +
  ручная раскладка + манифест с hash ядра; у нас — `manifest.yaml` лейна с ingredients и
  `target_path`. Плейсхолдеры `{{…}}` придётся отображать в наши параметры лейна, а
  `.claude/toolkit-manifest.md` — либо принять как второй источник истины, либо заменить.
- **Оси.** Для KMP: `[dual-line]` — вырезать (одна линия), `[vendored-dep]` — вырезать
  (autocore ставится с PyPI, локальной пересборки нет → `rebuild-autocore` и hook не нужны),
  `[test-stack]` — заменить словарь на canvas-стек, `[tms]` — оставить,
  `[browser-driver]` — Playwright без грида, `[living-aut-map]` — под вопросом
  (`AUT_OVERVIEW.md` в выгрузке нет).
- 7 скиллов, которых у нас нет (§A3) — переносить с той же адаптацией.

**Чего нет вовсе:** `update-autotest`, `retest-issue`, `launch-parse`, `worktree-land`,
`refactor-codebase`, `backlog-dashboard`, `aut-overview-bump-version`, `toolkit-update`,
`feedback-deliver`, различитель входов в `CLAUDE.md`, evals-методология, ledger
`.tasks/metrics.jsonl`, петля фидбэка в апстрим.

### C2. Что в нашем лейне заведомо мёртвое для KMP

Отсортировано по тяжести. Кандидаты lead-qa подтверждаются все четыре, плюс найдено ещё.

1. **`run-tests` целиком** (`LANE/ingredients/skills/run-tests/SKILL.md`) — детально в §B2.
   Матрица браузеров, `--browser-tag`, `--local`, `at-test-1-one`/`at-test-3-one`,
   selenoid-параллель, `suite_regress_simple/advanced`. Из 194 строк живыми остаются
   тест-селекторы и работа с `allure-raw`.
   **Самое показательное: наш лейн противоречит сам себе.** Наш же файл
   `LANE/ingredients/craft-stack/pytest-runner.md:26-31` (стек `pytest-playwright-canvas`,
   пришёл из kit) прямо пишет: «Браузерных маркеров (`-m browser_*`), тегов версий
   браузера, grid- и local-режимов в этом стеке **не существует**; "выбор браузера для
   верификации" как шаг умер»; и `:22` — «`--env` не ставим, дефолтный стенд инжектится
   conftest'ом (`kmp-stage`)». То есть внутри одной поставки есть и правильное описание
   стека, и скилл, предписывающий обратное.
2. **`rebuild-autocore`** (`LANE/ingredients/skills/rebuild-autocore/SKILL.md`) —
   11 хардкодов `$HOME/Projects/AT/…` (стр. 6-7, 20, 29-31, 41-42, 48-49, 56-57, 63-64,
   67, 72, 81-82, 88). Предполагает локальный клон `autocore` рядом; в KMP autocore —
   pip-пакет с внутреннего PyPI (`requirements.txt:17`, `README.md:15`). Плюс PostToolUse-хук,
   который лейн не провижнит. В апстриме скилла нет намеренно (`TK/docs/DESIGN.md:13`).
3. **Ссылки на `tools.testops` / `python -m tools.testops`** — 12 мест:
   `LANE/README.md:12,37,55,63,102,103,106,125`, `LANE/manifest.yaml:12,17,42,59,87,90,307,
   308,320,328`, `LANE/ingredients/config/secrets.yaml.example:5`,
   `LANE/ingredients/skills-codex/jira-issue-autotest/SKILL.md:57,62,104`,
   `LANE/ingredients/skills-codex/write-autotest/SKILL.md:134`. Модуля нет, путь отменён
   2026-07-05.
4. **Ссылки на `.kit/*` — 14 мест в телах, которые мы поставляем**, при том что `.kit/`
   в KMP нет: `.kit/answers.toml` — `LANE/README.md:168`,
   `ingredients/agents/{code-writer:10, dom-explorer:12, codebase-analyst:11}.md`,
   `ingredients/agents-codex/*.toml:20-22`,
   `ingredients/skills-codex/{write-autotest:34, fix-failed-test:33, batch-autotest:36}`;
   `.kit/schema.md` — `skills-codex/{write-autotest:42, fix-failed-test:89,
   batch-autotest:43}`; `.kit/craft/feature-mapping.md` — `skills-codex/write-autotest:231`,
   `skills-codex/batch-autotest:82`, `agents/codebase-analyst.md:64`,
   `agents-codex/codebase-analyst.toml:74`.
   **Либо мы поставляем `.kit/` вместе с лейном, либо эти ссылки надо переписать на наши
   параметры.** Сейчас — ни то ни другое.
5. **`tms:read`** — 6 мест (`skills-codex/batch-autotest:39,59,90`,
   `skills-codex/write-autotest:100,134-135`, `skills-codex/fix-failed-test:104`). Это имя
   скилла модуля `kit/tms`, которого у нас в поставке нет: мы взяли тела оркестраторов kit,
   но не взяли модуль, на который они ссылаются.
6. **`playwright-cli`** (`LANE/ingredients/skills/playwright-cli/SKILL.md`) — вендорённый
   снапшот апстрим-скилла с вписанными внутрь проектными конвенциями (стр. 10 — стенды
   `at-test-1-one`/`at-test-3-one` + требование открыть `.tasks/aut-research/AUT_OVERVIEW.md`
   «карта по версиям, последняя v2.0.0/Сборка 2.0.5»; стр. 56 — `playwright-cli open
   https://at-test-1-one.ivcs.su/`). `TK/adapters/browser-driver/reference-playwright-cli/
   README.md:5-14` называет ровно эту практику уроком-антипаттерном референс-инсталляции
   one-web: «апстрим-снапшот с вставками проектных конвенций прямо в тело SKILL.md → форк
   без ресинка». Плюс стенды не те и AUT_OVERVIEW описывает Angular-приложение, а не KMP.
7. **`jira-issue-autotest`** (`LANE/ingredients/skills-codex/jira-issue-autotest/SKILL.md`)
   — резолв стенда по ветке (`:45`), `TEST_BROWSER:browser_lin_chrome` (`:84,89`), пайплайн
   `glab`. CI-контура в выгрузке нет; браузер-переменная бессмысленна.
8. **Мёртвые ссылки на `_selenium-ref/`** — по `LANE/CHANGELOG.md` (0.3.0) 4 штуки вырезаны
   24.07. Но файлы, на которые они вели, в апстриме существуют:
   `TK/core/claude/skills/fix-failed-test/references/{manual-walkthrough,
   cross-browser-verification}.md` и `KIT/craft/stacks/pytest-selenium/{manual-walkthrough,
   cross-browser-verification}.md`. То есть мы вырезали ссылку вместо того, чтобы
   доукомплектовать поставку — для canvas-стека `cross-browser-verification` действительно
   не нужен, а `manual-walkthrough` нужен (в canvas-стеке его нет и в kit — потенциальная
   дыра в `fix-failed-test` Phase 3).

**Обоснование датировки нашего среза (≈2026-07-11/12).** Наш `fix-failed-test` содержит
`references/known-issues.md` (приехал 2026-07-10, `TK/CHANGELOG.md:564`) и
`batch-discovery`/`batch-report` (2026-07-11, `TK/CHANGELOG.md:478`), но наш `run-tests` —
дореформенный: нет пути B, нет развилки-вопроса, нет пре-чека `--collect-only`, нет
повторов, копия `allure-raw` делается безусловно, `/fix-failed-test` вызывается
автоматически. Всё это заменено 2026-07-12 (`TK/CHANGELOG.md:411-446`). Плюс отсутствуют
все четыре скилла, приехавшие 12.07 (`launch-parse`, `refactor-codebase`,
`aut-overview-bump-version`, `backlog-dashboard` — последний 10.07). Итог: старая половина
лейна — срез 11-12 июля, новая (write/batch/fix/craft-stack/субагенты) — из kit 22 июля.
**Разрыв внутри одной поставки — около 11 дней и одна смена архитектуры дистрибуции.**

### C3. `aqa-agent-toolkit` против `kit` — за базу брать **`kit`**

Основание фактическое, не оценочное.

**1. `kit` — прямой наследник toolkit, объявленный самим `kit`.**
`KIT/README.md:118-120`: «Прародители (**старый `aqa-agent-toolkit`** + агент-среда kmp,
ныне отделённая в репо one-web-kmp) поглощаются по петлям; **старый репо после пересадки
последнего потребителя уходит в архив**».
`KIT/CLAUDE.md:45-46`: «Ядра для поглощения: **старый тулкит**
`~/Projects/AT/aqa-agent-toolkit` (ветка `dev`) и агент-среда kmp (репо one-web-kmp);
**при расхождении kmp-версия — кандидат в канон**».

**2. Даты.** Файлы `TK` — 2026-07-15, файлы `KIT` — 2026-07-22. Наш снапшот kit на неделю
свежее выгруженного сегодня toolkit.

**3. Поглощение корпуса скиллов уже завершено — дыр нет.** Все 16 скиллов `TK` имеют
адрес в `KIT`:

| `TK` | `KIT` |
|---|---|
| write / fix / update / batch / jira-issue-autotest | `craft:write` / `craft:fix` / `craft:update` / `craft:batch` / `craft:issue` |
| run-tests / launch-parse / retest-issue | `track:run` / `track:parse` / `track:retest` |
| prepare-mr-branch / worktree-land | `ship:mr` / `ship:land` |
| retro / backlog-dashboard / refactor-codebase | `keep:retro` / `keep:dashboard` / `keep:refactor` |
| aut-overview-bump-version | `atlas:bump` |
| toolkit-update / feedback-deliver | `base:update` / `base:contribute` |
| (нет) | `audit:review`, `dispatch:invoke`, `vendor:rebuild`, `tms:read` — **новые в kit** |

**4. Для нашей задачи (canvas/KMP) разница решающая.**
- `KIT/craft/stacks/` содержит **два** профиля стека: `pytest-selenium` (11 файлов) и
  **`pytest-playwright-canvas`** (10 файлов) — ровно наш стек. В `TK` профиль один,
  `adapters/test-stack/reference-pytest-selenium/`, canvas-варианта нет.
- `TK/core/claude/agents/dom-explorer.md` — **0** упоминаний canvas/Compose/overlay.
  `KIT/craft/agents/dom-explorer.md` — 5 (canvas + accessibility-overlay, границы честности,
  реестр testTag, «Angular-снимок к canvas-стеку неприменим»).
- `KIT/track/skills/run/SKILL.md:22` явно описывает вырождение browser-резолва:
  «**Одно-браузерный стек** (референс: pytest-playwright-canvas — один headless chromium)
  вырождает browser-резолв: маркеров/тегов браузера нет, browser-строки таблиц не
  действуют». В `TK/core/claude/skills/run-tests/SKILL.md` такой ветки нет — там браузерная
  матрица снимается только ручным вырезанием осевого блока при bootstrap.
- Клиент TestOps в `KIT` — то же ядро, но параметризованное: три доменных словаря вынесены
  в новый `KIT/tms/scripts/testops/params.py` (читает секцию `[tms]` `.kit/answers.toml`),
  вызов через кроссплатформенный шим `KIT/tms/scripts/testops_cli.py` вместо
  `PYTHONPATH=`-префикса. Диффы `client.py`/`launch.py`/`testcase.py` — только докстринги +
  замена констант на `load_params()`; логика идентична.

**5. Оговорка — что `kit` НЕ решает, а `toolkit` решает.**
`KIT` раздаёт ядро через marketplace плагинов («Ядро живёт вне репо потребителя — в кэше
платформы; обновления привозит платформа» — `KIT/README.md:20-21`). Наш профиль обязан
работать **вместо** kit, то есть класть файлы в репо. Такой раскладки в `kit` нет — она
есть в `TK/INSTALL.md:126-151` (таблица «источник → назначение»). Из toolkit имеет смысл
взять именно **процедуру и карту раскладки, анкету плейсхолдеров и правило «core не
правится»**, а содержимое — из kit.

**Итог:** база — `kit` (содержание, стек-профиль canvas, актуальные тела скиллов, клиент
TestOps с `params.py`); из `aqa-agent-toolkit` — механика развёртки в репо (INSTALL §1-§2,
§7) и то, что в kit пока не поглощено. `TK` как источник тел скиллов брать нельзя: он
на неделю старше нашего же снапшота kit и не знает про canvas.

### C4. Что запросить у Евгения дополнительно

- **`.claude/` и `.tasks/` репо one-web-kmp** — выгрузка пришла без них, хотя
  `KMP/.gitignore:13-32` их подразумевает, а `KIT/CLAUDE.md:46` называет kmp-версию
  агент-среды «кандидатом в канон при расхождении». Это и есть то поколение, которое нам
  нужно, — и его в выгрузке нет. Вероятно нужна ветка `at-creation-main`
  (`KIT/CLAUDE.md:41-42`), а не дефолтная.
- **`.kit/answers.toml` и `.kit/schema.md`** живого потребителя — чтобы понять, чем
  заполняются параметры, на которые ссылаются наши тела скиллов.
- **Есть ли CI-контур** (`.gitlab-ci.yml`): от этого зависит, вырезать ли путь B в
  `run-tests` или сохранять.