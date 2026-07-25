---
title: PR-очередь на мерж (президенту)
type: note
permalink: tacticum/00-board/pr-queue-president
status: current
updated: '2026-07-24 23:30'
tags:
- pr-queue
- merge
- tz1
- tz2
- tz3
---

# PR-очередь на мерж — президенту (ночь 24→25.07)

Все PR прошли независимый гейт ГД (полный набор тестов + оба гардрейла + git-чистота). Готовы к созданию+мержу.

## 🔴 РЕШЕНИЕ ПО ПРОД-ДЕПЛОЮ ТЗ#2 — ЖДЁТ ТЕБЯ (a/b/c)
Ты авторизовал прод-деплой ТЗ#2. modes подготовил всё (pre-flight: коллизий нет, 12 created; pg_dump-бэкап снят — точка отката; депы резолвятся), НО остановился ДО сида на находке: пакет `iva-analysis-base` версионируется **атомарно** (прод 0.1.3 → цель 0.1.7), и между ними намешан **незадеплоенный контент ТЗ#3** (0.1.5/0.1.6 — fr: FR v2, анализаторы DM/EV, /update-feature) + это **Diaret-чувствительная база**. Засидить «только ТЗ#2» физически нельзя. Сам не пускаю (вне авторизации + Diaret-база). Прод в исходном, бэкап цел.
**Варианты:** (a) сид всё как есть — ТЗ#2 полный + осознанно едет ТЗ#3-аналитик-контент (аддитивно, откат готов); (b) 11 пакетов без iva-analysis-base = ТЗ#2 неполон (без гейта классификации/ADR-first); (c) отложить в общий утренний деплой всех 3 ТЗ. **Рекомендую (c).** Твоё слово.

## ✅ УЖЕ СМЕРЖЕНО президентом (4)
#152 C-канон · #153 B2 · #154 PR-B · #155 §1.2. main = 5884bcd.

## 🟢 НОВАЯ ОЧЕРЕДЬ — готовы к мержу (запушены чисто, прошли полный гейт)

