---
title: Разведка — дизайн-система в репозитории tacticum-dev
type: note
status: draft
created: 2026-07-27
tags:
- board
- design-system
- explore
permalink: tacticum/00-board/explore-ds-repo-2026-07-27
archived-at: 2026-08-04 10:01
---

# Разведка: ДС в tacticum-dev

Репозиторий `/Users/bubblemac/tacticum/tacticum-dev`, ветка `main`, только чтение.
Все утверждения ниже — с пруфом (путь + строка / хеш коммита / версия).

## 1. `design-systems/` — инвентаризация

Четыре директории (`ls design-systems/`): `iva-web`, `iva-mobile`, `iva-rn`, `tacticum-web`.
Ожидание тимлида подтверждено полностью — ровно эти четыре, ни больше ни меньше.

| ДС | версия | tokens.json | токенов (листьев) | бимодальных | code-bindings | organization_id | последний коммит |
|---|---|---|---|---|---|---|---|
| iva-web | 0.3.0 | 360K | 1021 | 342 | **49 компонентов**, figma_key заполнен 32/49 | `TBD` | `2f42190` 2026-07-23 Dmitry Solonko |
| iva-mobile | 0.2.0 | 320K | 966 | 331 | **34 компонента**, figma_key заполнен 0/34 (все `null`) | `TBD` | `bfbd33c` 2026-07-23 Dmitry Solonko |
| iva-rn | 0.1.0 | 16K | 120 | 40 | нет `$extensions` вообще | `TBD` | `2e5f1f2` 2026-06-18 Diaret |
| tacticum-web | 0.1.0 | 12K | 153 | 33 | нет `$extensions` вообще | `TBD` | `4aad5e7` 2026-07-19 Diaret |

`organization_id: TBD` во **всех четырёх** — пруф: `iva-web/design-system.yaml:22`,
`iva-mobile/design-system.yaml:19`, `iva-rn/design-system.yaml:39`,
`tacticum-web/design-system.yaml:18`. Это не забывчивость, а замысел: реальный UUID
подаётся флагом `--organization-id` при сиде (`seed_runner.py:96-101`, комментарий в
докстринге `seed_runner.py:9-11`).

Все четыре: `status: published`, `modes: [light, dark]`.
`source_type`: `tokens_studio_export` у iva-web/iva-mobile, `dtcg_manual_upload` у
iva-rn/tacticum-web.

### Состав файлов

- `iva-web/`, `iva-mobile/`, `iva-rn/` — ровно два файла: `tokens.json` + `design-system.yaml`.
- `tacticum-web/` — пять: плюс `DESIGN.md` (20K, человекочитаемое описание ролей токенов),
  `build_preview.py` (12K, standalone-генератор шоукейса, только stdlib) и `preview.html` (20K).

### Структура tokens.json

**iva-web** (27 групп верхнего уровня): `solid` (243 токена, 14 подгрупп-цветов),
`alpha` (199), `brandbook` (54, внутри `light`/`dark`), `brand` (34), типографика
(`font-family`, `font-weight`, `lineHeights` 24, `fontSize` 9, `heading` 12, `body` 12,
`paragraphSpacing` 9 …), раскладка (`gap` 27, `padding` 27, `radius` 14, `width` 3),
семантика (`bg` 180 в 87 подгруппах, `icon` 59, `text` 57, `border` 41, `functional` 5).

**iva-mobile** (36 групп): те же примитивы `solid`/`alpha`/`brandbook`/`brand`
(243/199/54/34 — **побайтово общие с web**, один Figma-файл примитивов, см. п.2),
плюс мобильная специфика: `context` (7), `bg (Grouped)` (4 — iOS-неймспейс),
восемь одиночных композитных BG-групп (`BG primary plain`, `BG secondary modal`, …),
`@ROW Left Edit padding` и т.п. Семантика: `bg` 170, `text` 56, `icon` 52, `border` 38.

**iva-rn** (11 групп, всё плоско и чисто): `color` 43, `personColor` 8, `spacing` 13,
`radius` 11, `fontSize` 7, `lineHeight` 5, `fontWeight` 4, `typography` 8, `avatarSize` 7,
`layout` 9, `motion` 5. Не Figma-экспорт — 1:1 снимок `rn-shared/src/theme`
(`palette.ts` + `tokens.ts`), см. шапку `iva-rn/design-system.yaml:3-17`.

