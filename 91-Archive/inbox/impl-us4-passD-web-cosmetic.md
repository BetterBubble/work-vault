---
status: draft
task: ТЗ#3 US#4 D-web — косметический фикс критика
worktree: /Users/bubblemac/tacticum-worktrees/us4-passD-web
branch: feat/us4-passD-web
permalink: tacticum/00-board/impl-us4-pass-d-web-cosmetic-1
archived-at: 2026-08-03 11:16
---

# impl-us4-passD-web — косметика (mail→web путь в примере К-2)

## Что сделано
Одна минимальная правка примера-таблицы К-2 в web pin. Убран mail-специфичный путь,
перенесённый из mail-профиля, приведён к web-конвенции Nx (`libs/<domain>/…`, уже
используемой в этом же файле, строка 23). Смысл примера не меняется, версия 0.4.1
не тронута, CHANGELOG не тронут, логика К-2/3/4 не тронута.

Файл: `templates/iva-web-brownfield/ingredients/skills/pin-authoring/SKILL.md` (строка 168)

Точная diff-строка (было/стало):
```
- | DM-2  | data model §1.7     | libs/mail/models/draft.model.ts     | расхождение  |
+ | DM-2  | data model §1.7     | libs/<domain>/models/draft.model.ts | расхождение  |
```
Выравнивание таблицы сохранено (ширина ячейки та же, правый pipe на месте).
Концепт «draft» оставлен намеренно — он согласован с соседней строкой CT-3
(`data-access sendDraft() (@ivcs)`), это web-термин, не mail.

## Проверка остальных примеров (grep 'libs/mail|libs/rn|react-native|kmp|ios')
В scope pin/tests/brd/start-task — только этот один mail-путь. Прочие совпадения — легитимные:
- `tests-authoring/SKILL.md:141` — `draft.model.spec.ts` — уже web-нейтральный (без `libs/mail`), не трогал.
- KMP/RN/mail в README/manifest/webrtc/openapi-codegen/local-skill-authoring — легитимные
  cross-surface контракты и упоминания родительского KMP-репо, НЕ примеры-пути. Не трогал.

## Кандидат ВНЕ scope (не трогал, на решение лида)
`ingredients/commands/run-tester.md:11` — в примере аргументов команды осталось
`UC-MAIL-042`, `libs/mail` (перенос из mail-профиля). Это command-файл, вне явного
grep-scope (pin/tests/brd/start-task), поэтому не расширял объём молча. Если нужно
причесать под web — отдельным касанием (напр. `UC-ORDERS-042`, `libs/orders`).

## Коммит
- amend в d0572c3 → новый HEAD **80607bc**, тело не менялось (`--amend --no-edit`).
- `git show --stat HEAD`: 6 файлов, +265/-4 (изменение внутри уже существовавшего diff pin SKILL.md).

## Тесты
- version-discipline `--diff-against origin/main`: **OK — 48 profile(s) clean**.
- pytest catalog (manifest_schemas + iva_role_presets + role_install_smoke): **206 passed** (3.75s).

Не мержил, не пушил.