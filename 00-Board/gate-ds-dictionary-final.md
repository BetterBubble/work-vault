---
title: gate-ds-dictionary-final
type: note
permalink: tacticum/00-board/gate-ds-dictionary-final
tags:
- board
- design-system
- controller
- gate
- tz1
- dictionary
---

# Гейт controller — финализированный словарь Iva*↔веб↔figma_key (ТЗ#1 Сц.4, Фаза 2)

**Вердикт: PASS.** Кто/когда: controller для lead-ds, 2026-07-24. Read-only, ничего не правил.
**Объект:** `00-Board/phase2-provisional-iva-web-dictionary` (resolved) + `00-Board/impl-ds-dictionary-final`.
**Авторитетный источник (сверял через jq):** `~/tacticum/tacticum-dev/design-systems/iva-web/tokens.json` → `.["$extensions"]["dev.tacticum.code-bindings"].components`.

## 1. Достоверность figma_key — ПРОШЛО
Все 9 использованных в таблице ключей извлёк из каталога независимо (jq) и сверил построчно — **каждый реально принадлежит указанному веб-компоненту**, ни одного выдуманного/перепутанного:
- Input `697be2226851f02957f5ca718d78399af939c4bd`, Textarea `c037c1058919f67040b34adb2beea04e15e39f37`, Button `253d9a2a2a3c4cf00172f09710bffc5d53ce5c1f`, Button Icon `349294e204e617819c8cd7e6a48b1ca1f5ac35c8`, Button Icon Round `9fad33aa078ee6846511b4344f7f93c161c0896c`, Toggle `1f01cf3888ff7c1109d365f890936f47276ca69b`, Dialog `b162c0e58bd501e68f59bff10447a33461ac9a51` (переиспользован bottom_sheet — тот же реальный ключ), Avatar `3a1c2a7d9e0303a009fc54912d9ad3e0093119cb`, Badge `439306e4c1105772365215b315db4f84369bad90`.
- Обратная проверка: в словаре нет ни одного ключа, которого нет в каталоге.
- **Баланс сходится точно:** независимый пересчёт jq — 32 непустых + 17 null = 49. Совпадает с заявленным.
- matched-null (4 роли: Input Search, Menu, Menu Item, Divider) — все реально имеют `figma_key=null` в каталоге, не фиктивный null.

## 2. Корректность матчинга — ПРОШЛО
Все заявленные алиасы реально присутствуют в поле `match` каталога (сверил): Radio←radiobutton, Tabs←tab/tabgroup, Scroll Area←scroll/scrollbar, Select←dropdown, Toggle←switch, Input←textfield, Button Icon←iconbutton, Toast←notification/snackbar, Spinner←loader. Матчи левой колонки (IvaSwitchAndText→Toggle, IvaDropdownMenu→Menu[contextmenu], IvaCircleButton→Button Icon Round) обоснованы по имени/роли/selector. Натянутых матчей нет.

## 3. Принцип президента (не выдумано) — ПРОШЛО
- IvaPushNotification НЕ замаплен на Toast, хотя у Toast есть алиас `notification` — решение корректно: системный push ≠ in-app toast, семантика разная. Честно в пробел, а не натянуто.
- 10 групп Iva* без веб-аналога — в «пробелы», веб-компонент/ключ НЕ выдуман.
- 23 веб-компонента с ключом + null-компоненты без Iva* в инвентаре — вынесены в «для будущего», Iva*-аналог не выдуман. Пересчёт сходится: 32 ключа − 9 использованных = 23 неиспользованных; 17 null = 4 замапленных + 13 отложенных.

## 4. Полнота/честность — ПРОШЛО
Секции пробелов обеих сторон присутствуют. «Что дальше» отражено (17 null через Figma только при нужде; полный инвентарь с сигнатурами; промоция в серверные code-bindings — отложена осознанно через ГД+президента). Самопроверка в обеих заметках присутствует и подтверждается независимой сверкой.

## Замечания (не блокеры, для сведения дизайнерам)
- **IvaBottomSheet→Dialog** — семантическое суждение: в каталоге у Dialog нет алиаса `bottomsheet` (match=`dialog,modal`). Решение прозрачно помечено как «bottom-sheet = вариант dialog» и переиспользует реальный ключ Dialog, фабрикации нет. Стоит подтвердить у владельца ДС при доснятии сигнатур — не блокер.
- Пробелы веб-стороны — норма для этой фазы (левая колонка из провизорного инвентаря без полных сигнатур); доснимутся отдельным шагом.

## Скоуп/чистота
Задача словарная (данные-заметки), не код-коммит — git-гейт неприменим. Код/worktree не трогались (подтверждено текстом отчёта и характером изменений — только 2 заметки на доске). Разрастания скоупа нет.

## Итог
Гейт «ключи реальны + матчи честны + сверх-каталога не выдумано» — **пройден**. Замечание по IvaBottomSheet→Dialog передать дизайнерам при доснятии полного инвентаря. Готово к OK президента (через ГД).

## Связано
[[phase2-provisional-iva-web-dictionary]] · [[impl-ds-dictionary-final]] · [[План ТЗ-1 Дизайн-процесс Figma↔код — Сц.4 перенос форм one→kmp (lead-ds)]]
