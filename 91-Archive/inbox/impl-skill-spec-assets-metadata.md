---
title: 'Фикс контракта: модель скилла принимает metadata.assets/scripts'
type: note
status: draft
created: 2026-07-27
tags:
- board
- impl
- catalog
- tech-debt
permalink: tacticum/00-board/impl-skill-spec-assets-metadata
archived-at: 2026-08-04 10:01
---

# Фикс контракта: модель скилла принимает `metadata.assets` / `scripts`

**Ветка:** `fix/skill-spec-assets-metadata` · **worktree:** `/Users/bubblemac/tacticum-worktrees/ds-skillspec`
**Коммит:** `488601a`. Не мержено, не пушено.

**Порядок мержа: `fix/skill-spec-assets-metadata` → `fix/kmp-role-compose-skills`.**
Указан в CHANGELOG-описании обеих веток и в коммите `47426bf` порции 0.

## Что было сломано

`templates/_schema/ingredient.v1.schema.json:81` разрешал у `skill_spec` поля `assets` и
`scripts`. `SkillMetadata` в `apps/backend/src/backend/catalog/domain/ingredients/skill_spec.py`
объявлена с `extra="forbid"` и этих полей не знала. Схемно-валидный манифест
(`tacticum-lite-base`, скилл `lite-task-workflow`) ронял рендер: `ValidationError` в
`renderer._orm_row_to_pydantic`.

**Почему прожило незамеченным:** `render_for_claude_code` работает по сырым ORM-строкам и модель
не трогает — падал только codex-путь. А codex-путь у KMP-роли до порции 0 падал ещё раньше, на
протухшей фикстуре лейнов, и настоящую причину закрывал собой.

Сама схема при этом ЗАЯВЛЯЕТ инвариант в своём `description`: «Per-kind metadata schemas mirror
Pydantic models in `backend.catalog.domain.ingredients.<KindSpec>`». Заявляла — и никто не проверял.

## Направление фикса

**Модель приведена к схеме, не наоборот** — как и просил тимлид. Убирать `assets` из
`tacticum-lite-base` нельзя: его `manifest.yaml` изменён в **28** открытых ветках, конфликт
гарантирован. `skill_spec.py` не трогает никто.

Аудит остальных восьми `kind`: расхождение было **только у `skill_spec`**. У всех прочих наборы
полей совпадали с точностью до имени.

**Честно про механику:** поля объявительные. Доставки указанных файлов нет ни до, ни после —
рендер ставит только `body`, а `assets`/`scripts` в `apps/backend/src/` не читаются нигде (проверено
грепом). Это написано в докстринге модели прямым текстом, чтобы наличие поля не приняли за наличие
механики. Доставка ассетов — фича, не починка контракта; в этой ветке её нет.

## Тесты — на класс, а не на один случай

Добавлены в `apps/backend/tests/catalog/test_manifest_schemas.py` (файл до этого валидировал только
JSON-схему и до Pydantic-моделей не доходил вовсе — ровно та щель, в которую всё и провалилось):

1. **`test_metadata_fields_mirror_between_schema_and_model`** — для всех 9 `kind` набор полей
   `metadata` совпадает у схемы авторинга и модели рендера, **в обе стороны**. Поле только в схеме
   ломает рендер схемно-валидного манифеста; поле только в модели мёртвое — его нельзя объявить.
2. **`test_model_required_fields_are_required_by_schema`** — всё, что модель требует обязательно,
   схема тоже обязана требовать. Иначе манифест засеется и упадёт на рендере. Обратное
   **намеренно не проверяется** и это сказано в докстринге: схема строже модели (`agent_spec`
   требует `model`, модель допускает `None`) отсекает манифест на входе — безопасная сторона.
3. **`test_skill_spec_with_assets_and_scripts_renders`** — регрессия ровно на теле
   `lite-task-workflow`, через ТУ ЖЕ дискриминированную union, которой пользуется
   `renderer._orm_row_to_pydantic`. Падение здесь = падение установки.

### Мутационное доказательство

Вернул `SkillMetadata` к прежнему виду (убрал два поля) → красные:

- `test_metadata_fields_mirror_between_schema_and_model[skill_spec]`:
  «зеркало схемы и модели разъехалось — только в схеме `['assets', 'scripts']`»;
- `test_skill_spec_with_assets_and_scripts_renders`: тот самый `extra_forbidden` на
  `skill_spec.metadata.assets` и `.scripts`.

Вернул фикс → зелёные. Прочие 7 тестов зеркала на мутации остаются зелёными — то есть падает
именно то, что сломано, а не всё подряд.

## Числа тестов (весь `apps/backend/tests`, Docker поднят)

| | main | ветка |
|---|---|---|
| тестов | 1462 | 1481 (+19 моих) |
| падений | 37 | 37 |
| skip | 1 | 1 |

**Новых падений нет. Зелёными не стал ни один тест — и это ожидаемо**, см. ниже.

Ruff и mypy по затронутым файлам — чисто.

## Какие роли расшивает фикс (проверено прогонами, а не рассуждением)

