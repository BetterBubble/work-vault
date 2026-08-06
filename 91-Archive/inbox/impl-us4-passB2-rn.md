---
status: draft
role: implementer
topic: US#4 Проход B2 — pin/tests К-2/К-4 для iva-rn-brownfield
branch: feat/us4-passB2-rn
repo: /Users/bubblemac/tacticum/tacticum-dev
worktree: /Users/bubblemac/tacticum/wt-us4-passB2-rn
date: 2026-07-24
permalink: tacticum/00-board/impl-us4-passB2-rn-1
archived-at: 2026-08-03 11:16
---

# US#4 Проход B2 — iva-rn-brownfield (implementer отчёт)

Ветка `feat/us4-passB2-rn` от свежего origin/main (c6be10a). 1 коммit (4c0a227). НЕ push. Autonomy off.

## Что сделано (под rn-стек, аддитивно)

Правил ТОЛЬКО свой профиль (4 файла), стек-специфику rn-pin/tests не ломал — надстройка К-2/К-4 сверху.

**pin-authoring/SKILL.md (+75):**
- **К-2** — на v2 FR (маркер `fr_skeleton: 2`) PIN реализует проектные серии по стабильному ID + статус-таблица `| ID | раздел | реализовано в | статус |` (реализован/расхождение/blocked). Под rn: `CT-n` §1.6 → интеграции/REST-эндпоинты/SDK-типы (verify `kb_verify_api_exists` + `sdk-codegen-pipeline`); `DM-n` §1.7 → TS-типы/интерфейсы/машины состояний (`kb_get_code_context` outline, cross-check `offline-sync-engine`); `EV-n` §1.8 → эмиттеры/подписчики/идемпотентность (trace через `pin-upstream-dependency-check`). Новый пункт 12 в Document structure + отдельная секция «Project-design realization».
- **К-3** — гейт `D-n`: раздел реализуется только при утверждённом решении (гейт на входе `/start-task`, Проход C-канон, НЕ трогал). Неутверждён → честный `blocked`, без имитации контракта.
- **К-4** — расширил существующий `## API Verification` gate (PHANTOM-детект = сверка FR-API с кодом) таблицей расхождений проектных разделов FR↔KB: `CT-n`/`DM-n` сверяются с кодом (`kb_verify_api_exists`, `kb_get_code_context` outline, + стек-аналоги `sdk-codegen-pipeline` shared-contracts, `offline-sync-engine` store shapes, `kb_get_block_compact(sections=["discrepancies"])`); конфликт = **критичное расхождение** (блокер в выводе PIN). Rules + анти-паттерны добавлены.

**tests-authoring/SKILL.md (+52):**
- **К-2** — на v2 FR TESTS порождает контрактные тесты с тегом `Covers: CT-n` (+`DM-n`/`EV-n`), IDs verbatim, + статус-таблица CT/DM/EV (covered/расхождение/blocked). Под rn: CT → typed assertions против `sdk-codegen-pipeline` клиента + mocked transport (msw/Jest); DM → type/reducer/state-machine тесты; EV → idempotency/order через RNTL/Jest mock. Новый пункт 5 в Document structure + секция «Contract tests for project series» + Rules + анти-паттерны.

**manifest.yaml:** version 0.5.2→0.5.3.
**CHANGELOG.md:** секция `## [0.5.3]` с описанием К-2/К-3/К-4.

## Backward-safe (прод-safe)
v1 FR (без маркера `fr_skeleton`) → серий CT/DM/EV нет, шаг проектных разделов/контрактных тестов пропускается, оба скилла работают как раньше. Явно прописано в обоих скиллах (Document structure, секции, Rules, анти-паттерны).

## Развилки (durably)
- В rn-pin **не было** буквальной «таблицы расхождений FR↔код» из спеки К-4. Ближайшее реальное — `## API Verification Gate` (PHANTOM-классификация = сверка FR-упомянутых API с кодом). Решение: К-4 расширяет именно её (project-section discrepancy table), а не выдуманную несуществующую таблицу. `discrepancy-register` в профиле — это KB-артефакт (`kb_get_block_compact(sections=["discrepancies"])`), его и подключил как один из источников сверки.
- Отдельного catalog-теста `test_iva_rn_brownfield_profile.py` в репо нет (у mail/ios/firebird/base есть, у rn — нет). Целевые гейты для doc-правки: version-discipline + manifest-schema (оба ниже).

## Проверки
- **version-discipline** (`check_profile_version_discipline.py`, static + `--diff-against origin/main`): **OK — 48 profile(s) clean** оба прогона.
- **pytest** `test_manifest_schemas.py` (PYTHONPATH=apps/backend, --noconftest): **38 passed**.
- `test_composition.py` — требует `db_session` (БД-фикстура), под `--noconftest` не запускается; к doc-правке скиллов не относится. Унасл. red `iva-role-web` (#149 ds) — в БД-backed тестах (role-presets/parity), в целевом --noconftest прогоне не всплывает. Игнор per спека.

## git diff --stat (origin/main..HEAD)
```
 templates/iva-rn-brownfield/CHANGELOG.md           | 30 +++++++++
 .../ingredients/skills/pin-authoring/SKILL.md      | 75 ++++++++++++++++++++++
 .../ingredients/skills/tests-authoring/SKILL.md    | 52 ++++++++++++++-
 templates/iva-rn-brownfield/manifest.yaml          |  2 +-
 4 files changed, 156 insertions(+), 3 deletions(-)
```
Коммит: `4c0a227` — feat(us4-B2): pin/tests-authoring реализуют проектные серии FR v2 (К-2/К-4) для iva-rn-brownfield. git status: clean.

НЕ трогал: brd-authoring, start-task, другие профили, tacticum-dev-base. НЕ push/merge/PR.

## Связано
[[spec-us4-passB2-pintests]]
</content>
</invoke>