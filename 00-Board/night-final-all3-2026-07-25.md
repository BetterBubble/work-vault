---
title: 'ИТОГ НОЧИ 24→25.07 — все 3 ТЗ к прод-готовности (сводка президенту)'
type: note
permalink: tacticum/00-board/night-final-all3-2026-07-25
status: current
updated: '2026-07-25 01:40'
tags:
- night-final
- tz1
- tz2
- tz3
- prod-readiness
- summary
---

# Итог ночи 24→25.07 — 3 ТЗ Солонко доведены к прод-готовности

За ночь (президент спал) доведены все 3 ТЗ до build-complete + независимо сверены по ТЗ + подтверждены на тест-стенде где возможно. Ниже — что сделано, что на президенте утром.

## Статус по ТЗ (все fidelity-зелёные vs доки Солонко, сверх-ТЗ = 0)

### ТЗ#1 — дизайн Figma↔код (lead-ds) — BUILD-COMPLETE
- В main: Сц.4 (перенос форм one→kmp), Сц.1/2 (навыки angular-ds-component authoring/usage), G5 (ui-mockup-match числовой).
- В очереди PR: **PR-C** (ось-1: G7 discovery framework/surface-fix + G8 web quickstart + iva-core skill + Сц.3-правило + usage/authoring lane-agnostic фикс).
- Финальная сверка: template-предел ДОСТИГНУТ по ТЗ; 1 пробел (usage over-claim G5) НАЙДЕН и ЗАКРЫТ (lane-agnostic, симметрично authoring). [[final-fidelity-tz1]]
- Остаток = чужие команды (handoff): Сц.3-миграция iva-one, серверная iva-core ДС+словарь, G2/G3, 17 null figma_key. [[handoff-tz1-deferred-remainder]]

### ТЗ#2 — режимы разработчика (lead-modes) — 100% в MAIN
- Всё в main: срез 1 (#142) + ось-2 + добор 3 пробелов + §1.2 инфра-свойство (#155).
- Финальная сверка: ПОЛНОТА ПО ТЗ = ДА (100% по коду). [[final-fidelity-tz2]]
- Рантайм: живой пилот на teststand ЗЕЛЁНЫЙ (5 смоуков). [[pilot-tz2-teststand-results]]
- Остаток = прод-сид (gated) + eval-датасет (Солонко).

### ТЗ#3 — аналитик проектирует требования (lead-fr) — BUILD-COMPLETE
- В main: US#1-5 аналитик-контент (FR v2, контракты 3.1/CT-n, DM/EV анализаторы, update-feature) — три предохранителя честности целы, зеркало байт-идентично. US#4 конвейер: канон brd/start-task + mail/rn + композиты ios/firebird + kmp pin/tests.
- В очереди PR: **E-remainder** (kmp brd+start-task), **D-web** (последний проход, GO контент, ждёт мержа PR-C для ребейза).
- Финальная сверка: ПОЛНОТА ПО ТЗ = ДА. [[final-fidelity-tz3]]
- Рантайм: v2-FR пилот на teststand ЗЕЛЁНЫЙ (5 смоуков, ядро handoff). [[pilot-tz3-teststand-results]]

## Очередь PR на мерж (утро) — все прошли независимый гейт ГД (полный набор)
Уже смержено президентом: #152 C-канон, #153 B2, #154 PR-B, #155 §1.2, + C-монолиты, E-pintests.
Ждут мержа: **E-remainder** · **PR-C** (финал) · **D-web** (после мержа PR-C → ребейз fr → push). Порядок и ссылки — [[pr-queue-president]].

## На президенте утром
1. **Мерж очереди PR** (E-remainder, PR-C; затем D-web после ребейза).
2. **Общий прод-деплой всех 3 ТЗ** (решение (c) — единый заход). Рунбук готов: [[prep-combined-prod-seed-all3]] — 26 пакетов, коллизий НОЛЬ, аддитивно, роль-дедуп, бэкап-точка снята, обязательный шаг пере-сверки версий.
3. **Два awareness-решения в прод-сиде:** (а) 4 пакета ВНЕ 3 ТЗ (architect/techwriter-профили — исключены, деплоить или нет?); (б) сид подтянет прод к main-parity (объём шире 3 ТЗ, накопленный лаг).
4. **ADR-канон** (позже, не блокер): ui-base 'unified'→surface-split (KMP-align ось-1 + mirror-дрейф DS-навыков).
5. **Внешнее:** eval-датасет ТЗ#2 (Солонко), рантайм-интеграция (команды Легина/iva-one).

## Что НЕ делалось (границы соблюдены)
Прод не тронут (сид отложен в (c)); серверы READ-ONLY кроме teststand-scratch пилотов; шаренный ui-base не тронут (ADR); чужие направления (architect/techwriter) не деплоятся; строго по ТЗ Солонко, без раздувания.
