---
title: Порция 0 — KMP-роль получает Compose-версии ДС-навыков + паритет по телам
type: note
status: draft
created: 2026-07-27
tags:
- board
- design-system
- impl
- kmp
permalink: tacticum/00-board/impl-kmp-role-compose-skills-1
archived-at: 2026-08-04 10:01
---

# Порция 0 — KMP-роль получает Compose-версии ДС-навыков + паритет по телам

**Ветка:** `fix/kmp-role-compose-skills` · **worktree:** `/Users/bubblemac/tacticum-worktrees/ds-porcia0`
**Коммиты:** `042e37f` (контент), `0420959` (тесты каталога), `2c72b3e` (e2e_install),
`47426bf` (golden codex). Не мержено, не пушено.

> **Порядок мержа: `fix/skill-spec-assets-metadata` → `fix/kmp-role-compose-skills`.**
> Блокер, описанный ниже, снят отдельной веткой — см. [[impl-skill-spec-assets-metadata]].
> `test_install_flow_role_kmp` теперь зелёный на ОБОИХ CLI, но codex — только при наличии
> той ветки: без неё рендер до golden'а не доходит.

## Какой способ выбран и почему

**Compose-версии четырёх навыков живут в стек-лейне `iva-kmp-development-base`, а в
`depends_on` роли `tacticum-ui-base` переставлен ПЕРЕД стек-лейном.** Приоритет даёт порядок;
механизм — штатный `compose` (later base wins), ничего нового не изобретено.

Это вариант (а) из постановки, но с уточнением, которое снимает названное в ней возражение.
Возражение было: «развернёт приоритет и для остальных навыков `tacticum-ui-base`». Проверено
программно — **разворачивать нечего**: до правки семь лейнов роли попарно не пересекались вовсе
(0 коллизий), в `tacticum-ui-base` кроме четырёх навыков есть только `playwright`, которого в
KMP-лейне нет. Порядок меняет исход ровно там, где я его меняю намеренно.

Что рассмотрено и отклонено:

- **(б) отдельные `ingredient_id` для Compose-версий** — отклонено: у пользователя оказались бы
  два навыка с конфликтующими телами и пересекающимися триггерами. Хуже текущего состояния.
- **(в) механизм переопределения на уровне роли** — искал, его нет; и он не нужен: роль по
  `test_role_carries_only_role_packs` несёт РОВНО пак-ингредиенты, контент — дело лейнов.
  Положить навыки в роль означало бы сломать инвариант тонкой роли.
