---
title: Best practices — Firebird web (источники)
type: note
status: current
created: 2026-08-03
updated: 2026-08-03
tags: [board, bestpractices, firebird, react]
permalink: tacticum/00-board/bp-firebird
---

# Firebird web — официальные источники best practices

> **Коротко:** в профиле — React 19, Vite 8, Lerna 9, Jest 30, RTK 2, Effect 3 (версия TypeScript в профиле не указана); актуальные на 2026-08-03 — React 19.2.7, Vite 8.2, **Lerna 10** (29.07.2026), Jest 30.4.2, Effect 3.22.1; всё поддерживается, на мажор отстаёт только Lerna.

## Стек по профилю

Что удалось установить по репозиторию `TacticumApps/dev` (`origin/main`, коммит `f7fa5e2`).
Указано, откуда именно.

| Слой | Что в стеке | Откуда это следует |
|---|---|---|
| Фреймворк | **React 19** SPA | `docs/user_manuals/firebird-web-brownfield-profile-quickstart.md` («React 19 SPA + Vite 8»); агент `coder.md` в `templates/firebird-web-development-base/ingredients/agents/coder.md` («Stack: React 19 + Vite 8») |
| Язык | **TypeScript** strict — `exactOptionalPropertyTypes`, `verbatimModuleSyntax`, обязательные явные return-типы, без `export default` | brownfield-quickstart; скилл `vite-lerna-build-verification`. **Конкретная версия TS в репозитории Firebird нигде не названа** |
| Сборка | **Vite 8** + `@vitejs/plugin-react` + `@vitejs/plugin-legacy` + terser | скилл `vite-lerna-build-verification` («Vite 8 with @vitejs/plugin-react + @vitejs/plugin-legacy + terser») |
| Монорепо | **Lerna 9 + npm workspaces**, 20 пакетов, `package-lock.json`, алиасы `@firebird/*` | скилл `vite-lerna-build-verification` («Lerna 9 + npm workspaces … NOT yarn/bun») |
| Стейт | **Redux Toolkit 2** — классические слайсы + фабрики async-thunk (`createCalendarAsyncThunk`, никогда сырой `createAsyncThunk`), `build*Actions(builder)` | скилл `rtk-effect-state` |
| FP-слой | **Effect 3** — `Either` / `Option` / `Match`, namespace-импорты `import { Either, Option, Match } from "effect"` | скилл `rtk-effect-state`; brownfield-quickstart |
| UI | **MUI 9 + SCSS BEM**, свои dark/light темы; `sx` и `styled()` **запрещены** | brownfield-quickstart |
| Тесты | **Jest 30 + ts-jest + @testing-library/react + jest-environment-jsdom**; политика «тестируем только чистые функции, JUMP-сессию не мокаем, e2e нет» | скилл `jest-rtl-ui-testing` |
| Линт-гейт | ESLint (`--max-warnings 0`, `--report-unused-disable-directives`), stylelint, prettier (**табы**), madge (circular deps), husky + lint-staged | скилл `vite-lerna-build-verification` |
| i18n | **react-i18next** — хардкод строк запрещён | `coder.md`, правило 5 |
| Витрина компонентов | **Storybook**, порт 6006, стори обязательна для shared-компонента | скилл `vite-run-launch`; `coder.md` |
| Runtime CI | **node:24** | `tacticum-workflow.toml`, «Environment GATE: node --version (CI node:24)» |
| Сеть | **JUMP** — проприетарный бинарный/JSON-протокол ИВА, НЕ REST/OpenAPI | скилл `jump-protocol-fragile-zone` |

**RTK — это действительно Redux Toolkit, и да, это React.** Подтверждено дословно: `coder.md`
пишет «Implement React/TypeScript code», `rtk-effect-state` описывает `createSlice`/`extraReducers`/
`createAsyncThunk` и хуки `useCalendarDispatch`. Формулировка «RTK effect-state» в quickstart-е роли
означает связку **Redux Toolkit + библиотека Effect**, а не «effect» в смысле side-effect.

