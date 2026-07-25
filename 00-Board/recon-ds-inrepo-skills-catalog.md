---
title: 'Recon: реальные in-repo навыки KMP (каталог для пере-связки web-to-kmp-screen-port)'
type: note
status: draft
permalink: tacticum/00-board/recon-ds-inrepo-skills-catalog
tags:
- board
- design-system
- lead-ds
- tz1
- recon
---

# Каталог реальных in-repo навыков (для пере-связки скелета)

**Источник:** `adp:/srv/iva/repos/kmp/AI common/skills/` (shared Kotlin-MP модуль `iva-m/android/kmp`, read-only, снято 2026-07-24). Это ЖИВОЙ канон целевого проекта su.ivcs.messenger (author r.yarullin). Наш `web-to-kmp-screen-port` = **оркестратор поверх них**, не дублирует. Ссылаться по имени навыка.

## Навыки, на которые ссылается оркестратор
| Навык (in-repo) | О чём / как использует наш оркестратор |
|---|---|
| **android-to-kmp-porting-su-ivcs-messenger-kmp** | Оркестратор переноса **Android→KMP = ПЕРЕМЕЩЕНИЕ** в commonMain + expect/actual, порядок model→domain→data→infra→feature, transformation-каталог (Dispatchers→DispatcherProvider, @Parcelize→@Serializable, Dagger→Factory, Retrofit→Ktor…). ⚠️ НАШ случай ДРУГОЙ: Angular→Compose = **ПЕРЕПИСЫВАНИЕ**. Ссылаемся как на родственный оркестратор (структура triage-дерева + бottom-up порядок + идея transformation-каталога), но явно оговариваем отличие move-vs-rewrite. |
| **decompose-su-ivcs-messenger-kmp** | Навигация/компоненты (Decompose 3.2.2): `BaseComponent.Render()`, `*RootComponent`/`*ScreenComponent`/`*PartComponent`, `childStack`, Parent↔Child через Bridge, `UIEvent`/`Action`/`News`, ручной Factory-DI, весь навиг-код в commonMain. → канон целевой структуры компонента экрана. |
| **mvi-state-machine-su-ivcs-messenger-kmp** | Состояние на MVIKotlin: Store (Reducer+Executor), `State`/`Intent`/`Msg`/`Label`, Harel sealed-interface режима экрана (без «мешков boolean»), one-shot effects через Label. Level 1/2/3. → сюда мапится Angular signalStore. (Спек-заметка «StateFlow default / MVIKotlin если уже» — это Level-1 vs Level-3; ссылаемся на реальный навык вместо общего описания.) |
| **compose-ui-patterns-su-ivcs-messenger-kmp** | Compose UI-канон: Screen = композиция `*PartComponent`-ов (эталон `ChatListScreen.kt`), stability (@Immutable, immutable-collections, key в LazyColumn), ресурсы через compose-resources (не R.*), Material3 jetbrains, диалоги через DialogManager, Iva DS-элементы. |
| **compose-multiplatform-ui** | Верхнеуровневый вход в Compose MP UI (IvaTheme, AppColors, Decompose wiring) → маршрутизирует в compose-ui-patterns. |
| **design-system-discovery** | Фаза дизайна: перечисляет ДС воркспейса и тянет их токены, чтобы UI ссылался на реальные токены, а не хардкод. |
| **design-token-usage** | Фаза реализации: резолв пути токена → значение / полный набор темы (light/dark), привязка через AppColors/IvaTheme. |
| **ui-mockup-match** | ⚠️ Паритет через **playwright + DOM-diff** — это WEB-runtime vs MOCKUPS. Для нашей ВЕБ-стороны (образец iva-one) годится; для Compose-паритета — не прямой (см. kmp-ui-testing/Roborazzi). Трёхсторонний паритет = ui-mockup-match(веб) + токены + Roborazzi/VLM(Compose). |
| **kmp-ui-testing** | Тесты KMP: `runComposeUiTest`, Roborazzi (скриншоты), Turbine, Store-тесты, Mokkery/fakes → критерий приёмки экрана. |
| **clean-architecture-su-ivcs-messenger-kmp** | Достигнутый канон core/model+domain: UseCase, VO (`@JvmInline value class` для ID), rich-агрегаты, invariants, маппер-санитайзер → правила приземления слоёв. |
| **iva-web-ecosystem** | Контекст веб-экосистемы ИВА (сторона источника iva-one). |
| **kmp-foundations-su-ivcs-messenger-kmp** | Базовые основы KMP-проекта (source sets, точки входа, DI-граф) — companion. |

## Что это меняет в скелете (для implementer)
1. **Секция «ссылки»** SKILL.md: заменить прежние (compose-multiplatform-ui / design-system-discovery / design-token-usage / ui-mockup-match — они и in-repo есть, ок) + ДОБАВИТЬ реальные `decompose-…`, `mvi-state-machine-…`, `compose-ui-patterns-…`, `clean-architecture-…`, `kmp-ui-testing`, `iva-web-ecosystem`, `android-to-kmp-porting-…`.
2. **Принцип «rewrite, не move»**: теперь можно ЯВНО противопоставить реальному `android-to-kmp-porting-su-ivcs-messenger-kmp` (он про move Android→KMP; наш — rewrite Angular→Compose). Раньше обходили (навыка якобы нет) — теперь ссылаемся с оговоркой отличия.
3. **Маппинг состояния**: signalStore → цель по канону `decompose-…` (компонент) + `mvi-state-machine-…` (Store, если Level-3) / `MutableStateFlow+onXxx` (Level-1). Ссылаться на реальные навыки, не описывать общо.
4. **Паритет**: развести — `ui-mockup-match` (playwright/DOM, веб-образец) vs `kmp-ui-testing`/Roborazzi (Compose-скриншот). Трёхсторонний = веб + токены + Compose-скриншот/VLM.
5. **Iva*-истина** — с adp `kmp` (49, commonMain), НЕ старый пилот 41.

## Связано
[[plan-tz-1-dizain-protsess-figma-kod-sts.4-perenos-form-one-kmp-lead-ds]] · `00-Board/impl-ds-web-to-kmp-skill-skeleton` · `00-Board/gate-ds-web-to-kmp-phase1`
