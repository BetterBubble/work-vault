---
status: draft
role: lead-fr
topic: ТЗ#3 US#4 — сводка «сделано vs ТЗ §4» (конвейер разработки читает FR v2)
repo: /Users/bubblemac/tacticum/tacticum-dev
date: 2026-07-25
permalink: tacticum/00-board/summary-us4-done-vs-tz4-1
archived-at: 2026-08-03 11:16
---

# US#4 — сводка «сделано vs ТЗ §4»

**Цель ТЗ §4:** конвейер разработки (brd → pin → tests → start-task) должен читать FR v2 (маркер `fr_skeleton: 2`, проектные разделы §1.4 FT-n / §1.5 UC-n + серии CT-n/DM-n/EV-n + плашки «требует утверждения: разработчик + CTO» / Q-n / D-n), наследовать проектирование аналитика и гейтить неутверждённое. Backward-safe: FR v1 читается как раньше.

## Требования ТЗ §4 (К-1…К-5) — статус

| К | Что требует ТЗ | Где реализовано | Статус |
|---|---|---|---|
| **К-1** | brd читает FT-n/UC-n из §1.4/§1.5 (v2) + детекция маркера; серии только ССЫЛАЕТ и передаёт в pin | канон brd (Проход A) → mail/rn (B1) → kmp (E) → web (D); композиты ios/firebird наследуют канон | ✅ все профили |
| **К-2** | pin реализует серии CT/DM/EV по стабильному ID + статус; tests `Covers: CT-n` | B2 (mail/rn/ios/firebird) + kmp (E-pintests) + web (D) | ✅ все профили |
| **К-3** | start-task: плашка+Q-n без D-n → честный BLOCKED (без имитации), plate-based, no fail-open | C-канон → C-монолиты (mail/rn) → kmp (E) → web (D); композиты наследуют канон | ✅ все профили |
| **К-4** | таблица расхождений FR↔KB на проектных разделах (CT+DM, kb_verify_api_exists; не EV) | pin: B2 + kmp + web | ✅ все профили |
| **К-5** | версия-осведомлённость (маркер `fr_skeleton`), backward-safe v1 | brd + start-task во всех профилях | ✅ все профили |

## Раскатка по профилям (модель: канон-owner + ручной синк монолитов; композиты наследуют)

| Проход | Профиль | Артефакт | PR / статус |
|---|---|---|---|
| A | tacticum-dev-base (канон) | brd К-1/5 | #148 в main |
| B1 | mail, rn (react-native) | brd К-1/5 | #150 в main |
| B2 | mail, rn, ios, firebird | pin/tests К-2/4 | #153 в main |
| C-канон | tacticum-dev-base | start-task К-3/5 | #152 в main |
| C-монолиты | mail, rn | start-task К-3/5 | #156 в main |
| E-pintests | iva-kmp-brownfield | pin/tests К-2/4 | #157 в main |
| E-remainder | iva-kmp-brownfield | brd К-1/5 + start-task К-3/5 | запушен, PR в очереди президенту |
| D-web | iva-web-brownfield | brd К-1/5 + pin/tests К-2/4 + start-task К-3/5 | батарея PASS (critic а + controller 7/7), HOLD push (окно ds PR-C) + ребейз 0.4.1→0.5.1 |
| — | ios, firebird (композиты) | brd + start-task | наследуют канон (ADR-0056 композиция) |

## Особенности реализации (для сверки)
- **kmp brd — diverged (axis-2)**: К-1/5 внесены аккуратным merge, axis-2/KMP-специфика (Jira-вход, D-n, серия (BRD), FR↔KB needs-info, grep self-check) MUST-KEEP сохранена дословно. v1-ветка мягкая (kmp-текущее плоское чтение, канон-П.A/П.B не навязан).
- **web brd — чистый синк**: был байт-в-байт == канон → взят канон verbatim (v1=канон-П.A/П.B, консистентно).
- **web/kmp start-task — diverged-вставка**: К-5/К-3 вставлены verbatim, стек-специфика (web: @ivcs/step-0/4-role; kmp: cross-repo/no-sub-agent) цела.
- **Гейт К-3 — plate-based**: триггер по плашке (не по номерам секций) → покрывает §1.6-1.8 И §1.9 UI И версия-независим (срабатывает даже при битом маркере). Идентичен во всех профилях (fidelity-by-identity).
- **Двухзонная честность (§2)**: строгая as-is в Части 2; проектные разделы Части 1 — проектные wire-имена под 3 гардрейлами (основание в Ч.2 + плашка/Q-n + boundary-валидатор). — это US#1-3 (в main), конвейер US#4 их ПОТРЕБЛЯЕТ.

## Проверки (на момент build-complete)
- version-discipline `--diff-against origin/main`: 48 profiles clean (каждый проход).
- check_mirror_sync: 64 зеркальных ингредиента в 6 парах синхронны.
- pytest catalog (schemas + role_presets + install_smoke): 206 passed (не-DB целевые; полный DB-CI — на сервере перед прод-сидом).

## Осталось до US#4 = 100%
1. D-web: батарея PASS ✅ (косметик-фикс пути в pin-примере в полёте) → HOLD push до окна ds (PR-C) → ребейз версии 0.4.1→0.5.1 → push → PR.
2. Мержи президента (утро): E-remainder + D-web PR.
3. Затем **доводка президента** (после всех US#4): fidelity-полнота всего ТЗ#3 → живой пилот на тест-стенде (реальный v2-FR → конвейер) → память → прод-сид (gated) + прод-verify.

## Связано
[[map-us4-remaining-to-prod]] · [[plan-us4-conveyor-execution]] · [[map-us4-passesDE]]