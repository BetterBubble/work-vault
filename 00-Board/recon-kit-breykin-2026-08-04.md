---
title: recon-kit-breykin-2026-08-04 — агентский кит Брейкина vs наш лейн
type: recon
status: draft
created: 2026-08-04
repo: mobile (ИВА, объединённый мобильный репозиторий)
project: tacticum-dev / профили / QA
role: explorer
for: lead-qa
tags:
- board
- recon
- qa
- agent-kit
- iva
- mobile
permalink: tacticum/00-board/recon-kit-breykin-2026-08-04
---

# recon-kit-breykin-2026-08-04

**Задача:** разведать агентский кит Никиты Брейкина в объединённом мобильном репозитории и сопоставить с нашим лейном.
**Статус:** draft, канон не пишу — повышает ответственная роль.

**Что смотрел:**
- ЕГО: `/private/tmp/claude-501/-Users-bubblemac-tacticum-vault/e3ce1ccf-6fa4-4521-8b94-29607561fd0c/scratchpad/mobile-new/mobile-main/` — `.agents/`, `.codex/`, `AGENTS.md`, плюс `tests/`, `tools/`, `README.md` для сверки фактов.
- НАШЕ: `/Users/bubblemac/tacticum/tacticum-dev-qa-combined/templates/iva-pytest-appium-autotest-base/` и `/Users/bubblemac/tacticum/tacticum-dev-qa-combined/templates/tacticum-autotest-core/`.

---

## КРАТКИЙ ВЫВОД

1. **Кит у него больше нашего по объёму и шире по охвату процесса.** 9798 строк `.md`/`.toml` в `.agents/`+`.codex/` против 7445 в нашем `tacticum-autotest-core/ingredients` + 722 в стеке `iva-pytest-appium-autotest-base/ingredients`. **14 скиллов против наших 8.**
2. **Разошлись не по качеству, а по оси.** Он унёс наверх **процесс** (7 скиллов, которых у нас в этих двух лейнах нет вообще: `worktree-land`, `update-autotest`, `retest-issue`, `launch-parse`, `refactor-codebase`, `backlog-dashboard`, `jira-issue-autotest`) и **рабочую директорию с реестрами** (`.agents/state/` — карта на 44 строки, 7 постоянных реестров). Мы ушли вглубь **стека**: наш `ingredients/stack/` — 722 строки против его 286 в `craft-stack/`, и написан он под мобильную специфику, а его — под фактические файлы проекта.
3. **Его канон стека — тонкий и «отсылочный», наш — толстый и «объясняющий».** Его `pytest-runner.md` = 23 строки и заканчивается «Источник истины при расхождении — фактические `conftest.py`, `pytest.ini` и `README.md`». Наш `runner.md` = 106 строк и объясняет, ПОЧЕМУ `--env` обязателен (`KeyError` в сессионной фикстуре) и почему нельзя брать первое значение из `config.yaml` (там первым стоит `prod`). **Ни того, ни другого предупреждения у него нет.**
4. **Наш стековый канон устарел по фактам продукта.** У него в коде `su.ivcs.aura` (12+ вхождений), сборки Android 1.3.5 (60) / iOS 1.3.5 (10), плоская раскладка `locators/{Android,iOS}/`. Наш канон описывает `su.ivcs.ucim` + `su.ivcs.kmp` и каталог `tests/pages/locators/Android/KMP/`, **которого в объединённом репо нет**.
5. **У него внутри кита три расхождения, которые он, похоже, не видит** (детали в п.5–6): орфанный мобильный `.codex/rules/page-objects.md`, орфанный `.codex/agents-instructions/dom-explorer.md`, и web-таксономия падений `failure-taxonomy.md` (DOM_CHANGED / WEBDRIVER_QUIRK / браузеры) — на мобильном стеке.
6. **Он использует наш MCP.** `.codex/config.toml`: `mcp.tacticum.dev/mcp` (`tacticum-mcp`) и `mcp.tacticum.ru/iva-atlassian/mcp` (`iva_mcp`), оба через `TACTICUM_TOKEN`.

---

## 1. Полный список ЕГО скиллов (14)

Все — `.agents/skills/<имя>/SKILL.md`. Числа — строк в SKILL.md / суммарно с `references/`.

| # | Скилл | Строк | Суть одной строкой |
|---|---|---|---|
| 1 | `backlog-dashboard` | 32 / 32 | Обёртка над `.codex/scripts/backlog_dashboard.py` (544 строки Python): собирает `.agents/state/TODO/` в самодостаточный HTML-дашборд с drill-down и поиском |
| 2 | `batch-autotest` | 105 / 581 | Оркестратор автоматизации набора 5+ TC из Allure TestOps; один TC делегирует в `$write-autotest` |
| 3 | `craft-stack` | 32 / 286 | Не вызывается пользователем — «общий дом канона стека `pytest-appium-mobile`», карта из 7 файлов для субагентов и оркестраторов |
| 4 | `fix-failed-test` | 279 / **2307** | Чинит упавший тест (локаторы / методы page-chunk / импорты); batch 1..N не-зелёных; 13 файлов `references/` — самый тяжёлый скилл кита |
| 5 | `jira-issue-autotest` | 84 / 84 | Сквозной флоу «задача трекера → автотесты → связанный ланч TestOps»; новые TC → batch, изменившиеся → update, доставка → prepare-mr-branch + `glab` |
| 6 | `launch-parse` | 111 / 111 | **Read-only** сводка ланча TestOps по URL: разбивка по kind, окна деградации стенда, сверка ревизий ланч↔локальный код, отметки по known-issues; не чинит |
| 7 | `prepare-mr-branch` | 417 / 881 | Cherry-pick проектных коммитов грязной агентской ветки в чистую MR-ветку (агентные файлы дропаются), 3 блока текста для MR, push/создание MR через `glab` |
| 8 | `refactor-codebase` | 150 / 150 | Санкционированная уборка ТЕСТ-кодобазы (dead-code, дубли, читаемость) с аппрувом на каждую правку; не агент-инфра |
| 9 | `retest-issue` | 137 / 137 | Прогон УЖЕ-автоматизированных кейсов задачи → привязанный TMS-ланч, БЕЗ write/update/MR |
| 10 | `retro` | 245 / 245 | Ретро и антиэнтропийная чистка агент-инфры: метрики, память, беклог, размеры контекст-файлов, дрейф агентов/веток, висячие ворктри |
| 11 | `run-tests` | 215 / 215 | Универсальная точка запуска: локальный/грид-прогон раннером или CI-пайплайн с ланчем TestOps |
| 12 | `update-autotest` | 106 / 244 | Ресинк теста под ИЗМЕНИВШИЙСЯ TC; детект по тегу `at-updating` (ставит человек); роутинг metadata/minor/major/removed |
| 13 | `worktree-land` | 310 / 330 | Сводит ворктри-задачу: squash WIP → rebase на базу → `merge --ff-only` → `$prepare-mr-branch` из диапазона ровно этой задачи |
| 14 | `write-autotest` | 247 / 664 | Пишет НОВЫЙ тест по CSV/URL TC/описанию **сразу под Android и iOS, строго в порядке Android → iOS** |

