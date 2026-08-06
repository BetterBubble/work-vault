---
title: 'Critic-ревью бандла #1 (web-to-kmp navыки + словарь) — с итогом правок'
type: note
status: resolved
permalink: tacticum/00-board/critic-bundle1-1
tags:
- board
- design-system
- lead-ds
- tz1
- critic
archived-at: 2026-08-03 11:16
---

# Critic-ревью бандла #1 (push-флоу ГД, шаг 1)

**Ревизор:** critic-агент, 2026-07-24 (не смог записать сам — персистит lead-ds). **Объект:** `web-to-kmp-screen-port` + `web-to-kmp-source-reference` + словарь Фазы-2. **Вердикт:** скелет сильный, почти готов к PR; **3 обязательные правки до PR** (внесены implementer'ом, commit `39ae642`, 0.7.0).

## Сильные стороны (не трогать)
- Доктрина «rewrite ≠ move» проведена чисто (§0 + description, точный контраст с android-to-kmp-porting: что реюзать / что НЕ тянуть).
- Пилотный фидбэк F-1/F-2/F-3/F-5 закрыт по существу (§1 полный источник, §7.1 LazyColumn-нюанс, §0 fix-parity триаж, §7.1 читать виджеты).
- Дисциплина «ссылаться, не копировать» соблюдена; разделение двух навыков чёткое, взаимно-ссылочное.
- Честность пробелов образцовая (словарь matched-key/matched-null/пробелы; «не выдумывать»; §9 inventory-of-truth; пилот всюду «н/д»).

## Обязательные правки до PR (ВНЕСЕНЫ)
1. **§7 «three legs» ≠ 4 пункта** → переформулировано: static gate (component level) + three-way parity (web-sample·tokens·Compose-render). ✅ commit 39ae642.
2. **Поведенческий/логический паритет размазан** (главный acceptance Сц.4 №2) → вынесен ОТДЕЛЬНЫМ явным легом (ручная code-сверка source↔target, не ui-mockup-match). ✅
3. **TODO-якорь словаря устарел** (словарь resolved) → помечен RESOLVED + reverse-keyed + crosswalk-ссылки в §1.6/§5. ✅ (частичный F-4 закрыт).

## Nice-to-have (внесены 4/5/7; 6/8 — отложены/сверены лидом)
- 4 (Level-2 gap-нота §3) ✅ · 5 (§9 timeless: убрать автора/числа) ✅ · 7 (§0 scope-нота) ✅.
- 6 (source-reference ссылки ivcs-libs-contract/angular-legacy-web-context) — **сверено лидом: оба навыка реальны** (iva-web-development-base / iva-rn-brownfield) → не блокер.
- 8 (ужать description) — отложено (не блокер).

## Связано
[[План ТЗ-1 Дизайн-процесс Figma↔код — Сц.4 перенос форм one→kmp (lead-ds)]] · `00-Board/impl-ds-critic-fixes` · `00-Board/gate-bundle1-git-final` · `00-Board/gate-bundle1-fidelity`