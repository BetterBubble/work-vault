---
status: draft
role: critic
topic: ТЗ#3 US#4 B2 — pin/tests К-2/К-4 по 4 профилям
repo: /Users/bubblemac/tacticum-wt/us4-passB2 (feat/us4-passB2)
date: 2026-07-24
permalink: tacticum/00-board/critic-us4-passB2
---

# critic US#4 B2 — вердикт: (а) все 4 профиля верны ТЗ §4, блокеров нет

Upstream BRD→PIN→TESTS цела во всех 4 (mail/rn own brd с К-2; ios/firebird наследуют канон). rn — эталон согласования.

## По профилям
- **mail** — К-2/К-3/К-4 pin ОК (Qt/vmime), tests Covers CT-n ОК. Замечания: (i) tests не определяет статус для PIN-`расхождение` члена; (ii) косметика — дубль нумерации «5.» в Document structure.
- **rn** — ОК, замечаний нет. Эталон: tests-вокаб включает `расхождение`=xfail-тест (К-4→tests трассировка закрыта).
- **ios** — К-2/К-3/К-4 ОК (лучшая стек-честность: fragile не мокается, pure-mapper contract-тест Covers CT-n). Доп. статус `smoke-only` оправдан (не слабее ТЗ). Замечание: tests не определяет статус для PIN-`расхождение`.
- **firebird** — К-2/К-3/К-4 ОК (JUMP-decoder). Замечания: (i) pin статус-токены на английском `{implemented,discrepancy,blocked}` vs русские ТЗ; (ii) tests не определяет статус для PIN-`расхождение`.

## Сверка К-2/К-3/К-4 — внесены во всех 4
- К-2 pin (реализовать/расхождение по стабильному ID + статус), К-3 (blocked без D-n, не имитация), К-4 (расхождения CT+DM, kb_verify_api_exists, критично; корректно НЕ раздуто на EV), К-2 tests (Covers: CT-n + статус). Backward-safe явно во всех 8 файлах. Не строже/не слабее ТЗ.

## Уточнения консистентности (не блокеры) — ПРИМЕНЯЮТСЯ
1. **firebird pin** — статус-токены → русские ТЗ `{реализован, расхождение, blocked}` (единый словарь для report/контролёра).
2. **mail/ios/firebird tests** — добавить правило: PIN-`расхождение` член → статус в coverage-таблице (xfail-тест фиксирует ожидаемый контракт), по образцу rn. Закрывает трассировку К-4→tests.
3. **mail tests** — перенумеровать дубль «5.», contract-tests как rn (item 5).
→ отправлено фикс-implementer'у в сборку B2.