Плюс `_shared/` (384 строки, 6 файлов): `commit-hygiene.md` (35), `delivery-tail.md` (95), `jira-candidates.md` (81), `ledger-and-deviations.md` (52), `pipeline-run.md` (71), `subagent-fallback.md` (50).

---

## 2. Три группы

### (а) Есть у обоих — 7
`batch-autotest`, `craft-stack`, `fix-failed-test`, `prepare-mr-branch`, `retro`, `run-tests`, `write-autotest`.

### (б) Только у него — 7
`backlog-dashboard`, `jira-issue-autotest`, `launch-parse`, `refactor-codebase`, `retest-issue`, `update-autotest`, `worktree-land`.

**Уточнение по `jira-issue-autotest`:** в указанных мне двух лейнах его нет, но **в репозитории он есть** — `templates/iva-qa-autotest-base/ingredients/skills/jira-issue-autotest/SKILL.md` и `templates/iva-qa-delivery-base/ingredients/skills/jira-issue-autotest/SKILL.md` (+ codex-варианты). Т.е. по-настоящему уникальны у него **6**.

**Остальные 6 не существуют нигде в `tacticum-dev-qa-combined`** — проверено `grep -rl` по всему `templates/`: `retest-issue`, `refactor-codebase`, `worktree-land`, `backlog-dashboard` — 0 вхождений; `launch-parse` и `update-autotest` — упоминаются только как перекрёстные ссылки внутри `iva-qa-autotest-base`, самих скиллов нет.

### (в) Только у нас — 1
`rebuild-autocore` (199 строк, есть и в `tacticum-autotest-core`, и в `iva-qa-autotest-base`). У него аналога нет — вместо этого правило в `AGENTS.md:158`: правки autocore только в `/Users/nbreykin/work/autocore`, установленная копия read-only.

### Также только у него (не скиллы, но части кита)
- `.codex/evals/EVALS.md` — мини-эвалы агент-инфры. **Ни в одном нашем лейне слова `EVALS` нет.**
- `_shared/commit-hygiene.md`, `_shared/pipeline-run.md`, `_shared/delivery-tail.md` — **0 вхождений** в нашем репо.
- `.codex/scripts/` — 1625 строк исполняемого кода: `testops/` CLI (818 строк, 4 модуля), `backlog_dashboard.py` (544), `worktree_bootstrap.sh` (63).

---

## 3. Насколько разошлось содержание общих 7

| Скилл | Его строк | Наш core (claude-тело) | Вердикт |
|---|---|---|---|
| `fix-failed-test` | 2307 (13 refs) | 1560 (9 refs) | **разъехались архитектурно** — см. ниже |
| `prepare-mr-branch` | 881 (3 refs) | 971 (3 refs) | сопоставимы, наш чуть богаче |
| `write-autotest` | 664 (6 refs) | 772 (6 refs) | одинаковый набор refs, наш чуть богаче |
| `batch-autotest` | 581 (3 refs) | 664 (3 refs) | сопоставимы |
| `run-tests` | 215 | 381 | **наш богаче почти вдвое** |
| `retro` | 245 | 119 | **его богаче вдвое** |
| `craft-stack` | 286 (5 канон-файлов) | 113 + 722 в стеке лейна | **наш богаче втрое** по канону, его — конкретнее |

Конкретика:

**`fix-failed-test` — разъехались принципиально по границе «скилл vs стек».** У него `failure-taxonomy.md` (363) и `fix-playbooks.md` (311) лежат **внутри скилла**, в `references/`. У нас те же два файла вынесены **в стековый слой**: `iva-pytest-appium-autotest-base/ingredients/stack/failure-taxonomy.md` (50) и `fix-playbooks.md` (90). Плюс у него два ref-файла, которых у нас нет вовсе: `cross-browser-verification.md` и `manual-walkthrough.md`. Остальные 9 refs совпадают по именам один в один.

**Содержательно таксономия падений — это разные документы, а не версии одного.** Его 10 классов: `DOM_CHANGED`, `UX_FLOW_CHANGED`, `API_CONTRACT_CHANGED`, `FLAKY_TIMING` (+ подтип «race до операции»), `IMPORT_OR_REGISTRY`, `PRODUCT_BUG`, `WEBDRIVER_QUIRK`, `STAND_INCIDENT`, deferred (`NON_REPRODUCIBLE_FLAKY`, `AUTOMATION_ONLY`), `UNKNOWN`. Дискриминатор дословно: *«Один браузер ломается, другие проходят? → скорее WEBDRIVER_QUIRK»*, и все примеры помечены «референс one-web». **Это веб-таксономия целиком, мобильной адаптации нет.** Наши 9 классов написаны под устройства: `ENV_CONFIG`, `DEVICE_UNAVAILABLE`, `APP_NOT_STARTED`, `LOCATOR_STALE`, `LANG_MISMATCH`, `RESOLUTION_MISMATCH`, `TIMING`, `WEBVIEW_CONTEXT`, `PRODUCT_BUG`. Пересечение — только `PRODUCT_BUG` и `TIMING`/`FLAKY_TIMING`. **У него нет ни одного класса про устройство; у нас нет `STAND_INCIDENT`, deferred-категорий, `API_CONTRACT_CHANGED` и `IMPORT_OR_REGISTRY`.**

