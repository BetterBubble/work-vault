---
title: Best practices — Python + Next.js (источники)
type: note
status: current
created: 2026-08-03
updated: 2026-08-03
tags: [board, bestpractices, python, nextjs]
permalink: tacticum/00-board/bp-python-next
---

# Python + Next.js — официальные источники best practices

> **Коротко:** в профиле версии не зафиксированы вообще (манифесты объявляют только метки `python` / `nextjs`), фактический стек по коду `tacticum-dev` — `next: ^15.1.0`, Python 3.12, Node 20; актуальное на 2026-08-03 — стабильная линия Next.js **16.2** (16.3 в статусе Preview), Python 3.14, Node 22/24 LTS; `^15.1.0` не поддерживается ни одной LTS-линией Next.js — Active LTS это 16.2, Maintenance LTS — 15.5, — а Node 20 EOL с 30.04.2026 и Python 3.12 уже только security-only.

## Стек по профилю

Профили `tacticum-role-platform` и `tacticum-platform-dev` объявляют `stack.required: [python, nextjs]`
(`templates/tacticum-role-platform/manifest.yaml`, `templates/tacticum-platform-dev/manifest.yaml` на
`origin/main`). Сами манифесты и quickstart'ы **никаких версий и инструментов не называют** — только
метки стека. Реальный стек установлен по коду репозитория `tacticum-dev`, для которого эти профили и
писались (`tacticum-platform-dev` прямо называет себя «dogfooding profile for developing the Tacticum
platform itself (FastAPI/SQLAlchemy backend + Next.js admin_web/Legends)»).

`tacticum-role-internal` — `stack: required: []`, пустой. Стек-специфики у роли нет: её лейн
`tacticum-internal-base` несёт скиллы `tacticum-standards` / `/check-standards` (соответствие
STD-001..012), workspace-KB и дизайн-токены. Про язык и фреймворки в нём ничего нет, поэтому в этой
заметке роль присутствует только как потребитель того же монорепо.

### Python — `apps/backend/pyproject.toml`

| Что | Значение | Откуда |
|---|---|---|
| Версия языка | `requires-python = ">=3.12"`, `ruff.target-version = "py312"`, `mypy.python_version = "3.12"` | `pyproject.toml` |
| Рантайм | `FROM python:3.12-slim` | `apps/backend/Dockerfile` |
| Веб-фреймворк | **FastAPI** `>=0.115.0` + `uvicorn[standard]>=0.30.0`. Django нет | `pyproject.toml` |
| MCP | `fastmcp==3.4.3` (пин жёсткий, с комментарием «unstable across minors»), `mcp>=1.2.0` | `pyproject.toml` |
| ORM / БД | **SQLAlchemy 2.0** `[asyncio] >=2.0.30`, `asyncpg`, `psycopg[binary]>=3.2`, `alembic>=1.13` | `pyproject.toml` |
| Валидация / конфиг | **Pydantic v2** `>=2.7.0`, `pydantic-settings>=2.4` | `pyproject.toml` |
| Линтер | **ruff** `>=0.5`, `line-length = 100`, `select = ["E","F","I","N","B","UP","ASYNC","S"]`, `ignore = ["S101"]` | `pyproject.toml` |
| Типы | **mypy** `>=1.10`, `strict = true`, `plugins = ["pydantic.mypy"]` | `pyproject.toml` |
| Тесты | **pytest** `>=8.0` + `pytest-asyncio>=0.23`, `asyncio_mode = "auto"`, `addopts = "-q"` | `pyproject.toml` |
| Сборка пакета | `hatchling` (`build-backend = "hatchling.build"`) | `pyproject.toml` |
| Менеджер пакетов | **uv** — в CI: `astral-sh/setup-uv@v5`, `uv.lock`, `uv run --python 3.12 --extra dev`. Poetry нет | `.github/workflows/install-e2e.yml` |
| Прочее | `structlog`, `qdrant-client`, `minio`, `pyjwt[crypto]`, `passlib[bcrypt]`, `httpx`, `jsonschema` | `pyproject.toml` |

