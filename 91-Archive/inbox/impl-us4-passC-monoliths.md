---
status: draft
task: US#4 Проход C-монолиты — раскатка канонического start-task К-3/К-5 в mail/rn
role: implementer
lead: lead-fr
date: 2026-07-24
permalink: tacticum/00-board/impl-us4-pass-c-monoliths-1
archived-at: 2026-08-03 11:16
---

# US#4 Проход C-монолиты — start-task К-3/К-5 в mail/rn

## ОБНОВЛЕНИЕ (после мержа C-канон+B2 в main) — ФИНАЛИЗАЦИЯ ребейзом

C-канон и B2 смержены в main (origin/main: tacticum-dev-base 0.2.7, mail 0.7.3, rn 0.5.3). Ветка `feat/us4-passC-monoliths` отребейзена на `origin/main`.

- **Новый HEAD:** `433ec95` — sync(us4-C): start-task гейт D-n (К-3) + fr_skeleton (К-5) в mail+rn монолиты. Один коммит поверх origin/main.
- **Rebase:** канон-коммиты уже в main (пропущены), реплеился только мой C-монолиты-коммит. start-task-правки применились **ЧИСТО** (mail/rn start-task в main никто не трогал). Конфликт был только в mail/rn CHANGELOG (ожидаемо, коллизия версии 0.7.3/0.5.3). manifest авто-смержился (обе стороны ставили одинаковое значение — не конфликт).
- **Разрешение конфликта версий:** manifest бампнут поверх B2 — **mail 0.7.4, rn 0.5.4**. CHANGELOG: обе записи сохранены хронологически:
  - `[0.7.4]` / `[0.5.4]` — моя C-монолиты (start-task гейт D-n + К-5);
  - `[0.7.3]` / `[0.5.3]` — B2 (pin/tests-authoring проектные серии), из main;
  - `[0.7.2]` / `[0.5.2]` — brd-authoring (ранее), без изменений.
- **РАЗВИЛКА ВЕРСИЙ ЗАКРЫТА:** финальные версии mail 0.7.4 / rn 0.5.4 (то, что раньше было отмечено как «ребейз после B2» — выполнено).

### Проверки после ребейза (свежий прогон)
- `git log origin/main..HEAD --oneline` → **1 коммит** (`433ec95`).
- `git diff --stat origin/main..HEAD` → **ровно 6 файлов** (mail/rn: start-task + manifest + CHANGELOG), +112/−2. БЕЗ C-канон/B2-файлов (они в main). git status clean.
- Версии: mail `0.7.4`, rn `0.5.4` (manifest).
- Гейт в start-task: grep `Approval gate|plate|fr_skeleton` → 12 совпадений в каждом (mail и rn). Стек-специфика `pin-ui-pipeline-check` сохранена (1 совпадение в каждом). mail start-task == rn start-task (identical).
- `diff` mail/rn start-task vs канон `tacticum-dev-base/start-task` (в дереве/main) → **единственное отличие — строка 73** (стек-скилл `pin-ui-pipeline-check` у монолитов; у канона его нет). Гейт-часть (К-3 plate-based + К-5) идентична канону.
- `check_profile_version_discipline.py --diff-against origin/main` → **OK — 48 profile(s) clean.**
- pytest `test_manifest_schemas.py` + `test_iva_role_presets.py` + `test_role_install_smoke.py` (PYTHONPATH=apps/backend, `--noconftest`) → **206 passed, 1 warning in 3.99s**. (Замечание: `--noconftest` на всей `apps/backend/tests/catalog/` даёт 34 collection-errors — отсутствуют conftest-фикстуры; это артефакт флага `--noconftest`, не регрессия. Целевые три файла коллектятся и проходят с `--noconftest` чисто.)

НЕ push. Ветка готова к мержу пользователем после ревью.

---

## (Исходный отчёт — до ребейза, для истории)


## Worktree и ветка
- Worktree: `/Users/bubblemac/tacticum-wt/us4-passC-monoliths`
- Ветка: `feat/us4-passC-monoliths` — **создана ОТ `feat/us4-passC-canon-starttask`** (стек на C-канон, как указал лид: канон нужен как эталон копирования; C-канон мержится первым, C-монолиты после).
- База проверена: содержит канонический start-task с plate-based гейтом (коммит `acb8268 feat(dev-base): start-task D-n approval gate + fr_skeleton version-awareness (US#4 К-3/К-5)`).
- Коммит работы: `6559cab`.
- НЕ push / НЕ PR / НЕ merge (autonomy off).

## Что синкнуто (человеческим языком)
Канонический start-task из `tacticum-dev-base` уже несёт два новых блока (К-3 гейт + К-5 версия-осведомлённость). В монолитах mail/rn этих блоков не было. Внёс их дословно в оба монолитных start-task, аккуратно вписав перед секцией «Target-dir resolution» (та же позиция, что в каноне):

