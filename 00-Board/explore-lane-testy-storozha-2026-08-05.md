---
title: 'Explore: тесты и сторожа вокруг переноса канала helm-iva-write в лейн iva-write-base'
type: note
status: draft
created: 2026-08-05
updated: 2026-08-05
permalink: tacticum/00-board/explore-lane-testy-storozha-2026-08-05
tags:
- board
- explore
- iva-write
---

# Explore: тесты и сторожа вокруг лейна `iva-write-base`

**Репо:** `/Users/bubblemac/tacticum-worktrees/iva-write-lane` (worktree tacticum-dev, origin/main, d607513). Read-only разведка, правок не делал.

**Что планируется:** завести `templates/iva-write-base/`, перенести туда ингредиент `helm-iva-write` из `templates/iva-analysis-fr/manifest.yaml:257`, тексты про канал — из паков `templates/iva-role-analyst/ingredients/repo-configs/*`, в `templates/iva-role-analyst/manifest.yaml:20` (`depends_on`) добавить `iva-write-base`.

---

## 1. СТОРОЖ `test_iva_write_roles_covers_every_role_that_gets_the_channel`

**Где:** `apps/backend/tests/e2e_install/test_install_flow.py:1067-1091`, опирается на `_catalog_roles_with_iva_write()` (`:1040-1064`) и рукописный `_IVA_WRITE_ROLES` (`:979-991`).

Код целиком:

```python
def _catalog_roles_with_iva_write() -> set[str]:
    """Недепрекированные роли, чья композиция несёт канал записи `helm-iva-write`. …"""
    import yaml as _yaml

    lanes_with_channel = set()
    for manifest_path in sorted(_TEMPLATES_DIR.glob("*/manifest.yaml")):
        manifest = _yaml.safe_load(manifest_path.read_text(encoding="utf-8")) or {}
        for ingredient in manifest.get("ingredients") or []:
            if ingredient.get("ingredient_id") == "helm-iva-write":
                lanes_with_channel.add(manifest["profile_id"])

    found = set()
    for manifest_path in sorted(_TEMPLATES_DIR.glob("*/manifest.yaml")):
        manifest = _yaml.safe_load(manifest_path.read_text(encoding="utf-8")) or {}
        if manifest.get("deprecated"):
            continue
        if set(manifest.get("depends_on") or []) & lanes_with_channel:
            found.add(manifest["profile_id"])
    return found


def test_iva_write_roles_covers_every_role_that_gets_the_channel() -> None:
    """Роль, которой достался канал, обязана быть в ``_IVA_WRITE_ROLES``. …"""
    unregistered = _catalog_roles_with_iva_write() - _IVA_WRITE_ROLES
    assert not unregistered, (
        "роли получают канал записи композицией, но не внесены в _IVA_WRITE_ROLES: "
        f"{', '.join(sorted(unregistered))}. Реши явно: либо впиши роль в набор и "
        "допиши в её паки, ЧТО этим каналом делать (инструмент без текста агенту "
        "бесполезен), либо убери лейн из её depends_on. Молча оставить нельзя — "
        "инструмент к человеку уже приедет."
    )
```

**Что он делает своими словами.** Два прохода по файловым манифестам `templates/*/manifest.yaml`:
1. собирает **лейны, ОБЪЯВЛЯЮЩИЕ** ингредиент `helm-iva-write` в своём блоке `ingredients` (сейчас это ровно `{iva-analysis-fr}`);
2. собирает **недепрекированные профили, у которых `depends_on` пересекается** с этим множеством.

То есть он **считает КОМПОЗИЦИЮ, а не прямое объявление у роли** — но композицию ровно на один уровень (`depends_on` depth-1, что для каталога и есть максимум: `test_lanes_are_depth1_bases`, `test_iva_role_presets.py:339-346`).

Требование: любая роль, которой канал достаётся композицией, обязана быть **записана руками** в `_IVA_WRITE_ROLES` (`:979`). Проверка ОДНОСТОРОННЯЯ: `found - _IVA_WRITE_ROLES`. Лишняя запись в наборе (роль числится, а канала у неё уже нет) сторожем НЕ ловится.

Факт по текущему каталогу (проверено прогоном логики теста на реальных манифестах):
- лейны с каналом: `{iva-analysis-fr}`;
- роли, получающие канал: `{iva-role-analyst, iva-role-architect}` — совпадает с `_IVA_WRITE_ROLES`, тест зелёный.

