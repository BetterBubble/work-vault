---
title: critic-mobile-fit-0804
type: note
permalink: tacticum/00-board/critic-mobile-fit-0804
---

# Ревизия: ложится ли мобильная роль на репозиторий заказчика

Задача — опровергнуть «ложится ровно». Источник правды — их репозиторий
(снапшот `mobile-main`, 04.08). Наш текст — ветка `qa-integrate`.

**Найдено 21 расхождение.** Ниже по убыванию серьёзности; каждое — «наш текст → что в их коде → чем грозит».

---

## 1. КРИТ · Основная команда прогона не запускает pytest

**Наш текст.** `stack/runner.md:10` — «`<раннер-проекта>` — интерпретатор окружения проекта».
`stack/runner.md:19-23` (та же форма в `:155-158` и `batch-conventions.md:25-28`):

```
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 <раннер-проекта> -p allure_pytest.plugin \
  <node-id | -k <имя>> -m <android|ios> ...
```

**В их коде.** `README.md:149-157` и их же `.agents/skills/craft-stack/pytest-runner.md:9`:
`PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 ./.venv/bin/python -m pytest -p allure_pytest.plugin …`.
Подстановка по нашему определению даёт `./.venv/bin/python -p allure_pytest.plugin …` — `-m pytest`
потерян, интерпретатор падает на неизвестной опции.

**Плюс внутренняя несовместимость плейсхолдера.** `runner.md:177` — `<раннер-проекта> -m py_compile
<изменённые .py>` требует, чтобы плейсхолдер был ГОЛЫМ интерпретатором. `runner.md:19/155/178` и
`batch-conventions.md:25` требуют, чтобы он уже включал `-m pytest` (иначе `-p`, `-k`,
`--collect-only` некуда деть). Одно значение обе роли не закрывает: если раннер = `python -m pytest`,
то дешёвый гейт превращается в marker-expression `-m py_compile` и молча не проверяет ничего.
Их `pytest-runner.md:20-21` разводит это явно: `./.venv/bin/python -m py_compile` против
`./.venv/bin/python -m pytest --collect-only`.

**Грозит.** Первая же команда агента либо не выполняется, либо «проходит» вхолостую.

---

## 2. КРИТ · Ни один файл канона не знает, что каждый прогон ПЕРЕУСТАНАВЛИВАЕТ приложение

**В их коде.** `conftest.py:69` — `aut.clear_data(do, dh, dn, is_pd)` вызывается в `prepare_driver`
ДО `aut.start`. `tools/aut/__init__.py:92-98` — Android: `apk_reinstall(device_name,
f'{os.getcwd()}/one.apk', …)`; `:67-91` — iOS: `xcrun simctl uninstall/install … iosApp.app` либо
`Ucim.ipa`. `tools/utils/__init__.py:86-88` — `pm clear` + `pm uninstall` пакета.
`conftest.py:68` — `restore_adb_uiautomator` вдобавок сносит `io.appium.uiautomator2.server*`.
`README.md:121-135` дословно: «Положи Android-сборку в корень репозитория под именем `one.apk`.
Перед каждым прогоном раннер переустанавливает приложение». `.gitignore:37-38` — `*.apk`, `*.app`
в git не приезжают.

**Наш текст.** `runner.md` §«Форма команды», §«Опции устройства», §«Дешёвые гейты» — четыре
обязательных значения и ни слова о сборке. `batch-conventions.md:73-77` «Гигиена» — только про
allure-raw и сессии. Роль-фрагмент §«Незыблемое» перечисляет `--env` + три опции устройства как
полный список предусловий.

**Грозит.** (а) Любой прогон или разведка без `one.apk` / `iosApp.app` в корне падает в
`prepare_driver` ДО контакта с Appium — класса под это в нашей таксономии нет.
(б) Агент, следующий нашей же гигиене «чужую сессию не убивать», молча сносит приложение с
устройства, за которым сидит человек. Их собственный ledger это знает:
`.agents/state/codebase-refactoring/health-snapshots.jsonl` — `"live_test":"not run: would
reinstall AUT on device"`.

