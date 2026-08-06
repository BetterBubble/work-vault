---
status: draft
from: implementer
to: lead-fr
task: ТЗ#3 US#4 E-remainder — точечный фикс критика (v1-локация FT/UC в kmp start-task)
branch: feat/us4-passE-remainder
head: 9829942
date: 2026-07-24
permalink: tacticum/00-board/impl-us4-eremainder-v1fix-1
archived-at: 2026-08-03 11:16
---

# impl: US#4 E-remainder — фикс v1-локации FT/UC (kmp start-task)

## Что сделано (одна хирургическая правка)
Файл: `templates/iva-kmp-brownfield/ingredients/commands/start-task.md`, v1-ветка блока К-5 (около §63-65).

Строка была дословно скопирована из канонического start-task и описывала канон-специфичный
layout (Приложение П.A/П.B), что противоречит kmp-brd (тот же профиль): в kmp v1 = плоское
чтение `fr.md`, канон-П.A/П.B не навязывается. Переформулировал ТОЛЬКО v1-локацию FT/UC под kmp.

### Было → Стало
**Было:**
```
- **marker absent** (v1, legacy layout) → old flow: `FT-n`/`UC-n` live in the
  Приложение (П.A/П.B), there are **no** §1.6/§1.7/§1.8 project sections and **no
  D-n gate**. Author exactly as before.
```
**Стало:**
```
- **marker absent** (v1, legacy layout) → old flow: `FT-n`/`UC-n` read flat from
  `fr.md` as before (KMP current), there are **no** §1.6/§1.7/§1.8 project sections
  and **no D-n gate**. Author exactly as before.
```

### Сверка с эталоном
kmp-brd `SKILL.md` (brd-authoring, §23-25): v1 → «читай `FT-n`/`UC-n` из `fr.md` как раньше
(плоское чтение, без §1.4/§1.5 и без серий CT/DM/EV) — существующее поведение KMP сохраняется».
Формулировка start-task теперь согласована с brd: «read flat from `fr.md` as before (KMP current)».

## Границы соблюдены
- v2-ветка — не тронута.
- Гейт-блок К-3 (plate-based D-n) — не тронут (дословно == канон, так и задумано).
- kmp-стек-специфика (cross-repo, no-sub-agent, step-0, Jira/Confluence fetch) — не тронута.
- Версия 0.5.2 — без изменений. CHANGELOG — не трогал (это уточнение формулировки уже-описанного).

## Коммит
- amend в c39ed5e (тело коммита не менялось). Новый HEAD: **9829942**.
- `git show --stat HEAD`: 4 files changed, 98 insertions(+), 2 deletions(-)
  - CHANGELOG.md | 27 ++
  - ingredients/commands/start-task.md | 36 ++
  - ingredients/skills/brd-authoring/SKILL.md | 35 ++
  - manifest.yaml | 2

## Проверки
- version-discipline `--diff-against origin/main`: **OK — 48 profile(s) clean.**
- pytest catalog (test_manifest_schemas + test_iva_role_presets + test_role_install_smoke):
  **206 passed in 3.28s** (прогон через `apps/backend/.venv`, т.к. общий conftest тянет
  alembic/pytest_asyncio, которых нет в ad-hoc uv-окружении).

## Не мержил, не пушил — ждёт ревью.