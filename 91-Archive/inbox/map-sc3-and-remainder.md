---
title: map-sc3-and-remainder
type: report
permalink: tacticum/00-board/map-sc3-and-remainder-1
status: draft
role: explorer
for: lead-ds
repo: ~/tacticum/tacticum-dev (read-only)
tz: scratchpad/ds-scan/figma-ds-scenario-3-migration.md + figma-ds-process-tz.md
context: 00-Board/map-existing-vs-gap-sc12, 00-Board/map-existing-vs-gap-pr-c-axis1
date: 2026-07-24
tags:
- figma-ds
- explore
- sc3
- migration
- gap-map
- axis1
- remainder
- lead-ds
archived-at: 2026-08-03 11:16
---

# Карта Сц.3 (миграция) + остаток ось-1 + прод-риски ТЗ#1

Вход для КАРТЫ ОСТАТКА ТЗ#1 лида президенту. Read-only, символьно/грепом по `~/tacticum/tacticum-dev`. Строго по ТЗ Солонко, без раздувания.

## 0. Что уже приземлено (база на 2026-07-24)
- **PR-A ЗАДЕПЛОЕН** (iva-web-brownfield CHANGELOG `[0.3.0]`): новые скиллы `angular-ds-component-authoring` (Сц.1 G1) + `angular-ds-component-usage` (Сц.2 G4+G6). Brownfield-only, по одной физ.копии — дрейфа пока нет.
- **PR-B** (ui-mockup-match → Figma-числа, G5) — правил тело `iva-web-brownfield/ui-mockup-match/SKILL.md` (по overlap-заметке PR-C; ветка `feat/ds-web-mockup-figma`).
- **PR-C (ось-1) НЕ приземлён** — web-копия `design-system-discovery` всё ещё хардкодит `iva-web` (стр.43), ui-base всё ещё `unified across surfaces` (стр.46-47,74-75). Это план, не факт.

---

## 1. Сценарий 3 — что требует ТЗ (пунктами)
Источник: `figma-ds-scenario-3-migration.md`. Живой кейс — **iva-one: миграция с замороженного `ui-kit` → новая `@iva/design-system`** (снимок 2026-07-24: 72 файла ещё с легаси-импортами, ~29 использований `ds-*`, 556 уже на `iva-*`, 43 сырых hex, 2 scss на ui-kit).
1. **Дизайнер** — публикует новую ДС + **переходную таблицу** (старое→новое: `ds-button` синий → `Button appearance=primary`; `#1457D9` → `bg/accent-primary`; удалён без аналога → Сц.1).
2. **Конвейер (RE/сервер)** — импорт токенов как **новая версия** (старая остаётся, версии параллельны/иммутабельны, пин не двигается сам — ADR-0009); сборка словаря новой ДС; **отчёт покрытия** (grep-механика: подсчёт легаси-импортов/селекторов/hex/scss → остаток в штуках).
3. **Разработчик + агент** — миграция **в 2 слоя батчами**: (a) слой токенов — механически, почти весь на агенте, hex→токены по таблице; (b) слой компонентов — экран за раз, старый→новый из словаря, каждый батч через **проверку Сц.2** (числовое сравнение/визревью); нет аналога → Сц.1.
4. **Администратор** — переводит пин на новую версию, когда готово (осознанно). Легаси удаляется **отдельной задачей** по нулю в отчёте (72→0).

**Исполнитель по ТЗ:** «Разработчик + агент» команды репозитория (iva-one), опираясь на переходную таблицу (дизайн), отчёт покрытия (конвейер RE) и словарь (Сц.1). То есть **исполнение миграции = команда iva-one**, а template-side/наша зона даёт им только правила и опоры (см. п.3).