1. **К-5 — осведомлённость о версии FR (`fr_skeleton`).** Перед авторингом команда читает маркер `fr_skeleton` из заголовка FR: `fr_skeleton: 2` → есть проектные разделы §1.6 контракты (`CT-n`) / §1.7 модель данных (`DM-n`) / §1.8 события (`EV-n`), они подпадают под гейт; маркер отсутствует (v1) → старый поток без гейта. Битый/неоднозначный маркер → трактуется как v1.
2. **К-3 — гейт утверждения `D-n` (plate-based, CRITICAL).** Срабатывает на **плашке** «Предложение, требует утверждения: разработчик + CTO» + открытый `Q-n` без зафиксированного `D-n` в П.D → честный **BLOCKED** (не выдумывать контракты/модель/события/UI, не симулировать реализацию, вернуть аналитику/овнеру). Триггер по плашке version-independent: покрывает §1.6–1.8, плашечный §1.9 UI (по ТЗ §2.2 п.3) и любые будущие проектные серии; срабатывает даже при битом маркере (no fail-open). FR v1 без плашек → гейт не применяется.

Файлы:
- `templates/iva-brownfield-mail/ingredients/commands/start-task.md` (+36)
- `templates/iva-rn-brownfield/ingredients/commands/start-task.md` (+36)

## Стек-специфика mail/rn start-task (durably)
- До правки mail и rn start-task были **byte-identical друг другу** (`diff` → identical), но отличались от канона **тремя вещами**: (а) отсутствовали блоки К-5 и К-3; (б) в списке design-скиллов у mail/rn есть дополнительный скилл **`pin-ui-pipeline-check`**, которого НЕТ в каноне (`tacticum-dev-base`). Это и есть стек-специфика монолитов (UI-конвейер).
- **`pin-ui-pipeline-check` сохранён** — вносил только дельту К-3/К-5, стек-обёртку не трогал.
- После правки единственное отличие mail/rn от канона — эта строка скиллов (`diff canon vs mail` показывает ровно одну изменённую строку 73, где у монолитов добавлен `pin-ui-pipeline-check`). mail и rn по-прежнему идентичны друг другу.
- Итог: mail/rn start-task теперь несут гейт D-n (К-3, plate-based) + К-5, как канон, но со своей стек-обёрткой (UI-pipeline-скилл).

## РАЗВИЛКА ВЕРСИЙ (durably — требует внимания лида при интеграции)
- **Бамп в этой ветке (относительно базы C-канон):** mail `0.7.2 → 0.7.3`, rn `0.5.2 → 0.5.3`. Моя база — C-канон, которая НЕ содержит прохода B2; там mail/rn на 0.7.2/0.5.2, поэтому бамп +1 патч относительно базы корректен и version-discipline зелёный.
- **⚠️ РЕБЕЙЗ ПОСЛЕ B2:** проход B2 (ещё не в main) уже бампит mail→0.7.3 / rn→0.5.3. C-монолиты мержится ПОСЛЕ B2. Значит при интеграции возникнет коллизия версий: мои 0.7.3/0.5.3 совпадут с версиями B2. **Лиду при интеграции нужно ребейзнуть версии этой ветки на mail `0.7.4` / rn `0.5.4`** (поверх B2) — и в manifest.yaml, и в заголовке CHANGELOG-секции. Тело CHANGELOG-записи (описание К-3/К-5) переносится как есть, меняется только номер.
- Порядок мержа по плану: C-канон → (B2) → C-монолиты. Я НЕ гадал финальную версию, зафиксировал развилку здесь.

## CHANGELOG
- `iva-brownfield-mail/CHANGELOG.md` — добавлена секция `[0.7.3] — 2026-07-24`.
- `iva-rn-brownfield/CHANGELOG.md` — добавлена секция `[0.5.3] — 2026-07-24`.
- Обе описывают синк start-task (К-3 гейт + К-5) и явно отмечают сохранение стек-специфики `pin-ui-pipeline-check`.

## Прод-safe
Backward-совместимо: v1-FR (нет проектных разделов/плашек) → гейт не срабатывает, авторинг как раньше. Стек-специфику start-task не ломал.

## Проверки (свежий прогон)
- **version-discipline** `--diff-against feat/us4-passC-canon-starttask` (стек-база): `OK — 48 profile(s) clean.` EXIT=0.
- **pytest целевые** (env `uv run --with pyyaml --with pytest --with jsonschema` + докинул `pytest-asyncio`/`alembic`/`sqlalchemy` для conftest, `PYTHONPATH=apps/backend`):
  - `test_manifest_schemas.py` + `test_iva_role_presets.py` + `test_role_install_smoke.py` → **206 passed in 4.04s**. Унаследованного red нет.
- **git diff --stat** (vs база C-канон): 6 файлов, +110 / −4.
- **git log**: `6559cab sync(us4-C): start-task гейт D-n (К-3) + fr_skeleton (К-5) в mail+rn монолиты`.
- **git status**: clean (всё закоммичено).

## НЕ трогал
tacticum-dev-base (канон), композиты ios/firebird (наследуют), web/kmp (окна), brd/pin/tests, другие профили. Только mail/rn start-task + их manifest/CHANGELOG.