**`run-tests` — наш богаче (381 против 215).** Его description упоминает браузеры («прогони test_chat.py на сафари», «браузер(ы)») — то есть тоже не переписан под мобильный стек. Наш описывает архивирование `./allure-raw-<метка>` с удалением копий старше трёх последних.

**`retro` — его богаче вдвое (245 против 119) и другой по составу.** У него в скоуп входит «дрейф агентов/веток, **висячие ворктри**» и бюджетный триггер `AGENTS.md>15KB`; у нас — `CLAUDE.md > 15 KB`. Ключевое расхождение: наш `retro` (строки 52, 117) прямо пишет **«петли беклога в поставке нет»** и запрещает её заводить («Отсутствие — не находка ретро»). Его `retro` беклог **ведёт**, и ревью беклога — штатная фаза.

**`write-autotest` — его версия уже мобильная, наша нет.** Его description: *«сразу для Android и iOS, строго в порядке Android → iOS»*. Наш: *«раннер и форма теста — из канона стека»*, без платформенной оси. Набор `references/` идентичен (у нас `feature-mapping.template.md`, у него `feature-mapping.md` — суффикс `.template` наша шаблонизация).

**`prepare-mr-branch` — одинаковые по сути.** Он дропает `.agents/`, `.codex/`, `.agents/state/`, `.gitignore`, `AGENTS.md`; мы — `.agent-kit/`, `.claude/`, `.agents/`, `.codex/`, `.tasks/`, `.playwright-cli/`. Наш дополнительно несёт Phase 7 «post-MR цикл после вливания MR» — у него в description этого нет.

**`craft-stack` — см. п.4.**

---

## 4. Попарное сравнение канона стека

Его дом: `.agents/skills/craft-stack/` — **286 строк**, 5 канон-файлов + SKILL.md.
Наш дом: `iva-pytest-appium-autotest-base/ingredients/stack/` — **722 строки**, 7 файлов + `rules/`.

| Его файл | Строк | Наш файл | Строк |
|---|---|---|---|
| `recon.md` | 125 | `recon.md` | 95 |
| `pytest-runner.md` | **23** | `runner.md` | **106** |
| `test-templates.md` | 36 | `test-templates.md` | 86 |
| `batch-conventions.md` | 29 | `batch-conventions.md` | 66 |
| `raw-results.md` | 41 | `allure-raw-parser.md` | 52 |
| — | — | `failure-taxonomy.md` | 50 |
| — | — | `fix-playbooks.md` | 90 |
| ссылка на `.agents/rules/{page-objects,tests}.md` | 222+100 | `rules/page-objects.md`, `rules/tests.md` | 80+97 |

### `pytest-runner.md` (его, 23) ↔ `runner.md` (наш, 106)

**Переименование не косметическое — файлы почти не пересекаются.**

Что знает ОН и не знает наш канон:
- **`PYTEST_DISABLE_PLUGIN_AUTOLOAD=1` + `-p allure_pytest.plugin`** в базовой команде — у нас этого нет вообще. Это реальная механика запуска, которую он выяснил на живом проекте.
- **`allure-raw/` создаётся настройкой `pytest.ini`, отдельный `--alluredir` не нужен.** Наш `batch-conventions.md` наоборот велит писать `--alluredir=./allure-raw-<метка>` в команду.
- **Гейты синтаксиса:** `python -m py_compile <files>` и `python -m pytest --collect-only -q <selector>`.
- **«Платформенный маркер обязан совпадать с `--device-os`»** — явное правило, у нас его нет.
- **«Перед прогоном очищай только артефакты выбранного запуска; чужие сессии не останавливай.»**

Что знает НАШ и не знает он:
- **Почему `--env` обязателен**: опция объявлена без `default`, потребляется в **сессионной** фикстуре → `get_config()[None]` → `KeyError` → падает **весь прогон**, а не тест.
- **Предупреждение про `prod`:** *«`prod` стоит в списке первым. Никакой логики "взять первое" или "взять дефолтное" здесь быть не должно: первое значение — боевое окружение заказчика.»* **У него этой защиты нет ни в одном файле кита.**
- **Таблица всех 9 опций с дефолтами** (`--ver`=Version2, `--lang`=MC.LANG_RU, `--local`, `--from-pipeline`, `--no-exim-reset`, `--no-db-restore`).
- **Полный список маркеров** и явный запрет переносить `browser_*` / `--browser-tag` с selenium-стека.
- **Секция «Что ещё не выяснено — не выдавать за канон».**

Оба файла кончаются взаимоисключающими декларациями об источнике правды: его — *«Источник истины при расхождении — фактические `conftest.py`, `pytest.ini` и `README.md`»*; наш — *«Всё продуктовое ниже — референс инсталляций-источников… при установке подставляются значения своего проекта»*.

### `recon.md` ↔ `recon.md`

**Разъехались принципиально — это два разных документа под одним именем.**

