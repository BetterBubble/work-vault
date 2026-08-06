---
permalink: tacticum/00-board/bp-swift-lang
---

---
title: Best practices — Swift: язык и платформа (источники)
type: note
status: current
created: 2026-08-03
updated: 2026-08-03
tags: [board, bestpractices, swift]
permalink: tacticum/00-board/bp-swift-lang
---

# Swift: язык и платформа — официальные источники best practices

> **Коротко:** в профиле версия Swift **не зафиксирована вовсе** — в манифестах стоит только `stack: required: [swift]` без версии; актуальная на 2026-08-03 — Swift 6.3 (релиз 24.03.2026), в ходу 6.3.3, 6.4 показан превью на WWDC26; сверять версию языка не с чем. Косвенные пины живут в текстах скиллов и означают Swift 5-семантику: у One — swift-tools 6.1 при `swiftLanguageMode(.v5)` на многих таргетах, у UCIM — `SWIFT_VERSION 5.2`.

## Стек по профилю

Дословно из quickstart-таблиц (репозиторий `tacticum-dev`, `origin/main`).

**IVA One iOS** — строка про лейн `development-ios` (`docs/user_manuals/iva-role-ios-profile-quickstart.md`):

> стек iOS: coder/tester/test-runner (iOS-тексты), fastlane-сборка, Xcode-запуск, Observation-state, fragile-зоны calls/AV и SDK-бриджей, IVALink

**IVA Connect iOS / UCIM** — строка про лейн `iva-connect-ios-development-base` (`docs/user_manuals/iva-role-connect-ios-profile-quickstart.md`):

> стек iOS/UCIM: coder/tester/test-runner (iOS-тексты), сборка SPM+XcodeGen (`ucim-spm/setup`) и CocoaPods-путь, Xcode-запуск на симуляторе `IVA`, MPAK-состояние (Module+Slot), fragile-зоны WebRTC/медиа и генерируемых DTO, реалтайм-транспорт

Профили: роли `iva-role-ios` (0.3.2) и `iva-role-connect-ios` (0.1.0), стек-лейны `iva-ios-development-base` (0.3.0) и `iva-connect-ios-development-base` (0.1.0). Числа в скобках — версии самих профилей, не версии Swift.

### Зафиксирована ли в профиле версия Swift / Xcode

**Нет. Ни в одном из четырёх манифестов версии языка нет** — поле `stack` во всех четырёх выглядит одинаково:

```yaml
stack:
  required:
  - swift
  optional: []
```

Ни `stack`, ни `version`, ни `persona.scope` версию Swift или Xcode не называют: `version` — это версия профиля (0.3.2 / 0.1.0 / 0.3.0 / 0.1.0), `persona.scope` говорит «Swift/Xcode» и «Swift/MPAK/XcodeGen/CocoaPods» без чисел. **Версия языка в профиле не зафиксирована — сверять не с чем**, и это надо читать буквально: агент, установивший профиль, не знает, на какой Swift целиться.

Единственное, что версионировано, — упоминания внутри текстов скиллов, и они описывают репозитории, а не задают норму профиля:

| Где | Что зафиксировано | Продукт |
|---|---|---|
| `ios-module-integration/SKILL.md`, `swift-observation-state/SKILL.md` | «Swift-tools version is 6.1, but many targets stay on `swiftLanguageMode(.v5)`» (например `Core/Networking`) | One |
| `xcode-run-launch/SKILL.md` | минимальный деплой-таргет **iOS 17.0** (`IPHONEOS_DEPLOYMENT_TARGET = 17.0`) | One |
| `ucim-build-verification/SKILL.md`, `ucim-run-launch/SKILL.md`, `pin-authoring/SKILL.md`, агенты `test-runner` | **`SWIFT_VERSION 5.2`**, деплой-таргет **iOS 15.8**, `CODE_SIGN_STYLE Manual` | UCIM |

