---
title: explore-ds-sheremetev-history-2026-07-27
type: note
status: draft
created: 2026-07-27
tags:
- board
- design-system
- explore
- history
permalink: tacticum/00-board/explore-ds-sheremetev-history-2026-07-27
archived-at: 2026-08-04 10:01
---

# Разведка: подход Шереметьева + история наших работ по ДС

Разведка read-only, ничего не правил. Две части: **А** — что известно про подход Шереметьева
(с жёстким разделением «сказано голосом» / «проверено в коде и данных»), **Б** — хронология
наших работ по ДС и список отложенного.

**Как искал:** `grep -ri` по всему `/Users/bubblemac/tacticum-vault` (варианты «Шереметьев /
Шереметев / Sheremetev / Sheremetyev»), `textutil` по RTF-транскриптам в `90-Materials/`,
`grep` по `/Users/bubblemac/tacticum` (репозитории + датасеты `helm/data/real/*`), разбор
repomix-снимков `iva-one.xml` и `kmp.xml` (снято 2026-07-03), чтение `tacticum-dev` напрямую.
**MCP basic-memory в моём наборе инструментов нет** — поиск по памяти сделан грепом по файлам vault.

---

# ЧАСТЬ А — подход Шереметьева

## A0. Короткий ответ

В **памяти** про подход Шереметьева известно **ровно 4 факта** — все со слов руководителя, ни один
не подтверждён нами напрямую (мы с Шереметьевым не общались, ОС от Димы в vault не приехала).

Но подход **проверяем по данным и коду**: Шереметьев — реальный человек с корпоративной почтой,
192 коммита в `iva-one`, и он **автор новой IVA Design System в вебе**. Её устройство читается по
repomix-снимку `iva-one` (2026-07-03) — это уже не догадки, а код. Ниже разведено построчно.

## A1. Что СКАЗАНО на созвонах (4 факта, не проверено нами)

| # | Факт | Источник (файл: строка), дата |
|---|---|---|
| 1 | «Вчера с Шереметьевым общались, он показал, как у них это сделано. Оказалось, что у разных команд по-разному» — говорит руководитель; инициатором названы **Савицкий** и он сам | `90-Materials/Созовон 24:07 10-30.rtf` (сырой транскрипт), 2026-07-24 |
| 2 | Фронт-команды применяют дизайн по-разному: кто выгружает CSS из Figma, кто держит **rulebook (элементы / экраны / композиции) — «как у Шереметьева»** | `01-Sessions/call-2026-07-24-10-30 — дизайн-система…md:31`; `11-Directions/Направление- Единая дизайн-система Tacticum (Figma-токены → код).md:26` |
| 3 | Задача: доработать **нашу** ДС **по наработкам Шереметьева** (и по просьбе Савицкого), свести всех на одну. Диме поручено: (1) созвон с Шереметьевым → собрать ОС, (2) ТЗ → Александру Шульге | `Направление- Единая дизайн-система…md:30,33`, 2026-07-24 |
| 4 | «Возможно, **подход Шереметьева решит ситуацию — я не знаю, нужно пробовать**; нужно разобраться, что работает нормально, а что нет». Разбираться **завтра может начать Солонко** | `01-Sessions/call-2026-07-27-1130 — приоритет дизайн…md:64-66`, 2026-07-27 |

Контекст задачи из того же созвона 27.07: у KMP «дизайн собирали по скриншотам, натягивали на
макеты — получалось так себе, потому что **нет подходов**»; формулировка проблемы —
«промышленное решение есть, дизайн плохой», цель — унифицировать работающие подходы легаси-команды
и перенести на KMP (`call-2026-07-27-1130…md:57-65`).

**Открытый вопрос (не решён):** кто разбирается с подходом — мы или Солонко.
Зафиксировано в `00-Board/call-review-2026-07-27.md:73-77` (п.10, ждёт решения Президента).

