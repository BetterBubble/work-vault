---
status: draft
role: lead-fr
topic: ТЗ#3 US#0 — материалы малого PR (амендмент прошёл гейт, ждёт fidelity-сверки ГД → президента)
repo: /Users/bubblemac/tacticum/tacticum-dev
worktree: /Users/bubblemac/tacticum/tacticum-dev-fr-contracts
branch: feat/fr-analyst-contracts
commit: 192c90a
date: 2026-07-24
permalink: tacticum/00-board/pr-us0-mirror-depreciation-ready
---

# US#0 — материалы для малого PR (амендмент прошёл гейт)

**Состав президент-апрувнут (3→5).** autonomy off + правило READ-ONLY: **push/мерж — президент лично с лидом.** Я подготовил, не пушу.

## Заголовок PR
`feat(fr-analyst): вывести 5 ингредиентов из mirror-пары iva-analysis-base ↔ iva-fr-analyst (ТЗ#3 US#0)`

## Описание PR
**Что делает.** Enabling-шаг ТЗ#3: разъединяет mirror-зеркало для пяти ингредиентов, чтобы последующие US#1–2 (и US#5) могли править их в `iva-fr-analyst` без нарушения байтовой синхронизации с владельцем. Содержимое ингредиентов здесь НЕ меняется — только разъединение зеркала. Плюс закрыта протокольная дыра в норме (frozen-owner).

**Почему (расхождение аудиторий).** `iva-analysis-base` учит аналитиков честной постановке as-is (правило §2 без ослаблений; прод обучения Diaret, профиль заморожен). `iva-fr-analyst` эволюционирует в design-capable направлении (релаксация §2 в Части 1 под предохранителями). Держать оба под байтовым зеркалом больше нельзя — расхождение осознанное (решение президента, вариант Б).

**Состав (5 из 6 ингредиентов пары):** `api-contracts-discovery`, `fr-authoring`, `update-feature` (правят US#2/#1/#5) + `design-system-discovery`, `mockup-authoring` (правят US#1 — переезд UI-требований П.H → §1.9). Под зеркалом остаётся РОВНО `start-feature` (тонкая обёртка, правок под ТЗ#3 не требует).

**Границы.** Владелец `iva-analysis-base` НЕ тронут (0 файлов).

**Файлы (4):**
- `templates/_mirrors.yaml` — из первой пары выведены 5 ингредиентов + в ШАПКУ (норму) добавлена оговорка frozen-owner (при заморож. владельце запись в CHANGELOG зеркала, владельца не трогаем) — закрывает протокольную дыру.
- `docs/adr/0059-…-role-packs.md` §7 — счётчик обновлён + абзац о первом осознанном расхождении (5 ингредиентов, причина + отсылка на US#1 §1.9) + frozen-owner исключение.
- `templates/iva-fr-analyst/CHANGELOG.md` — запись `[0.1.11]` (5 ингредиентов).
- `templates/iva-fr-analyst/manifest.yaml` — версия `0.1.10 → 0.1.11`.

**Примечание для ревьюера:** счётчик в ADR §7 меняется `61 → 57`. Удалено **пять** ингредиентов, но арифметика «61−5=56» НЕ сходится, потому что прежний текст ADR содержал доко-ошибку off-by-one — фактический baseline по `check_mirror_sync.py` был **62**, не 61. Итог 62−5 = **57** корректен и подтверждён тулом; правка заодно чинит прежнюю доко-ошибку.

**Проверки (перепроверены controller-гейтом независимым прогоном, PASS по 6 пунктам):**
- `python scripts/check_mirror_sync.py` → exit 0: «OK — 57 зеркальных ингредиентов в 6 парах синхронны.»
- `pytest apps/backend/tests/catalog/test_role_replacement_parity.py --noconftest` → 77 passed (block A byte-check: на 2 кейса меньше; block B replacement-parity зелёный, не затронут).
- `python scripts/check_profile_version_discipline.py --diff-against main` → exit 0: «OK — 46 profile(s) clean.»
- Самосогласованность: ADR §7 = check_mirror_sync = grep по HEAD = **57**.

## Прошло проверки лида
- critic (`critic-us0-adr-depreciation`): блокер закрыт (frozen-owner в норме), риск отработан (design-system-discovery/mockup-authoring выведены), гигиена закрыта.
- controller-гейт после амендмента (`gate-us0-amendment`): **PASS** по 6 пунктам, owner нетронут, git чист, 0 секретов/AI-подписей.

## Что нужно от президента (лично с лидом)
1. Разрешение на `git push origin feat/fr-analyst-contracts`.
2. Ревью + мерж малого PR.

## Последовательность (US#1-2)
US#1–2 правят выведенные ингредиенты. Их ветка — от main ПОСЛЕ мержа US#0. Спеки готовы (`spec-us1-fr-authoring-v2`, `spec-us2-api-contracts-format-31`).

## Связано
[[impl-us0-amendment]] · [[gate-us0-amendment]] · [[critic-us0-adr-depreciation]] · [[report-us0-check-battery]] · [[plan-tz3-us-breakdown-detailed]]
