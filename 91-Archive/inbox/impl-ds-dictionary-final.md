---
title: impl-ds-dictionary-final
type: note
permalink: tacticum/00-board/impl-ds-dictionary-final-1
tags:
- board
- design-system
- lead-ds
- tz1
- dictionary
- report
archived-at: 2026-08-03 11:16
---

# Отчёт implementer — финализация словаря Iva*↔веб (Фаза 2, ТЗ#1 Сц.4)

**Статус:** draft. **Кто/когда:** implementer для lead-ds, 2026-07-24. **Autonomy:** off. **ssh:** не понадобился (всё локально).

## Что сделано
Провизорный словарь `Iva*(KMP)↔веб` → **непровизорный (resolved)**. `figma_key` заполнены из авторитетного каталога code-bindings в репе (Figma напрямую не нужна).
- **Источник:** `~/tacticum/tacticum-dev/design-systems/iva-web/tokens.json` → `.["$extensions"]["dev.tacticum.code-bindings"].components` (извлёк через `jq`, python-парсер не трогал). 49 компонентов: **32 непустых figma_key + 17 обоснованных `null`**.
- **Обновил заметку** `00-Board/phase2-provisional-iva-web-dictionary` (permalink стабилен): статус provisional→**resolved**, таблица `Iva* | роль | веб-компонент (name+selector) | figma_key | статус`, две секции пробелов, самопроверка.

## Цифры матчинга (по ролям левой колонки)
- **matched-key (реальный figma_key):** 8 ролей — text_field→Input, buttons→Button, icon_button→Button Icon + Button Icon Round, toggle→Toggle, dialog→Dialog, bottom_sheet→Dialog, circle_photo→Avatar, badge→Badge.
- **matched-null (обоснованный null, name-match, мастер в другом файле multi-file DS):** 3 роли — search_bar→Input Search, drop_down_menu→Menu/Menu Item, dividers→Divider.
- **пробел (Iva* без веб-аналога в каталоге):** 10 ролей — item_container, banner, header_screen, top_bar/nav, spacers, info_panel, chart, push, empty_state, containers.

Реально использованных figma_key из каталога: 9 (Input, Textarea, Button, Button Icon, Button Icon Round, Toggle, Dialog, Avatar, Badge; Dialog переиспользован bottom_sheet).

## Пробелы — обе стороны
**Iva* → пробел (10 групп):** item_container (IvaItem*), banner (IvaBanner*), header_screen (IvaHeaderScreen*), top_bar/nav (IvaMessengerBottomBar/NavigationRail), spacers (IvaSpacer), info_panel (IvaInfoPanel), chart (IvaLineChart/PieChart), push (IvaPushNotification), empty_state (IvaChatDetailEmptyPlaceholder), containers (*RoundedBackground/ColumnFullScreen/ColumnItemsContainerStatic). Веб-аналог/ключ не выдуман.

**Веб-каталог → пробел (без Iva* в текущем инвентаре, для будущего):** Chip/Chip User/Chip Button, Button Ghost/Icon Ghost/Icon Overlay/Icon Round Elevated/Vertical/Navigation, Avatar Group/Edit, Badge Counter/Status, Input Password/Number, Select, Slider, Checkbox, Radio, Checkable List, Form Field/Error/Group, Calendar/Calendar Picker/Time Picker, Tabs, Popover, Toast, Tooltip, Spinner, Progress Circular, Skeleton, Microphone Indicator, Scroll Area, Icon(any), Textarea. Вероятно у части есть Iva*-аналог — доснять при полном инвентаре с сигнатурами.

## Алиасы, которые применил
- Каталожные name-divergence (поле `match`): **Radio↔radiobutton, Tabs↔tab/tabgroup, Scroll Area↔scroll/scrollbar, Select↔dropdown, Toggle↔switch, Input↔textfield, Button Icon↔iconbutton, Toast↔notification/snackbar, Spinner↔loader**.
- Из левой колонки реально сработали: `IvaSwitchAndText`→Toggle (switch), `IvaBottomSheet`→Dialog (bottom-sheet как вариант dialog), `IvaDropdownMenu`→Menu (contextmenu), `IvaCircleButton`→Button Icon Round.
- Важное решение: `IvaPushNotification` НЕ замаплен на Toast, хотя у Toast есть алиас `notification` — семантика разная (системный push ≠ in-app toast). Оставлен в пробелах, чтобы не выдумывать матч.

## Самопроверка (пройдена)
- Все figma_key в таблице сверены с `tokens.json` через jq — не выдуманы.
- Баланс каталога сходится: 32 непустых + 17 null = 49.
- Незамапленное честно в пробел; null там, где каталог даёт null (обоснованный, не TODO) — по принципу президента.

## Что дальше (для lead-ds/ГД)
1. 17 null-ключей каталога дозаполнять через Figma только при реальной нужде (отдельно, требует Figma-доступа) — сейчас не блокер.
2. Полный инвентарь Iva* с сигнатурами закроет пробелы обеих сторон.
3. Промоция в серверные code-bindings (боевой tacticum-mcp) — потом, требует server-write через ГД+президента.

## Границы задачи
Строго в рамках: только чтение локального `tokens.json` + обновление двух заметок на доске. Код в репозитории/worktree не трогал (задача — словарь на доске, не правка кода). Расширения скоупа не было.

## Связано
[[phase2-provisional-iva-web-dictionary]] · `00-Board/prep-ds-phase2-kmp-dictionary` · [[План ТЗ-1 Дизайн-процесс Figma↔код — Сц.4 перенос форм one→kmp (lead-ds)]]