---
title: recon-mobile-repo-delta-2026-08-04
type: note
permalink: tacticum/00-board/recon-mobile-repo-delta-2026-08-04
status: draft
date: '2026-08-04'
project: tacticum-dev / QA / стек pytest-appium
tags:
- board
- qa
- recon
- mobile
- appium
- canon
- delta
---

# Слияние мобильных реп: дельта к канону стека `pytest-appium`

**Задача:** заказчик слил `one-mobile-ios` + `one-mobile-android` в одну репу. Наш канон
(`templates/iva-pytest-appium-autotest-base/ingredients/stack/`, 9 файлов) написан по двум старым.
Нужна точная дельта.

## Источники (всё проверено в коде, не пересказ)

- **Новая репа:** `/private/tmp/claude-501/-Users-bubblemac-tacticum-vault/e3ce1ccf-6fa4-4521-8b94-29607561fd0c/scratchpad/mobile-new/mobile-main`
  (origin — `git@git.hi-tech.org:ivaqa/mobile.git`, `README.md:46`).
- **Старые две:** ssh на `teststand` **недоступен** — у explorer нет тула `ssh-manager`, а host
  `teststand` не резолвится обычным ssh. Взял равноценный первоисточник: выгрузки
  `~/tacticum-vault/90-Materials/one-mobile-ios-main.zip` и `one-mobile-android-main.zip` (29.07),
  распакованы в `.../scratchpad/old/`. Это те же архивы, по которым писался канон.
- **Наш канон:** `/Users/bubblemac/tacticum/tacticum-dev-qa-combined/templates/iva-pytest-appium-autotest-base/ingredients/stack/`

## Что вообще произошло при «слиянии» — главный факт

**Это не объединение двух реп. Это KMP-репа (`one-mobile-android`), переименованная и дополненная
iOS-локаторами. Нативное поколение выброшено целиком.**

`diff -rq old/one-mobile-android-main new/mobile-main` даёт всего 12 отличий по коду. При этом из
`one-mobile-ios` **не перенесено ничего**: исчезли 6 тест-файлов (`test_chats`, `test_contacts`,
`test_links`, `test_login`, `test_meetings`, `test_rooms`), ~18 page-объектов (`login.py`, `home.py`,
`meetings.py`, `rooms.py`, `contacts.py`, `schedule_meeting.py`, `chat_about.py`,
`view_settings_list.py`, …) и оба нативных пакета локаторов
(`locators/Android/*.py` нативные, `locators/iOS/*.py` нативные, `chunks/bottom_menu.py`,
`browser/mobile_browser.py`).

Переименования: `tests/pages/kmp_auth.py` → `auth.py`, `kmp_chats.py` → `chats.py`,
`kmp_passcode.py` → `passcode.py`; `locators/Android/KMP/` → `locators/Android/`;
классы `LKmpAuth`/`LKmpCommon`/… → `LAuth`/`LCommon`/…; i18n-ключи `kmp_sign_in_title` →
`sign_in_title` и т.д. (`tools/i18n/strings.py`, ~49 ключей).

Новое, чего не было ни в одной старой репе: `tests/pages/locators/iOS/` (4 файла — `common`, `auth`,
`chats`, `passcode`), кросс-платформенный реестр `get_page(platform, page_name)`, `.agents/`,
`.codex/`, `AGENTS.md`, `secrets.yaml.example`, переписанный `README.md` (282 строки).

## ГЛАВНОЕ: канон против фактов

