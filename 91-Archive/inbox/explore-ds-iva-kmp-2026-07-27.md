---
title: explore-ds-iva-kmp-2026-07-27
type: note
permalink: tacticum/00-board/explore-ds-iva-kmp-2026-07-27
status: draft
created: 2026-07-27
tags:
- board
- design-system
- explore
- kmp
archived-at: 2026-08-04 10:01
---

# explore-ds-iva-kmp — где физически лежит дизайн ИВА и что из этого есть у нас

**От:** explorer (read-only, ничего не правил, на серверы не ходил). **Кому:** тимлид направления ДС.
**Зона поиска:** `~/tacticum`, `~/tacticum-worktrees`, `~/tacticum-vault`, `~/` (maxdepth 2–6).
**Срез:** 2026-07-27, `tacticum-dev` @ `main` = `119ccb0`.

---

## 0. Четыре ответа, которые запрашивал лид

1. **`iva-mobile` code-bindings: 34 компонента, `figma_key: null` — 34 из 34 (100%).** Ожидание лида подтверждено. Литерала `pending` в файле НЕТ, значение именно `null`; слово «pending» стоит только в прозе `design-system.yaml` и в поле `usage` словаря.
2. **Исходников KMP-мессенджера на диске НЕТ.** Известны: URL `https://git.hi-tech.org/iva-m/android/kmp/`, модуль `:core:design-system`, пакет `su.ivcs.messenger.designsystem`, коммит извлечения `8f65eea2ff 2026-07-23`. **Ветка неизвестна, владелец в файле не указан.**
3. **Доступа к репозиториям легаси-фронтов с этой машины НЕТ ни к одному** (Qt desktop / Angular web / iOS / Android). Доступ существует только через зеркало на сервере `adp_emb` — машинный SSH-ключ, режим read-only.
4. **Все четыре ДС-worktree мёртвые** — ветки целиком влиты в `main`, незакоммиченного нет нигде.

---

## 1. KMP-мессенджер и `su.ivcs.messenger.designsystem`

### 1а. Исходников KMP локально НЕТ — что именно искал

| Что искал | Команда | Результат |
|---|---|---|
| каталоги ivcs / messenger / kmp | `find /Users/bubblemac -maxdepth 2 \( -iname "*ivcs*" -o -iname "*messenger*" -o -iname "*kmp*" -o -iname "iva-*" \)` | только наши worktree и `iva-rag*` (это RAG, не мессенджер) |
| пакет `su/ivcs/...` | `find /Users/bubblemac -type d -name designsystem` и `-name ivcs` | **0 совпадений** |
| Kotlin-исходники | `find /Users/bubblemac -maxdepth 6 -name "*.kt"` | 3 файла, все — тест-фикстуры `KB-Brownfield-Bootstrap/tests/fixtures/kotlin/` |
| клон ивовского GitLab | `find ... -name config -path "*/.git/*" | xargs grep -l hi-tech` | **0 совпадений** |
| remote всех git-репо под `~/tacticum` и `~/tacticum-worktrees` | `git remote -v` по 41 репо/worktree | **все** = `github.com/TacticumApps/*` (tacticum-dev, helm, platform, agents, graph-builder, KB-Brownfield-Bootstrap, rag_eval_service, tei_service). Ни одного `git.hi-tech.org` |

Единственный ивовский артефакт на диске: `/Users/bubblemac/tacticum/KB-Brownfield-Bootstrap/IVCS-dump-fixed.zip` — дамп для тестов бутстрапа, не исходники ДС.

### 1б. Что известно о репозитории KMP из наших файлов

Источник — `/Users/bubblemac/tacticum/tacticum-dev/design-systems/iva-mobile/tokens.json`, ключ `$extensions."dev.tacticum.code-bindings".codebase`:

```
repo:                  https://git.hi-tech.org/iva-m/android/kmp/
module:                :core:design-system
package:               su.ivcs.messenger.designsystem
components_root:       core/design-system/src/commonMain/kotlin/su/ivcs/messenger/designsystem
extracted_from_commit: 8f65eea2ff 2026-07-23
```