**Расхождение внутри репозитория:** CI ставит зависимости через uv по `uv.lock`, а
`apps/backend/Dockerfile` — через `pip install -e .` без локфайла. То есть прод собирается не из того
же разрешения зависимостей, что проверяет CI.

### Next.js — `apps/web` (Legends UI) и `apps/admin_web` (superadmin)

| Что | Значение | Откуда |
|---|---|---|
| Next.js | **`^15.1.0`**, App Router, `output: "standalone"` | обоих `package.json`, `next.config.mjs` |
| React | **`^19.0.0`** + `react-dom ^19.0.0` | обоих `package.json` |
| TypeScript | **`^5.7.0`**, `strict: true`, `target: ES2022`, `moduleResolution: "bundler"`, `paths: {"@/*": ["./src/*"]}` | `package.json`, `tsconfig.json` |
| Линтер | **ESLint 9** + `eslint-config-next ^15.1.0`, запуск через `next lint` | `package.json` |
| Пакетный менеджер | **pnpm** `>=9` (в Docker `pnpm@9.15.0` через corepack), workspace-монорепо | корневой `package.json`, `pnpm-workspace.yaml`, Dockerfile |
| Node | `engines.node >= 20`, образ `node:20-alpine` | корневой `package.json`, Dockerfile |
| Данные | `@tanstack/react-query ^5.62.0` | обоих `package.json` |
| Формы | `react-hook-form ^7.54` + **`zod ^3.24`** + `@hookform/resolvers ^3.9` (только `admin_web`) | `apps/admin_web/package.json` |
| Тесты фронта | **`vitest ^2.1`** только в `apps/web` (`"test": "vitest run"`). В `admin_web` тестов нет | обоих `package.json` |
| Конфиг | `reactStrictMode: true`, `outputFileTracingRoot`, `rewrites()` на `/api-backend/*` → бэкенд | `next.config.mjs` |

**Что осталось неясным:**

- Есть ли отдельный `eslint.config.*` или `.eslintrc` — в дереве `origin/main` файлов ESLint-конфига
  нет вообще (`grep` по `git ls-tree` даёт только `vitest.config.ts` и `pnpm-workspace.yaml`).
  Конфигурация, судя по всему, целиком опиралась на `next lint`, который в Next 16 удалён.
- Нет CI-гейта на фронт: три workflow (`install-e2e`, `nightly-install-e2e`,
  `profile-version-discipline`) гоняют только Python-тесты. `lint`/`typecheck`/`vitest` в CI не
  вызываются.
- Формально «стек профиля» нигде не зафиксирован версиями — ни в манифестах, ни в quickstart'ах, ни в
  скиллах лейнов. Всё, что выше, — вывод из кода монорепо, не из декларации профиля.

## Источники

Все ссылки открыты и проверены 2026-08-03.

