---
title: 'Prep: Фаза 2 — словарь Iva*↔веб-компонент (ТЗ#1 Сц.4)'
type: note
status: draft
permalink: tacticum/00-board/prep-ds-phase2-kmp-dictionary
tags:
- board
- design-system
- lead-ds
- tz1
- prep
---

# Prep Фазы 2 — словарь `Iva*` (KMP) ↔ веб-мастер-компонент

**Кто:** lead-ds. **Когда:** 2026-07-24 (пауза Шага 0, доступ к kmp-репо добивается ГД→президент). **Зачем:** чтобы на разблокировке репо стартовать Фазу 2 мгновенно — весь дизайн ниже готов, implementer'у остаётся исполнить по коду.

## Установленный факт (из спека Сц.4)
Мобильной библиотеки мастер-компонентов в Figma **нет** (файл IVA Mobile DS пустой). Мобильные экраны рисуются на **веб-компонентах** (UI KIT, 81 сет, ключи 107 шт. выгружены). Мобильная Figma несёт только токены-переопределения. → Словарь мапит `Iva*`-composable из **кода KMP** на **веб-мастер-компонент** (не на несуществующий мобильный).

## Что это за артефакт
Четвёртая привязка в цепочке: Figma-макет + веб-компонент + веб-реализация iva-one → указывают на один веб-мастер-компонент; добавляем привязку к **KMP-коду** (`Iva*`-composable). Резолв токенов — рантайм через `tacticum-mcp` (`design_resolve_token`), отдельного yaml-словаря токенов не плодим.

## РЕШЕНО (ГД 2026-07-24 16:17): источник инвентаря `Iva*`
Авторитетный набор `Iva*` для словаря = **shared Kotlin-MP модуль `iva-m/android/kmp` на adp** (`/srv/iva/repos/kmp/core/design-system`, **49 composable, commonMain, текущий**). Целевое приложение su.ivcs.messenger **потребляет** shared kmp → словарь маппим на shared (49). Старую teststand-копию (41, release26Q1) как истину НЕ использовать; teststand НЕ рефрешить (серверы read-only). Реальный вопрос структуры app (если не решается по коду) → контакт Легина через ГД.

## Вход (что читать, read-only)
1. **Инвентарь `Iva*`-composable** — с adp `kmp/core/design-system` (commonMain, 49): имена, сигнатуры (параметры/слоты), KDoc, `@Preview`. Это левая колонка словаря.
2. **Ключи веб-UI-KIT** (107 шт., уже выгружены — свериться где лежат) — правая колонка.
3. `APP_FEATURE_MAP.md` + parity-заметки (`*_WEB_DESKTOP.md`, `PARITY_*.md`) — контекст соответствия экранов (для Шага 0 и проверки покрытия).

## Алгоритм матчинга (черновик, implementer уточнит по коду)
1. **По имени (первичный):** нормализовать имена (`IvaFormField`→`iva-form-field`, `IvaInput`→`ivaInput/iva-input`, `IvaScrollArea`→`iva-scroll-area`) → кандидат-матч к веб-мастер-компоненту по kebab-имени.
2. **По роли/семантике (вторичный):** где имя не совпадает — матч по роли (input/field/scroll-area/button/list-item) + сверка слотов/параметров.
3. **Уточнение ключом:** привязать к конкретному ключу веб-UI-KIT (стабильный figma_key), не к имени, чтобы переживать переименования.
4. **Незамапленный `Iva*` → в список пробелов** (не выдумывать веб-аналог; это вход для дизайнеров/владельца).

