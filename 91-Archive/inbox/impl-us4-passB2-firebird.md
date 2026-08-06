---
status: draft
role: implementer
topic: US#4 Проход B2 — pin/tests К-2/К-4 для firebird-web-brownfield
for: lead-fr
repo: /Users/bubblemac/tacticum/tacticum-dev
worktree: /Users/bubblemac/tacticum-wt/us4-passB2-firebird
branch: feat/us4-passB2-firebird
date: 2026-07-24
permalink: tacticum/00-board/impl-us4-passB2-firebird-1
archived-at: 2026-08-03 11:16
---

# US#4 B2 — firebird-web-brownfield (own pin/tests) — готово, НЕ push

Спека: [[spec-us4-passB2-pintests]]. Профиль own pin/tests подтверждён (оба файла существуют, не наследуются). brd/start-task/base НЕ трогал.

## Что сделано (под firebird-стек)

Правил ТОЛЬКО 4 файла профиля. Аддитивно, backward-safe: на v1-FR (нет серий CT/DM/EV) оба скилла авторятся как раньше.

**pin-authoring `SKILL.md`:**
- **К-2** — новый раздел «Project sections — CT-n / DM-n / EV-n (v2 FR only)» + пункт 12 в структуре документа. Каждый член серии реализуется по стабильному ID, замаплен на firebird-стек:
  - `CT-n` (§1.6 контракт) → JUMP-контракт: schema + `createBackendDecoder(schema)` + `Either`-граница (в `@firebird/types`/`*-core`/`util`). JUMP-граница компилятором невидима: если backend-schema ещё нет на стенде — член `blocked` (серверная сторона), честно, live-stand smoke.
  - `DM-n` (§1.7 модель/состояния) → `@firebird/types` структуры + RTK slice state machine (async-thunk factory, `build*Actions`, `Safe`-Option-селекторы). Матрица операции×роли/машина состояний → slice states + селекторы.
  - `EV-n` (§1.8 события/консистентность) → dispatched actions / порядок thunk-fulfilment, идемпотентность стейта, `Match` по tagged-union. Серверный push-event → external, не планируется здесь.
  - **Report статус:** таблица `## Project sections status` (`implemented`/`discrepancy`/`blocked`) по стабильному ID.
- **К-3** — раздел «D-n design gate»: раздел планируется `implemented` только если `D-n` утверждён (гейт на входе `/start-task`, Проход C-канон). Неутверждён/отсутствует → `blocked` (Evidence `pending D-n`), не имитируем.
- **К-4** — расширил существующую таблицу `## API Verification` (FR↔код) на проектные разделы: `CT-n`/`DM-n` сверяются с кодом через `kb_verify_api_exists` + firebird-аналоги (чтение JUMP-schema, `@firebird/types`, decoder-сайт через `kb_get_code_context`). Конфликт проект↔реальность = **критичное расхождение** (`CONFLICT`), выводится явно, не «сглаживается» под код. Добавлена колонка статуса `EXISTING/NEW/PHANTOM/CONFLICT`.
- + правила и анти-паттерны (verbatim-ID, не имитировать при неутв. D-n, конфликт = критично).

**tests-authoring `SKILL.md`:**
- **К-2** — новый раздел «Contract & project-section coverage (v2 FR only)» + пункт 6 в структуре. Контрактные тесты `Covers: CT-n` = **чистый тест декодера на JUMP-фикстуре** (`Either` decode) — точно ложится в firebird-политику (тестируем pure-декодер, session НЕ мокаем, no e2e). Живой round-trip остаётся ручным live-stand smoke.
  - `DM-n` покрыт через pure-тесты mapper/`Safe`-селекторов/переходов reducer'а; `EV-n` — через reducer/action-sequence (идемпотентность + порядок) и `Match`-ветки.
  - **Report статус:** таблица `## Project-section coverage` (`covered`/`blocked`/`gap`) по стабильному ID.
- + правила и анти-паттерны (verbatim-ID; `CT-n`-тест НЕ мокает `session.request` — только фикстура).

**Версия:** `manifest.yaml` 0.1.2→0.1.3 + `CHANGELOG.md` секция `[0.1.3]`.

## Проверки
- `check_profile_version_discipline.py --diff-against origin/main` → **OK — 48 profile(s) clean**.
- `pytest test_firebird_web_brownfield_profile.py + test_manifest_schemas.py` (PYTHONPATH=apps/backend, --noconftest) → **51 passed**.
- Полный `tests/catalog/` под `--noconftest`: статические тесты зелёные; DB-backed (`test_tacticum_init`, `test_seed_*`, `role_replacement_parity`) дают ERROR — инфраструктура (нет Postgres:5432 + фикстура `db_session` недоступна при `--noconftest`), НЕ связано с правками контента.
- Известный унасл. red iva-role-web (#149 ds) — не всплыл в моём наборе.

## git diff --stat (origin/main..HEAD)
```
 templates/firebird-web-brownfield/CHANGELOG.md                          | 28 +++++
 .../firebird-web-brownfield/.../skills/pin-authoring/SKILL.md           | 72 ++++++++
 .../firebird-web-brownfield/.../skills/tests-authoring/SKILL.md         | 48 +++-
 templates/firebird-web-brownfield/manifest.yaml                        |  2 +-
 4 files changed, 148 insertions(+), 2 deletions(-)
```
Коммит: `099b44d feat(us4-B2): firebird pin/tests implement & cover CT-n/DM-n/EV-n project series`. Ветка ahead 1, НЕ push.

## Развилки / заметки (durably)
- **«Существующая таблица расхождений FR↔код» (К-4)** в firebird pin = раздел `## API Verification` (EXISTING/NEW/PHANTOM). Отдельной «таблицы расхождений» в base/firebird pin нет — расширил именно её (добавил статус `CONFLICT` + строки проектных разделов). Если лид имел в виду иную таблицу — скажи, поправлю.
- **`D-n` гейт (К-3)** в dev-base ingredients пока не материализован (Проход C-канон ещё не в main) — сослался на него как на будущий гейт `/start-task`, pin честно репортит `blocked`/`pending D-n`. Аддитивно, ничего не ломает.
- venv для проверок брал из основного дерева (`apps/backend/.venv`) — в worktree своего venv нет; правки только в файлах worktree.

## Связано
[[spec-us4-passB2-pintests]] · [[map-us4-passB-rollout]]