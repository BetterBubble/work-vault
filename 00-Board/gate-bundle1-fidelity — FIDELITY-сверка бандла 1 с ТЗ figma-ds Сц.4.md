---
title: 'gate-bundle1-fidelity — FIDELITY-сверка бандла #1 с ТЗ figma-ds Сц.4'
type: note
permalink: tacticum/00-board/gate-bundle1-fidelity-fidelity-sverka-bandla-1-s-tz-figma-ds-sts.4
tags:
- board
- controller
- gate
- tz1
- fidelity
- web-to-kmp
---

# Гейт: FIDELITY-сверка бандла #1 ↔ ТЗ figma-ds Сц.4

**Роль:** controller (read-only, вердикт). **Дата:** 2026-07-24.
**Объект:** worktree `tacticum-dev-ds-web-to-kmp` — навыки `web-to-kmp-screen-port/SKILL.md`, `web-to-kmp-source-reference/SKILL.md` + словарь Фазы-2 (`00-Board/phase2-provisional-iva-web-dictionary`, resolved).
**Истина:** `ds-scan/figma-ds-scenario-4-kmp-port.md` (Сц.4) + `figma-ds-multirepo-and-selection.md` §Ось-2 п.3.

## ВЕРДИКТ: FIDELITY-PASS (0 фабрикаций сверх-ТЗ; отклонения обоснованы разведкой/реальностью репо)