### КЛЮЧЕВОЙ ВОПРОС: сломается ли он от нашей правки — НЕТ

После правки: `lanes_with_channel = {iva-write-base}` → `found = {iva-role-analyst}` → `unregistered = {} ` → **тест остаётся зелёным без единой правки**.

**Но именно это и есть риск, а не хорошая новость.** Наша правка **молча отбирает канал у `iva-role-architect`**: он композит `iva-analysis-fr` (`templates/iva-role-architect/manifest.yaml:26-30`), а канал оттуда уедет. Архитектор при этом останется в `_IVA_WRITE_ROLES` (`test_install_flow.py:990`), и это НЕ поймает ничто:
- сторож односторонний (см. выше);
- `_GENERIC_ROLES` (`:867-971`) архитектора не содержит → цикл `:993-996`, который переносит `helm-iva-write` между `present`/`absent`, его не касается;
- e2e-голдена у архитектора нет (`golden/` его не содержит) — он в `PACKLESS_ROLES` и в `_NOT_IN_SMOKE`;
- `test_analyst_role_composes_the_iva_write_channel` (`tests/catalog/test_role_install_smoke.py:285`) пинит только аналитика.

**Что дописать, чтобы сторож не ослаб.** Зеркальную половину — проверку `_IVA_WRITE_ROLES - _catalog_roles_with_iva_write()` («роль числится в наборе, но канал ей больше не достаётся»). Структура уже есть, нужен второй assert в том же тесте либо соседний тест по образцу `test_not_a_role_list_has_no_stale_entries` (`test_iva_role_presets.py:249`) / `test_not_in_smoke_list_has_no_stale_entries` (`test_role_install_smoke.py:105`) — в каталоге это устоявшийся паттерн «список не копит мусор». Плюс решение головой: либо `iva-write-base` идёт и в `depends_on` архитектора, либо запись `iva-role-architect` из `_IVA_WRITE_ROLES` удаляется с объяснением (комментарий `:981-990` тогда врёт и требует переписи).

---

## 2. ВСЕ места, где встречается `helm-iva-write` / `iva-write`

### 2.1 Тесты и константы

| Файл:строка | Что |
|---|---|
| `apps/backend/tests/e2e_install/test_install_flow.py:871-877` | `_GENERIC_ROLES["iva-role-analyst"]["present"]` содержит `"helm-iva-write"` |
| `…test_install_flow.py:979-991` | `_IVA_WRITE_ROLES = {"iva-role-analyst", "iva-role-architect"}` |
| `…test_install_flow.py:993-996` | цикл, дописывающий `helm-iva-write` в `present`/`absent` каждой роли `_GENERIC_ROLES` |
| `…test_install_flow.py:1040-1064` | `_catalog_roles_with_iva_write()` |
| `…test_install_flow.py:1067-1091` | сторож (см. §1) |
| `apps/backend/tests/catalog/test_role_install_smoke.py:276-303` | `test_analyst_role_composes_the_iva_write_channel` — пин ингредиента И адреса: `url == "https://helm.tacticum.ru/mcp/iva-write"`, `transport == "http"`, `auth_type == "bearer"`, `env_required == ["TACTICUM_TOKEN"]` |

### 2.2 Каталог (шаблоны)

- **Объявление ингредиента:** `templates/iva-analysis-fr/manifest.yaml:239-268` (комментарий-обоснование + сам блок с `:257`), встречный комментарий у `iva-atlassian-write` — `:270`; шапка-счётчик и текст описания — `:32-33`, `:50`, `:288-290`.
- **Тексты паков роли:** `templates/iva-role-analyst/ingredients/repo-configs/claude-code/CLAUDE.md.fragment:18,20-27`; `…/codex/AGENTS.md.fragment:18,20-27`; `…/codex/config.toml.template:10,22-23`.
- Документация лейна: `templates/iva-analysis-fr/README.md:7,23-31`, `templates/iva-analysis-fr/CHANGELOG.md:10-40`, `templates/iva-role-analyst/CHANGELOG.md:9,28`.
- Исторические упоминания «старого лейна `iva-write`» (снесён, к делу не относится, но имя созвучно): `templates/iva-role-architect/CHANGELOG.md:60,80,88`, `templates/tacticum-role-techwriter/CHANGELOG.md:32,51,59`, `templates/iva-role-qa/CHANGELOG.md:235-294`.
- **Внимание к имени.** Имя `iva-write-base` УЖЕ занимала прежняя (удалённая) сущность, и в каталоге в трёх местах стоит предупреждение «прежний шлюз `mcp.tacticum.ru/iva-write/mcp` (удалённый лейн `iva-write-base`) не воспроизводить»: `templates/iva-analysis-fr/manifest.yaml:255-256`, `templates/iva-analysis-fr/README.md:28-29`, `tests/catalog/test_role_install_smoke.py:282-284`, `docs/runbooks/iva-write-rollout-to-roles.md:44-46`. Новый лейн с тем же именем сделает эти предупреждения двусмысленными — их придётся переписать, иначе читатель решит, что «не воспроизводить» относится к нашему новому лейну.