| Что говорит наш канон | Что в новой репе | Вердикт |
|---|---|---|
| `runner.md:3-4` «две поверхности — нативный клиент (`one-mobile-ios`, `su.ivcs.ucim`) и KMP-клиент (`one-mobile-android`, `su.ivcs.kmp`)» | Одна репа `ivaqa/mobile`, один идентификатор `su.ivcs.aura` на обе платформы (`tools/aut/__init__.py:31` appPackage, `:57`/`:59` bundleId; `tools/utils/__init__.py:9` `android_package`). Две поверхности теперь = Android и iOS **одного** клиента | **устарело** |
| `runner.md:11-34` «`--env` обязателен, дефолта нет, падает весь прогон» | `conftest.py:169` `parser.addoption('--env', action='store', help='Environment name')` — без `default`; потребляется в `conftest.py:44` внутри `@pytest.fixture(scope='session') def setup` (`:39-52`). **`conftest.py` байт-в-байт совпадает со старым android** | **верно** |
| `runner.md:38-49` «список окружений: `prod · at-develop1 · at-develop3 · at-develop5 · at-develop6 · at-develop7`, `prod` первый, стенд спрашивать» | `config.yaml:1,10,42,76,110,144` — список тот же, `prod` первый. **Но** у команды есть договорный дефолт: `AGENTS.md:33` «Стенд: опция pytest `--env`, дефолт развёртки `at-develop3`», `README.md:80` то же | **верно, но неполно** — «спроси у проекта» уже имеет ответ в репе |
| `runner.md:51-70` таблица опций, 10 addoption, три опции устройства без дефолтов | `grep -c 'parser.addoption' = 10`; `--device-host/-os/-name` — `conftest.py:174-176`, без `default`. Совпадает построчно | **верно** |
| `runner.md:74` «`pytest.ini` **обеих реп** идентичны» + таблица 11 маркеров (`:77-81`) | `pytest.ini` новой репы **байт-в-байт** = старая android = старая ios. 11 маркеров, `pytest.ini:9-19`, все 11 в нашей таблице есть. Формулировка «обеих реп» устарела, содержание верно | **верно** (формулировка устарела) |
| `runner.md:89-96` команда одного теста: `<раннер> -k <name> --env … --tb=native --showlocals` | **На фактическом окружении не запустится.** `README.md:139-144`: под Python 3.13 обязателен `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1`, и Allure-плагин подключается явно `-p allure_pytest.plugin`. Без этого — `Empty suite` и traceback из `trio` (`README.md:266-269`). То же в их `craft-stack/pytest-runner.md:9-12` | **устарело / опасно** |
| `runner.md:101-102` «чем снимать дерево — инструмент мобильной разведки не выбран»; `recon.md:92` «готового инструмента разведки не существует — ни у нас, ни в апстриме» | В репе есть свой канон разведки `.agents/skills/craft-stack/recon.md` (125 строк: preflight, модель поверхности, формат доказанного локатора) и субагент `.codex/agents-instructions/dom-explorer.md`. Каналы названы: Appium 3.6.0, UiAutomator2 8.2.2, XCUITest 12.1.3 (`README.md:36-41`) | **опровергнуто** |
| `runner.md:104` «где берутся устройства — неизвестно» | `mobile_env.yaml`: 3 Android (`R92X539MQZK`, `R92X539NBNZ`, `emulator-0000`) и 3 iOS устройства, хост `10.6.0.219`, порты `15555-15557` / `25557-25559`, у каждого свои `device_port`/`system_port`. Локальный режим — `--device-host local:15555` (Android) / `local:25555` (iOS), `README.md:92-99` | **опровергнуто (ответ есть)** |
| `runner.md:106` «поведение при пустых опциях устройства не проверено» | Проверок на пустоту в `conftest.py`/`tools/aut` по-прежнему нет | **верно** |
| `recon.md:42-43` «логин у продукта внутри `WebView` на KMP-Android, работают веб-локаторы `username`, `password`, `kc-login`» | Для **Android** — верно и сохранилось: `tests/pages/locators/Android/auth.py:53` (`@resource-id="kc-page-title"`), `:57` `username`, `:61` `password`, `:71` `kc-login`. Для **iOS** — иначе: `locators/iOS/auth.py:37-50` адресует нативные `XCUIElementTypeTextField`/`SecureTextField`; `.agents/state/aut-research/AUT_OVERVIEW.md:70` прямо: на iOS 26.3 `WEBVIEW_*` может указывать на hidden/stale document, а видимая ASWebAuthenticationSession публикуется в `NATIVE_APP`, и управлять надо через неё | **верно для Android, неполно в целом** |
| `recon.md:54-56` «нативный клиент: дерево нормальное, есть `resource-id` (Android) / `accessibility id` (iOS)» | Нативного поколения в репе **нет вовсе**. Новый iOS-пакет написан НЕ на accessibility id: `locators/iOS/common.py:13-26` — всё через `AppiumBy.IOS_PREDICATE` по `type` + видимому тексту из i18n; `ACCESSIBILITY_ID` встречается **один раз** (`locators/iOS/auth.py:19`, литерал `'https://'`). `AUT_OVERVIEW.md:60`: «Host вводится в единственный `XCUIElementTypeTextView`; в пустом состоянии у него нет identifier/name/label» | **опровергнуто** |
| `recon.md:57-66` + `rules/page-objects.md:11-14,28-34` «Compose/KMP: вырожденное дерево, порядок id → content-desc → текст через i18n → bounds» | Утверждение **сохранилось**: `tests/pages/locators/Android/README.md:10-17` — тот же текст, снято только слово «KMP» («rendered mostly through Compose»). Bounds-локаторы живы (`locators/Android/auth.py:25`, `bottom_menu.py:9`). Подтверждено живой разведкой: `AUT_OVERVIEW.md:29` «Protection/PIN — native Compose без resource-id/content-desc на целевых элементах» | **верно по существу** |
| `recon.md:84` и `rules/page-objects.md:17` ссылка на `tests/pages/locators/Android/KMP/README.md` | Каталог `KMP/` удалён. Файл теперь `tests/pages/locators/Android/README.md` | **битая ссылка** |
| `recon.md:85-87` «вкладки Mail и Calendar роняли сборку на `emulator-5554`» | Текст всё ещё в `locators/Android/README.md:23-26`, но он датирован сборкой **1.2.10** (`:6`). Живая карта — сборка **Android 1.3.5 (60) / iOS 1.3.5 (10)** (`AUT_OVERVIEW.md:7`), и её раздел ограничений (`:88-94`) про Mail/Calendar молчит | **неполно** — ограничение не подтверждено на актуальной сборке |
| `rules/page-objects.md:57-61` три каталога локаторов: `iOS/`, `Android/`, `Android/KMP/` | Фактически **два**: `tests/pages/locators/Android/` и `tests/pages/locators/iOS/`. `Android/KMP/` удалён | **устарело** |
| `rules/page-objects.md:63-64` «платформа и поколение — разные оси, локаторы не взаимозаменяемы» | Ось поколения исчезла: остались только платформы. Деление внутри платформы отсутствует | **устарело** |
| `rules/page-objects.md:35-36` «для нативного клиента — обычный порядок: `accessibility id` (iOS) / `id` (Android), затем предикаты и XPath по типам» | Неприменимо (нативного поколения нет). Плюс в самой репе лежат **два конфликтующих** канона приоритетов, и наш не совпадает ни с одним: `.agents/skills/craft-stack/recon.md:62-77` (Android: a11y id → resource-id → Compose `testTag` → UIAutomator → i18n-текст → XPath → координаты; iOS: a11y identifier → label/value → class chain/predicate → XPath) против `.codex/rules/page-objects.md:144-168` (Android: `id` → xpath+resource-id → content-desc → текст; iOS: accessibility id по конвенции `Zero.<name>` → `name` → xpath → predicate → class chain → class name) | **устарело + конфликт в самой репе** |
| `rules/page-objects.md:66-76`, `test-templates.md:5-9`, `rules/tests.md:20-23` «репозитории — заготовки, кроме тестов от другого продукта там ничего нет, конвенции page-слоя не переносить, согласовать с командой» | Тест по-прежнему **один** (`tests/test_one_login.py`), но page-слой стал настоящим и конвенционным: реестр `tests/pages/__init__.py:6-20` `get_page(platform, page_name)` с ключами `auth`/`chats`/`passcode` на **обе** платформы; паттерн общая логика + платформенный подкласс (`chats.py:6-24` `_PBaseChats` → `PAndroidChats(_PBaseChats, LAndroidChats)` / `PIOSChats(…)`; `auth.py:16,58`; `passcode.py:12`); `log.step(..., where='IVA One Mobile')`; ассерты через `assert_presence`/`assert_fn` базового `_PBase`. Конвенции у команды **уже написаны** — `.agents/rules/page-objects.md`, `.codex/rules/page-objects.md`, `.agents/skills/craft-stack/test-templates.md` | **устарело** |
| `rules/page-objects.md:78-80` «на iOS + Compose репозитория не существует вовсе; открытый вопрос к команде» | iOS живёт в той же репе, тот же `su.ivcs.aura`, локаторы написаны, живая разведка iOS проведена 03–04.08 (`AUT_OVERVIEW.md:45-86,98-100`: iPhone 17 Simulator, iOS 26.3, сборка 1.3.5 (10)) | **опровергнуто** |
| `rules/tests.md:36-40`, `test-templates.md:35-37` пример с **одним** маркером платформы | Живой тест несёт **оба**: `test_one_login.py:19-20` `@pytest.mark.android` + `@pytest.mark.ios`. Это норма проекта: `README.md:225-233` «Новый тест пишется сразу для Android и iOS», `craft-stack/batch-conventions.md:3-4` | **неполно** |
| `rules/tests.md:46-54` «связь с TMS держит `log.tc`; дублировать в имени функции — конвенция проекта» | Теперь **и то, и другое**: `test_one_login.py:10` `@allure.id('468')` + `:27` `log.tc('468')`, плюс `@allure.title/label/tag/story/feature` (`:11-15`). `.agents/rules/tests.md:16-28` требует полный набор Allure-метаданных как обязательный | **неполно** |
| `test-templates.md:26-47` скелет: `from tests.pages import <page-объекты>`, прямой импорт классов | Прямой импорт был в **старой** репе (`from tests.pages import PAndroidKmpAuth, …`) и специально заменён: `test_one_login.py:6` `from tests import pages`, `:20` `platform = setup_aut['setup']['cmd']['device_os']`, `:31` `pages.get_page(platform, 'auth')(driver, lang=lang)`. Их шаблон — `craft-stack/test-templates.md:6-33`, с явным запретом ветвить `if device_os` в теле теста (`:30-33`) | **устарело** |
| `test-templates.md:26-47` в скелете нет `teardown.append(lambda: driver.quit())` | В живом тесте есть — `test_one_login.py:29`. Словами у нас сказано (`rules/tests.md:64-65`), в скелете — нет | **неполно** |
| `rules/tests.md:60-72`, `test-templates.md:13-22` «две фикстуры, обе обязательны: `setup_aut`, `teardown`» | Фикстур пять: `run_around_tests` (autouse, `conftest.py:25`), `version_reporter` (`:33`), `setup` (`:39`), `setup_aut` (`:76`), **`setup_multiple_auts` (`:82-98`)** и helper `prepare_driver` (`:54-74`). `setup_multiple_auts` поднимает N драйверов по `mobile_env.yaml` через `get_mobile_env()` (`:22`) — наш канон о нём не знает вовсе | **неполно** |
| `rules/tests.md:94` «никаких фиксированных `sleep`» | Как инвариант тестов — верно. Но в базовом слое проекта `sleep` есть: `tests/pages/base.py:58` `time.sleep(1)` внутри `wait_absence`. Утверждение «на этом стеке sleep'ов нет» было бы ложным | **верно как правило, неполно как факт** |
| `failure-taxonomy.md:25` / `fix-playbooks.md:57-60` `WEBVIEW_CONTEXT` — «элемент виден, драйвер не находит → переключись в веб-контекст» | Верно, но у них зафиксирована **обратная** ошибка: `AUT_OVERVIEW.md:32` — наличие `WEBVIEW_su.ivcs.aura` в списке contexts **не** значит, что экран в WebView; для protection/PIN правильный context — `NATIVE_APP`. Плюс `AUT_OVERVIEW.md:70` про stale WEBVIEW на iOS 26.3 | **неполно** |
| `fix-playbooks.md:34-35` «разные поколения — разные идентификаторы (`su.ivcs.ucim` против `su.ivcs.kmp`)» | Идентификатор один — `su.ivcs.aura`. Дискриминатор `APP_NOT_STARTED` по «не то поколение» больше не работает | **устарело** |
| `failure-taxonomy.md:44-49`, `fix-playbooks.md:87-90`, `batch-conventions.md:65` «живых прогонов на этом стеке не было ни одного — ни у нас, ни у них» | **Опровергнуто прямым артефактом.** `.agents/state/aut-research/AUT_OVERVIEW.md:5` «Последняя живая проверка: **2026-08-04**»; `:7` сборки Android 1.3.5 (60) / iOS 1.3.5 (10); `:10` стенд `at-develop3`; `:11` Appium 3.6.0; changelog `:98-100` — 03.08 и 04.08. Флоу TC-468 пройден на обеих платформах, включая системный consent, PIN, notification permission | **опровергнуто** |
| `failure-taxonomy.md` таблица классов | Не хватает класса «системная поверхность перекрыла AUT»: `AUT_OVERVIEW.md:36-37` — Android notification dialog из `com.google.android.permissioncontroller`, кнопка `com.android.permissioncontroller:id/permission_allow_button`; `AUT_OVERVIEW.md:66-67` — системный alert iOS, невидимый в `source`, но доступный через `driver.switch_to.alert`. Наш `LOCATOR_STALE` от этого не отличается | **неполно** |
| `batch-conventions.md:22` команда пачки с `--alluredir=./allure-raw-<метка>` | Расходится с их правилом: `craft-stack/pytest-runner.md:19` «`allure-raw/` создаётся настройкой `pytest.ini`, отдельный `--alluredir` не нужен». Плюс команда без `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1` / `-p allure_pytest.plugin` не заработает | **неполно / конфликт** |
| `batch-conventions.md:12-17` «параллелить можно по устройствам — каждому прогону своё `--device-name`» | Подтверждено и шире: `mobile_env.yaml` даёт по 3 устройства на платформу с раздельными `device_port`/`system_port`, а `conftest.py:82-98` `setup_multiple_auts` умеет несколько драйверов **внутри одного теста**. Наш канон трактует несколько устройств только как несколько прогонов | **верно, но неполно** |
| `batch-conventions.md:54` «дискриминирующий режим — другое устройство или `--local`» | Уточнение из их канона: `craft-stack/raw-results.md:29-34` — `device_host=local:<port>` **не** доказывает флаг `--local`; при отсутствии доказательства режим ставится `unknown` | **неполно** |
| `allure-raw-parser.md:8` «`addopts` **обеих реп** содержит `--alluredir=./allure-raw`» | `pytest.ini:7` — да, `--alluredir=./allure-raw`. Формулировка «обеих реп» устарела | **верно (формулировка устарела)** |
| `allure-raw-parser.md:26-29` «скриншоты именуются `Device-{udid}-…`, устройство всегда есть в имени» | Верно только частично: `tests/pages/base.py:104` и `conftest.py:158` (ветка multi-driver) — да, с udid. **Одиночная ветка `setup_aut` пишет без udid**: `conftest.py:150` `name=item.name`. Устройство надёжно берётся не из имени вложения, а из `environment.properties`, который пишет `pytest_runtest_makereport` (`conftest.py:117-129`: `stand=`, `os=`, `device=`) — этого в нашем каноне нет | **неполно** |
| `allure-raw-parser.md:21` «привязка к тест-кейсу — метка `tc` из `log.tc`» | Теперь ещё `@allure.id`. Их правило: `craft-stack/raw-results.md:15-16` «TC-id ищи в Allure labels/links, затем в метаданных теста» | **неполно** |
| `allure-raw-parser.md:49-52` «загрузка в TestOps идёт `allurectl`, доступ по env `TESTOPS_*` либо gitignored `secrets.yaml`» | Секретная часть — **верно** (`secrets.yaml.example:6-14`: `TESTOPS_ENDPOINT`/`TESTOPS_TOKEN`/`TESTOPS_PROJECT_ID`). Но `allurectl` в CI **закомментирован** (`.gitlab-ci.yml:19,24`), а TestOps читается своей обёрткой: `AGENTS.md:126-127` `PYTHONPATH=.codex/scripts ./.venv/bin/python -m testops`, «сырой curl не дёргать» | **неполно** |

