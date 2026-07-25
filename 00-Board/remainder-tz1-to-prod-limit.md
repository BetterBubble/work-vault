---
title: 'КАРТА ОСТАТКА ТЗ#1 — до прод-предела (для президента, ночной план)'
type: note
status: draft
permalink: tacticum/00-board/remainder-tz1-to-prod-limit
tags:
- board
- design-system
- lead-ds
- tz1
- remainder
- prod-readiness
---

# Карта остатка ТЗ#1 (figma-ds) — до полной прод-готовности

**Для:** президента (через ГД, ночной план). **Кто:** lead-ds, 2026-07-24. **Принцип:** строго по ТЗ Солонко, без раздувания сверх. Разметка зоны: 🟢 наша (template, lead-ds) · 🟡 отдельная команда/RE/server · 🔵 ADR/процесс.

## СДЕЛАНО (template-side capability)
| Юнит | Статус |
|---|---|
| **Сц.4** (перенос экрана Angular→KMP): навык web-to-kmp-screen-port + reference + словарь | ✅ в main (#149) |
| **Сц.1/2** ядро (PR-A): angular-ds-component-authoring + -usage (G1/G4/G6) | ✅ в main (#149, iva-web-brownfield 0.3.0) |
| **Сц.2 ш7** (PR-B, G5): ui-mockup-match Figma numeric-mode | 🕐 запушен, в очереди президенту |
| **Ось-1 клиент** (PR-C, G7/G8+iva-core): discovery framework/surface fix + web quickstart + iva-core тонкий скилл | 🕐 готов, держу до мержа PR-B |

## ОСТАТОК до прод-предела ТЗ#1

### 🟢 Наша зона (template) — что ещё МОЖЕМ (по ТЗ, минимально)
- **Сц.3 тонкое правило миграции** (ОПЦИОНАЛЬНО, маленькое): правило «2 слоя батчами (токены→компоненты), не смешивать ДС, нет аналога→Сц.1» поверх уже задеплоенных authoring/usage-навыков. Только вместе с/после PR-C. **Не крупный PR.** Если президент хочет — вкладываем строкой в authoring-навык; если нет — Сц.3 целиком на команду iva-one.
- Всё остальное по Сц.1/2/4/ось-1-клиент — СДЕЛАНО (PR-A/B/C).

### 🟡 Отдельная команда / RE / server (НЕ наш template)
- **Сц.3 сам прогон миграции** iva-one (72 легаси-файла ui-kit→@iva/design-system, 556 уже) = **команда iva-one**. Отчёт покрытия/пин версий = **RE-конвейер**. Наш вклад = максимум тонкое правило (выше).
- **iva-core серверная ДС + словарь code-bindings** (ось-1) = **server/RE/DS-команда** (в репо нет design-systems/iva-core/). Наш template-кусок (тонкий скилл+router) — в PR-C.
- **G2/G3** (словарь v2 доп-поля + авто-пересборка+CI) = **RE-конвейер** (вне веб-профиля, из карты Сц.1/2).
- **17 null figma_key** словаря + подтверждение Figma-фреймов = Figma-доступ/дизайнеры (не блокер, обоснованный null).
- **rn-brownfield/mail** ещё хардкодят iva-web в discovery = **заходы других лидов** (не наш скоуп).

### 🔵 ADR / процесс (решение президента/архитектурное)
- **Канон tacticum-ui-base**: допущение «Iva DS unified across surfaces» → surface-split. Уровень **ADR** (ADR-0056/0059), НЕ косметика скилла. ГД фиксирует президенту отдельным заходом. Пока держится «unified»-долг.
- **_mirrors для DS-навыков**: расширить CI-лок на web-копии (design-system-discovery 7 копий/6 хэшей; ui-mockup-match 5 копий все разные, вне _mirrors) ИЛИ осознанно оставить brownfield-only. Процессное решение.

## ПРОД-READINESS РИСКИ
1. **Mirror-дрейф DS-навыков между стеками** — CI mirror-проверка ловит ТОЛЬКО задекларированные пары; design-system-discovery (7 копий) и ui-mockup-match (5 копий) рассинхронены по стекам, PR-B/PR-C усилили web↔остальные. Риск: правка в одном стеке не доходит до других. Митигация: расширить _mirrors ИЛИ явно признать копии независимыми (ADR).
2. **«Unified»-долг ui-base** блокирует корректный surface-routing на уровне роли (роль тянет discovery через ui-base-копию с «unified»); наш G7-фикс — только в brownfield-копии. До ADR роль не знает про iva-core-поверхность.
3. **Seed `framework_hint: react` при Angular** iva-one (design-systems/iva-web/design-system.yaml) — дрейф seed; наш G7 учит читать framework_hint, но сам seed надо поправить (server/DS-команда).
4. **Coverage-регрессия роли** (урок PR-A): любой НОВЫЙ навык в brownfield → врезка в iva-web-development-base (иначе CI-red). Зафиксировано как обязательный шаг (полный pytest catalog в каждой батарее). PR-C уже с врезкой.
5. **Отложенный server-side** (iva-core ДС/словарь, G2/G3) — capability описывает подход, но рантайм-резолв заработает только когда server/RE заведут. Честно помечено «deferred/do not fake».

## ВЕРДИКТ (рекомендация президенту)
**Template-side прод-предел ТЗ#1 = PR-A(merged)+PR-B(queued)+PR-C(ready)** — покрывают Сц.1/2/4 + ось-1-клиент. За этим:
- Сц.3-прогон = команда iva-one (наш вклад — опц. тонкое правило); iva-core-server/G2/G3 = server/RE; канон-ui-base + _mirrors = ADR/процесс; rn/mail = другие лиды.
- Прод-риски #1/#2 (mirror-дрейф + unified-долг) — реальные, но АРХИТЕКТУРНЫЕ (ADR), не блокируют мерж PR-A/B/C; решаются отдельным заходом когда президент вернётся.
**Рекомендую предел:** довести PR-B→PR-C до мержа (полная батарея каждый), опц. Сц.3-тонкое-правило по слову президента, остальное — вынести как ADR/handoff, не раздувать в наши PR.

## Связано
[[plan-tz-1-dizain-protsess-figma-kod-sts.4-perenos-form-one-kmp-lead-ds]] · `00-Board/map-sc3-and-remainder` · `00-Board/map-existing-vs-gap-sc12` · `00-Board/map-existing-vs-gap-pr-c-axis1`
