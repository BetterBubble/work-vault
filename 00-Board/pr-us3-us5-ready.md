---
status: draft
role: lead-fr
topic: ТЗ#3 US#3+US#5+§2.4 — материалы PR + полный отчёт (ждёт разводки очереди мержа с lead-modes → fidelity ГД → президент)
repo: /Users/bubblemac/tacticum/tacticum-dev
worktree: /Users/bubblemac/tacticum-worktrees/us3-dm-ev
branch: feat/us3-dm-ev
commits: 366c798 (US#3) + 98639dc (US#5) + 8ffff73 (§2.4)
date: 2026-07-24
permalink: tacticum/00-board/pr-us3-us5-ready
---

# US#3+US#5+§2.4 — материалы PR + полный отчёт (гейты PASS)

⚠️ **НЕ ПУШУ** — жду разводки очереди мержа с lead-modes (общий файл `iva-analysis-base`) + fidelity ГД → президент.

## ⚠️ КОНФЛИКТ-ЗОНА С lead-modes (общий `iva-analysis-base`)
Оба трогаем `iva-analysis-base` параллельно → конфликт гарантирован при слепом пуше (урок #143):
- **Я (ТЗ#3):** version `0.1.5 → 0.1.6`; CHANGELOG новая запись `[0.1.6]`; manifest — +2 skill_spec (data-model-analyzer, events-analyzer), счётчик ingredients `17 → 22` (комментарий «8 skill_spec» → «12 skill_spec»).
- **lead-modes (ТЗ#2):** тоже bump version + CHANGELOG + добор tacticum-workflow/start-task в тот же base.
- **Разводка нужна ГД:** кто мержит первым; второй ребейзит version (→0.1.7) + CHANGELOG-стек + счётчик ingredients в manifest.

## ПОЛНЫЙ ОТЧЁТ ПО ГЕЙТАМ (оба PASS)

### critic §2 → (а) верно ТЗ, блокеров нет (`critic-us5`)
- Валидатор границы (iii) US#5 **дыру НЕ открыл**: основание (ii) безусловно в ОБОИХ состояниях («требует утверждения» плашка+Q-n / «утверждён» D-n); лазейки «D-n без основания» нет (перекрёст ii↔iii явный).
- Пер-раздельная перегенерация (ТЗ §2.4 п.5): раздел по природе → перезапуск только нужного навыка. ✅
- D-n-фиксация (ТЗ §2.2 п.3): утверждение dev+CTO → D-n в П.D, снятие плашки/Q-n. ✅
- §2.4-шапка в api-contracts согласована с DM/EV (ссылка честная). ✅
- Регресс US#3/US#1-2 — не сломан, §2-дисциплина едина в 4 файлах. Мелкая заметка (не дефект): «вход +§1.4/5» в шапках оправдано трассировкой §3.1 п.6.

### controller-гейт → PASS 7/7 (`gate-us3-us5`)
- Git: 3 коммита, дерево чистое, 15 файлов (+1005/−61), скоуп ровно ожидаемый.
- Зеркало: `diff -q` owner↔mirror identical по всем 5 тронутым ингредиентам; `check_mirror_sync` → **OK 64** (6 пар).
- Скоуп: design-system-discovery/mockup-authoring/start-feature НЕ тронуты.
- Прод-safe: аддитивно, валидатор дыру не открыл, базовый флоу update-feature цел.
- Секреты/AI-подписи: **NONE** (пути `.claude/…` — легитимные target-пути установки).
- Версии: base 0.1.6 / fr-analyst 0.1.12; `version-discipline` → clean, 48 профилей.
- Зелёность (независимый прогон): mirror 64, version-discipline clean, pytest целевые (schemas+parity+role_presets+install_smoke) → **290 passed**; **parity зелёный БЕЗ allowlist**, REPLACEMENTS не тронут.

### Sync main (правило #143)
`origin/main` = `928fe37` = наша merge-base (не двигался с ветвления US#3) → ветка уже на свежем main, sync-merge не нужен, конфликта в git-дереве нет. (Координация с lead-modes — на уровне ОЧЕРЕДИ МЕРЖА, т.к. оба зайдут в один base.)

## Заголовок PR
`feat(analyst): навыки DM/EV (§1.7/§1.8) + /update-feature пер-раздельно+D-n + §2.4-симметрия — ТЗ#3 US#3+US#5`

## Описание PR
**Что делает.** Завершает контентную часть ТЗ#3 в профиле аналитика: (US#3) 2 навыка-анализатора `data-model-analyzer` (§1.7, DM-n) + `events-analyzer` (§1.8, EV-n) с консолидацией событийной серии (EV-n=EV-nn, единый источник); (US#5) `/update-feature` пер-раздельная перегенерация (перезапуск только затронутого навыка) + фиксация утверждений проектных разделов как D-n; (+§2.4) явная шапка «единый контракт анализатора» в api-contracts для честной кросс-ссылки DM/EV. Вариант А: правки в владельце `iva-analysis-base`, зеркало `iva-fr-analyst` байт-в-байт → обе роли (обучение + design-capable).
**Границы/прод-safe.** Аддитивно: as-is флоу и Часть 2 не тронуты; валидатор границы основание держит безусловным; parity зелёный без allowlist.
**Файлы (15):** data-model-analyzer/SKILL.md ×2, events-analyzer/SKILL.md ×2, fr-authoring/SKILL.md ×2, api-contracts-discovery/SKILL.md ×2, commands/update-feature ×2, _mirrors.yaml, manifest.yaml ×2, CHANGELOG.md ×2.
**Тестирование:** check_mirror_sync 64 · version-discipline 48 clean · pytest 290 passed · parity без allowlist. critic §2 + controller-гейт PASS.

## Порядок доставки
Разводка очереди мержа с lead-modes (ГД) → fidelity-сверка ГД → sync/ребейз если lead-modes мержится первым → президент push+PR.

## Связано
[[critic-us5]] · [[gate-us3-us5]] · [[impl-us3-dm-ev]] · [[impl-us5-update-feature]] · [[plan-tz3-us-breakdown-detailed]]