### 2.3 Рукописные списки, которые придётся дополнять при появлении нового лейна

| Список | Файл:строка | Надо ли трогать |
|---|---|---|
| `ROLE_LANES` | `tests/catalog/test_iva_role_presets.py:48-200`, аналитик — `:49` | **ДА.** `test_depends_on_is_the_declared_lanes_in_order` (`:331-335`) сверяет `depends_on` С ТОЧНОСТЬЮ ДО ПОРЯДКА. Не впишешь `iva-write-base` в строку 49 — тест красный |
| `_GENERIC_ROLES` | `tests/e2e_install/test_install_flow.py:867-971` | Ключей менять не нужно (роль та же). `present` аналитика уже содержит `helm-iva-write` (`:876`) |
| `_IVA_WRITE_ROLES` | `test_install_flow.py:979-991` | См. §1 — решение по архитектору |
| `ROLES` (install-smoke) | `tests/catalog/test_role_install_smoke.py:39-57` | НЕТ: список ролей, не лейнов; сторож полноты `test_smoke_covers_every_composition_in_catalog` (`:93`) считает композиции по наличию `depends_on` — у лейна его нет |
| `_NOT_IN_SMOKE` / `_NOT_A_ROLE` | `test_role_install_smoke.py:65-74` / `test_iva_role_presets.py:219-223` | НЕТ, пока у лейна нет `depends_on` |
| `KNOWN_OVERRIDES` | `test_iva_role_presets.py:361-370` | НЕТ, если новый лейн не пересекается по `ingredient_id` с остальными лейнами роли. Пересечётся — `test_single_owner_lanes_are_pairwise_disjoint` (`:374`) и `test_golden_parity_union_equals_sum_of_lanes` (`:519`) сразу красные |
| `ROLE_PERSONA` | `test_iva_role_presets.py:257-276` | НЕТ (только роли) |
| `_MODE_COMMANDS` | `tests/e2e_install/test_role_modes_routed.py:47-52` | НЕТ: только четыре режимных лейна. НО правя паки аналитика, нельзя выкинуть упоминание `/start-research` — `tacticum-research-base` у него в `depends_on`, и тест требует имя команды в тексте пака (`:137-151`) |
| `_catalog_roles_with_stack_lane` | `test_install_flow.py:998-1015` | НЕТ: фильтр `lane.endswith("-autotest-base")`; `iva-write-base` под него не попадает (попал бы при имени `…-autotest-base`) |
| `REPLACEMENTS` / `MIRROR_PAIRS` | `tests/catalog/test_role_replacement_parity.py:90-91`, `templates/_mirrors.yaml` | НЕТ: `helm-iva-write` не входит в зеркальную пару `iva-analysis-fr ↔ iva-fr-analyst` (`_mirrors.yaml`, список ингредиентов — только skill/command) |

---

## 3. ГОЛДЕНЫ

**Что это.** `apps/backend/tests/e2e_install/golden/<profile_id>/<target_cli>.json` — JSON-словарь `relpath → sha256` для ВСЕГО установленного дерева, ключи с префиксами `repo/` и `user/` (README `tests/e2e_install/README.md:63-68`, генерация — `oracles.py:151-176`, снимок — `oracles.snapshot_tree`, `oracles.py:132-140`).

Голдены есть только у **ролей и воркфлоу-профилей** (24 каталога), у лейнов их нет: `iva-analysis-fr` голдена НЕ имеет, `iva-role-analyst` — имеет (`claude-code.json`, `codex.json`, 31 и 30 файлов).

**ГЛАВНОЕ (ответ на вопрос лида): да, голден фиксирует ИТОГОВЫЙ состав установки — но как множество путей и sha256 их байтов.** Он и есть машинная проверка «состав обязан совпасть»: пропал ингредиент → пропал ключ; изменилось тело → изменился хеш.

