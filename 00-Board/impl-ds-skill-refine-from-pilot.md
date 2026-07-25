---
title: impl-ds-skill-refine-from-pilot
type: note
permalink: tacticum/00-board/impl-ds-skill-refine-from-pilot
status: draft
tags:
- board
- design-system
- lead-ds
- tz1
- web-to-kmp
- implementer
- skill-refine
---

# impl — уточнение навыка `web-to-kmp-screen-port` по фидбэку первого пилота (ContactDetail)

**Кто/когда:** implementer для lead-ds, 2026-07-24. **Worktree:** `/Users/bubblemac/tacticum/tacticum-dev-ds-web-to-kmp` (ветка `feat/ds-web-to-kmp`). **Коммит:** `65e4fbf` (поверх `d715669`). НЕ мержено/не пушено (autonomy off).

**Вход:** grounded-фидбэк «§5 Фидбэк на навык» из `00-Board/pilot-sts.4-contact-detail-sukh-oi-progon-web-to-kmp-screen-port` (реальные пробелы первого прогона, не гипотезы). Правлены только F-1/F-2/F-3/F-5. F-4 (словарь `figma_key`) не трогался — Figma-пауза у президента.

## Изменённые файлы (git diff --stat)

```
 templates/iva-kmp-development-base/CHANGELOG.md    | 26 +++++++++++++++++
 .../skills/web-to-kmp-screen-port/SKILL.md         | 33 +++++++++++++++++++++-
 templates/iva-kmp-development-base/manifest.yaml   |  2 +-
 3 files changed, 59 insertions(+), 2 deletions(-)
```

Не трогались: `iva-role-kmp`, зеркало `iva-kmp-brownfield`, шаренный `brownfield-task-workflow`, `web-to-kmp-source-reference`. Ссылки на in-repo навыки в SKILL.md сохранены.

## Что именно уточнено (F → секция)

- **F-1 → §1 «Read the source».** Добавлен явный абзац «Pull the full sample, not just `*.component.ts`»: на экран нужны `*.component.ts` + `*.component.html` (шаблон: состав/`@if`/`@for`/DS-теги → шаги 6/7) + `data-access` (REST-контракт → шаг 8) + shell/route-файл (какой store провайдится → шаг 2). Явно сказано: только с `.ts` — Transloco (§1.7) и REST (§1.8) неизвлекаемы (строки в шаблоне, контракт в data-access).
- **F-2 → §7 leg 1 (Component level).** Общая формулировка «lists are `LazyColumn` with key» смягчена до «lists are keyed» + подпункт «Keyed how — `LazyColumn` vs `Column`»: `LazyColumn`+key только для ПОТЕНЦИАЛЬНО ДЛИННЫХ/безграничных (верхний скролл); КОРОТКИЕ ОГРАНИЧЕННЫЕ ВЛОЖЕННЫЕ (2–3 email/phone в карточке) — keyed `forEach`/`key()` в `Column` (нельзя `LazyColumn` в `LazyColumn`/scrollable). Один внешний `LazyColumn` на экран, keyed `Column` внутри.
- **F-3 → §0 (rewrite-port vs move-port).** Добавлен блок «Fix-parity vs greenfield — route the effort first»: различает МЕЛКИЙ UI-фикс (сырой `Spacer`/`Color`/`dp` → `Iva*`/токен, локально, без скоуп-флага) vs СТРУКТУРНУЮ правку (новый view-state/VO/поле маппера — трогает State+Component+mapper по bottom-up §0, с явной оценкой скоупа). Явно помечено «это маршрутизация, не новый запрет; и fix-parity, и greenfield в scope» — в рамках ТЗ Сц.4 «либо создать, либо починить».
- **F-5 → §7 leg 1 (Component level).** Добавлен подпункт «Read the widgets, not just `*Screen.kt`»: статприёмка (собран из `Iva*` / состояние поднято / списки keyed) требует читать part-компоненты/виджеты (`*Widget.kt`, `*PartComponent`), т.к. сырьё (`Color`/`dp`/списки) чаще в них, а `*Screen.kt` — часто stateless-шелл.

**Bump:** `iva-kmp-development-base` 0.5.0 → 0.6.0 + CHANGELOG `[0.6.0]` (перечень F-1/2/3/5, F-4 явно не трогается). Доктрина/структура навыка не менялись — только точечные добавления в существующие секции §0/§1/§7.

## Валидаторы (venv `apps/backend/.venv/bin/python`, все зелёные)

```
version-discipline (static):     OK — 46 profile(s) clean.  exit=0
version-discipline (--diff HEAD): OK — 46 profile(s) clean.  exit=0
mirror-sync:                     OK — 62 зеркальных ингредиентов в 6 парах синхронны.  exit=0
manifest schema (pytest, 38 tests): все прошли.  exit=0
```

## Самопроверка

- SKILL.md читается, структура (§0–§9 + TODO) цела, ссылки на in-repo навыки не тронуты.
- Каждая правка прослеживается до конкретного пункта фидбэка (F-1/2/3/5); сверх-ТЗ ничего не добавлено — F-3 сформулирован как маршрутизация, НЕ как новый запрет/скоуп (принцип президента соблюдён).
- F-4 не трогался (Figma-пауза). Зеркало/роль/шаренный workflow не тронуты — mirror-sync подтверждает байтовую синхронность.
- Не мержено, не пушено.

## Связано
`00-Board/pilot-sts.4-contact-detail-sukh-oi-progon-web-to-kmp-screen-port` · `00-Board/phase2-provisional-iva-web-dictionary`