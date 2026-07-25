---
title: PR-C create-link + тело PR (ось-1 + lane-agnostic, для ГД→президента)
type: note
permalink: tacticum/00-board/pr-c-create-link-telo-pr-os-1-lane-agnostic-dlia-gd-prezidenta
status: draft
tags:
- board
- design-system
- lead-ds
- tz1
- pr-c
- push-done
---

# PR-C — доставлено на origin, готово к PR

**Ветка:** `feat/ds-web-axis1` (origin @ `2d919de`, подтверждено fetch: remote = local HEAD). **Пушнул:** lead-ds, 2026-07-25. **База:** `main`. **Мерж:** президент (очередь, утро).

## CREATE-LINK (создать PR вручную через веб)
https://github.com/TacticumApps/tacticum-dev/compare/main...feat/ds-web-axis1?expand=1

## Коммиты (6, все на origin, 0 AI-подписей)
```
2d919de authoring: lane-agnostic формулировка Figma numeric-compare mode (обе копии)
789dbde usage: lane-agnostic формулировка Figma numeric-compare mode (обе копии)
290f448 fix(ds-web): usage — ui-mockup-match Figma mode shipped (not future), unblocks Сц.2 step 7
ab4a693 fix(ds-web): authoring — ui-mockup-match shipped + Сц.3 batch=screen + transition-table source
54bcc6f feat(ds-web): thin Scenario-3 migration rule in angular-ds-component-authoring
a07f504 fix(ds-web): quickstart G8 — source not .mdx/slots + surface scope note
eb70dcb feat(ds-web): iva-core skill + design-system-discovery ось-1 fix + web quickstart + role coverage
```
Дельта vs main: **12 файлов, +489 / −43.**

---

## ЗАГОЛОВОК PR
`Дизайн-процесс Figma↔код (web): ось-1 несколько ДС по поверхности + приёмка макета lane-agnostic`

## ОПИСАНИЕ PR (готово к вставке, без атрибуции)

**Что сделано**

Веб-часть дизайн-процесса Figma↔код: поддержка нескольких дизайн-систем по поверхности продукта (ось-1) и корректная приёмка экрана по макету в двух конфигурациях профиля.

- **Ось-1 (несколько ДС по поверхности).** `design-system-discovery` перестал жёстко предполагать одну веб-ДС: читает platform/framework_hint профиля и маршрутизирует по поверхности (iva-one → `@iva/design-system`, конференц-поверхность → `iva-core`). Добавлен тонкий навык `iva-core-design-system` (конвенции пакета iva-core; рантайм-резолв по словарю включится, когда серверная ДС iva-core заведена — до тех пор fallback на demo-app).
- **Приёмка макета — точная в обоих профилях.** Навыки `angular-ds-component-authoring` и `angular-ds-component-usage` описывают приёмку через `ui-mockup-match` условно: числовое сравнение с Figma-фреймом (ΔE/размеры/соответствие токенам), когда профиль этот режим предоставляет; иначе — HTML-режим / дизайн-ревью. Формулировка верна и для профиля с co-located `ui-mockup-match`, и для профиля, где UI-приёмка берётся из общей базы (HTML-режим). Навыки только ссылаются на приёмочный инструмент, не реализуют матчер.
- **Сценарий 3 (миграция).** В `angular-ds-component-authoring` добавлено тонкое правило миграции: два слоя батчами (токены по transition-таблице → компоненты по словарю), батч = экран, не смешивать ДС в одном батче, нет аналога → сначала наполнение библиотеки, удаление легаси — отдельным шагом.
- **Документация.** Quickstart по маппингу Figma↔web (`docs/user_manuals/iva-web-figma-mapping-quickstart.md`): поля словаря code-bindings, резолв по selector, приёмка.
- **Покрытие роли.** Навыки продублированы в базовый веб-профиль (byte-identical), чтобы веб-роль покрывала заменяемый профиль — тест покрытия зелёный.

**Что тестировалось**

- Валидаторы каталога: синхронизация зеркал (64 ингредиента / 6 пар) — зелёно; дисциплина версий профилей (48 профилей, в т.ч. `--diff-against origin/main`) — зелёно.
- Тесты каталога: покрытие роли заменяемым профилем — passed; паритет замены ролей (84) — passed; контент-тесты — passed. Не-DB набор — 290 passed (100%); падения только у тестов, требующих поднятого Postgres/Docker (инфраструктура окружения, не контент).
- Байт-идентичность обеих копий каждого навыка (usage / authoring / iva-core) — подтверждена (`cmp` IDENTICAL).
- Достоверность словаря: все ключи/поля сверены с реальным `design-systems/iva-web/tokens.json`.

**Вне скоупа (отдельными заходами)**

Серверная ДС iva-core + её словарь code-bindings (server/RE); авто-пересборка словаря + CI-проверка; дозаполнение 17 null-ключей словаря (дизайнеры/Figma-доступ); фактический прогон миграции iva-one (команда iva-one); канон общей UI-базы (унификация приёмки по поверхностям) — на уровне ADR.

## Связано
`00-board/tz1-build-complete-summary` · `00-board/handoff-tz1-deferred-remainder` · [[plan-tz-1-dizain-protsess-figma-kod-sts.4-perenos-form-one-kmp-lead-ds]]