---

## 3. КРИТ · Таксономия не стыкуется с их живой: 9 классов против 11, пересечение — один `PRODUCT_BUG`

**Наш текст.** `stack/failure-taxonomy.md:16-27` — `ENV_CONFIG`, `DEVICE_UNAVAILABLE`,
`APP_NOT_STARTED`, `LOCATOR_STALE` (+подтип), `LANG_MISMATCH`, `RESOLUTION_MISMATCH`, `TIMING`,
`WEBVIEW_CONTEXT`, `PRODUCT_BUG`.

**В их коде.** `.agents/skills/fix-failed-test/references/failure-taxonomy.md:3` — «9 категорий
(8 тестовых + `STAND_INCIDENT`)»: `DOM_CHANGED`, `UX_FLOW_CHANGED`, `API_CONTRACT_CHANGED`,
`FLAKY_TIMING`, `IMPORT_OR_REGISTRY`, `PRODUCT_BUG`, `WEBDRIVER_QUIRK`, `NON_REPRODUCIBLE_FLAKY`,
`AUTOMATION_ONLY`, `STAND_INCIDENT`, `UNKNOWN`. Имена — контракт, а не проза: их потребляют
`.agents/state/metrics-README.md` (`result: stand-incident | known-issue |
covered-by-undelivered`), корзины `state.md` (`deferred_flaky`, `deferred_automation`,
`stand_incident`, `covered_by_undelivered`) и задание субагенту `code-writer (mode=fix,
category=DOM_CHANGED)`.

**Грозит.** Агент выдаёт класс, которого не понимает ни `code-writer`, ни `metrics.jsonl`, ни
`known-issues.md`. Обратно: у нас нет ни фолбэка `UNKNOWN`, ни статусов deferred — при несовпадении
ни с одним из 9 классов у агента нет легального хода.

---

## 4. КРИТ · Пропущены три класса, которые в ИХ репе самые вероятные

**`IMPORT_OR_REGISTRY`.** `tests/pages/__init__.py:6-20` — `get_page` возвращает `None` для
незарегистрированного ключа (никакого `raise`), и зарегистрировано ровно три экрана: `auth`,
`chats`, `passcode`. При этом файлов локаторов Android семь (`auth`, `bottom_menu`, `calls`,
`chats`, `contacts`, `more`, `passcode`). Первый же новый тест на «Контакты» даёт
`TypeError: 'NoneType' object is not callable`. Класса нет — попадёт в `LOCATOR_STALE`.

**`API_CONTRACT_CHANGED`.** Наш `test-templates.md:74-75` и `rules/tests.md:94-98` предписывают
готовить данные REST-обёртками `autocore`. Класса под смену их сигнатуры у нас нет; у них есть, с
отдельной веткой «функции в обёртке вообще нет → тест не править, `NEEDS_DEP`».

**`WEBDRIVER_QUIRK` / «только под автоматизацией».** `tests/pages/auth.py:113-174` — весь iOS-логин
состоит из обходов драйвера: поллинг стабильности `rect`, проверка `hittable`, показать/скрыть
пароль чтобы попасть в поле, `hide_keyboard`, ожидание исчезновения `XCUIElementTypeKeyboard`.
Наш `PRODUCT_BUG` (`failure-taxonomy.md:27`) — «воспроизводится вручную теми же шагами», без
дискриминатора «руками работает, под драйвером нет». Их канон предупреждает прямо: дефолт при
сомнении — `WEBDRIVER_QUIRK`, потому что ошибка в сторону `PRODUCT_BUG` дороже (эскалация на
команду продукта).

---

## 5. КРИТ · `ENV_CONFIG` мис-диагностирует самый частый в этой репе пустой отбор

**Наш текст.** `failure-taxonomy.md:18` — «падает всё сразу … **или не отбирается ни один тест** …
Второй облик того же класса — `Empty suite` … потеряны `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1` /
`-p allure_pytest.plugin`». `runner.md:34-36` — «`Empty suite` на этом стеке — в первую очередь
диагноз команды, а не отбора. **Прежде чем править маркеры и селектор**, проверь обе части формы».