## 2. Сц.3 EXISTING (что уже есть в шаблонах/репо)
- **Навыка миграции/codemod/coverage — НЕТ нигде.** `find templates -path '*/skills/*'` по `migrat|coverage|codemod|legacy|transition|ui-kit` → только go-миграции БД (`sqlc-goose-migrations`), `angular-legacy-web-context` (rn), `legacy-mfe-adapter` (iva-2-shell) — **не про ДС-миграцию**. Отдельного «migration/coverage-report» скилла нет.
- **Отчёт покрытия / переходная таблица** — как артефакт в репо **отсутствуют**. По ТЗ снимок iva-one посчитан **вручную grep-ом** (не тулингом). `scripts/` = `check_mirror_sync / check_profile_version_discipline / governance_gate / tei_local / wiki_sync_manuals` — coverage-тулинга нет.
- **Косвенная опора существует:** `angular-ds-component-authoring/SKILL.md` (стр.148) содержит гардрейл «не расширять **frozen `ui-kit`**» — то есть анти-легаси-правило уже вшито в авторинг. `design-system-discovery` (стр.52) и `design-token-usage` (стр.54,74) уже **различают поверхности** `libs/design-system` (iva-one) vs `@ivcs/ui-kit` (iva-connect) — частичная осведомлённость о легаси, но БЕЗ логики миграции.
- **Проверка батча = Сц.2** — уже есть (`angular-ds-component-usage` + `ui-mockup-match`), Сц.3 переиспользует.
- **Задеплоено ли что-то именно под Сц.3 (#132/#134 или иное):** НЕТ. #132/#134 — это словарь/скиллы Сц.1/2 (PR-A). Под Сц.3 в шаблонах **не задеплоено ничего**.

## 3. Сц.3 GAP + ЧЬЯ ЗОНА — ВЕРДИКТ
Реальный gap делится на три адресата:

**A. Template-side (НАША зона lead-ds) — тонкий, если вообще нужен сейчас:**
- Опционально — **скилл/правило «ds-migration»** для web-агента: два слоя (сначала токены, потом компоненты), «не смешивать старую/новую ДС в одном экране», «нет аналога → Сц.1», «легаси удалять отдельной задачей по нулю». Это правила процесса, тонкая надстройка над уже существующими `design-token-usage` + `angular-ds-component-usage` + проверкой Сц.2. **Не обязательно новый скилл** — может быть секцией/router-note.
- Зависит от ось-1 (PR-C): агент должен корректно определять целевую ДС/поверхность — иначе миграция «в новую ДС» не на что опереть.

**B. Отдельная команда/заход (НЕ наша зона):**
- **Сама миграция репо iva-one (72→0 файлов)** = работа **команды iva-one** (разработчик+агент батчами). Это не template-профиль, а прогон по конкретному репозиторию. Не наш PR.
- **Переходная таблица** = артефакт **дизайна** (Figma). Внешняя зависимость.

**C. Server-side / RE-конвейер (ОТЛОЖЕНО, DS-команда):**
- **Отчёт покрытия как тулинг** (регуляризация grep-снимка), **импорт новой версии токенов**, **параллельные иммутабельные версии + пин**, **сборка словаря новой ДС** — всё это конвейер RE/сервер, не шаблоны.

**ИТОГОВЫЙ ВЕРДИКТ:** Сц.3 — **преимущественно НЕ template-side**. Наша зона = максимум тонкое правило миграции (2 слоя + гардрейлы) поверх уже задеплоенных Сц.1/2-навыков, и то только **после/вместе с PR-C** (ось-1: без выбора целевой ДС мигрировать некуда). Основной вес Сц.3 — это (B) прогон командой iva-one + (C) серверный конвейер отчёта покрытия/версий. **Рекомендация: в карту остатка президенту Сц.3 ставить как "исполнение — отдельная команда/RE, template-вклад минимальный/опциональный", НЕ как крупный наш PR.**

## 4. Остаток ось-1 ПОСЛЕ PR-C (что ещё)
PR-C (template-side ось-1) закрывает: web-копию `design-system-discovery` (хардкод `iva-web`→выбор по platform/framework_hint+поверхность), ui-base «unified»→surface-split, router-note iva-core, G8 web-quickstart, тонкий скилл iva-core. **После PR-C остаётся:**
- **iva-core серверная ДС + словарь code-bindings = server/RE — ПОДТВЕРЖДЕНО отложить.** В репо НЕТ `design-systems/iva-core/` (есть только iva-mobile/iva-rn/iva-web/tacticum-web), НЕТ iva-core seed, НЕТ токенов VCSWEB. Требует реальных Figma-токенов + extraction словаря в RE. Не блокирует пилот iva-one.
- **Канон `tacticum-ui-base` «unified»→surface-split = уровень ADR.** «Iva DS unified across surfaces» (стр.46-47,74-75) — заявленный принцип композиции (ADR-0056/0059). Замена на surface-routing (iva-core = отдельная ДС на конференц-поверхности) — семантическое изменение, **тянет ADR/решение ответственной роли**, не косметика скилла. (Правку тела скилла делает PR-C, но легитимация принципа — ADR.)
- **_mirrors для копий iva-core:** новый скилл iva-core, если его тиражировать на >1 профиль (web + conference/iva-connect), потребует пары в `_mirrors.yaml` + CI-лок. Сейчас `_mirrors.yaml` держит только `design-system-discovery` (одна пара analysis-base↔fr-analyst). Решение о новой паре — процесс, отдельно.
- **Другие поверхности с легаси-хардкодом (вне ось-1-web-цель):** `iva-rn-brownfield` и `iva-brownfield-mail` копии `design-system-discovery` всё ещё хардкодят `iva-web` — отдельные заходы соответствующих лидов, не наш PR-C.

## 5. ПРОД-READINESS РИСКИ (весь ТЗ#1)
Свежие факты `git hash-object` на 2026-07-24:
- **Дрейф `design-system-discovery`: 7 физ.копий, 6 уникальных хэшей.** Байт-идентична только пара `iva-analysis-base`==`iva-fr-analyst` (5fabaced, под CI-локом `_mirrors.yaml`). Остальные 5 — все разные (web-brownfield e77e3ab, ui-base 16729cf, kmp 6495764, rn 3ee0f08, mail cf2223e). Web↔ui-base дрейф ЛЕГАЛЕН (не в паре), но = риск рассинхрона при правке ось-1. Массовый фикс rn/mail отложен.
- **Дрейф `ui-mockup-match`: 5 копий, ВСЕ 5 хэшей разные** (ui-base 3d7b6bc, web-brownfield 58094ce, kmp 67dc791, rn 8e8a825, mail 9080dd8). PR-B тронул только web-копию → усилил дрейф с остальными 4. `ui-mockup-match` в `_mirrors.yaml` НЕ задекларирован → CI не запирает. Риск: Figma-числовой режим есть только в web, остальные стеки отстают.
- **Новые PR-A навыки** (`angular-ds-component-authoring/usage`) — brownfield-only, по 1 копии, в `_mirrors.yaml` нет → дрейфа пока нет, но если понадобятся в role-web через композицию (ui-base/development-base) — решить owner заранее.
- **«unified»-долг в ui-base** (см. п.4) — прод-риск ось-1: пока принцип «unified across surfaces» жив, агент на конференц-поверхности возьмёт неверную ДС (iva-web вместо iva-core). Блокер корректного surface-routing.
- **Отложенный server-side:** словари/серверная ДС iva-core, авто-пересборка словаря (G2/G3 Сц.1), регуляризация gap/coverage-отчётов, импорт версий+пин Сц.3 — всё в RE-конвейере, вне шаблонов. Пилот iva-one не блокируют, но без них Сц.1/Сц.3 остаются полу-ручными (снимки grep-ом, словарь v0.3.0 без kind/host/requires/slots/import/category/mdx_path).
- **Дрейф seed-метаданных:** `design-systems/iva-web/design-system.yaml` — `platform: web` + `framework_hint: react` при том что iva-one Angular (стр.17-18). Это seed серверной ДС (server-side фикс с DS-командой); скилл лишь должен корректно ЧИТАТЬ. Риск неверного framework-резолва.
- **CI/тесты:** `check_mirror_sync.py` + `test_role_replacement_parity.py` проверяют ТОЛЬКО задекларированные пары байт-в-байт. DS-навыки вне пар (ui-mockup-match, web design-system-discovery, angular-ds-*) под CI-лок НЕ попадают → рассинхрон между стеками CI не поймает. Автотеста на «coverage-отчёт/миграцию» нет (тулинга нет).

## Пути (якоря)
- Сц.3 ТЗ: `scratchpad/ds-scan/figma-ds-scenario-3-migration.md`
- Нет навыка миграции: `~/tacticum/tacticum-dev/templates/*/ingredients/skills/` (поиск migrat/coverage/codemod → пусто по ДС)
- Косвенная опора: `templates/iva-web-brownfield/ingredients/skills/angular-ds-component-authoring/SKILL.md:148` (frozen ui-kit); `.../design-token-usage/SKILL.md:54,74`; `.../design-system-discovery/SKILL.md:52`
- PR-A задеплой: `templates/iva-web-brownfield/CHANGELOG.md` `[0.3.0]`
- ось-1 остаток: `templates/iva-web-brownfield/ingredients/skills/design-system-discovery/SKILL.md:43` (хардкод); `templates/tacticum-ui-base/.../design-system-discovery/SKILL.md:46-47,74-75` (unified)
- seed дрейф: `design-systems/iva-web/design-system.yaml:17-18`
- iva-core отсутствует: `design-systems/` (нет iva-core); `templates/` (нет скилла iva-core)
- mirror-лок: `templates/_mirrors.yaml:20`; `scripts/check_mirror_sync.py`