**Чего в словах НЕТ:** ни одного технического слова про сам подход. «Rulebook элементы/экраны/
композиции» — единственная техническая деталь, и она пересказ руководителя, не Шереметьева.

## A2. Кто это по данным (проверено, датасеты helm)

- `helm/data/real/manual/teams.csv` — **Шереметьев Александр Юрьевич**, `a.sheremetyev@iva.ru`,
  Frontend-developer, **Senior**, команда **Трек А (FE)**, продукт «One 1.5 / One 1.6 / путь к
  One 2.0», проект Jira **IVAONE**. В колонке `grade` стоит `Out 2` — значение не расшифровано,
  проверять у владельца данных.
- `helm/data/real/identity/identity_map.csv` — git-имена `Alexander Sheremetev` / `asheremetev`,
  **106 коммитов** по идентити-карте, источники `d10+git+jira`, страниц Confluence — 0.
- `helm/data/real/git/commits.csv` — **192 строки** с его авторством, **все в репозитории
  `iva-one`**, период **2025-03-04 … 2026-06-09** (граница — дата снятия датасета).
- `helm/data/real/git/merge_requests.csv` — 7 MR в `iva/one/web/iva-one`, все merged в `master`,
  свежие: `!2153` (2026-07-02, DsToast→IvaToastService), `!2151` (2026-07-02, DsTooltip→IvaTooltip).
- `helm/data/real/jira/jira_issues.csv` — 6 задач на нём, все под эпиком **IVAONE-6743
  «[WEB] Новая дизайн система»** (Design Task, в работе, автор `anton.smirnov`, дизайнер
  `m.efremova`, заведён 2025-07-07). Задачи вида «Migrate DsX to new IVA Design System component»,
  часть закрыта 30.06–02.07.2026, `IVAONE-12072` — **переоткрыт**.
- **Страниц вики за ним не найдено:** `grep -i sheremet helm/data/real/wiki/pages_index.csv` — пусто.
  То есть **rulebook как документа в Confluence нет** — документация живёт в коде (см. A3).

## A3. Что такое «подход Шереметьева» ПО КОДУ (проверено, снимок iva-one 2026-07-03)

Источник: `/Users/bubblemac/tacticum/helm/data/real/git/repomix/iva-one.xml` (repomix `--compress`,
снято 2026-07-03, см. `repomix/README.md`). Библиотека — `libs/design-system`, пакет
`@iva/design-system`, **2188 файлов**.

**1. Отдельная публикуемая библиотека, а не папка в приложении.**
Публикуется в корпоративный npm-registry `npm-repository.hi-tech.org:1234`, ставится
`yarn add @iva/design-system`, требует Angular 21+ (`libs/design-system/README.md`).
CI выделен: `devops/design-system/ci.yml`; MR от 2026-04-22 — «ci(devops): consolidate
design-system CI into **iva-ui-libs**» (т.е. библиотека выезжает в отдельный репозиторий).

**2. Структура библиотеки** (по путям в снимке):
`src/lib/components` 1438 файлов · **`src/lib/figma-integration` 448** · `behaviors` 117 ·
`colors` 26 · `styles` 22 · `elements` 15 · `typography` 13 · `docs` 12 · `directives` 12 ·
`theme` 7 · `tokens` 7 + `.storybook/`.

**3. Figma → код: три отдельных генератора, вход ручной, выход — сгенерённые файлы.**
Каталог `src/lib/figma-integration/`:
- **Цвета** — `colors/color-tokens.mjs` + `.sh`/`.ps1`. Вход: `color-tokens.json`, который
  **выгружается из Figma вручную плагином `var-exporter`**. Выход: `colors/generated/
  color-tokens-{light,dark}.scss` и `.ts`. Заголовок сгенерённого файла:
  `// Do not edit this file manually. Generated by figma-integration.`
  Токены **семантические, вложенные**: `bg: (accent-primary, badge: (green…)), text:…`.
