---
title: 'Финальная сверка полноты ТЗ#2 vs proposal Солонко'
type: note
permalink: tacticum/00-board/final-fidelity-tz2
status: current
updated: '2026-07-25 00:20'
tags:
- final-fidelity
- tz2
- prod-readiness
---

# Финальная сверка ТЗ#2 (workflow-modes-proposal) vs main @ 8f8287c

**Ревизор:** независимый critic-агент. **ВЕРДИКТ: ПОЛНОТА ПО ТЗ = ДА (100% по коду).**

Все пункты proposal доставлены в main, привязаны к path:line:
- §1/§1.2 режимы lite (только refactoring-S/feature-S) + research лейны; инфра = свойство (не режим, не эскалация) — `tacticum-lite-base` 0.1.3.
- §2/§4 1-й гейт классификации в `/start-task` (bugfix→fix-bug, refS/featS→lite, research→start-research) + «чему не доверять».
- §3 2-й слой mode-review в Фазе-1 (полный→lite/research/split) + на Фазах 3-4 (blocker/3 итерации→split) + lite→research мини (в SKILL И командном теле).
- ADR-first вход = параметр `/start-task` + контракт research→build.
- handoff.md единый формат на всех плечах. §6 role-wiring (research→7 ролей, lite→6 dev-ролей).

**Реальных пробелов: НЕТ** (§1.2 закрыт). **Сверх-ТЗ: НЕТ** (добор аддитивен).
**Осознанно вне ТЗ (proposal НЕ требовал как обязательное):** кампании рефакторинга (по спросу), /task-диспетчер (опц.), §5 диалоги (иллюстрации), GPT-5.6 (только eval-кохорты), eval 2.1 (внешний гейт, отложен президентом до датасета Солонко).

**⚠️ Оговорка приёмки:** ось-2 (`iva-kmp-brownfield` cross-repo) — НЕ часть workflow-modes-proposal, а отдельная cross-scope спека ГД; приёмится отдельно, не засчитывается ни в полноту, ни в сверх-ТЗ proposal.

**Остаток до прода:** только прод-сид режимов в каталог (внешний gated-шаг президента). Контент годится как фундамент, критичных правок не требует.
