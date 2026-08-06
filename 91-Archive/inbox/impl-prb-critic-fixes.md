---
status: draft
role: implementer
topic: PR-B — правки критика в навыке ui-mockup-match (Figma-режим)
repo: /Users/bubblemac/tacticum/tacticum-dev-web-mockup
worktree: /Users/bubblemac/tacticum/tacticum-dev-web-mockup
branch: feat/ds-web-mockup-figma @ 09e523a
date: 2026-07-24
permalink: tacticum/00-board/impl-prb-critic-fixes-1
archived-at: 2026-08-03 11:16
---

# PR-B — critic fix-round для ui-mockup-match (Figma-режим)

Внесены правки критика перед вехой. Только веб-копия навыка; зеркала/owner/другие 4 копии не тронуты.

## Изменённые файлы

- `templates/iva-web-brownfield/ingredients/skills/ui-mockup-match/SKILL.md`
- `templates/iva-web-brownfield/CHANGELOG.md`

## git diff --stat

```
 templates/iva-web-brownfield/CHANGELOG.md          | 22 +++++++++++++++---
 .../ingredients/skills/ui-mockup-match/SKILL.md    | 27 ++++++++++++++--------
 2 files changed, 36 insertions(+), 13 deletions(-)
```

## По пунктам

1. **(ОБЯЗ.) Colour ΔE — computed style вместо bbox-семпла скриншота.** §Metrics п.1
   переписан: рантайм-цвет читается из computed style через `browser_evaluate`
   (`color` для text/icon, `background-color` для fill, `border-color` для border —
   тот же `getComputedStyle`-путь, что метрика размера п.2 и шаг §1 capture), затем
   ΔE (CIEDE2000) против токен-значения. Явно убран bbox-семпл и добавлено
   объяснение почему он неверен: `get_metadata` bbox = координаты Figma-фрейма, не
   рантайм-лейаут; для text/icon/border bbox-семпл вернёт фон, не искомый цвет.
   Формулировка «not a blind pixel-diff» усилена: computed-style-цвет ≠ пиксель.

2. **(дешёвая) Tolerance — висячая «uiMatch» убрана.** Из секции Tolerance удалены
   обе отсылки («per the uiMatch example» и «These defaults follow the uiMatch
   reference»). Дефолты ΔE≤2.0/±2px теперь опираются на самодостаточное обоснование
   «0/ΔE=0 недостижим из-за анти-алиасинга → допуск обязателен». То же выравнивание
   сделано в CHANGELOG (строка про Tolerance).

3. **(дешёвая) Детект активной темы.** В «Honest dependencies → ΔE needs the active
   theme mode» добавлена строка: определять light/dark по рантайму через
   `browser_evaluate` (класс `.dark` / атрибут `data-theme` на `<html>`/root, либо
   значение theme CSS-переменной), не предполагать light по умолчанию.

4. **(дешёвая) Опечатка CHANGELOG.** «This is the **veb**» → «web». Проверено:
   `grep veb` по каталогу профиля — пусто.

## Версия

Оставлена **0.4.0** (правки внутри уже-добавленного на 0.4.0 навыка, не в main).
Валидатор version-discipline `--diff-against origin/main` — clean, бамп не требует.
В CHANGELOG [0.4.0] добавлена подсекция `### Fixed` с пометой «critic fix-round (PR-B)»,
кратко описывающая все три содержательных фикса.

## Валидаторы (venv `apps/backend/.venv/bin/python`)

- `scripts/check_mirror_sync.py` → `OK — 64 зеркальных ингредиентов в 6 парах синхронны.`
- `scripts/check_profile_version_discipline.py` (static) → `OK — 48 profile(s) clean.`
- `scripts/check_profile_version_discipline.py --diff-against origin/main` → `OK — 48 profile(s) clean.`
- `pytest -k manifest_schemas` → `38 passed, 1399 deselected`.

## Commit

`09e523a` — `fix(ds-web): ui-mockup-match critic fixes — computed-style colour sampling + theme detect (PR-B)` (без AI-подписей). НЕ запушен.

## Самопроверка

- Цвет снимается через computed style (browser_evaluate), bbox-подход удалён — да.
- Tolerance без висячей ссылки на uiMatch (SKILL + CHANGELOG) — да.
- Детект темы описан явно — да.
- Опечатка veb→web исправлена — да.
- Защищённое лидом/fidelity не тронуто: токен-якоря (radius.control-m / padding.* /
  gap.* / color-токены), installation_id-дисциплина, разведение HTML/Figma режимов,
  запрет слепого pixel — не менялись. HTML-режим не тронут.
- Только web-копия (`iva-web-brownfield`); зеркала/другие копии — не тронуты
  (mirror-sync зелёный).