**В их коде.** `pytest.ini:8-21` объявляет 11 маркеров, а тестов в репозитории **один** —
`tests/test_one_login.py`, и он несёт четыре: `suite_smoke`, `suite_regress`, `android`, `ios`.
Семь маркеров (`suite_regress_simple`, `suite_regress_advanced`, `suite_regress_full`,
`qaauto_smoke`, `wip`, `requiredb`, `lang_en`) не носит ни один тест. Наш `runner.md:140-146` и
`batch-conventions.md:43-45` предлагают все шесть suite-маркеров как рабочие наборы. Их
`run-tests/SKILL.md:51` знает правило, которого нет у нас: «мёртвые маркеры (объявлены, но тестов
не носят) в словарь не заводить — запуск по ним даёт `no tests collected`».

**Грозит.** Агент гонит `-m "suite_regress_full and android"`, получает пустой отбор, по нашей норме
объявляет это ошибкой команды и начинает чинить корректную команду.

---

## 6. КРИТ · Наш канон приземляется РЯДОМ с их каноном; все живые ссылки ведут в старую копию

**Наш текст.** `iva-pytest-appium-autotest-base/manifest.yaml:104-202` — все девять файлов ставятся
в `.agent-kit/craft/stacks/pytest-appium/`, `.agents/` не трогается. Роль-фрагмент
(`AGENTS.md.fragment:3-7`) прямо отдаёт приоритет их файлам: собственные правила репозитория
«remain authoritative».

**В их коде.** Живые указатели ведут в другое место и под другими именами:
`.codex/agents-instructions/dom-explorer.md:95,118,127,300` → `.agents/skills/craft-stack/recon.md`;
`run-tests/SKILL.md:131,142` → `craft-stack/SKILL.md § «Артефакты прогона»` и
`craft-stack/raw-results.md`; `craft-stack/SKILL.md:18-26` — карта файлов с именами
`pytest-runner.md` и `raw-results.md` (у нас — `runner.md` и `allure-raw-parser.md`); правила —
`.agents/rules/page-objects.md`, `.agents/rules/tests.md`.

**Грозит.** Две конкурирующие копии канона, и выигрывает старая — на неё ссылаются, и наш же
фрагмент отдаёт ей приоритет. Причём старая — это неперенесённый веб-текст: `.agents/rules/tests.md:45-58`
предписывает `web_preamble`, `users_factory`, `relogin_session`, `multiple_browsers`,
`parallel_unsafe`, а `.agents/rules/page-objects.md:130-136,185-191` — браузерные гарды
`driver.name == 'safari'`, chunk `toast`, синтетический JS-клик. Ничего из этого в мобильном
репозитории нет.

---

## 7. ВАЖНО · Структура page-слоя описана неверно для двух экранов из трёх

**Наш текст.** `stack/rules/page-objects.md:112-118` — «общая логика + платформенный подкласс:
`_PBase<Экран>` держит сценарий, а `PAndroid<Экран>` / `PIOS<Экран>` подмешивают платформенные
локаторы». `:120-121` — «имя метода и порядок параметров **обязаны совпадать** между Android и iOS,
иначе общий сценарий в `_PBase` не соберётся».

**В их коде.** Так устроен только `tests/pages/chats.py:6-24` (`_PBaseChats` + два пустых подкласса).
`tests/pages/auth.py:16-56` и `:58-180`, `tests/pages/passcode.py:12-46` и `:49-88` — **два
независимых класса с полностью продублированным сценарием**, общего `_PBase<Экран>` нет.
Причина видна там же: сигнатуры локаторов НЕ совпадают —
`locators/Android/auth.py:56` `L_LOGIN_USERNAME_INPUT()` без `lang` против
`locators/iOS/auth.py:37` `L_LOGIN_USERNAME_INPUT(lang=consts.LANG_RU)`; у Android есть
`L_CLEAR_SERVER_BUTTON`, `L_PRODUCT_LABEL`, `L_LOGIN_WEBVIEW`, у iOS — `L_I18N_LOGIN_HEADER`,
`L_LOGIN_VISIBLE_PASSWORD_INPUT`, `L_I18N_LOGIN_HIDE_PASSWORD_BUTTON`; `LChats` — 15 методов на
Android против одного на iOS.

