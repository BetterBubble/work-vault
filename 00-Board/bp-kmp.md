---
title: Best practices — Kotlin Multiplatform (источники)
type: note
status: current
created: 2026-08-03
updated: 2026-08-03
tags: [board, bestpractices, kmp]
permalink: tacticum/00-board/bp-kmp
---

# Kotlin Multiplatform — официальные источники best practices

> **Коротко:** в профиле — Kotlin Multiplatform 2.4 (Compose Multiplatform, Ktor и Room названы без версий); актуальная на 2026-08-03 — Kotlin 2.4.10 от 14.07.2026, линия 2.4 текущая; версия языка в профиле поддерживается, но остальной стек не версионирован, а внутри самой 2.4 уже сменились скелет KMP-проекта и мажор Room.

## Стек по профилю

Дословно из `TABLE_BLURBS` (`scripts/wiki_sync_manuals.py`, репозиторий TacticumApps/dev):

> Kotlin Multiplatform 2.4 + Compose Multiplatform; Gradle (Kotlin DSL); Decompose + MVIKotlin; Ktor / Room; kotlin.test + Compose UI test; архитектурные гарды `./gradlew check`

Приложение KmpMessenger: Android / iOS / Desktop / Web из общего `commonMain`. Профили агентов: `iva-role-kmp`, `iva-kmp-brownfield`.

Что из этого имеет живой официальный источник, а что нет — в таблице ниже. Отдельно: «архитектурные гарды `./gradlew check`» — это внутренняя конвенция проекта, официального источника у неё нет и быть не может; сверять её не с чем.

## Источники

Все ссылки открыты и проверены 2026-08-03. Колонка «Чей» важна: у KMP два независимых вендора норм — JetBrains (язык, KMP, Compose MP, Ktor) и Google (Android-стиль, Room, Lint), и они расходятся в деталях.