**Поправка к постановке:** *«фрагмент голдена, где виден `helm-iva-write` и тексты из паков», показать нельзя — в голдене нет содержимого, только хеши.* Канал живёт внутри двух файлов:
- claude-code: `repo/.mcp.json` = `96b6943447210886d294b34067d3c34883bd01c3e2bd510477fc7369b7096e7f`
- codex: `user/.codex/config.toml` = `471aeb9e2ae4e02280c7a23c0b6450cc2448b0d1ba381dcd72baf4d1561bff31`

Тексты паков — внутри:
- `repo/CLAUDE.md` = `973bbf2b62ae058e4be1df91d5daf7c8e987de757009290ddf899ecd4ed3c675`
- `repo/AGENTS.md` = `613d4b498142ae339001d08ab2dce78c4280c54bc40e7b75fae2784bc0c6d81c`
- `repo/.codex/config.toml` = `e59a876f5524ac64a3b8b814355eea2929d913347cbfdb8d5af5f93adb008679`

### Точный прогноз diff'а голдена — это и есть приёмка

Разобрал механику применения (`oracles.apply_actions`, `oracles.py:51-118`) и рендер:

- **`repo/.mcp.json` ОБЯЗАН остаться байт-в-байт прежним.** Каждый MCP — отдельный `merge_json` по своему `json_pointer`, а файл пишется через `json.dumps(data, indent=2, sort_keys=True)` (`oracles.py:114`). Ключи сортируются → **порядок лейнов на результат не влияет**. Если хеш `.mcp.json` изменился — состав канала реально поехал (адрес, `env_required`, `auth_type`), и регенерировать голден нельзя.
- **`user/.codex/config.toml` СКОРЕЕ ВСЕГО изменится, и это нормально.** У codex все `mcp_server_spec` рендерятся в один и тот же путь (`renderers/codex.py:164`) и склеиваются конкатенацией в порядке ингредиентов — `_dedupe_actions_by_path` (`domain/renderer.py:367-397`). Перенос `helm-iva-write` в лейн, стоящий в `depends_on` на другой позиции, меняет ПОРЯДОК TOML-блоков. Перед регенерацией diff обязан показать те же четыре блока `[mcp_servers.*]`, просто в другом порядке.
- **`repo/CLAUDE.md`, `repo/AGENTS.md`, `repo/.codex/config.toml`** изменятся ровно настолько, насколько правятся тексты паков.
- **Любой другой файл в diff = задето лишнее** (та же формулировка в рунбуке, `docs/runbooks/iva-write-rollout-to-roles.md:103-106`).

### Как генерируются

`oracles.py:157-161`: при `E2E_INSTALL_REGEN_GOLDEN=1` голден перезаписывается вместо сравнения. Команды (из `apps/backend/`):

```bash
E2E_INSTALL_REGEN_GOLDEN=1 uv run pytest tests/e2e_install -q            # весь набор (README:73)
E2E_INSTALL_REGEN_GOLDEN=1 uv run pytest tests/e2e_install -k analyst    # точечно (рунбук:103)
```

Дальше — смотреть diff как обычный код-ревью и коммитить (README `:70-77`).

---

## 4. КАК ЗАПУСКАТЬ ТЕСТЫ (из CI, не по памяти)

**Нужен Docker.** `tests/conftest.py:29-102` поднимает `postgres:16-alpine` в контейнере на session-scope и сам ставит `DATABASE_URL`. Переменных окружения задавать не нужно; порт можно зафиксировать через `CATALOG_TEST_PG_PORT`. Makefile в репозитории нет; единственный скриптовый оркестратор — nightly S5b (`apps/backend/dev/e2e/run_install_e2e.ps1`), он к нашей задаче не относится.

Команды CI (`.github/workflows/install-e2e.yml`, шаги в `working-directory: apps/backend`):

```bash
# (в) только каталожные тесты — быстрые, падают первыми
uv run --python 3.12 --extra dev python -m pytest tests/catalog -q -p no:randomly

# (б) только e2e_install
uv run --python 3.12 --extra dev python -m pytest tests/e2e_install -q -p no:randomly
```

Вариант из рунбука (`docs/runbooks/iva-write-rollout-to-roles.md:110-116`), если окружение уже собрано:

