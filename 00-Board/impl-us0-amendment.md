---
status: draft
role: implementer
task: ТЗ#3 US#0 — АМЕНДМЕНТ по вердикту critic
worktree: /Users/bubblemac/tacticum/tacticum-dev-fr-contracts
branch: feat/fr-analyst-contracts
date: 2026-07-24
permalink: tacticum/00-board/impl-us0-amendment
---

# US#0 амендмент — decouple ещё 2 ингредиента + frozen-owner норма

Дополнил существующий US#0-коммит по вердикту critic и пред-одобрению ГД.
Амендмент сделан через `git commit --amend` в единый чистый US#0-коммит
(история без мусора, один малый PR). Ветка НЕ пушена.

## Что сделано (человеческим языком)

1. **Норма `_mirrors.yaml` — оговорка frozen-owner.** В шапку (текст НОРМЫ, после
   строк про «объясни в CHANGELOG владельца») добавлена фраза: «Если владелец
   заморожен (deprecated/frozen прод) — запись делается в CHANGELOG ЗЕРКАЛА,
   CHANGELOG владельца НЕ трогаем.» Закрыт БЛОКЕР: агент, читающий норму, при
   frozen-owner больше не полезет в CHANGELOG замороженного владельца.

2. **Ещё 2 ингредиента выведены из первой пары.** Из пары
   `iva-analysis-base ↔ iva-fr-analyst` дополнительно удалены
   `design-system-discovery` и `mockup-authoring`. В первой паре осталось
   РОВНО ОДИН зеркало — `start-feature` (не тронут). Другие 5 пар не тронуты.
   Контент самих навыков НЕ менялся — только разъединение зеркала.

3. **ADR-0059 §7.** Счётчик 59 → **57** (пересчитан фактически скриптом: baseline
   62 − выведено 5 = 57). Абзац расхождения обновлён с трёх до пяти ингредиентов;
   добавлено, что `design-system-discovery`/`mockup-authoring` формируют раздел
   UI §1.9 Части 1 и правятся в US#1 (переезд UI-требований П.H → §1.9);
   `start-feature` остаётся зеркалом (правок не требует).

4. **CHANGELOG `iva-fr-analyst` [0.1.11].** Запись обновлена с трёх до пяти
   ингредиентов (+ причина: правятся в US#1, раздел §1.9). Версия 0.1.11 не
   менялась (та же запись). Отступление frozen-owner в записи сохранено.

## Проверки (свежий прогон)

| Проверка | Результат |
|---|---|
| `check_mirror_sync.py` | **OK — 57** зеркальных ингредиентов в 6 парах синхронны. exit 0. Число = ADR §7 (57). |
| `pytest test_role_replacement_parity.py --noconftest -q` | **77 passed**, 0 failed (блок A меньше на 2 кейса, блок B parity зелёный — новых ингредиентов не добавляли). |
| `check_profile_version_discipline.py --diff-against main` | **OK — 46 profile(s) clean.** exit 0. |

(Запуск через `uv run --with pyyaml [--with pytest]` — в worktree нет системного pyyaml.)

## Git

`git log main..HEAD --oneline`:
```
192c90a feat(fr-analyst): decouple 5 ingredients from mirror pair + frozen-owner norm (ТЗ#3 US#0)
```

`git diff --stat main..HEAD`:
```
 docs/adr/0059-single-axis-process-lanes-and-role-packs.md |  4 +++-
 templates/_mirrors.yaml                                   |  7 ++-----
 templates/iva-fr-analyst/CHANGELOG.md                     | 22 ++++++++++++++++++++++
 templates/iva-fr-analyst/manifest.yaml                    |  2 +-
 4 files changed, 28 insertions(+), 7 deletions(-)
```
(`manifest.yaml` +2/−1 — бамп версии до 0.1.11 из исходного US#0-коммита, в амендменте не трогался.)

## Явные инварианты

- **owner `iva-analysis-base` НЕ тронут** — 0 файлов владельца в диффе.
- **`start-feature` остался зеркалом** — единственный ингредиент первой пары, не выведен.
- Контент навыков `design-system-discovery`/`mockup-authoring`/`fr-authoring`/`api-contracts-discovery` НЕ правился (это US#1–2).
- НЕ push / НЕ PR / НЕ merge — ветка локальная.