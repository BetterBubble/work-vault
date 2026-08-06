---
permalink: tacticum/00-board/bp-swift-build
---

---
title: Best practices — Swift: сборка, зависимости, тесты (источники)
type: note
status: current
created: 2026-08-03
updated: 2026-08-03
tags: [board, bestpractices, swift, build]
permalink: tacticum/00-board/bp-swift-build
---

# Swift: сборка, зависимости, тесты — официальные источники best practices

> **Коротко:** в профиле версия Xcode **не зафиксирована нигде** — ни в манифестах (`stack: required: [swift]` без версии), ни в скиллах сборки и запуска; актуальная на 2026-08-03 — Xcode 26.6 стейбл и Xcode 27 beta 4; сверять не с чем. Из инструментов профиля два зафиксированы и оба проблемные: CocoaPods-путь UCIM упирается в read-only trunk 02.12.2026, а Ruby 2.6 ниже требуемого fastlane минимума 3.0+.

## Стек по профилю

Дословно из quickstart-таблиц (репозиторий `tacticum-dev`, `origin/main`).

**IVA One iOS** — строка про лейн `development-ios` (`docs/user_manuals/iva-role-ios-profile-quickstart.md`):

> стек iOS: coder/tester/test-runner (iOS-тексты), fastlane-сборка, Xcode-запуск, Observation-state, fragile-зоны calls/AV и SDK-бриджей, IVALink

**IVA Connect iOS / UCIM** — строка про лейн `iva-connect-ios-development-base` (`docs/user_manuals/iva-role-connect-ios-profile-quickstart.md`):

> стек iOS/UCIM: coder/tester/test-runner (iOS-тексты), сборка SPM+XcodeGen (`ucim-spm/setup`) и CocoaPods-путь, Xcode-запуск на симуляторе `IVA`, MPAK-состояние (Module+Slot), fragile-зоны WebRTC/медиа и генерируемых DTO, реалтайм-транспорт

Профили: роли `iva-role-ios` (0.3.2) и `iva-role-connect-ios` (0.1.0), стек-лейны `iva-ios-development-base` (0.3.0) и `iva-connect-ios-development-base` (0.1.0). Числа в скобках — версии самих профилей, не версии тулчейна.

### Зафиксирована ли в профиле версия Swift / Xcode

**Нет.** Поле `stack` во всех четырёх манифестах одинаковое и безверсионное:

```yaml
stack:
  required:
  - swift
  optional: []
```

Поле `version` — это версия профиля, не тулчейна; `persona.scope` называет «Swift/Xcode» и «Swift/MPAK/XcodeGen/CocoaPods» без чисел. **Версия языка в профиле не зафиксирована — сверять не с чем.** Версия Xcode не названа тем более: ни в манифестах, ни в `fastlane-build-verification`, ни в `xcode-run-launch`, ни в `ucim-build-verification` минимальной версии Xcode или macOS нет — скиллы говорят только «требуется macOS».

Что в скиллах всё же версионировано и относится к сборке:

| Где | Что зафиксировано | Продукт |
|---|---|---|
| `ios-module-integration`, `swift-observation-state` | swift-tools 6.1, многие таргеты на `swiftLanguageMode(.v5)` | One |
| `xcode-run-launch` | деплой-таргет **iOS 17.0**, схема `Messenger`, сборка через `xcodebuild` / fastlane (`Gemfile`-pinned, `bundle exec`) | One |
| `ucim-build-verification`, `ucim-run-launch`, агенты `test-runner` | XcodeGen-проект `Ucim`: **деплой-таргет iOS 15.8, `SWIFT_VERSION 5.2`, `CODE_SIGN_STYLE Manual`**; легаси-путь CocoaPods на **Ruby 2.6**; симулятор `IVA`; WebRTC через git-lfs | UCIM |

## Что покрывает эта часть

Инструментальный слой iOS-профилей IVA (`iva-role-ios`, `iva-ios-brownfield`, `iva-role-connect-ios`/UCIM), без самого языка — Swift/Concurrency/Observation/SwiftUI лежат в соседней части:

- **SPM** — `Package.swift`, разрешение зависимостей, `swift build`/`swift test`;
- **Xcode** — версии, release notes, требования к машине, схемы и сборка;
- **CocoaPods** — статус проекта, путь `pod install` в UCIM;
- **XcodeGen** — генерация `.xcodeproj` из `project.yml` (`ucim-spm/setup`);
- **fastlane** — сборка, тесты и подпись из CI;
- **Тесты** — Swift Testing против XCTest, запуск на симуляторе.

