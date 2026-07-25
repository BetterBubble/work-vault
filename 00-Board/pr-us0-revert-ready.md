---
status: draft
role: lead-fr
topic: ТЗ#3 — материалы revert-PR (откат US#0, вариант А), прошёл гейт, ждёт президента
repo: /Users/bubblemac/tacticum/tacticum-dev
worktree: /Users/bubblemac/tacticum/wt-revert-us0
branch: revert/us0-mirror-depreciation
commit: 381f906
date: 2026-07-24
permalink: tacticum/00-board/pr-us0-revert-ready
---

# Revert US#0 — материалы малого PR (гейт PASS)

Разворот варианта Б → А (решение президента): правим владельца, зеркало следует → разъединение US#0 больше не нужно, откатываем. autonomy off + READ-ONLY: **push/мерж — президент** (лично, как US#0).

## Заголовок PR
`revert(fr-analyst): откат разъединения mirror-пары iva-analysis-base ↔ iva-fr-analyst (ТЗ#3, разворот на вариант А)`

## Описание PR
**Что делает.** Откатывает US#0 (PR #140): возвращает 5 ингредиентов в mirror-пару `iva-analysis-base ↔ iva-fr-analyst`, восстанавливая протокол зеркала. Причина — смена модели доставки ТЗ#3 на **вариант А**: правим общие скиллы в ВЛАДЕЛЬЦЕ `iva-analysis-base`, зеркало `iva-fr-analyst` следует байт-в-байт → обе роли (обучение + design-capable) получают способность. Разъединение (вариант Б) держалось на заморозке владельца; заморозка снята → основание отпало.

**Что сохранено (намеренно).** Оговорка frozen-owner в тексте нормы `_mirrors.yaml` (общеполезна, вреда нет) — НЕ откатывается. off-by-one фикс счётчика тоже сохранён (счётчик = 62, истинный baseline).

**Границы.** Тела скиллов НЕ тронуты; владелец `iva-analysis-base` НЕ тронут.

**Файлы (4, +7/−26):**
- `templates/_mirrors.yaml` — 5 ингредиентов возвращены в первую пару (полный список 6); оговорка frozen-owner в шапке сохранена.
- `docs/adr/0059-…-role-packs.md` §7 — счётчик `57 → 62`, удалён абзац о «первом осознанном расхождении».
- `templates/iva-fr-analyst/CHANGELOG.md` — удалена запись `[0.1.11]`.
- `templates/iva-fr-analyst/manifest.yaml` — версия `0.1.11 → 0.1.10`.

**Проверки (перепроверены controller-гейтом независимым прогоном, PASS по 6 пунктам):**
- `check_mirror_sync.py` → «OK — 62 зеркальных ингредиентов в 6 парах синхронны.»
- `pytest test_role_replacement_parity.py --noconftest` → 82 passed (= pre-US#0 baseline).
- `check_profile_version_discipline.py --diff-against origin/main` → «OK — 46 profile(s) clean» (понижение версии дисциплину не уронило).
- Самосогласованность: ADR §7 = check_mirror_sync = grep = 62.

## Прошло проверки лида
- controller-гейт `gate-us0-revert`: **PASS** по 6 пунктам, owner+тела скиллов нетронуты, git чист, 0 секретов/AI-подписей, счётчик самосогласован.

## Что нужно от президента (лично с лидом)
1. Разрешение на `git push origin revert/us0-mirror-depreciation`.
2. Ревью + мерж малого revert-PR.

## После мержа revert
main возвращается в owner-mirror состояние → US#1-А: применяю готовый контент `fr-authoring/SKILL.md` (US#1, коммит edc5cc2) к владельцу `iva-analysis-base` + зеркало байт-в-байт одним PR → controller-гейт → US#2.

## Связано
[[impl-us0-revert]] · [[gate-us0-revert]] · [[impl-us1-fr-authoring-v2]] · [[plan-tz3-us-breakdown-detailed]]