Прогнал `test_install_flow_roles_generic[codex-<роль>]` по каждой роли на main и на ветке,
классифицируя причину падения:

| Роль | main | ветка |
|---|---|---|
| `iva-role-web` | **ValidationError `metadata.assets`** | golden (дальше моего фикса) |
| `iva-role-mail` | **ValidationError `metadata.assets`** | golden (дальше моего фикса) |
| `iva-role-ios` | **ValidationError `metadata.assets`** | golden (дальше моего фикса) |
| `iva-role-analyst` | golden | golden |
| `firebird-role-web` | golden | golden |
| `iva-role-qa` | golden | golden |
| `tacticum-role-internal` | golden | golden |
| `tacticum-role-platform` | golden | golden |

Счёт ошибок `metadata.assets` во всём `tests/e2e_install`: **3 → 0**.

Плюс две роли, у которых ошибка была ЗАКРЫТА более ранним падением и всплывает, как только его
чинят: `iva-role-kmp` (расшито порцией 0, подтверждено) и `iva-role-go` (упирается в свой
протухший `ROLE_GO_LANES`, чужая порция).

**Итого пять ролей**, как и говорил: web, mail, ios, kmp, go. Три подтверждены прямым прогоном
до/после, kmp — прогоном в ветке порции 0, go — статически (его фикстура не доходит до рендера).

## Goldens: перегенерировать нечего, и это не отговорка

**Список пустой.** Обоснование, а не отсутствие работы:

Фикс **не меняет ни одного отрендеренного байта** — он только перестаёт ронять рендер. `assets`
не участвует в сборке дерева (проверено грепом по `apps/backend/src/`). Значит дерево установки
после фикса ровно такое, каким было бы, если бы падения не случилось.

Для web/mail/ios codex-golden после фикса не сходится — но разбор диффа показывает чужой долг:
добавились файлы lite/research-лейнов (`lite-task-workflow`, `research`, `start-research-cmd`,
`lite-task-cmd`) и разошлись `bug-fix`, `fix-bug-cmd`, `run-implementation-cmd`. Это
неперегенерированный дрейф владельцев этих лейнов. Регенерация втянула бы его в мою ветку —
ровно то смешивание, которое просили не делать.

Единственный golden, который перегенерирован, — **`iva-role-kmp/codex.json`, и он лежит в ветке
порции 0** (коммит `47426bf`), потому что содержит МОЙ контент. Механика: временно приложил фикс
`skill_spec.py` в worktree `ds-porcia0` (без коммита), перегенерировал, вернул файл на место,
закоммитил только golden. `test_install_flow_role_kmp` теперь зелёный на ОБОИХ CLI —
но только при наличии этой ветки. Порядок мержа поэтому обязателен.

## Инцидент: чужие правки приехали в мой worktree через общий stash

**Требует внимания, я это не чинил сам.**

В worktree `ds-skillspec` оказался изменённым `apps/backend/scripts/merge_iva_tokens.py` — файл,
который я не трогал и трогать не должен (прямой запрет в постановке порции 0). В мой коммит он
НЕ попал; лежит как незакоммиченное изменение.

**Причина:** `git stash` — ref уровня репозитория (`refs/stash`), а не worktree. Я дважды делал
`git stash -u` / `git stash pop`, чтобы снять базовые числа на чистом main. Стек общий на все
worktree'ы, и pop забрал чужую запись.

**Что с работой владельца.** Владелец — `ds-config` (ветка `feat/ds-profiles-config`), у него файл
тоже изменён и **сильно больше**: 950 строк против 463 у меня, но **155 строк есть только в моей
копии**. Определить, это отрефакторенный промежуточный слепок или потерянная работа, я не могу и
угадывать не стал.

**Сохранено, ничего не удалено:**
- `…/scratchpad/merge_iva_tokens.FROM-SKILLSPEC.py` — полный файл;
- `…/scratchpad/merge_iva_tokens.FROM-SKILLSPEC.patch` — дифф от main (226 строк).

(Полный путь скретчпада:
`/private/tmp/claude-501/-Users-bubblemac-tacticum-vault/141f72ca-c2cd-4979-8a2f-27663e7da4ef/scratchpad/`)

Откатить файл в своём worktree я попытался — операция не прошла по разрешениям, и это правильно:
вслепую затирать 155 чужих строк нельзя. Оставил как есть, решение за владельцем `ds-config`.

**Вывод для всех воркеров, а не только для меня:** снимать базовые числа через `git stash` в
worktree небезопасно, пока рядом работают другие агенты. Правильный способ — отдельный worktree
на `main` (как и сделана эта ветка), либо `git worktree add --detach` на нужный коммит.

## Сверка границ

Тронуто: `apps/backend/src/backend/catalog/domain/ingredients/skill_spec.py` (+15 строк),
`apps/backend/tests/catalog/test_manifest_schemas.py` (+112 строк). Больше ничего.
НЕ тронуто: `templates/` целиком (включая `tacticum-lite-base`), чужие goldens, порции 1/2/4,
`main`.