---
status: draft
role: implementer
task: ТЗ#3 US#0 revert (ВАРИАНТ А доставки)
lead: lead-fr
branch: revert/us0-mirror-depreciation
worktree: /Users/bubblemac/tacticum/wt-revert-us0
base: origin/main (66bbc0d, US#0 = 192c90a PR#140)
date: 2026-07-24
permalink: tacticum/00-board/impl-us0-revert
---

# impl: откат US#0 (разъединение зеркала iva-analysis-base ↔ iva-fr-analyst)

Президент сменил модель доставки на **ВАРИАНТ А** (правим владельца
`iva-analysis-base`, зеркало следует). Откатил US#0 малым revert-коммитом,
оставив полезную frozen-owner оговорку в норме.

## Где сделано
- Worktree: `/Users/bubblemac/tacticum/wt-revert-us0`
- Ветка: `revert/us0-mirror-depreciation` (от `origin/main`)
- Коммит: `381f906`
- НЕ push / НЕ PR / НЕ merge (autonomy off, серверы read-only, всё локально).

## Правки (человеческим языком)

1. **`templates/_mirrors.yaml`** — в первую пару (owner `iva-analysis-base` /
   mirror `iva-fr-analyst`) вернул 5 выведенных ингредиентов, список снова
   полный (6), порядок как в исходнике до US#0 (сверено с `20412ff`):
   `api-contracts-discovery`, `design-system-discovery`, `fr-authoring`,
   `mockup-authoring`, `start-feature`, `update-feature`.
   **Оговорка frozen-owner в шапке нормы (строки 12–13: «Если владелец заморожен
   — запись в CHANGELOG зеркала, CHANGELOG владельца не трогаем») СОХРАНЕНА** —
   диф трогает только список ингредиентов, шапку нет.

2. **`docs/adr/0059-...md` §7** — счётчик `57 → 62` (истинный baseline, off-by-one
   фикс сохранён, НЕ откат на ошибочный 61). Удалён абзац о «первом осознанном
   расхождении ингредиентов внутри пары» (divergence-специфика US#0 — расхождения
   больше нет).

3. **`templates/iva-fr-analyst/CHANGELOG.md`** — удалена запись `[0.1.11]` про
   вывод 5 ингредиентов (divergence-специфика).

4. **`templates/iva-fr-analyst/manifest.yaml`** — версия `0.1.11 → 0.1.10`
   (откат bump'а US#0).

## Проверки (свежий прогон, `uv run --with pyyaml [--with pytest]`)

1. **`scripts/check_mirror_sync.py`** → **OK — 62 зеркальных ингредиентов в 6
   парах синхронны.** (exit 0). После возврата 5 ингредиентов в зеркало энфорсер
   снова требует байт-идентичности owner↔mirror — все 5 уже байт-идентичны на
   origin/main (US#0 тела скиллов не трогал), красных нет.

2. **`scripts/check_profile_version_discipline.py --diff-against origin/main`** →
   **OK — 46 profile(s) clean.** (exit 0). Понижение версии `0.1.11 → 0.1.10`
   **проходит** — version-discipline не запрещает его (проверяет факт изменения
   версии при изменении контента, а не монотонность возрастания).
   **Развилки по версии НЕ возникло** — выбран минимально-инвазивный вариант
   (чистый откат bump'а до 0.1.10), альтернатива с сохранением 0.1.11 не
   потребовалась.

3. **`pytest apps/backend/tests/catalog/test_role_replacement_parity.py
   --noconftest`** → **82 passed** (exit 0).
   - origin/main (US#0) даёт **77 passed** (проверено через stash).
   - Мой revert даёт **82 passed** = **+5** (5 возвращённых ингредиентов, каждый
     добавляет параметризованный байт-чек: 77 → 82).
   - 82 совпадает с pre-US#0 baseline (main tree `20412ff` = 82) — revert точно
     воспроизводит состояние до US#0.
   - ⚠️ Оценка лида в ТЗ была «79». Фактическая дельта +5 → **82**, а не 79.
     Все зелёные, падений нет; расхождение — только в ожидаемом числе из ТЗ.

## git

`git diff --stat origin/main..HEAD`:
```
 ...059-single-axis-process-lanes-and-role-packs.md |  4 +---
 templates/_mirrors.yaml                            |  5 +++++
 templates/iva-fr-analyst/CHANGELOG.md              | 22 ----------------------
 templates/iva-fr-analyst/manifest.yaml             |  2 +-
 4 files changed, 7 insertions(+), 26 deletions(-)
```

`git log origin/main..HEAD --oneline`:
```
381f906 revert(fr-analyst): откат разъединения зеркала iva-analysis-base↔iva-fr-analyst (ТЗ#3 US#0)
```

`git status`: clean (нет неотслеживаемых/незакоммиченных).

## Подтверждения
- **frozen-owner оговорка в норме (`_mirrors.yaml` шапка) — СОХРАНЕНА.**
- **Тела скиллов НЕ тронуты** (fr-authoring / api-contracts-discovery /
  design-system-discovery / mockup-authoring / start-feature / update-feature) —
  диф затрагивает только `_mirrors.yaml` / ADR / CHANGELOG / manifest.
- **Владелец `iva-analysis-base` НЕ тронут** (ни manifest, ни skills).

## На заметку лиду
- Развилки по версии не было — понижение 0.1.11→0.1.10 прошло чисто.
- Единственное отклонение от ТЗ — ожидаемое число pytest: ТЗ ждало 79, факт 82
  (+5, а не +2 от 77). Логика сходится (5 ингредиентов = +5 байт-чеков),
  оценка в ТЗ была неточна. Все тесты зелёные.