```bash
cd apps/backend
PYTHONPATH=src python -m pytest tests/catalog tests/e2e_install -q
```

**(а) весь backend-набор:** отдельного «всего набора» в CI нет — гейт состоит ровно из этих двух шагов. Полный прогон = `uv run --python 3.12 --extra dev python -m pytest -q` из `apps/backend` (тот же Docker-Postgres). Важное ограничение порядка, записанное в `tests/e2e_install/README.md:89-93`: `tests/catalog` обязан собираться РАНЬШЕ `tests/e2e_install` (алфавит), иначе инвариант `test_migration_0008_data.py` про `copilot` в `supports[]` падает.

Второй гейт (`.github/workflows/profile-version-discipline.yml`, Python 3.12 + PyYAML, без Docker):

```bash
python scripts/check_profile_version_discipline.py --diff-against "origin/main"
python scripts/check_mirror_sync.py
python scripts/check_install_links.py --mode paths  --all-profiles
python scripts/check_install_links.py --mode xrefs  --all-profiles
python scripts/check_install_links.py --mode links --profile iva-role-qa --profile iva-role-qa-web
```

`--all-profiles` включает и одиночные лейны (`scripts/check_install_links.py:666-676`) — значит **новый `templates/iva-write-base/` попадёт под `paths` и `xrefs` сразу**.

---

## 5. ВАЛИДАТОР МАНИФЕСТОВ — что потребует от `templates/iva-write-base/`

**Чем валидируется:**
- схема — `templates/_schema/manifest.v2.schema.json` (+ `ingredient.v1.schema.json`), тесты схемы — `tests/catalog/test_manifest_schemas.py`; но `test_manifest_validates_against_v2_schema` (`test_iva_role_presets.py:292-294`) гоняет схему только по `ROLE_LANES`, то есть **по ролям, не по лейнам**. Схему нового лейна проверит серверный `seed_profile` при сиде в e2e (`seed_real_template`, `tests/e2e_install/conftest.py`) — он же и есть настоящий валидатор.
- Формально `required` у схемы всего один — `schema_version`; де-факто обязательный минимум задан минимальным манифестом в `test_manifest_schemas.py:47-62`: `schema_version: "2"`, `profile_id`, `name`, `version`, `maintainer`, `license`, `description`, `persona`, `target_tasks`, `stack`, `ide_targets`, `profiles`, `ingredients`.

**Что потребуется от нового лейна:**
1. `templates/iva-write-base/manifest.yaml` с `profile_id: iva-write-base`, `deprecated: false`, **без `depends_on`** — иначе роль станет depth-2 и `test_lanes_are_depth1_bases` (`test_iva_role_presets.py:339-346`) упадёт, а сид отвергнет с `depends_on_depth_exceeded` (`tests/catalog/test_seed_depends_on.py`).
2. `templates/iva-write-base/CHANGELOG.md` с заголовком `## [<version>]`, точно совпадающим с `manifest.version`, — `check_profile_version_discipline.check_changelog_has_entry_for_manifest_version` (`scripts/check_profile_version_discipline.py:144-166`) обходит ВСЕ каталоги в `templates/` и даёт `CHANGELOG.md missing` для нового тоже.
3. Уникальность `ingredient_id` относительно остальных лейнов аналитика — `test_single_owner_lanes_are_pairwise_disjoint` (`:374-382`) и `test_golden_parity_union_equals_sum_of_lanes` (`:519-531`).
4. Если в лейн едут ПАКИ (тексты про канал), а не только `mcp_server_spec`:
   - `marker_id` пака лейна **обязан отличаться** от `tacticum:iva-role-analyst`, иначе `assert_markers_once` (`test_install_flow.py:1183`) увидит два маркера. Прецедент: лейны `iva-qa-autotest-base` / `iva-qa-tms-base` используют маркеры вида `iva-qa-autotest-base:secrets`;
   - `merge_strategy` обязан быть склеивающим (`append_section` для CLAUDE.md/AGENTS.md, `create_if_missing` для `.codex/config.toml`) — иначе `test_lane_target_files_are_pairwise_disjoint` (`:483-515`) увидит два владельца одного пути. Список склеивающих стратегий — `test_iva_role_presets.py:425-427`;
   - **роль обязана сохранить хотя бы один свой пак**: `test_role_carries_only_role_packs` (`:307-328`) требует непустой `ingredients` у роли вне `PACKLESS_ROLES` (`:303`). Вынести паки аналитика ЦЕЛИКОМ в лейн нельзя без внесения роли в `PACKLESS_ROLES`, а это прямое ослабление (дыра Ф-3);
   - `mcp_server_spec` в манифесте РОЛИ класть нельзя (`ROLE_PACK_KINDS`, `:299`) — ровно это и есть причина заводить лейн.