**Грозит.** Агент, «следующий референсу», либо ищет несуществующий `_PBaseAuth`, либо начинает
рефакторить чужой page-слой, чтобы выполнить наш инвариант паритета сигнатур.

---

## 8. ВАЖНО · Число и адрес `bounds`-локаторов названы неверно

**Наш текст.** `rules/page-objects.md:91-93` — «Живых `bounds`-локаторов в Android-пакете сейчас
**шесть** — в экранах ввода сервера, PIN, контактов и нижнего меню».

**В их коде.** Жёстко зашитых `bounds` — **десять**, в **шести** файлах:
`locators/Android/auth.py:25`, `passcode.py:40,60`, `contacts.py:13,17`, `chats.py:13,17`,
`calls.py:13,17`, `bottom_menu.py:9` (плюс параметризованный `common.py:37`, его мы посчитали
отдельно и верно). Пропущены в перечислении именно `chats` и `calls` — экраны, до которых доходит
их единственный живой тест.

**Грозит.** Агент, сверяющий координатный долг по нашему числу, считает `chats`/`calls` чистыми и не
помечает `bounds` как долг.

---

## 9. ВАЖНО · «Протухли только реквизиты README» — неправда, мёртвый пакет живёт в коде

**Наш текст.** `rules/page-objects.md:46-48` — «Шапка этого README протухла … Правила выбора
локатора в нём актуальны, продуктовые реквизиты — нет; сверяй их с `tools/aut` и `tools/utils`».
`recon.md:105-107` — то же самое. При этом `fix-playbooks.md:43-46` говорит противоположное и
верное: «один живой Android-локатор адресован по `su.ivcs.kmp:id/…`».

**В их коде.** `tests/pages/locators/Android/common.py:12` — `L_APP_ROOT()` возвращает
`('id', 'su.ivcs.kmp:id/action_bar_root')`, тогда как `tools/aut/__init__.py:31` ставит
`appPackage: su.ivcs.aura`, а `tools/utils/__init__.py:9` — `android_package = 'su.ivcs.aura'`.

**Грозит.** Профильные файлы адресации (`page-objects.md`, `recon.md`) уверяют, что код чист, — агент
берёт `L_APP_ROOT()` корневым якорем, и он не найдётся никогда. Плюс наш канон противоречит сам себе
в трёх местах.

---

## 10. ВАЖНО · `--local` объявлен рабочим механизмом, которого нет

**Наш текст.** `batch-conventions.md:70` — «Дискриминирующий режим на этом стеке — другое устройство
или локальный режим (`--local`)». `runner.md:102` — `--local` в таблице живых опций; мёртвыми
названы только `--no-exim-reset` / `--no-db-restore` (`runner.md:104-111`).

**В их коде.** `conftest.py:173` — единственное вхождение `--local` во всём репозитории;
`option.local` не читает никто (grep по `*.py` даёт ноль потребителей). «Локальность» задаётся не
флагом, а строкой `local` в `--device-host`: `tools/aut/__init__.py:23-24` (`device_ip == 'local'`
→ `0.0.0.0`) и `tools/utils/__init__.py:13` (локальный `subprocess` вместо ssh).

**Грозит.** Единственный названный нами дискриминирующий приём разбора пачки — no-op: повтор
«в локальном режиме» даёт ровно тот же прогон.

---

## 11. ВАЖНО · Дискриминатор «устройство в имени скриншота» построен на мёртвом методе

**Наш текст.** `allure-raw-parser.md:31-45` — «Скриншоты в референсе именуются с идентификатором
устройства — базовый класс страниц прикрепляет их как `Device-{udid}-{сообщение}`… Падение,
воспроизводящееся на одном устройстве, — это `RESOLUTION_MISMATCH`… без учёта устройства этот
дискриминатор недоступен».

