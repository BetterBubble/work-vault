---
title: 'Фаза 2 — словарь Iva*(KMP)↔веб-мастер-компонент (RESOLVED по 32 ключам code-bindings)'
type: note
status: resolved
permalink: tacticum/00-board/phase2-provisional-iva-web-dictionary
tags:
- board
- design-system
- lead-ds
- tz1
- dictionary
- resolved
---

# Словарь `Iva*` (KMP) ↔ веб-мастер-компонент — RESOLVED

**Статус:** resolved (по 32 ключам). **Кто/когда:** implementer для lead-ds, 2026-07-24.
**Источник figma_key (авторитетный, read-only):** `~/tacticum/tacticum-dev/design-systems/iva-web/tokens.json` → `.["$extensions"]["dev.tacticum.code-bindings"].components` — 49 веб-компонентов, у каждого `name/selector/match/figma_key` (32 непустых ключа + 17 обоснованных `null`). Figma напрямую НЕ требовалась — авторитетный каталог уже в репе.
**Левая колонка (Iva*)** — из провизорной таблицы (shared `iva-m/android/kmp` `core/design-system`, commonMain, per решение ГД).

Провизорность снята: `figma_key`=TODO заменены на реальные ключи из каталога либо на обоснованный `null` (name-match на компонент, чей мастер лежит в другом файле multi-file DS — это НЕ TODO, а осознанный null по замыслу каталога). Незамапленное вынесено в «пробелы» — веб-аналог/ключ НЕ выдуман (принцип президента).

## Матчинг — методика
Нормализация имени/роли/`selector` + алиасы `match` из каталога (lowercase, без пробелов/дефисов/подчёркиваний — поле `usage`). Применённые name-divergence алиасы каталога: **Radio↔Radiobutton, Tabs↔Tab Group, Scroll Area↔Scroll, Select↔Dropdown**. Из левой колонки реально применены: bottom-sheet→Dialog, dropdown-menu→Menu/Menu Item, circle-button→Button Icon Round, switch→Toggle.

## Таблица словаря
| Iva* (composable) | роль (element/) | веб-компонент (name + selector) | figma_key | статус |
|---|---|---|---|---|
| IvaTextField, IvaTextFieldWithIcons, IvaTextFieldWithCounts | text_field | Input · `input[ivaInput]` (многострочный → Textarea · `textarea[ivaTextarea]`) | `697be2226851f02957f5ca718d78399af939c4bd` (Textarea: `c037c1058919f67040b34adb2beea04e15e39f37`) | matched-key |
| IvaSearchTextField, IvaSearchBar | search_bar | Input Search · `iva-input-search` | `null` — name-match, компонент в другом файле multi-file DS | matched-null |
| (текстовые/основные кнопки) | buttons | Button · `button[ivaButton], a[ivaButton]` | `253d9a2a2a3c4cf00172f09710bffc5d53ce5c1f` | matched-key |
| IvaIconButton | icon_button | Button Icon · `button[ivaButtonIcon], a[ivaButtonIcon]` | `349294e204e617819c8cd7e6a48b1ca1f5ac35c8` | matched-key |
| IvaCircleButton | icon_button | Button Icon Round · `button[ivaButtonIconRound], a[ivaButtonIconRound]` | `9fad33aa078ee6846511b4344f7f93c161c0896c` | matched-key |
| IvaSwitchAndText | toggle | Toggle · `iva-toggle` (алиас switch) | `1f01cf3888ff7c1109d365f890936f47276ca69b` | matched-key |
| IvaAlertDialog, IvaConfirmationDialog, IvaWarningDialog, IvaErrorInfoDialog, IvaBaseAlertDialog, IvaActionWithTextDialog, IvaAddLinkDialog | dialog | Dialog · `iva-dialog` (алиас modal) | `b162c0e58bd501e68f59bff10447a33461ac9a51` | matched-key |
| IvaBottomSheet | bottom_sheet | Dialog · `iva-dialog` (bottom-sheet = вариант dialog) | `b162c0e58bd501e68f59bff10447a33461ac9a51` | matched-key |
| IvaAvatar, IvaAvatarInitials | circle_photo | Avatar · `iva-avatar` | `3a1c2a7d9e0303a009fc54912d9ad3e0093119cb` | matched-key |
| IvaBadge | badge | Badge · `iva-badge` | `439306e4c1105772365215b315db4f84369bad90` | matched-key |
| IvaDropdownMenu, IvaDropdownMenuShell | drop_down_menu | Menu · `iva-menu` (алиас contextmenu) | `null` — name-match, компонент в другом файле multi-file DS | matched-null |
| IvaDropDownMenuItem | drop_down_menu | Menu Item · `button[ivaMenuItem], a[ivaMenuItem]` | `null` — name-match, компонент в другом файле multi-file DS | matched-null |
| IvaHorizontalDivider, IvaBottomShadowHorizontalDivider | dividers | Divider · `iva-divider` (алиас separator) | `null` — name-match, компонент в другом файле multi-file DS | matched-null |

