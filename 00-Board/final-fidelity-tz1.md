---
title: 'Финальная сверка полноты ТЗ#1 (figma-ds) vs доставленный контент'
type: note
permalink: tacticum/00-board/final-fidelity-tz1
status: current
updated: '2026-07-25'
tags:
- final-fidelity
- tz1
- figma-ds
- prod-readiness
---

# Финальная сверка ТЗ#1 (Figma↔код, Солонко) vs main @ 8f8287c + PR-C (feat/ds-web-axis1)

**Ревизор:** независимый critic-агент (read-only). **Источник истины:** `90-Materials/figma-ds-process.zip` (6 док + notes).

## ВЕРДИКТ: template-предел ДОСТИГНУТ ПО ТЗ = ДА, с ОДНОЙ обязательной правкой перед прод-мержем.

Все 4 сценария + ось-1 (web) + ось-2 (reference) доставлены как capability template-профиля; отложенное чисто ложится на роли ТЗ (Дизайнер/Конвейер/Администратор/чужие репо), НЕ на «Разработчик+агент». Сверх-ТЗ: НЕ найдено (0 выдуманных скоуп-ограничений). НО: один реальный coherence-дефект в `angular-ds-component-usage` (главный навык Сц.2).

## Покрытие по сценариям/осям

| Сценарий / ось | ТЗ требует | Доставлено | Статус |
|---|---|---|---|
| **Сц.1** библиотека | навык авторинга по конвенциям репо (анатомия, директива/элемент, IvaControlBase+CVA, слоты, токены-only, полнота вариантов, Storybook+.mdx+spec, приёмка) | `angular-ds-component-authoring` (main) — покрыто целиком | **ПОЛНО (template)** |
| **Сц.2** новый экран | pre-flight макета, резолв по словарю, стоп-на-незамапленном, сборка по selector, .mdx-first, числовая приёмка | `angular-ds-component-usage` (main) + `ui-mockup-match` Figma numeric-mode G5 (main) + quickstart G8 (PR-C) | **ПОЛНО (template), но coherence-дефект** ↓ |
| **Сц.3** миграция | тонкое правило-дисциплина (2 слоя, не смешивать ДС, нет аналога→Сц.1, удаление легаси отдельно) | `angular-ds-component-authoring §Migration` (PR-C) | **ПОЛНО (template-часть)** |
| **Сц.4** Angular→KMP | навык-оркестратор (чтение источника, Angular→Compose табл., маппинг состояния, гардрейлы, 3-way parity, верификация) + reference-скилл двух деревьев | `web-to-kmp-screen-port` + `web-to-kmp-source-reference` (main, iva-kmp-development-base) | **ПОЛНО (template)** |
| **Ось-1** несколько ДС | discovery читает platform/framework_hint, surface-router iva-one↔iva-core, тонкий iva-core скилл — **шаблон + выровнять KMP** | web: `design-system-discovery` фикс + `iva-core-design-system` скилл (PR-C). KMP: НЕ выровнен | **ЧАСТИЧНО: web ПОЛНО, KMP отложен** ↓ |
| **Ось-2** два репо | п.3 reference-скилл (read-only source); п.1/2/4 start-task-arg+workflow+гейт | reference-скилл (main); п.1/2/4 → lead-modes лейн | **ПОЛНО (template-часть), остальное handoff** |

## Отложенное — проверка «чужой скоуп vs наш»

Все явно-отложенные пункты ложатся на роли, которые ТЗ Солонко САМ отделяет от «Разработчик+ИИ-агент» (таблица участников: Дизайнер / Конвейер / Администратор):
- **Сц.3-прогон iva-one** → команда iva-one (template даёт только правило; прогон конкретного репо = его команда). Легитимно.
- **серверная ДС iva-core + словарь** → server/RE (ТЗ multirepo: «завести отдельную серверную ДС» = инфра-конвейер). Легитимно.
- **словарь v2 (G2) + авто-пересборка+CI (G3)** → RE-конвейер (ТЗ: «Конвейер собирает словарь / проверяет»). Легитимно.
- **17 null figma_key + Figma-фреймы** → дизайнеры/владелец ДС + Figma-доступ (обоснованный null, матч по имени работает). Легитимно.
- **словарь KMP (Iva*↔web)** — RESOLVED на доске (32 ключа + 17 null), ждёт repo-native доставки в `AI common/skills/` KMP-репо. Легитимно (KMP-команда).

ИТОГ по отложенному: всё вне зоны design-process template-профиля ПО ТЗ. Ни один отложенный пункт ТЗ не требовал от нас как от template-стороны.

## РЕАЛЬНЫЕ ПРОБЕЛЫ (ТЗ требовал, не доставлено/не согласовано)

1. **[ОБЯЗАТЕЛЬНО ДО ПРОДА] `angular-ds-component-usage` — устаревшие ссылки на G5.**
   В main (после PR-B, 0.4.0) навык Сц.2 в трёх местах (depends-on табл. стр.37, Step 7 стр.~159, anti-patterns стр.176) утверждает, что числовая Figma-приёмка — «separate, not-yet-shipped mode (PR-B / gap G5)». Но G5 УЖЕ в main (0.4.0), и PR-C починил sibling `angular-ds-component-authoring`, а `usage` НЕ тронул. Дефект в ОБЕИХ копиях (iva-web-brownfield И iva-web-development-base). Последствие: агент, читая главный навык Сц.2, НЕ вызовет доступный числовой matcher и не выполнит приёмку ш.7 ТЗ Сц.2, считая её недоступной. Это регресс против уже доставленной способности, не косметика. Фикс: обновить те же 3 места, как в authoring («ui-mockup-match Figma numeric-compare mode», без «not-yet-shipped»).

## Отложено-с-раскрытием, но ТЗ называл явно (не блокер, но читатель должен знать)

2. **KMP `design-system-discovery` НЕ выровнен.** Ось-1 change #1 ТЗ дословно: «design-system-discovery (шаблон + live, **и выровнять KMP**)». Web-копия получила framework_hint/surface-router (PR-C), а KMP-копия сохраняет допущение «Iva design system unified across surfaces», не читает platform/framework_hint, без surface-router на iva-core. Раскрыто в `handoff-tz1-deferred-remainder §5` (канон ui-base + _mirrors 7 копий/6 хэшей → ADR-0056/0059). Т.е. ось-1 в силе только для web; на KMP-поверхности разведение ДС не работает до ADR-канона. Легитимная отложка, но ось-1 = web-only по факту.

## Сверх-ТЗ: НЕ найдено
Навыки верны ТЗ, без выдуманных скоуп-ограничений. Level-1/3 taxonomy, fix-parity/greenfield routing, 3-way parity reframe (ui-mockup-match = web-side lock + tokens + Roborazzi/VLM) — всё заземлено на репо-навыки KMP и точнее/честнее исходного ТЗ, не сужает design-process.

## Мелочи (не блокеры)
- `web-to-kmp-screen-port` имеет висячие «§TODO» кросс-ссылки (§1.6/§5 → «see §TODO») и самопометку раздела «skeleton, refined but not final» + словарь ждёт repo-delivery. Приемлемо для пилота (сухой прогон пройден, рантайм отложен), но перед мержем в KMP-репо якоря надо доразрешить.

## Остаток до прода
Мерж PR-C (после устранения пробела #1 заодно, т.к. usage — тот же лейн web) + прод-сид версий ДС (внешний gated-шаг). Контент годится как фундамент; критичен только фикс #1.