## Ответы на точечные вопросы задачи

### 1. Структура тестов и page-слоя

**Было.** `one-mobile-ios`: 6 тест-файлов, ~18 page-объектов, `chunks/` с рабочим реестром
`get_chunk(platform, chunk_name)` и `bottom_menu`, `browser/` с `get_browser(platform)`,
локаторы `locators/iOS/` + `locators/Android/` (нативные) с подкаталогами `chunks/`, `browser/`
внутри каждого. `one-mobile-android`: 1 тест-файл, 3 page-объекта (`kmp_*`), локаторы только
`locators/Android/KMP/`, `chunks/__init__.py` и `browser/__init__.py` — **пустые**.

**Стало.** 1 тест-файл (`tests/test_one_login.py`), 3 page-объекта (`auth.py`, `chats.py`,
`passcode.py`) — каждый содержит **обе** платформенные реализации, локаторы в двух каталогах:
`tests/pages/locators/Android/` (9 файлов + `README.md`) и `tests/pages/locators/iOS/` (4 файла).

`tests/pages/chunks/` **существует, но пуст** (`__init__.py` — 0 байт), как и
`tests/pages/browser/`. При этом `AGENTS.md:41` и `README.md:215` перечисляют `chunks/` в структуре
проекта, а `.agents/rules/page-objects.md:118` требует регистрировать каждый chunk в реестре
`tests/pages/chunks/__init__.py:get_chunk(...)` — реестра нет. **Chunks в нашем каноне не
упоминаются ни разу** — это пробел, а не устаревание.