5. Бамп версий: `iva-analysis-fr` (у него убывает ингредиент, `0.2.0` → выше), `iva-role-analyst` (`0.3.0` → выше: меняются `depends_on` и паки) + записи в их CHANGELOG. Без этого — `bump-needed` / `changelog-missing` и на сиде `version_already_exists_with_different_content`.

---

## 6. РУНБУКИ

### 6.1 `docs/runbooks/iva-write-rollout-to-roles.md` — наша прямая инструкция

Важно: рунбук описывает **другую стратегию** — «класть ингредиент в УЖЕ СУЩЕСТВУЮЩИЙ лейн роли», а не заводить отдельный. Цитата (`:50-54`):

> «Роли — тонкие пресеты (ADR-0057) и несут ТОЛЬКО паки; `mcp_server_spec` в манифесте роли завалит `test_role_carries_only_role_packs`. Класть надо в лейн, и лейн выбирать по тому, **кто его композит** — иначе канал достанется лишним ролям.»

Наш вариант («свой лейн под канал») это правило не нарушает и решает ровно ту проблему, из-за которой архитектор получил канал случайно, — но рунбук после правки станет неверным и потребует обновления.

Порядок шагов (`:76-106`), пересказ по пунктам:
1. **Ингредиент** — блок `helm-iva-write` в `mcp_server_spec`-секцию нужного лейна, рядом с `iva-atlassian-write`, с русским комментарием (целевой канал, пишет от имени сотрудника, требует однократного согласия, без согласия отказ).
2. **Встречный комментарий** у `iva-atlassian-write`: «замена уже объявлена выше — helm-iva-write». *«Без него читатель увидит два похожих канала и не поймёт, какой настоящий.»*
3. **Счётчики в шапке файла** (`# N ingredients: … + 2 mcp_server_spec`, `# --- mcp_server_spec (2) ---`) — поднять.
4. **Версия лейна** — minor bump. *«Пропустишь — упадёт `scripts/check_profile_version_discipline.py`, а на сиде профиль отвергнется с `version_already_exists_with_different_content`.»*
5. **CHANGELOG лейна** — запись `## [<новая версия>]`; *«Без неё тот же скрипт даёт `changelog-missing`»*. README — по формату файла.
6. **Паки роли** — в `CLAUDE.md.fragment`, `AGENTS.md.fragment`, `config.toml.template` назвать новый канал основным, старый временным. *«Правишь паки → bump версии РОЛИ + её CHANGELOG (версия лейна тут не поможет: это разные профили).»*
7. **Тесты** — добавить id роли в `_IVA_WRITE_ROLES`, *«одна строка. Ингредиент автоматически переедет из `absent` в `present`»*.
8. **Голдены** — `E2E_INSTALL_REGEN_GOLDEN=1 uv run pytest tests/e2e_install -k <роль>`; *«Перед регенерацией посмотри на падение: diff обязан называть ровно `repo/.mcp.json` (claude-code) и `user/.codex/config.toml` (codex). Другие файлы в diff = задето лишнее, регенерировать нельзя.»*

Обязательный минимум проверки (`:108-119`) — команды §4 выше; эталон на момент написания: **1081 passed, 0 failed**, 59 профилей чисто, 73 зеркальных ингредиента в 6 парах синхронны. Явно назван предел: *«Чего этими тестами не проверить: живой ответ сервера по адресу.»*

«Решения головой» (`:126-143`): `allowed_tools` (у аналитика их нет — доступна вся поверхность; сужение проверять живым запросом ДО простановки) и снятие `iva-atlassian-write` — **отдельный шаг, не часть раскатки**, потому что он завязан на `KNOWN_OVERRIDES` архитектора.

Известные расхождения, чинить попутно не надо (`:145-156`): ADR-0058 Решение 5 описывает техучётку, а задеплоено однократное согласие сотрудника; шаги согласия — в квикстарте аналитика, с **обязательным порядком «сначала Jira, потом Confluence»**.

### 6.2 `docs/user_manuals/role-migration-runbook.md`

