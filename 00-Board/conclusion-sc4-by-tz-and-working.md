---
title: 'Заключение: Сц.4 (web-to-kmp) — строго по ТЗ и рабочее (для президента)'
type: note
status: resolved
permalink: tacticum/00-board/conclusion-sc4-by-tz-and-working
tags:
- board
- design-system
- lead-ds
- tz1
- conclusion
- acceptance
---

# Заключение lead-ds: Сц.4 — по-ТЗ и рабочее?

**Для:** президента (через ГД). **Требование:** убедиться, что отдаём РАБОЧЕЕ, не фигню; навык делает ровно что ТЗ Сц.4 требует, статприёмка железная, находки честны, нет дыр за готовое. **Рантайм=b** (принято): без живой сборки (стенд не билд-среда), «тест» = статприёмка + fidelity-полнота по ТЗ; полный рантайм — при интеграции у команды Легина.

## ВЕРДИКТ
- **По-ТЗ Сц.4 (содержание capability): ДА.** Навык-оркестратор + reference-скилл + словарь покрывают ВСЕ требования ТЗ Сц.4, 0 сверх-ТЗ (независимая fidelity-сверка).
- **Рабочее (статически): ДА, железно.** Первый реальный прогон навыка на живом экране `ContactDetailScreen` (su.ivcs.messenger): собран-из-Iva*/state-hoisting проверяемы, незамапленное→СТОП сработал, компоненты НЕ выдуманы.
- **Полная рантайм-приёмка (runComposeUiTest+Roborazzi+VLM): НЕ выполнена — и это НЕ скрыто.** Стенд teststand не билд-среда KMP (нет Android SDK, проект там не компилируется). Честно отложено до интеграции у команды Легина. НЕ выдаётся за готовое.
- **Готово к передаче** как capability #1 (в main). Рантайм-валидация — известный, помеченный следующий шаг у KMP-команды.

## По-требовательная сверка с ТЗ Сц.4 (что просили → что отдано → чем подтверждено)
| Требование ТЗ Сц.4 | Отдано | Подтверждено |
|---|---|---|
| Навык `web-to-kmp-screen-port` (оркестратор) | ✅ в main (0.7.0) | fidelity-PASS |
| Процедура чтения источника (структура/store/view-state/DS/Transloco/REST) | ✅ §1 (+F-1: полный набор .ts+.html+data-access+route) | fidelity, critic |
| Таблица Angular→Compose | ✅ §2 (все строки) | fidelity |
| Маппинг состояния signalStore→Decompose/StateFlow (L1) / MVIKotlin (L3) | ✅ §3 | fidelity, critic |
| Гардрейлы приземления (feature/impl/commonMain + Iva*, не ucim/presentation) | ✅ §4 | fidelity |
| Что не переносить (web-only/WebRTC/Electron) | ✅ §6 (дословно) | fidelity |
| Верификация: static gate + трёхсторонний паритет (веб-образец·токены·Compose) + **behavioural parity** отдельным легом | ✅ §7 (critic-фикс: три-vs-4 снят, поведенческий паритет вынесен) | critic-fix гейт PASS |
| «rewrite не move» (контраст с реальным android-to-kmp-porting) | ✅ §0 | fidelity, critic |
| Оркестрирует in-repo навыки, не дублирует | ✅ §8 (11 реальных + source-reference) | critic |
| Ось-2 п.3: два дерева source-RO/target-write, «в источник не писать», ДС письма=цель | ✅ `web-to-kmp-source-reference` в main | fidelity, gate |
| Словарь Iva*↔веб-мастер-компонент, незамапленное→стоп, figma_key | ✅ resolved (32 ключа + 17 обоснованный null) | dictionary-gate (ключи сверены с tokens.json) |
| Приёмка: собран из Iva*, паритет, незамапленный→стоп | ✅ статически на ContactDetail (dry-пилот) | pilot + gate |
| Приёмка рантайм: runComposeUiTest+Roborazzi+VLM зелёные | ⏳ НЕ выполнено (стенд не билд-среда) → у Легина | env-finding (честно) |

## Статприёмка (dry-пилот ContactDetail) — что реально проверено
- **state hoisting — ПРОШЁЛ** (экран stateless, StateFlow+onUiEvent, Decompose владеет состоянием).
- **незамапленное→СТОП — ПРОШЁЛ** (Iva* НЕ выдуман: 4 пробела честно — LastPastCalls/guest-profile/skeleton/MediaList).
- **собран-из-Iva* — вскрыл РЕАЛЬНЫЙ дефект целевого кода** (не навыка): `ContactDetailScreen.kt:70-77` использует сырой `Spacer/Modifier.padding/Dp` вместо `IvaSpacer` → это находка для реального порта, честно зафиксирована, НЕ скрыта.
- Парити-анализ вскрыл верный анкор источника (деталь = profile-sidebar/employee-info, не список) — навык реально ведёт к правильному разбору.

## Честность (нет дыр за готовое)
Всё незавершённое — ЯВНО помечено, НЕ выдаётся за готовое: рантайм-сборка (env), полный инвентарь Iva* с сигнатурами (пробелы словаря обеих сторон), 17 null-ключей (обоснованный null, не TODO), Figma-фрейм экрана, промоция словаря в серверные code-bindings, `IvaBottomSheet→Dialog` (семантический матч — подтвердить у владельца ДС). Пилот всюду помечает «н/д» где данных не было.

## Основание (независимая верификация — не самосертификация)
- **fidelity-гейт:** FIDELITY-PASS, 0 сверх-ТЗ, все пункты покрыты (`gate-bundle1-fidelity`).
- **critic:** доктрина образцовая, F-1/2/3/5 закрыты, честность пробелов; 3 обязательные правки внесены и прогейчены (`critic-bundle1` + `gate-bundle1-critic-fixes`).
- **git/controller:** скоуп/чистота/валидаторы PASS (`gate-bundle1-git-final`).
- **dictionary-гейт:** все figma_key сверены с `tokens.json` (jq), не выдуманы (`gate-ds-dictionary-final`).
- Всего 9 controller-гейтов по ходу, 0 самовольных мержей, серверы read-only весь путь.

## Итог
**Сц.4 по-ТЗ = ДА, статически рабочее = ДА (железно), готово к передаче.** Полная рантайм-приёмка — у команды Легина при интеграции (среда), помечена честно. Не фигня: реальный прогон навыка дал корректный разбор + вскрыл настоящий дефект целевого кода, ничего не выдумано.

## Следующее (по решению президента)
Память (остаток ТЗ#1: Сц.1/2/3, ось-1) → прод-сид (gated) + проверка.

## Связано
[[plan-tz-1-dizain-protsess-figma-kod-sts.4-perenos-form-one-kmp-lead-ds]] · `gate-bundle1-fidelity` · `critic-bundle1` · `pilot-runtime-env-finding` · `phase2-provisional-iva-web-dictionary`