**Что такое JUMP.** Это **не** библиотека и не публичный стандарт — это внутренний протокол общения
web-клиента Firebird с бэкендом. Клиент живёт в пакетах `jump-client-core` (транспорт-агностик) и
`jump-client-browser`. Вызов выглядит как `session.request({ method: "calendarOpen", payload })` и
возвращает `Either`, который декодируется схемой через `createBackendDecoder(schema)`. Контракты
лежат в `docs/JUMP/` **внутри репозитория Firebird**, наружу не опубликованы. Практический смысл:
компилятор эту границу не видит, «зелёный билд ≠ рабочий вызов», после правок обязателен smoke
против живого стенда. **Внешних best practices по JUMP не существует и быть не может** — единственный
источник правды здесь корневой `CLAUDE.md` репозитория Firebird и `docs/JUMP/`.

### Что осталось неясным

- **Роутер не назван нигде.** Ни `react-router`, ни TanStack Router, ни собственное решение — по
  всем шаблонам `firebird-*` нет ни одного упоминания роутинга (grep по `react-router|tanstack|
  routing` даёт только `firebird-local-knowledge-routing`, это про маршрутизацию к знаниям, не URL).
  Не додумываю: **роутер в профиле не зафиксирован**.
- **Версии TypeScript, ESLint, Storybook, Node в самом репозитории Firebird** — в профиле не названы
  (кроме «CI node:24»). Соответственно, отставание по этим позициям я оценить не могу, только
  обозначить как зону проверки.
- **Набор ESLint-плагинов** назван частично: из правил видны `@typescript-eslint/explicit-function-
  return-type` и `unicorn/prevent-abbreviations` → typescript-eslint и eslint-plugin-unicorn точно
  есть. Есть ли `eslint-plugin-react-hooks` — **из профиля не следует**, а это ключевой линтер стека.

## Источники

Все ссылки открыты и проверены 2026-08-03. Дата сверки для всех строк — **2026-08-03**.

### React

