---
title: impl-g8-web-quickstart
type: report
permalink: tacticum/00-board/impl-g8-web-quickstart-1
status: draft
role: implementer
for: lead-ds
task: ТЗ#1 figma-ds, PR-C, gap G8 — новый web figma-mapping quickstart (аналог KMP,
  которого для web НЕ было)
repo: worktree ~/tacticum/tacticum-dev-web-axis1 (ветка feat/ds-web-axis1)
date: 2026-07-24
tags:
- figma-ds
- impl
- g8
- pr-c
- axis1
- lead-ds
- ds-web
archived-at: 2026-08-03 11:16
---

# impl-g8-web-quickstart — web figma-mapping quickstart (G8)

## Что сделано

Заавторен ОДИН новый файл (строго по границам задачи):

`docs/user_manuals/iva-web-figma-mapping-quickstart.md`

- Веб-аналог `iva-kmp-figma-mapping-quickstart.md` — та же форма, 10 секций, шаги 0–4.
- Язык — русский (образец KMP и все `*-profile-quickstart.md` в этом каталоге на русском; SKILL.md — на английском, но это manual, не skill).
- git: файл **untracked, коммита НЕТ**. Консолидацию (если понадобится индекс/ссылки) делает лид.

## Границы (важно)

`git status --porcelain` показывает ещё `M design-system-discovery/SKILL.md` и `?? iva-core-design-system/` — **это НЕ моё**, они лежали в worktree до меня (PR-C G7-работа). Я их не трогал. Моё — ровно один untracked файл `docs/user_manuals/iva-web-figma-mapping-quickstart.md`.

## Структура секций (зеркало KMP-образца)

1. Заголовок + intro + статус «пилот» (словарь iva-web: `$extensions."dev.tacticum.code-bindings"` с 0.2.0; `figma_key` 32/49 + name-алиасы с 0.3.0).
2. **Что понадобится (общее)** — таблица: профиль `iva-web-brownfield` (установка → основной quickstart), `installation_id` из `.tacticum/context.yaml`, Figma desktop, endpoint `http://127.0.0.1:3845/mcp`. Плашка про запущенный Figma + квоту ~200/день.
3. **Шаг 0** — включить MCP-сервер в Figma (пользователь).
4. **Шаг 1** — обновить профиль (агент): `check_updates` → `pull_installation_content update=true` по `installation_id`.
5. **Шаг 2** — подключить Figma MCP (пользователь): `figma-desktop`, проверка тулов → `get_metadata`, `get_code_connect_map`, `get_screenshot`, `get_variable_defs`, `get_design_context`.
6. **Шаг 3** — разовое правило в `AGENTS.md` (пользователь): предпроверка фрейма → `get_metadata` (fileKey+node-id) → резолв (Code Connect → словарь по `figma_key`/имени, нормализация lowercase+strip spaces/dashes/underscores + `match[]`-алиасы) → чтение `.mdx`/`source` перед use, сборка по `selector` (директива vs элемент) → именованные токены → инстанс не в словаре = СТОП по элементу.
7. **Шаг 4** — финальная проверка связки (агент): `design_get_tokens(design_system_id="iva-web", installation_id=…)` → 49 компонентов; ветки ошибок (`installation_id_required`, нет `$extensions`, `AuthError`).
8. **Как пользоваться** — ТЗ + ссылка на фрейм (fileKey+node-id), краткий поток навыка `angular-ds-component-usage`, критерии успеха/провала, ссылка на числовую приёмку `ui-mockup-match` (Figma numeric-compare).
9. **Если что-то пошло не так** — таблица симптомов (нет тулов figma-desktop, квота, `installation_id_required`, нет `$extensions`, null `figma_key`, «не замаплен», игнор словаря).

## Ссылки на навыки (по ТЗ, не дублирую тела)

- `angular-ds-component-usage` — сборка экрана (Шаг 3, «Как пользоваться»).
- `angular-ds-component-authoring` — если компонента нет в словаре.
- `ui-mockup-match` — числовая сверка, вкл. Figma numeric-compare режим.
- `design-token-usage` — именованные токены, ноль hex.
- `tacticum-context` — источник `installation_id` в `.tacticum/context.yaml`.

## Что сверено с реальностью (tokens.json + skills)

Источник: `~/tacticum/tacticum-dev/design-systems/iva-web/tokens.json` → `$extensions."dev.tacticum.code-bindings"` (читал через python json, не выдумывал).

- **49 компонентов** (`components[]`, len=49). Из них **32 с `figma_key`**, **17 с null** (штатно: компонент в другом файле многофайловой ДС → резолв по имени) — отражено в тексте и в таблице ошибок.
- Реальные поля биндинга: `name`, `match[]`, `figma_key`, `selector`, `kind`, `source`, `storybook`, `inputs{}` (+ опц. `notes`). Выдуманных полей нет.
- `usage`-правило нормализации взято дословно из словаря: lowercase + strip spaces/dashes/underscores; матч по `name` и `match[]`.
- Реальные name-алиасы (примеры в тексте): Radio↔radiobutton, Select↔dropdown, Tabs↔tab/tabgroup, Scroll Area↔scroll/scrollbar — все присутствуют в `match[]` (проверил).
- `figma_file_key` = `AG11paSthGC7zSoovfjip0` (использован в примере ссылки на макет).
- `design_get_tokens(design_system_id="iva-web", installation_id=…)` — сигнатура и обязательность `installation_id` сверены со скиллами `angular-ds-component-usage`, `ui-mockup-match`, `tacticum-context` (team `phk_*` без него → `installation_id_required`). `installation_id` проставлен во ВСЕХ `design_*`-упоминаниях.
- Figma MCP тулы (`get_metadata`, `get_code_connect_map`, `get_screenshot`, `get_variable_defs`, `get_design_context`) — из реальных web-скиллов, не из KMP-набора.
- design-system.yaml iva-web: версия 0.3.0, статус published — использовано для «пилот» и версий словаря.

## Самопроверка

- Создан ТОЛЬКО один файл (мой untracked = один). OK
- Коммита нет; чужие изменения в worktree (G7 SKILL.md, iva-core skill) НЕ тронуты. OK
- `installation_id` — во всех `design_*`-вызовах. OK
- Ноль выдуманного API/полей; всё сверено с tokens.json. OK
- Сверх-ТЗ не расширял (принцип президента). OK

## Заметки лиду

- Индексация файла (ссылка из основного `iva-web-brownfield-profile-quickstart.md` или README каталога `docs/user_manuals/`) — вне моего файла, за лидом (в KMP-образце такой обратной ссылки тоже нет).
- Пилотный словарь помечен «49 компонентов» — если later REST-export дольёт `figma_key` до 49/49, формулировку «32 из 49» в статусе можно уточнить.