**В их коде.** `tests/pages/base.py:102-107` `take_screenshot` действительно так именует — но он
**не вызывается ниоткуда** (grep `take_screenshot` даёт только определение). Реальное вложение
делает `conftest.py:147-152`: для `setup_aut` (единственная используемая фикстура) —
`allure.attach(..., name=item.name)`, **без udid**. Префикс `Device-{udid}-` есть только в ветке
`setup_multiple_auts` (`conftest.py:154-159`).

**Грозит.** Агент ищет udid в именах вложений одиночного прогона, не находит и либо считает
результаты битыми, либо теряет предписанный нами дискриминатор.

---

## 12. ВАЖНО · Новые петли пишут в `.tasks/`, а дом артефактов у них — `.agents/state/`

**Наш текст.** `skills/refactor-codebase/SKILL.md:42-43,71-76,110-111` →
`.tasks/codebase-refactoring/health-snapshots.jsonl`, `.tasks/work/refactor-<дата>/`,
`.tasks/metrics.jsonl`. `skills/update-autotest/SKILL.md:66-73` → `.tasks/work/update-{id}/tc.md`,
`meta.json`, `diff.md`.

**В их коде.** `.agents/state/README.md` — «В корне `.agents/state/` не появляется ничего вне
таблицы… новый инструмент не изобретает себе места». В таблице УЖЕ есть ровно наши артефакты, но по
другим путям: `codebase-refactoring/` (с непустыми `health-snapshots.jsonl` и `smell-inventory.md`),
`metrics.jsonl`, `TODO/`, `work/update-{id}/`, `work/refactor-<дата>/`. `.gitignore` покрывает
`.agents/state/_scratch/` и `.agents/state/worktree-*/`; под `.tasks/` не покрыто ничего.

**Грозит.** Второй параллельный дом тех же реестров; существующий `smell-inventory.md` с закрытой
записью от 2026-08-01 игнорируется и заводится пустой; scratch петель уезжает в коммит.

---

## 13. ВАЖНО · Наша норма «беклога в поставке нет» отменяет их работающий беклог

**Наш текст.** `refactor-codebase/SKILL.md:84-85` — «Крупные кандидаты … **в беклог не заводим — его
в поставке нет**»; `:87-88` — «Проект ведёт `.tasks/TODO/` своей конвенцией — положи туда тот же
текст; не ведёт — каталог под это **не заводить**».

**В их коде.** Беклог есть и работает: `.agents/state/TODO/INDEX.md`, `_TEMPLATE.md`,
`.codex/scripts/backlog_dashboard.py`, скилл `backlog-dashboard`. Их версия того же скилла
(`.agents/skills/refactor-codebase/SKILL.md:88-90`) предписывает обратное — «завести карточкой в
`.agents/state/TODO/`». Путь `.tasks/TODO/` у них не существует, значит по нашей букве агент
заключит «проект не ведёт» и не запишет отложенные кандидаты никуда.

**Грозит.** Хвост мероприятия теряется молча — ровно та потеря контекста между сессиями, против
которой их `INDEX.md` и заведён.

---

## 14. ВАЖНО · Верификация обеих петель делегируется `/run-tests`, который у них собирает нерабочую команду

**Наш текст.** `update-autotest/SKILL.md:99-104` и `refactor-codebase/SKILL.md:92-93,102-105` —
прогон «ДО/ПОСЛЕ» и верификация ресинка идут через `/run-tests`.

**В их коде.** `.agents/skills/run-tests/SKILL.md:99` — база команды:
`./.venv/bin/python -m pytest <paths> [-k …] -m "<markers>" --env <env> [--browser-tag current |
--local]`. Нет `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1`, нет `-p allure_pytest.plugin`, нет
`--device-os/--device-host/--device-name`. Зато есть браузерный словарь (`:57-67`), `--browser-tag`,
`-n N --dist loadgroup` (xdist в `requirements.txt` отсутствует), маркер `parallel_unsafe` (в
`pytest.ini` не объявлен → `--strict-markers` уронит прогон) и путь B через CI (`.gitlab-ci.yml` —
два `echo`).

