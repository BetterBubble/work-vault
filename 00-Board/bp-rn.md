---
title: Best practices — React Native (источники)
type: note
status: current
created: 2026-08-03
updated: 2026-08-03
tags: [board, bestpractices, react-native]
permalink: tacticum/00-board/bp-rn
---

# React Native — официальные источники best practices

> **Коротко:** в профиле — RN 0.77 (21.01.2025); актуальная — 0.86.x (0.86.0 — 09.06.2026, в npm 0.86.2); 0.77 вне поддержки с лета 2025, security-патчей нет.

## Стек по профилю

Дословно из `TABLE_BLURBS` в `scripts/wiki_sync_manuals.py` (репозиторий TacticumApps/dev):

> React Native 0.77 + React + TypeScript; Metro / Hermes / Gradle (New Architecture); react-native-web; Jest + RNTL

Приложение `rn-main`: mail/chat/calendar/contacts/disk/tasks с дизайн-токенами. Профиль агента — `iva-rn-brownfield`.

Главное, что нужно знать до чтения остального: **0.77 вышел 21 января 2025 года и снят с поддержки.** Актуальная ветка на 03.08.2026 — 0.86.x, в npm `react-native@0.86.2`. Разрыв — девять минорных релизов и примерно полтора года.

## Источники

| Источник (ссылка) | Тип | Чей | Что именно берём | Как часто обновляется | Дата сверки |
|---|---|---|---|---|---|
| [reactnative.dev/docs/releases](https://reactnative.dev/docs/releases) | релизы | Meta / React Foundation, ядро RN | Таблица всех веток со статусом поддержки (Future / Active / End of Cycle / Unsupported) и датами. **Единственный источник правды по вопросу «наша версия ещё живая?»** | Каждый релиз, ~раз в 2 месяца | 2026-08-03 |
| [reactnative.dev/blog](https://reactnative.dev/blog) | блог релизов | ядро RN | Разбор каждого минора: что нового, что ломается, как мигрировать. Здесь живёт полный список breaking changes с примерами кода | ~раз в 2 месяца | 2026-08-03 |
| [github.com/reactwg/react-native-releases](https://github.com/reactwg/react-native-releases) → [docs/support.md](https://github.com/reactwg/react-native-releases/blob/main/docs/support.md) | релизы / политика | Release Working Group RN | Формальная политика поддержки: живы текущий минор + два предыдущих. Расписание веток, pick requests, RC-каналы | Непрерывно | 2026-08-03 |
| [github.com/react/react-native/blob/main/CHANGELOG.md](https://github.com/react/react-native/blob/main/CHANGELOG.md) | релизы | ядро RN | Машинный полный changelog по версиям — то, что не влезло в блог. Обратить внимание на новый путь: репозиторий переехал `facebook/react-native` → `react/react-native` | Каждый релиз | 2026-08-03 |
| [react-native-community.github.io/upgrade-helper](https://react-native-community.github.io/upgrade-helper/) | миграция | RN community, официальный инструмент | Дифф нативных и конфигурационных файлов между двумя любыми версиями с комментариями «зачем». Основной инструмент при подъёме версии | При каждом релизе | 2026-08-03 |
| [reactnative.dev/docs/upgrading](https://reactnative.dev/docs/upgrading) | доки | ядро RN | Процедура апгрейда, две дорожки (голый RN и Expo). Рекомендация поднимать версии по одной, а не прыжком | Редко | 2026-08-03 |
| [reactnative.dev/architecture/landing-page](https://reactnative.dev/architecture/landing-page) | доки | ядро RN | New Architecture: зачем, JSI, синхронный layout, конкурентный рендер. **Каноническое место** — гайды старой рабочей группы объявлены устаревшими и отсылают сюда | По мере изменений | 2026-08-03 |
| [github.com/reactwg/react-native-new-architecture](https://github.com/reactwg/react-native-new-architecture) | миграция | New Architecture WG | Turbo Native Modules и Fabric Native Components — как писать свои нативные модули. **Гайды помечены deprecated**, брать только как исторический контекст и обсуждения | Затухает | 2026-08-03 |
| [reactnative.dev/docs/typescript](https://reactnative.dev/docs/typescript) | доки | ядро RN | Настройка TS в RN, база `@react-native/typescript-config`, path aliases. Обновлена 07.05.2026 | Редко | 2026-08-03 |
| [react.dev](https://react.dev) + [react.dev/reference/rules](https://react.dev/reference/rules) | доки | React team | Rules of React — три раздела: компоненты и хуки чисты, React сам их вызывает, Rules of Hooks. **Это то, по чему надо переписывать «стиль 2025»**, а не по подборкам паттернов | По мере изменений | 2026-08-03 |
| [react.dev/blog](https://react.dev/blog) | блог | React team | Анонсы React (19.2, React Compiler v1.0), security-бюллетени, React Foundation | Несколько раз в год | 2026-08-03 |
| [react.dev/versions](https://react.dev/versions) | релизы | React team | Таблица версий React с датами. Последний стабильный — 19.2, патч 19.2.7 от 01.06.2026 | Каждый патч | 2026-08-03 |
| [react.dev/learn/react-compiler/installation](https://react.dev/learn/react-compiler/installation) | доки | React team | React Compiler: установка через Babel, а значит и через Metro в RN. Явно сказано, что RN поддерживается | По мере изменений | 2026-08-03 |
| [github.com/react/react-native/tree/main/packages/eslint-config-react-native](https://github.com/react/react-native/tree/main/packages/eslint-config-react-native) | линтер | ядро RN | `@react-native/eslint-config` — официальный конфиг ESLint + Prettier. Версия следует за RN (сейчас 0.86.2). Flat config: `require('@react-native/eslint-config/flat')`, старый путь — `"extends": "@react-native"` | Каждый релиз RN | 2026-08-03 |
| [github.com/react/react/blob/main/packages/eslint-plugin-react-hooks/README.md](https://github.com/react/react/blob/main/packages/eslint-plugin-react-hooks/README.md) | линтер | React team | `eslint-plugin-react-hooks`, сейчас 7.1.1 (31.07.2026). Пресет `recommended-latest` включает правила React Compiler: purity, immutability, set-state-in-effect, set-state-in-render, preserve-manual-memoization, refs, static-components и другие. **Практика, выраженная машиной** — самый дешёвый способ поймать устаревший стиль | Часто | 2026-08-03 |
| [metrobundler.dev](https://metrobundler.dev/) / [конфигурация](https://metrobundler.dev/docs/configuration) | доки | Meta | Metro: конфиг, резолвер, API. В npm сейчас `metro@0.87.0` | По мере изменений | 2026-08-03 |
| [github.com/facebook/hermes](https://github.com/facebook/hermes) + [reactnative.dev/docs/hermes](https://reactnative.dev/docs/hermes) | доки | Meta | Hermes. **Отдельного сайта доков больше нет**: `hermesengine.dev` редиректит на README в репозитории. Страница в доках RN отвечает только на «как проверить и как отключить» | Редко | 2026-08-03 |
| [oss.callstack.com/react-native-testing-library](https://oss.callstack.com/react-native-testing-library/) | доки | Callstack (RN partner) | RNTL, версия доков 14.x, в npm `@testing-library/react-native@14.0.1`. Есть разделы Common Mistakes, How to Query, Understanding Act и отдельный **LLM Guidelines** — прямо то, что кладётся в скилл агента | Регулярно | 2026-08-03 |
| [github.com/react-native-community/discussions-and-proposals](https://github.com/react-native-community/discussions-and-proposals) | RFC | RN community + ядро | RFC-процесс React Native, заметки core meetings, ранние сигналы «куда едет стек» | Непрерывно | 2026-08-03 |
| [github.com/reactjs/rfcs](https://github.com/reactjs/rfcs) | RFC | React team | RFC React. Команда честно предупреждает, что не гарантирует своевременное ревью — ценность в обсуждениях, не в статусах | Медленно | 2026-08-03 |
| [necolas.github.io/react-native-web](https://necolas.github.io/react-native-web/docs/) + [discussions/2816](https://github.com/necolas/react-native-web/discussions/2816) | доки / статус | necolas | Доки RNW и тред о будущем проекта. Второе важнее первого — см. ниже | Редко | 2026-08-03 |

Итого 20 источников, все открыты и проверены 03.08.2026 (HTTP 200).

## Насколько отстал RN 0.77

**Отстал критически. Версия снята с поддержки и не получает даже security-патчей.**

- 0.77.0 — 21 января 2025. Текущая ветка — 0.86.x (0.86.0 — 9 июня 2026, в npm 0.86.2).
- Между ними девять минорных релизов: 0.78, 0.79, 0.80, 0.81, 0.82, 0.83, 0.84, 0.85, 0.86.
- Политика поддержки: живы текущий минор и два предыдущих. Сейчас это 0.86 (Active), 0.85 (Active), 0.84 (End of Cycle). Всё, что 0.83 и ниже, — **Unsupported**: новых релизов не будет, исключения только под очень важные регрессии.
- 0.77 выпал из поддержки ещё летом 2025 года, когда вышел 0.80.
- 0.87 уже отрезан в ветку 06.07.2026, релиз назначен на 10.08.2026 — то есть отставание вырастет до десяти релизов через неделю.

Три вещи, которые ломают наивный апгрейд «поднять версию в package.json»:

1. **Old Architecture перестала существовать.** С 0.82 (октябрь 2025) флаги `newArchEnabled=false` и `RCT_NEW_ARCH_ENABLED=0` просто игнорируются — New Architecture включена всегда. В 0.84 legacy-код начали физически вырезать: на iOS он компилируется out по умолчанию, на Android удалены классы (`LazyReactPackage`, `BridgeDevSupportManager`, классы layout-анимаций). Официальный маршрут миграции — сначала подняться до **0.81 включительно** (последняя версия с Legacy Architecture), там включить New Architecture и всё оттестировать, и только потом идти на 0.82+. Прыгать с 0.77 сразу на 0.86 нельзя.
2. **Deep imports вида `react-native/Libraries/...` мертвы.** С 0.80 они дают предупреждения, с 0.82 пути удаляются. В проекте возрастом с 0.77 таких импортов почти наверняка много, и это работа отдельным проходом.
3. **Требования окружения выросли.** Node ≥20.19.4 с 0.81 и ≥22.11 с 0.84; Xcode ≥16.1 с 0.81; Gradle 9.0 с 0.82; target Android 16 (API 36) с обязательным edge-to-edge с 0.81.

### Статус New Architecture

Вопрос «включать ли» больше не стоит: она по умолчанию с 0.76 и единственная возможная с 0.82. Формулировка в профиле «Gradle (New Architecture)» верна по факту, но её надо переписать — сейчас она читается как выбор, а это данность. Агент должен исходить из того, что Fabric и TurboModules — норма, а любые советы «сделай bridge-модуль по-старому» — прямая ошибка.

### Судьба react-native-web

**Проект в режиме поддержки, не развития.** Автор (necolas) в треде [discussions/2816](https://github.com/necolas/react-native-web/discussions/2816) прямо пишет: *«I will continue to review PRs and merge fixes. But I don't expect to put significant time into major development initiatives»*. Его основной фокус — `react-strict-dom`. Он говорил о готовности взять мейнтейнеров от партнёров RN (Shopify Mobile, Expo) через React Discord, но публичного результата в треде нет: последний вопрос «чем кончилось» задан в марте 2026 и остался без ответа.

Фактическая жизнь есть: `react-native-web@0.21.2` опубликован 02.06.2026, багфиксы идут. Но разрыв с ядром RN растёт — RNW не догоняет ни New Architecture, ни новые API. Формально ничего не депрекейтнуто, планового вывода из эксплуатации не объявлено. Для нас это значит: **на RNW можно продолжать, закладываться на его развитие — нельзя**, и в профиле стоит пометить его как риск с явным альтернативным путём (react-strict-dom либо отдельный веб-клиент).

## Что изменилось за последний год

Окно август 2025 — август 2026, релизы 0.81…0.86.

1. **Конец Old Architecture (0.82, октябрь 2025).** Первая версия, работающая только на New Architecture. Одновременно Gradle 8 → 9.0, `ReactNativeFeatureFlags` ушёл в private API, и — тихая, но болезненная правка — **необработанные reject промисов теперь пишут `console.error`** вместо молчания. После апгрейда ждать всплеска ошибок в мониторинге: это не новые баги, это ранее проглоченные старые.
2. **Hermes V1 (0.82 опционально → 0.84 по умолчанию).** Новый компилятор и VM: заявлено 3–9% быстрее загрузка бандла, до 7.6% лучше TTI. Миграции не требует, откат — через override пакета `hermes-compiler` на 0.15.0. JavaScriptCore при этом вынесен из ядра ещё в 0.81 в community-пакет.
3. **Стабилизация публичного JS API и Strict TypeScript API (с 0.80, доводится дальше).** Импортировать разрешено только из корня `'react-native'`. Строгие типы генерируются из исходников RN, а не пишутся руками; включаются через `"customConditions": ["react-native-strict-api"]` в `tsconfig.json`. Объявлено, что через пару релизов строгий API станет умолчанием, а старые типы удалят. **Для агентского профиля это самое практичное изменение**: включённый strict API машинно запрещает писать по-старому.
4. **React 19.2 и React Compiler v1.0.** RN 0.83 принёс React 19.2 с `<Activity>` и `useEffectEvent`, 0.84 — 19.2.3. React Compiler вышел стабильным в октябре 2025, его правила переехали в `eslint-plugin-react-hooks` (пресет `recommended-latest`). Ручная мемоизация перестала быть добродетелью — компилятор её делает сам, а линтер ругается на `preserve-manual-memoization`.
5. **Инструментарий и сборка.** Precompiled iOS-бинарники стали умолчанием в 0.84 (раньше — экспериментальный флаг в 0.81, до 10× быстрее компиляция). ESLint 9 flat config поддержан с 0.84. **Jest-пресет переименован в 0.85**: `preset: 'react-native'` → `preset: '@react-native/jest-preset'` — прямо касается нашей связки Jest + RNTL. `StyleSheet.absoluteFillObject` удалён там же. `SafeAreaView` из ядра депрекейтнут ещё в 0.81 в пользу `react-native-safe-area-context`.
6. **Организационное.** В феврале 2026 запущен React Foundation под Linux Foundation, и в 0.86 репозиторий RN переехал `facebook/react-native` → `react/react-native` (старые ссылки редиректят, но новые доки и changelog надо брать по новому пути). Отдельно: два последних релиза, 0.83 и 0.86, вышли **без user-facing breaking changes** — то есть после того, как мы один раз доберёмся до 0.84+, дальнейшие апгрейды станут дешёвыми.

## Что переписать в профиле прямо сейчас

- Переписать «Gradle (New Architecture)» → «New Architecture (единственная возможная с 0.82)» и запретить в скиллах советы «сделай bridge-модуль по-старому»: флаги `newArchEnabled=false` / `RCT_NEW_ARCH_ENABLED=0` игнорируются с 0.82, в 0.84 legacy-классы физически удалены. [architecture](https://reactnative.dev/architecture/landing-page)
- Запретить deep imports `react-native/Libraries/...` → импорт только из корня `'react-native'`: с 0.80 предупреждение, с 0.82 пути удалены. Отдельным проходом вычистить их в `rn-main`. [releases](https://reactnative.dev/docs/releases)
- Заменить в jest-конфиге `preset: 'react-native'` → `preset: '@react-native/jest-preset'` (переименован в 0.85), а `StyleSheet.absoluteFillObject` (удалён там же) и `SafeAreaView` из ядра (депрекейтнут в 0.81) → `react-native-safe-area-context`. [blog](https://reactnative.dev/blog)
- Убрать ручную мемоизацию как признак хорошего кода: `useMemo`/`useCallback` — escape hatch, а не норма. Подключить `eslint-plugin-react-hooks` пресетом `recommended-latest` (правила React Compiler ловят устаревший стиль машинно). [eslint-plugin-react-hooks](https://github.com/react/react/blob/main/packages/eslint-plugin-react-hooks/README.md)
- Записать маршрут апгрейда как ступенчатый: 0.77 → **0.81** (последняя с Legacy Architecture, там включить и оттестировать New Architecture) → 0.82+ → 0.86. Прыжок «поднять версию в package.json» запретить прямо в тексте скилла. [upgrading](https://reactnative.dev/docs/upgrading)
- Пометить `react-native-web` как риск с альтернативным путём (react-strict-dom либо отдельный веб-клиент): автор публично держит проект в режиме поддержки, не развития. [discussions/2816](https://github.com/necolas/react-native-web/discussions/2816)

## Чего не нашёл / где источник слабый

- **Нет официального документа «RN 0.77 → 0.86 одним куском».** Upgrade Helper даёт дифф между любой парой версий, но осмысленного пути с девятью релизами не существует по определению: 0.81 — обязательная промежуточная остановка из-за архитектуры. Разбивку на шаги придётся составить самим, официального аналога нет.
- **У Hermes больше нет сайта документации.** `hermesengine.dev` редиректит на README репозитория. Всё, что осталось официального, — README, исходники и короткая страница в доках RN. Для профиля, где Hermes указан явно, это дыра: подробностей про Hermes V1 в доках RN нет вообще, только в блог-постах 0.82 и 0.84.
- **Официальных «best practices» как жанра у RN нет.** Есть доки, релиз-ноуты и линтеры. Всё, что называет себя «RN best practices 2026», — сторонние подборки, и в каталог они не входят намеренно. Ближайший официальный аналог — Rules of React на react.dev плюс пресет `recommended-latest`; на них и надо опирать скилл агента.
- **Судьба react-native-web не решена публично.** Есть заявление автора о режиме поддержки и оборванный на полуслове разговор про передачу мейнтейнерства. Ни дорожной карты, ни дат, ни официального статуса. Это не «источник слабый», это «источника нет»: решение по вебу придётся принимать нам на неполных данных.
- **Точный состав React в 0.85 и 0.86 не подтверждён.** Для 0.83 (19.2) и 0.84 (19.2.3) версии названы в блоге явно; в постах 0.85 и 0.86 их нет. Если понадобится точно — смотреть `package.json` соответствующего тега в репозитории.
- **`oss.callstack.com` отдаёт 403 части автоматических клиентов** (сайт живой, curl получает 200). Если поставим агенту периодическую сверку по этому источнику — надо учесть, иначе проверка будет ложно падать.
- **Даты релизов 0.87–0.89 в таблице — плановые** (0.87 на 10.08.2026, 0.88 на 12.10.2026, 0.89 на 07.12.2026), а не свершившиеся. Сверять при следующем обходе.

## Владелец ревью

| Роль | Кто | Дата сверки |
|---|---|---|
| Владелец стека | _не назначен — впишите себя_ | 2026-08-03 |

Правьте эту страницу под свой стек: вычёркивайте источники, которыми не пользуетесь, дописывайте свои, поправьте описание стека, если оно разошлось с реальностью.
