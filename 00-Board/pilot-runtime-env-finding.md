---
title: 'Рантайм-пилот Фазы 4 — находка по окружению (teststand НЕ собирает KMP)'
type: note
status: resolved
permalink: tacticum/00-board/pilot-runtime-env-finding
tags:
- board
- design-system
- lead-ds
- tz1
- pilot
- finding
---

# Рантайм-пилот (вариант B, teststand-scratch) — НЕ осуществим на teststand

**Кто/когда:** lead-ds, 2026-07-24. **Мандат:** решение президента B (разовое точечное исключение — изолированный scratch на teststand, teardown после). **Итог:** сборка/тесты НЕ запущены — окружение не поддерживает; scratch снесён.

## Что проверено (read-only + свой scratch)
1. Создан изолированный scratch `/home/tacticum/scratch-lead-ds-pilot` (свой путь, не касался kmp-full/kmp-pilot/srv/iva/repos).
2. Source-only копия kmp-build (73M, без build/.git), диск ок (6.4→6.3G).
3. Тулчейн-пробы: Java 21 ✅, gradlew ✅, gradle-кэш tacticum 3.4G (оффлайн-деп есть) ✅, git-remote = локальный мирор `/srv/iva/repos/su.ivcs.messenger` ✅.
4. **Android SDK — ОТСУТСТВУЕТ:** `ANDROID_HOME` пуст; `local.properties`/`sdk.dir` нет; широкий `find /` по platform-tools/build-tools/sdkmanager — пусто.
5. **kmp-full НИКОГДА не собирался на teststand:** `full.done`/full.log — это установка/верификация ПРОФИЛЯ-РОЛИ (25 скиллов + агенты + guard-чек), в логе дословно: **«Коммитов, push, сборки и тестов не запускал»**. «Тулчейн уровня-3» = приёмка установки профиля, НЕ среда компиляции KMP.
6. **Roborazzi/runComposeUiTest у модуля contact-detail — нет** (grep по impl/build.gradle.kts + tasks пусто); модуль Android-flavored (`src/main` только, compileDebugSources).
7. **VLM-паритет** требует Figma-фрейм (отложен президентом/дизайнером).

## Вывод
Рантайм-верификация (компиляция Android-модуля + runComposeUiTest + Roborazzi + VLM) на teststand **невозможна**: нет Android SDK, проект там не компилируется (профиль-стенд, не билд-стенд), тестов Roborazzi у модуля нет, VLM без Figma. Фальшивый «pass» не выдаю — это была бы недостоверность.

## Teardown
`rm -rf /home/tacticum/scratch-lead-ds-pilot` ✅ подтверждено (ls → No such file). kmp-full/kmp-pilot/srv на месте, диск 6.4G восстановлен. Разовое исключение закрыто.

## Что реально нужно для рантайм-пилота (рекомендация президенту через ГД)
- **Настоящая KMP/Android билд-среда:** машина/CI с Android SDK (+ build-tools/platforms) + доступ к внутренним Gradle-деп (git.hi-tech.org — учётки/мирор). Teststand это не даёт.
- ЛИБО прогон на **машине KMP-команды** (Легин) — у них есть рабочая сборка (их `kmp-run-launch`/`kmp-build-verification` скиллы) — через президента.
- ЛИБО принять, что **статическая приёмка (сухой пилот C) уже пройдена** и достаточна для валидации навыка на данном этапе; рантайм отложить до интеграции у KMP-команды при реальном порте.
- Дефект целевого кода (raw Spacer→IvaSpacer, ContactDetailScreen.kt:70-77) и IvaSpacer(size:Dp) как дроп-ин — подтверждены статически, чинятся при реальном порте в билд-среде команды.

## Связано
[[План ТЗ-1 Дизайн-процесс Figma↔код — Сц.4 перенос форм one→kmp (lead-ds)]] · `00-Board/proposal-phase4-pilot-run-model` · `00-Board/pilot-sts.4-contact-detail-sukh-oi-progon-web-to-kmp-screen-port`