**tacticum-web** (12 групп): `palette` 54 (navy/cobalt/graphite/green/amber/red/cyan),
`color` 31 (surface/text/border/interactive/status), `typography` 19, `space` 10,
`control` 3, `density` 8 (comfortable/compact), `radius` 5, `elevation` 4, `focus` 1,
`motion` 6, `layout` 6, `z` 6. Рукописная DTCG, не экспорт.

### Темы (modes)

Во всех четырёх ДС темы живут **не** отдельными деревьями, а внутри токена:
`"$value": {"light": …, "dark": …}`. Пример: `iva-web/tokens.json`, токен
`bg.light-primary-hovered` → `{"light": "{solid.gray.300}", "dark": "{solid.gray.300}"}`.
Схлопывание в один режим делает рантайм (`design/domain/token_resolver.py`), а не файл.

### `$extensions."dev.tacticum.code-bindings"`

Есть только у iva-web и iva-mobile, схема `tacticum-code-bindings/v0`, ручная курация.

- **iva-web**: 49 компонентов, `figma_file_key: AG11paSthGC7zSoovfjip0`,
  кодовая база `@iva/design-system` в `iva-one`, снято с коммита `daf1424ad 2026-07-20`,
  Storybook `https://iva-ui.ivcs.su/`. Поля компонента: `name`, `match` (алиасы),
  `figma_key`, `selector`, `kind`, `source`, `storybook`, `inputs`, `notes`.
  **32 из 49 figma_key заполнены**; `null` = компонент лежит в другом файле
  мультифайловой ДС (`iva-web/design-system.yaml:13-16`).
- **iva-mobile**: 34 компонента, `stack: compose-mp`, `figma_file_key: null`,
  кодовая база KMP `:core:design-system` (`su.ivcs.messenger.designsystem`), коммит
  `8f65eea2ff 2026-07-23`, Storybook отсутствует — вместо него `@Preview`-композаблы.
  **figma_key = null у всех 34**: авторитетный мобильный Figma-файл ещё не подтверждён
  дизайнерами (`iva-mobile/design-system.yaml:12-13`).

## 2. `apps/backend/scripts/merge_iva_tokens.py` (341 строка)

**Где захардкожены профили.** Словарь `PROFILES` — **строки 48-67**:
`iva-web` — строки 49-55, `iva-mobile` — строки 56-66. Второй словарь `PROFILE_METADATA`
(имя/описание/платформа/framework_hint/modes) — **строки 69-96**: `iva-web` 70-81,
`iva-mobile` 82-95. Третьего профиля нет ни в одном.

**Какие файлы ожидает.** Точные имена из `PROFILES`:

- iva-web: `IVA DS_ Color primitives.json` (49), `IVA DS_ Color typography.json` (50),
  `IVA DS_ Color functional.json` (51), `IVA DS_ Color tokens - light.json` (52),
  `IVA DS_ Color tokens - dark.json` (53).
- iva-mobile: тот же `IVA DS_ Color primitives.json` (57 — примитивы общие с web),
  `IVA Mobile DS_ Functional.json` (61 — на деле типографика, имя врёт),
  `IVA Mobile DS_ Typography.json` (62 — на деле композитные BG, имя врёт),
  `IVA Mobile DS_ Color tokens - context.json` (63),
  `IVA Mobile DS_ Color tokens - light.json` (64), `… - dark.json` (65).

**Откуда читает.** `SRC = REPO / "docs" / "concept" / "design" / "tokens"` — **строка 42**,
`REPO` = `parents[3]` от файла скрипта (строка 41), т.е. корень репозитория.

> **Этой папки на машине нет.** `ls docs/concept/design/tokens/` → `No such file or directory`.
> `/docs/concept` целиком в `.gitignore:33`, `git ls-files docs/concept/design` пуст.
> Вывод: **скрипт сейчас физически не запустится** — входные Figma-экспорты в репозиторий
> не коммитятся и локально отсутствуют.

