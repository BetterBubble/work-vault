---
title: impl-g4-usage
type: report
permalink: tacticum/00-board/impl-g4-usage-1
status: draft
role: implementer
for: lead-ds
repo: ~/tacticum/tacticum-dev-web-sc12 (worktree, branch feat/ds-web-sc12)
task: ТЗ#1 figma-ds Сц.2 — новый навык angular-ds-component-usage (gap G4 + вложенный
  G6)
date: 2026-07-24
tags:
- figma-ds
- impl
- sc2
- g4
- g6
- lead-ds
archived-at: 2026-08-03 11:16
---

# impl-g4-usage — навык angular-ds-component-usage (Сц.2, gap G4+G6)

## Что сделано

Заавторен ОДИН новый файл (по границам задачи):

`templates/iva-web-brownfield/ingredients/skills/angular-ds-component-usage/SKILL.md`

- git: файл untracked, коммита НЕТ (консолидацию manifest/CHANGELOG делает лид).
- Ничего кроме этого файла не тронуто (`git status --porcelain` = только новый каталог).
- Язык — английский (весь корпус SKILL.md на английском: design-system-discovery, ui-mockup-match, design-token-usage, ivcs-libs-contract).

## Frontmatter

- `name: angular-ds-component-usage`
- `description:` (с триггерами) — предлагаемый **metadata.description_trigger**:
  > Use during the implementation phase when building a new (or reworked) screen from a Figma frame — assembling it out of ready IVA design-system components by resolving each instance through the code-bindings dictionary, never by hand-drawing markup. Triggers on "build screen from Figma", "assemble from design system", "новый экран по макету", "собрать экран из компонентов", "реализовать макет Figma", "Figma mockup to Angular", "ivaButton", "iva-form-field", "component usage", "node-id".

## Содержание по секциям (строго ТЗ Сц.2 шаги 1-7)

- **Depends on / hands off to** — таблица ссылок на 4 навыка (см. ниже).
- **Tools** — Figma MCP `get_metadata` / `get_code_connect_map` / `get_screenshot`, `design_get_tokens`; блок дисциплины квоты ~200/день + «не копировать React+Tailwind как есть».
- **Step 1 — Pre-flight the frame (G6)** — ВКЛЮЧЁН как первый дешёвый гейт: auto layout? инстансы распознаются? иначе СТОП → дизайнеру с конкретикой (node-id / detached copy). Явно: не генерить свалку позиционированных блоков; fail = design-side fix.
- **Step 2 — Read structure** — `get_metadata` на фрейм → точечные вызовы по под-узлам, беречь квоту.
- **Step 3 — Resolve instance→component** — Code Connect first (`get_code_connect_map`); иначе словарь `design_get_tokens("iva-web")` → `$extensions."dev.tacticum.code-bindings"`; предпочесть `figma_key`, fallback на имя; нормализация по `usage`-правилу (lowercase, strip spaces/dashes) + `match[]`-алиасы (Radio↔Radiobutton и т.д.). Перечислены реальные поля биндинга (name/match/figma_key/selector/kind/source/storybook/inputs).
- **Step 4 — Not in dictionary → STOP по элементу** — не выдумывать разметку; роут либо в Сц.1/`angular-ds-component-authoring`, либо возврат дизайнеру; стоп per-element (что можешь — собрать, незамапленное — в блокеры).
- **Step 5 — Assemble** — читать `.mdx`/`source` компонента ПЕРЕД использованием; правила web-композиции по `selector`: директива `button[ivaButton]` → `<button ivaButton>` vs элемент `iva-*`; поля в `iva-form-field`; меню через `[ivaMenuTriggerFor]`; биндить только объявленные inputs.
- **Step 6 — Tokens** — делегировано `design-token-usage`, только именованные, ноль hex/px, логику резолва НЕ дублирую.
- **Step 7 — Acceptance** — ноль самодельной разметки (формальный критерий); `get_screenshot` только как reference; числовая сверка (pixel-diff+ΔE+size-deltas+допуск) = Figma-режим `ui-mockup-match`, **PR-B / G5, НЕ РЕАЛИЗОВАН тут — только ССЫЛКА**.
- **Anti-patterns / guardrails** — не галлюцинировать компоненты/пропсы (только реальная поверхность словаря), незнакомое → стоп; ноль raw div+hex; не паста React/Tailwind; не тянуть страницы целиком; «no pixel matcher here» (G5/PR-B).

## Как включён G6

G6 (предпроверка макета) вложен как **Step 1** — самый первый гейт до любого чтения/резолва, с явным СТОП→дизайнеру. Отдельного навыка под G6 не создавал (по указанию — внутрь G4).

## Навыки-ссылки (не дублирую)

- `design-system-discovery` — design-фаза, откуда приходит DS id.
- `design-token-usage` — резолв токенов, шаг 6 полностью делегирован.
- `angular-ds-component-authoring` — парный G1, хэндофф шага 4 (инстанс не в словаре → авторинг/Сц.1).
- `ui-mockup-match` — шаг 7; текущий HTML-режим; числовой Figma-режим помечен как отдельный PR-B (G5), не реализован здесь.

## Самопроверка

- SKILL.md читается, frontmatter `name` + `description` присутствуют. OK
- Создан ТОЛЬКО один файл (git: единственный untracked каталог). OK
- Числовой matcher НЕ реализован — только ссылка на PR-B/G5 (шаг 7 + анти-паттерн). OK
- Коммита нет; manifest.yaml/CHANGELOG.md не тронуты. OK
- Сверх-ТЗ правил не добавлял (принцип президента). OK

## Заметки лиду (не расширял объём — фиксирую как факт)

- Словарь v1 (G2) пока БЕЗ полей `mdx_path/slots` — поэтому в шаге 5 путь к `.mdx` описан как «рядом с компонентом, из `source`/`storybook` биндинга», а не из отдельного `mdx_path`. Когда G2 нальёт поля — можно уточнить формулировку.
- Размещение/owner (brownfield-only vs _mirrors) и подключение в композицию role-web — вне моего файла, за лидом (см. риски gap-карты).