Сделано по ТЗ, без отсебятины. Ни одного выдуманного компонента, ключа, фичи или запрета скоупа. Элаборации сверх текста ТЗ есть — но все привязаны к реальным in-repo навыкам / фрейморк-реальности, не фабрикация. Один пункт (#2) неверифицируем из этого worktree — передан verifier.

## Покрытие по пунктам

**1. Содержание `web-to-kmp-screen-port` — ПОКРЫТО.**
- Процедура чтения источника (порядок): §1 — 8 шагов, точный порядок ТЗ (architecture.md+signal-store→page-route→store shape→view-state enum→list/select/detail→DS-имена→Transloco→REST). ТЗ «1-9» — опечатка (список «что читает агент» = 1-8). ✓
- Таблица Angular→Compose + «не над-переводить»: §2 — все 15 строк ТЗ 1:1; «не над-переводить» в шапке §2 + §5. ✓
- Маппинг состояния signalStore→Decompose/StateFlow, Level-1/3: §3 — Level-1 (MutableStateFlow+onXxx, default) / Level-3 (MVIKotlin, только если экран уже на нём), пофично (withComputed→mapper+stateIn, withMethods→onXxx). ✓
- Гардрейлы приземления: §4 — только `feature/<name>/impl/commonMain` + `Iva*`; never `ucim`/`presentation.*`; навигация Decompose `News`. ✓
- Что не переносить (web-only): §6 — routing/outlets · DOM/CSS · RxJS/DI · WebRTC/calls/conf · Electron. Совпадает дословно. ✓
- Верификация (Iva*/паритет/токены/тесты): §7 — component level (Iva*, 0 raw Color/dp, hoisting, keyed) · web-sample parity · token parity · Compose-render (Roborazzi+VLM) + тесты. ✓
- Принцип «rewrite не move»: §0 + description — раскрыт сильно (контраст с move-port). ✓

**2. Ссылки на реальные in-repo навыки — НЕВЕРИФИЦИРУЕМО ИЗ WORKTREE → verifier.**
§8 перечисляет навыки с суффиксом `-su-ivcs-messenger-kmp` и заявляет «All are real skills in the KMP repo». KMP-репо (su.ivcs.messenger / iva-m/android/kmp) и iva-one в этой среде ОТСУТСТВУЮТ (проверено `ls`). Существование этих навыков — вопрос подлинности (authenticity), не fidelity к ТЗ. Fidelity-аспект: ТЗ требует «оркестрировать и ссылаться, не дублировать» (Сц.4 строка 186) — бандл делает ровно это. **Хэндофф verifier:** подтвердить, что 12 суффиксных навыков §8 реально существуют в KMP-репо (не выдуманы).

**3. Reference-скилл (Ось-2 п.3) — ПОКРЫТО.**
- Два дерева source read-only + target write: таблица «The two trees» (Source iva-one READ-ONLY / Target su.ivcs.messenger WRITE). ✓
- «В источник не писать»: HARD RULE «Never write into the source». ✓
- ДС письма = целевой: «The design system for what you write is the TARGET's, never the source's». ✓
- Доступ: sibling clone / `kb_discover` источника, привязка к своему `.tacticum/context.yaml`; «по образцу angular-legacy-web-context». ✓ (Ось-2 п.3)

**4. Словарь Iva*↔веб-мастер-компонент — ПОКРЫТО.**
- Маппинг Iva*→веб-мастер-компонент (UI KIT) с `figma_key`: 32 реальных ключа, 17 обоснованных null. ✓
- Незамапленное → «пробелы» (обе стороны), «веб-аналог/ключ НЕ выдуман» — не галлюцинирует. ✓ (принцип президента)
- figma_key где есть; извлечены из `tokens.json` (jq), самопроверка приложена. ✓

**5. 0 СВЕРХ-ТЗ — ПОДТВЕРЖДЕНО (нет фабрикаций).**
Ни новых фич, ни выдуманных компонентов/ключей, ни изобретённых запретов скоупа. Скилл остаётся оркестратором; словарь — честная карта с явными пробелами. Явно избегает изобретения (§5 «do not hallucinate», словарь «НЕ выдуман»).

## Отклонения / элаборации сверх текста ТЗ (все с причиной, не блокеры)

1. **Импорт triage-tree + bottom-up order `model→domain→data→infra→feature` из android-to-kmp** (§0). В тексте Сц.4 этого порядка НЕТ. Причина: заимствование конвенции реального in-repo `clean-architecture`/`android-to-kmp` навыков как «structure donor»; поддерживает гардрейл §4. Обосновано, не фабрикация, но **вне буквы ТЗ** — вынести на осведомлённость лида/президента.
2. **`ui-mockup-match` переосмыслен как web-only (playwright+DOM), не Compose-компаратор** (§7.2). ТЗ строка 117-121 сам говорит «turnkey кросс-фреймворк-диффа нет, гейт = токены+VLM». Отклонение = уточнение по реальности инструмента. Обосновано. ✓
3. **android-to-kmp-porting подан как контраст, а не move-навык** (§0) — это НЕ отклонение: ТЗ строки 20-25 сами так его рамят. Faithful.
4. **Пример модуля `feature/contacts`→`feature/contact-detail`** (§3, TODO). Мелкая фактическая правка якоря (выбранный первый экран ContactDetailScreen). Вероятно корректна по репо. Минорно.
5. **Naming `-su-ivcs-messenger-kmp`, «Level-1/Level-3», «Harel sealed-interface mode», §7.1 запрет вложенного LazyColumn** — уточнения от фреймворк-реальности / in-repo навыков (mvi-state-machine, compose-ui-patterns). Вне буквы ТЗ, но грунтед, не изобретённый скоуп. Минорно.

## Осознанные отложения (норма, НЕ недоделка)
- TODO Figma-фрейм/полный инвентарь Iva*/словарь code-bindings/Figma-мост в DS-навыках — явные TODO в §TODO скилла и «Что дальше» словаря. Совпадает с ТЗ (эти правки — отдельные расширения других навыков, не скоуп этого бандла).

## Хэндофф
- **Verifier:** подтвердить существование 12 in-repo навыков §8 в KMP-репо (пункт #2, из worktree неверифицируемо).
- **Лид/президент:** ознакомиться с элаборацией #1 (bottom-up order вне буквы Сц.4) — оставить как грунтед-реюз или урезать до буквы ТЗ.

## Связано
Фаза 2 — словарь Iva*(KMP)↔веб-мастер-компонент (RESOLVED по 32 ключам code-bindings)