Деление по платформам внутри — **только в локаторах** (`locators/Android/` vs `locators/iOS/`).
В page-слое платформы сведены в один модуль через общий `_PBase<Name>` + два подкласса, выбор — в
реестре `get_page(platform, …)`, не в тесте.

### 2. Идентификатор приложения — **подтверждаю: `su.ivcs.aura` для обеих платформ**

Все места, где фигурирует (полный список):

| Файл:строка | Что |
|---|---|
| `tools/aut/__init__.py:31` | `"appium:appPackage": "su.ivcs.aura"` (было `su.ivcs.kmp`) |
| `tools/aut/__init__.py:57` и `:59` | `caps["appium:bundleId"] = "su.ivcs.aura"` (было `su.ivcs.ucim`) — **обе ветки `if/else` идентичны**, ветвление мёртвое (было мёртвым и раньше) |
| `tools/aut/__init__.py:72` | `xcrun simctl uninstall … su.ivcs.aura` (новый вызов, в старой репе его не было) |
| `tools/utils/__init__.py:9` | `android_package = 'su.ivcs.aura'` |
| `tools/utils/__init__.py:73` | `xcrun devicectl … uninstall app … su.ivcs.aura` |
| `README.md:133`, `README.md:281` | «Bundle ID приложения — `su.ivcs.aura`» |
| `.agents/state/aut-research/AUT_OVERVIEW.md:8` | «Application id / Bundle id: `su.ivcs.aura`» |
| `.agents/state/aut-research/AUT_OVERVIEW.md:32` | контекст `WEBVIEW_su.ivcs.aura` |

