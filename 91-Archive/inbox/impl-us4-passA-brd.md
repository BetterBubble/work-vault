---
status: draft
type: report
task: US#4 Проход A — dev-конвейер учится читать FR v2 (brd-authoring)
tz: ТЗ#3 §4 (К-1/К-5/К-2-brd)
for: lead-fr
from: implementer
date: 2026-07-24
permalink: tacticum/00-board/impl-us4-pass-a-brd-1
archived-at: 2026-08-03 11:16
---

# impl US#4 Проход A — brd-authoring читает канонический FR v2

## Скоуп прохода (как задано)
Только КАНОНИЧЕСКИЙ владелец brd-authoring в `tacticum-dev-base`. Монолиты
(web/kmp/mail/rn), start-task, pin, tests, analysis-base — НЕ трогал (следующие
проходы/окна). Конвейер-скилл НЕ в _mirrors.yaml — синка не требует.

## Worktree и ветка
- Репо: `/Users/bubblemac/tacticum/tacticum-dev`
- Worktree: `/Users/bubblemac/tacticum-wt/us4-conveyor-brd`
- Ветка: `feat/us4-conveyor-brd` от свежего `origin/main` (`384997f`, бандл
  US#3+US#5 уже в main — handoff-контракт `fr_skeleton: 2` в fr-authoring на месте)
- Коммит: `81de699`
- НЕ push / НЕ PR / НЕ merge (autonomy off, серверы read-only). Без AI-подписей.

## Что сделал (человеческим языком)
Правил ОДИН скилл-файл + версию/CHANGELOG (3 файла всего). brd-authoring теперь
умеет читать новую раскладку FR, которую выдаёт аналитик:

- **К-5 (распознавание версии по маркеру `fr_skeleton`).** Добавлен раздел
  «Input: reading the FR» — скилл СНАЧАЛА смотрит маркер `fr_skeleton` в шапке FR:
  - `fr_skeleton: 2` → новая раскладка;
  - маркера НЕТ → v1 (старая раскладка). Читать по-старому.
  Обе версии поддержаны явно — backward-safe.
- **К-1 (откуда берутся FT/UC).** В v2 конвейер наследует `FT-n` из **§1.4** и
  `UC-n` из **§1.5** Части 1 (Постановка), а НЕ из Приложения (в v2 Приложение —
  только as-is факты, старых П.A/П.B там нет). В v1 — по-прежнему из **П.A/П.B**.
  Идентификаторы наследуются как есть, без перенумерации (стабильность = сквозная
  трассировка до автотестов).
- **К-2 (brd-часть: наследование проектных серий).** В v2 brd РЕГИСТРИРУЕТ в
  постановке для dev проектные серии из Части 1 — `CT-n` (§1.6 контракты),
  `DM-n` (§1.7 модель данных), `EV-n` (§1.8 события) — ссылками по стабильному ID,
  чтобы вниз подхватили pin (реализует) и tests (покрывает) — это следующие
  проходы, здесь только brd-ссылки. В v1 таких серий нет — шаг пропускается.
- Правила (Rules) и анти-паттерны дополнены: читать FT/UC по маркеру; не читать
  FT/UC из Приложения на v2; не перенумеровывать FT/UC/CT/DM/EV.
- Bump `tacticum-dev-base` `0.2.5 → 0.2.6` (manifest) + запись в CHANGELOG
  (человеческим языком, [0.2.6] — 2026-07-24).

## Backward-совместимость v1 (подтверждаю)
FR без маркера читается по-старому — FT/UC из П.A/П.B, без ожидания §1.4/§1.5 и
CT/DM/EV. Существующие опубликованные FR не ломаются; при malformed-маркере скилл
трактует как v1 и помечает допущение. Существующий brd-флоу (структура документа,
KB-грундинг, НФТ-подраздел) не тронут.

## Результаты проверок (свежий прогон)
- `check_profile_version_discipline.py --diff-against origin/main` →
  **OK — 48 profile(s) clean.**
- `check_mirror_sync.py` → **OK — 64 зеркальных ингредиента в 6 парах синхронны**
  (число не изменилось — brd-authoring не в mirror, ничего не сломано).
- Целевые catalog-тесты (из `apps/backend`, env
  `uv run --with pyyaml --with pytest --with jsonschema --with pydantic`):
  `test_manifest_schemas.py` + `test_iva_role_presets.py` +
  `test_role_install_smoke.py` → **206 passed** (72+72+62), 0 failed.
  (parity не гонял — не относится: конвейер-скилл не в mirror.)
- `git diff --stat origin/main..HEAD` → ровно **3 файла**: SKILL.md (+47/−1),
  CHANGELOG.md (+15), manifest.yaml (+1/−1).
- `git status` → чисто (`.venv` от uv в gitignore, в индекс не попал).

## Развилки / примечания (durable)
- **Формулировка «читает ТЗ» уточнена на «читает FR».** В шапке скилла заменил
  «from ТЗ» на «from the requirement (ТЗ / FR)» и добавил, что вход — опубликованный
  FR (выход fr-authoring). Терминология согласована с fr-authoring; на поведение
  v1 не влияет.
- **start-task не трогал**, хотя именно он передаёт ТЗ в brd (аргумент `$ARGUMENTS`).
  Он абстрактен к структуре FR и правки не требует для этого прохода; если lead
  захочет явно упомянуть `fr_skeleton` в start-task — это отдельный проход (в
  scope US#4 не заявлено).
- В `tacticum-dev-base` до этого прохода не было НИ ОДНОГО упоминания
  `fr_skeleton`/серий CT/DM/EV — brd стал первым скиллом конвейера, читающим v2.
  Следующие проходы (pin/tests) должны опираться на серии, которые brd уже
  зарегистрировал.