**Грозит.** Обязательный гейт «зелёный ДО и ПОСЛЕ» и верификация ресинка исполняются командой,
которая не поднимает сессию и отдаёт `Empty suite`, — фаза формально «пройдена».

---

## 15. СРЕДНЕ · `recon.md` противоречит себе в способе запуска и не называет ни одной точки входа

**Наш текст.** `recon.md:17-20` — «Собирай сценарий разведки одним скриптом на драйвере проекта и
запускай его **как обычный python-файл**». `recon.md:39-42` — «Скрипт разведки запускается **той же
формой команды**, что и прогон: на Python 3.13 без `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1` сессия не
поднимется через pytest-харнесс».

**В их коде.** `conftest.py:181-183` — глобаль `option` заполняется только в `pytest_configure`,
поэтому `setup` / `prepare_driver` вне pytest недоступны в принципе. Годная точка входа —
`tools.aut.start(device_os, device_host, device_name, is_pd, dp, sp)`
(`tools/aut/__init__.py:21`) вместе с `tools.utils.is_real_device` (`tools/utils/__init__.py:37`),
но наш канон их не называет.

**Грозит.** Две взаимоисключающие инструкции и ни одного имени функции — агент выбирает наугад.

---

## 16. СРЕДНЕ · Роль-фрагмент и манифест роли описывают репозиторий, которого больше нет

**Наш текст.** `iva-role-qa-mobile/.../codex/AGENTS.md.fragment:13-15` и
`claude-code/CLAUDE.md.fragment:13-15` — «Роль обслуживает **оба поколения клиента** — нативное
(`su.ivcs.ucim`, iOS и Android) и KMP (`su.ivcs.kmp`, Android)»; `:76` — «нативный клиент —
нормальное дерево, штатные `id` / `accessibility id`»; `:91-92` — «референс — форма логина на
KMP-Android». `manifest.yaml:10-19` — про две репы `ivaqa/one-mobile-ios` /
`ivaqa/one-mobile-android`; `:100-102` — `stack.optional: [pytest-appium, one-mobile-ios,
one-mobile-android]`; `:17` — путь `locators/Android/KMP/README.md`.

**В их коде.** Репозиторий один — `git@git.hi-tech.org:ivaqa/mobile.git` (`README.md:46`).
Приложение одно — `su.ivcs.aura` на обеих платформах (`tools/aut/__init__.py:31,57-59`;
`tools/utils/__init__.py:9`). `su.ivcs.ucim` не встречается нигде (только в именах файлов сборок
`Ucim.ipa`/`Ucim.app`). `su.ivcs.kmp` — только в протухшей шапке `locators/Android/README.md:4` и в
мёртвом `L_APP_ROOT`. Путь — `tests/pages/locators/Android/README.md`, уровня `KMP/` нет. iOS
accessibility id — ровно один на весь пакет, и тот на литерале `'https://'`
(`locators/iOS/auth.py:19`).

**Плюс роль противоречит нашему же канону.** `runner.md:3-5` — «деления на поколения клиента у
раннера нет»; `rules/page-objects.md:10-12` — «Разделения на поколения клиента … в коде больше нет».

**Грозит.** Роль-фрагмент always-on в `AGENTS.md`/`CLAUDE.md`, читается первым и уводит агента искать
штатные accessibility id на iOS и второе поколение клиента.

---

## 17. СРЕДНЕ · `rules/page-objects.md` без front-matter не подхватится их механикой правил

**Наш текст.** `stack/rules/tests.md:1-6` несёт маску путей
(`--- paths: - "tests/test_*.py" ---`); `stack/rules/page-objects.md:1` начинается сразу с
заголовка — front-matter нет.

**В их коде.** Оба файла правил несут маску: `.agents/rules/tests.md:1-6` и
`.agents/rules/page-objects.md:1-6` (`"tests/pages/**/*.py"`).

**Грозит.** Самый нужный при правке page-слоя файл не привязывается к путям и подтягивается только
вручную; плюс два наших rules-файла оформлены по-разному без причины.

---

## 18. СРЕДНЕ · Шаблон теста не берёт `server`, без которого их флоу входа не начинается