Все ссылки ниже открыты вручную 2026-08-03; страниц, отдавших 404 или пустоту, в таблице нет.

## Источники

| Источник (ссылка) | Тип | Чей | Что именно берём | Как часто обновляется | Сверено |
|---|---|---|---|---|---|
| [SwiftPM Documentation](https://docs.swift.org/swiftpm/documentation/packagemanagerdocs/) | docs | Swift project (Apple) | Канон по SPM: структура пакета, зависимости, команды | С релизами Swift | 2026-08-03 |
| [PackageDescription API](https://docs.swift.org/swiftpm/documentation/packagedescription/) | API reference | Swift project | Точный справочник по `Package.swift`: `Package`, `Context`, `Version`, `WarningLevel`, `GitInformation` | С релизами Swift | 2026-08-03 |
| [Swift 6.3 Released](https://www.swift.org/blog/swift-6.3-released/) | блог релиза | swift.org | Что приехало в SPM и Swift Testing в 6.3 (24.03.2026) | Раз в мажор/минор | 2026-08-03 |
| [What's new in Swift: March 2026](https://www.swift.org/blog/whats-new-in-swift-march-2026/) | дайджест | swift.org | Направление тулинга: Swift Build, принятые предложения (31.03.2026) | Периодически | 2026-08-03 |
| [Xcode Release Notes (индекс)](https://developer.apple.com/documentation/xcode-release-notes) | release notes | Apple | Список версий; на 08.2026 — **Xcode 26.6 stable, Xcode 27 beta 4** | Каждый релиз и бета | 2026-08-03 |
| [Xcode 26 Release Notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-26-release-notes) | release notes | Apple | Compilation Caching, explicit modules по умолчанию, требования (macOS 15.6+, Swift 6.2, SDK 26) | Разово по релизу | 2026-08-03 |
| [Xcode 26.6 Release Notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-26_6-release-notes) | release notes | Apple | Текущий стейбл: Swift 6.3, SDK 26.5, **минимум macOS Tahoe 26.2** | Разово по релизу | 2026-08-03 |
| [XCTest](https://developer.apple.com/documentation/xctest) | docs | Apple | **Прямая позиция Apple** по выбору фреймворка тестов (цитата ниже) | С релизами Xcode | 2026-08-03 |
| [Swift Testing](https://developer.apple.com/documentation/testing) | docs | Apple | Справочник: `@Test`, `#expect`/`#require`, traits, параметризация, attachments | С релизами Xcode/Swift | 2026-08-03 |
| [Migrating a test from XCTest](https://developer.apple.com/documentation/testing/migratingfromxctest) | гайд | Apple | Механика переноса: `import Testing`, структуры вместо `XCTestCase`, `init`/`deinit` вместо `setUp`/`tearDown`, сосуществование в одном таргете | С релизами Xcode/Swift | 2026-08-03 |
| [swiftlang/swift-testing](https://github.com/swiftlang/swift-testing) | репозиторий | Swift project | Живость и платформы; релизы тегируются вместе с тулчейном (`swift-6.3.2-RELEASE`, 13.05.2026) | С релизами Swift | 2026-08-03 |
| [CocoaPods: Support & Maintenance Plans](https://blog.cocoapods.org/CocoaPods-Support-Plans/) | анонс мейнтейнеров | CocoaPods core team | **Первоисточник по режиму поддержки**, 13.08.2024 | Разово | 2026-08-03 |
| [CocoaPods Trunk Read-only Plan](https://blog.cocoapods.org/CocoaPods-Specs-Repo/) | анонс мейнтейнеров | CocoaPods core team | **Дата остановки trunk — 02.12.2026** и таймлайн; 30.11.2024, обновлён 05.2025 | По мере приближения даты | 2026-08-03 |
| [CocoaPods Blog](https://blog.cocoapods.org/) | блог | CocoaPods core team | Единственный канал официальных объявлений, включая security-инциденты trunk | Нерегулярно | 2026-08-03 |
| [CocoaPods releases](https://github.com/CocoaPods/CocoaPods/releases) | releases | CocoaPods | Проверка, выполняется ли обещание «2 релиза в год»: **1.16.2 (31.10.2024) → 1.17.0 (06.07.2026)** | Фактически ~раз в 20 месяцев | 2026-08-03 |
| [fastlane releases](https://github.com/fastlane/fastlane/releases) | releases | fastlane (Google) | Живость и совместимость с Xcode; последний — **2.237.0 от 05.07.2026** | ~раз в 1–2 месяца | 2026-08-03 |
| [fastlane docs](https://docs.fastlane.tools/) | docs | fastlane | Getting Started iOS, Running Tests, Codesigning, справочник действий; **требования к Ruby** | Вместе с релизами | 2026-08-03 |
| [XcodeGen releases](https://github.com/yonaskolb/XcodeGen/releases) | releases | Yonas Kolb (сообщество) | Живость; последний — **2.46.0 от 16.07.2026** | Неравномерно | 2026-08-03 |
| [XcodeGen ProjectSpec](https://github.com/yonaskolb/XcodeGen/blob/master/Docs/ProjectSpec.md) | reference | XcodeGen | Справочник `project.yml`: `targets`, `schemes`, `settings`, `configs`, `packages`, `*Templates`, порядок слияния настроек `groups → base → configs` | С релизами | 2026-08-03 |

Итого 19 проверенных страниц.

## Статус инструментов (жив / поддержка / планировать замену)

| Инструмент | Статус | Основание |
|---|---|---|
| **SPM** | **Жив, активно развивается** | Swift 6.3 (24.03.2026): превью Swift Build внутри SPM, prebuilt Swift Syntax для макросов, `swift package show-traits`; принято SE-0509 (генерация SBOM) |
| **Xcode** | **Жив** | 26.6 стейбл + 27 beta 4 на 08.2026 |
| **Swift Testing** | **Жив, умолчание для нового кода** | Прямая рекомендация Apple в доке XCTest; едет в составе тулчейна |
| **XCTest** | **Жив, но с суженной ролью** | Apple: новый код — Swift Testing, XCTest остаётся для UI- и перформанс-тестов |
| **fastlane** | **Жив, стабильный ритм** | 2.237.0 (05.07.2026), релизы каждые 1–2 месяца |
| **XcodeGen** | **Жив, но одиночный мейнтейнер и рваный ритм** | 2.44.1 (07.2025) → провал → 2.45.0 (03.2026) → 2.46.0 (07.2026) |
| **CocoaPods** | **Режим поддержки, trunk замирает 02.12.2026 — планировать уход** | Два поста мейнтейнеров + фактический разрыв в релизах на 20 месяцев |

### CocoaPods — точные формулировки первоисточников

Пост **«CocoaPods Support & Maintenance Plans»**, 13.08.2024:

> «We're still keeping it ticking, but we're being more up-front that CocoaPods is in maintenance mode.»

Что команда **обязуется** делать: закрывать системные проблемы безопасности trunk, выпускать **минимум два релиза в год** под обновления Xcode, разбирать обращения примерно **раз в полгода**, держать инфраструктуру сайта, принимать PR на совместимость с будущими версиями.
Что **не** обязуется: следить за GitHub issues, развивать функциональность, гарантировать ревью PR.

Пост **«CocoaPods Trunk Read-only Plan»**, 30.11.2024 (обновлён 05.2025):

> «In two years we plan to turn CocoaPods trunk to be read-only. At that point, no new versions or pods will be added to trunk.»

Таймлайн из поста: 05.2025 — блокировка новых подов с полем `prepare_command`; вторая половина 2025 — рассылка всем контрибьюторам podspec; 09–10.2026 — финальное предупреждение за месяц; 01–07.11.2026 — тестовый прогон read-only; **02.12.2026 — trunk окончательно перестаёт принимать новые podspec**.

Что это значит на практике:

- **Сборки не ломаются.** Specs Repo и CDN продолжают отдавать уже опубликованное, пока живы GitHub и jsDelivr; `pod install` со старым `Podfile.lock` работает.
- **Ломается развитие.** После 02.12.2026 через trunk **не приедут обновления зависимостей** — в том числе security-фиксы вендоров и совместимость с новыми Xcode/SDK.
- **Не затрагивает** тех, кто держит собственный specs repo или вендорит зависимости целиком.

**Отдельно — расхождение обещания с фактом.** Мейнтейнеры обещали два релиза в год под Xcode. Между 1.16.2 (31.10.2024) и 1.17.0 (06.07.2026) прошло ~20 месяцев без единого релиза. То есть заявленный уровень поддержки уже не выдерживается, и на «под новый Xcode нам починят» закладываться нельзя.

### Позиция Apple по тестам — точная цитата

Из документации [XCTest](https://developer.apple.com/documentation/xctest):

> «Xcode 16 and later includes Swift Testing, a framework for writing unit tests that takes advantage of the powerful capabilities of the Swift programming language. Consider using Swift Testing for new unit test development and migrating existing tests as described in Migrating a test from XCTest. A test target can contain tests using both Swift Testing and XCTest, however don't mix API from the two frameworks in the same test. Continue to use XCTest for user interface tests and performance tests.»

Три следствия, которые надо держать в правилах профиля:

1. новый unit-код — Swift Testing;
2. **UI-тесты (XCUITest) и перформанс-тесты остаются на XCTest** — это не временное состояние, а явное указание Apple;
3. смешивать API двух фреймворков **внутри одного теста** нельзя, при этом в одном таргете они уживаются — миграция инкрементальная.

## Что изменилось за последний год

- **CocoaPods вошёл в финальный отрезок.** 02.12.2026 — уже внутри горизонта планирования: осталось около четырёх месяцев. В мае 2025 отвалились поды с `prepare_command`, осенью 2026 придёт финальное предупреждение.
- **CocoaPods выпустил 1.17.0 (06.07.2026)** после 20-месячной паузы — то есть проект не мёртв, но и обещанного ритма не держит.
- **SPM получил Swift Build.** В Swift 6.3 (24.03.2026) новый движок сборки доступен как опция, а на `main` Swift он уже стал системой сборки по умолчанию — единый движок под все платформы. Это ближайшее место, где может поехать поведение сборки.
- **Xcode прошёл цикл 26 → 26.6, вышла бета 27.** Существенно для машин сборки: Xcode 26 требовал macOS Sequoia 15.6, а **Xcode 26.6 — уже macOS Tahoe 26.2**. Планка ОС на CI и на машинах разработчиков поднялась в течение одного мажора.
- **Compilation Caching в Xcode 26** — опциональное кеширование результатов компиляции Swift и C-семейства, заметно на переключении веток и чистых сборках. Включается настройкой сборки `Enable Compilation Caching`.
- **Explicit modules стали умолчанием** для всех Swift-таргетов в Xcode 26 — типовая причина новых ошибок сборки на brownfield-проектах.
- **Swift Testing дорос**: в 6.3 добавились issues уровня warning (не роняют тест), `try Test.cancel()`, вложения-картинки. Принято предложение ST-0021 про взаимную совместимость Swift Testing и XCTest — API одного фреймворка начинают корректно работать при вызове из другого.
- **fastlane поднимает планку Ruby**: документация заявляет поддержку Ruby 3.0+, предпочтение 3.3+, и предупреждает, что скоро потребуется **Ruby 3.3.0 или новее**. Для CI-образов это отложенная поломка.
- **XcodeGen ожил** после паузы 07.2025 → 03.2026: 2.45.x в марте и 2.46.0 в июле 2026.

## Риски по инструментам — коротко

| Риск | Насколько срочно | Что делать |
|---|---|---|
| CocoaPods trunk read-only 02.12.2026 | **Высокий, срок фиксированный** | Для UCIM — считать CocoaPods-путь наследием: инвентаризация подов, у кого есть SPM-версия; путь SPM+XcodeGen делать основным. Минимум — зафиксировать `Podfile.lock` и вендорить критичное |
| CocoaPods не держит обещанные 2 релиза в год | Высокий | Не закладываться на совместимость с будущими Xcode; проверять `pod install` на каждой новой мажорной версии Xcode самостоятельно |
| XcodeGen — один мейнтейнер, рваный ритм | Средний | Проект жив, замена не нужна, но `project.yml` держать простым и не завязываться на свежие фичи; знать, что альтернатива — нативные Xcode-проекты или Tuist |
| Планка macOS растёт внутри мажора Xcode | Средний | Держать версию Xcode и минимальную macOS явно в документации профиля и в CI; обновление Xcode на CI перестало быть безболезненным |
| Swift Build становится дефолтом SPM | Средний, ещё не наступил | Пока опция — не включать без нужды; следить за релизами Swift, поведение сборки может измениться |
| fastlane потребует Ruby 3.3+ | Низкий, но неизбежный | Проверить версию Ruby в CI-образах заранее |
| Explicit modules по умолчанию (Xcode 26) | Низкий | На brownfield учитывать как вероятную причину новых ошибок сборки |

## Что переписать в профиле прямо сейчас

Действия над текстом манифестов и скиллов лейнов `iva-ios-development-base` (One) и `iva-connect-ios-development-base` (UCIM).

1. **Вписать минимальные Xcode и macOS в скиллы сборки и запуска.** Сейчас `fastlane-build-verification`, `xcode-run-launch` и `ucim-build-verification` говорят только «нужен macOS». Заменить на явную пару — например «Xcode 26.6, macOS Tahoe 26.2 минимум»: планка ОС поднялась внутри одного мажора Xcode (26 требовал macOS 15.6), и без числа в профиле CI и машины разработчиков расходятся молча.
2. **One: «Swift Testing only (no XCTest)» → «для нового unit-кода — Swift Testing; XCTest остаётся для UI- и перформанс-тестов».** Формулировка «There is no XCTest in this repo» в `tester` и `pin-authoring` подана как запрет фреймворка и противоречит прямому указанию Apple: XCTest для UI и перформанса — не временное состояние. Правило «не смешивать API двух фреймворков внутри одного теста» при этом сохранить.
3. **UCIM: CocoaPods-путь пометить наследием с датой.** В `ucim-build-verification` и текстах агентов рядом с «две системы сборки» дописать: trunk становится read-only **02.12.2026**, после этого через него не приедут обновления зависимостей, включая security-фиксы; SPM+XcodeGen — основной путь, CocoaPods выбирается только когда явно сказано.
4. **UCIM: Ruby 2.6 зафиксировать как известный риск, а не как норму окружения.** fastlane заявляет поддержку Ruby 3.0+, предпочтение 3.3+ и предупреждает о скором требовании 3.3.0 — строка «Ruby 2.6» в скилле сборки должна нести эту оговорку, иначе она читается как проверенная конфигурация.
5. **В troubleshooting сборки обоих лейнов добавить explicit modules.** С Xcode 26 они включены по умолчанию для всех Swift-таргетов — это типовая причина новых ошибок сборки на brownfield, и сейчас в списке «первая ошибка — инфраструктурная» её нет.
6. **UCIM: в `pin-authoring` рядом с «match the neighbour's `SWIFT_VERSION` (5.2)» дописать, что это осознанное наследие XcodeGen-конфига**, а не целевое состояние — иначе правило закрепляет Swift 5.2 навсегда и блокирует любой подъём тулчейна.

## Чего не нашёл / где источник слабый

- **У Apple нет отдельного документа «best practices сборки»** — только release notes и справочники API. Нормы приходится собирать из release notes по версиям, а они пишутся как перечень изменений, а не как рекомендация.
- **Формулировку «Swift Testing рекомендован для нового кода» пришлось искать не там, где ожидалось.** На странице самого [Swift Testing](https://developer.apple.com/documentation/testing) её нет — обзор описывает возможности и не упоминает XCTest вовсе. Прямая рекомендация живёт в документации **XCTest**, то есть в старом фреймворке. Ссылаться надо именно туда.
- **Release notes Xcode 26.6 не содержат ничего про сборку, SPM и тесты** — только Coding Intelligence и Organizer. Точка сверки по сборке — release notes мажора (26) и блог swift.org, а не последний минор.
- **CocoaPods: нет отдельного объявления о конце режима поддержки.** Есть два поста 2024 года, дальше — тишина. Оценка «уровень поддержки не выдерживается» построена мной на датах релизов в GitHub, а не на заявлении команды. Это вывод, а не цитата.
- **XcodeGen и fastlane — сторонние проекты**, их «официальность» это официальность самого проекта, не Apple. Единственный надёжный индикатор живости — даты релизов; политики поддержки версий ни у одного из них нет.
- **Не искал** позицию по Tuist, Bazel и Buck2 как альтернативам XcodeGen — задача про существующий стек, но при планировании замены CocoaPods этот вопрос всплывёт.
- **Не проверял на живых страницах** документацию по test plans и `xcodebuild` — фокус был на статусе инструментов; если понадобится раздел про запуск тестов на симуляторе, доки надо будет сверить отдельно.

## Владелец ревью

| Роль | Кто | Дата сверки |
|---|---|---|
| Владелец стека | _не назначен — впишите себя_ | 2026-08-03 |

Правьте эту страницу под свой стек: вычёркивайте источники, которыми не пользуетесь, дописывайте свои, поправьте описание стека, если оно разошлось с реальностью.