Ветка и владелец в файле не указаны. По карте доступов (`20-Architecture/Доступы- серверы и репозитории ИВА (adp - teststand - git.hi-tech.org).md`):

- `kmp` — **shared-модуль Kotlin Multiplatform**, зеркало `adp_emb:/srv/iva/repos/kmp`; содержит `AI common/skills/` — 40 репо-нативных навыков команды ИВА; **49 `Iva*` в commonMain** названы там авторитетным набором для словаря.
- `su.ivcs.messenger` — One Android (Compose), `git.hi-tech.org/iva-m/android/su.ivcs.messenger`, ответственный **Легин Денис**.
- **`one-web-kmp` как отдельного репозитория НЕТ** (подтверждено в той же заметке).

**Расхождение URL внутри нашего же репозитория.** `tokens.json` называет кодовой базой `iva-m/android/kmp/`, а навык `templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-source-reference/SKILL.md` (таблица «The two trees») целью записи называет `git.hi-tech.org/iva-m/android/su.ivcs.messenger` при shared `iva-m/android/kmp`. Это два разных репозитория; из какого взят коммит `8f65eea2ff` — по файлам не определяется.

**Гэп покрытия.** В словаре 34 компонента против **49 `Iva*` в commonMain** по карте доступов. Разница ~15 нигде не объяснена.

### 1в. Структура code-bindings — точные числа

Посчитано скриптом по `tokens.json`, не глазами.

| | iva-mobile | iva-web | iva-rn | tacticum-web |
|---|---|---|---|---|
| schema | `tacticum-code-bindings/v0` | `tacticum-code-bindings/v0` | — | — |
| `stack` | `compose-mp` | поля нет | — | — |
| компонентов | **34** | **49** | **0** (нет extensions) | **0** (нет extensions) |
| `figma_key` заполнен | **0** | **32** | — | — |
| `figma_key: null` | **34 (100%)** | **17** | — | — |
| `figma_file_key` | **null** | `AG11paSthGC7zSoovfjip0` | — | — |
| версия ДС | 0.2.0 | 0.3.0 | 0.1.0 | 0.1.0 |
| размер tokens.json | 324 796 Б | 366 677 Б | 13 160 Б | 12 177 Б |

**Поля биндинга iva-mobile** (34 записи): `name` 34/34, `match` 34/34, `figma_key` 34/34, `selector` 34/34, `kind` 34/34 (все `composable`), `source` 34/34, `notes` 16/34, `example` 2/34 (Bottom Sheet, List Item), `inputs` **1/34** (только Avatar). Алиасов в `match` суммарно **83**. Все 34 `source` лежат в одной папке-корне `element/`.

**34 компонента:** Avatar · Badge · System Badge · Room Status Badge · Banner · Bottom Sheet · Calendar Month · Line Chart · Pie Chart · Rounded Container · Full Screen Column · Alert Dialog · Confirmation Dialog · Divider · Dropdown Menu · Screen Header · Icon Button · Circle Button · Play Button · Info Panel · List Item · List Item Switch · List Item Input · Bottom Bar · Navigation Rail · Push Notification · Search Bar · Skeleton · Spacer · Text Field · Switch · Toast · Empty State · Highlighted Text.

**Чем web-словарь устроен иначе** (важно при унификации): у iva-web есть поле `storybook` (49/49) и семь разных `kind` — `component` 29, `attribute-component` 11, `directive` 4, `directive+component` 2, `directive-component` 1, `component+service` 1, `generated-components` 1. У mobile один `kind` и никакого Storybook: в словаре прямо сказано «no Storybook; components carry @Preview functions next to the source». Значит, посмотреть KMP-компонент, не имея клона репозитория, **невозможно**.

