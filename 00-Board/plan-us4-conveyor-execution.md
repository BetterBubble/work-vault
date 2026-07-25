---
status: draft
role: lead-fr
topic: ТЗ#3 US#4 — execution-план dev-конвейера К-1…К-5 (окно-сериализованный)
repo: /Users/bubblemac/tacticum/tacticum-dev
date: 2026-07-24
permalink: tacticum/00-board/plan-us4-conveyor-execution
---

# US#4 — execution-план (по факт. модели ADR-0056, не mirror)

База: [[explore-us4-conveyor-scope]]. Модель доставки (ТЗ-коррекция принята ГД): правка канонического владельца `tacticum-dev-base` + ручной синк монолитов; композиты (ios/firebird) наследуют. НЕ mirror-байт. Дифф ГД перед КАЖДЫМ push. autonomy off.

## К-items → файлы → зависимость
- **К-1** brd читает §1.4/§1.5 (FT/UC в Части 1), v1/v2 по `fr_skeleton`. Файл: `brd-authoring`. US#1 в main (handoff-контракт `fr-authoring:431`) → **можно сразу**.
- **К-5** маркер `fr_skeleton:2`, нет=v1 (backward-safe). Тот же `brd-authoring` (+ start-task). **можно сразу** (brd-часть).
- **К-2** наследование серий: brd ссылается (CT/DM/EV) · pin реализует · tests «Covers: CT-n» · report статус. CT-n (US#2)✅ + DM/EV (US#3 бандл смержен)✅ → **разблокировано полностью**. Файлы: brd (ссылки) + per-stack pin/tests.
- **К-3** гейт D-n → BLOCKED. Файл: `start-task`/run-implementation. ⚠️ **/start-task → ПОСЛЕ мержа добора lead-modes** (ГД).
- **К-4** таблица расхождений FR↔KB на проектные разделы. Файл: per-stack `pin-authoring` (verification).

## Окна (сериализация ГД)
| Профиль | Статус | Когда |
|---|---|---|
| `tacticum-dev-base` (канон brd/start-task) | нет активного лида | brd-часть (К-1/2/5) СРАЗУ; start-task (К-3/5) — после добора lead-modes |
| `iva-brownfield-mail`, `iva-rn-brownfield` | независимо | синк brd + own pin/tests свободно |
| `iva-ios-brownfield`, `firebird-web-brownfield` | композиты (brd наследуют) | own pin/tests (К-2/К-4) свободно; brd придёт из базы |
| `iva-web-brownfield` | shared lead-ds (активен Сц.1/2) | **ПОСЛЕ lead-ds** (ГД сериализует) |
| `iva-kmp-brownfield` | двойная контенция modes+ds, brd diverged | **ОКНО ГД** — отдельным проходом после стабилизации axis-2 |
| `iva-analysis-base` | территория lead-fr | own brd/pin/tests — внутри лейна, окно не нужно |
| go-backend / brownfield-task-workflow | deprecated/frozen | **НЕ трогать** |

## Порядок исполнения (окно-сериализованный)
1. **Проход A (СЕЙЧАС, безопасно):** К-1/К-5(brd)/К-2(brd-ссылки) в каноническом `tacticum-dev-base/brd-authoring`. Не-/start-task, не-контестед. → дифф ГД → push (первый в очереди на tacticum-dev-base).
2. **Проход B:** синк brd-изменений А в независимые монолиты (mail, rn) + analysis-base(own) + К-2(pin/tests)/К-4 для их стеков. Композиты наследуют brd автоматически.
3. **Проход C (после добора lead-modes):** К-3/К-5(start-task) в tacticum-dev-base + монолиты.
4. **Проход D (окно lead-ds):** синк в iva-web-brownfield.
5. **Проход E (окно ГД kmp):** iva-kmp-brownfield (brd diverged — аккуратный merge после axis-2).
Каждый проход: implementer → батарея (critic/controller где применимо) → дифф+версия ГД → окно/очередь ГД → push → веха.

## Связано
[[explore-us4-conveyor-scope]] · [[plan-tz3-us-breakdown-detailed]]