Заодно сменилась activity: `tools/aut/__init__.py:32` `"appium:appActivity": "su.ivcs.messenger.MainActivity"`
(в старой android-репе строка была **закомментирована**).

**Три места, где старые идентификаторы остались и это протухшие хвосты в самой репе:**

- 🔴 `tests/pages/locators/Android/common.py:12` — **живой локатор** `return ('id', 'su.ivcs.kmp:id/action_bar_root')`.
  При `appPackage = su.ivcs.aura` этот id не найдётся. Дефект, а не косметика.
- `tests/pages/locators/Android/README.md:4` — `Application id: su.ivcs.kmp` (документ снят на 1.2.10).
- `.codex/rules/page-objects.md:147,150` — примеры с `su.ivcs.ucim:id/header_text`, `…:id/item_text`.

### 3. Две стратегии адресации — что стало

- **README есть**, но переехал: `tests/pages/locators/Android/KMP/README.md` → `tests/pages/locators/Android/README.md`.
  Диффа два слова: заголовок `# Android KMP locator research` → `# Android locator research`,
  и «rendered mostly through Compose/KMP» → «through Compose», «when the KMP app exposes them» →
  «when the app exposes them». Приоритет `id → content-desc → точный текст через i18n → bounds`
  (`:12-17`) — **сохранён дословно**.
