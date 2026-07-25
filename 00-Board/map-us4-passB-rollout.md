---
status: draft
role: lead-fr
topic: ТЗ#3 US#4 Проход B — карта раскатки монолитов + pin/tests (для апрува ГД ПЕРЕД правкой)
repo: /Users/bubblemac/tacticum/tacticum-dev
date: 2026-07-24
permalink: tacticum/00-board/map-us4-passB-rollout
---

# US#4 Проход B — карта раскатки (ПЕРЕД правкой, апрув ГД)

Проход A (канон `tacticum-dev-base/brd-authoring` 0.2.6) смержен в main. Проход B = раскатка К-1/К-5/К-2-brd в монолиты + К-2/К-4 в per-stack pin/tests. Модель НЕ mirror (ручной синк). autonomy off, дифф+версии ГД перед push.

## Предлагаю СПЛИТ Прохода B на две части (чистота + параллель)

### B1 — синк brd-authoring (механический, безопасные профили)
Копирование канонических изменений brd (К-1/К-5/К-2) из `tacticum-dev-base` в собственные копии:
| Профиль | brd сейчас | Действие | Bump | Окно |
|---|---|---|---|---|
| `iva-brownfield-mail` 0.7.1 | own (был byte-identical канону) | синк brd К-1/5/2 | 0.7.1→0.7.2 | нет активного лида — **свободно** |
| `iva-rn-brownfield` 0.5.1 | own (byte-identical) | синк brd К-1/5/2 | 0.5.1→0.5.2 | свободно |
| `iva-analysis-base` (0.1.x) | own (лейн аналитика, lead-fr) | ⚠️ **вопрос ГД:** синкать ли? это профиль аналитика, не dev-конвейер; его brd byte-identical канону. Синк для консистентности ИЛИ оставить (не dev-потребитель FR)? | по решению | lead-fr территория |
| `iva-web-brownfield` | own | **НЕ трогаю** — Проход D, окно lead-ds | — | D |
| `iva-kmp-brownfield` | own (diverged axis-2) | **НЕ трогаю** — Проход E, окно ГД | — | E |
| `iva-ios/firebird-brownfield` | **inherit** (нет own brd) | ничего — наследуют канон из базы автоматически | — | — |
| `iva-go-backend-brownfield`, `brownfield-task-workflow` | deprecated | **НЕ трогать** | — | — |
Параллель: mail и rn — независимые файлы → могу гнать 2 implementer'а параллельно.

### B2 — pin-authoring/tests-authoring К-2/К-4 (глубокий, per-stack)
К-2 (pin реализует серии CT/DM/EV, tests «Covers: CT-n») + К-4 (таблица расхождений FR↔KB на проектные разделы, kb_verify_api_exists). pin/tests genuinely per-stack (все разные).
| Профиль | own pin | own tests | Действие К-2/К-4 | Окно |
|---|---|---|---|---|
| `iva-brownfield-mail` | own | own | pin: наследовать CT/DM/EV + FR↔KB; tests: Covers CT-n | свободно |
| `iva-rn-brownfield` | own | own | то же под rn-стек | свободно |
| `iva-ios-brownfield` | own | own | то же под ios | свободно (композит, brd inherit) |
| `firebird-web-brownfield` | own | own | то же под firebird | свободно |
| web/kmp pin/tests | own | own | **НЕ трогаю** — окна D/E | D/E |
Параллель: 4 независимых профиля (mail/rn/ios/firebird) — по стеку, независимые файлы → параллельные implementer'ы.

## Пересечение окон (сверка ГД)
- mail/rn/ios/firebird — по explorer нет активного конкурирующего лида → **свободно, можно параллелить**.
- web-brownfield — lead-ds активен (Сц.1/2) → **Проход D, после ds** (не в B).
- kmp-brownfield — двойная контенция + diverged → **Проход E, окно ГД** (не в B).
- go-backend/brownfield-task-workflow — deprecated → не трогаю.
- start-task (К-3/К-5-start-task) — **Проход C** (канон tacticum-dev-base + монолиты; не заблокирован по lead-modes, отдельно).

## Прошу у ГД
1. Апрув карты B1+B2 (профили/бампы/окна) ПЕРЕД правкой.
2. Решение по `iva-analysis-base` brd-синку (синкать/оставить).
3. ОК на параллель implementer'ов по независимым профилям (mail/rn для B1; mail/rn/ios/firebird для B2).
Порядок: B1 (brd синк) → B2 (pin/tests) → C (start-task) → D (web) → E (kmp). Каждый: батарея → дифф+версии ГД → push по отмашке.

## Связано
[[plan-us4-conveyor-execution]] · [[explore-us4-conveyor-scope]]