- **Типографика** — `typography/typography.mjs`: **ходит в Figma API** и забирает узел `4151:90`,
  вытаскивает font-family/weight/size → `typography/generated/typography.scss` + `.ts`.
- **Иконки** — `icons/`: SVG кладут руками из Figma в `source/monochrome` (цвет снимается) или
  `source/colored` (цвет сохраняется), `optimize-icons.mjs` (svgo) → `icon-components.mjs`
  генерит Angular-компоненты по шаблону `icon.component.tpl` + `icon-list.ts` для Storybook.
  В снимке **429 исходных SVG**.
- Общий слой `shared/api.js` — тонкий клиент Figma REST (`getDocument/getNode/getNodes/
  getStyles/getSvgImageUrl`).

**4. Темы — рантайм, без пересборки.** Два бандла `colors/light.scss` и `colors/dark.scss`
подключаются `inject: true`, переключает `IvaColorSchemeSwitcher` через атрибут
`data-iva-scheme` на `<html>`; провайдер `provideIvaColorScheme({defaultScheme: 'auto'})`.

**5. Публичный SCSS-API один.** `@use 'lib/main'` + `main.ivaGetColor(text, primary)`,
`main.ivaFontPreset(body, m, medium)`. Хардкод цвета в компоненте не нужен по определению.

**6. Документация = Storybook, живёт в репозитории.** `https://iva-ui.ivcs.su/`.
В `src/lib/docs/guides/` лежит именно то, что руководитель назвал «rulebook»:
`development/naming.mdx` (правило имён `Iva` + Base + Modifier, приоритеты «Consistency >
Discoverability > Readability > Brevity»), `development/installation.mdx`, `development/release.mdx`,
`development/custom-form-controls.mdx`, `storybook/story-standards.mdx`, `component-documentation.mdx`,
`pseudo-states.mdx`, `tags.mdx` + шаблоны сторей.

**7. Миграция легаси идёт задачами, а не разом.** `src/lib/docs/design-system-comparison.md` —
таблица «Design System (`libs/design-system`) vs UI-Kit (`libs/ui-kit`)» по компонентам со
статусами ✅/❌/🔄. По коммитам 2026-05…07: `ds-menu → iva-menu`, `ds-avatar → iva-avatar`,
`DsToast → IvaToastService`, `ivaGetColor` вместо `dsGetColor`; ветки вида
`feature/IVAONE-12066-migrate-ds-menu-to-iva-menu`, часть MR цепочкой один в другой.

**Итого подход = 6 механик:** публикуемый npm-пакет · генераторы Figma→код по трём осям
(цвет/типографика/иконки) с ручным входом · семантические токены light/dark и рантайм-переключение ·
единый SCSS-API вместо хардкода · Storybook как единственная документация с писаными конвенциями ·
пофайловая миграция легаси по таблице сравнения.

## A4. Что из этого У НАС уже есть — с путями (проверено)

**Главное: мы уже привязаны именно к ДС Шереметьева, а не к абстрактной «веб-ДС».**
`/Users/bubblemac/tacticum/tacticum-dev/design-systems/iva-web/tokens.json`,
`$extensions."dev.tacticum.code-bindings"` содержит:
`{"library": "@iva/design-system", "components_root": "libs/design-system/src/lib/components",
"repo": "https://git.hi-tech.org/iva/one/web/iva-one", "extracted_from_commit": "daf1424ad 2026-07-20",
"storybook_base": "https://iva-ui.ivcs.su/"}`.

- `design-systems/iva-web/design-system.yaml` — версия **0.3.0**, 49 биндингов, у 32 проставлен
  стабильный `figma_key`, есть алиасы расхождений имён (Radio↔Radiobutton, Tabs↔Tab Group,
  Scroll Area↔Scroll, Select↔Dropdown).