**Наш текст.** `test-templates.md:26-52` — скелет достаёт `lang`, `platform`, `creds`, `driver`.

**В их коде.** `tests/test_one_login.py:24` — дополнительно `server =
setup_aut['setup']['config']['server']`, и он передаётся первым же вызовом `auth_page.set_server()`.
При этом у стенда `prod` ключа `server` на верхнем уровне НЕТ (`config.yaml:1-8` — только
`credentials.user1.server`), то есть тест по нашему шаблону с `--env prod` даёт `KeyError: 'server'`.

**Грозит.** Тест, написанный строго по нашему шаблону, не доходит до первого действия. Побочно — ещё
один довод против `prod`, которого в каноне нет.

---

## 19. МЕЛКО · «`--from-pipeline` нигде не читается» опровергается грепом

**Наш текст.** `runner.md:103` — «`--from-pipeline` … в инсталляции-источнике **нигде не читается**».

**В их коде.** `tools/aut/options.py:4,25` — `from autocore.test.config import get_from_pipeline` и
`if get_from_pipeline():`. Функционально это мёртвый веб-остаток (`generate_chrome_options` никто не
вызывает, а внутри неё неопределённое имя `OPTIONS_CHROME_IN_DOCKER` — `NameError` при вызове), но
формулировка проверяется грепом за секунду и рушит доверие к соседним утверждениям того же абзаца
(про `--no-exim-reset`/`--no-db-restore`, которые как раз верны).

---

## 20. МЕЛКО · `--lang en` подан как рабочее измерение; практически он мёртв

**Наш текст.** `rules/tests.md:86-92` — «Тест, исполнимый только на английском UI, помечается
`lang_en`»; `failure-taxonomy.md:31-35` — класс `LANG_MISMATCH`.

**В их коде.** `tools/i18n/strings.py` — 126 ключей, из них 86 значений `'TODO'`.
`tools/i18n/__init__.py` — `get_str` вернёт литерал `TODO`, а не бросит.

**Грозит.** Прогон на `--lang en` не даст `LANG_MISMATCH`, а тихо пойдёт искать элементы с видимым
текстом «TODO» — падение будет выглядеть как `LOCATOR_STALE`.

---

## 21. МЕЛКО · `xfail_strict` не покрыт нашим каноном

`pytest.ini:2` — `xfail_strict = true`; их `.agents/rules/tests.md:86-88` описывает режим и правило
«reason всегда ссылается на issue». Наш `rules/tests.md:100-107` §«Инварианты» и `test-templates.md`
про xfail молчат. Пробел, а не ошибка: сигнальные красные — штатный исход их конвейера
(`metrics-README.md`, `result: red-product-bug`), и без правила агент выберет форму сам.

---

# Проверено и совпало

- **`--env` без дефолта, читается в СЕССИОННОЙ фикстуре** — `conftest.py:169` (`addoption` без
  `default`), `conftest.py:39-44` (`scope='session'`, `get_config()[option.env]`). Вывод «падает весь
  прогон, а не тест» верен.
- **Список окружений и `prod` первым** — `config.yaml`: `prod`, `at-develop1`, `at-develop3`,
  `at-develop5`, `at-develop6`, `at-develop7`.
- **Договорный стенд `at-develop3`** — `README.md:78-79`, `AUT_OVERVIEW.md` §«Снимок карты».
- **Три опции устройства без дефолтов, проверок на пустоту нет** — `conftest.py:174-176`,
  `prepare_driver` (`:54-74`).
- **`--ver` = `Version2`, `--lang` = `MC.LANG_RU`** — `conftest.py:170-171`, `consts.py:16`.
- **`--no-exim-reset` / `--no-db-restore` объявлены и мертвы** — `conftest.py:177-178`, потребителей
  нет.
- **Маркеров ровно 11 и группировка** — `pytest.ini:8-21`, совпадает построчно.
- **`--strict-markers`, `--tb=native`, `--showlocals`, `--alluredir=./allure-raw`, `-v`,
  `-p no:cacheprovider` в `addopts`** — `pytest.ini:7`.