| Источник | Тип | Чей | Что именно берём | Как часто обновляется | Дата сверки |
|---|---|---|---|---|---|
| [Kotlin coding conventions](https://kotlinlang.org/docs/coding-conventions.html) | Конвенции | JetBrains | Базовый стиль: организация исходников, layout класса, нейминг, форматирование, KDoc, идиоматичное использование фич, отдельный раздел для библиотек | Редко, точечно (страница обновлена 16.07.2026) | 2026-08-03 |
| [Android Kotlin style guide](https://developer.android.com/kotlin/style-guide) | Конвенции | Google | Стиль для Android-части: файлы, импорты, переносы, нейминг, backing properties. Отличается от JetBrains-конвенций в мелочах | Тело документа датировано 2023-09-06, страница живая | 2026-08-03 |
| [Basics of KMP project structure](https://kotlinlang.org/docs/multiplatform/multiplatform-discover-project.html) | Доки | JetBrains | Канон по `commonMain`/targets/source sets, иерархия и default hierarchy template (`appleMain`, `iosMain` и пр.), где живут тесты | Активно (обновлена 27.07.2026) | 2026-08-03 |
| [Compose Multiplatform и Jetpack Compose](https://kotlinlang.org/docs/multiplatform/compose-multiplatform-and-jetpack-compose.html) | Доки | JetBrains | Прямое указание: API общий, гайды Google по Jetpack Compose применимы напрямую; список Android-only компонентов | Редко (обновлена 27.11.2025) | 2026-08-03 |
| [Stability of supported platforms](https://kotlinlang.org/docs/multiplatform/supported-platforms.html) | Доки | JetBrains | Что можно тащить в прод: уровни стабильности по платформам для ядра KMP и отдельно для Compose MP | Редко (обновлена 10.09.2025) | 2026-08-03 |
| [Compose MP: совместимость и версии](https://kotlinlang.org/docs/multiplatform/compose-compatibility-and-versioning.html) | Доки | JetBrains | Таблица соответствия Compose MP ↔ Jetpack Compose ↔ минимальный Kotlin; требования к JDK и минимальные версии ОС | Каждый релиз Compose MP | 2026-08-03 |
| [Test your multiplatform app](https://kotlinlang.org/docs/multiplatform/multiplatform-run-tests.html) | Доки | JetBrains | `kotlin.test` в общем коде, платформенные тесты (JUnit на Android, Kotlin/Native test runner на iOS), задача `allTests`, отчёты | Активно (обновлена 15.05.2026) | 2026-08-03 |
| [Interoperability with Swift using Swift export](https://kotlinlang.org/docs/native-swift-export.html) | Доки | JetBrains | Замена Objective-C-интеропа: `suspend` → `async/await`, `Flow` → `AsyncSequence`; список ограничений Alpha | Активно (обновлена 29.05.2026) | 2026-08-03 |
| [What's new in Kotlin 2.4.0](https://kotlinlang.org/docs/whatsnew24.html) | Release notes | JetBrains | Полный перечень новых Stable/Experimental фич текущей версии языка | Раз в feature-релиз (~6 мес) | 2026-08-03 |
| [Compatibility guide for Kotlin 2.4](https://kotlinlang.org/docs/compatibility-guide-24.html) | Migration guide | JetBrains | **Главный источник «что устарело»**: что подняли до ERROR, что до WARNING, что удалили. Именно эту страницу надо диффать при обновлении | Раз в feature-релиз | 2026-08-03 |
| [Kotlin release process](https://kotlinlang.org/docs/releases.html) | Политика | JetBrains | Календарь: language 2.x.0 раз в 6 мес, tooling 2.x.20 через 3 мес, bugfix без графика. Окно поддержки stdlib | Каждый релиз | 2026-08-03 |
| [Kotlin roadmap](https://kotlinlang.org/docs/roadmap.html) | Роадмап | JetBrains | Куда едет язык и KMP в ближайшие полгода — чтобы не писать код под то, что вот-вот сменится | Раз в 6 мес (обновлён 26.02.2026, следующий — август 2026) | 2026-08-03 |
| [Kotlin/KEEP](https://github.com/Kotlin/KEEP) | Процесс эволюции | JetBrains | Предложения по языку и stdlib с номерами и статусами (public discussion → in progress → experimental → stable/declined). Сигнал за 1–2 релиза до появления фичи | По мере подачи предложений | 2026-08-03 |
| [Kotlin blog — категория Releases](https://blog.jetbrains.com/kotlin/category/releases/) | Блог | JetBrains | Точка входа: анонсы Kotlin, Compose MP, Ktor. С неё дешевле всего ловить «что вышло с прошлой сверки» | Несколько раз в месяц | 2026-08-03 |
| [Kotlin 2.4.0 Released](https://blog.jetbrains.com/kotlin/2026/06/kotlin-2-4-0-released/) | Блог-релиз | JetBrains | Разбор релиза 2.4.0 человеческим языком | Разовый пост (июнь 2026) | 2026-08-03 |
| [A New Default Project Structure for KMP](https://blog.jetbrains.com/kotlin/2026/05/new-kmp-default-structure/) | Блог-решение | JetBrains | Смена канона структуры проекта + причина (требование AGP 9.0). **Прямо задевает скелеты, которые генерят агентные профили** | Разовый пост (май 2026) | 2026-08-03 |
| [KotlinConf'26 Keynote Highlights](https://blog.jetbrains.com/kotlin/2026/05/kotlinconf26-keynote-highlights/) | Блог-анонс | JetBrains | Годовой срез направления: Kotlin Toolchain/Amper, Kotlin Language Server, Swift export, статус web | Раз в год (май 2026) | 2026-08-03 |
| [Compose MP 1.11.0](https://blog.jetbrains.com/kotlin/2026/05/compose-multiplatform-1-11-0/) | Блог-релиз | JetBrains | Что включили по умолчанию (concurrent rendering на iOS) и что задепрекейтили (старые UI-test API) | Разовый пост (май 2026) | 2026-08-03 |
| [Compose MP 1.10.0](https://blog.jetbrains.com/kotlin/2026/01/compose-multiplatform-1-10-0/) | Блог-релиз | JetBrains | Единая `@Preview` в `commonMain`, Navigation 3, стабильный Compose Hot Reload | Разовый пост (январь 2026) | 2026-08-03 |
| [compose-multiplatform / releases](https://github.com/JetBrains/compose-multiplatform/releases) | Release notes | JetBrains | Построчный changelog, включая beta-ветку — видно, что готовится к следующей мажорной | Каждый релиз и пре-релиз | 2026-08-03 |
| [Ktor releases](https://ktor.io/docs/releases.html) | Release notes | JetBrains | Индекс всех версий Ktor с датами (последняя — 3.5.1 от 26.06.2026) и ссылками на what's-new. Выделенные what's-new есть не у всех версий, у остальных — changelog на GitHub | Каждый релиз | 2026-08-03 |
| [What's new in Ktor 3.5.0](https://ktor.io/docs/whats-new-350.html) | Release notes | JetBrains | Текущий срез API сервера и клиента, список депрекейтов | Раз в feature-релиз | 2026-08-03 |
| [Migrating from 2.2.x to 3.0.x (Ktor)](https://ktor.io/docs/migrating-3.html) | Migration guide | JetBrains | Переход на `kotlinx-io` (`ByteReadChannel`/`ByteWriteChannel`), удалённые артефакты. Нужен, только если в коде остались хвосты Ktor 2 | Стабилен | 2026-08-03 |
| [Set up Room Database for KMP](https://developer.android.com/kotlin/multiplatform/room) | Доки | Google | Каноничная настройка Room в KMP: артефакты `androidx.room3`, `BundledSQLiteDriver`, требование `suspend` в DAO вне `androidMain` | Активно, следует за релизами Room | 2026-08-03 |
| [Room 3.0 — Modernizing Room](https://android-developers.googleblog.com/2026/03/room-30-modernizing-room.html) | Блог-решение | Google | Обоснование мажора и полный список ломающих изменений 2.x → 3.0 | Разовый пост (март 2026) | 2026-08-03 |
| [ktlint — code styles](https://ktlint.github.io/ktlint/latest/rules/code-styles/) | Линтер | ktlint (не вендор) | Три стиля: `ktlint_official` (дефолт с 1.0), `intellij_idea`, `android_studio`; задаются через `.editorconfig` | Каждый релиз ktlint | 2026-08-03 |
| [ktlint / releases](https://github.com/ktlint/ktlint/releases) | Линтер | ktlint | Новые и повышенные до стандартных правила, поддерживаемые версии Kotlin | Каждый релиз | 2026-08-03 |
| [detekt — таблица совместимости](https://detekt.dev/docs/introduction/compatibility/) | Линтер | detekt (не вендор) | **Проверять первым делом при апгрейде Kotlin**: какая версия detekt под какой Kotlin | Каждый релиз detekt | 2026-08-03 |
| [detekt — changelog и migration guide](https://detekt.dev/changelog/) | Линтер | detekt | Изменения правил и дефолтов между версиями | Каждый релиз | 2026-08-03 |
| [Android Lint](https://developer.android.com/studio/write/lint) | Линтер | Google | Правила корректности/производительности/доступности для Android-части, включая Compose-специфичные (`ComposableNaming`, `ComposeUnrememberedState`); настройка через `lint.xml` и блок `lint {}` | Активно (обновлена 14.07.2026) | 2026-08-03 |
| [Gradle Kotlin DSL Primer](https://docs.gradle.org/current/userguide/kotlin_dsl.html) | Доки | Gradle | Нормы `build.gradle.kts`: блок `plugins {}`, type-safe accessors, `named()`/`register()` вместо `getByName()`/`create()`, `=` вместо `.set()` | Каждый релиз Gradle (сейчас 9.6.1) | 2026-08-03 |
| [Decompose](https://arkivanov.github.io/Decompose/) | Доки библиотеки | arkivanov (OSS, один мейнтейнер) | Компоненты, жизненный цикл, навигация (stack/slot/pages/panels), расширения для Compose, FAQ и tips | Нерегулярно | 2026-08-03 |
| [MVIKotlin](https://arkivanov.github.io/MVIKotlin/) | Доки библиотеки | arkivanov (OSS, один мейнтейнер) | Store, View, binding и lifecycle, сохранение состояния, логирование, time travel | Нерегулярно | 2026-08-03 |

Итого: **33 проверенных источника**.

Минимальный набор для регулярной сверки (если гонять не всё): Compatibility guide текущей версии Kotlin → Kotlin roadmap → блог JetBrains (Releases) → таблица совместимости detekt → Compose MP versioning.

## Что изменилось за последний год

Отсчёт от лета 2025. Отобрано то, что реально меняет генерируемый агентом код, а не просто попало в release notes.

**1. K1 окончательно мёртв, `-language-version=1.9` больше не компилируется.** По [compatibility guide 2.4](https://kotlinlang.org/docs/compatibility-guide-24.html) K1-компилятор удалён, а вместе с ним до уровня ERROR подняты цепочки депрекейтов, тянувшиеся с 2.0–2.3: nullable type arguments для Java-типов, бессмысленные `is`-проверки, понижение видимости в inline-функциях, array literals вне аннотаций (нужен `arrayOf()`), exhaustiveness для Java sealed-классов. Код «в стиле 2025» здесь ломается не предупреждением, а ошибкой сборки.

**2. Context parameters и explicit backing fields стали Stable.** [Kotlin 2.4.0](https://kotlinlang.org/docs/whatsnew24.html) (релиз 3 июня 2026, текущая линия — 2.4.10 от 14 июля 2026) вывел из эксперимента context parameters, explicit backing fields, `@all` meta-target для аннотаций и UUID API в общем stdlib. Это значит, что агент больше не обязан обкладывать их `@OptIn` и городить обходные пути — но и не должен продолжать писать по-старому там, где новая конструкция уместна. Заодно с 2.4.0 введено окно поддержки stdlib 18 месяцев на линию релиза.

**3. Сменился дефолтный скелет KMP-проекта.** [Решение мая 2026](https://blog.jetbrains.com/kotlin/2026/05/new-kmp-default-structure/): вместо одного модуля `composeApp`, который смешивал библиотеку и приложения, теперь `shared` (чистая KMP-библиотека) плюс отдельные модули приложений. Причина не косметическая — AGP 9.0 требует, чтобы точка входа Android-приложения лежала отдельно от общего кода. Новый визард уже отдаёт эту структуру. **Это самый прямой удар по агентным профилям**: скелеты и инструкции, написанные под старую раскладку, теперь генерируют устаревший проект.

**4. Room переехал в `androidx.room3` — и это ломающий мажор.** [Room 3.0](https://android-developers.googleblog.com/2026/03/room-30-modernizing-room.html) (март 2026): убраны SupportSQLite API в пользу `androidx.sqlite`-драйверов, только Kotlin-кодогенерация, только KSP (KAPT и Java-обработка выброшены), «coroutines first» — DAO вне `androidMain` обязаны быть `suspend`. Добавлены JS и WasmJS. [Официальная страница настройки Room под KMP](https://developer.android.com/kotlin/multiplatform/room) уже показывает `androidx.room3` (3.0.1) и `BundledSQLiteDriver` как рекомендуемый путь, а Room 2.8 переведён в maintenance. Профиль, называющий «Room», должен теперь уточнять, какой именно.

**5. Compose Multiplatform: единая `@Preview` и новые API тестов, старые задепрекейчены.** В [1.10.0](https://blog.jetbrains.com/kotlin/2026/01/compose-multiplatform-1-10-0/) (январь 2026) появилась одна общая аннотация `@Preview`, работающая в `commonMain`, — все прежние варианты объявлены устаревшими; там же Navigation 3 на не-Android платформах и стабильный Compose Hot Reload прямо в Gradle-плагине. В [1.11.0](https://blog.jetbrains.com/kotlin/2026/05/compose-multiplatform-1-11-0/) (май 2026) задепрекейчены `runComposeUiTest`, `runSkikoComposeUiTest` и `runDesktopComposeUiTest` в пользу v2-API `ComposeUiTest` со `StandardTestDispatcher` по умолчанию, а concurrent rendering на iOS включён по умолчанию. Пункт профиля «Compose UI test» указывает ровно на тот API, который только что устарел.

**6. Swift export дорос до Alpha и заявлен как замена Objective-C-интеропу.** С Kotlin 2.4.0 ([доки](https://kotlinlang.org/docs/native-swift-export.html), обновлены 29.05.2026) `suspend`-функции экспортируются как Swift `async`, а `Flow` — как `AsyncSequence`, без Objective-C-заголовков и манглинга имён. Пока Alpha с существенными ограничениями (только final-классы, generic-типы стираются до верхней границы, нет кросс-языкового наследования) — но направление зафиксировано, и писать iOS-интероп «навсегда через Objective-C» уже неверно.

Дополнительно, менее срочное: доки KMP переехали с `jetbrains.com/help/kotlin-multiplatform-dev/` на `kotlinlang.org/docs/multiplatform/` (старые адреса отвечают 301, но закладки в профилях лучше переписать); Kotlin/Wasm получил инкрементальную компиляцию по умолчанию; CMS GC стал дефолтным сборщиком в Kotlin/Native, а минимальные версии Apple-платформ подняты (iOS 14 → 15); на KotlinConf'26 объявлены Kotlin Toolchain на базе Amper и Kotlin Language Server в статусе Alpha — это про инструментарий, не про код, но в горизонте года повлияет.

## Что переписать в профиле прямо сейчас

Действия над текстом скиллов профилей `iva-role-kmp` и `iva-kmp-brownfield`. Каждый пункт — замена конкретной формулировки, а не тема для изучения.

1. **Скелет проекта: `composeApp` → `shared` + отдельные модули приложений.** Инструкции и генерируемая раскладка, где общий код и Android-приложение живут в одном модуле, дают устаревший проект: AGP 9.0 требует точку входа Android отдельно от общего кода, и новый визард JetBrains отдаёт уже другую структуру.
2. **`Room` в инструкциях уточнить до `androidx.room3`.** Вместе с этим прописать: только KSP (KAPT и Java-обработка выброшены), `BundledSQLiteDriver` вместо SupportSQLite API, **DAO вне `androidMain` обязаны быть `suspend`**. Слово «Room» без версии сейчас указывает на 2.8, переведённый в maintenance.
3. **`Compose UI test` заменить на v2-API `ComposeUiTest`.** `runComposeUiTest`, `runSkikoComposeUiTest` и `runDesktopComposeUiTest` задепрекейчены в Compose MP 1.11.0; строка профиля «kotlin.test + Compose UI test» указывает ровно на устаревший API.
4. **`@Preview`: убрать платформенные варианты, оставить одну общую аннотацию в `commonMain`.** С Compose MP 1.10.0 прежние варианты объявлены устаревшими — если в скиллах перечислены платформенные `@Preview`, их надо свести к одной.
5. **iOS-интероп: «через Objective-C-заголовки» → Swift export.** В правилах интеропа зафиксировать `suspend` → `async`, `Flow` → `AsyncSequence` и рядом — ограничения Alpha (только final-классы, generic стираются, нет кросс-языкового наследования), чтобы агент не писал Objective-C-интероп как безальтернативный.
6. **Ссылки на доки KMP переписать с `jetbrains.com/help/kotlin-multiplatform-dev/` на `kotlinlang.org/docs/multiplatform/`**, а в требования к линтерам добавить явный выбор версии detekt под Kotlin 2.4 (стабильная 1.23.8 держит только 2.0.21) — иначе «прогнать detekt» в инструкции неисполнимо.

## Чего не нашёл / где источник слабый

**Нет официального документа «KMP best practices».** Это главная дыра. У JetBrains есть доки, туториалы и release notes, но единой страницы «как правильно писать KMP-код» не существует — набор практик приходится собирать из structure/testing/platform-страниц и блога. Любой документ, который команда назовёт «best practices KMP», будет нашей компиляцией, а не цитатой вендора; это стоит проговорить на ревью, чтобы его не воспринимали как выдержку из официального источника.

**Страница стабильности платформ подозрительно старая.** [supported-platforms](https://kotlinlang.org/docs/multiplatform/supported-platforms.html) обновлялась 10.09.2025 и показывает Web (Kotlin/Wasm) как Beta для ядра и для Compose MP. Роадмап и keynote за 2026 год говорят о продолжающейся работе над web, но явного перевода в Stable я не нашёл. Для KmpMessenger, у которого Web в списке таргетов, это существенно: **на 2026-08-03 официального подтверждения стабильности web-таргета нет**. Если для проекта это критично — вопрос стоит задать в трекере JetBrains, а не выводить из косвенных признаков.

**Линтеры — не вендорские, и это структурный риск.** Ни ktlint, ни detekt не принадлежат JetBrains: ktlint лишь *старается* отражать официальные конвенции, detekt поддерживает актуальный Kotlin с задержкой. Прямо сейчас [таблица совместимости detekt](https://detekt.dev/docs/introduction/compatibility/) даёт последнюю стабильную версию 1.23.8 под Kotlin **2.0.21**, а Kotlin 2.4.0 поддержан только в 2.0.0-alpha (там же — переход с K1 на официальный Kotlin Analysis API). То есть проект на Kotlin 2.4 сегодня выбирает между альфой линтера и линтером на три минорные версии позади. Это надо решать явно, а не молча. Отдельно: на KotlinConf'26 упомянута работа Kotlin Foundation с Meta над стандартизацией инструментов форматирования — возможно, через год расклад изменится.

**Decompose и MVIKotlin — самое слабое место каталога.** Обе библиотеки держит фактически один мейнтейнер (arkivanov). Документация живая и содержательная, но на сайтах **не указана текущая версия и нет раздела миграций между версиями** — сверяться приходится по GitHub Releases, а не по докам. Гарантий совместимости с новыми версиями Kotlin и Compose MP никто не даёт. Для двух библиотек, на которых держится вся архитектура KmpMessenger, это стоит зафиксировать как отдельный риск.

**Даты релизов с GitHub Releases не проверены.** Страницы `github.com/ktlint/ktlint/releases` и `github.com/JetBrains/compose-multiplatform/releases` живые, но при машинном чтении отдают относительные метки времени, которые разбираются ненадёжно — в моей выборке даты вышли явно противоречивыми. Номера версий брать оттуда можно, даты — нет; для дат нужен блог или страница релизов на сайте продукта.

**Ktor и «архитектурные гарды».** По Ktor есть индекс релизов и what's-new, но отдельного документа про рекомендованную структуру клиента в KMP-проекте я не нашёл — в what's-new 3.5.0 KMP-специфика клиента почти не раскрыта. А «архитектурные гарды `./gradlew check`» из профиля — внутренняя конвенция проекта; официального источника у неё нет, и в этом каталоге ей соответствовать нечему.

## Владелец ревью

| Роль | Кто | Дата сверки |
|---|---|---|
| Владелец стека | _не назначен — впишите себя_ | 2026-08-03 |

Правьте эту страницу под свой стек: вычёркивайте источники, которыми не пользуетесь, дописывайте свои, поправьте описание стека, если оно разошлось с реальностью.
