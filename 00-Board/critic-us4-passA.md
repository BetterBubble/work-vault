---
status: draft
role: critic
topic: ТЗ#3 US#4 Проход A — brd-authoring читает FR v2 (fidelity ТЗ §4)
repo: /Users/bubblemac/tacticum-wt/us4-conveyor-brd (feat/us4-conveyor-brd @ 81de699)
date: 2026-07-24
permalink: tacticum/00-board/critic-us4-passA
---

# critic US#4 Проход A — вердикт: (а) верно ТЗ §4, готово

Источник: ТЗ §4. Файл: tacticum-dev-base/ingredients/skills/brd-authoring/SKILL.md.

- **К-1 — ОК.** v2: FT-n из §1.4, UC-n из §1.5 Части 1 (НЕ Приложение); v1: П.A/П.B. ID verbatim, не переномеровано. Anti-pattern запрещает читать Приложение на v2. Точно по ТЗ §4 К-1.
- **К-5 — ОК.** Маркер `fr_skeleton` детектится ДО чтения: `2`→v2, нет→v1. Backward-совместимость явная («both supported, prod-safe»).
- **К-2 brd-часть — ОК.** brd регистрирует серии CT-n(§1.6)/DM-n(§1.7)/EV-n(§1.8) по стабильному ID для наследования вниз (pin реализует, tests покрывает — downstream, не здесь). v1: серий нет → шаг пропускается.
- **Не строже/не слабее; скоуп — ОК.** Ничего сверх ТЗ §4 (кроме edge-кейса ниже); start-task/pin/tests не тронуты (лишь упомянуты downstream).
- **Регресс — ОК.** v1-флоу дословно сохранён.

## 2 опциональные заметки (не блокеры)
1. **Битый маркер → v1 + пометка** (SKILL §33): этого кейса в ТЗ НЕТ (ТЗ: «отсутствие=v1»). critic: безопасный backward-совместимый дефолт, в духе ТЗ, не добавляет требований конвейеру, низкий риск. Keep допустимо. → **лид флагует ГД: keep (safe default) или remove (строго по ТЗ)?**
2. **Legacy-«ТЗ» generic-имя входа** в description/rules (§3/§74-75): читается консистентно с §11 (requirement=FR). Косметика уравнивания на «FR / ТЗ». Не блокер.

**Апрув Прохода A не держат.**