- **C-монолиты** (ТЗ#3 US#4) — `https://github.com/TacticumApps/dev/pull/new/feat/us4-passC-monoliths`
  `sync(conveyor): start-task гейт D-n в mail/rn — ТЗ#3 US#4 C-монолиты` · mail 0.7.3→0.7.4, rn 0.5.3→0.5.4. Независим, мержить в любой момент.
- **E-pintests** (ТЗ#3 US#4) — `https://github.com/TacticumApps/dev/pull/new/feat/us4-passE-kmp-pintests`
  `feat(conveyor): kmp pin/tests К-2/К-4 — ТЗ#3 US#4 E-pintests` · iva-kmp-brownfield 0.5.0→0.5.1. ⚠️ мержить ПЕРЕД E-remainder (kmp brd+start-task стекнут на него, 0.5.1→0.5.2).

- **E-remainder** (ТЗ#3 US#4) — `https://github.com/TacticumApps/dev/pull/new/feat/us4-passE-remainder`
  `feat(conveyor): kmp brd v2-FR + start-task гейт D-n — ТЗ#3 US#4 E-remainder` · iva-kmp-brownfield 0.5.1→0.5.2. ⚠️ мержить ПОСЛЕ E-pintests (уже в main #157). axis-2 цел.

- **PR-C** (ТЗ#1 ось-1 — ЗАВЕРШАЕТ ТЗ#1, ✅ полон в origin @ 2d919de) — `https://github.com/TacticumApps/dev/pull/new/feat/ds-web-axis1`
  `feat(iva-web): design-system-discovery surface/framework fix + iva-core skill + web figma quickstart + Сц.3 rule + usage/authoring lane-agnostic` · iva-web-brownfield 0.4.0→0.5.0, iva-web-development-base 0.1.1→0.1.2. coverage зелёный, ui-base не тронут, over-claim usage/authoring закрыт (финальный перегейт GO).

## ⏳ СТРОИТСЯ — последний кусок
- **D (web)** (ТЗ#3 US#4, ПОСЛЕДНИЙ проход) — fr строит; push после мержа PR-C (контенция iva-web-brownfield). После него US#4 build-complete.

## ✅ BUILD-COMPLETE по ТЗ
- **ТЗ#1** — template-предел ДОСТИГНУТ (Сц.4+Сц.1/2+G5 в main; ось-1+Сц.3-правило = PR-C в очереди; остальное = handoff [[handoff-tz1-deferred-remainder]] для iva-one/RE/server/дизайнеров). Сводка [[tz1-build-complete-summary]].
- **ТЗ#2** — код 100% в main (§1.2 #155). Сводка [[summary-tz2-vs-proposal]].
- **ТЗ#3 US#4** — 5/6 проходов в main/очереди; остаётся D-web.

## ⚠️ ДЛЯ ТВОЕГО УТРЕННЕГО РЕШЕНИЯ (в общем прод-сиде, не блокеры)
- **4 пакета ВНЕ 3 ТЗ** в дельте прода: architect-профиль (iva-architect-mcp/iva-role-architect) + techwriter (iva-techwriter-mcp/tacticum-role-techwriter) — чужие направления, modes ИСКЛЮЧИЛ из захода. Решить: деплоить их или нет (отдельно от 3 ТЗ).
- **Прод отстаёт шире 3 ТЗ:** утренний сид подтянет весь накопленный лаг к main-parity (kmp-development-base 0.1→0.7, web-brownfield 0.2.1→0.4 и др.), аддитивно, но объём больше «вчерашнего». Единый чеклист: [[prep-combined-prod-seed-all3]] (26 пакетов, коллизий ноль).

## 📦 БЫВШАЯ «готовы» (в старой версии заметки) — уже смержены выше

1. **C-канон** (ТЗ#3 US#4) — `https://github.com/TacticumApps/dev/pull/new/feat/us4-passC-canon-starttask`
   `feat(conveyor): start-task гейт D-n (plate-based) + К-5 — ТЗ#3 US#4 Проход C-канон` · tacticum-dev-base 0.2.6→0.2.7, 3 файла.
2. **B2** (ТЗ#3 US#4) — `https://github.com/TacticumApps/dev/pull/new/feat/us4-passB2`
   `feat(conveyor): pin/tests К-2/К-4 в mail/rn/ios/firebird — ТЗ#3 US#4 Проход B2` · 16 файлов, 4 профиля.
3. **PR-B** (ТЗ#1 Сц.2) — `https://github.com/TacticumApps/dev/pull/new/feat/ds-web-mockup-figma`
   `feat(iva-web): ui-mockup-match Figma numeric-compare (Сц.2 ш7, G5)` · iva-web-brownfield 0.3.0→0.4.0, 3 файла.

**Порядок:** C-канон и B2 — disjoint, мержить в любом порядке. PR-B — независим (iva-web-brownfield). Все три можно мержить сразу.

## ⛔ НУЖЕН ТВОЙ ХОД (1 PR — блок публикации)

4. **§1.2 инфра-свойство** (ТЗ#2, закрывает на 100%) — **НЕ запушен**: харнес-классификатор в окне lead-modes блокирует `git push` (у fr/ds окна разрешают, у modes — нет). Гейт GO уже есть (290 не-DB passed, чисто).
   **Действие:** либо разреши push в окне modes, либо запушь сам:
   `git push origin feat/workflow-modes-infra` из `/Users/bubblemac/tacticum-worktrees/modes-infra`
   → после этого modes мгновенно даст create-link, смержишь.

## ⏳ В РАБОТЕ (придут после мержей выше)

- **C-монолиты** (fr) — контент-готов, стек на C-канон; ребейз+push после мержа C-канон+B2 (версии mail/rn 0.7.4/0.5.4).
- **PR-C** (ds) — G8 quickstart + iva-core-тонкий + G7-узкий (iva-web-brownfield); строится, push после мержа PR-B (контенция iva-web-brownfield).
- **D (web) / E (kmp)** (fr) — карты готовы, окна брокерю.

## 🔶 НА ТВОЁ РЕШЕНИЕ (не блокер, после возврата)

- **Канон-ADR G7:** навык `design-system-discovery` в `tacticum-ui-base` несёт допущение «Iva DS unified across surfaces», которое ось-1 должна снять (iva-core = отдельная поверхность). Это шаренный пакет (kmp/ios/mail-роли) + в mirror-паре lead-fr → правка = ADR-класс + координация с lead-fr. Сейчас НЕ трогаем (узкий фикс в iva-web-brownfield закрывает немедленную нужду). Отдельным ADR-заходом решим с тобой.
- **ТЗ#2 gated-остаток:** прод-сид (чеклист [[prep-tz2-prod-seed-checklist]]) + eval-ask Солонко [[prep-tz2-eval-readiness]].