- **Деление по поколениям НЕ сохранилось.** Каталога `KMP/` нет; Android-пакет один и он же
  Compose-пакет. Ось «нативное vs Compose» в репе больше не выражена никак.
- **Утверждение про Compose применимо** — и усилено живой разведкой (`AUT_OVERVIEW.md:28-29,42-43`).
- **Утверждение про «нативный клиент даёт штатные id» НЕ применимо ни к чему в этой репе.** Нативного
  поколения нет. Новый iOS-пакет написан на `-ios predicate` по `type` + видимому тексту через i18n
  (`locators/iOS/common.py:13-26`), то есть по сути та же текстовая стратегия, что у Compose, а не
  «accessibility id». Единственный accessibility id — `locators/iOS/auth.py:19`.
- ⚠️ Отдельно: `locators/iOS/auth.py:15` — `('xpath', '//XCUIElementTypeOther[@name="баннер"]')`,
  **русский литерал в локаторе мимо i18n**, плюс на `:14` закомментированный
  `('name', 'Вход в учетную запись')`. Нарушение нашего же правила (`rules/page-objects.md:38-42`)
  и их собственного (`.agents/rules/page-objects.md:153`).

### 4. `conftest.py` — сравнение построчно

**`conftest.py` новой репы БАЙТ-В-БАЙТ совпадает с `one-mobile-android-main/conftest.py`.
`diff -u` пустой. Не изменилось НИЧЕГО.** Перепроверка подтверждает твоё наблюдение и закрывает
вопрос «что ещё изменилось»: ничего.