- `design-systems/iva-mobile/design-system.yaml` — **0.2.0**, 34 компонента → Compose Multiplatform
  `su.ivcs.messenger.designsystem`, коммит `8f65eea2ff 2026-07-23`, **`figma_key` pending**.
- Ещё две ДС в репо: `iva-rn`, `tacticum-web`.
- Сборка: `apps/backend/scripts/merge_iva_tokens.py` — **профили захардкожены**, поддерживаются
  ровно `iva-web` и `iva-mobile` (`PROFILES` на строке 48, `choices=sorted(PROFILES.keys())`
  на 251). Вход — экспорт Tokens Studio (ADR `docs/adr/0027-tokens-studio-import-bootstrap.md`).
- Навыки профилей: `templates/tacticum-ui-base/ingredients/skills/` — `design-system-discovery`,
  `design-token-usage`, `ui-mockup-match`, `pin-ui-pipeline-check`;
  `templates/iva-web-brownfield/ingredients/skills/` — `angular-ds-component-authoring`,
  `angular-ds-component-usage`, `iva-core-design-system`, `design-token-usage`;
  `templates/iva-kmp-development-base/ingredients/skills/` — `web-to-kmp-screen-port`,
  `web-to-kmp-source-reference`.

**Совпадение подходов:** его конвенции авторинга компонента ≈ наш навык
`angular-ds-component-authoring` (анатомия, только токены, полнота вариантов, Storybook);
его таблица DS↔UI-Kit ≈ наш словарь `code-bindings`; его правило миграции батчами ≈ наше
правило Сц.3.

## A5. Что у нас сделано ИНАЧЕ и где реальный разрыв (проверено)

**Разрыв 1 — вход из Figma у нас другой.** У него: `var-exporter` (цвета) + Figma API по узлу
(типографика) + ручной экспорт SVG (иконки), выход — генерённые SCSS/TS **внутри библиотеки**.
У нас: плагин **Tokens Studio** → `merge_iva_tokens.py` → `design-systems/<ds>/tokens.json` для
сервера ADP. Это **два независимых маршрута из одной Figma**, они не связаны и могут разъезжаться.

**Разрыв 2 — KMP. Здесь и есть та самая дыра, про которую говорил руководитель.**
По снимку `kmp.xml` (2026-07-03):
- ДС существует: модуль `core/design-system`, пакет `su.ivcs.messenger.designsystem`,
  **251 файл**, `element/` — 134 файла (buttons, dialog, drop_down_menu, toast, text_field,
  navigation_bottom_bar, shimmer…), `theme/` — 8 файлов.
- **Файлов с `figma` в пути — 0.** Никакого `figma-integration`, ни одного генератора.
  Единственные следы Figma в KMP — **ссылки в комментариях** к отдельным векторам
  (`figma.com/design/cFPZlcH7oqufG78KHVTgdG/Web-Mail?node-id=…`, `…/MOBILE?node-id=155-6997`).
  Это ровно то, что руководитель назвал «собирали по скриншотам».
- Цвета — **написаны руками** в Kotlin: `theme/color/IvaMessengerColors.kt`, `StandardColors.kt`,
  `DarkColors.kt`, `AppColors.kt`. Имена **палитровые** (`primary80`, `oPrimary`, `secondary20`,
  `critical70`), у веба — **семантические** (`bg.accent-primary`, `text.primary`).
  Сопоставление палитры и семантики автоматически не выводится.
- Документация KMP-ДС — RE-сгенерённый навык в их же репо:
  `AI common/skills/compose-ui-patterns-su-ivcs-messenger-kmp/references/03-design-system-iva.md`
  (описывает структуру пакетов, `IvaTheme`, Material3, правило `expect/actual`). Это описание
  того, что есть, а не конвенция «как делать» — аналога его `docs/guides/` в KMP нет.

