---
title: Механика композиции лейнов — что произойдёт при переносе ингредиентов в новый лейн iva-write-base
type: note
status: draft
created: 2026-08-05
updated: 2026-08-05
permalink: tacticum/00-board/explore-lane-compose-mehanika-2026-08-05
tags: [board, explore, iva-write]
---

Репозиторий: `/Users/bubblemac/tacticum-worktrees/iva-write-lane` (worktree tacticum-dev, origin/main, d607513). Только чтение, правок нет.

## 0. Главный ответ (риск, ради которого разведка)

**Да, итоговый CLAUDE.md/AGENTS.md установки аналитика ИЗМЕНИТСЯ**, даже если приедут ровно те же строки текста. Причём по-разному на двух CLI, и на codex — с потерей маркера роли.

Проверено прогоном настоящих рендереров на синтетическом наборе (композиция: `rule_set` из core-base → пак нового лейна → пак роли):

**claude-code** (`render_for_claude_code`, `apps/backend/src/backend/catalog/domain/renderer.py:159`) — дедупликации по пути НЕТ. Получается ДВА отдельных `write_file` в `CLAUDE.md`:

```
{"path": "CLAUDE.md", "marker_id": "tacticum:iva-write-base",   "content": "<!-- tacticum:iva-write-base -->\nLANE TEXT\n<!-- /tacticum:iva-write-base -->\n"}
{"path": "CLAUDE.md", "marker_id": "tacticum:iva-role-analyst", "content": "<!-- tacticum:iva-role-analyst -->\nROLE TEXT\n<!-- /tacticum:iva-role-analyst -->\n"}
```

Итог на диске: ДВЕ секции, лейн ПЕРВЫМ (базы идут до блока роли), роль второй. Сегодня секция одна. Плюс `expected_action_count` вырастает.

**codex** (`render_for_codex` → `_render_via_canonical` → `_dedupe_actions_by_path`, `renderer.py:364,367`) — действия с одним путём СКЛЕИВАЮТСЯ в одно, а `marker_id` берётся у ПЕРВОГО действия, которое его несёт (`renderer.py:390`). Первым идёт `rule_set maintainer-feedback` из `tacticum-core-base` (codex рендерит rule_set фрагментом в AGENTS.md, `renderers/codex.py:129-138`) — он без маркера, поэтому маркер подхватывается у следующего. Сейчас это пак роли, после правки — пак лейна:

```
СЕГОДНЯ: {"path":"AGENTS.md","marker_id":"tacticum:iva-role-analyst","content":"<!-- tacticum:iva-role-analyst -->\nRULE BODY\n\nROLE TEXT\n<!-- /... -->"}
СТАНЕТ:  {"path":"AGENTS.md","marker_id":"tacticum:iva-write-base",  "content":"<!-- tacticum:iva-write-base -->\nRULE BODY\n\nLANE TEXT\n\nROLE TEXT\n<!-- /... -->"}
```

Чем именно плохо:
1. **Маркер `tacticum:iva-role-analyst` исчезает из AGENTS.md.** Текст роли уезжает ВНУТРЬ секции лейна.
2. **На уже установленных инсталляциях это даёт дубль.** Клиент (`builtins/tacticum_context_skill.md:160-162`, оракул `tests/e2e_install/oracles.py:75-89`) при `append_section` ищет свой маркер: `tacticum:iva-write-base` в файле отсутствует → секция ДОПИСЫВАЕТСЯ, а старая секция `tacticum:iva-role-analyst` остаётся навсегда. В AGENTS.md два блока, старый — протухший.
3. **Формальная приёмка миграции ломается:** чек-лист в `builtins/tacticum_context_skill.md:393` требует «CLAUDE.md / AGENTS.md contain markers of the role ONLY».
4. Позиция лейна в `depends_on` НЕ помогает: любая база идёт до блока роли.