**Что делает.** Грузит JSON'ы (264) → мобильный `context` заворачивает в неймспейс
`context/`, чтобы не столкнулся с семантическим `bg` (269-270) → срезает неймспейс
`legacy` из light и dark (277-278; у web он есть, у mobile нет) → `merge_modes` (135-188)
склеивает одинаковые по форме деревья light и dark в один токен с
`$value: {light, dark}` → всё одномодовое кладёт в корень, семантику последней (293-303,
при коллизии верхнеуровневых ключей побеждает поздний) → проверяет, что каждый
алиас `{path.to.token}` разрешается в существующий токен (307-309) → печатает статистику.

**Что на выходе.** Строки 328-332: `design-systems/<profile>/tokens.json` (`json.dumps`,
indent=2) и `design-systems/<profile>/design-system.yaml` (из `build_yaml_stub`, 221-241).
Код возврата 0, если неразрешённых алиасов нет, иначе 2 (336).

### РИСК: повторный прогон затрёт ручную работу

Это не гипотеза, это следует из кода:

1. **`$extensions."dev.tacticum.code-bindings"` исчезнет.** `merged` (строка 293)
   собирается только из входных Figma-файлов; ни одна строка скрипта не читает и не
   переносит `$extensions` верхнего уровня. Запись на строке 331 — безусловный
   `write_text`. Итог: 49 биндингов iva-web и 34 iva-mobile, включая 32 добытых через
   Figma REST API `figma_key`, будут стёрты.
2. **`design-system.yaml` откатится на 0.1.0.** `build_yaml_stub` хардкодит
   `version: 0.1.0` (строка 237), `source_type: tokens_studio_export` (238),
   `source_ref: docs/concept/design/tokens/` (240), `organization_id: TBD` (235),
   а описание берёт из `PROFILE_METADATA`, где нет ни абзацев про 0.2.0/0.3.0, ни
   упоминания code-bindings. Текущие 0.3.0 и 0.2.0 — результат ручной правки поверх
   генерации; прогон их снесёт.

### Что нужно изменить для ТРЕТЬЕЙ ДС

Минимально:

1. Добавить ключ в `PROFILES` (после строки 66) — имена входных файлов новой ДС.
2. Добавить ключ в `PROFILE_METADATA` (после строки 95) — name/description/platform/
   framework_hint/modes. Без него `build_yaml_stub` (строка 222) упадёт по `KeyError`.
3. Флаг `--profile` расширять не надо: `choices=sorted(PROFILES.keys())` (строка 251)
   подхватывает автоматически.

Дополнительно — если у новой ДС другая форма источника:

4. **Строки 277-282** жёстко требуют ключи `light` и `dark` в `loaded`:
   `strip_namespace(loaded["light"], …)` и `merge_modes(loaded["light"], loaded["dark"])`.
   Одномодовая или трёхмодовая ДС здесь падает по `KeyError` — нужна ветка.
5. **Строка 42** — единственная папка-источник на все профили. ДС из другого места
   потребует `source_dir` в записи `PROFILES`.
6. **Строки 269-270** — заворачивание `context` захардкожено по имени ключа; на новом
   профиле с ключом `context` сработает молча.
7. **Строки 328-332** — если у новой ДС тоже будут `$extensions`/code-bindings, нужен
   merge-с-существующим-файлом вместо перезаписи (см. РИСК выше). Это же чинит и iva-*.

## 3. `seed_design_system`

Это **не MCP-тул**, а application-service + CLI-раннер. Два файла:

- `apps/backend/src/backend/design/application/seed_design_system.py` (181 строка) —
  сама операция.
- `apps/backend/src/backend/design/application/seed_runner.py` (118 строк) — CLI-обёртка.

**Запуск — ручной.** `python -m backend.design.application.seed_runner <path>
--organization-id <uuid> [--database-url <dsn>]` (`seed_runner.py:3-5, 86-113`).
`DATABASE_URL` можно отдать переменной окружения (109-111).

**Что заливает.** `build_payload` (`seed_runner.py:39-58`) читает из директории
`design-system.yaml` + `tokens.json`, собирает `DesignSystemVersionPayload`.
`seed_design_system` пишет три строки в три таблицы:
`design_systems` (upsert по паре `(organization_id, design_system_id)`, строки 71-95),
`design_system_versions` (новая версия с `tokens_json`, `modes`, `published_at`,
строки 146-155), `design_system_imports` (аудит-строка с `raw_payload_hash`, 156-167).