**Разрыв 3 — Storybook-эквивалента в KMP нет.** У веба живая витрина `iva-ui.ivcs.su` с
конвенциями и pseudo-states; в KMP витрины в снимке не нашёл.

**Вывод разведки (не решение):** «перенести подход Шереметьева на KMP» механически = завести в
KMP (а) генерацию цвет/типографика/иконки из Figma, (б) семантический слой токенов поверх
палитры `IvaMessengerColors`, (в) писаные конвенции и витрину. Сегодня в KMP нет ни одного из трёх.

## A6. Чего НЕ найдено (искал, не нашёл)

- **ОС от Димы после созвона с Шереметьевым** — в vault нет. Искал: грепом по `01-Sessions`,
  `11-Directions`, `00-Board`, `12-Features`; в направлении задача Диме стоит с 24.07 без отметки
  о выполнении.
- **ТЗ Шереметьева / его документ про подход** — нет ни в vault, ни в `tacticum-dev`.
- **Страницы Confluence за его авторством** — нет в `pages_index.csv`.
- **Упоминания Шереметьева в коде/доках `tacticum-dev`** — нет (только через `@iva/design-system`
  в code-bindings, без имени).
- **Прямой контакт (кроме почты `a.sheremetyev@iva.ru`)** — нет.

---

# ЧАСТЬ Б — история наших работ по ДС

## Б1. Хронология

**До 24.07 (фон).** В `main` уже лежали `#132` iva-web DS 0.3.0 (стабильные `figma_key` в
code-bindings) и `#134` iva-mobile DS 0.2.0 (KMP code-bindings + figma quickstart) —
`Направление- Единая дизайн-система…md:37`. Последнее изменение `design-systems/` — 23.07,
Дмитрий Солонко (`…md:50`).

**2026-07-24 — заведено направление.** По созвону 10:30: `11-Directions/Направление- Единая
дизайн-система Tacticum (Figma-токены → код).md` (статус `incubating`), change management —
на Александра Шульгу.

**2026-07-24, день — ТЗ#1 Сц.4 (перенос экранов web → KMP), полный цикл за сутки.**
План: `91-Archive/plans/План ТЗ-1 Дизайн-процесс Figma↔код — Сц.4 перенос форм one→kmp (lead-ds).md`.
Доска (время файла = время работы):
- 15:27 `recon-ds-skill-format-web-to-kmp` — разведка формата навыка.
- 15:33 `impl-ds-web-to-kmp-skill-skeleton` → 15:36 `gate-ds-web-to-kmp-phase1` — **PASS** (скелет).
- 16:23 `impl-ds-web-to-kmp-relink` → 16:27 `gate-ds-web-to-kmp-relink` — **PASS**: пере-связка
  на реальные in-repo навыки KMP (`recon-ds-inrepo-skills-catalog`, снято с `adp:/srv/iva/repos/kmp/
  AI common/skills/`).
- 16:42 `impl-ds-web-to-kmp-variant1` → 16:48 `gate-ds-web-to-kmp-variant1` — **PASS 5/5**:
  решение Президента «дом доставки = Вариант 1», репо-нативный путь мобильной команды.
- 17:04 `impl-ds-source-reference-skill` → 17:07 `gate-ds-source-reference-skill` — **ПРОШЛО**:
  второй навык `web-to-kmp-source-reference` (ось-2, два дерева: source read-only, target write).
- 17:37 `impl-ds-skill-refine-from-pilot` → 17:40 `gate-ds-skill-refine-from-pilot` — **PASS**:
  4 правки навыка по итогам сухого пилота.
- Пилот: `Пилот Сц.4 — ContactDetail (СУХОЙ прогон web-to-kmp-screen-port).md` — 5 расхождений
  с веб-версией, ported-код остался черновиком (валидация, не доставка).
- 18:11 `gate-bundle1-git-final`, 18:23 `impl-ds-critic-fixes` + `critic-bundle1`,
  18:27 `gate-bundle1-critic-fixes`; тело PR — `pr-bundle1-ds-web-to-kmp-body`.
  **Бандл #1 смержен в main 2026-07-24 18:54 Президентом** (заголовок в плане).