**Ловушка «дать лейну тот же маркер `tacticum:iva-role-analyst`» — хуже.** На codex станет корректно, но на claude-code второе действие с ТЕМ ЖЕ маркером не допишет, а ЗАМЕНИТ секцию первого (`oracles.py:84-86`, `tacticum_context_skill.md:161`) — текст лейна молча пропадёт. И это не поймает ни один оракул: `assert_markers_once` (`oracles.py:188`) считает ровно один open/close и останется зелёным.

Безопасный вариант — переносить в лейн ТОЛЬКО mcp_server_spec (пункт «а» постановки), а тексты про канал записи оставить в паках роли; либо переносить тексты целиком вместе с паками (тогда у роли `CLAUDE.md`-пака не остаётся, а `test_role_carries_only_role_packs`/`PACKLESS_ROLES` требуют явного решения, `tests/catalog/test_iva_role_presets.py:305,363`). Выбор — за ответственной ролью, я его не делаю.

## 1. Где код композиции (ADR-0056 / ADR-0057)

- Чистая функция merge: `apps/backend/src/backend/catalog/domain/composition.py:20` — `compose(base_ingredient_lists, profile_ingredients)`. Источники конкатенируются в порядке: базы в порядке `depends_on`, затем блок профиля (`composition.py:41`). Коллизия `ingredient_id` — НЕ ошибка: строку даёт ПОЗДНИЙ источник, позиция остаётся от ПЕРВОГО вхождения (`composition.py:32-35,45`).
- Загрузка из БД: `apps/backend/src/backend/catalog/application/composition.py:43` `load_composed_ingredients`. Рёбра читаются из `profile_version_dependencies` с `ORDER BY position` (`composition.py:53`), внутри профиля порядок — `ORDER BY (kind, ingredient_id)` (`composition.py:38`), он load-bearing для seq-контракта manifest+fetch (Issue #630).
- Через этот хелпер обязаны ходить ВСЕ точки рендера: `tacticum_init.py:144`, `tacticum_init_manifest.py:112`, `tacticum_fetch_action.py:88`, `pull_installation_content.py:86`, `pull_installation_content_manifest.py:94`.
- Глубина строго 1, проверка на сиде, не на рендере: `application/seed_profile.py:301-316` (`depends_on_depth_exceeded`) — база, у которой есть свои `depends_on`, отвергается; этим же ломаются циклы. Само-ссылка — `seed_profile.py:164`, дубликаты в списке — `seed_profile.py:172`.
- Коллизия между базами легальна, но пишется warning `seed_base_ingredient_collision` (`seed_profile.py:319-340`).
- Рёбра неизменяемы: пересид той же версии с другим/переставленным `depends_on` → `depends_on_immutable` (`seed_profile.py:242-258`). `depends_on` входит в `payload_hash` (`seed_profile.py:106-109`), то есть добавление лейна к роли ТРЕБУЕТ bump версии роли.
- Порядок сида: сначала манифесты без `depends_on`, потом зависимые (`apps/backend/scripts/seed_community.py:149-162`) — новый лейн сядет раньше роли автоматически.

## 2. marker_id и merge_strategy

- Модель: `domain/ingredients/instruction_pack.py:10-18` — `merge_strategy ∈ {replace, append_section, deep_merge, create_if_missing}`, дефолт `append_section`; `target_file` обязателен; `marker_id` опционален. `extra="forbid"`.
- Что делает marker: рендерер ВШИВАЕТ маркеры в тело — `_wrap_body_with_markers` (`renderer.py:121-156`), формат `<!-- {marker_id} -->\n{body}<!-- /{marker_id} -->\n`. Оборачиваются только `append_section` с маркером; TOML `create_if_missing` (`.codex/config.toml`) намеренно НЕ оборачивается (`renderer.py:144`, комментарий 134-139).
- Семантика на клиенте (это контракт, а не сервер): `append_section` — заменить блок между маркерами, если он есть, иначе дописать; `create_if_missing` — не трогать существующий файл; `deep_merge` — слияние JSON (используется для `.claude/settings.json`, у аналитика — `repo_config claude-settings`, `templates/iva-role-analyst/manifest.yaml:112-119`); прочее — перезапись. Описано в `builtins/tacticum_context_skill.md:155-180` и `docs/agents/installing-new-profile.md:194-201`, исполняется оракулом `tests/e2e_install/oracles.py:69-95`.
- Важная деталь применения: марkер-ветка срабатывает только `if dest.exists()` (`oracles.py:72,75`). На чистом репо первое действие просто создаёт файл, второе — дописывает секцию.
- **Валидации/запрета на несколько instruction_pack в один target_file НЕТ нигде.** Схема (`templates/_schema/ingredient.v1.schema.json:24-37`) этого не знает; сид (`seed_profile.py`) сравнивает только `ingredient_id`; тест «single-owner по пути установки» (`tests/catalog/test_iva_role_presets.py:485`) явно ИСКЛЮЧАЕТ склеиваемые стратегии (`_MERGING_STRATEGIES`, там же:425-429) — `append_section`/`create_if_missing`/`deep_merge` считаются штатным разделением файла. То же исключение в `scripts/check_install_links.py:104-106`.
- Единственное ограничение на маркеры — для РОЛЕЙ: `tests/catalog/test_iva_role_presets.py:318-322` требует у пака роли ровно `tacticum:<role_id>`. Про лейны правил нет (живой контрпример: `iva-qa-tms-base` с маркером `iva-qa-tms-base:secrets`).

## 3. Сколько лейнов реально несут instruction_pack

**Утверждение «12 существующих лейнов несут instruction_pack» фактам не соответствует.** По каталогу (проверено разбором всех `templates/*/manifest.yaml`): instruction_pack есть у **30 профилей**, но из них ЛЕЙНОВ (тех, на кого кто-то ссылается через `depends_on`) — **ДВА**:

| profile_id | кто подключает | target_file'ы пака |
|---|---|---|
| `iva-qa-tms-base` | iva-role-qa, -qa-web, -qa-mobile, -qa-desktop | `secrets.yaml.example`, `.gitignore` (маркер `iva-qa-tms-base:secrets`, `templates/iva-qa-tms-base/manifest.yaml:111,123-125`) |
| `tacticum-dev-base` | firebird-web-brownfield, iva-ios-brownfield, tacticum-internal-dev | `CLAUDE.md`, `AGENTS.md`, `.codex/config.toml` (маркер `tacticum:tacticum-dev-base`, `templates/tacticum-dev-base/manifest.yaml:320-344`) |

Остальные 28 — это РОЛИ (`iva-role-*`, `tacticum-role-*`, `firebird-role-web`) и одиночные монолиты-brownfield (`iva-web-brownfield`, `iva-kmp-brownfield`, `iva-fr-analyst`, `iva-system-analyst` и т.д.), их через `depends_on` не подключают.

Пример лейна с instruction_pack, который реально подключается: `tacticum-dev-base` (см. строки выше). Но у трёх его потребителей собственных паков в те же файлы НЕТ (их паки — только `.github/copilot-instructions.md`).

**Прямой проверкой по всему каталогу: сегодня НИ В ОДНОЙ композиции нет двух instruction_pack/repo_config в один и тот же `target_file`.** Ноль коллизий. То есть предлагаемая правка создаёт ПЕРВЫЙ такой случай в репозитории — прецедента, на который можно сослаться, нет.

## 4. Лейн-образец «один MCP + инструкции к нему»

Три кандидата прочитаны целиком:

- `templates/iva-architect-mcp/manifest.yaml` — v0.1.1, **1 ингредиент**, `mcp_server_spec iva-atlassian-write` (`:85-103`), без `depends_on`, файлы: manifest + README + CHANGELOG.
- `templates/iva-qa-mcp/manifest.yaml` — v0.1.0, **2 ингредиента**: `iva-atlassian-write` (`:101`) + `helm-analyst` (`:122`), оба `mcp_server_spec`.
- `templates/iva-techwriter-mcp/manifest.yaml` — v0.1.0, **1 ингредиент**, `iva-atlassian-write` (`:81-95`).

Общая форма всех трёх: `schema_version: "2"`, `profile_id`, `name`, `version`, `maintainer`, `license`, `deprecated: false`, `description`, `persona{role,scope}`, `target_tasks[]`, `stack{required,optional}`, `ide_targets`, `profiles{trial,full}`, `post_install_notes`, `non_goals`, `ingredients[]`. Тонкий leaf-лейн БЕЗ `depends_on`, `body: ""` у MCP, скоуп через `metadata.allowed_tools`.

**Ближе всего — `iva-architect-mcp`**: ровно один MCP write-канала, тот же контур (публикация в Jira/Confluence), тот же паттерн «роль ссылается на лейн», и в комментарии манифеста (`:8-11`) прямо описан инвариант «READ приходит из другого лейна, здесь не дублируем».

**Но ни один из трёх НЕ несёт instruction_pack** — образца «MCP + инструкции к нему в одном лейне» в репозитории нет. Пункт (б) постановки прецедента не имеет.

Отдельно: `iva-architect-mcp` — участник `KNOWN_OVERRIDES` (`test_iva_role_presets.py:361-368`): его `iva-atlassian-write` СПЕЦИАЛЬНО перекрывает полный канал из analysis-лейна за счёт позиции ПОСЛЕ него в `depends_on`. Это готовый механизм, если понадобится скоупить `helm-iva-write` по ролям.

## 5. Схема и обязательные файлы

- `templates/_schema/manifest.v2.schema.json` — required только `["schema_version"]` (`:7`), enum `["2"]`. Описаны `superseded_by`, `depends_on` (array, `minItems:1`, `uniqueItems`, `:15-21`), `ide_targets`. Всё остальное схемой НЕ проверяется.
- Реальные обязательные поля задаёт сид: `seed_community._build_payload` (`apps/backend/scripts/seed_community.py:110-133`) читает по ключу (KeyError при отсутствии) `profile_id`, `version`, `schema_version`, `name`; остальное — `.get()`. У ингредиента обязательны `ingredient_id`, `kind`, `tier` (`seed_community.py:95-102`) и ровно один из `body`/`body_path` (`:60-73`).
- **CHANGELOG.md обязателен** — `scripts/check_profile_version_discipline.py:145-165`: нет файла или нет заголовка `## [<version>]` под текущую версию → красный CI. Проверено: CHANGELOG есть у всех 60 шаблонов.
- **README.md формально НЕ обязателен**: он в чек-листе авторинга (`docs/templates/authoring.md:20,49`), но ни один скрипт/тест его не требует, и у 26 шаблонов его нет. Фактически — конвенция, все три MCP-лейна README имеют.
- Валидаторы и как запускаются (все — из корня репозитория, CI `.github/workflows/profile-version-discipline.yml` на `templates/**`):
  - `python scripts/check_profile_version_discipline.py --diff-against origin/main` — version ↔ CHANGELOG ↔ контент;
  - `python scripts/check_mirror_sync.py` — байтовая сверка зеркал;
  - `python scripts/check_install_links.py --mode paths --all-profiles`; `--mode xrefs --all-profiles`; `--mode links --profile iva-role-qa --profile iva-role-qa-web`;
  - pytest: `apps/backend/tests/catalog/test_manifest_schemas.py`, `test_iva_role_presets.py`, `test_role_install_smoke.py`, `test_composition.py`, `test_seed_depends_on.py`; e2e `apps/backend/tests/e2e_install/`.
- Что придётся править в тестах вместе с манифестами:
  - `ROLE_LANES["iva-role-analyst"]` (`tests/catalog/test_iva_role_presets.py:49`) — список задан руками, при изменении `depends_on` тест `test_depends_on_is_the_declared_lanes_in_order` (`:335`) упадёт; новый лейн БЕЗ своего `depends_on` в `_NOT_A_ROLE` вносить не нужно (`_catalog_compositions` смотрит только на наличие `depends_on`, `:226-233`).
  - **Голдены** `apps/backend/tests/e2e_install/golden/iva-role-analyst/claude-code.json` и `codex.json` — это словарь путь→sha256 применённого дерева; хеши `repo/CLAUDE.md` и `repo/AGENTS.md` изменятся. Регенерация: `E2E_INSTALL_REGEN_GOLDEN=1` (`tests/e2e_install/oracles.py:151-160`).
  - Пин канала записи `test_analyst_role_composes_the_iva_write_channel` (`tests/catalog/test_role_install_smoke.py:283-303`) считает состав настоящим `compose` по `depends_on` (`:124-140`), поэтому переезд `helm-iva-write` в другой лейн он переживёт — но только если лейн реально попал в `depends_on` роли. Он же пинит адрес `https://helm.tacticum.ru/mcp/iva-write` и `env_required == ["TACTICUM_TOKEN"]`.
  - Версии bump'ать придётся у ТРЁХ профилей: `iva-analysis-fr` (0.2.0, откуда уезжает MCP), `iva-role-analyst` (0.3.0, меняется `depends_on` = контент хеша) и новый лейн.

## 6. _mirrors.yaml

`templates/_mirrors.yaml` — декларация пар «владелец (лейн) ↔ зеркало (старый живой профиль)» переходного периода (US #714, ADR-0059 Решение 7). Смысл: пока старый монолит жив, копии ингредиента у обеих сторон обязаны быть БАЙТ-В-БАЙТ; правки — только у владельца, зеркало обновляется тем же PR (`_mirrors.yaml:1-13`). Сверяет `scripts/check_mirror_sync.py` (сравнивает `body_path`/`codex_body_path`, для скиллов — папку целиком, `:37-61`) плюс `tests/catalog/test_role_replacement_parity.py`.

**Новый лейн туда вписывать НЕ надо**, если он не дублирует чужие тела. Правило: запись заводится только когда один и тот же ингредиент физически лежит и у лейна-владельца, и у ещё не депрекированного старого профиля.

Конкретно по нашему переносу: `helm-iva-write` в парах не значится, зеркало `iva-fr-analyst` его вообще не объявляет (у него только `iva-atlassian-write`, `templates/iva-fr-analyst/manifest.yaml:188`), а `body` у MCP пустой — сверять нечего. Действий по `_mirrors.yaml` не требуется. Существующая пара `iva-analysis-fr ↔ iva-fr-analyst` (`_mirrors.yaml:19-29`) перечисляет только скиллы и команды и переносом MCP не затрагивается.

## 7. Прочее, что стоит знать до правки

- Имя `iva-write-base` уже использовалось и было ретайрено: `git log` — `f69524d feat(iva-write-base): скелет leaf-лейна write-канала`, `81fa9fd architect+techwriter: ретайр iva-write-base → own iva-atlassian-write`. В трёх местах стоят предупреждения «прежний шлюз `mcp.tacticum.ru/iva-write/mcp` (удалённый лейн `iva-write-base`) не воспроизводить»: `templates/iva-analysis-fr/manifest.yaml:255-257`, `templates/iva-analysis-fr/README.md:29`, `docs/runbooks/iva-write-rollout-to-roles.md:45`. Переиспользование того же `profile_id` даст в БД профиль с историей — вопрос к ответственной роли.
- В `iva-analysis-fr` рядом с `helm-iva-write` живёт interim `iva-atlassian-write` (`manifest.yaml:269-283`), и в комментарии зафиксировано: снятие interim требует ещё и переиздания архитектора (у него тот же id в `KNOWN_OVERRIDES`). Если `helm-iva-write` уедет в отдельный лейн, а interim останется в `iva-analysis-fr` — канал записи окажется размазан по двум лейнам.
- Для codex `mcp_server_spec` рендерится в `~/.codex/config.toml` (user-scope, `renderers/codex.py:164`), а пак роли `codex-config-toml` — в `.codex/config.toml` (repo-scope). Пути разные, коллизии между ними нет.
- ADR по теме: `docs/adr/0056-profile-composition-depends-on-tacticum-base.md`, `0057-capability-layers-and-role-presets.md`, `0058-requirement-as-jira-us-ivareq-iva-write-role-profiles.md`, `0059-single-axis-process-lanes-and-role-packs.md`.
