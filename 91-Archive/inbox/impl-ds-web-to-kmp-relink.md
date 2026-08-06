---
title: impl-ds-web-to-kmp-relink
type: note
permalink: tacticum/00-board/impl-ds-web-to-kmp-relink-1
status: draft
tags:
- board
- design-system
- lead-ds
- tz1
- implementer
archived-at: 2026-07-31 17:27
---

# Impl: пере-связка web-to-kmp-screen-port на реальные in-repo навыки

**Роль:** implementer для lead-ds (ТЗ#1 Сц.4). **Worktree:** `/Users/bubblemac/tacticum/tacticum-dev-ds-web-to-kmp`, ветка `feat/ds-web-to-kmp`. **Коммит:** `0e52b6a`. Не мержено/не пушено (autonomy off).

## Изменённые файлы (git diff --stat HEAD~1)
```
 templates/iva-kmp-development-base/CHANGELOG.md              |  25 +++
 .../skills/web-to-kmp-screen-port/SKILL.md                  | 216 +++++++------
 templates/iva-kmp-development-base/manifest.yaml            |   9 +-
 3 files changed, 182 insertions(+), 68 deletions(-)
```
Манифест `iva-role-kmp` и зеркало `iva-kmp-brownfield` НЕ тронуты (web-to-kmp-screen-port owner-only, в `_mirrors.yaml` не значится).

## Что пере-связано (по пунктам ТЗ 1-5)
1. **Секция ссылок (§8 + frontmatter):** таблица «Reused in-repo skills» переписана на 11 РЕАЛЬНЫХ навыков `iva-m/android/kmp` по именам: `decompose-…`, `mvi-state-machine-…`, `compose-ui-patterns-…`, `compose-multiplatform-ui`, `clean-architecture-…`, `design-system-discovery`, `design-token-usage`, `ui-mockup-match`, `kmp-ui-testing`, `iva-web-ecosystem`, `android-to-kmp-porting-…`. Явно сказано: наш навык = оркестратор ПОВЕРХ них, не дублирует.
2. **rewrite ≠ move (новая §0):** обход убран, дана прямая ссылка на реальный `android-to-kmp-porting-su-ivcs-messenger-kmp` с оговоркой — он про ПЕРЕМЕЩЕНИЕ Android→KMP в commonMain (source-set move + expect/actual); наш случай = ПЕРЕПИСЫВАНИЕ Angular→Compose (смена фреймворка). Переиспользованы: triage-дерево, bottom-up порядок `model→domain→data→infra→feature`, идея transformation-каталога (но наш — cross-framework, §2). Android-механики (@Parcelize/Dagger/expect-actual) явно исключены.
3. **Маппинг состояния (§3):** signalStore → Decompose-компонент (по `decompose-…`) + state-holder: Level-1 `MutableStateFlow + onXxx()` по умолчанию (как contact-detail) ИЛИ Level-3 MVIKotlin `Store` по `mvi-state-machine-…` (State/Intent/Msg/Label, Harel sealed-mode вместо мешка boolean, one-shot через Label) — только если экран уже на MVIKotlin. Уровень берётся из целевого кода, не из веб-источника.
4. **Трёхсторонний паритет (§7):** разведены механизмы — `ui-mockup-match` (playwright/DOM-diff, ВЕБ-runtime vs макет; годится сверить образец iva-one, НЕ Compose-компаратор) vs `kmp-ui-testing`/Roborazzi (Compose-скриншот+VLM). Полный паритет = веб-образец (`ui-mockup-match`) + токены (`design-token-usage`) + Compose-скриншот/VLM (`kmp-ui-testing`).
5. **Iva*-истина (новая §9):** инвентарь `Iva*` и код-истина берутся с текущего shared-модуля `iva-m/android/kmp` (~49 composable, commonMain), НЕ со старого пилот-снапшота (~41).

**Плюс:** TODO-секция — первый экран выбран: `ContactDetailScreen` (feature/contact-detail, кейс «починить существующий до паритета», уже на Level-1; Figma-фрейм отложен — паритет по коду).

## Версия/CHANGELOG
- `manifest.yaml`: version `0.2.0` → `0.3.0`; обновлён `description_trigger` навыка (реальные имена + трёхсторонний паритет).
- `CHANGELOG.md`: секция `## [0.3.0] — 2026-07-24`, раздел Changed.

## Результат валидаторов (все зелёные)
- `check_profile_version_discipline.py` (static): `OK — 46 profile(s) clean`.
- `check_profile_version_discipline.py --diff-against HEAD~1` (bump-needed): `OK — 46 profile(s) clean`.
- `check_mirror_sync.py`: `OK — 62 зеркальных ингредиентов в 6 парах синхронны`.
- `ingredient.v1.schema.json` (Draft7 по всем 15 ингредиентам манифеста): 0 ошибок.
- `pytest tests/catalog/test_manifest_schemas.py` (38 тестов): все прошли.

## Самопроверка
- SKILL.md читается; frontmatter сохранён (keys = name, description).
- Все 11 обязательных имён присутствуют; из `*-su-ivcs-messenger-kmp` токенов — только 5 реальных из каталога, выдуманных нет.
- Мёртвых ссылок нет; манифест консистентен (schema + version-discipline + mirror-sync).

**⚠️ Открытый вопрос (не решаю сам):** дом доставки навыка ГД эскалировал президенту — навык пока остаётся в `iva-kmp-development-base` (это контентная правка, не переезд). Если решат перенести — потребуется отдельный PR.

## Связано
`00-Board/recon-ds-inrepo-skills-catalog` · `00-Board/impl-ds-web-to-kmp-skill-skeleton` · `00-Board/gate-ds-web-to-kmp-phase1`
</content>