| Источник | Тип | Чей | Что берём | Как часто обновляется |
|---|---|---|---|---|
| [react.dev](https://react.dev) | Документация | React Team (React Foundation) | Learn + Reference — канон по хукам, эффектам, паттернам | непрерывно |
| [react.dev/reference/rules](https://react.dev/reference/rules) | Правила | React Team | «Rules of React»: чистота компонентов и хуков, иммутабельность props/state, правила хуков. Прямой вход для скилла | редко (стабильный документ) |
| [react.dev/blog](https://react.dev/blog) | Блог релизов | React Team | Официальные анонсы: React 19.2, React Compiler v1.0, React Foundation | ~5–10 постов в год |
| [react.dev/versions](https://react.dev/versions) | Матрица версий | React Team | Что сейчас stable, какие патчи вышли | при каждом релизе |
| [react.dev/reference/eslint-plugin-react-hooks](https://react.dev/reference/eslint-plugin-react-hooks) | Линтер | React Team | 17 правил `recommended`-пресета: `rules-of-hooks`, `exhaustive-deps`, `purity`, `refs`, `immutability`, `static-components`. **Практика, выраженная машиной** | вместе с плагином |
| [react.dev/learn/react-compiler/installation](https://react.dev/learn/react-compiler/installation) | Инструкция | React Team | Как подключить React Compiler в Vite-проекте (нужен `@vitejs/plugin-react` ≥6.0.0 + `@rolldown/plugin-babel`) | при изменении интеграций |
| [github.com/facebook/react/releases](https://github.com/react/react/releases) | Release notes | React Team | Патчи React и релизы `eslint-plugin-react-hooks` | каждые ~4–6 недель |
| [github.com/reactjs/rfcs](https://github.com/reactjs/rfcs) | RFC-процесс | React Team | Куда едет фреймворк. **Честная оговорка самих авторов:** «Currently, the React Team cannot commit to reviewing RFCs in a timely manner», большинство community-RFC не мержится — это источник направления, а не расписания | нерегулярно |

### Redux / Redux Toolkit

| Источник | Тип | Чей | Что берём | Как часто обновляется |
|---|---|---|---|---|
| [redux.js.org/style-guide](https://redux.js.org/style-guide/) | **Style Guide** | Redux maintainers | Приоритеты A/B/C. A (обязательно): не мутировать state, редьюсеры без сайд-эффектов, только сериализуемое в state, один стор. B: RTK как стандарт, feature-folders, логика в редьюсерах, экшены как события, а не сеттеры. C: `domain/eventName`, `selectThing`, мемоизированные селекторы | редко, документ зрелый |
| [redux.js.org/usage](https://redux.js.org/usage/) | Usage Guide | Redux maintainers | Structuring Reducers, Deriving Data with Selectors, Writing Logic with Thunks, Usage With TypeScript, Writing Tests | по мере надобности |
| [redux.js.org/usage/migrating-to-modern-redux](https://redux.js.org/usage/migrating-to-modern-redux) | Migration | Redux maintainers | Таблица «легаси → современное»: `createStore`→`configureStore`, константы+экшены+редьюсер→`createSlice`, ручные thunk-и→RTK Query, `connect()`→`useSelector`/`useDispatch`, типы выводить из стора (`RootState = ReturnType<typeof store.getState>`) | редко |
| [redux-toolkit.js.org](https://redux-toolkit.js.org/) | Документация RTK | Redux maintainers | API `createSlice`/`createAsyncThunk`/`configureStore`, RTK Query, TypeScript-туториал | непрерывно |
| [github.com/reduxjs/redux-toolkit/releases](https://github.com/reduxjs/redux-toolkit/releases) | Release notes | Redux maintainers | Минорные релизы 2.x | ~раз в 1–2 месяца |

### Vite

| Источник | Тип | Чей | Что берём | Как часто обновляется |
|---|---|---|---|---|
| [vite.dev/blog](https://vite.dev/blog) | Блог релизов | VoidZero / Vite Team | Анонсы мажоров: Vite 8 (12.03.2026), Vite 8.1 (23.06.2026) | 2–4 поста в год |
| [vite.dev/blog/announcing-vite8](https://vite.dev/blog/announcing-vite8) | Анонс мажора | Vite Team | Rolldown вместо esbuild+Rollup, Vite Devtools, встроенная поддержка `tsconfig` paths, требования Node 20.19+/22.12+ | разово |
| [vite.dev/guide/migration](https://vite.dev/guide/migration) | Migration guide | Vite Team | Переименования конфига: `esbuild`→`oxc`, `optimizeDeps.esbuildOptions`→`optimizeDeps.rolldownOptions`, `build.rollupOptions`→`build.rolldownOptions`; поднятые browser targets; изменённый CJS-interop | при каждом мажоре |
| [vite.dev/guide/features](https://vite.dev/guide/features) | Документация | Vite Team | Что Vite умеет из коробки (TS, HMR, CSS). **Важно:** раздела про тестирование там нет — Vitest в основной документации Vite не рекомендуется явно | непрерывно |

### Тесты

| Источник | Тип | Чей | Что берём | Как часто обновляется |
|---|---|---|---|---|
| [testing-library.com/docs/guiding-principles](https://testing-library.com/docs/guiding-principles/) | Принципы | Testing Library | «The more your tests resemble the way your software is used, the more confidence they can give you»; тестировать DOM-узлы, а не инстансы компонентов. Это ровно та норма, из которой растёт правило Firebird «assert on roles/test ids/text, не на приватных внутренностях» | почти не меняется |
| [github.com/testing-library/eslint-plugin-testing-library](https://github.com/testing-library/eslint-plugin-testing-library) | Линтер | Testing Library | Официальный ESLint-плагин, конфиг `testing-library/react`. Ловит типовые ошибки RTL машинно | несколько релизов в год |
| [jestjs.io/blog](https://jestjs.io/blog) | Блог релизов | Jest / OpenJS Foundation | Анонсы мажоров. **Последний пост — Jest 30, 04.06.2025** | редко |
| [jestjs.io/docs/upgrading-to-jest30](https://jestjs.io/docs/upgrading-to-jest30) | Upgrade guide | Jest | Требования Jest 30: Node ≥18, **TypeScript ≥5.4**, jsdom 26; убраны алиасы матчеров (`toBeCalled`→`toHaveBeenCalled`), `--testPathPattern`→`--testPathPatterns`, deep-импорты во внутренности пакетов больше не работают | разово |
| [vitest.dev/guide/comparisons](https://vitest.dev/guide/comparisons) | Сравнение | Vitest / VoidZero | Официальная позиция: «in a world where we have Vite providing support for the most common web tooling … Jest represents a duplication of complexity»; конфиг dev/build/test одним `vite.config.js` | редко |
| [vitest.dev/guide/migration](https://vitest.dev/guide/migration) | Migration from Jest | Vitest | Что реально ломается при переезде: `globals` не включены по умолчанию, другой `mockReset`, фабрика в `vi.mock` обязана вернуть все экспорты явно, `requireActual`→`vi.importActual`, `JEST_WORKER_ID`→`VITEST_POOL_ID` | при мажорах |

### TypeScript и линтеры

| Источник | Тип | Чей | Что берём | Как часто обновляется |
|---|---|---|---|---|
| [typescriptlang.org/docs/handbook](https://www.typescriptlang.org/docs/handbook/intro.html) | Handbook | Microsoft | Канон по языку. Сами авторы оговаривают: это «not a complete language specification», а практический гид | редко |
| [devblogs.microsoft.com/typescript](https://devblogs.microsoft.com/typescript/) | Блог релизов | Microsoft | Анонсы мажоров и миноров | ~раз в квартал |
| [devblogs.microsoft.com/typescript/announcing-typescript-7-0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/) | Анонс мажора | Microsoft | TS 7.0 (08.07.2026) — нативный порт на Go, 8–12× быстрее; наследует строгие дефолты TS 6.0: `strict` по умолчанию, `module` = `esnext`, `types` по умолчанию `[]` вместо автообнаружения всех `@types`, `rootDir` = `./` | разово |
| [typescript-eslint.io/users/configs](https://typescript-eslint.io/users/configs/) | Наборы правил | typescript-eslint | `recommended`, `recommended-type-checked`, `strict`, `strict-type-checked`, `stylistic`, `stylistic-type-checked`. **Практика, выраженная машиной** — прямой ориентир для линт-гейта Firebird | при релизах |
| [typescript-eslint.io/users/dependency-versions](https://typescript-eslint.io/users/dependency-versions/) | Матрица совместимости | typescript-eslint | Дословно: «The version range of TypeScript currently supported is `>=4.8.4 <6.1.0`» — критично, см. раздел про риски | при релизах |
| [eslint.org/blog](https://eslint.org/blog) | Блог релизов | ESLint / OpenJS | Релизы; там же объявлено, что **ESLint 9.x EOL — 2026-08-06** | ~2 поста в месяц |
| [eslint.org/docs/latest/use/migrate-to-10.0.0](https://eslint.org/docs/latest/use/migrate-to-10.0.0) | Migration guide | ESLint | ESLint 10: формат `.eslintrc` **удалён полностью**, только flat config; Node ≥20.19/22.13/24; в `eslint:recommended` добавлены `no-unassigned-vars`, `no-useless-assignment`, `preserve-caught-error`; `eslint-env`-комментарии теперь ошибка | при мажорах |

### Монорепо, UI, FP-слой

| Источник | Тип | Чей | Что берём | Как часто обновляется |
|---|---|---|---|---|
| [lerna.js.org](https://lerna.js.org/) | Документация | Nx (Nrwl) | Актуальная документация; никаких пометок о deprecation/maintenance mode нет, в подвале «© 2026 Copyright Nx» | по релизам |
| [github.com/lerna/lerna/releases](https://github.com/lerna/lerna/releases) | Release notes | Nx | Ключевое: **Lerna 10.0.0 вышла 29.07.2026** | 1–2 мажора в год, миноры чаще |
| [effect.website/docs](https://effect.website/docs/getting-started/introduction/) | Документация | Effectful Technologies | Канон по `Either`/`Option`/`Match` — ровно то, на чём стоит `rtk-effect-state` | непрерывно |
| [effect.website/blog](https://effect.website/blog) | Блог | Effectful Technologies | Состояние Effect v4 beta | ~ежемесячно |
| [mui.com/material-ui](https://mui.com/material-ui/getting-started/) | Документация | MUI | Компоненты, темизация, migration-раздел | непрерывно |
| [storybook.js.org/docs/releases/migration-guide](https://storybook.js.org/docs/releases/migration-guide) | Migration guide | Storybook | Storybook 10: ESM-only, `.storybook/main.ts` обязан быть валидным ESM, Node 20.19+/22.12+, апгрейд через `npx storybook@latest upgrade` | при мажорах |
| [nodejs.org/en/about/previous-releases](https://nodejs.org/en/about/previous-releases) | Матрица версий | Node.js / OpenJS | Что сейчас LTS. Node 24 — активный LTS, Node 26 — Current, Node 20 — EOL | непрерывно |

**Итого 30 проверенных источников.** Все открыты вручную 2026-08-03; ни одной ссылки «по памяти».

## Что изменилось за последний год

Ниже — только то, что подтверждено официальными страницами, и только то, что задевает
конкретно этот стек.

**1. TypeScript совершил два мажора подряд, и второй — переписан на Go.**
TS 6.0 — 23.03.2026, TS 7.0 — 08.07.2026, в npm сейчас `typescript@7.0.2`. TS 7 это нативный порт
(8–12× быстрее), логика проверки типов структурно идентична TS 6.0, но дефолты изменились жёстко:
`strict` включён по умолчанию, `module` = `esnext`, `types` по умолчанию **пустой массив** вместо
автообнаружения всех `@types` (то есть `"types": ["node", "jest"]` теперь надо писать руками),
`rootDir` = `./`. Для Firebird это самое большое изменение года — и одновременно самое рискованное,
см. следующий раздел.

**2. React Compiler дошёл до 1.0 (07.10.2025) и изменил норму про мемоизацию.**
Официальная формулировка: `useMemo`/`useCallback` больше не нужны по умолчанию — компилятор
мемоизирует сам, в том числе условно (после early return, чего ручная мемоизация не умеет).
Ручные `useMemo`/`useCallback` остаются как escape hatch для точечного контроля. Отдельно: React
рекомендует «Upgrade to `eslint-plugin-react-hooks@latest` to enforce Rules of React with
compiler-powered rules» — плагин теперь тянет диагностику компилятора **даже если сам компилятор
не подключён**. Это ровно тот случай, о котором задача: «код в стиле 2025», где ручная мемоизация
считается хорошим тоном, а авторы уже рекомендуют иначе.

**3. React 19.2 (01.10.2025): `<Activity />`, `useEffectEvent`, Performance Tracks.**
`useEffectEvent` — прямая замена типовому костылю «функция в эффекте, которую не хочется класть в
массив зависимостей»; Effect Event всегда видит свежие props/state и **не должен** попадать в
зависимости. Там же: `eslint-plugin-react-hooks` v6 перешёл на flat config по умолчанию и получил
правила от компилятора; `useId` сменил префикс `:r:` → `_r_`. React сейчас — 19.2.7 (01.06.2026),
активных мажоров после 19 не выходило.

**4. Vite 8 (12.03.2026) сменил бандлер: Rolldown и Oxc вместо esbuild и Rollup.**
Заявлено 10–30× ускорение сборки при сохранении совместимости плагинов. Конфиг переименован
(`build.rollupOptions` → `build.rolldownOptions`, `esbuild` → `oxc`), есть временный слой
совместимости, который в будущих версиях уберут. В npm сейчас `vite@8.2.0`, Vite 8.1 — 23.06.2026.

**5. Lerna 10.0.0 — 29.07.2026, за пять дней до этой сверки.**
ESM-only, минимальный Node поднят до **22.13.0**, добавлен bun как поддерживаемый пакетный менеджер,
изменено поведение в CI (теперь ошибка, если локальный чекаут отстал от remote при version/publish —
раньше проверка работала только вне CI), переписана генерация CHANGELOG на актуальные
conventional-changelog API.

**6. ESLint 10 полностью убрал `.eslintrc`.** Только flat config, `eslint-env`-комментарии стали
ошибкой. И жёсткая дата: **ESLint 9.x EOL — 06.08.2026**, то есть через три дня после этой сверки.

**7. Effect v4 в бете с февраля 2026.** Переписанный runtime, меньше бандл, унифицированная система
пакетов; последние beta-апдейты — 31.07.2026. Стабильная линия по-прежнему 3.x (`effect@3.22.1`).

**8. Storybook 10 (октябрь 2025) — ESM-only,** требует Node 20.19+/22.12+, добавлена поддержка
Vitest 4; CSF Factories переведены из Experimental в Preview и станут дефолтом в Storybook 11.

**9. React обрёл институциональный дом:** React Foundation под эгидой Linux Foundation
(объявлено 07.10.2025, запущено 24.02.2026). На технические практики не влияет, но меняет ответ на
вопрос «чей это стандарт» — теперь это фонд, а не одна компания.

## Где стек рискует отстать

### Lerna — жива, но Firebird на мажор позади

**Слух о смерти Lerna неверен.** С 2022 года её сопровождает Nx (Nrwl), сайт `lerna.js.org` не несёт
ни одной пометки о deprecation, релизы идут регулярно, 10.0.0 вышла 29.07.2026. Под капотом Lerna
работает на движке задач Nx. Позиционирование авторов: Lerna — про версионирование и публикацию
npm-пакетов, Nx — про большие монорепозитории целиком. Firebird пакеты наружу не публикует
(`@firebird/*` резолвятся друг в друга внутри workspace), так что ключевая ценность Lerna здесь —
`lerna run` по графу зависимостей, а это ровно то, что Nx делает нативно.

Практический вывод для Firebird — **не «срочно уходить с Lerna», а «Lerna 9 → 10»**, и это не
косметика: ESM-only и Node ≥22.13.0. CI на node:24 требование по Node закрывает, но ESM-only может
задеть конфиги, а изменённое CI-поведение при отставании чекаута от remote — способ внезапно уронить
пайплайн на ровном месте. Мигрировать стоит осознанно, не автообновлением.

Отдельно честно: **сравнений «Lerna vs Nx vs Turborepo» из официальных источников не существует** —
это по определению маркетинговый жанр, и все найденные подборки были на dev.to/Medium, то есть вне
границ задачи. Единственное, что можно утверждать по официальным страницам: Lerna поддерживается,
принадлежит Nx, релизится.

### Jest vs Vitest — Vite себя тестовым раннером не назначает

Здесь надо быть точным, потому что соблазн ответить «Vite рекомендует Vitest» велик.

**Что говорят официальные страницы:**
- В документации Vite (`vite.dev/guide/features`) **раздела про тестирование нет вообще**. В анонсе
  Vite 8 Vitest **не упоминается**.
- Vitest сам о себе (`vitest.dev/guide/comparisons`) говорит так: «in a world where we have Vite
  providing support for the most common web tooling (TypeScript, JSX, most popular UI Frameworks),
  Jest represents a duplication of complexity», и главный аргумент — один `vite.config.js` на dev,
  build и test. Формулировки «официально рекомендованный раннер для Vite» на странице **нет**.
- Jest жив: `jest@30.4.2`, но последний пост в официальном блоге — Jest 30 от 04.06.2025, то есть
  больше года назад. Мажора 31 нет.
- Косвенный, но говорящий сигнал: Vitest и Vite сейчас развиваются под одним зонтиком (VoidZero), а
  Storybook 10 отдельно отметил поддержку **Vitest 4**.

**Вывод для Firebird.** Стек на Jest 30 — это не «устаревший стек», это **актуальная версия Jest**.
Реальная асимметрия в другом: проект собирается Vite 8 (Rolldown/Oxc), а тесты гоняются
Jest + ts-jest, то есть **вторым, независимым тулчейном трансформации TypeScript**. Отсюда и
знаменитая боль профиля — скрипты `test:win`, где приходится глушить `verbatimModuleSyntax`
(`TS_NODE_COMPILER_OPTIONS={"verbatimModuleSyntax":false}`), чтобы ts-jest не спотыкался об
`import type`. Это ровно тот класс проблем, который Vitest снимает по построению, потому что
трансформирует тем же конвейером, что и сборка. Формально Firebird не отстаёт; фактически он платит
за дублирование тулчейнов, и симптом уже задокументирован в собственном скилле.

Рекомендация по факту, а не по вкусу: **это кандидат на осознанное решение, а не на автоматическое
«обновить»**. Обязательное к учёту при оценке — `vitest.dev/guide/migration`: `globals` не включены
по умолчанию, `mockReset` ведёт себя иначе, фабрика `vi.mock` обязана возвращать все экспорты явно.
Смягчающее обстоятельство: политика тестов Firebird («только чистые функции, сессию не мокаем, e2e
нет») означает, что мок-специфичных мест в тестах немного — а это самая дорогая часть переезда.

### Третья зона, не заказанная, но найденная: typescript-eslint не поддерживает TypeScript 7

Это важнее первых двух и всплыло при проверке линтеров. Дословно с
[typescript-eslint.io/users/dependency-versions](https://typescript-eslint.io/users/dependency-versions/):

> «The version range of TypeScript currently supported is `>=4.8.4 <6.1.0`.»

То есть **TS 7 (и даже TS 6.1) за границей поддержки typescript-eslint**, а Firebird держит
линт-гейт на `--max-warnings 0` с правилами вроде `@typescript-eslint/explicit-function-return-type`.
Апгрейд на TS 7 сегодня ломает линт-гейт, а не улучшает его. Микрософт в анонсе TS 7 честно пишет,
что Vue, Svelte и Angular тоже пока не могут перейти — ждут стабильного API в 7.1, и предлагает
пакет совместимости `@typescript/typescript6`.

**Вывод:** «обновить TypeScript до последней» для Firebird сейчас — вредный совет. Правильная
формулировка нормы для скилла: целиться в **TS 6.0** (там уже строгие дефолты, и это внутри
поддерживаемого typescript-eslint диапазона), а TS 7 держать в наблюдении до момента, когда
typescript-eslint объявит поддержку.

### Ещё две даты, которые стоит держать в календаре

- **ESLint 9.x EOL — 06.08.2026** (через три дня после сверки). Версия ESLint в Firebird в профиле не
  указана; если там 9.x, окно закрывается прямо сейчас. Переход на 10 означает обязательный flat
  config — `.eslintrc` больше не читается.
- **Node 20 — EOL.** CI на node:24 (активный LTS), так что здесь всё в порядке; но Lerna 10 поднимает
  планку до 22.13.0, и это надо сверить с локальными машинами разработчиков, а не только с CI.

## Что переписать в профиле прямо сейчас

- Не обновлять TypeScript до 7 — typescript-eslint поддерживает `>=4.8.4 <6.1.0`, целиться в **TS 6.0**; TS 7 держать в наблюдении до объявления поддержки. Сегодня апгрейд на 7 ломает линт-гейт `--max-warnings 0`. [dependency-versions](https://typescript-eslint.io/users/dependency-versions/)
- Убрать из скиллов ручную мемоизацию как норму: `useMemo`/`useCallback` — escape hatch, React Compiler v1.0 мемоизирует сам, в том числе после early return. Типовой костыль «функция в эффекте вне массива зависимостей» заменить на `useEffectEvent` (React 19.2), который в зависимости попадать **не должен**. [react.dev/blog](https://react.dev/blog)
- Проверить и зафиксировать в линт-гейте `eslint-plugin-react-hooks` (пресет `recommended-latest`, flat config с v6): сейчас его наличие из профиля не следует, а он основной носитель правил React и диагностики компилятора. Проверяется одной строкой в `eslint.config.js`. [eslint-plugin-react-hooks](https://react.dev/reference/eslint-plugin-react-hooks)
- Перевести конфиг ESLint на flat config, если он ещё `.eslintrc`: в ESLint 10 формат удалён полностью, а **ESLint 9.x EOL — 06.08.2026**, то есть через три дня после сверки. [migrate-to-10.0.0](https://eslint.org/docs/latest/use/migrate-to-10.0.0)
- В скилле `vite-lerna-build-verification` заменить имена опций Vite: `build.rollupOptions` → `build.rolldownOptions`, `esbuild` → `oxc`, `optimizeDeps.esbuildOptions` → `optimizeDeps.rolldownOptions` — Vite 8 сменил бандлер, слой совместимости временный. [vite.dev/guide/migration](https://vite.dev/guide/migration)
- Запланировать **Lerna 9 → 10** как осознанную миграцию, а не автообновление: ESM-only, Node ≥22.13.0 (CI на node:24 закрывает, локальные машины — нет), и новое CI-поведение — ошибка, если чекаут отстал от remote при version/publish. [lerna/releases](https://github.com/lerna/lerna/releases)

## Чего не нашёл / где источник слабый

- **Роутер стека.** Не назван ни в одном файле профиля. Пока не установлен — раздел «роутинг» в
  каталоге источников открывать нечем. Требует ответа от команды Firebird, а не поиска.
- **Версии TS / ESLint / Storybook / MUI-минор в самом репозитории Firebird.** Профиль называет
  только мажоры части библиотек и `CI node:24`. Без `package.json` репозитория `firebird-html5-client`
  (он вне доступного клона) точный размер отставания не измерить — весь раздел про риски написан от
  того, что зафиксировано в профиле.
- **JUMP.** Внешних источников нет и не будет — протокол внутренний. Единственный канон: корневой
  `CLAUDE.md` репозитория Firebird и `docs/JUMP/`. Для скилла это значит: сверять по расписанию
  нечего, зато обязателен пункт «после изменения контракта обновить схему + `docs/JUMP/` вместе».
- **Официального сравнения монорепо-инструментов не существует.** Всё, что нашлось по запросу
  «Lerna vs Nx vs Turborepo 2026», — dev.to и Medium, то есть за границей задачи. В каталог не
  включено намеренно.
- **Позиция Vite по тестовому раннеру — источник слабый по построению.** Прямой официальной
  рекомендации «используйте Vitest» на `vite.dev` нет; есть только позиция самого Vitest о себе.
  Утверждать «Vite рекомендует Vitest» было бы натяжкой, поэтому в разделе выше это разведено.
- **`eslint-plugin-react-hooks` в Firebird** — не подтверждён. Страница правил в каталоге есть, но
  подключён ли плагин в репозитории, из профиля не следует. Учитывая, что React 19.2 и React
  Compiler v1.0 сделали этот плагин основным носителем «правил React», его отсутствие было бы
  заметной дырой в гейте. Проверяется одной строкой в `eslint.config.js` Firebird.
- **`react.dev/reference/eslint-plugin-react-hooks` помечен как `version: rc`** — страница живая и
  официальная, но сама себя маркирует как release candidate. При автоматической сверке это надо
  учитывать: содержимое может измениться до финального релиза.

## Владелец ревью

| Роль | Кто | Дата сверки |
|---|---|---|
| Владелец стека | _не назначен — впишите себя_ | 2026-08-03 |

**Вопрос команде Firebird: какой роутер в стеке?** По репозиторию установить не удалось — ни `react-router`, ни TanStack Router, ни собственное решение в шаблонах `firebird-*` не упоминаются. Пока нет ответа, раздел «роутинг» в каталоге источников открывать нечем: это требует ответа от вас, а не поиска.

Правьте эту страницу под свой стек: вычёркивайте источники, которыми не пользуетесь, дописывайте свои, поправьте описание стека, если оно разошлось с реальностью.