- **Не дублировать, а дописать Compose-специфику в существующий `compose-multiplatform-ui`
  (он уже есть в лейне и уже говорит про `AppColors`/`IvaTheme`).** Отклонено: сильнейшая часть
  потери — `ui-mockup-match` активно предписывает НЕПРИМЕНИМУЮ процедуру (снимок DOM через
  playwright, а у Compose Desktop/Android DOM'а нет). Ссылка из соседнего навыка не отменяет
  инструкции в теле того, который сработает по триггеру. Кстати, `compose-multiplatform-ui`
  уже ссылался на `design-token-usage` и `pin-ui-pipeline-check` — и это подмену не остановило.

Механизм существует и как прецедент: `KNOWN_OVERRIDES` в `test_iva_role_presets.py` уже нёс
`iva-role-architect: {iva-atlassian-write}` — более специфичный лейн стоит позже и перекрывает
общий. KMP-случай — тот же приём.

## Что перенесено

Все четыре навыка `tacticum-ui-base` расходились с версиями старого `iva-kmp-brownfield` —
не два, как в постановке, а четыре:

| Навык | Что было в роли (ui-base) | Что приезжает теперь (стек-лейн) |
|---|---|---|
| `ui-mockup-match` | снимок рантайма только playwright'ом по DOM | + блок: у Compose Desktop/Android DOM'а нет, снимок из merged semantics tree `runComposeUiTest`; на `:webApp` DOM-путь работает как есть |
| `design-token-usage` | CSS-переменные / Tailwind / styled-components / `ThemeProvider` | tree-first выборка токенов + привязка через `AppColors` / `IvaTheme`, запрет литерального hex и сырых `dp`/`sp` |
| `pin-ui-pipeline-check` | «QML/QWidget/React/Vue component», дефолт `QLineEdit` | `Composable`, Decompose-компонент, `Column`/`LazyColumn`/`Dialog` |
| `design-system-discovery` | «не блокируй макеты» в общем виде | то же про Compose desktop/web |

Тексты — **байт-в-байт** копии из `iva-kmp-brownfield` и добавлены в `templates/_mirrors.yaml`
(пара `iva-kmp-development-base` ↔ `iva-kmp-brownfield`), так что дальше владелец и зеркало
держатся синхронно механикой, а не обещанием. `check_mirror_sync.py` — 68 ингредиентов в 6 парах,
зелёный.

Бампы: лейн `0.7.0 → 0.8.0`, роль `0.1.1 → 0.2.0`, CHANGELOG у обоих.
`check_profile_version_discipline.py` (и статически, и `--diff-against main`) — зелёный.

**Состав роли по `ingredient_id` не изменился** — изменились тела четырёх навыков.

## Как починен тест паритета и мутационное доказательство

`test_role_replacement_parity.py` сверял только идентификаторы — поэтому подмена и прошла молча.
Добавлен слой **stack-fidelity** поверх тех же пар «роль ← старый профиль»:

1. **Эффективный источник тела считается реальным доменным `compose`** (импорт
   `backend.catalog.domain.composition.compose`), а не переписанным правилом приоритета. Если
   правило слияния когда-нибудь изменится, тест поедет за механизмом, а не за представлением о нём.
2. **Чужой словарь запрещён:** тело, которое роль реально отдаёт, не должно содержать
   `Tailwind` / `styled-components` / `CSS variable` / `ThemeProvider` / `QLineEdit` / `QWidget` /
   `QML` / `React` / `Vue` / `Angular` / `SwiftUI` / `UIKit`. Исключения — в `FOREIGN_ALLOWED` с
   причиной; сейчас их три и каждая объяснима (`kb-navigation` перечисляет чужие стеки как примеры
   репозиториев; `web-to-kmp-screen-port` и `web-to-kmp-source-reference` — Angular там ИСХОДНЫЙ
   стек порта по определению скилла).
3. **Стековое не подменяется нестековым:** если тело того же ингредиента в СТАРОМ профиле несло
   маркеры стека (`Compose`, `Composable`, `AppColors`, `IvaTheme`, `commonMain`, `Kotlin`,
   `Gradle`, `Decompose`), новое обязано их тоже нести. Что считать стековым — выводится из самого
   старого профиля, не из ручного списка. Осознанное обезстечивание (процессная обвязка лейнов
   стек-агностична by design) — в `STACK_NEUTRALIZED` с причиной; сейчас пять записей
   (`getting-started`, `kb-navigation`, `run-implementation`, `run-test-runner`,
   `setup-code-intelligence`).
4. **Оба списка обязаны худеть** — закрывшееся исключение валит тест, как и в существующем
   allowlist покрытия.

Почему не байт-в-байт: тексты лейнов законно эволюционируют относительно замороженного старого
профиля. Измерено: у пары `iva-role-kmp ← iva-kmp-brownfield` расходятся **18 тел из 31**
покрытых — строгая сверка утонула бы в шуме и её бы отключили. Маркеры ловят ровно класс
«вместо Compose-версии приехала веб-версия».

### Мутационное доказательство

| Мутация | Результат |
|---|---|
| вернуть `tacticum-ui-base` в конец `depends_on` | **красный:** `test_role_never_delivers_foreign_stack_body` (design-token-usage: `CSS variable, CSS-in-JS, Tailwind, ThemeProvider, styled-components, useTheme()`; pin-ui-pipeline-check: `QLineEdit, QML, QWidget, React, Vue`) и `test_stack_specific_bodies_stay_stack_specific` (все четыре навыка: «старое тело несло `[AppColors, Composable, Compose, IvaTheme]`, новое — ничего») |
| подложить веб-версию `ui-mockup-match` прямо в стек-лейн | **красный:** `test_stack_specific_bodies_stay_stack_specific` + зеркальный инвариант `test_mirror_content_is_byte_identical` |
| откатить мутации | **зелёный** |

Важно: `ui-mockup-match` — тот самый флагманский случай — чужим словарём НЕ ловится (веб-версия
там не называет чужих технологий, она просто предписывает DOM-путь). Его ловит именно проверка №3.
Поэтому обе проверки нужны, одной не хватает.

## Числа тестов

Замеры на одном стенде (Docker подняли ради постгреса в тест-контейнерах).

| Набор | main (до) | ветка (после) |
|---|---|---|
| `tests/catalog` | 671 тест, 0 падений | **691 тест, 0 падений** (+20 моих) |
| `tests/catalog` + `tests/e2e_install` | 763 теста, 37 падений | 783 теста, **36 падений** |
| весь `apps/backend/tests` | — | 1482 теста, 36 падений, 1 skip |

Дельта к main по именам тестов: **стало зелёным** `test_install_flow_role_kmp[claude-code]`;
**новых падений нет** ни одного.

Ruff по затронутым файлам — чисто.

### Что падает и почему (всё — не моё)

Все 36 падений — в `tests/e2e_install`, все были красными на main до этой работы:

- **20** — `profile_not_found: iva-go-backend-brownfield` (профиль недоступен, тесты не обновлены);
- **~14** — `installed tree does not match golden`: неперегенерированные goldens у
  `tacticum-dev-base`, `iva-web-brownfield`, `firebird-web-brownfield`, `iva-ios-brownfield`,
  `e2e-two-base-dependent` и остальных ролей — чужой дрейф контента;
- **`test_install_flow_role_go[codex|claude-code]`** — та же протухшая захардкоженная копия
  лейнов, что была у KMP (`ROLE_GO_LANES` = 4 лейна, манифест = 6). Свою починил, go оставил:
  его починка тянет перегенерацию go-goldens и правку его собственных чисел — это порция
  владельца go-роли;
- **`test_install_flow_role_kmp[codex]`** — см. ниже, блокер.

## Блокер (не мой, но мешает — нужен владелец lite-лейна)

`test_install_flow_role_kmp[codex]` НЕ зелёный. Причина не в моём изменении:

```
skill_spec.metadata.assets
  Extra inputs are not permitted [type=extra_forbidden,
  input_value=['references/work-order-template.md']]
```

`templates/tacticum-lite-base/manifest.yaml:86` объявляет `metadata.assets` у
`lite-task-workflow`. Схема авторинга это РАЗРЕШАЕТ (`templates/_schema/ingredient.v1.schema.json:81`
— `assets` и `scripts` для `skill_spec`), а модель рендера — ЗАПРЕЩАЕТ
(`apps/backend/src/backend/catalog/domain/ingredients/skill_spec.py`, `SkillMetadata` с
`extra="forbid"`). Расходятся два нормативных артефакта, и падает тот путь, который исполняется.

Почему это видно только сейчас: `render_for_claude_code` работает по сырым ORM-строкам и
проходит; codex-путь гоняет через pydantic-union (`renderer.py:236`) и падает. До моей правки
KMP-фикстура вообще не засевала `tacticum-lite-base` — тест падал раньше, на
`depends_on_missing_ref`.

Затрагивает **все роли, композирующие `tacticum-lite-base`, на codex**: kmp, go, web, mail, ios.
На main эта ошибка встречается 6 раз.

Кандидаты на фикс (не делал — см. ниже):

1. `SkillMetadata` принимает `assets: list[str] | None` и `scripts: list[str] | None` — привести
   модель к схеме. Две строки, `src/backend/catalog/domain/ingredients/skill_spec.py`, эту ветку
   никто больше не трогает.
2. Убрать `assets:` из `tacticum-lite-base` — но `manifest.yaml` этого лейна изменён в **28**
   открытых ветках, конфликт почти гарантирован. И `assets` сейчас ничем не используется:
   `references/work-order-template.md` не доставляется ни при каком варианте.

**Почему не починил сам:** это платформенное изменение контракта ингредиентов, а не опечатка;
протаскивать его внутри контентной ветки про KMP — ровно то расширение объёма, которое просили
не делать молча. Решение и порция — за тимлидом. Как только оно есть, codex-golden роли
регенерируется одной командой:
`E2E_INSTALL_REGEN_GOLDEN=1 pytest "tests/e2e_install/test_install_flow.py::test_install_flow_role_kmp[codex]"`.

## Мелочи, починенные по дороге (в зоне правки)

- `ROLE_KMP_LANES` в `tests/e2e_install/conftest.py` больше не захардкожен — читается из
  `manifest.depends_on`. Протухшая копия была причиной, по которой e2e KMP-роли был красным.
- `assert len(ids) == 38` в `test_install_flow.py` → `44` с раскладкой. 38 не сходилось с main
  уже после добавления `web-to-kmp-*` в стек-лейн.
- `_composed` в `test_role_install_smoke.py` считается через реальный `compose`. Простая
  конкатенация давала две строки на один `ingredient_id` и ложную «коллизию целевых путей» у
  роли с задекларированным override.
- `KNOWN_OVERRIDES` получил guard от протухания: запись обязана оставаться реальным
  пересечением лейнов.

## Тех-долг (наружу, списком)

1. **`tacticum-lite-base` × codex** — блокер выше. Владелец lite-лейна.
2. **`iva-role-go`: та же протухшая копия лейнов** в `ROLE_GO_LANES` (4 против 6 в манифесте) —
   `test_install_flow_role_go` красный по той же причине, что была у KMP. Чинится тем же приёмом
   (`_declared_lanes`), но тянет перегенерацию go-goldens.
3. **Тот же класс подмены у остальных UI-ролей.** `iva-role-ios`, `iva-role-mail`,
   `iva-role-web`, `firebird-role-web` тоже композируют `tacticum-ui-base`, и стековых версий
   ДС-навыков у них в лейнах нет. Для web/firebird это, вероятно, корректно (веб-версия и есть
   их версия), для **ios и mail — почти наверняка тот же баг**. Машинерия stack-fidelity
   написана обобщённо; чтобы её включить, достаточно добавить строку в `_STACK_MARKERS` /
   `_FOREIGN_MARKERS`. Сознательно не включал: это порции других ролей, и включение сразу дало бы
   красный, который чинить не мне.
4. **Мелкая неточность в тексте `ui-mockup-match`:** «`playwright` MCP server (in this profile
   since v0.3.1)» — отсылка к истории версий `iva-kmp-brownfield`, в лейне звучит странно.
   Не трогал ради байт-в-байт с зеркалом; правится вместе с зеркалом одним PR.

## Сверка границ

Тронуто: `templates/iva-kmp-development-base`, `templates/iva-role-kmp`, `templates/_mirrors.yaml`,
тесты `tests/catalog` и `tests/e2e_install` (последние — по пунктам 3–4 чеклиста постановки).
НЕ тронуто: `apps/backend/scripts/merge_iva_tokens.py`, `design-systems/`, `src/`, чужие репозитории,
порции 1/2/4, `main`.