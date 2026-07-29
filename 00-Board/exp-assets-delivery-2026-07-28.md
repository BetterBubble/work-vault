---
title: 'Разведка: доставка metadata.assets / metadata.scripts у skill_spec'
type: note
status: current
created: 2026-07-28
updated: 2026-07-28
tags:
- board
- lite-base
- tech-debt
permalink: tacticum/00-board/exp-assets-delivery-2026-07-28
---

# Разведка: доставка assets/scripts у skill_spec

Репозиторий: `/Users/bubblemac/tacticum/tacticum-dev`, ветка `main`, только чтение.

## Вердикт

**Доставки НЕТ.** Поля `metadata.assets` / `metadata.scripts` чисто объявительные:
схема авторинга их разрешает, модель рендера их парсит — и на этом всё. Ни один
путь рендера/установки их не читает и файлов по ним не пишет.

## Ключевой факт (весь поиск в одну строку)

По всему `apps/backend/src/` слово `assets` встречается **ровно в одном файле** —
в самой модели, которая его объявляет:

    apps/backend/src/backend/catalog/domain/ingredients/skill_spec.py:32

Слово `scripts` как ключ/атрибут в `src/` **не встречается нигде**. Потребителя у
полей нет by construction.

## Поверхность (символы → что делают)

| Символ | Файл:строка | Что пишет |
|---|---|---|
| `SkillMetadata.assets` / `.scripts` | `catalog/domain/ingredients/skill_spec.py:32-33` | только объявление, `list[str]` |
| `ClaudeCodeRenderer._render_skill` | `catalog/infrastructure/renderers/claude_code.py:46-56` | **один** `RenderedFile` = frontmatter + `s.body` |
| `render_for_claude_code` | `catalog/domain/renderer.py:207-212` | `{"action": "write_file", "path": path, "content": body}` — один файл |
| `_render_via_canonical` (codex/copilot) | `catalog/domain/renderer.py:307-341` | один `RenderedFile` на ингредиент, через `_rendered_file_to_action` |
| `RenderedFile` | `catalog/infrastructure/renderers/base.py:17-27` | dataclass с **одним** `relative_path` + `content` |

**Структурный аргумент, а не только эмпирический:** контракт `RenderedFile`
(`base.py:17`) и протокол `TargetRenderer.render → RenderedFile | None`
(`base.py:33`) физически не способны выразить «много файлов». Даже при желании
доставить assets текущий контракт рендера этого не позволяет — нужна смена
сигнатуры (`RenderedFile` → `list[RenderedFile]`), а не «дописать чтение поля».

## Что попадает в каталог при сиде

`apps/backend/scripts/seed_community.py:44-71` — резолвит **только** `body_path` в
строку `body`. Обхода дерева каталога нет: в файле нет ни `rglob`, ни `os.walk`;
единственный `glob` (строки 136-138) ищет `*/manifest.yaml`, то есть манифесты
профилей, а не файлы ингредиента.

Следствие: `ingredients/skills/<id>/references/**` в БД **не попадает вообще**.
В ORM (`catalog/infrastructure/models.py`) у ингредиента есть `body` (Text) и
`metadata_` (JSONB) — колонки под вложения нет. Файл физически не доезжает до
каталога, поэтому и на диск пользователя ему взяться неоткуда.

## Конкретный разрыв на `lite-task-workflow`

Путь `references/work-order-template.md` встречается во всём репозитории **4 раза**,
и ни одного — в коде доставки:

1. `templates/tacticum-lite-base/manifest.yaml:86-87` — объявление `assets:`
2. `templates/tacticum-lite-base/ingredients/skills/lite-task-workflow/SKILL.md:139` —
   тело навыка велит агенту: **read `references/work-order-template.md`**
3-4. `apps/backend/tests/catalog/test_manifest_schemas.py:375,381` — тест парсинга

Файл в репозитории существует:
`templates/tacticum-lite-base/ingredients/skills/lite-task-workflow/references/work-order-template.md`

При установке (`target_path_template: ".claude/skills/{ingredient_id}/SKILL.md"`,
`manifest.yaml:81`) на диск ляжет только `SKILL.md`. Относительная ссылка из тела
разрешится в `.claude/skills/lite-task-workflow/references/work-order-template.md` —
**этого файла у установленного профиля не будет**. Навык на шаге сборки ордера
упрётся в отсутствующий файл.

`install_scope: user` (`manifest.yaml:80`) — навык ставится в домашний каталог,
что разрыв не меняет, но означает: чинить придётся и user-scope путь тоже.

## Тесты: доставки никто не ждёт

Единственный тест про assets — `test_manifest_schemas.py:359-382`
(`test_skill_spec_with_assets_and_scripts_renders`). Он проверяет **парсинг**, не
доставку: `assert parsed.metadata.assets == [...]`. Это регрессия на прошлый баг
(`extra="forbid"` ронял схемно-валидный манифест), а не контракт установки.

E2E-установка (`apps/backend/tests/e2e_install/`) ассертит только `SKILL.md`
(`test_install_flow.py:336,430,523`; `test_oracle_mutations.py:37`). Ни один
golden-фикстур в `tests/e2e_install/golden/*/*.json` не содержит пути `references/`.
Golden-фикстуры для `tacticum-lite-base` нет вообще.

Вывод: механика не «частичная» — её нет ни на одном уровне, и ни один тест её
отсутствие не ловит.

## Смежный риск (не по задаче, но рядом)

Дефолтный путь для `skill_spec` **расходится** между двумя путями рендера:

- `catalog/domain/renderer.py:30` → `.claude/skills/{ingredient_id}.md` (плоский файл)
- `catalog/infrastructure/renderers/claude_code.py:47` → `.claude/skills/{ingredient_id}/SKILL.md`

Для `lite-task-workflow` расхождение замаскировано явным `target_path_template` в
манифесте. Любой навык **без** явного шаблона получит разный путь в зависимости от
того, каким путём его отрендерили. Отдельная проблема, чинится отдельно.

## Что нужно для доставки (объём, не рецепт)

Затронуты все четыре уровня, поэтому это не однострочная правка:
сид (класть файлы в каталог) → хранилище (колонка/таблица под вложения) →
контракт рендера (`RenderedFile` → множество файлов) → клиентский apply.