## ХОД Фазы 2 (2026-07-24, read-only adp)
- **Левая колонка (Iva*) — СОБРАНА.** Инвентарь shared `kmp/core/design-system` (commonMain): ~49–65 `Iva*`-composable (65 `fun Iva*` включая *Preview/*UiState-варианты → реальных компонентов меньше). Таксономия по `element/` (39 ролей): buttons, icon_button, text_field(s), search_bar, dialog, bottom_sheet, item_container, containers, header_screen, top_bar, navigation_bottom_bar, dividers, badge, banner, avatar/circle_photo, chart, drop_down_menu, toggle, info_panel, empty_state, toast, file, gallery, emoji, keyboard, selection_text, network_status, spacers, modifier, lazy, dialer, chat_settings, profile_menu, push, content_info, cursor_context_menu, animated_icon, preview. Примеры: IvaTextField/IvaTextFieldWithIcons/IvaSearchTextField, IvaItemInput/IvaItemSwitch/IvaItemInfo/IvaItemRemovable, IvaIconButton/IvaCircleButton, IvaHeaderScreenWidget, IvaBottomSheet, IvaDropdownMenu, IvaAvatar/IvaAvatarInitials, IvaBadge, IvaSwitchAndText…
- **⚠️ Правая колонка (107 веб-мастер-ключей) — БЛОКЕР (Figma-доступ).** Per `figma-component-mapping-pilot.md`: ключи ещё НЕ выгружены, требуют Figma REST-токен (`GET /v1/files/:key/components`) → заполнить `figma_key` (это 0.3.0 mapping). Figma-доступ ГД ОТЛОЖИЛ (нужен Figma MCP/дизайнер, отдельный доступ — как и Figma-фрейм экрана). → Полный словарь с `figma_key` сейчас НЕ собрать. Максимум **провизорный матч по имени/роли** (шаги 1-2 алгоритма), привязка ключом (шаг 3) — после Figma-доступа.
- **Развилка → ГД:** строить провизорный name/role-словарь сейчас (левая колонка + предполагаемый веб-компонент по роли, `figma_key`=TODO) ИЛИ придержать Фазу 2 до Figma-доступа (избежать churn при пере-раскладке по ключам). Тот же Figma-доступ, что уже отложен.
- **РЕШЕНИЕ ГД (16:32): (а) GO — строим провизорный словарь сейчас**; Figma-доступ проверить ДО объявления блока.
- **Figma-доступ ПРОВЕРЕН (2026-07-24, все 4 пути) — из этой сессии НЕ достаётся:** ☁️ облачный claude_ai_Figma MCP (мой аккаунт) → нет edit-доступа к UI-KIT `AG11paSthGC7zSoovfjip0` (bulk-компоненты требуют editor); 🖥️ локальный Figma desktop MCP `127.0.0.1:3845` → не запущен (HTTP 000); 🔌 tacticum-mcp `design_get_tokens("iva-web")` (готовый code-bindings) → нет в сессии (installation `7c5854f6`, профиль iva-web-brownfield); 🔑 REST-токен → значения нет в материалах. → gap эскалирован ГД. Опции разблокировки: (A) дать мне значение Figma read-токена → REST `GET /v1/files/AG11.../components`; (B) прогнать `design_get_tokens("iva-web")` из профиля iva-web-brownfield (там installation) — отдаёт словарь code-bindings напрямую; (C) запустить Figma desktop+DevMode на машине с агентом (порт 3845); (D) дать моему Figma-аккаунту library-доступ к UI-KIT. Рекомендация: **B или A**.

## Формат вывода (по конвенции репо)
Markdown-таблица **в теле навыка** (как в `design-system-discovery`/`ui-mockup-match`) — отдельного yaml/json-словаря навыки не заводят. Колонки: `Iva*-composable | роль | веб-мастер-компонент | figma_key | статус (matched/gap)`. Плюс секция «пробелы» (незамапленные).

## Приёмка Фазы 2
- Каждый `Iva*` из инвентаря либо замаплен (с figma_key), либо явно в «пробелах».
- Ноль выдуманных веб-компонентов (сверка ключей реальны).
- Таблица встроена в `web-to-kmp-screen-port` SKILL.md (заменяет TODO-заглушку словаря), version bump пакета, гейт controller.

## Смежное (капability без пилот-репо, но требует координации ГД)
Спек Сц.4 §«Дополнить существующие навыки» просит доработать `design-system-discovery`/`design-token-usage` (Figma-мост) и `ui-mockup-match` (трёхсторонний паритет Compose↔Figma↔веб). **Эти навыки — owner `tacticum-ui-base` (шаренный базовый пакет, от него зависят и веб-профили).** Правка = пересечение скоупов (потенциально lead-fr/веб). **Гейт-правило:** прежде чем править `tacticum-ui-base` — сигналить ГД. Пока НЕ трогаю; подниму вопрос скоупа при разблокировке репо, вместе с запросом доступа (не плодить сигналы сейчас).

## Связано
[[plan-tz-1-dizain-protsess-figma-kod-sts.4-perenos-form-one-kmp-lead-ds]] · `00-Board/impl-ds-web-to-kmp-skill-skeleton` · `00-Board/gate-ds-web-to-kmp-phase1`