- **Плагина рандомизации нет** — `requirements.txt`.
- **`PYTEST_DISABLE_PLUGIN_AUTOLOAD=1` + `-p allure_pytest.plugin` обязательны, симптом
  `Empty suite` + trio** — `README.md:139-144,266-269`.
- **`option.allure_report_dir` читает conftest** — `conftest.py:126`.
- **Локальные координаты `local:15555` / `local:25555`** — `README.md:94-99`.
- **`mobile_env.yaml`: по три устройства на платформу со своими `device_port`/`system_port`,
  читает многодрайверная фикстура** — `mobile_env.yaml`, `conftest.py:82-98`.
- **CI — два `echo`, рабочая команда закомментирована** — `.gitlab-ci.yml`.
- **Форма теста**: `from tests import pages`, `log.tc('<id>')`, `teardown.append(lambda:
  driver.quit())`, `setup_aut['setup']['cmd']['lang'|'device_os']`,
  `setup_aut['setup']['config']['credentials'][…]`, `pages.get_page(platform, …)(driver, lang=lang)`,
  два платформенных маркера, никаких `if device_os` в теле — `tests/test_one_login.py` совпадает с
  нашим скелетом построчно.
- **`@allure.id` рядом с `log.tc`, id в имени функции не требуется** — `tests/test_one_login.py:10,27`.
- **Page-слой на Selenium-совместимом API** — `tests/pages/base.py:1-13,35-77`.
- **Android: Compose, вырожденное дерево, порядок `id` → `content-desc` → текст через `i18n` →
  `bounds`; цитата README воспроизведена точно** — `locators/Android/README.md:10-17`.
- **Шапка того README протухла (`su.ivcs.kmp`, сборка 1.2.10)** — `:4,6`.
- **Mail/Calendar роняли сборку на `emulator-5554`, локаторов нет** — `:24-26`.
- **`bounds` объявлены временной мерой самим автором** — `:27-28`.
- **iOS: предикат по типу + текст из `i18n`, `ACCESSIBILITY_ID` ровно один раз и на `'https://'`** —
  `locators/iOS/common.py:12-26`, `locators/iOS/auth.py:19`.
- **`bounds` в iOS-пакете нет ни одного** — grep пуст.
- **Нарушение i18n на iOS: заголовок логина XPath по русскому литералу** —
  `locators/iOS/auth.py:13-15` (`//XCUIElementTypeOther[@name="баннер"]`).
- **Логин Android внутри WebView, узлы `username`/`password`/`kc-login`/`kc-page-title`** —
  `locators/Android/auth.py:42,51-71`.
- **Обратная ошибка: `WEBVIEW_*` в списке ≠ экран в WebView; PIN и защита входа — `NATIVE_APP`;
  на iOS `ASWebAuthenticationSession` публикуется в `NATIVE_APP`** — `AUT_OVERVIEW.md:32,70`.
- **Системный alert на iOS не виден в source, берётся `driver.switch_to.alert`** —
  `AUT_OVERVIEW.md:67-68`, `tests/pages/auth.py:176-180`.
- **Диалог разрешений Android принадлежит другому пакету** — `AUT_OVERVIEW.md:36-37`,
  `locators/Android/passcode.py:52-57`.
- **У поля ввода сервера на iOS нет ни identifier, ни name, ни label** — `AUT_OVERVIEW.md:60`,
  `locators/iOS/auth.py:22-26`.
- **Пустые `chunks/` и `browser/`, реестра `get_chunk` нет** — оба `__init__.py` пусты.
- **Идентификатор один на обе платформы** — `tools/aut/__init__.py:31,57-59`.
- **Живая карта AUT от 2026-08-04, сборки 1.3.5, Appium 3.6.0, TC-468 пройден на обеих платформах** —
  `AUT_OVERVIEW.md:5-12`; наша оговорка «у них прогоны есть, у нас нет» честна.
- **`allure-raw` не коммитить, доступ TestOps через `TESTOPS_*` / gitignored `secrets.yaml`** —
  `.gitignore:3,36`, `secrets.yaml.example`.