| Источник | Тип | Чей | Что именно берём | Применимо к | Как часто обновляется | Сверено |
|---|---|---|---|---|---|---|
| [devguide.python.org/versions](https://devguide.python.org/versions/) | статус-таблица | CPython core devs | какие версии живы, какая в bugfix/security, даты EOL | Python | при каждом релизе/EOL | 2026-08-03 |
| [peps.python.org](https://peps.python.org/) | индекс | PEP editors | куда едет язык: Accepted / Open / Provisional, топики typing и packaging | Python | постоянно | 2026-08-03 |
| [PEP 8](https://peps.python.org/pep-0008/) | стайлгайд | Guido van Rossum, Barry Warsaw, Alyssa Coghlan | базовый стиль кода; статус Active, посл. правка 2025-04-04 | Python | редко, но живой | 2026-08-03 |
| [docs.python.org/3/whatsnew/3.14](https://docs.python.org/3/whatsnew/3.14.html) | release notes | CPython | что нового и что удалено в мажоре | Python | раз в год | 2026-08-03 |
| [blog.python.org](https://blog.python.org/) (Python Insider) | блог релизов | CPython core team | анонсы релизов, беты, security-фиксы | Python | несколько раз в месяц | 2026-08-03 |
| [typing.python.org](https://typing.python.org/en/latest/) | спека + гайды | Typing Council | спецификация системы типов, guides по протоколам/дженерикам/narrowing | Python | по мере принятия typing-PEP | 2026-08-03 |
| [docs.astral.sh/ruff](https://docs.astral.sh/ruff/) · [rules](https://docs.astral.sh/ruff/rules/) · [default-rules](https://docs.astral.sh/ruff/default-rules/) | линтер + правила | Astral | практика, выраженная машиной: 900+ правил, точный дефолтный `select` | Python | каждые 2–4 недели | 2026-08-03 |
| [astral.sh/blog/ruff-v0.16.0](https://astral.sh/blog/ruff-v0.16.0) | release notes | Astral | смена дефолтов 59 → 413 правил (23.07.2026) и как сохранить старое поведение | Python | по мажорам ruff | 2026-08-03 |
| [docs.astral.sh/uv](https://docs.astral.sh/uv/) | доки инструмента | Astral | проекты, локфайл, `uv run`, управление версиями Python, Docker/GitHub Actions | Python | непрерывно | 2026-08-03 |
| [github.com/astral-sh/uv/releases](https://github.com/astral-sh/uv/releases) | release notes | Astral | breaking changes uv (напр. 0.12.0) — uv релизится очень часто | Python | ~еженедельно | 2026-08-03 |
| [mypy.readthedocs.io](https://mypy.readthedocs.io/en/stable/) · [changelog](https://mypy.readthedocs.io/en/stable/changelog.html) | доки + changelog | mypy (Python org) | strict-режим, конфиг-файл, breaking changes 2.x | Python | по релизам | 2026-08-03 |
| [docs.astral.sh/ty](https://docs.astral.sh/ty/) | доки инструмента | Astral | альтернативный тайпчекер на Rust — держать в поле зрения, но статус стабильности на странице не объявлен | Python | непрерывно | 2026-08-03 |
| [fastapi.tiangolo.com/release-notes](https://fastapi.tiangolo.com/release-notes/) | release notes | FastAPI | единственный официальный список изменений; секции «Breaking Changes» по версиям | Python | несколько раз в неделю | 2026-08-03 |
| [Pydantic version policy](https://pydantic.dev/docs/validation/latest/get-started/version-policy/) | политика версий | Pydantic | что V2 не ломает в минорах, что deprecated доживёт до V3, политика по версиям Python | Python | редко | 2026-08-03 |
| [What's New in SQLAlchemy 2.1](https://docs.sqlalchemy.org/en/21/changelog/migration_21.html) | migration guide | SQLAlchemy | поведенческие изменения 2.0 → 2.1 (autoflush, dataclass-дефолты, `filter_by`, psycopg3 по умолчанию) | Python | по мажорам | 2026-08-03 |
| [pytest-asyncio changelog](https://pytest-asyncio.readthedocs.io/en/stable/reference/changelog.html) | changelog | pytest-dev | что сломалось в 1.0 (удалён `event_loop`-фикстура, legacy mode), текущая стабильная 1.4.0 | Python | по релизам | 2026-08-03 |
| [nextjs.org/docs/app/guides/upgrading/version-16](https://nextjs.org/docs/app/guides/upgrading/version-16) | migration guide | Vercel | полный список breaking changes 15 → 16 — главный документ для нашего фронта | Next.js | по мажорам | 2026-08-03 |
| [nextjs.org/docs/app/guides/upgrading/codemods](https://nextjs.org/docs/app/guides/upgrading/codemods) | инструмент миграции | Vercel | `@next/codemod upgrade` и точечные codemod'ы (`middleware-to-proxy`, `next-lint-to-eslint-cli`, …) | Next.js | по мажорам | 2026-08-03 |
| [nextjs.org/docs/app/guides/production-checklist](https://nextjs.org/docs/app/guides/production-checklist) | best practices | Vercel | официальный чеклист: рендеринг, данные и кэш, безопасность (Data Access Layer, tainting, CSP), метаданные, типы | Next.js | по релизам (посл. 2026-03-10) | 2026-08-03 |
| [nextjs.org/docs/app/api-reference/config/eslint](https://nextjs.org/docs/app/api-reference/config/eslint) | конфиг линтера | Vercel | flat-config `eslint.config.mjs`, `eslint-config-next/core-web-vitals` и `/typescript`, полный список правил `@next/eslint-plugin-next` | Next.js | по релизам | 2026-08-03 |
| [nextjs.org/blog](https://nextjs.org/blog) | блог релизов | Vercel | что вышло и когда; там же — security-релизы | Next.js | ~ежемесячно | 2026-08-03 |
| [Security release program](https://nextjs.org/blog/next-security-release-program) + [July 2026 Security Release](https://nextjs.org/blog/july-2026-security-release) | политика + CVE | Vercel | новая модель LTS и предобъявленных ежемесячных патчей; CVE с номерами и патч-версиями | Next.js | ежемесячно | 2026-08-03 |
| [nextjs.org/docs/llms.txt](https://nextjs.org/docs/llms.txt) (+ `llms-full.txt`, суффикс `.md` к любой странице) | машинный индекс доков | Vercel | как агенту читать актуальные доки, а не тренировочные данные | Next.js | вместе с доками | 2026-08-03 |
| [Next.js 16.3: AI Improvements](https://nextjs.org/blog/next-16-3-ai-improvements) | блог | Vercel | AGENTS.md-блок от `next dev`, first-party Skills, DevTools MCP, `/docs/messages` написанные под агентов | Next.js | по релизам | 2026-08-03 |
| [react.dev/versions](https://react.dev/versions) · [react.dev/blog](https://react.dev/blog) | версии + блог | Meta / React Foundation | текущая стабильная версия React, релизы и security-адвайзори | Next.js | несколько раз в год | 2026-08-03 |
| [nodejs.org/en/about/previous-releases](https://nodejs.org/en/about/previous-releases) + [schedule.json](https://raw.githubusercontent.com/nodejs/Release/main/schedule.json) | график поддержки | Node.js | точные даты LTS/maintenance/EOL — машиночитаемо | Next.js | по релизам | 2026-08-03 |
| [devblogs.microsoft.com/typescript](https://devblogs.microsoft.com/typescript/) ([TS 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)) | блог релизов | Microsoft | мажоры TS, смена дефолтов компилятора | Next.js | по релизам | 2026-08-03 |
| [zod.dev](https://zod.dev/) | доки | Zod | текущий мажор и migration guide v3 → v4 | Next.js | по релизам | 2026-08-03 |

## Что изменилось за последний год

Всё сверено с источниками выше 2026-08-03. Сортировка — по тому, насколько сильно расходится с тем,
что сейчас в репозитории.

### 1. Next.js: мажор 16, и наши `^15.1.0` уже вне поддерживаемой линии

Стабильный Next.js сейчас — **16.2**; 16.3 на 03.08.2026 в статусе Preview (дословно у Vercel: «As the Next.js 16.3 Preview nears a stable release»). Vercel ввела **LTS-модель**: Active LTS — линия
**16.2**, Maintenance LTS — **15.5**. Патчи безопасности выходят **раз в месяц с предварительным
анонсом** (программа объявлена 13.07.2026). Июльский релиз (20.07.2026) закрыл 4 HIGH и 5 MEDIUM
CVE — включая DoS через Server Actions (CVE-2026-64641), обход middleware/proxy (CVE-2026-64642) и
два SSRF (CVE-2026-64645, CVE-2026-64649); патч-версии — **16.2.11** и **15.5.21**.

Мы на `next: ^15.1.0`, то есть даже внутри 15.x — не на Maintenance LTS линии 15.5. Плюс отдельно:
в декабре 2025 были RCE в React Server Components (CVE-2025-66478, CVSS 10.0) и связка
DoS + source-code exposure (CVE-2025-55184 / CVE-2025-55183), затрагивавшие 13.x–16.x.

Что ломается при переходе 15 → 16 (полный список — в migration guide, здесь то, что заденет нас):

- **Turbopack по умолчанию** в `next dev` и `next build`. Свой webpack-конфиг → билд падает
  специально; выход — `--webpack` или миграция.
- **`next lint` удалён**, опция `eslint` в `next.config` тоже. Наш `"lint": "next lint"` в обоих
  приложениях перестанет работать. Codemod: `next-lint-to-eslint-cli`. `@next/eslint-plugin-next`
  теперь по умолчанию **flat config** (`eslint.config.mjs`), под ESLint v10, который legacy-конфиги
  дропает.
- **`middleware.ts` → `proxy.ts`**, экспорт `middleware` → `proxy`, `edge`-рантайм в `proxy` не
  поддерживается. Codemod: `middleware-to-proxy`.
- **Синхронный доступ к `cookies`/`headers`/`draftMode`/`params`/`searchParams` полностью удалён**
  (в 15 он был временно совместим). Плюс `params`/`id` стали промисами в `opengraph-image`,
  `twitter-image`, `icon`, `sitemap`.
- **`revalidateTag` требует второй аргумент** (`cacheLife`-профиль); появились `updateTag` и
  `refresh`. `cacheLife`/`cacheTag` вышли из `unstable_`.
- **PPR**: `experimental.ppr` и `experimental_ppr` удалены, вместо них `cacheComponents`; механика
  другая, не «переименование».
- **`next/image` — пять изменений дефолтов**: `minimumCacheTTL` 60с → 4ч, `qualities` только `[75]`,
  из `imageSizes` убран `16`, локальные IP запрещены, редиректы ограничены тремя.
- **Минимумы**: Node.js **20.9+**, TypeScript **5.1+**, браузеры Chrome/Edge/Firefox 111+, Safari 16.4+.
- Удалены: AMP, `serverRuntimeConfig`/`publicRuntimeConfig`, `unstable_rootParams`. Все parallel-route
  слоты теперь требуют явный `default.js`, иначе билд падает.

### 2. Next.js стал писать доки под агентов — прямо в тему нашей задачи

С 16.2 доки Next.js **бандлятся в проект** (`node_modules/next/dist/docs/`), а с 16.3 `next dev` сам
дописывает в `AGENTS.md` блок между маркерами `<!-- BEGIN:nextjs-agent-rules -->` с текстом «This is
NOT the Next.js you know … Read the relevant guide before writing any code». То есть авторы фреймворка
решают ровно ту проблему, из-за которой заведена эта заметка. Механики три:

- **бандленные доки + AGENTS.md** (codemod `agents-md` для проектов на 16.1 и старше);
- **`/docs/llms.txt` и `/docs/llms-full.txt`** плюс суффикс `.md` к любому URL доки — машинное чтение;
- **first-party Skills** (`next-dev-loop`, `next-cache-components-adoption`,
  `next-cache-components-optimizer`, ставятся `npx skills add vercel/next.js --skill <name>`) и
  DevTools MCP (`next-devtools-mcp`). Прежние «knowledge»-скиллы с skills.sh **ретайрнуты** — их
  заменили бандленные доки.

Страницы `/docs/messages/*` теперь написаны в формате Patterns / Trade-offs / Gotchas именно для
агентов, а ошибки в терминале и в оверлее печатают меню фиксов со ссылками на нужную секцию.

### 3. Ruff сменил дефолтный набор правил — 59 → 413

**Ruff 0.16.0** (23.07.2026) впервые с 0.1.0 переписал дефолты: по умолчанию включено **413 правил**
из 968. Из дефолтов при этом убрали 18 «мнением нагруженных» правил (`E401`, `E402`, `E701` и др.).
Сохранить старое поведение можно явным `select = ["E4", "E7", "E9", "F"]`.

Нас это касается косвенно: у нас `select` задан явно (`E,F,I,N,B,UP,ASYNC,S`), так что смена дефолтов
нас не тронет — но и не даст. Наш набор из 8 префиксов теперь заметно уже нового дефолта Ruff, и
пересмотреть его стоит по странице [default-rules](https://docs.astral.sh/ruff/default-rules/). Плюс
в 0.16 появились форматирование Python-блоков в Markdown и комментарии `ruff: ignore` /
`ruff: file-ignore`.

### 4. mypy ушёл в 2.x — а у нас `>=1.10`

Текущая документируемая версия — **mypy 2.3.0**. В 2.0 сменились дефолты, и при `strict = true` это
заденет нас напрямую:

- `--local-partial-types` включён по умолчанию (меняет вывод типов между скоупами);
- `--strict-bytes` по умолчанию (PEP 688): `bytearray`/`memoryview` больше не подставляются под `bytes`;
- `--allow-redefinition` теперь работает как прежний `--allow-redefinition-new`;
- дропнут таргет `--python-version 3.9`;
- новое: параллельная проверка `--num-workers N` (до 5x), бинарный кэш и SQLite-кэш по умолчанию.

В 2.2 добавились closed TypedDict (PEP 728) и полные дефолты тайпваров (PEP 696), `TypeForm` вышел из
экспериментального. Рядом с mypy подрос **ty** от Astral (Rust, заявлено 10–100x) — на его доках
статус стабильности не объявлен, так что это «наблюдать», а не «переезжать».

### 5. Node.js 20 стал EOL — а мы на нём собираемся

По официальному `schedule.json`: **Node 20 (Iron) — end-of-life 2026-04-30**, то есть уже. Актуальные
LTS — **22 (Jod, до 2027-04-30)** и **24 (Krypton, до 2028-04-30)**; **26** вышел 2026-05-05 и станет
LTS 2026-10-28. У нас `engines.node >= 20` и `node:20-alpine` в трёх Dockerfile'ах — образ без
обновлений безопасности. Начиная с Node 27 меняется сама модель: ежегодный цикл, каждый мажор уходит
в LTS после 6 месяцев Current.

### 6. Python: 3.12 уже только security, вышел 3.14, на подходе 3.15

- **3.12 (наша версия) — фаза security-only** с 2025 года; EOL ~октябрь 2028. Бинарники больше не
  выпускаются, только исходники с фиксами безопасности.
- **3.13** — bugfix (EOL ~2029-10), **3.14** — вышел 07.10.2025, bugfix (EOL ~2030-10).
- **3.15** — в бете (beta 4 от 18.07.2026), релиз запланирован на 2026-10-01.
- **PEP 2026 (календарное версионирование, «3.26 вместо 3.15») — Rejected.** Схема версий не меняется;
  в поиске эта информация часто подаётся как принятая — это неверно.

Главное в 3.14: свободнопоточный Python официально поддерживается (PEP 779, штраф на однопоточном коде
5–10%), отложенное вычисление аннотаций (PEP 649/749 + модуль `annotationlib`), t-строки (PEP 750),
`concurrent.interpreters` (PEP 734), внешний отладчик (PEP 768), zstd (PEP 784).

### 7. Библиотеки бэкенда

- **FastAPI**: текущая — **0.141.1** (29.07.2026), мы на `>=0.115`. 1.0 не вышло. За 2026 год как
  минимум шесть релизов помечены «Breaking Changes» (0.125, 0.127, 0.128, 0.129, 0.131, 0.132, 0.137);
  0.137.0 (14.06.2026) — самый крупный. Новое из свежего: `app.frontend()` для локальной разработки,
  снижение потребления памяти в зависимостях.
- **SQLAlchemy**: серия **2.1** в бете (2.1.0b3, 27.06.2026), у неё есть свой «What's New».
  Поведенческие изменения, которые заденут нас: сессия теперь **безусловно flush'ит перед любым**
  execute (не только ORM), дефолты dataclass больше не пишутся в `__dict__`, `filter_by()` бросает
  `AmbiguousColumnError` вместо молчаливого выбора, **PostgreSQL по умолчанию едет на psycopg v3**
  вместо psycopg2 (у нас в зависимостях уже есть оба — asyncpg и psycopg3).
- **Pydantic**: V2 остаётся текущей, **V3 запланирована, но не вышла**; в V2 минорах ломающих
  изменений не будет, deprecated живёт до V3. Практическая деталь: **доки переехали** с
  `docs.pydantic.dev` на `pydantic.dev/docs/validation/...` — старые ссылки в скиллах отдадут 301.
- **pytest-asyncio**: стабильная — **1.4.0**, мы на `>=0.23`. В 1.0 удалена фикстура `event_loop` и
  legacy-режим. Наш `asyncio_mode = "auto"` задан явно, поэтому смена дефолта на `strict` нас не
  ломает, но диапазон `>=0.23` разрешает поставить и 0.23, и 1.4 — то есть версия де-факто
  определяется локфайлом, а не конфигом.
- **uv**: текущая — **0.12.1** (31.07.2026). В **0.12.0** (28.07.2026) ломающее: `uv init` теперь по
  умолчанию проставляет build-систему `uv_build`; ужесточена валидация архивов и хэшей (MD5-only
  отвергается); изменена работа с пре-релизами. Появился `uv check` (в превью — с `--fix`).

### 8. TypeScript 7.0 — нативный порт, вышел 08.07.2026

Компилятор переписан на Go, ускорение на реальных кодовых базах 8–12x (VS Code: 125.7с → 10.6с),
память минус 6–26%. **Сменились дефолты компилятора**: `strict` включён по умолчанию, `module` по
умолчанию `esnext`, `types` теперь `[]` (без автообнаружения), `rootDir` — `./`. Проекты на `es5`,
`amd`/`umd` или `baseUrl` требуют правок конфига; рекомендованный путь — сначала 6.0, потом 7.0, с
пакетом совместимости `@typescript/typescript6`. Мы на `^5.7.0`.

### 9. React и Zod

- **React**: стабильная — **19.2** (01.10.2025), 19.3/20 нет. React Compiler достиг **1.0** (07.10.2025)
  и в Next.js 16 поддержка компилятора стабильна (`reactCompiler: true`, по умолчанию выключено).
  React перешёл под **React Foundation** (Linux Foundation) — объявлено 07.10.2025, запуск 24.02.2026. Отдельно:
  App Router в Next.js 16 использует React **Canary**, а не ровно 19.2.
- **Zod**: **v4 стабильна**, есть release notes и migration guide на `zod.dev/v4`. Мы на `^3.24`.
  Пакет и импорт (`npm install zod`, `import * as z from "zod"`) не менялись.

## Что переписать в профиле прямо сейчас

- **Заменить `next: ^15.1.0` на линию LTS** — минимум `15.5.21` (Maintenance LTS), целевое `16.2.11` (Active LTS): `^15.1.0` вне обеих линий, то есть не получает даже ежемесячных патчей безопасности, включая июльский набор из 4 HIGH и 5 MEDIUM CVE.
- **Убрать `"lint": "next lint"` и опцию `eslint` из `next.config`, прописать ESLint CLI с flat-config `eslint.config.mjs`** — в Next.js 16 `next lint` удалён, а `@next/eslint-plugin-next` по умолчанию перешёл на flat config; готовый путь — codemod `next-lint-to-eslint-cli`.
- **Поднять `engines.node` и базовые образы с 20 на 22 или 24** — Node 20 (Iron) EOL 30.04.2026 по официальному `schedule.json`, а Next.js 16 требует минимум 20.9; `node:20-alpine` в трёх Dockerfile'ах — образ без обновлений безопасности.
- **Пересобрать `select` у ruff (`E,F,I,N,B,UP,ASYNC,S`) по странице default-rules** — в 0.16.0 (23.07.2026) дефолтный набор переписан впервые с 0.1.0, 59 → 413 правил; наш явный `select` смену дефолта не ломает, но теперь заметно уже нового дефолта.
- **Заменить в скиллах ссылки на доки Pydantic** — `docs.pydantic.dev/...` → `pydantic.dev/docs/validation/...`: доки переехали, старые адреса отдают 301.
- **Зафиксировать в манифестах и quickstart'ах сам стек версиями, а не метками `python` / `nextjs`** — сейчас профиль встанет на репозиторий с Django и poetry и ничего не заметит; вся таблица стека в этой заметке — вывод из кода монорепо, а не декларация профиля.

## Чего не нашёл / где источник слабый

- **Стек профиля нигде не задекларирован версиями.** Ни манифесты, ни quickstart'ы, ни скиллы лейнов
  не называют ни фреймворка, ни версии — только метки `python` / `nextjs`. Вся таблица выше — вывод из
  кода `tacticum-dev`. Если профиль когда-нибудь поставят на репозиторий с другим питон-стеком (Django,
  poetry), эта заметка к нему не применима, и в самом профиле нет ничего, что бы это поймало.
- **`tacticum-role-internal` — `stack: []`.** Формально к этой заметке роль не относится вообще; я
  включил её только потому, что она работает по тому же монорепо. Считать её «Python + Next.js» —
  домысел, и в задачу это лучше вернуть отдельным вопросом.
- **FastAPI: официальных migration guide нет.** Есть только сплошная лента release notes, где
  «Breaking Changes» — заголовок секции внутри записи о версии. Что именно сломалось в 0.137.0,
  вытащить со страницы не удалось: она слишком велика и обрезается при загрузке. Для реальной сверки
  придётся читать GitHub Releases по каждой версии отдельно. Это самое слабое место каталога:
  фреймворк с 0.x-версионированием и десятком ломающих релизов в год, без единого документа миграции.
- **`ty` (Astral) — статуса стабильности на официальной странице нет.** Ни «alpha», ни «beta», ни
  версии. Рекомендовать его как замену mypy нельзя, пока авторы сами не назовут стадию.
- **TanStack Query** — доки на `tanstack.com/query/latest` живые, но текущий мажор и наличие
  migration guide со страницы overview не считываются. Версию (`^5.62.0` у нас) сверять придётся по
  npm/GitHub, а не по докам.
- **Alembic** — отдельно не сверял; у него нет самостоятельного цикла breaking changes, миграции
  тянутся за SQLAlchemy.
- **Доки отдают 16.2 — и это верно: 16.2 и есть стабильная линия.** Страницы отдают
  `version: 16.2.12`; про 16.3 говорят только блог-посты. Возможно расхождение между блогом и
  reference-докой; в спорном случае верить reference, а не блогу.
- **CI по фронту отсутствует**, поэтому проверить утверждения о лишних/сломанных правилах ESLint или
  типах на реальном прогоне нельзя — сверка велась по конфигам, не по прогону.

## Владелец ревью

| Роль | Кто | Дата сверки |
|---|---|---|
| Владелец стека | _не назначен — впишите себя_ | 2026-08-03 |

Правьте эту страницу под свой стек: вычёркивайте источники, которыми не пользуетесь, дописывайте свои, поправьте описание стека, если оно разошлось с реальностью.
