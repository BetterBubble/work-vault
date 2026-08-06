---
title: impl-ds-web-to-kmp-skill-skeleton
type: note
permalink: tacticum/00-board/impl-ds-web-to-kmp-skill-skeleton-1
status: draft
role: implementer
task: ТЗ#1 Сц.4 Фаза 1 — скелет навыка web-to-kmp-screen-port
tags:
- impl
- lead-ds
- kmp
- skill
- scenario-4
archived-at: 2026-07-31 17:27
---

# impl: скелет навыка `web-to-kmp-screen-port` (Сц.4, Фаза 1)

status: draft · роль: implementer · autonomy off · НЕ мержено/НЕ запушено

## Worktree / ветка
- worktree: `/Users/bubblemac/tacticum/tacticum-dev-ds-web-to-kmp`
- ветка: `feat/ds-web-to-kmp` (от `main` @ `20412ff`)
- коммит: `7882902` feat(kmp): add web-to-kmp-screen-port skill skeleton (ТЗ#1 Сц.4)

## Созданные / изменённые файлы
| Файл | Что |
|---|---|
| `templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-screen-port/SKILL.md` | НОВЫЙ, тело навыка, 138 строк (≤150) |
| `templates/iva-kmp-development-base/manifest.yaml` | +запись `kind: skill_spec` (стиль пакета: `supports: [claude-code, codex]`, без copilot); bump `version` 0.1.0 → 0.2.0 |
| `templates/iva-kmp-development-base/CHANGELOG.md` | +секция `## [0.2.0] — 2026-07-24` |

Манифесты `iva-role-kmp` (роль тянет через depends_on) и `iva-kmp-brownfield` (зеркало) НЕ трогались — owner-only, как рекомендовала разведка. `_mirrors.yaml` не трогался (новый ингредиент owner-only, mirror-скрипт его не требует).

## Что в скелете есть
Frontmatter = только `name` + `description` (trigger-фразы, вкл. русские: «перенос экрана», «Angular в Compose», «экран iva-one»…). Тело:
1. **Процедура чтения источника** iva-one — порядок 1-8 (architecture.md+signal-store rfc → page-route → форма store → view-state enum → пайплайн список/выбор/деталь → имена DS-компонентов → Transloco → REST-контракт).
2. **Таблица Angular→Compose** — перенесена целиком (14 строк: компонент→@Composable, @Input→параметр, @Output→lambda, *ngIf→if, *ngFor→LazyColumn+key, ng-content→slot, signal/computed→mutableStateOf/derivedStateOf, effect→LaunchedEffect, RxJS→Flow, signalStore→Decompose+StateFlow, withComputed→mapper+stateIn, withMethods/rxMethod→onXxx, forms/CVA→state hoisting, Material/DS→Iva*, DI→ручной Factory).
3. **Маппинг состояния**: signalStore→Decompose со StateFlow; default уровень-1 (MutableStateFlow+onXxx, как feature/contacts), MVIKotlin только если экран уже на нём.
4. **Гардрейлы приземления**: только `feature/<name>/impl/commonMain` + `core/design-system` (`Iva*`); НИКОГДА `ucim`/`presentation.*`; навигация через Decompose `News`, не роутер.
5. **Правила против ошибок ИИ-переноса**: не над-переводить (DI/RxJS/two-way не дословно), не галлюцинировать компоненты (ограничить реальной поверхностью `Iva*`+токены, verify-шаг), обязательный state hoisting.
6. **Что НЕ переносить**: роутинг/DOM/CSS/RxJS-DI-глю/WebRTC-звонки/Electron.
7. **Верификация** — 4 критерия (компонентный уровень без сырых Color/dp; паритет с образцом; паритет с Figma токены+VLM; тесты runComposeUiTest+Roborazzi).
8. **Ссылки на существующие навыки** (переиспользует, не дублирует): `compose-multiplatform-ui`, `design-system-discovery`, `design-token-usage`, `ui-mockup-match`.

Принцип «rewrite-port, не move-port» сформулирован БЕЗ ссылки на несуществующий `android-to-kmp-porting` (как контраст-принцип). `angular-ds-component-usage` тоже не упоминается. Проверено grep'ом — мёртвых ссылок нет.

## Что помечено TODO (на паузе до пилот-репо KMP)
Раздел «TODO — blocked on pilot KMP repo» в конце тела:
- выбор первого экрана для переноса (кандидат из `APP_FEATURE_MAP.md`+PARITY-заметок);
- реальный словарь `Iva*`↔веб-мастер-компонент (ключи веб-UI-KIT ~107);
- подтверждение у дизайнеров (моб. Figma = только токены);
- Figma-мост в DS-навыках (расширение навыков `tacticum-ui-base`, не здесь).

## Валидация
- JSON-схема `ingredient.v1.schema.json` (ветка skill_spec, `metadata.description_trigger` required) — **OK** для нового ингредиента.
- `manifest.v2.schema.json` для всего манифеста — **OK**.
- `scripts/check_mirror_sync.py` — **OK** (62 ингредиента в 6 парах синхронны, новый не задет).
- `scripts/check_profile_version_discipline.py` (static + `--diff-against main`) — **OK** (46 профилей чисто; bump+CHANGELOG консистентны).
- Прогон через `uv run --with pyyaml --with jsonschema` (venv в репо нет).

## git status / diff --stat (vs main)
```
 M templates/iva-kmp-development-base/CHANGELOG.md   | 14 ++++++++++++++
 M templates/iva-kmp-development-base/manifest.yaml  | 17 ++++++++++++++++-
 ?? .../ingredients/skills/web-to-kmp-screen-port/SKILL.md (новый, в коммите)
 2 files changed (+ новый файл), 30 insertions(+), 1 deletion(-)
```
(на момент отчёта всё закоммичено в `7882902`, рабочее дерево чистое)

## Самопроверка
- SKILL.md читается, frontmatter `name: web-to-kmp-screen-port` = имя папки.
- Манифест: запись вставлена между `compose-multiplatform-ui` и `proguard-r8-discipline`, стиль соседних записей соблюдён, version=0.2.0, ингредиентов 15.
- CHANGELOG heading `## [0.2.0]` совпадает с manifest.version (version-discipline зелёный).

## Для тимлида
Скелет, не финал — реальный словарь компонентов и выбор экрана требуют пилот-репо. Не расширял объём за рамки Фазы 1. Готово к ревью; при апруве — можно двигать к Фазе 2 (наполнение словаря на пилоте).