Практический смысл: обе кодовые базы фактически пишутся по правилам Swift 5, а не Swift 6, при том что тулчейн у One — 6.1. Версии Xcode нет нигде: ни манифесты, ни скиллы сборки и запуска её не называют.

## Что покрывает эта часть

Язык Swift и Apple-платформа для iOS-профилей IVA (`iva-role-ios`, `iva-ios-brownfield`,
`iva-role-connect-ios`): стиль и именование, Swift Concurrency и data-race safety,
Observation-состояние (`@Observable` против `ObservableObject`), SwiftUI/UIKit, версии
языка и SDK, Swift Evolution, машинно-выраженная практика (SwiftLint).

**Не покрывает** (делает другой исследователь): сборка, SPM/XcodeGen/CocoaPods, fastlane,
тесты и CI.

Все ссылки ниже открыты и проверены 2026-08-03. Оговорки по тем, где отдалось не всё
содержимое, — в разделе «Чего не нашёл».

## Источники

| Источник (ссылка) | Тип | Чей | Что именно берём | Как часто обновляется | Сверка |
|---|---|---|---|---|---|
| [API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/) | доки | Swift project (Apple) | Единственный официальный канон именования и дизайна API: ясность в точке использования важнее краткости, документирующий комментарий на каждое объявление, правила меток аргументов | Редко (стабилен с Swift 3) | 2026-08-03 |
| [The Swift Programming Language (TSPL)](https://www.swift.org/documentation/tspl/) | доки | Swift project | Референс языка: A Swift Tour, Language Guide, Language Reference. Разделы Concurrency, Macros, Memory Safety — база правил профиля. Полный текст — [docs.swift.org/swift-book](https://docs.swift.org/swift-book/) | С каждым релизом языка | 2026-08-03 |
| [Swift Concurrency Migration Guide](https://www.swift.org/migration/documentation/migrationguide) | доки | Swift project | Главный документ по concurrency-практике. 10 глав: DataRaceSafety, EnableDataRaceSafety, MigrationStrategy, IncrementalAdoption, CommonProblems, SourceCompatibility, RuntimeBehavior, LibraryEvolution, FeatureMigration. Читаемый исходник — [swiftlang/swift-migration-guide/Guide.docc](https://github.com/swiftlang/swift-migration-guide/tree/main/Guide.docc) | По мере изменений в concurrency | 2026-08-03 |
| [MigrationStrategy.md (сырой текст)](https://raw.githubusercontent.com/swiftlang/swift-migration-guide/main/Guide.docc/MigrationStrategy.md) | доки | Swift project | Механика уровней проверки: targeted (по одной upcoming-фиче — `DisableOutwardActorInference`, `GlobalConcurrency`, `InferSendableFromCaptures`) → complete → Swift 6 mode. Здесь же прямая формулировка «в Swift 5 mode это предупреждения, в Swift 6 — hard errors» | Вместе с гайдом | 2026-08-03 |
| [Swift Evolution (дашборд)](https://www.swift.org/swift-evolution/) | evolution | Swift project | Статусы предложений: что принято, что реализовано и в какой версии | Постоянно | 2026-08-03 |
| [evolution.json](https://download.swift.org/swift-evolution/v1/evolution.json) | evolution | Swift project | Машинный срез всех SE: `id`, `title`, `status.state`, `status.version`, `authors`, `summary`, `implementation`. Единственный способ снять «что принято за год» скриптом. Снимок от 2026-08-02, ~780 КБ | Ежедневно | 2026-08-03 |
| [SE-0466 Control default actor isolation inference](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md) | evolution | Swift project | Implemented (Swift 6.2). Флаг компилятора: неаннотированный код модуля по умолчанию `@MainActor`, а не `nonisolated` | Разово | 2026-08-03 |
| [SE-0461 Run nonisolated async functions on the caller's actor](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md) | evolution | Swift project | Implemented (Swift 6.2). Вводит `nonisolated(nonsending)` и `@concurrent`, включается upcoming-флагом `NonisolatedNonsendingByDefault`. Отменяет поведение SE-0338 | Разово | 2026-08-03 |
| [Swift 6.2 Released](https://www.swift.org/blog/swift-6.2-released/) (15.09.2025) | релизы | Swift project | Перелом в concurrency: default actor isolation на `@MainActor`, `@concurrent`, async в контексте вызывающего, `InlineArray`, `Span`, управление предупреждениями по диагностическим группам | Разово | 2026-08-03 |
| [Swift 6.3 Released](https://www.swift.org/blog/swift-6.3-released/) (24.03.2026) | релизы | Swift project | Текущий мажор: `@c`, module name selectors (`Swift::Task`), `@specialize` / `@inline(always)` / `@export(implementation)`, официальный Swift SDK для Android | Разово | 2026-08-03 |
| [What's new in Swift: June 2026](https://www.swift.org/blog/whats-new-in-swift-june-2026/) (02.07.2026) | релизы/дайджест | Swift project | Актуальная точка отсчёта: в ходу 6.3.3, 6.4 показан превью на WWDC26. Свежие SE: 0474 (yielding accessors), 0527 (`RigidArray`/`UniqueArray`), 0529 (`FilePath` в stdlib) | Ежемесячно | 2026-08-03 |
| [Observation framework](https://developer.apple.com/documentation/observation) | доки | Apple | Рамка нового состояния: `@Observable`, `@ObservationIgnored`, `withObservationTracking`. Доступно с iOS 17 | С релизом SDK | 2026-08-03 |
| [Migrating from ObservableObject to the Observable macro](https://developer.apple.com/documentation/swiftui/migrating-from-the-observable-object-protocol-to-the-observable-macro) | доки | Apple | Позиция Apple прямым текстом: убрать conformance `ObservableObject` и `@Published`, `@ObservedObject` → обычное свойство, `@StateObject` → `@State`, исключения через `@ObservationIgnored` | С релизом SDK | 2026-08-03 |
| [What's New in iOS (Apple)](https://developer.apple.com/ios/whats-new/) | релизы | Apple | Сводка цикла iOS 27 / Xcode 27: SwiftUI (document-based apps, переупорядочиваемые списки и сетки, ленивая подгрузка подвью с префетчем), UIKit (раскладки под iPhone Mirroring), Foundation Models, Core AI, App Intents | Раз в цикл | 2026-08-03 |
| [iOS & iPadOS 27 Release Notes](https://developer.apple.com/documentation/ios-ipados-release-notes/ios-ipados-27-release-notes) | релизы | Apple | Построчные изменения, депрекейты и known issues по бетам (на 03.08.2026 — beta 4) | Каждая бета | 2026-08-03 |
| [SwiftLint Rule Directory](https://realm.github.io/SwiftLint/rule-directory.html) | линтер | Realm (не Apple) | Дефолтный набор правил — практика, выраженная машиной. Версия документации 0.65.0: ~120 правил по умолчанию, ~75 opt-in, 5 analyzer. Из дефолтных: `force_cast`, `identifier_name`, `type_name`, `line_length`, `file_length`, `cyclomatic_complexity`, `control_statement`, `duplicate_imports`, `colon`, `comma`, `trailing_whitespace` | С каждым релизом линтера | 2026-08-03 |
| [realm/SwiftLint](https://github.com/realm/SwiftLint) | линтер | Realm | Конфигурация, `swiftlint rules` (список правил своей версии), `swiftlint generate-docs`, интеграция в сборку | Постоянно | 2026-08-03 |

## Что изменилось за последний год

Отсчёт от лета 2025 к августу 2026.

1. **Актуальная версия — Swift 6.3 (релиз 24.03.2026), в ходу 6.3.3; 6.4 показан превью
   на WWDC26.** За год прошли два мажора: 6.2 (15.09.2025) и 6.3. Любая рекомендация
   «целимся в Swift 6.0/6.1» устарела.

2. **Data-race safety: разница не между версиями компилятора, а между language mode.**
   В Swift 6 language mode нарушения изоляции — **ошибки компиляции**; тот же анализ в
   Swift 5 mode остаётся предупреждениями. Гайд прямо предлагает промежуточный путь:
   сидеть в Swift 5 mode, включать upcoming-фичи по одной (targeted), затем complete,
   и только потом переключать mode — «чтобы сборка и тесты оставались рабочими».
   Практический вывод для brownfield: переключение language mode — отдельное решение
   с датой, а не побочный эффект обновления Xcode.

3. **Swift 6.2 развернул подход к concurrency: «approachable concurrency».** Два принятых
   предложения меняют то, чему учили в 2025-м:
   - **SE-0466** — модуль можно целиком объявить `@MainActor` по умолчанию. Совет
     «расставляйте `@MainActor` руками по вью-моделям» заменяется на «включите default
     actor isolation = MainActor для UI-таргета».
   - **SE-0461** — `nonisolated async` больше не прыгает на глобальный executor, а
     исполняется на акторе вызывающего (флаг `NonisolatedNonsendingByDefault`; предложение
     прямо отменяет поведение SE-0338 от 2022 года). Появились `nonisolated(nonsending)`
     (явно «остаюсь у вызывающего») и `@concurrent` (явно «ухожу в параллель»). Код 2025 года, который
     полагался на то, что `nonisolated async` автоматически уходит с главного потока,
     **меняет поведение**, а не только диагностику — это самая опасная точка для
     хрупких зон calls/AV.

4. **Observation окончательно вытеснил `ObservableObject`.** У Apple есть отдельная
   миграционная статья, и она односторонняя: `@Observable` + `@State` вместо
   `ObservableObject` + `@StateObject`/`@ObservedObject`/`@Published`. Для brownfield это
   правило перехода, а не вкусовщина. Порог — iOS 17.

5. **Новые примитивы памяти и производительности вошли в stdlib.** 6.2 — `InlineArray`
   (фиксированный размер, без кучи, синтаксис `[40 of Sprite]`) и `Span` (безопасный доступ
   к непрерывной памяти, нулевой рантайм-оверхед); июнь 2026 — принятые SE-0527
   (`RigidArray`, `UniqueArray`) и SE-0529 (`FilePath` переехал из swift-system в stdlib).
   Для буферов аудио и видео это означает, что `UnsafeBufferPointer` перестал быть
   единственным вариантом и в новом коде должен обосновываться.

6. **Платформа: iOS 27 / Xcode 27, объявлены на WWDC26 (08.06.2026), релиз ожидается
   в сентябре 2026.** На 03.08.2026 SDK в бете 4 — прод-таргет сейчас ещё iOS 26, но окно
   перехода открыто. По SwiftUI в цикле: document-based apps, переупорядочиваемые списки
   и сетки, ленивая подгрузка подвью с префетчем; по UIKit — раскладки под iPhone Mirroring.

7. **Управление предупреждениями стало гранулярным (6.2).** Диагностические группы можно
   поштучно повышать до ошибок или глушить через настройки в манифесте пакета. Это заменяет
   грубое «-warnings-as-errors» и позволяет закреплять правила профиля компилятором,
   а не только линтером.

## Что переписать в профиле прямо сейчас

Действия над текстом манифестов и скиллов лейнов `iva-ios-development-base` (One) и `iva-connect-ios-development-base` (UCIM).

1. **Дописать версию языка туда, где сейчас голое `swift`.** В `persona.scope` и в скиллах вместо «Swift/Xcode» — «Swift 6.3 toolchain, language mode 5» для One и «Swift 6.3 toolchain, `SWIFT_VERSION 5.2`» для UCIM. Без этого агент подставляет версию по своим данным, а не по профилю, и упирается в `swiftLanguageMode(.v5)` уже на сборке.
2. **One: `@Observable` + `@State` вместо `ObservableObject`** — в `swift-observation-state` и текстах `coder` закрепить как правило перехода со ссылкой на миграционную статью Apple (`@ObservedObject` → обычное свойство, `@StateObject` → `@State`, исключения через `@ObservationIgnored`). Порог iOS 17 у One уже выдержан (`IPHONEOS_DEPLOYMENT_TARGET = 17.0`).
3. **UCIM: `VM : ObservableObject` в `coder.md` оставить, но подписать причину — деплой-таргет iOS 15.8, ниже порога Observation (iOS 17).** Сейчас конструкция подана как местная конвенция; без причины следующий апдейт скилла «осовременит» её и сломает сборку. Правило звучит как «`ObservableObject` здесь вынужденный, пересматривать при подъёме деплой-таргета».
4. **Concurrency: «расставляйте `@MainActor` по вью-моделям» → «включите default actor isolation = MainActor для UI-таргета» (SE-0466).** Ручная расстановка остаётся только там, где модуль сознательно не `@MainActor`.
5. **В скиллы хрупких зон calls/AV добавить SE-0461.** Правило: `nonisolated async` больше не уходит на глобальный executor сам — явно писать `nonisolated(nonsending)` или `@concurrent`. Это меняет поведение существующего кода 2025 года, а не только диагностику, и в calls/AV это самая опасная точка.
6. **Правило «`StrictConcurrency` не ослаблять» дополнить порядком перехода:** targeted upcoming-фичи по одной → complete → Swift 6 mode, с явной датой смены language mode как отдельного решения, а не побочного эффекта обновления Xcode.

## Чего не нашёл / где источник слабый

- **`https://www.swift.org/documentation/concurrency/` — не самостоятельный документ.**
  В карте документации swift.org он подписан как «Enabling Complete Concurrency Checking»,
  но фактически это редирект на миграционный гайд. Отдельной страницы про уровни проверки
  нет — всё в гайде. Ссылаться надо на гайд.
- **Таблицы «что стало ошибкой в 6.2 / 6.3» официально не существует.** Собирается только
  вручную из SourceCompatibility гайда и текстов конкретных SE. Отдельный проход, если
  профилю нужен точный список.
- **Depreкейты iOS 27 SDK построчно не сверил.** Release notes Apple рендерятся на клиенте,
  обычным fetch отдаётся только заголовок. Страница живая, содержимое смотреть глазами или
  в Xcode. То же ограничение у `developer.apple.com/documentation/observation` — страница
  есть, тело не отдаётся.
- **Официальных best practices Apple по calls/AV (CallKit, AVFoundation, PushKit) в связке
  со strict concurrency нет.** Есть API-референс, есть общий concurrency-гайд, а документа
  «как безопасно дёргать делегаты CallKit из акторного кода» не существует. Самая
  чувствительная для IVA зона закрывается собственной нормой профиля, а не ссылкой.
- **Swift Evolution как «принято за последний год» глазами не снимается.** Дашборд
  swift.org и листинг директории на GitHub обрезаются. Рабочий способ — `evolution.json`
  (проверен, живой) с фильтром по `status.version`; я взял из него точечно SE-0461 и
  SE-0466, полный годовой срез не строил.
- **SwiftLint — не Apple.** Дефолтный набор выражает практику сообщества, версия 0.65.0
  может отставать от синтаксиса Swift 6.3, а часть правил конфликтует с новым
  concurrency-кодом. Годится как проверяемый минимум, не как канон. Единственный
  не-Apple источник в списке — по прямому разрешению постановки.

## Владелец ревью

| Роль | Кто | Дата сверки |
|---|---|---|
| Владелец стека | _не назначен — впишите себя_ | 2026-08-03 |

Правьте эту страницу под свой стек: вычёркивайте источники, которыми не пользуетесь, дописывайте свои, поправьте описание стека, если оно разошлось с реальностью.