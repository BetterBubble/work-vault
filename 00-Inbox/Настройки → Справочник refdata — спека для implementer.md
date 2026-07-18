---
title: Настройки → Справочник refdata — спека для implementer
type: note
permalink: tacticum/00-inbox/nastroiki-spravochnik-refdata-speka-dlia-implementer
tags:
- helm
- settings
- refdata
- spravochnik
- implementer
- spec
---

Фича «Настройки → Справочник»: раздел на сайте для удобного ведения справочных сущностей. Подтверждено руководителем. Ветка `feature/settings-refdata` от `main`, worktree `~/tacticum-worktrees/helm-settings`. Репо `/Users/bubblemac/tacticum/helm`. Только добавления, существующее не ломать.

## Скоуп справочников (вкладки)
Продукты · Поколения · Блоки (+ЕОЛ-владелец) · Команды · Цели. «Роли» = вариант (а): ЕОЛ-владельцы, правятся полем `eol_email` в формах блоков/продуктов — ОТДЕЛЬНОЙ таблицы ролей НЕ делать.

## Backend
- **Переиспользовать существующий CRUD** (`routers/inputs.py`, prefix `/api`): `goals` (POST/GET/PUT/DELETE), `blocks` (POST/GET/PUT/DELETE — есть block_id/name/eol_email/product/generation/cross), `teams` (POST/GET/PUT). Аннотации — паттерн аудита.
- **ДОБАВИТЬ CRUD:**
  - **Продукты** (`/api/products`, таблица `product`): GET list / POST / PUT / DELETE. Поля: name (PK, канон), active; владелец продукта (eol_email/owner) — из `product_owners_ref`, добавь поле если удобно.
  - **Поколения** (`/api/generations`, таблица `generation`): GET/POST/PUT/DELETE. Поля: name (PK), active.
- **Персистентность (важно):** сид refdata (`repository._upsert_refdata`, SEED_PRODUCTS/SEED_GENERATIONS) НЕ должен затирать правки оператора. Реши чисто: fill-only-missing ИЛИ флаг `operator_edited` ИЛИ CRUD-таблицы авторитетны и сид только доливает недостающее. Правки должны переживать пере-сид.
- **Аудит:** каждая правка справочника → запись в `annotation` (kind=`refdata_edit`, что/почему), как уже делается для тем.
- **Auth-гейт:** редактирование справочника — под full-access/оператором (`a.shulga` уже в allowlist). Навесь зависимость (full_access или отдельная operator-роль); read можно шире.

## Frontend
- **Новый пункт верхнего меню «⚙ Настройки»** в `App.tsx` (показывать только если у принципала full-access/оператор; сверься с `me.data_classes`/roles).
- **Экран `web/src/screens/Settings.tsx`** с под-вкладками: Продукты · Поколения · Блоки · Команды · Цели. Каждая вкладка = таблица (список из `/api/<entity>`) + **инлайн добавить/править/удалить** (удобно: правка по клику, сохранить/отмена, удаление с подтверждением). Блоки-вкладка включает ЕОЛ (eol_email) + product/generation/cross.
- `api.ts`: клиентские вызовы products/generations CRUD + переиспользовать goals/blocks/teams. Типы в `types.ts`.

## Приёмка
- CRUD работает: создать/править/удалить персистит и **переживает пере-сид**. UI удобный (инлайн). Auth-гейт. `uv run pytest -q` зелёно, ruff+mypy+web tsc чисто.
- Коммить в `feature/settings-refdata`, НЕ пушить.

## Отчёт (текстом to:main, НЕ idle)
Что добавил (эндпоинты/экран), как решил персистентность-vs-сид, точку auth-гейта, **число passed pytest**, заминки.