**Идемпотентность и иммутабельность.** Хеш — sha256 канонического JSON только
**токенов**, метаданные исключены намеренно (`seed_design_system.py:47-60`).
Если версия уже есть и хеш совпал → `noop` (119-130); если хеш другой → **`rejected`**,
изменение опубликованной версии запрещено по ADR-0009 (131-141). То есть поправить
tokens.json, не подняв версию, невозможно — сид откажет.

**organization_id.** В YAML лежит `TBD`; `build_payload` его вообще не читает —
подставляется значение обязательного флага `--organization-id`
(`seed_runner.py:52`, `96-101`, докстринг `9-11`).

**Привязка к Workspace НЕ делается сидом.** Сид создаёт только DS + версию.
Привязка — отдельный админский HTTP-эндпоинт (см. п.4). Таблица
`workspace_design_systems` в `seed_design_system.py` не упоминается.

**Валидация на входе.** `DesignTokensBlob` (`design/domain/tokens.py:58-75`) —
`RootModel[dict]` с одним инвариантом: каждый алиас `{path}` в любом `$value`
резолвится в реальный токен того же дерева. `$extensions` не валидируется и не
трогается — но и не теряется: в БД уезжает всё дерево целиком
(`seed_design_system.py:151` → `payload.tokens.model_dump()`).

## 4. Backend design-домен

Bounded context `design` — шестой (ADR-0026), 20 файлов в
`apps/backend/src/backend/design/`.

### Таблицы (`design/infrastructure/models.py`, 202 строки, миграция alembic 0015)

| Таблица | Модель | Ключевое |
|---|---|---|
| `design_systems` | `DesignSystem` (37-80) | FK на `organizations`, `UNIQUE(organization_id, design_system_id)` (73-75), status ∈ draft/published/deprecated (76-79) |
| `design_system_versions` | `DesignSystemVersion` (83-125) | `tokens_json` JSONB (100), `modes` JSONB (101-103), `UNIQUE(design_system_id_fk, version)` (118-120) |
| `design_system_imports` | `DesignSystemImport` (128-165) | `raw_payload_hash` LargeBinary (143), `source_type` ∈ tokens_studio_export / dtcg_manual_upload / figma_rest_api (156-159) |
| `workspace_design_systems` | `WorkspaceDesignSystem` (168-201) | N:M Workspace↔DS, композитный PK (198-200), `design_system_version_pinned` (187-189) |

### MCP-тулы (4 штуки, регистрируются в общем сервере `tacticum-mcp`)

Регистрация — `apps/backend/src/backend/platform/app_factory.py:173-176`:
`design_list_systems`, `design_get_tokens`, `design_get_theme_tokens`, `design_resolve_token`.
Отдельного сервера нет — это решение ADR-0029.

- `design_list_systems(installation_id=None)` —
  `design/interface/mcp/design_list_systems.py:24`. Отдаёт ДС, **привязанные к воркспейсу
  вызывающего** (цепочка Installation → workspace_id, строки 36-53). Поля:
  `design_system_id` (slug), `name`, `platform`, `framework_hint`, `version_pinned`,
  `attached_at`. Не привязана — не видна.
- `design_get_tokens(design_system_id, version=None, installation_id=None)` —
  `design_get_tokens.py:29`. Версия по умолчанию — приколотая в
  `workspace_design_systems.design_system_version_pinned` (строка 66). Отдаёт сырое
  DTCG-дерево как есть.
