---
title: gate-ds-web-to-kmp-variant1
type: note
permalink: tacticum/00-board/gate-ds-web-to-kmp-variant1-1
tags:
- gate
- controller
- tz1
- web-to-kmp
- lead-ds
archived-at: 2026-07-31 17:27
---

# Gate: web-to-kmp-screen-port — Вариант 1 (репо-нативная доставка)

**Роль:** controller-гейт для lead-ds (ТЗ#1 Сц.4). **Read-only, вердикт.** Всё прогнано САМ (venv `apps/backend/.venv/bin/python`), не со слов implementer'а.
**Объект:** worktree `/Users/bubblemac/tacticum/tacticum-dev-ds-web-to-kmp`, ветка `feat/ds-web-to-kmp`, коммит `62bc27c` (поверх `0e52b6a`). НЕ мержено/не пушено (autonomy off).

## ВЕРДИКТ: ✅ PASS (5/5), без блокеров

---

### 1. Гит-чистота / скоуп — ✅ PASS
- `git diff --stat 62bc27c~1..62bc27c` = ровно **3 файла**, все в `templates/iva-kmp-development-base/`: `manifest.yaml` (8 строк), `CHANGELOG.md` (+19), `ingredients/skills/web-to-kmp-screen-port/SKILL.md` (1 строка). Рабочее дерево чистое (`git status` пусто).
- Запретные зоны НЕ тронуты (проверено `git diff --name-only | grep`): `iva-role-kmp`, `iva-kmp-brownfield`, `ROLE_LANES`/тест-матрица, `iva-web-brownfield`, `iva-analysis-base`, `_mirrors` — none.
- Секретов/мусора/AI-подписей нет (grep по диффу: `Co-Authored-By|Generated with|claude.ai|Claude-Session|API_KEY|SECRET|BEGIN` — пусто). Ветка `feat/ds-web-to-kmp`, не main.

### 2. Схема-комбо `install_scope: repo` + `target_path_template` (kind=skill_spec) — ✅ PASS (главное)
Прогнал jsonschema-валидацию сам:
- **`ingredient.v1.schema.json`: VALID.** Схема для skill_spec требует только `metadata.description_trigger` (присутствует). `install_scope`/`target_path_template`/`codex_target_path`/`body_path` схемой не ограничены (нет `additionalProperties:false` на уровне ingredient) → repo+template разрешены.
- **`manifest.v2.schema.json`: VALID** (валидирует только `schema_version` + top-level).
- **Доменная модель `IngredientBase`** (`apps/backend/src/backend/catalog/domain/ingredients/base.py`): `InstallScope = Literal["repo","user","none"]` — **repo это дефолт и валидное значение**; `target_path_template: str | None` — 'AI common/skills/...' валидно.
- **Полей implementer НЕ выдумывал.** `codex_target_path` — пре-существующее поле: встречается в **10+ template-манифестах**, потребляется рендером (`renderer.py:282` `_CLI_BODY_KEYS`). Оно уже было в записи (значение `.agents/skills/...`) — implementer изменил только ЗНАЧЕНИЯ трёх пре-существующих полей, новых имён не вводил.
- **Валидаторы (мой прогон, venv репо):**
  - `check_mirror_sync.py`: `OK — 62 зеркальных ингредиентов в 6 парах синхронны` (rc=0).
  - `check_profile_version_discipline.py` (static): `OK — 46 profile(s) clean` (rc=0).
  - `check_profile_version_discipline.py --diff-against 62bc27c~1`: `OK — 46 profile(s) clean` (rc=0) — bump зафиксирован в диффе.

### 3. Версия — ✅ PASS
`manifest.yaml` version `0.3.0 → 0.4.0`; в `CHANGELOG.md` добавлена секция `## [0.4.0] — 2026-07-24`. Совпадает.

### 4. Унификация §8/TODO — ✅ PASS
- Рассинхрон «tacticum-ui-base skills» УБРАН: `grep 'tacticum-ui-base'` по SKILL.md — **0 совпадений**. Строка TODO (222): `«those tacticum-ui-base skills»` → `«those in-repo DS skills (AI common/skills/)»`. Формулировка дома навыка сквозная (in-repo `AI common/skills/`) — согласована с §8/§9.
- **Доктрина НЕ сломана** (правка = 1 строка, таблицу/секции не задела): §8 reference-таблица = **11 реальных in-repo навыков** (android-to-kmp-porting, decompose, mvi-state-machine, compose-ui-patterns, compose-multiplatform-ui, clean-architecture, design-system-discovery, design-token-usage, ui-mockup-match, kmp-ui-testing, iva-web-ecosystem); §0 move-vs-rewrite контраст цел; §7 трёхсторонний паритет (ui-mockup-match=web/playwright + design-token-usage + kmp-ui-testing/Roborazzi) цел; §9 Iva*=shared **~49 composables** `core/design-system` (старый снапшот 41 явно помечен stale).

### 5. Память — ✅ PASS
Отчёт implementer'а на доске (`00-Board/impl-ds-web-to-kmp-variant1`). Карточка `12-features/plan-tz-1-…-lead-ds` обновлена записью «2026-07-24 16:4x — Вариант-1 оформлен, commit 62bc27c» + секция «Дом доставки — РЕШЕНО (президент): Вариант 1».

---

## Замечания (НЕ блокеры)
- **codex_target_path = вариант (a)** (единый репо-нативный путь). Документирующее: канонический skill-рендер codex не читает это поле (пишет в `.agents/skills/{id}` хардкодом), в контент-хэш не входит → хэш/рендер не ломает. Соответствует контексту задачи (codex_target_path=репо-путь — ожидаемо). Вариант (b) «убрать поле» тоже валиден — открытый не-блокирующий вопрос лиду.
- **AGENTS.md-роутер** (ручной индекс) — этап доставки KMP-командой (Легин), вне worktree. Отсутствие в пакете — норма, не дефект.
- Скелет+TODO до Figma-ключей — норма (Figma-доступ эскалирован президенту).

## Дальше
Тимлид → доклад ГД → OK президента. Мерж/PR — президент (autonomy off), НЕ сейчас (фича не готова целиком: Фазы 2-4 ждут Figma-ключи/пилот).