Про наш перенос напрямую ничего не говорит: это регламент **миграции установок** со старых профилей на роли (ADR-0059 §8), порядок `реестр установок → пилот на смоук-стенде → когорта по одному пользователю → архив legacy → deprecate`, *«Ни один шаг не пропускается; deprecate — ПОСЛЕДНИЙ, а не первый»* (`:13`). Единственное, что нас касается: существующие установки аналитика запинованы на версию (ADR-0009), новый лейн доедет до людей только после бампа версии роли и повторного pull.

---

## 7. СПИСОК ТЕСТОВ, обязательных ДО и ПОСЛЕ правки

Прогнать **до** (зафиксировать зелёный baseline) и **после**:

**Быстрые, без БД по смыслу (но каталожный каталог целиком требует Docker — в нём есть seed-тесты):**
1. `tests/catalog/test_iva_role_presets.py` — `ROLE_LANES`, порядок `depends_on`, depth-1, single-owner по id и по пути установки, golden-parity, `ide_targets`.
2. `tests/catalog/test_role_install_smoke.py` — в первую очередь `test_analyst_role_composes_the_iva_write_channel` (пин адреса и параметров канала) + сторожа полноты.
3. `tests/catalog/test_role_replacement_parity.py` — роль остаётся надмножеством `iva-fr-analyst` / `iva-system-analyst`.
4. `tests/catalog/test_manifest_schemas.py`, `test_dangling_xrefs.py`, `test_stack_placeholder_links.py`, `test_seed_depends_on.py`.
   → одной командой: `uv run --python 3.12 --extra dev python -m pytest tests/catalog -q -p no:randomly`

**Сквозные (Docker обязателен):**
5. `tests/e2e_install/test_install_flow.py::test_iva_write_roles_covers_every_role_that_gets_the_channel` — сторож §1.
6. `tests/e2e_install/test_install_flow.py::test_install_flow_roles_generic[claude-code-iva-role-analyst]` и `[codex-iva-role-analyst]` — полный путь сид → provision → pull → apply → сверка состава с файловыми манифестами → **golden**.
7. `tests/e2e_install/test_install_flow.py::test_generic_roles_covers_every_role_with_a_stack_lane`.
8. `tests/e2e_install/test_role_modes_routed.py` — маршруты режимов не потерялись при правке паков.
9. Остальной `tests/e2e_install` (голдены прочих ролей обязаны остаться нетронутыми — это доказательство, что правка не задела соседей).
   → `uv run --python 3.12 --extra dev python -m pytest tests/e2e_install -q -p no:randomly`

**Скриптовые гейты (без Docker):**
10. `python scripts/check_profile_version_discipline.py --diff-against origin/main`
11. `python scripts/check_mirror_sync.py`
12. `python scripts/check_install_links.py --mode paths --all-profiles`
13. `python scripts/check_install_links.py --mode xrefs --all-profiles`
14. `python scripts/check_install_links.py --mode links --profile iva-role-qa --profile iva-role-qa-web`

**Машинное доказательство «состав не поехал»** (сильнее, чем «все зелёные»):
- `repo/.mcp.json` в `golden/iva-role-analyst/claude-code.json` обязан остаться `96b6943447210886…7096e7f`;
- голдены ВСЕХ остальных ролей — без изменений;
- изменённые файлы в diff голдена аналитика: только `user/.codex/config.toml` (порядок TOML-блоков) и те паки, которые правились осознанно.

---

## 8. Риски, найденные разведкой (не правил, только фиксирую)

1. **Архитектор молча теряет канал**, и ни один тест этого не увидит (§1). Требуется явное решение + зеркальная половина сторожа.
2. **Имя `iva-write-base` уже использовалось** снесённым лейном и фигурирует в четырёх предупреждениях каталога как «то, что не воспроизводить» (§2.2).
3. **`ROLE_LANES` сверяет порядок**, а не множество (`test_iva_role_presets.py:333`) — позиция нового лейна в `depends_on` должна быть внесена ровно та же, что в манифесте.
4. **Голден codex поедет от порядка**, а не от состава (§3) — это ожидаемо, но diff нужно читать глазами, а не регенерировать вслепую.
5. **Роль нельзя оставить без собственных паков** (`test_role_carries_only_role_packs`) — переносить в лейн можно ТЕКСТ про канал, но не сами пак-ингредиенты целиком.
6. Рунбук `iva-write-rollout-to-roles.md` после правки станет описывать несуществующую топологию — его обновление входит в объём.