ОН знает и мы нет:
- **Полный Appium preflight из 7 пунктов**: `/status` endpoint, доступность драйвера (UiAutomator2/XCUITest), `adb devices` / `xcrun simctl list devices`, установлено ли приложение, доступен ли backend, поднятие сессии через штатный `prepare_driver()`, фиксация platform/driver/device/orientation/contexts/current activity.
- **Порядок разрешения конфигурации** (явные параметры задания → `conftest.py` / `config.yaml` / `mobile_env.yaml` / `tools/aut/` / `tests/pages/` / `tools/i18n/`).
- **Модель поверхности `NATIVE_APP` / `WEBVIEW_*` / системные поверхности** (permissions, клавиатура, share sheet, file picker, SSO — «могут принадлежать другому package и отсутствовать в дереве AUT»).
- **Критерии приёмки локатора — 6 пунктов** (найден в правильном context, уникален, видим, повторно находится после возврата, не зависит от данных/времени/локали, подтверждён наблюдаемым последствием).
- **Формула фиксации шага:** `экран → context → стратегия → селектор → действие → наблюдаемое следствие`.
- **Гигиена секретов:** *«Не помещай сюда секреты, реальные пароли, OTP или токены»*, *«Не копируй credentials из `config.yaml` в `.agents/state/`, логи, команды или отчёты»*, *«Не записывай session ID в долгоживущие артефакты»*.
- **Запрет автопереключения платформы:** *«Если задача не задаёт Android/iOS… — верни блокер»*.
- **Явные пути артефактов:** `mode=write` → `.agents/state/work/tc-<id>/locators.md`; `mode=fix` → `<batch_dir>/fix-<test>-locators.md`.
- **Правила жестов/ожиданий:** прокрутка с фиксированным контейнером и максимумом попыток, запрет `sleep()` как синхронизации, «не объявляй элемент отсутствующим, пока не проверены context, прокрутка, клавиатура, системная поверхность и loading overlay».

МЫ знаем и он нет:
- **Почему `playwright-cli` неприменим** и что готового инструмента разведки не существует ни у нас, ни в апстриме.
- **Таблица «что даёт драйвер»** (`page_source`, `find_element(s)`, `get_screenshot_as_png`, `current_activity`).
- **Compose/KMP-специфика**: вырожденное дерево, `bounds` как долг с обязательной пометкой.
- **«Одно устройство — одна сессия»** и следствия (нельзя параллелить, эмулятор ≠ устройство, ферма добавляет задержку).
- **Известные ограничения референса** (вкладки Mail/Calendar роняли сборку на `emulator-5554`).

### `test-templates.md` (36 ↔ 86)

Его — **готовый рабочий скелет** с `pages.get_page(platform, 'example')(driver, lang=lang)`, `teardown.append(lambda: driver.quit())`, обоими маркерами `@pytest.mark.android` + `@pytest.mark.ios` на одной функции, и правилом *«Если Android и iOS используют разные классы, выбор реализации остаётся в `pages.get_page`, а не в условии `if device_os` в тестовой функции»* — у нас этого правила нет.

Наш — **объясняющий**: таблица «что даёт харнесс», 6 правил-запретов (маркер платформы обязателен, ни одного литерала UI-текста, драйвер только в page-объектах, данные через REST, креды из конфига, одна сессия — одно устройство) и таблица «чего в мобильном тесте быть не должно» с указанием, откуда тянут (selenium/canvas-канон).

Наш открыто дисклеймит: *«шаблоны ниже выведены из харнесса… а не скопированы из тех тестов»* — **этот дисклеймер теперь неверен**, у него в репо есть настоящий page-слой (22 `.py` в `tests/pages`).

### `batch-conventions.md` (29 ↔ 66)

ОН: чеклист из 6 пунктов на TC (Allure id, путь+имя функции, маркеры, затронутые файлы, команды финальной проверки обеих платформ, число fix-итераций для `metrics.jsonl`); минимальные гейты (`py_compile` + targeted collection + живой прогон на Android И iOS); **секция «Стиль функции» про REST-обёртку продукта** (одна функция = один endpoint, новые функции `autocore` только в исходном репо) — у нас такой секции нет.

МЫ: устройство как дефицитный ресурс, разбор пачки **по классу падения, а не по тесту** (упало всё → окружение; группа на одном экране → адресация, чинится один page-объект; одиночные → тайминги; одинаково на всех устройствах → дефект продукта), решение по Phase-2 один раз на группу, оговорка «живых пачечных прогонов на этом стеке не было».

### `raw-results.md` (41) ↔ `allure-raw-parser.md` (52)

ОН знает и мы нет:
- Список полей: `start`/`stop` для хронологии, `attachments` + файлы по `source`, `fullName`, `parameters`.
- **Правило `XPASS`:** *«строгий XPASS приходит как failed»*.
- **Target-platform extraction** — приоритет источников (`parameters` → labels → environment attachment), нормализация в `Android`/`iOS`, **и явное «если источники конфликтуют — результат неоднозначен, платформу нужно передать явно; имя каталога является только последней подсказкой»**.
- **Local-mode detection:** `device_host=local:<port>` **не доказывает** значение флага `--local`; нет доказательства → режим `unknown`.
- **Сканирование log-attachment:** искать первое исключение шага, Appium/WebDriver ошибки, момент потери сессии; **«teardown-ошибку после первичного падения не выдавай за корень проблемы»**.
- Ланч TestOps читается обёрткой `.codex/scripts/testops/`, «а не этим форматом».

МЫ знаем и он нет:
- **Скриншоты именуются `Device-{udid}-{сообщение}`** и «падение на одном устройстве vs на всех» как дискриминатор.
- Публикация через `allurectl` + `TESTOPS_*` (у него — свой Python CLI, см. п.6).
- «Каталог сырых результатов не коммитить» с отсылкой на прецедент web-стека.

---

## 5. Субагенты: `.codex/agents/*.toml` + `agents-instructions/*.md` против наших трёх

**Роли совпадают полностью — те же три имени:** `codebase-analyst`, `dom-explorer`, `code-writer`. **Механика подачи промпта разная и у него непоследовательная.**