- `design_get_theme_tokens(design_system_id, mode, version, groups, max_chars=50_000,
  installation_id)` — `design_get_theme_tokens.py:20`. Схлопывает бимодальные `$value`
  в один режим и разрешает алиасы (`resolve_all_tokens`, строка 50), умеет выборку
  подветок `groups` (52-68) и обрезку по `max_chars` с полями `truncated` /
  `omitted_groups` (69, 86-88). Обрезка добавлена коммитом `f3e0391` (2026-07-10,
  «fix(#642)»): полное дерево iva-web не влезает в лимит tool-результата клиента.
- `design_resolve_token(design_system_id, token_path, mode, version, installation_id)` —
  `design_resolve_token.py:20`. Один токен → `{value, type, source_path}`.

### Премиум-гейт (ADR-0028)

`require_premium_tier(scope)` — `design/interface/mcp/common.py:7-23`. Проверка:
`scope.tier not in ("full", "trial")` → `AuthError("seat_required")` (строки 19-23).
Вызывается напрямую в `design_list_systems.py:33` и `design_get_tokens.py:40`;
`design_get_theme_tokens` и `design_resolve_token` наследуют гейт, потому что оба
идут через `design_get_tokens`.

**Расхождение нормы и кода, зафиксированное в самом коде.** ADR-0028 Decision 3
(строка 52 ADR) требует строго: `subscription.status='active' AND plan IN
('trial','team','enterprise')`, free → 402. Реализовано мягче — по `tier`, а не по
`plan`; докстринг `common.py:15-17` это признаёт прямым текстом: «relaxed for MVP …
strict plan-level enum is a v1.1 tightening». Не расхождение по недосмотру, а
осознанное послабление, но норма и код сейчас говорят разное.

Отдельно: `@require_seat` для design-тулов не годится — он смотрит только в
`current_scope` (legacy `tac_*`), а design-тулы принимают ещё и per-developer `phk_*`
(`common.py:10-13`).

### Админский HTTP-API

- `apps/backend/src/backend/design/interface/admin/design_systems.py` (81 строка),
  префикс `/admin/design-systems` (строка 19): `GET ""` (22-23, фильтр по
  `organization_id`), `GET /{design_system_id}/versions` (49-50, фильтр по status).
- `apps/backend/src/backend/design/interface/admin/workspace_attach.py` (205 строк),
  префикс `/admin/workspaces` (строка 32): `POST /{workspace_id}/design-systems`
  (52-53) — **та самая привязка к воркспейсу с пином версии**,
  `GET /{workspace_id}/design-systems` (128-129),
  `DELETE /{workspace_id}/design-systems/{design_system_id}` (163-164).

Оба роутера смонтированы в `app_factory.py:365-366`.

**Полный путь токена наружу:** Figma → (ручной экспорт) → `docs/concept/design/tokens/`
→ `merge_iva_tokens.py` → `design-systems/<ds>/` → (ручной `seed_runner`) →
Postgres → (ручной `POST /admin/workspaces/{id}/design-systems`) → MCP-тулы → агент в IDE.
**Три ручных шага подряд**, ни одного автоматического.

## 5. ADR по дизайн-системе и токенам

Всего в `docs/adr/` 60 ADR + README. Прямо про ДС — пять:

| № | Заголовок | Статус | Суть |
|---|---|---|---|
| 0026 | Design as 6th bounded context (DesignSystem + Workspace-level multi-attach) | **Accepted** (2026-05-23) | Design — отдельный 6-й BC со своим доменом, а не ingredient-kind в Catalog и не часть Knowledge. Cardinality 1:N tenant→DS (у IVA web+mobile в одном тенанте), привязка на уровне Workspace. |
| 0027 | Tokens Studio export as MVP import bootstrap (reject .fig parser + Figma REST direct) | **Accepted** (2026-05-23) | Импорт на MVP — ручной экспорт плагином Tokens Studio силами Tacticum-staff, коммит в репо. Парсер `.fig` и Figma REST напрямую отклонены. Триггер — **manual**, никаких вебхуков и расписаний (Decision 4). |
| 0028 | Tacticum Design two-phase rollout (backend MVP → design_web Phase 2) | **Accepted** (2026-05-23) | Phase 1 — только бэкенд (MCP-тулы + импорт), без нового фронтенда. Тир-гейт: MCP design-тулы — премиум strict, free → 402 (Decision 3, стр. 52). `draft` на MVP не используется, `published` ставится сидом. |
| 0029 | Design MCP tools под существующим `tacticum-mcp` (namespace через tool name prefix) | **Accepted** (2026-05-23) | Отдельного сервера `tacticum-design-mcp` не будет: 4 тула регистрируются в общем `tacticum-mcp`, неймспейс — префиксом `design_*`. Причина — стоимость настройки у клиента (5 CLI × свой конфиг). |
| 0046 | Design Studio: agent-driven design authoring в Legends | **Proposed** (2026-06-18) | Авторинг ДС внутри платформы вместо Figma: снимает vendor-lock, даёт дизайнеру поверхность в продукте. Со временем дополняет/заменяет путь ADR-0027. **Не принят.** |

Косвенно упоминают ДС (не решают про неё): 0018 (admin_web MVP scope, 4 упоминания),
0034 и 0051 (по 8 упоминаний, но про Knowledge BC и project-hub как IdP соответственно),
0041, 0043, 0004, 0037, 0045, 0032, 0049, 0050, 0052, 0053 — по 1-3 упоминания.
ADR-0031 упоминает design **documents** (документы проектирования), к дизайн-системе
отношения не имеет — 0 упоминаний ДС.

## 6. Гит-свежесть и автоматизация

### История (`git log --oneline -20 -- design-systems/ apps/backend/scripts/merge_iva_tokens.py`)

Всего 7 коммитов за всю историю:

```
bfbd33c 2026-07-23 Dmitry Solonko  feat(design): iva-mobile DS 0.2.0 — KMP code-bindings + kmp figma quickstart
2f42190 2026-07-23 Dmitry Solonko  feat(design): iva-web DS 0.3.0 — stable figma_key in code-bindings
6ccd176 2026-07-22 Dmitry Solonko  feat(design): iva-web DS 0.2.0 — Figma component code-bindings ($extensions) (#122)
4aad5e7 2026-07-19 Diaret          Add tacticum-web design system and skill
2e5f1f2 2026-06-18 Diaret          Add IVA RN design system and tokens
ed2272f 2026-05-23 Diaret          Add design seed service, runner, and IVA mobile
82dfe0b 2026-05-23 Diaret          Add design system tokens, import script, ADRs
```

Последняя активность — 2026-07-23, четыре дня назад, Dmitry Solonko, три коммита подряд
про code-bindings. Сами токены (значения) с 2026-05-23 (iva-web/iva-mobile) и
2026-06-18 / 2026-07-19 (iva-rn / tacticum-web) не менялись — все три июльских коммита
трогают только `$extensions`.

Бэкенд design-BC свежее не сильно: `f3e0391` 2026-07-10 (обрезка вывода
`design_get_theme_tokens` по `max_chars`).

### Автоматизация пересборки токенов

**Автопересборки нет. Проверено:**

- `.github/workflows/` — два файла: `nightly-install-e2e.yml`, `profile-version-discipline.yml`.
  Ни один не упоминает `design`/`seed_runner` (грепал).
- `.gitignore:37-41` игнорирует все workflow, кроме этих двух явных исключений.
- `ci/gitlab/governance-gate.gitlab-ci.yml` — единственный gitlab-CI в репо, `design` не упоминает.
- `.gitlab-ci.yml` в корне отсутствует.
- `deploy/` — упоминаний `design`/`seed_runner` нет.
- `package.json` — 4 скрипта, все про `admin_web`; ни husky, ни `.pre-commit-config.yaml` нет.

**Отдельная находка — призрачный workflow.** Тест
`apps/backend/tests/design/test_seed_workflow_yaml.py:18` ожидает файл
`.github/workflows/seed-design-systems.yml` (триггер на push в main по пути
`design-systems/**`, шаг с `backend.design.application.seed_runner`). Файла нет,
и `git log --all -- .github/workflows/seed-design-systems.yml` **пуст** — он никогда
не коммитился. Тест это переживает: строка 31 — `pytest.skip("… not present
(gitignored local-only workflow)")`. То есть автосид по коммиту был задуман (ADR-0027
Decision 1 п.6, ADR-0028 Decision 4 «published auto на CI seed»), тест на него написан,
но самого workflow в репозитории нет — он либо живёт только у кого-то локально, либо
не существует вовсе. Со стороны репозитория проверить нельзя.

## Что осталось непроверенным

- Реально ли отрабатывает автосид где-то вне репозитория (GitHub Actions в другом
  месте, ручной прогон на сервере) — из репозитория не видно, **не проверял**.
- Состояние Postgres: какие ДС реально засижены, какие версии приколоты к каким
  воркспейсам — **не проверял**, нужен доступ к БД.
- Совпадают ли `design-systems/*/tokens.json` с тем, что сейчас в Figma, — источники
  (`docs/concept/design/tokens/`) на машине отсутствуют, сверить **не с чем**.
- Актуальность code-bindings относительно `iva-one` (`daf1424ad`) и KMP (`8f65eea2ff`) —
  чужие репозитории, **не проверял**.