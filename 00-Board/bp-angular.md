---
title: Best practices — Angular-веб (источники)
type: note
status: current
created: 2026-08-03
updated: 2026-08-03
tags: [board, bestpractices, angular]
permalink: tacticum/00-board/bp-angular
---

# Angular-веб — официальные источники best practices

> **Коротко:** в профиле — Angular 21; актуальная — v22 (03.06.2026); v21 с этого дня в LTS до 06.2027, поддерживается — обновление не аварийное, но описание стека отстаёт на мажор.

## Стек по профилю

Дословно из `TABLE_BLURBS` (`scripts/wiki_sync_manuals.py`, репозиторий TacticumApps/dev):

> Angular 21 + Nx monorepo, NgRx signals, TypeScript strict; yarn/bun; Jest / Vitest (Analog); transloco / ngx-translate; ng-openapi-gen; контракт @ivcs/*; зона конференций WebRTC/SFU

Профили, использующие этот стек: `iva-role-web`, `iva-web-brownfield`.

**Сразу к делу: описание стека отстаёт на мажор.** Angular v22 вышел 2026-06-03, v21 (2025-11-19)
с этого дня в LTS. Текущая документация на angular.dev помечена `v22.1.0`. Всё, что ниже, сверялось
против v22.

---

## Источники

### Angular — ядро

| Источник | Тип | Чей | Что именно берём | Обновляется | Сверка |
|---|---|---|---|---|---|
| [angular.dev/style-guide](https://angular.dev/style-guide) | доки / style guide | Angular Team (Google) | канон стиля: имена файлов через дефис, структура проекта, `inject()` вместо конструктора, `protected` для членов под шаблон, `readonly` для инициализируемых Angular, `class`/`style` вместо `NgClass`/`NgStyle`, имя обработчика по действию | с каждым мажором | 2026-08-03 |
| [angular.dev/assets/context/best-practices.md](https://angular.dev/assets/context/best-practices.md) | машиночитаемый свод правил | Angular Team | **главный файл для агента.** Готовый список правил в формате инструкций LLM: `input()`/`output()`/`model()` вместо декораторов, `computed()`, `linkedSignal()`, standalone всегда, `providedIn: 'root'`, нативный control flow. Запреты: `any`, `@HostBinding`/`@HostListener`, `ngClass`/`ngStyle`, `mutate` на сигнале, `*ngIf`/`*ngFor`/`*ngSwitch`, внешние шаблоны/стили для мелких компонентов | вместе с релизами | 2026-08-03 |
| [angular.dev/llms.txt](https://angular.dev/llms.txt) + [llms-full.txt](https://angular.dev/assets/context/llms-full.txt) | индекс для LLM | Angular Team | карта разделов доков в машинном виде — чем разворачивать полный контекст | вместе с релизами | 2026-08-03 |
| [angular.dev/ai/develop-with-ai](https://angular.dev/ai/develop-with-ai) | доки | Angular Team | точка входа во все три файла выше + готовые `GEMINI.md`, `copilot-instructions.md`, `cursor.md` | вместе с релизами | 2026-08-03 |
| [github.com/angular/skills](https://github.com/angular/skills) + [angular.dev/ai/agent-skills](https://angular.dev/ai/agent-skills) | скиллы для агентов | Angular Team | официальные скиллы `angular-developer` и `angular-new-app`. Ставятся `npx skills add https://github.com/angular/skills`. Внутри — прямые требования: анализировать версию Angular перед советом, гонять `ng build` после генерации, `httpResource` для данных, signal forms для v21+, не использовать legacy-DSL анимаций | по коммитам в репо | 2026-08-03 |
| [angular.dev/ai/mcp](https://angular.dev/ai/mcp) | инструмент | Angular Team | MCP-сервер Angular CLI, 9 инструментов. Прямо релевантны: `get_best_practices`, `search_documentation`, `onpush_zoneless_migration`, `run_target`, `devserver.*` | с релизами CLI | 2026-08-03 |
| [angular.dev/reference/releases](https://angular.dev/reference/releases) | политика | Angular Team | каденция релизов, окна Active/LTS, политика депрекации (API живёт минимум один мажор) | по изменению политики | 2026-08-03 |
| [angular.dev/reference/versions](https://angular.dev/reference/versions) | матрица совместимости | Angular Team | Node / TypeScript / RxJS по версиям Angular — чем проверять, можно ли вообще обновиться | с каждым мажором | 2026-08-03 |
| [angular.dev/reference/migrations](https://angular.dev/reference/migrations) | автомиграции | Angular Team | 13 схематик `ng generate @angular/core:…` — standalone, control-flow, inject, signal inputs/outputs/queries, self-closing tags, NgClass→class, NgStyle→style, CommonModule→standalone, cleanup-unused-imports | с каждым мажором | 2026-08-03 |
| [angular.dev/update-guide](https://angular.dev/update-guide) | инструмент | Angular Team | пошаговый план апгрейда под конкретную пару версий; ссылка на 21→22: `?v=21.0-22.0` | с каждым мажором | 2026-08-03 |
| [angular.dev/roadmap](https://angular.dev/roadmap) | роадмап | Angular Team | куда едет стек: tsgo-компилятор, Angular Aria из preview в stable, миграция Karma→Vitest, AI-генерация UI | несколько раз в год | 2026-08-03 |
| [angular.dev/events/v22](https://angular.dev/events/v22) | релизная страница | Angular Team | сводка v22 + ссылки на changelog и блог | разово на мажор | 2026-08-03 |
| [blog.angular.dev](https://blog.angular.dev/announcing-angular-v22-c52bb83a4664) | блог релизов | Angular Team | анонсы мажоров. **Физически размещён на Medium** — прямой заход даёт редирект через `medium.com` | на каждый релиз | 2026-08-03 |
| [github.com/angular/angular/discussions/categories/rfcs](https://github.com/angular/angular/discussions/categories/rfcs) | RFC-процесс | Angular Team | видно, куда едет фреймворк до релиза. Свежее: «RFC: Setting OnPush as the default Change Detection Strategy» (2026-01-27, Complete), «RFC: AI-Enabled Applications» (2025-09-12, Closed) | по мере появления | 2026-08-03 |

### Angular — предметные разделы, по которым агент чаще всего пишет «по-старому»

| Источник | Тип | Что берём | Сверка |
|---|---|---|---|
| [angular.dev/guide/zoneless](https://angular.dev/guide/zoneless) | доки | «Zoneless is the default in Angular v21+»; как выпиливать zone.js из polyfills, `angular.json`, package.json | 2026-08-03 |
| [angular.dev/guide/signals](https://angular.dev/guide/signals) | доки | `signal`, `computed`, `effect`, `linkedSignal`, `resource` | 2026-08-03 |
| [angular.dev/guide/forms/signals/overview](https://angular.dev/guide/forms/signals/overview) | доки | Signal Forms: `form`, `FormField`, `required`, `email` из `@angular/forms/signals`. **Важная оговорка самих доков:** для существующих приложений на reactive forms и там, где нужны production-гарантии, reactive forms остаются нормальным выбором | 2026-08-03 |
| [angular.dev/guide/aria/overview](https://angular.dev/guide/aria/overview) | доки | `@angular/aria` — headless-директивы WAI-ARIA: Autocomplete, Listbox, Select, Multiselect, Combobox, Menu, Menubar, Toolbar, Accordion, Tabs, Tree, Grid | 2026-08-03 |
| [angular.dev/guide/testing](https://angular.dev/guide/testing) | доки | новые проекты — vitest + jsdom из коробки; Karma «ещё поддерживается», но вынесен в отдельный раздел | 2026-08-03 |
| [angular.dev/guide/testing/migrating-to-vitest](https://angular.dev/guide/testing/migrating-to-vitest) | миграция | билдер `@angular/build:unit-test` вместо `@angular/build:karma`; **сама миграция помечена experimental**; `karma.conf.js` вручную, build-опции переносятся в отдельный build-таргет | 2026-08-03 |
| [angular.dev/tools/cli/build-system-migration](https://angular.dev/tools/cli/build-system-migration) | миграция | application builder на esbuild/Vite; webpack-система и `browser`-билдер объявлены устаревшими | 2026-08-03 |
| [angular.dev/reference/configs/file-structure](https://angular.dev/reference/configs/file-structure) | доки | канон структуры workspace и `projects/` для мультипроектных репозиториев | 2026-08-03 |

### Nx

| Источник | Тип | Что берём | Сверка |
|---|---|---|---|
| [nx.dev/changelog](https://nx.dev/changelog) | релизы | текущая версия Nx **23.1** (2026-07-13); история релизов 2026 | 2026-08-03 |
| [nx.dev/blog/nx-23-1-release](https://nx.dev/blog/nx-23-1-release) | блог релиза | поддержка Angular 22 (Signal Forms stable, OnPush по умолчанию, декоратор `@Service`, `injectAsync`), TypeScript 6, ESLint v9 + typescript-eslint v8, per-run performance reports | 2026-08-03 |
| [nx.dev/blog](https://nx.dev/blog) | блог | канал релизов Nx и Nx Cloud | 2026-08-03 |
| [nx.dev/blog/nx-2026-roadmap](https://nx.dev/blog/nx-2026-roadmap) | роадмап | 2026-02-04. «2026 is the year Nx becomes infrastructure for autonomous AI agents»: self-healing CI, агентные `nx migrate`, task sandboxing | 2026-08-03 |
| [nx.dev/technologies/angular](https://nx.dev/technologies/angular) | доки | точка входа в Angular-раздел Nx | 2026-08-03 |
| [nx.dev/docs](https://nx.dev/docs) | доки | база по Nx core | 2026-08-03 |

### Линтер — практика, выраженная машиной

| Источник | Тип | Что берём | Сверка |
|---|---|---|---|
| [github.com/angular-eslint/angular-eslint](https://github.com/angular-eslint/angular-eslint) | линтер | `@angular-eslint/eslint-plugin` (TS) и `@angular-eslint/eslint-plugin-template` (HTML). Версии синхронизированы с `@angular/cli` (22.x ↔ 22.x). **Требует ESLint v9+, поддерживает только flat config** (`eslint.config.js`), `.eslintrc` не поддерживается вовсе. Документация — файлами в репо: `docs/CONFIGURING_ESLINT.md`, `docs/RULES_REQUIRING_TYPE_INFORMATION.md`, `docs/ANGULAR_VERSION_SUPPORT.md` | 2026-08-03 |
| [angular.dev/reference/migrations](https://angular.dev/reference/migrations) | схематики | схематики — вторая машинная форма практики: они не советуют, а переписывают код | 2026-08-03 |

### Остальное по стеку

| Источник | Тип | Чей | Что берём | Сверка |
|---|---|---|---|---|
| [devblogs.microsoft.com/typescript](https://devblogs.microsoft.com/typescript/) | блог релизов | Microsoft | **TS 6.0 (2026-03-23) — последний релиз на JS-кодовой базе. TS 7.0 (2026-07-08) — нативный порт на Go, ~10x** | 2026-08-03 |
| [vitest.dev/guide](https://vitest.dev/guide/) | доки | Vitest | текущая 4.1.10; требует Vite ≥6, Node ≥20; browser mode, миграционные гайды с Jest и на 4.0 | 2026-08-03 |
| [analogjs.org/docs/features/testing/vitest](https://analogjs.org/docs/features/testing/vitest) | доки | Analog | схематика `@analogjs/vitest-angular:setup`. Актуально для проектов до v21, но **это уже не единственный путь** — у Angular CLI теперь свой vitest-билдер | 2026-08-03 |
| [github.com/jsverse/transloco](https://github.com/jsverse/transloco) | репозиторий | jsverse | пакет переехал `@ngneat/transloco` → `@jsverse/transloco` (v7+); старый scope обновлений не получает | 2026-08-03 |
| [jsverse.gitbook.io/transloco](https://jsverse.gitbook.io/transloco) | доки | jsverse | официальные доки, есть `llms.txt` | 2026-08-03 |
| [github.com/ngx-translate/core](https://github.com/ngx-translate/core) | репозиторий | ngx-translate | **заявлена поддержка Angular 16–21**; политика «поддерживаем только текущую версию»; EOL-версии — платно через HeroDevs | 2026-08-03 |
| [github.com/cyclosproject/ng-openapi-gen](https://github.com/cyclosproject/ng-openapi-gen) | репозиторий | cyclosproject | OpenAPI 3.0/3.1; заявлено «генерируемый код совместим с Angular 16+» | 2026-08-03 |
| [bun.com/blog](https://bun.com/blog) | блог релизов | Oven | Bun 1.3.14 (2026-05-13) | 2026-08-03 |
| [yarnpkg.com](https://yarnpkg.com/) | доки | Yarn | документация покрывает Yarn 4+; для 3.6 и ниже — архив `v3.yarnpkg.com` | 2026-08-03 |
| [w3.org/TR/webrtc](https://www.w3.org/TR/webrtc/) | спецификация | W3C | WebRTC — W3C Recommendation от 2025-03-13, с candidate amendments поверх | 2026-08-03 |
| [material.angular.dev](https://material.angular.dev/) | доки | Angular Team | Angular Material / CDK. Сайт живой; содержимого вытащить не удалось, версия и миграции не подтверждены | 2026-08-03 |

---

## Что изменилось за последний год

Порядок — по тому, насколько больно агенту, который пишет «в стиле 2025».

- **Вышел Angular v22 (2026-06-03), v21 ушёл в LTS.** Профиль стека всё ещё говорит «Angular 21».
  Поддержка: v22 Active до 2027-06, LTS до 2028-06; v21 — LTS до 2027-06; v20 — LTS до 2026-11-28.
  [releases](https://angular.dev/reference/releases), [events/v22](https://angular.dev/events/v22)

- **Каденция мажоров сменилась: с v22 — раз в 12 месяцев, а не раз в 6.** Дословно в доках: «Until
  Angular v22, Angular had a 6-month major release cycle». Окно поддержки прежние 24 месяца, но
  теперь это 12 Active + 12 LTS. Практический вывод: период сверки скиллов можно ставить не «раз в
  полгода под мажор», а «раз в квартал под минор» — крупное теперь приходит реже, но большими
  порциями. [releases](https://angular.dev/reference/releases)

- **OnPush стал дефолтной стратегией change detection для новых приложений в v22**, а
  `ChangeDetectionStrategy.Default` **переименован в `Eager`**. Решение прошло через RFC (2026-01-27,
  статус Complete). Агент, который по привычке пишет `ChangeDetectionStrategy.Default`, теперь пишет
  устаревшее имя. [блог v22](https://blog.angular.dev/announcing-angular-v22-c52bb83a4664),
  [RFC](https://github.com/angular/angular/discussions/categories/rfcs)

- **Zoneless — дефолт с v21** (stable ещё с 20.2). Доки прямо велят выпиливать zone.js: из
  `polyfills`, из `angular.json`, из зависимостей. Существующие проекты держат zone.js, пока сами не
  откажутся, — но новый код писать под него нельзя.
  [guide/zoneless](https://angular.dev/guide/zoneless)

- **Signal Forms стабильны в v22** (в v21 были экспериментальными), с документацией, поддержкой
  Material и Aria. Но у самих доков есть оговорка, которую агент обязан воспроизводить: для
  существующих приложений на reactive forms и там, где нужны production-гарантии, **reactive forms
  остаются нормальным выбором**. То есть «signal forms для нового, reactive для brownfield» — это не
  компромисс, а официальная позиция. Прямо ложится на пару профилей `iva-role-web` /
  `iva-web-brownfield`. [forms/signals/overview](https://angular.dev/guide/forms/signals/overview)

- **Vitest вытеснил Karma, а Jest в официальных доках Angular не упоминается вообще.** Новые проекты
  получают `vitest` + `jsdom` из коробки, билдер — `@angular/build:unit-test`. Karma переехал в
  отдельный раздел «ещё поддерживается». При этом миграция существующего проекта Karma→Vitest сама
  помечена **experimental**, и `karma.conf.js` не переносится автоматически.
  [guide/testing](https://angular.dev/guide/testing),
  [migrating-to-vitest](https://angular.dev/guide/testing/migrating-to-vitest)

- **Asynchronous Reactivity стабильна: `resource` и `httpResource`.** Официальный скилл
  `angular-developer` прямо велит использовать `httpResource` для получения данных — вместо
  привычного `HttpClient` + `subscribe`/`async pipe`.
  [блог v22](https://blog.angular.dev/announcing-angular-v22-c52bb83a4664),
  [skills](https://github.com/angular/skills/blob/main/angular-developer/SKILL.md)

- **Angular Aria стабильна в v22** — 12 headless WAI-ARIA паттернов в пакете `@angular/aria`, с
  test harnesses. Это третья first-party UI-библиотека рядом с Material и CDK; в 2025 её не
  существовало. [guide/aria/overview](https://angular.dev/guide/aria/overview)

- **Webpack-сборка объявлена устаревшей:** `@angular-devkit/build-angular`, `@ngtools/webpack`.
  Канон — application builder на esbuild/Vite.
  [build-system-migration](https://angular.dev/tools/cli/build-system-migration)

- **TypeScript пережил два мажора за четыре месяца. TS 6.0 (2026-03-23) — последний релиз на
  JS-кодовой базе, TS 7.0 (2026-07-08) — нативный порт на Go, ~10x быстрее.** Angular v22 требует
  строго `>=6.0.0 <6.1.0`; v21 — `>=5.9.0 <6.0.0`. То есть TS 7 с Angular v22 пока не сочетается, и
  это ловушка: «поставить самый свежий TypeScript» ломает сборку.
  [TS blog](https://devblogs.microsoft.com/typescript/),
  [reference/versions](https://angular.dev/reference/versions)

- **Требования к Node подросли.** Angular v22: `^22.22.3 || ^24.15.0 || ^26.0.0` — Node 20 выпал
  (в v21 он ещё был). [reference/versions](https://angular.dev/reference/versions)

- **Nx: текущая 23.1 (2026-07-13), поддержка Angular 22 и TypeScript 6.** Angular 21 приехал в Nx
  22.3, Angular 21.2 — в 22.6. Nx 23.0 (2026-06-16) добавил агентный `nx migrate` — AI-ассистированные
  апгрейды. Роадмап Nx на 2026 целиком про автономных агентов.
  [changelog](https://nx.dev/changelog), [nx-23-1](https://nx.dev/blog/nx-23-1-release),
  [roadmap](https://nx.dev/blog/nx-2026-roadmap)

- **angular-eslint принимает только flat config и только ESLint 9/10.** Формат `.eslintrc` не
  поддерживается. Nx 23.1 умеет мигрировать конфиги автоматически.
  [angular-eslint](https://github.com/angular-eslint/angular-eslint)

- **NgRx 21: `@ngrx/signals/events` стал стабильным**, `withEffects` переименован в
  `withEventHandlers`, `withFeature` и `events` переехали в ядро `@ngrx/signals` из
  экспериментального/toolkit-статуса. Это ровно тот случай, когда агент пишет имя API, которого уже
  нет.

- **Появился официальный контур «Angular для AI-агентов», и в v22 он стабилизировался.** Это самое
  прямое попадание в задачу: у Angular теперь есть **машиночитаемый свод правил, который агент может
  забирать сам** — `assets/context/best-practices.md`, `llms.txt`, `llms-full.txt`, репозиторий
  скиллов `angular/skills`, MCP-сервер Angular CLI с инструментами `get_best_practices` и
  `onpush_zoneless_migration` (devserver-инструменты доведены до stable в v22).
  [ai/develop-with-ai](https://angular.dev/ai/develop-with-ai),
  [ai/agent-skills](https://angular.dev/ai/agent-skills), [ai/mcp](https://angular.dev/ai/mcp)

---

## Что переписать в профиле прямо сейчас

- Убрать из скиллов `ChangeDetectionStrategy.Default` → писать `Eager`, а для новых компонентов не указывать стратегию вовсе: в v22 OnPush стал дефолтом, а `Default` переименован. [блог v22](https://blog.angular.dev/announcing-angular-v22-c52bb83a4664)
- Заменить «Angular 21» в описании стека → «Angular 22 (v21 — LTS до 06.2027)» и там же снять формулировки про zone.js: zoneless — дефолт с v21, новый код под zone.js писать нельзя. [guide/zoneless](https://angular.dev/guide/zoneless)
- Не советовать `HttpClient` + `subscribe`/`async pipe` как способ получения данных → `httpResource`/`resource`: этого прямо требует официальный скилл `angular-developer`. [skills](https://github.com/angular/skills/blob/main/angular-developer/SKILL.md)
- Развести формы по профилям: `iva-role-web` (новое) → signal forms из `@angular/forms/signals`, `iva-web-brownfield` → reactive forms остаются нормальным выбором. Это официальная оговорка доков, а не компромисс. [forms/signals/overview](https://angular.dev/guide/forms/signals/overview)
- Не обновлять TypeScript до 7 — Angular v22 требует строго `>=6.0.0 <6.1.0`, целиться в 6.0; заодно поднять требование Node: `^22.22.3 || ^24.15.0 || ^26.0.0`, Node 20 выпал. [reference/versions](https://angular.dev/reference/versions)
- Убрать из скиллов имена API, которых уже нет: NgRx `withEffects` → `withEventHandlers`, `@ngrx/signals/events` больше не экспериментальный; в конфигах линтера `.eslintrc` → flat config `eslint.config.js` (angular-eslint иного формата не читает). [angular-eslint](https://github.com/angular-eslint/angular-eslint)

## Чего не нашёл / где источник слабый

- **`@ivcs/*` — публичного источника нет вообще.** Это внутренний контракт; сверять агента не с чем,
  кроме самого репозитория. Единственный элемент стека, у которого «официальный источник» — это мы
  сами.

- **Совместимости i18n-библиотек с Angular 22 не подтвердил ни у одной.** У `ngx-translate/core` в
  README прямо написано «Angular 16 - 21» — про 22 нет ни слова. У Transloco версию и матрицу
  совместимости вытащить не удалось: GitBook отдаёт тонкую вводную страницу, npm возвращает 403,
  README на GitHub номера версии не показывает. Это реальная дыра, а не лень: **перед апгрейдом на
  Angular 22 совместимость i18n надо проверять руками**, официальной таблицы нет.

- **`ng-openapi-gen` — политика версий отсутствует.** В README только «совместим с Angular 16+»,
  без разбивки по мажорам и без заявленной поддержки 21/22. Даты релизов со страницы репозитория
  считать не удалось. Источник живой, но по вопросу «можно ли на v22» молчит.

- **NgRx: сайт документации плохо верифицируется, официального блога на своём домене нет.**
  `ngrx.io` — клиентский SPA, фетчер получает оболочку главной страницы вместо содержимого; URL
  разделов (`/guide/signals`, `/guide/signals/signal-store`, `/guide/signals/signal-store/events`,
  `/guide/migration/v20`) подтверждены только через поисковый индекс, не открытием. Страница
  `github.com/ngrx/platform/releases` пуста — «There aren't any releases here». Анонсы версий команда
  публикует на dev.to под официальным аккаунтом NgRx: это официальный канал, но не собственный сайт,
  и по критерию «сайт авторов» он проходит с натяжкой. **Наличия NgRx v22 под Angular v22 я не
  подтвердил.**

- **Angular Material — сайт живой, содержимого не достал.** `material.angular.dev` отвечает, но
  фетчер вернул только заголовок. Версию, состояние Material 3 и миграции не подтверждаю.

- **Karma→Vitest: официальный путь есть, но сам помечен experimental.** То есть на вопрос «как
  правильно мигрировать существующие тесты» официального *стабильного* ответа пока нет — есть
  экспериментальная схематика и список того, что придётся доделать руками.

- **Jest в стеке профиля не имеет официальной опоры.** Angular документирует Vitest и (как
  legacy) Karma; Jest не упоминается ни разу. Analog документирует Vitest, не Jest. Если Jest
  реально используется — это конфигурация, которую ни один официальный источник не описывает.

- **Bun как рантайм/пакетный менеджер для Angular официальной позиции не имеет.** В доках Angular
  bun встречается только в строчках установки пакетов наравне с npm/yarn/pnpm. Отдельного гайда
  «Angular + bun» нет.

- **WebRTC/SFU — официальных best practices не существует в природе.** Есть спецификация W3C
  (Recommendation от 2025-03-13) — это стандарт, а не руководство по практике. Практики зоны
  конференций определяются конкретным SFU (mediasoup / Janus / LiveKit / собственный), а какой SFU
  у ИВА, из описания стека не следует. Пока это не уточнено, сверять агента здесь не с чем.

- **Полный changelog Angular открыть не удалось** — `CHANGELOG.md` в `angular/angular` весит 969 KB
  (9774 строки) и не отдаётся фетчеру целиком. Файл существует, но как источник для регулярной
  автоматической сверки он неудобен; для этой цели лучше `reference/releases` + `update-guide`.

- **Отдельной веб-страницы со списком правил angular-eslint не нашёл** — путь
  `packages/eslint-plugin/src/configs/README.md` отдаёт 404, домена `angular-eslint.io` не
  существует. Актуальный список правил и конфигов живёт файлами внутри репозитория
  (`docs/CONFIGURING_ESLINT.md` и каталоги правил).

## Владелец ревью

| Роль | Кто | Дата сверки |
|---|---|---|
| Владелец стека | _не назначен — впишите себя_ | 2026-08-03 |

Правьте эту страницу под свой стек: вычёркивайте источники, которыми не пользуетесь, дописывайте свои, поправьте описание стека, если оно разошлось с реальностью.