- 10 `addoption`, `--env` без `default` (`:169`) — да.
- Единственное расхождение существует не с новой, а **между двумя старыми**: в
  `one-mobile-ios-main/conftest.py` фикстура `run_around_tests` содержала блок
  `stands.reset_exim_queue()` / `stands.restore_db_snapshot()` / `set_admin_lang_ru()` под флагами
  `--no-exim-reset` / `--no-db-restore`. В android-репе его нет — и **в новую репу он не попал**.
  Следствие: опции `--no-exim-reset` и `--no-db-restore` объявлены (`:177-178`), но **нигде не
  читаются**. Мёртвые флаги; наш `runner.md:65` описывает их как «пропуск подготовительных шагов»,
  которых больше нет.
- Мелкий дефект, унаследованный из обеих старых реп: `conftest.py:42` логирует `option.version`,
  тогда как объявлен `--ver` (dest `ver`, `:170`). `option.version` — встроенный флаг pytest, то есть
  в лог уходит не версия клиента. Не наш канон врёт, но знать стоит.
- `option.allure_report_dir` (`:126`) приходит не из `pytest_addoption`, а из плагина allure —
  ещё одна причина, почему `-p allure_pytest.plugin` обязателен.

### 5. `pytest.ini` — маркеры

**Тоже байт-в-байт: новая = старая android = старая ios.** 11 маркеров (`pytest.ini:9-19`):
`wip`, `requiredb`, `suite_smoke`, `suite_regress`, `suite_regress_simple`, `suite_regress_advanced`,
`suite_regress_full`, `lang_en`, `ios`, `android`, `qaauto_smoke`.

Наш `runner.md:77-81` перечисляет **ровно эти 11** — совпадение полное, ни одного лишнего или
пропущенного. `addopts` (`:7`) тоже без изменений: `-v -p no:cacheprovider --tb=native --showlocals
--strict-markers --alluredir=./allure-raw`.

Попутно: наша прошлая разведка от 30.07 писала «набор из **12** маркеров» — это была ошибка счёта,
их 11.

### 7. `.gitlab-ci.yml` — как гоняют в CI

**Никак. Пайплайн не работает — он состоит из двух `echo`.**

```yaml
before_script:
  - echo '>>>'
#    - curl … allurectl … ; chmod +x allurectl ; pip install -r requirements.txt
script:
  - echo '<<<'
#    - ./allurectl watch -- python -m pytest -m "android and suite_smoke" --lang ru --env at-develop3 \
#        --device-host 10.6.0.219:15554 --device-os Android --device-name RF8W80MY1ML
```
(`.gitlab-ci.yml:17-24`)

- Единственное отличие от старой android-репы — добавлен раннер-тег: `tags: [ci-one-mobile]` (`:10-11`).
- **В старой ios-репе пайплайн был живой** (`one-mobile-ios-main/.gitlab-ci.yml:15-20` — без
  комментариев). При слиянии взяли закомментированную android-версию, и рабочая команда осталась
  только как комментарий.
- `PYTHON_VERSION: "3.9"` (`:5`) против фактического Python 3.13 (`README.md:25,49`) —
  переменная протухла; `image: gradle` (`:12`) для python-проекта.