**История файлов ДС** (`git log -- design-systems/`): три последних коммита — **Dmitry Solonko**, 22–23.07: iva-web DS 0.2.0 (#122), затем iva-web DS 0.3.0 (stable figma_key), затем iva-mobile DS 0.2.0 (KMP code-bindings + kmp figma quickstart) = `bfbd33c` 23.07. Мы этот словарь не писали и не ведём.

---

## 2. Ивовский контур: что на диске, чего нет

### 2а. Каталоги `iva-*` под `~/tacticum` — не про ДС и не git-репозитории

`iva-rag`, `iva-rag2`, `iva-rag1-engine`, `iva-rag1-docs`, `iva-rag-repo-plan` — RAG-контур (bff / eval / index / ingest), у всех пяти `.git` отсутствует.

### 2б. Все упоминания `git.hi-tech.org` в `tacticum-dev` (51 вхождение)

| Путь | Раз | Где |
|---|---|---|
| `git.hi-tech.org/iva/one/ios/messenger` | 5 | iOS-профиль |
| `git.hi-tech.org/rn/rn` | 3 | `design-systems/iva-rn/design-system.yaml`, RN-профиль |
| `git.hi-tech.org/ivaqa/kit` | 3 | QA-профиль |
| `git.hi-tech.org/iva/one/web/iva-one` | 3 | `design-systems/iva-web/tokens.json`, web-профиль, `web-to-kmp-source-reference` |
| `git.hi-tech.org/web/iva-connect` | 1 | навык `iva-core-design-system` |
| `git.hi-tech.org/iva-m/android/su.ivcs.messenger` | 1 | `web-to-kmp-source-reference` |
| `git.hi-tech.org/iva-m/android/kmp/` | 1 | `design-systems/iva-mobile/tokens.json` |

**Это упоминания в тексте, а не доступ.** Ни клонов, ни remote-ов, ни конфигов доступа к GitLab ИВА на машине не найдено.

### 2в. Модель доступа (по карте доступов в vault; серверы сам не трогал)

- Своей веб-учётки в GitLab ИВА у нас **нет**. Доступ обеспечивает машинный SSH-deploy-ключ на `adp_emb` (194.36.208.242) — детали в заметке `20-Architecture/Доступы- серверы и репозитории ИВА ...`. Он даёт clone/fetch по зеркалу `/srv/iva/repos/` (~220 репо, юзер `tacticum`). Для записи и для GitLab API нужен отдельный доступ, которого нет.
- Правило Президента от 2026-07-24: **все серверы read-only**.
- Figma: доступ на чтение выпущен (d.solonko) — источник: план ТЗ-1, раздел «Подопытные / люди». 32 ключа iva-web извлечены через Figma REST под owner-issued доступом (`design-systems/iva-web/design-system.yaml`, 0.3.0). Figma MCP в окружении есть, **к профилям не подключён**.

---

## 3. Worktree по ДС и вебу

`git worktree list` из `tacticum-dev` даёт 20 worktree. К ДС и вебу относятся 5.

| Worktree | Ветка | Грязных файлов | Последний коммит | Своих коммитов вне main | PR | Вердикт |
|---|---|---|---|---|---|---|
| `/Users/bubblemac/tacticum/tacticum-dev-ds-web-to-kmp` | `feat/ds-web-to-kmp` | 0 | `6c0db8b` 2026-07-24 18:46 Александр Шульга | **0** (main +64) | **#144** смержен 24.07 (`034ef1b`) | **мёртвый**, влит |
| `/Users/bubblemac/tacticum/tacticum-dev-web-axis1` | `feat/ds-web-axis1` | 0 | `2d919de` 2026-07-25 01:18 Александр Шульга | **0** (main +12) | **#159** смержен 25.07 (`7ffe389`) | **мёртвый**, влит |
| `/Users/bubblemac/tacticum/tacticum-dev-web-mockup` | `feat/ds-web-mockup-figma` | 0 | `ff7589c` 2026-07-24 21:38 Александр Шульга | **0** (main +37) | **#154** смержен 24.07 (`b9f5610`) | **мёртвый**, влит |
| `/Users/bubblemac/tacticum/tacticum-dev-web-sc12` | `feat/ds-web-sc12` | 0 | `a222127` 2026-07-24 20:36 Александр Шульга | **0** (main +45) | **#149** смержен 24.07 (`5552118`) | **мёртвый**, влит |
| `/Users/bubblemac/tacticum-worktrees/kmp-axis2` | `feat/kmp-multirepo-axis2` | 0 | `b1a23e3` 2026-07-24 18:56 Александр Шульга | **0** (main +60) | **#145** смержен 24.07 (`928fe37`) | **мёртвый**, влит |

Метод проверки: `git rev-list --left-right --count main...HEAD` (правое число 0 означает, что ветка целиком в main) плюс `git branch --merged main`. **Ждущих мержа ДС-веток нет, незакоммиченного нет нигде.**

**Что в каждой ветке делалось** (`git diff --stat <merge>^1 <merge>`):

- **#144 `ds-web-to-kmp`** — 4 файла, +555/-1. Навыки `web-to-kmp-screen-port` и `web-to-kmp-source-reference` в `templates/iva-kmp-development-base/`. Это Сц.4 из ТЗ-1.
- **#145 `kmp-multirepo-axis2`** — 6 файлов, +131/-2. Мульти-репо ось-2 в `iva-kmp-brownfield`: аргумент source-репо в `start-task` плюс три CLI-тела `tacticum-workflow` (claude / codex / copilot).
- **#149 `ds-web-sc12`** — 4 файла, +404/-2. Сценарии 1 и 2 для веба: `angular-ds-component-authoring` и `angular-ds-component-usage`.
- **#154 `ds-web-mockup-figma`** — 3 файла, +197/-8. Режим Figma numeric-compare в `ui-mockup-match`.
- **#159 `ds-web-axis1`** — 12 файлов, +489/-43. Ось-1: ДС `iva-core` навыком, роутер поверхностей в `design-system-discovery`, зеркальная раскатка на `iva-web-development-base`, `docs/user_manuals/iva-web-figma-mapping-quickstart.md`.

Остальные 14 worktree (QA, FR-аналитик, workflow-modes, helm, rag) к ДС не относятся.

---

## 4. Четыре поверхности плюс KMP

Репозитории, стек и ответственные распарсены из `90-Materials/План индексаций.xlsx` (файл Глеба, лист 1). Локально нет ни одного из этих репозиториев.

| Поверхность | Репозиторий (git.hi-tech.org) | Стек | Ответственный | Приор. | Токены/ДС у нас |
|---|---|---|---|---|---|
| **Desktop Qt** | `imail/desktop/coredesktop` (Ядро) | C++/Qt | Лукьянчиков Олег | 1 | **НЕТ** |
| | `imail/desktop/ivaplugins` (Плагины) | C++/Qt | Фёдоров Сергей | 1 | **НЕТ** |
| | `imail/desktop/jump` (Почта+Календарь) | C++/Qt | Круглецов Егор | 1 | **НЕТ** |
| | `desktop/ivcs` (IVA Connect Desktop) | C++/Qt | Прохоров Иван | 1 | **НЕТ** |
| **Web Angular** | `iva/one/web/iva-one` (One) | TS/Angular | Савицкий Максим | 1 | **ДА** — `iva-web` 0.3.0, 49 биндингов, 32 ключа |
| | `web/iva-admin` | TS/Angular | Гришкин Николай | 1 | отдельной ДС нет |
| | `web/iva-connect` | TS/Angular | не указан | 1 | ДС `iva-core` описана навыком, **файла нет** |
| | `web/iva-outlook-plugin_web` | TS | Трегубов Антон (?) | 1 | нет |
| **iOS** | `iva/one/ios/messenger` (One iOS) | Swift/SwiftUI | Гузеев Максим | 3 | **НЕТ** |
| | `ivaone-sdk`, `calls`, `calendarsdk`, `imailframework` | Swift | Гузеев Максим | 3 / либы | нет |
| | `gitlab-mobile.msk/IvaMailMobile/apple-rapido` | Swift | Лихачёв Андрей | 3 | нет |
| | `mobile/apple/messenger` (Connect iOS) | Swift | Капелько Михаил | 2 | нет |
| **Android** | `iva-m/android/su.ivcs.messenger` (One Android) | Kotlin/Compose | Легин Денис | 2 | частично — через `iva-mobile` 0.2.0 |
| | `gitlab-mobile.msk/IvaMailMobile/android-orion` | Kotlin | Гусельников Дмитрий | 1 | нет |
| | `mobile/ucim-android` (Connect Android) | Kotlin | Ольховой Михаил | 2 | нет |
| **KMP (shared)** | `iva-m/android/kmp` | Kotlin/Compose MP | в Excel не указан | 1 | **ДА** — `iva-mobile` 0.2.0, 34 биндинга, **0 ключей** |
| **React Native** | `rn/rn` (rn-main) | TS/React Native | в Excel не указан | 1 | **ДА** — `iva-rn` 0.1.0, **без биндингов**, собрана из `rn-shared/src/theme`, не из Figma |

**Честные «не знаю»:**

- Про Desktop Qt в нашем репозитории и в vault **нет ничего, кроме строк Excel** — ни профиля, ни навыка, ни токенов, ни описания дизайна. Единственное упоминание Qt в vault — пересказ созвона 27.07.
- Про iOS есть профиль `iva-ios-brownfield`, но своей ДС для iOS нет: токены идут только если ДС приложена к Workspace.
- Ветка и владелец KMP-репозитория, кроме коммита `8f65eea2ff`, неизвестны.

---

## 5. Риски и точки, которые надо разруливать

1. **Мобильной Figma-библиотеки не существует** — установленный факт из плана ТЗ-1: «мобильной Figma-библиотеки компонентов НЕТ (файл IVA Mobile DS пустой), мобильные экраны рисуются на веб-компонентах». Там же названо 107 выгруженных ключей веб-мастер-компонентов — **в `tokens.json` этих 107 нет** (49 биндингов, 32 ключа), расхождение не объяснено. Отсюда и 0 из 34 ключей у мобильной ДС.
2. **17 незаполненных `figma_key`** в web-словаре — по board-заметке `map-existing-vs-gap-pr-c-axis1` это работа дизайнеров и вопрос доступа к Figma, вынесена наружу.
3. **Пятая ДС `iva-core`** (конференц/VCS: npm-пакет `iva-core`, Figma-файл **VCSWEB**, репо `web/iva-connect`, токены `get-color()` и `--primary-color`, RGB-триплеты руками) — **описана навыком, но каталога `design-systems/iva-core/` НЕТ**. Общий словарь с `@iva/design-system` невозможен по архитектуре токенов.
4. **Маршрут сборки захардкожен и без источника.** `apps/backend/scripts/merge_iva_tokens.py` поддерживает ровно два профиля (`iva-web`, `iva-mobile`); `source_ref: docs/concept/design/tokens/` в обоих yaml — **этой папки в репозитории нет** (`git log` по ней пуст, в `docs/concept/` два файла). Исходные экспорты Tokens Studio не хранятся вообще.
5. **Расхождение URL KMP-репозитория** внутри наших же файлов (см. 1б) — упрёмся при первом же обращении к коду.
6. **34 против 49 `Iva*`** — неполнота словаря KMP нигде не заявлена.
7. **Все ДС-ветки влиты** — текущий пласт работы закрыт, новый заход стартует с чистого main.

---

## Оговорка про запись этой заметки

Первые попытки записать файл целиком были отклонены системой разрешений; заметка собрана по частям. Причина отклонений — фрагменты с путём к SSH-ключу доступа: перефразировал без них, содержание не пострадало.

## Связано

`11-Directions/Направление- Единая дизайн-система Tacticum (Figma-токены → код).md` · `20-Architecture/Доступы- серверы и репозитории ИВА (adp - teststand - git.hi-tech.org).md` · `91-Archive/plans/План ТЗ-1 Дизайн-процесс Figma↔код — Сц.4 перенос форм one→kmp (lead-ds).md` · `01-Sessions/call-2026-07-27-1130 — приоритет дизайн, история фронтов ИВА, решение по iva-write.md`