- Ось-2 (multi-repo для lead-modes): 18:17 `impl-axis2-kmp` (6 файлов в `iva-kmp-brownfield`) →
  18:21 `gate-axis2-kmp` — **GO**; исследование — `explore-axis2-brownfield`, `explore-axis2-gapmap`,
  критик — `critic-fidelity-axis2`.
- 19:39 — **Президент расширил предел ТЗ#1**: после Сц.4 доделываем Сц.1/2 → Сц.3 → ось-1.
- Вечер 24.07 — PR-A (Сц.1/2) → регрессия и CI-fix (21:0x) → PR-B (G5, `impl-g5-mockup-figma`
  20:50) → PR-C (ось-1, `impl-g7-dsdiscovery` 22:06, coverage-guard).
- 23:20 `map-us4-Ebrd-kmp-divergence` — карта расхождений канона `brd-authoring` и KMP-копии
  (что не затирать, что донести).

**2026-07-25.**
- 00:0x — **PR-C запушен, ТЗ#1 BUILD-COMPLETE** (`gate-prc-final`, `gate-prc-final-regate` 01:40).
- 00:29 `final-fidelity-tz1` — независимая сверка полноты: все 4 сценария покрыты, «сверх ТЗ» нет,
  один дефект (навык врал про недоступность числовой приёмки) исправлен до прода.
- Фаза 2, словарь: `prep-ds-phase2-kmp-dictionary` → `phase2-provisional-iva-web-dictionary`
  (status `resolved`, 32 ключа) → `impl-ds-dictionary-final` → `gate-ds-dictionary-final` — **PASS**
  (гейт отдельно отметил: figma_key достоверны, не выдуманы).
- 20:13 `gate-bundle1-fidelity` — FIDELITY-сверка бандла 1 с ТЗ.
- **Прод-сид 25.07** вместе с ТЗ#2/#3 (`prod-deploy-3tz-done-2026-07-25`, verify зелёный).
- 18:50 заведено досье `12-Features/tz1-figma-ds-process.md` — статус **«ждёт живой проверки»**.
- 18:55 заведён `12-Features/tz1-figma-ds-handoff-ostatok.md` — остаток другим командам.

**2026-07-26.** Подтверждений от владельцев остатка нет (`tz1-figma-ds-handoff-ostatok.md:75`).

**2026-07-27.** Три созвона. Проверен фактический маршрут «Figma → профиль» и записан в направление
(`…md:40-52`): 4 шага, 3 ручных, профили захардкожены. Руководитель требует **схему обновления**
ДС, а не разовый перенос; приоритет назван вслух — **дизайн**. Развилки — `call-review-2026-07-27`
(п.1 дом выгрузки, п.10 кто разбирается с Шереметьевым).

## Б2. Что отложено и кому передано

Источник — `12-Features/tz1-figma-ds-handoff-ostatok.md` (5 пунктов, у каждого назван владелец):

| # | Что | Кому передано | Состояние на 27.07 |
|---|---|---|---|
| 1 | Сц.3 — фактическая миграция `iva-one`: ~72 легаси-файла с `ui-kit`/`dsGetColor` → `@iva/design-system`/`ivaGetColor` (556 экранов уже на новом), батчами, приёмка как Сц.2 | **команда iva-one** | не подтверждено. **Пересечение с A2/A3: это ровно та работа, которую Шереметьев и делает задачами эпика IVAONE-6743** |
| 2 | `iva-core` — серверная ДС + словарь code-bindings для VCSWEB-поверхности | **server/RE + DS-команда** | template-часть в проде (PR-C), рантайм-резолв ждёт их |
| 3 | Словарь v2: поля `mdx_path/host/requires/slots/import/category` + **авто-пересборка из кода iva-one + CI-проверка** существования компонентов/пропсов + gap-отчёт «есть в Figma — нет в коде» | **RE-конвейер** | не подтверждено; сейчас словарь v1 собран руками |
| 4 | 17 пустых `figma_key` + подтверждение auto-layout фреймов пилотных экранов | **дизайнеры / владелец ДС (Д. Солонко)** | не блокер (матч по имени работает) |
| 5 | Внутренние ADR/процесс: канон `tacticum-ui-base` «Iva DS unified across surfaces» → surface-split (ADR-0056/0059); `_mirrors` для DS-навыков (`design-system-discovery` — 7 копий/6 хэшей, CI-лок только на паре) | **решение Президента** | открыто. **Прод-риск:** правка в одной копии не доезжает до других |

