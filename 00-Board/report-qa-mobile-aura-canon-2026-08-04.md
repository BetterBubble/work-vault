---
title: Канон мобильного стека переписан под репозиторий заказчика (04.08)
type: report
status: current
date: 2026-08-04
repo: tacticum-dev
branch: feat/qa-mobile-aura-canon
project: tacticum-dev / профили / QA
tags:
- board
- qa
- mobile
- appium
permalink: tacticum/00-board/report-qa-mobile-aura-canon-2026-08-04
---

# Канон стека приведён к их репе: 722 → 970 строк

Ветка `feat/qa-mobile-aura-canon`, коммит `f62d234` от `origin/main`. 13 файлов, +585/−180.
Не пушено. Версии: лейн `iva-pytest-appium-autotest-base` 0.1.0 → **0.2.0**, роль
`iva-role-qa-mobile` 0.1.0 → **0.2.0** (бамп вынужденный по гейту дисциплины версий).

## Что исправлено

**Идентификатор** — `su.ivcs.aura` на обе платформы, пара `ucim`/`kmp` убрана как различитель.
Проверено в шести местах их кода. Попутно: обе ветки `if/else` для `bundleId` идентичны —
**ветвление мёртвое**, и дискриминатор `APP_NOT_STARTED` «не то поколение» больше не работает.

**Ось адресации переформулирована как уточнение**, а не снос: «две платформы одного приложения,
у каждой свой пакет локаторов и свой приоритет».

- Android: приоритет `id → content-desc → точный текст через i18n → bounds` жив дословно,
  «rendered mostly through Compose» на месте. Живых bounds-локаторов шесть.
- iOS: проверено по коду — `type == 'XCUIElementTypeStaticText' AND name == '<i18n>'`, поля ввода
  через `XCUIElementTypeTextField`. `ACCESSIBILITY_ID` во всём iOS-пакете **один раз**, и тот
  литерал `'https://'`. `bounds` на iOS нет вовсе.

**Новый вывод, которого не было ни у нас, ни у них:** опора у обеих платформ в итоге одна —
**видимый текст через i18n**, значит `--lang` ломает адресацию на обеих сразу.

**Прочее:** битые ссылки на `locators/Android/KMP/README.md` починены; маркеров 11 (цифры «12» в
профилях и не было — она жила только в нашей разведке 30.07); мёртвые `--no-exim-reset` /
`--no-db-restore` названы мёртвыми; `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1` и `-p allure_pytest.plugin`
внесены как обязательные с объяснением, лишний `--alluredir` убран.

**Оговорка про прогоны переписана, не вычеркнута**, одной формулой везде: у команды прогоны есть
(карта AUT 04.08, сборки 1.3.5, `at-develop3`, Appium 3.6.0, TC-468 на обеих платформах), у нас —
ни одного. Десктоп не тронут.

**Стенд и координаты** внесены (`at-develop3`, `local:15555` / `local:25555`), но правило
переписано в три ступени: стенд из задачи → договорный стенд проекта → **стоп и вопрос**. Запрет
«взять первое из `config.yaml`» сохранён — там по-прежнему `prod`.

## Дефекты их репозитория — нести заказчику

1. **Их исследовательский документ описывает предыдущее приложение:**
   `locators/Android/README.md` — `Application id: su.ivcs.kmp`, `Observed build: 1.2.10`, тогда
   как код на `su.ivcs.aura`, а живая карта ведётся по 1.3.5. Правила выбора локатора там
   актуальны, реквизиты — нет.
2. **Живой протухший локатор:** `locators/Android/common.py:12` → `su.ivcs.kmp:id/action_bar_root`,
   при `appPackage = su.ivcs.aura` не найдётся.
3. **Русский литерал мимо i18n:** `locators/iOS/auth.py:15`.
4. **Мёртвые флаги** `--no-exim-reset` / `--no-db-restore`.
5. **Два их внутренних канона приоритета локаторов противоречат друг другу**
   (`craft-stack/recon.md` против `.codex/rules/page-objects.md`), и наш не совпадает ни с одним.
   Взят третий источник — фактический код локаторов, единственный проверяемый.

## Числа

```
check_install_links --profile iva-role-qa-mobile --mode links → OK, 2 пары
check_profile_version_discipline                              → OK, 63 профиля
то же --diff-against origin/main                              → OK, 63 профиля
pytest tests/catalog                                          → 1041 passed, 42 skipped
```

## Оставшийся долг — отдельной задачей

**Роль `iva-role-qa-mobile` правлена только в одной строке.** Остальное в ней по-прежнему ложно:
`manifest.yaml:10-11,19,51` (две репы, пара идентификаторов, ссылка на удалённый README),
`CHANGELOG.md:10-11`, и оба фрагмента точки входа — claude и codex — утверждают, что «роль
обслуживает оба поколения клиента».

Мельче: `allure-raw-parser` знает про скриншоты `Device-{udid}` только наполовину (одиночная
фикстура пишет без udid); `rules/tests.md` запрещает фиксированные `sleep`, а в их
`base.py:58` он есть; фикстур в харнессе пять, а не две, и про `setup_multiple_auts` канон не
знает.

## Связано

[[reshenie-perenimaem-kit-2026-08-04]] · [[recon-mobile-repo-delta-2026-08-04]] ·
[[recon-kit-breykin-2026-08-04]]