| Роль | Его `.toml` | Его `agents-instructions/*.md` | Наш `.toml` | Наш `agents/*.md` |
|---|---|---|---|---|
| `code-writer` | **8 строк** — указатель | 262 | 181 (тело verbatim) | 170 |
| `codebase-analyst` | **5 строк** — указатель | 318 | 187 (тело verbatim) | 176 |
| `dom-explorer` | **1095 строк — тело инлайн** | 414 | 180 (тело verbatim) | 169 |

Его паттерн для двух ролей — тонкий указатель: *«Твоя роль определена в `.codex/agents-instructions/<роль>.md` — прочитай этот файл целиком ПЕРВЫМ действием и строго следуй ему»*. Наш паттерн — тело в `developer_instructions` verbatim.

### Находка: `dom-explorer` у него раздвоился

`dom-explorer.toml` содержит **1088 строк инлайн-тела** («# Субагент: разведка мобильного UI через Appium и подбор адресации»), а `agents-instructions/dom-explorer.md` (414 строк) — **другой, старый веб-текст**: frontmatter `tools: Bash, Read, Write, Glob, Grep`, секции про `AUT_OVERVIEW.md`, Angular/transloco, **23 вхождения «one-web»**. Diff даёт 9 расходящихся блоков, первый же — полная замена шапки.

**На `dom-explorer.md` никто не ссылается**, кроме двух его же файлов, которые велят его читать в аварийном режиме: `_shared/subagent-fallback.md:25` — *«Прочитать целиком `.codex/agents-instructions/<роль>.md` перед любым локальным действием»* — и `retro/SKILL.md`. То есть **fallback-сценарий для `dom-explorer` подсунет старый веб-промпт вместо мобильного**. Не проверено, знает ли он об этом.

### Расхождения в метаданных

| | Он | Мы |
|---|---|---|
| model | `gpt-5.6-sol` (у `codebase-analyst` модель не задана вовсе) | `gpt-5.4` у всех трёх |
| `sandbox_mode` | **отсутствует у всех трёх** | `read-only` / `workspace-write` — явно у каждого |
| `[agents] max_threads` | `4` в `.codex/config.toml` | не проверено |

Наш `code-writer.toml` содержит явный FLAG в комментарии: *«Claude `tools:` restriction has no exact Codex-native equivalent… Mapped best-effort onto sandbox_mode. FLAG: tool-scoping fidelity gap»*. **У него скоупинга тулов нет вообще** — ни `tools:`, ни `sandbox_mode`.

### Что у него есть сверх ролей
`_shared/subagent-fallback.md` (50 строк): если платформа не смогла запустить роль **до начала её работы**, основной агент читает md роли, локально сохраняет её границы и формат выхода, фиксирует отступление. *«Частично начавшую работу роль так не дублировать.»* — **у нас такого механизма нет.**

---

## 6. Его правила `.agents/rules/` и `.codex/rules/` — чего нет у нас

### Состав
| Файл | Строк | Что это |
|---|---|---|
| `.agents/rules/page-objects.md` | 222 | frontmatter `paths: tests/pages/**/*.py`; **23 вхождения «референс one-web»** — веб-версия |
| `.agents/rules/tests.md` | 100 | frontmatter `paths: tests/test_*.py`; **22 вхождения «one-web»** — веб-версия |
| `.codex/rules/page-objects.md` | 238 | **мобильная версия**, без frontmatter |
| `.codex/rules/default.rules` | 7 | **allowlist команд — у нас аналога нет** |

### Находка: правила слоя раздвоены, и грузится веб-версия

`AGENTS.md:83` говорит: *«Правила реализации находятся в `.agents/rules/`»*. Автозагрузка у него — по frontmatter `paths:`, и он есть **только у `.agents/rules/*.md`**. То есть агент грузит **веб-производные** правила (`PBase` из `autocore.base.web.base_page`, `Version2Locators`, `multiple_browsers`, `parallel_unsafe`+xdist, `users_factory`, `lin_chrome`).

Мобильная версия `.codex/rules/page-objects.md` **не упоминается ни в одном файле кита** (`grep -rn "\.codex/rules"` → 0 вхождений). Не проверено, подхватывает ли Codex `.codex/rules/` автоматически по конвенции клиента.

### `.codex/rules/default.rules` — allowlist, которого у нас нет
```
prefix_rule(pattern=["git", "status"], decision="allow")
prefix_rule(pattern=["git", "diff"], decision="allow")
prefix_rule(pattern=["git", "log"], decision="allow")
prefix_rule(pattern=["./.venv/bin/python", "-m", "pytest"], decision="allow")
prefix_rule(pattern=["python3", "-m", "py_compile"], decision="allow")
prefix_rule(pattern=["bash", "-n"], decision="allow")
prefix_rule(pattern=["PYTHONPATH=.codex/scripts", "./.venv/bin/python", "-m", "testops"], decision="allow")
```

### Содержательно: его `.codex/rules/page-objects.md` (238) vs наш `stack/rules/page-objects.md` (80)

**Его богаче в разы там, где речь о фактическом коде** — он писал по живому page-слою и ловит конкретные баги в нём:
- Паттерн `_PBase<Name>` + `PAndroid<Name>` + `PIOS<Name>`, **запрет ветки `if device_os` внутри общего метода** (примеры: `go_back()`, `_enter_password`, `accept_alerts`, `login_adfs`, `decline_update_app`).
- **«Экран может существовать только на одной платформе»** — пример `chat_about.py` с `#TODO: for future` вместо пустышки.
- **Обязательный `@staticmethod` на локаторах** с указанием конкретного недосмотра: `L_I18N_BTN_ADFS_LOGIN` в `locators/iOS/login.py`.
- **Запрет второго недостижимого `return`** (`L_I18N_TEXTFIELD_SERVER` в `locators/iOS/pick_server.py`) и **задвоенных имён методов** (три пары в `locators/iOS/chats.py`: `L_I18N_CHAT_START_AUDIO_CALL_BUTTON`, `L_I18N_CHAT_ANSWER_CALL_BUTTON`, `L_I18N_CHAT_JOIN_TO_CALL_BUTTON`).
- **Унификация сигнатуры** — первый параметр всегда `lang`, динамический вторым со значением `''`; имя метода и порядок параметров обязаны совпадать между Android и iOS.
- **Приоритеты селекторов по платформам:** Android 4 уровня (`id` → `xpath` с `@resource-id` → `content-desc` → чистый текст); iOS 6 уровней (`accessibility id` с конвенцией `Zero.<name>` → `name` → `xpath` по `XCUIElementType*` → `-ios predicate string` → `-ios class chain` → `class name`).
- **Дословный дубль XPath кнопки «назад»** в трёх файлах iOS (`schedule_meeting.py`, `edit_schedule_meeting.py`, `rooms.py`) — с указанием проверять все три.
- Ассерты: `assert_presence`/`assert_absense` пишут allure-шаг сами, не оборачивать в `log.step`; сообщение — констатация, не долженствование.
- Логирование: `log.step(<инфинитив>, where='IVA Mobile Клиент')`.

**Но этот файл устарел относительно его же объединённого репо.** Он ссылается на `tests/pages/contacts.py`, `chats.py`, `chat_about.py`, `chunks/bottom_menu.py`, `browser/mobile_browser.py`, `locators/iOS/{login,pick_server,meeting,meeting_login,schedule_meeting,edit_schedule_meeting,rooms}.py`. Фактически в объединённом репо: `tests/pages/` = `auth.py`, `base.py`, `chats.py`, `passcode.py`; `chunks/` и `browser/` содержат **только `__init__.py`**; `locators/iOS/` = `auth.py`, `chats.py`, `common.py`, `passcode.py`. **Большинства упомянутых файлов не существует.**

**Наш богаче там, где речь о технологии отрисовки:** таблица «нативная vs Compose/KMP» с цитатой их README, три хрупкости (язык UI, `bounds` как долг, WebView-логин), «платформа и поколение — разные оси». **Этого у него в правилах нет** — только в `craft-stack/recon.md` частично.

---

## 7. `.agents/state/` — рабочая директория и реестры

**Это его изобретение по раскладке, но не по идее.** У нас та же концепция живёт под именем **`.tasks/`** (см. `tacticum-autotest-core/ingredients/skills/retro/SKILL.md`: `.tasks/metrics.jsonl`, `.tasks/batch-*/`, `.tasks/retro/`, `.tasks/jira-candidates/`, `.tasks/work/`). Diff `_shared/jira-candidates.md` показывает прямое переименование: у него `.agents/state/jira-candidates/`, у нас `.tasks/jira-candidates/` — **тексты в остальном родственные**.

### Карта его `.agents/state/README.md` (44 строки) — 12 классов элементов

| Элемент | Класс | Жизненный цикл | Есть ли у нас |
|---|---|---|---|
| `README.md` | сама карта | постоянный | **нет** — карты раскладки у нас нет |
| `TODO/` (`INDEX.md` + карточки `NNN-*.md`) | беклог проекта | постоянный | **нет и намеренно нет** |
| `metrics.jsonl` + `metrics-README.md` | append-only ledger метрик | постоянный | есть (`.tasks/metrics.jsonl`) |
| `known-issues.md` | диагностированные падения | постоянный | есть |
| `jira-candidates/` | кандидаты продуктовых багов, отработанные → `archive/` | постоянный | есть |
| `codebase-refactoring/` | `health-snapshots.jsonl` + `smell-inventory.md` | постоянный | **нет** |
| `tc-review/` (`TC-{id}.md`) | замечания владельцам ТК | до передачи | есть как ref в write-autotest |
| `aut-research/` | живая карта AUT + скриншоты + drift-отчёты | постоянный, версия по сборке | есть упоминание, файла нет |
| `retro/` | отчёты `$retro` | удаляемые | есть |
| `work/` | рабочая зона, каталог на задачу | до конца задачи | есть |
| `worktree-<task>/` | per-task состояние ворктри (`base`, `PROGRESS.md`), gitignored | до `$worktree-land` | **нет** |
| `_scratch/<скилл>/` | эфемерный scratch, gitignored | сносится без жалости | **нет** |

**Ключевая норма, которой у нас нет — инвариант корня:** *«В корне `.agents/state/` не появляется ничего вне таблицы ниже… Любая новая работа — каталог в `work/`. Новый постоянный элемент корня добавляется отдельным изменением агентной инфраструктуры: сначала карточка в проектном беклоге, затем обновление этой карты и MR в `main`.»* И: *«новый инструмент не изобретает себе места»*.

Именование в `work/`: `batch-{YYYYMMDD-HHMMSS}/`, `tc-{id}/` (`input.md`, `analysis.md`, `locators.md`), `update-{id}/` (`tc.md`, `diff.md`), `tms-research/` (PROGRESS кампании), `refactor-<дата>/`, `<тикет>/`, `adhoc-<тема>/`.

### Фактическое наполнение реестров — почти пустое, кит свежий

| Файл | Размер | Содержимое |
|---|---|---|
| `TODO/INDEX.md` | 16 строк / 1337 б | **пустая таблица**, «Последний $retro: —» |
| `TODO/_TEMPLATE.md` | 16 строк | шаблон карточки, поле «Источник» |
| `metrics.jsonl` | **1 строка** | `{"date":"2026-08-04","tc_id":468,"skill":"fix-failed-test","wall_minutes":18,"fix_iterations":2,"dom_explorer_calls":2,"user_questions":0,"tc_truth_minutes":0,"tc_review":"issues","result":"deferred","browsers":[],"platform":"codex-desktop"}` |
| `metrics-README.md` | 63 строки | формат ledger отдельным файлом |
| `known-issues.md` | 4 строки | **пустая таблица** (Тест / Категория / Сигнатура / Issue / Диагноз / Обновлено) |
| `codebase-refactoring/health-snapshots.jsonl` | 1 строка | `{"date":"2026-08-01","smell_class":"obsolete_naming","removed":{"semantic_occurrences":183,"tracked_paths":14},...}` |
| `codebase-refactoring/smell-inventory.md` | 15 строк | одна закрытая запись 2026-08-01 «устаревшее экспериментальное именование», открытых нет |
| `tc-review/TC-468.md` | 13 строк | одно замечание к TC-468 «Авторизация пользователя», вердикт «годен с замечаниями», owner Vitaly |
| `aut-research/AUT_OVERVIEW.md` | 100 строк | живая карта, «Область проверки: только flow авторизации TC-468» |
| `jira-candidates/` | **каталога не существует** | заявлен в карте и в AGENTS.md, но не создан |

**Обратите внимание на `metrics.jsonl`:** поле `browsers: []` — схема ledger не адаптирована под мобильный стек (нет устройства/платформы Android-iOS; `platform: "codex-desktop"` — это про клиента-агента).

### `aut-research/AUT_OVERVIEW.md` — самый ценный артефакт

Шапка (снимок на 2026-08-04):
```
- Платформа: Android и iOS
- Сборка: Android 1.3.5 (60); iOS 1.3.5 (10)
- Application id / Bundle id: su.ivcs.aura
- Устройство: Android Emulator, Android 17; iPhone 17 Simulator, iOS 26.3
- Окружение: at-develop3
- Канал наблюдения: Appium 3.6.0, UiAutomator2 8.2.2 / XCUITest
- Область проверки: только flow авторизации TC-468
```
Флоу Android: `Выбор сервера [NATIVE_APP] → IVA ID [native accessibility nodes WebView] → Защита входа [NATIVE_APP] → Создание PIN → Подтверждение PIN → Разрешение уведомлений [system permission] → Чаты`.

Наблюдения, которых нет ни в одном нашем файле:
- *«После успешного OIDC login первым появляется экран "Защита входа в приложение", а не экран создания PIN.»*
- *«Setup PIN: после четырёх цифр активная "Далее" требует click. Verify PIN: после четвёртой цифры переход выполняется автоматически; второй click "Далее" не требуется.»*
- *«Наличие `WEBVIEW_su.ivcs.aura` в списке contexts после OIDC **не означает**, что protection/PIN находятся в WebView: текущий и правильный context — `NATIVE_APP`.»*

**Аналога `AUT_OVERVIEW.md` у нас в лейне нет** — только упоминания в промптах субагентов.

### Факты продукта: наш стековый канон устарел

| Факт | У него в коде | В нашем каноне |
|---|---|---|
| app id / bundle id | `su.ivcs.aura` (`tools/aut/__init__.py:31,57,59`, `tools/utils/__init__.py:9`, `README.md:133,281`) | `su.ivcs.ucim` (нативный) + `su.ivcs.kmp` (KMP) |
| launch activity | `su.ivcs.messenger.MainActivity` | не задан |
| раскладка локаторов | плоская: `locators/Android/{auth,bottom_menu,calls,chats,common,contacts,more,passcode}.py`, `locators/iOS/{auth,chats,common,passcode}.py` | `locators/Android/`, `locators/iOS/`, **`locators/Android/KMP/`** |
| репозитории | **один объединённый** | «две репы» (`one-mobile-ios`, `one-mobile-android`) |
| состояние тестов | **23 `.py` в `tests/`, 22 в `tests/pages/`**, есть `base.py`, реестры | *«репозитории — заготовки… в KMP-репе один тестовый файл»* |

**Каталога `Android/KMP/` в объединённом репо нет**, но `tests/pages/locators/Android/README.md` сохранился и по-прежнему заявляет `Application id: su.ivcs.kmp`, `Observed build: 1.2.10` — то есть это остаток от старой репы. Его `smell-inventory` за 2026-08-01 фиксирует переименование: 183 семантических вхождения устаревшего префикса, 14 tracked-путей, *«Исключение: фактический Android application id сохранён без изменений»* — при этом `locators/Android/common.py:12` всё ещё возвращает `('id', 'su.ivcs.kmp:id/action_bar_root')`.

**Что из нашего канона это подтверждает, а не ломает:** формулировки про Compose и WebView-логин дословно взяты из `locators/Android/README.md` и там по-прежнему написаны (*«The app is rendered mostly through Compose… The login form is inside `android.webkit.WebView`… `username`, `password`, `kc-login`, `kc-page-title`»*), а `locators/Android/auth.py:71` реально содержит `//android.widget.Button[@resource-id="kc-login"...]`. Но AUT_OVERVIEW от 2026-08-04 добавляет поправку: **после OIDC контекст `NATIVE_APP`, не WebView.**

---

## 8. Явные утверждения об источнике правды и о том, что он отвергает

### Отвергает

| Где | Дословно |
|---|---|
| `AGENTS.md:14` | **«Внешний `aqa-agent-toolkit` не является источником и для работы не используется.»** |
| `AGENTS.md:83–84` | «…**внешняя документация для установки агентной среды не требуется**.» |
| `AGENTS.md:104` | «**Все используемые проектом skills хранятся в текущем репозитории.**» |
| `AGENTS.md:126–127` | «Чтение TC из TMS — **только через CLI-обёртку** `PYTHONPATH=.codex/scripts ./.venv/bin/python -m testops` из текущего репозитория. **Сырой curl не дёргать.**» |
| `AGENTS.md:158` | «`autocore`: установленную копию в `.venv/.../autocore` не редактировать; правки только в `/Users/nbreykin/work/autocore`.» |
| `AGENTS.md:159` | «Исходники продукта не подключены; если появятся — читать только read-only.» |
| `AGENTS.md:146` | «**Никаких `Co-Authored-By: Codex ...`** в сообщениях коммитов.» |
| `craft-stack/raw-results.md` | «Ланч TestOps читается проектной обёрткой `.codex/scripts/testops/`, **а не этим форматом**.» |

### Утверждает как источник правды

| Где | Дословно |
|---|---|
| `AGENTS.md:8` | **«Источник тестового кода, агентов и правил — этот репозиторий, основная ветка `main`.»** |
| `AGENTS.md:76` | «Агенты и конвенции изменяются **прямо в** `.agents/`, `.codex/` и `AGENTS.md`.» |
| `craft-stack/pytest-runner.md:23` | **«Источник истины при расхождении — фактические `conftest.py`, `pytest.ini` и `README.md`.»** |
| `craft-stack/SKILL.md` | «Дом стека — `.agents/skills/craft-stack/`; **он общий для поддерживаемых агентных клиентов**. Файлы лежат прямо в этом каталоге и **изменяются вместе с текущим репозиторием**.» |
| `AGENTS.md:59–62` | «**одно знание живёт в одном слое, остальные ссылаются**: конвенция кода → `.agents/rules/`; операционная конвенция проекта → этот файл; знание о продукте/стенде/инфре → память; процедура → `.agents/skills/`; факт об UI → карта AUT; состояние батча → `PROGRESS*.md`; отложенная задача → `.agents/state/TODO/`.» |
| `batch-autotest/SKILL.md:78` | «`PROGRESS.md` в `.agents/state/work/tms-research/` — **единственный источник правды** по состоянию батча.» |
| `EVALS.md` | «Источник правды по самим конвенциям — `.agents/rules/*.md` и `.md` субъектов; здесь — только эталонные кейсы и процедура.» |
| `AGENTS.md:163–165` | «Персистентная память проекта — файлы в `.codex/memory/`. На старте сессии прочитай `.codex/memory/MEMORY.md`.» (фактически: **3 строки, «Пока устойчивых записей нет»**) |
| `AGENTS.md:15` | «Стенд локальных прогонов по умолчанию — **`at-develop3`**.» |

### Границы, которые он себе поставил
- `AGENTS.md:131–137` — **write-actions:** «Читать и выполнять явно заказанные шаги — можно; **писать во внешние системы по своей инициативе — нельзя**: комментарии/labels/merge в MR, retry/play пайплайнов, мутации трекера/TMS, внешние push.»
- `AGENTS.md:116–118` — **«Под осознанно-ручные вершины (release-решения человека: удаление теста, снятие update-флага Allure TestOps, промоушен билда/линии, ревью и мерж MR) скиллы не заводим: флаг / TODO, исполняет человек.»**
- `AGENTS.md:10–13` — агентская инфра и тестовый код **не смешиваются**: для `.agents/`, `.codex/`, `AGENTS.md` и `.agents/state/` — отдельный коммит и отдельный MR.
- `AGENTS.md:120–122` — **мини-эвалы:** «смысловая правка `.md` субагента/skill'а → до коммита прогони профильный эвал».

### Про MCP — он на нашем контуре
`.codex/config.toml`:
```toml
[agents]
max_threads = 4
[mcp_servers.tacticum-mcp]
url = "https://mcp.tacticum.dev/mcp"
bearer_token_env_var = "TACTICUM_TOKEN"
[mcp_servers.iva_mcp]
url = "https://mcp.tacticum.ru/iva-atlassian/mcp"
bearer_token_env_var = "TACTICUM_TOKEN"
[mcp_servers.confluence]
command = "uvx"
args = ["mcp-atlassian", "--read-only"]
```
Верхняя половина файла (`sandbox_mode`, `approval_policy`, `[features] hooks/multi_agent/browser_use`, `network_access`) — **закомментирована**.

---

## Риски и открытые вопросы

1. **Наш стековый канон надо пересобрать по объединённому репо.** `su.ivcs.ucim`/`su.ivcs.kmp` → `su.ivcs.aura`; «две репы» → одна; `Android/KMP/` → плоская раскладка; дисклеймер «репозитории — заготовки» больше не соответствует (22 файла page-слоя).
2. **`--env`/`prod` защита есть только у нас, поведение с пустыми опциями устройства не проверено ни у него, ни у нас.** Наш `runner.md` это честно фиксирует, его — нет.
3. **Его `failure-taxonomy.md` и `run-tests` — веб-производные без мобильной адаптации.** Если брать его кит как есть, разбор мобильных падений пойдёт по классам DOM_CHANGED/WEBDRIVER_QUIRK.
4. **Три орфана в его ките:** мобильные `.codex/rules/page-objects.md` (никто не ссылается), устаревший `.codex/agents-instructions/dom-explorer.md` (используется только fallback-путём), `.agents/rules/*.md` веб-версии (грузятся автоматически по `paths:`).
5. **У него нет `sandbox_mode`/скоупинга тулов у субагентов** — у нас есть, включая явный `read-only` для `codebase-analyst`.
6. **Реестры почти пустые** (1 строка метрик, 1 TC-review, 0 known-issues, нет `jira-candidates/`) — кит собран, но боевого пробега за ним ещё почти нет. Первая метрика датирована 2026-08-04, исход `deferred`.

## Не проверено
- Подхватывает ли Codex `.codex/rules/*.md` автоматически (без frontmatter `paths:`) — механику клиента не смотрел.
- `max_threads`/параллельность в нашем codex-слое — не сверял.
- Содержимое `.codex/scripts/testops/*` построчно (818 строк) — читал только размеры и факт наличия команд.
- История git его репо — распакованный архив, `.git` отсутствует, датировки взял из содержимого файлов.
- Есть ли у него `.claude/`-слой — `grep "\.claude"` по всему киту даёт 0 вхождений, то есть кит codex-only; но проверял только текстом.

## Примечание о доставке
Заметка положена файлом напрямую: `mcp__basic-memory__write_note` вернул ошибку «Cloud routing requested but no credentials found. Run 'bm cloud api-key save <key>' or 'bm cloud login' first.»