Отдельно отложено внутри ТЗ#1 (`tz1-figma-ds-process.md:44-51`):
- **рантайм-проверка** собранного экрана — при интеграции у команды мобильного клиента (наш
  контур read-only, чужой тулчейн);
- **хвосты словаря** — полные сигнатуры компонентов, промоция в серверные связки.

## Б3. Реестр board-заметок по ДС (для навигации)

`00-Board/`: `recon-ds-skill-format-web-to-kmp` · `recon-ds-inrepo-skills-catalog` ·
`impl-ds-web-to-kmp-skill-skeleton` · `gate-ds-web-to-kmp-phase1` · `impl-ds-web-to-kmp-relink` ·
`gate-ds-web-to-kmp-relink` · `impl-ds-web-to-kmp-variant1` · `gate-ds-web-to-kmp-variant1` ·
`impl-ds-source-reference-skill` · `gate-ds-source-reference-skill` ·
`impl-ds-skill-refine-from-pilot` · `gate-ds-skill-refine-from-pilot` · `impl-ds-critic-fixes` ·
`prep-ds-phase2-kmp-dictionary` · `phase2-provisional-iva-web-dictionary` ·
`impl-ds-dictionary-final` · `gate-ds-dictionary-final` · `pr-bundle1-ds-web-to-kmp-body` ·
`critic-bundle1` · `gate-bundle1-git-final` · `gate-bundle1-critic-fixes` · `gate-bundle1-fidelity` ·
`Пилот Сц.4 — ContactDetail (СУХОЙ прогон web-to-kmp-screen-port)` · `explore-axis2-brownfield` ·
`explore-axis2-gapmap` · `impl-axis2-kmp` · `gate-axis2-kmp` · `critic-fidelity-axis2` ·
`spec-axis2-workflow-for-lead-modes` · `map-us4-Ebrd-kmp-divergence` · `impl-g5-mockup-figma` ·
`impl-g7-dsdiscovery` · `final-fidelity-tz1` · `call-review-2026-07-27`.

Архив: `91-Archive/plans/План ТЗ-1 Дизайн-процесс Figma↔код — Сц.4 перенос форм one→kmp (lead-ds).md`
(полная хроника исполнения). Других ДС-материалов в `91-Archive` нет — искал `ls | grep -iE
"ds|figma|design|kmp"` по `inbox/`, `concepts/`, `stubs/`, `qa/`.

---

## Ограничения разведки

- Снимки `iva-one.xml` / `kmp.xml` статичны на **2026-07-03**; коммиты Шереметьева в датасете
  обрываются на **2026-06-09**, MR — на **2026-07-02**. Что в `iva-one` изменилось за июль —
  по этим данным не видно.
- Repomix снят с `--compress`: у `.ts`/`.kt` в снимке сигнатуры без тел. Тексты `.md`/`.mdx`/
  `.scss` — полные, всё цитируемое выше взято из них.
- Значение `grade: Out 2` у Шереметьева не расшифровано — если это «уволен/на выходе», у задачи
  «перенять подход» другой срок жизни. **Проверить у владельца данных.**