- Устройство в закомментированной команде — `10.6.0.219:**15554**`, а в `mobile_env.yaml`
  Android-хосты `15555/15556/15557`. Согласованности нет.
- `--from-pipeline` (`conftest.py:172`) объявлен и не используется нигде.

**Вывод:** отсутствие CI-лейна в нашем каноне — не пробел, а соответствие реальности. Прогоны
локальные, командами из `README.md:146-172`.

## Дополнительно: в репе уже живёт агентный кит — и он конфликтует сам с собой

Это не было в задании, но напрямую бьёт по нашему стеку.

`.agents/` + `.codex/` + `AGENTS.md` (165 строк) — установленный агентный комплект. В нём есть
**свой дом стека**: `.agents/skills/craft-stack/` со стеком, названным `pytest-appium-mobile`
(`SKILL.md:6`), и файлами `recon.md`, `pytest-runner.md`, `test-templates.md`,
`batch-conventions.md`, `raw-results.md` — то есть покрытие 5 из наших 9.

Три расхождения, которые надо развести до того, как мы что-то поставим:

1. **`.agents/rules/page-objects.md` и `.agents/rules/tests.md` — это НЕадаптированные копии
   one-web.** Там браузеры, DOM, тосты, `users_factory`, `web_preamble`, `multiple_browsers`,
   `testid`, Safari/Firefox (`page-objects.md:130-136,185-212`, `tests.md:40-64`). При этом
   `craft-stack/SKILL.md:21-22` ссылается **именно на них** как на канон page-слоя мобильного стека.
2. **`.codex/rules/page-objects.md` (238 строк) — мобильный и осмысленный, но написан против СТАРОЙ
   ios-репы.** Он ссылается на файлы, которых в слитой репе больше нет: `locators/iOS/login.py`
   (`:110`), `pick_server.py` (`:126`), `chats.py` с задвоенными методами (`:129-130`),
   `contacts.py` (`:133`), `schedule_meeting.py`/`edit_schedule_meeting.py`/`rooms.py` (`:136-137`),
   `meeting.py` (`:174`), `meeting_login.py` (`:165`). Плюс идентификатор `su.ivcs.ucim` (`:147,150`)
   и конвенция accessibility id `Zero.<name>` (`:158`), которой в новом iOS-пакете нет.
3. **Три разных порядка приоритета локаторов** — наш, `craft-stack/recon.md:62-77` и
   `.codex/rules/page-objects.md:144-168`. Ни один не совпадает с другим.

## Риски и хвосты (факты, не предложения)

- 🔴 `locators/Android/common.py:12` — живой локатор со старым package `su.ivcs.kmp`.
- 🔴 Наша команда запуска в `runner.md` и `batch-conventions.md` на Python 3.13 даёт `Empty suite`.
- 🟡 `consts.py:20` — захардкоженный пароль CI-пользователя в открытом виде (значение не привожу);
  унаследовано из обеих старых реп, не изменилось.
- 🟡 `--no-exim-reset` / `--no-db-restore` — мёртвые опции, наш канон их описывает как рабочие.
- 🟡 `tests/pages/chunks/` и `browser/` — пустые пакеты; `tools/aut/__init__.py:7` импортирует
  `browser`, а `:131` в закомментированном коде зовёт `browser.get_browser(...)`, которого нет.
- 🟡 `requirements.txt`: `pytest` без пина (README фиксирует 7.4.4), `psycopg2-binary` распинован,
  добавлен `aiohttp`, `PyYAML` 6.0 → 6.0.3.

## Открытые вопросы (данных нет, догадок не пишу)

- **[?]** Планируется ли восстановление нативных тестов (`test_meetings`, `test_rooms`,
  `test_contacts`, `test_links`) в новой репе или они списаны вместе с поколением. В репе следов
  переноса нет.
- **[?]** Какой из двух внутренних канонов адресации у них главный — `craft-stack/recon.md` или
  `.codex/rules/page-objects.md`. Оба живые, оба в `main`.
- **[?]** Будет ли CI оживлён и на каком раннере (`ci-one-mobile` тег добавлен, но команда
  закомментирована).

## Связано

[[recon-qa-mobile-appium-2026-07-30]] · [[QA — карта поверхностей, стеков и ролей]] ·
[[canon-profili — канон постановки по профилям и поверхностям]]