**Итог по левой колонке:** 8 ролей matched-key (реальный ключ), 3 роли matched-null (обоснованный null), 10 ролей — пробел (ниже).

## Пробелы — Iva* без веб-аналога в каталоге
Веб-компонент/ключ НЕ выдуман. Это честный вход для дизайнеров/владельца ДС.
| Iva* | роль | почему пробел |
|---|---|---|
| IvaItemInput, IvaItemSwitch, IvaItemInfo, IvaItemRemovable, IvaItemDescription, IvaItemContainer | item_container | в каталоге нет list-item/cell мастер-компонента |
| IvaBannerCard, IvaBannerHost, IvaBannerIconView | banner | в каталоге нет Banner |
| IvaHeaderScreenWidget, IvaHeaderScreenStartTitleRow, IvaHeaderScreenCenteredTitleRow, IvaHeaderAttentionWidget | header_screen | в каталоге нет Header/App-bar |
| IvaMessengerBottomBar, IvaMessengerNavigationRail | top_bar / navigation_bottom_bar | в каталоге нет navbar/tabbar (есть лишь одиночный Button Navigation — это кнопка, не бар) |
| IvaSpacer | spacers | Compose-лэйаут/spacing-токен, не мастер-компонент |
| IvaInfoPanel | info_panel | в каталоге нет Info-panel/Notice |
| IvaLineChart, IvaPieChart | chart | в каталоге нет Chart |
| IvaPushNotification | push | веб Toast = in-app notification, НЕ системный push — не маплю (разная семантика) |
| IvaChatDetailEmptyPlaceholder | empty_state | в каталоге нет Empty-state |
| IvaBoxRoundedBackground, IvaColumnRoundedBackground, IvaRowRoundedBackground, IvaColumnFullScreen, IvaColumnItemsContainerStatic | containers | Compose-лэйаут-контейнеры, не мастер-компонент |

## Пробелы — веб-компоненты каталога без Iva* в текущем инвентаре (для будущего)
Эти веб-мастер-компоненты НЕ имеют Iva*-строки в провизорной таблице. Вероятно, соответствующие `Iva*` есть в KMP DS, но не сняты в текущий инвентарь — доснять при полном инвентаре с сигнатурами (привязку сейчас НЕ выдумываю).
- **С ключом (не использованы левой колонкой):** Chip, Chip User, Chip Button, Button Ghost, Button Icon Ghost, Button Icon Overlay, Button Icon Round Elevated, Button Vertical, Avatar Edit, Badge Counter, Badge Status, Input Number, Slider, Checkbox, Radio, Calendar, Tabs, Toast, Tooltip, Spinner, Skeleton, Microphone Indicator, Scroll Area. (Textarea частично покрыт text_field.)
- **С null (из 17 в каталоге):** Button Navigation, Avatar Group, Input Password, Select, Checkable List, Form Field, Form Error, Form Group, Calendar Picker, Time Picker, Popover, Progress Circular, Icon (any) — плюс уже замапленные null (Input Search, Menu, Menu Item, Divider).

## Что дальше
1. **17 null-ключей** каталога — дозаполнять через Figma ТОЛЬКО если реально нужно (отдельно, требует Figma-доступа: `design_get_tokens("iva-web")` из профиля iva-web-brownfield либо REST). Сейчас обоснованный null (мастер компонента в другом файле multi-file DS) — не блокер и не TODO.
2. **Доснять полный инвентарь Iva* с сигнатурами** (параметры/слоты) — закрыть пробелы обеих сторон (item_container/banner/header/nav ↔ Checkbox/Radio/Slider/Tabs/Chip/Calendar и т.д.).
3. **Промоция в серверные code-bindings** (боевой tacticum-mcp) — потом, требует server-write через ГД+президента. Локальный `tokens.json` остаётся авторитетным зеркалом.

## Самопроверка
- Все `figma_key` в таблице реально извлечены из `tokens.json` (jq), не выдуманы.
- Число ключей сходится с каталогом: 32 непустых / 17 null (49 всего).
- Незамапленное — честно в пробел; null там, где каталог даёт null.

## Связано
[[План ТЗ-1 Дизайн-процесс Figma↔код — Сц.4 перенос форм one→kmp (lead-ds)]] · `00-Board/prep-ds-phase2-kmp-dictionary` · `00-Board/impl-ds-dictionary-final`
