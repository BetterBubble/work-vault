---
title: explore-gates-blindspots-2026-07-28 — периметр проверок и его слепые зоны
type: note
status: current
created: 2026-07-28 10:10
repo: tacticum-dev
tags:
- board
- design-system
- gates
permalink: tacticum/00-board/explore-gates-blindspots-2026-07-28
updated: 2026-07-28 10:10
---

Разведка периметра проверок `tacticum-dev` @ `main` `26f5301`. Только чтение. Вопрос не «где баги», а «куда контур проверок вообще не смотрит».

## 0. Главный факт: pytest не запускается нигде автоматически

В `.github/workflows/` ровно два файла:

| Workflow | Триггер | Что делает | Статус |
|---|---|---|---|
| `.github/workflows/profile-version-discipline.yml` | PR + push в main по путям `templates/**` | `scripts/check_profile_version_discipline.py` + `scripts/check_mirror_sync.py` | **единственный живой гейт** |
| `.github/workflows/nightly-install-e2e.yml` | только `workflow_dispatch` | реальный Codex ставит `tacticum-dev-base` | **выключен by design** (нет секрета `CODEX_AUTH_JSON`, строки 7–21; fail-fast на шаге 30–38) |

Весь backend-suite (`apps/backend/tests/`, ~200 файлов, включая `catalog/`, `e2e_install/`, `design/`) **не вызывается ни одним workflow**. Он запускается только вручную (`uv run pytest`, Docker обязателен — `apps/backend/tests/conftest.py:29` поднимает `postgres:16-alpine`).

Серена-память при этом утверждает обратное: `.serena/memories/task_completion_checklist.md:50` и `.serena/memories/deployment.md:90` пишут «workflow'ы только `backend-ci` + `profile-version-discipline`». **Workflow `backend-ci` не существует и никогда не существовал** — `git log --all -- .github/workflows/backend-ci.yml` пуст, упоминаний в репозитории нет. Норма описывает несуществующий механизм.

Ещё две дыры того же рода:

- `ci/gitlab/governance-gate.gitlab-ci.yml` в шапке называет себя «symmetric to `.github/workflows/governance-gate.yml`». Этого файла нет → governance-gate (`scripts/governance_gate.py`) не вызывается ни из GitHub, ни (для этого репо) из GitLab.
- `manifest-lint.yml` — единственная проверка, которая валидировала **каждый** `templates/*/manifest.yaml` против JSON-схемы, — 09.05.2026 переименована в `.github/.workflows/` (коммит `29af0a9`, точка в имени = GitHub её не видит), затем удалена (`b52c382`). Сегодня сплошной валидации манифестов нет.

Git-хуков в репозитории нет (`.git/hooks` пуст, кроме sample).

## 1. Инвентаризация проверок

### 1.1. Запускаются автоматически

**`scripts/check_profile_version_discipline.py`** — сверяет `manifest.yaml version:` ↔ `CHANGELOG.md` ↔ факт изменения контента в диффе. Три режима отказа: контент менялся без бампа; бамп без записи в CHANGELOG; CHANGELOG впереди манифеста.
*Мимо проходит:* любое содержание. Бампнутая версия с мусором внутри — зелено.

**`scripts/check_mirror_sync.py`** — байтовая сверка пар «владелец (лейн) ↔ зеркало (старый профиль)» из `templates/_mirrors.yaml` (6 пар, ~60 ингредиентов). Сверяет: наличие ингредиента в обоих манифестах, побайтовое равенство body-файлов (для скиллов — папка целиком), непротухшесть пары.
*Мимо проходит:* **одинаково устаревшие копии**. Это сверка «две наши копии согласованы», а не «копия актуальна». Если оба файла протухли — проверка зелёная и активно фиксирует протухшее состояние.

### 1.2. Существуют, но вызываются только руками

**`apps/backend/tests/catalog/`** (~40 файлов). Ключевые:

- `test_role_replacement_parity.py` — паритет ролей и замещаемых профилей (см. §2а);
- `test_role_install_smoke.py` — файловый смоук композиции ролей: body-файл существует и непуст; **заявленные** целевые пути уникальны; instruction-packs не дублируются; роль даёт входную точку;
- `test_manifest_schemas.py` — зеркало «JSON-схема авторинга ↔ Pydantic-модель рендера» по набору полей `metadata` и по required (строки 321–356); плюс точечная регрессия `test_skill_spec_with_assets_and_scripts_renders` (359–382);
- `test_iva_role_presets.py` — структура пресетов: валидность против v2-схемы, роль несёт только пак-kinds, `depends_on` = задекларированные лейны, лейны попарно дизъюнктны;
- `test_seed_*` — поведение сида (депрекация, видимость, `depends_on`, коллизии).

**`apps/backend/tests/e2e_install/`** — сид реальных `templates/` в БД → `provision_installation` → pull → применение действий на диск → оракулы + сверка дерева с golden. Оракулы (`oracles.py`): `assert_action_count`, `assert_unique_ingredients`, `assert_markers_once`, `assert_mcp_entries_runnable`, `assert_no_stale_id_advice`, `assert_composite_override`, `assert_bulk_equals_manifest_fetch`, `assert_tree_matches_golden`.

**`apps/backend/tests/design/`** — токены и ДС-конфиг (`merge_iva_tokens`, контраст, резолвер, миграция 0015, сид ДС). Про **навыки** ДС не знает ничего: проверяет `design-systems/*/design-system.yaml` и `apps/backend/scripts/design-profiles.yaml`, а не тела `design-token-usage` / `ds-component-adoption`.

**Прочие скрипты:** `scripts/governance_gate.py` (ADR ↔ стандарты через прод-MCP, никем не вызывается), `scripts/wiki_sync_manuals.py` (публикатор, не проверка), `apps/backend/scripts/installation_registry.py` (отчёт «кто на каком профиле» по прод-БД, без вердикта).

## 2. Ответы на целевые вопросы

### (а) Сверяется ли содержимое тел с чем-либо

`apps/backend/tests/catalog/test_role_replacement_parity.py` (переделан 27.07) сверяет тела **только между нашими копиями**:

1. `test_mirror_content_is_byte_identical` (220–233) — владелец ↔ зеркало, оба в `templates/`.
2. `test_role_never_delivers_foreign_stack_body` (337–358) — тело не содержит подстрок чужого стека (`_FOREIGN_MARKERS`, 250–262: `Tailwind`, `React`, `QLineEdit`, `SwiftUI`…).
3. `test_stack_specific_bodies_stay_stack_specific` (364–392) — если **старое тело того же репозитория** несло маркер стека (`Compose`, `AppColors`, `Gradle`…), новое обязано нести тоже.

Эталон во всех трёх случаях — другой файл **этого же репозитория**. Внешнего эталона нет.

**Проверки на мусор или устаревание внутри `SKILL.md` не существует ни одной.** Ни валидности упомянутых классов/путей, ни свежести, ни лимита строк, ни ссылок. Максимум, что знает контур про тело, — «непустое» (`test_role_install_smoke.py:100-102`) и «содержит/не содержит подстроку из ручного словаря маркеров».

Отдельно: маркерная проверка ловит **подмену стека**, а не устаревание. Тело, которое говорит `Compose` про удалённый класс, зелёное по всем трём тестам.

### (б) Что проверяют golden

Формат: `golden/<profile_id>/<target_cli>.json` = плоский словарь `relpath → sha256`, ключи с префиксами `repo/` и `user/`. Фиксирует **состав дерева и байты доставленных файлов** — не схему, не смысл.

Есть golden (18 директорий × 2 CLI): `brownfield-task-workflow`, `e2e-composite-dependent`, `e2e-two-base-dependent`, `firebird-role-web`, `firebird-web-brownfield`, `iva-ios-brownfield`, `iva-role-analyst`, `iva-role-go`, `iva-role-ios`, `iva-role-java`, `iva-role-kmp`, `iva-role-mail`, `iva-role-qa`, `iva-role-web`, `iva-web-brownfield`, `tacticum-dev-base`, `tacticum-role-internal`, `tacticum-role-platform`.

Нет golden — 30 профилей из 48 в `templates/`. **`iva-kmp-brownfield` подтверждаю — эталона нет.** Из них устанавливаемых напрямую (не лейнов): `iva-kmp-brownfield`, `iva-go-backend-brownfield`, `iva-brownfield-mail`, `iva-rn-brownfield`, `iva-fr-analyst`, `iva-system-analyst`, `iva-2-client-shell-dev`, `tacticum-internal-dev`, `tacticum-platform-dev`, `iva-role-architect`, `tacticum-role-techwriter`. Остальные — лейны (`*-base`, `tacticum-development-core`), которые попадают в дерево только через роль.

Природа механизма: golden регенерируется командой `E2E_INSTALL_REGEN_GOLDEN=1` и коммитится. Он фиксирует **то, что есть**, а не то, что должно быть. Протухшее тело попадает в golden как норма; проверка становится защитой протухшего от случайного исправления.

### (в) Смотрит ли что-нибудь на прод

**Ни одна проверка не смотрит на состояние каталога/установок на проде.**

Что есть на прод-контуре и почему это не проверка:

- `docker-compose.prod.yml:53,70,287` — контейнерные healthcheck'и (liveness);
- ручной ранбук `.serena/memories/deployment_prod_catalog_mcp.md` шаги 6–7: `curl /readyz` → 200 и `curl /mcp` → ожидается 401. Это транспорт, не содержание: каталог может раздавать что угодно, оба останутся зелёными;
- `apps/backend/scripts/installation_registry.py` — читает прод-БД, печатает таблицу «кто на каком профиле». Отчёт без вердикта, exit-code всегда 0, запускается вручную;
- `scripts/governance_gate.py` — единственный код, ходящий в живой сервис (`mcp.tacticum.ru/arch/mcp`), но только про ADR и **ни одним workflow не вызывается**.

**Канал `feedback` не читает ничего.** `submit_feedback` (`apps/backend/src/backend/workflow/interface/mcp/submit_feedback.py:34-47`) пишет строку `telemetry_events` с `event_type='feedback'`. Читателей две штуки, и оба фидбэк не выделяют:

- `apps/backend/src/backend/platform/usage_report.py:44` — `WHERE te.event_type = 'tool_call'`, то есть фидбэк отфильтрован явно;
- `apps/backend/src/backend/identity/interface/admin/memberships.py:437-498` — админская лента активности; фидбэк туда попадает как обычная строка (`tool_name = ev.event_type` при пустом `tool_name`, строка 495), без пометки, без статуса «прочитано», без уведомления. Прочитать его можно, только если целенаправленно открыть ленту и заметить среди `tool_call`.

### (г) Согласие манифеста с рендерером

**Такого теста нет, и по конструкции быть не может — две подсистемы работают с разными источниками пути.**

`infrastructure/renderers/codex.py:79` для `SkillSpec` хардкодит `f".agents/skills/{s.ingredient_id}/SKILL.md"` и `codex_target_path` не читает вообще. Единственное место, где `codex_target_path` жив, — `domain/renderer.py:281-304` (`_CLI_BODY_KEYS` / `_cli_body_passthrough`), и оно срабатывает **только при наличии парного `codex_body`**, то есть фактически только для агентских `.toml`. Для скиллов поле мёртвое.

Для claude-code симметрии нет: `infrastructure/renderers/claude_code.py:47` `target_path_template` честно уважает.

Расхождение материализовано прямо в golden `iva-role-kmp`:

```
claude-code.json → "repo/AI common/skills/ds-component-adoption/SKILL.md"
codex.json       → "repo/.agents/skills/ds-component-adoption/SKILL.md"
```

Манифест `templates/iva-kmp-development-base/manifest.yaml:222-223, 248-249, 273-274` объявляет для обоих CLI `'AI common/skills/{ingredient_id}/SKILL.md'` (три ингредиента: `ds-component-adoption`, `web-to-kmp-screen-port`, `web-to-kmp-source-reference`).

**Почему расхождение не считается ошибкой:** golden — снимок фактического вывода рендерера, а не сверка с манифестом; никакой оракул в `oracles.py` манифест не читает. При этом `apps/backend/tests/catalog/test_role_install_smoke.py:110-132` читает **заявленный** `codex_target_path` и проверяет по нему уникальность путей. Получаются две параллельные вселенные: смоук проверяет декларацию, golden фиксирует реализацию, и **ничто их не сопоставляет**.

### (д) Проверяется ли, что засиженное на проде согласовано с кодом

**Нет. Проверяется только то, что лежит в репозитории, и то выборочно.**

Ни один тест не прогоняет `templates/*/manifest.yaml` целиком через модель ингредиента: сплошного цикла по манифестам в `apps/backend/tests/` не существует (проверено грепом по `glob`/`rglob`/`iterdir`). Единственная сплошная валидация — удалённый `manifest-lint.yml`, и он проверял против JSON-схемы, которая `assets` как раз **разрешала**, — то есть даже живой не поймал бы.

Регрессия `test_skill_spec_with_assets_and_scripts_renders` (`test_manifest_schemas.py:359-382`) — это **синтетический** ингредиент, руками воспроизводящий манифест `tacticum-lite-base` («ровно тот манифест, что несёт `tacticum-lite-base`»). Настоящий файл не читается. Разъедется ещё раз — тест не заметит.

## 3. Что должно было поймать четыре находки

| Находка | Кто обязан был поймать | Существует? | Почему прошло мимо |
|---|---|---|---|
| **1. Установка ролей под codex ломалась с 25.07** (`skill_spec.metadata.assets`) | `tests/e2e_install/test_install_flow.py` — codex-путь роли, зависящей от `tacticum-lite-base` (это `iva-role-go/java/mail/web/kmp/ios`) | **Да**, и он бы упал | Три причины сразу. (1) Suite не в CI — никто не запустил. (2) Golden ролей датированы 24.07 15:15, `tacticum-lite-base` с `assets:` появился 24.07 16:29 (`2ab690f`) — эталоны сняты до появления лейна. (3) Фикстуры `iva-role-go` и `iva-role-java` **до сих пор** хардкодят по 4 лейна (`conftest.py:310-315, 351-356`), тогда как манифесты объявляют 6 — `tacticum-lite-base` и `tacticum-research-base` в их e2e не участвуют. Для `iva-role-kmp` это чинили 27.07 (`2c72b3e`, `_declared_lanes`), для go/java — нет. Плюс `render_for_claude_code` идёт по сырым ORM-строкам мимо Pydantic-модели, поэтому claude-путь был зелёный и маскировал |
| **2. `calls-voip-fragile-zone` предписывает править удалённый класс** | Проверка тела против внешнего KMP-репозитория | **Нет вообще** | Внешнего эталона в контуре не существует. Хуже: `calls-voip-fragile-zone` объявлен зеркальной парой (`templates/_mirrors.yaml:29`), и `check_mirror_sync.py` **активно требует**, чтобы обе наши копии оставались побайтово равны. То есть единственная работающая в CI проверка гарантирует синхронность устаревшего, а не его свежесть |
| **3. Каталог раздаёт навык старее репозитория команды** (82 строки против 94 — `templates/iva-kmp-development-base/ingredients/skills/design-system-discovery/SKILL.md`) | То же: сверка с апстримом команды | **Нет вообще** | Ни один скрипт не читает внешний репозиторий: в `scripts/` и `apps/backend/scripts/` нет ни клона, ни fetch, ни ссылки на `one-web-kmp`. `check_profile_version_discipline` следит за бампом версии, а не за тем, откуда контент. `design-system-discovery` также в `_mirrors.yaml:33` — обе наши копии согласованно старые, CI зелёный |
| **4. Два фидбэка пролежали сутки непрочитанными** | Читатель канала `feedback` — алерт, дайджест, отчёт | **Нет** | `usage_report.py:44` фильтрует `event_type = 'tool_call'`, отбрасывая фидбэк явно. Единственный путь увидеть — админская лента `memberships.py:437-498`, где фидбэк лежит неразмеченной строкой среди тысяч `tool_call`. Ни уведомления, ни состояния «не прочитано», ни счётчика |

## 4. Граница периметра: чего контур не видит в принципе

Не «плохо проверяет» — **не смотрит туда вообще**. Пять классов:

**I. Всё, что снаружи репозитория.** Единственный эталон контура — другой файл того же репозитория. Ни строчки кода не читает репозитории команд (`one-web-kmp` и прочие), апстрим-навыки, внешние доки. Следствие: навык может ссылаться на удалённый класс, на переименованный модуль, на несуществующий путь — контур зелёный по определению. Находки 2 и 3 — один класс, и он не «пропущен», а вне области определения.

**II. Смысл и свежесть тела.** Проверки тела ровно три: непустое; байтово равно нашей же копии; содержит/не содержит подстроку из ручного словаря `_STACK_MARKERS`/`_FOREIGN_MARKERS`. Никакой проверки «это ещё правда» и «здесь нет мусора». Ручной словарь маркеров вдобавок заведён только для `iva-role-kmp` (`test_role_replacement_parity.py:244-262`) — для остальных ролей и этого нет.

**III. Прод.** Ноль проверок, читающих состояние прода: какие версии засижены, что реально раздаётся, ломается ли установка. Прод-контур наблюдается только на уровне «процесс жив» (`/readyz`, 401 на `/mcp`, docker healthcheck). Между «код в main» и «работает у пользователя» проверок нет — находка 1 прожила два дня именно в этом зазоре.

**IV. Согласие декларации и реализации.** Манифест и рендерер сверяются между собой нигде. Смоук верит `codex_target_path`, golden фиксирует хардкод `codex.py:79`, оракулы манифест не читают. Любое поле манифеста, которое рендерер игнорирует, — тихо мёртвое: сегодня это `codex_target_path` у скиллов, завтра любое другое.

**V. Обратная связь от пользователей.** Канал `feedback` — write-only. Единственный источник, который мог бы сообщить о I–IV раньше, чем через сутки, не подключён ни к чему.

**Сверх этих пяти — процедурная дыра, которая обесценивает всё остальное:** самая содержательная часть контура (весь backend pytest, включая e2e с golden) не запускается автоматически ни разу. В CI живёт только дисциплина версий и байтовое зеркало — то есть проверяется **аккуратность оформления изменений**, а не их корректность. Вчерашняя приёмка была зелёной потому, что тот единственный работающий гейт действительно был зелёным: он про CHANGELOG и байтовое равенство копий, и обе вещи были в порядке.

## Наблюдения

- [факт] `.github/workflows/` содержит два файла; второй (`nightly-install-e2e.yml`) выключен by design — весь pytest-suite вне CI
- [факт] Workflow `backend-ci`, который упоминают `.serena/memories/task_completion_checklist.md:50` и `deployment.md:90`, не существует и отсутствует во всей истории git
- [факт] `manifest-lint.yml` — единственная сплошная валидация всех манифестов — отключена переименованием в `.github/.workflows/` (`29af0a9`, 09.05.2026) и затем удалена (`b52c382`)
- [факт] `codex.py:79` хардкодит `.agents/skills/…`; `codex_target_path` для скиллов не читается нигде (жив только в `_cli_body_passthrough` при наличии `codex_body`)
- [факт] Golden `iva-role-kmp` фиксирует расхождение путей между claude-code (`AI common/skills/…`) и codex (`.agents/skills/…`) как норму
- [факт] Фикстуры `iva-role-go` и `iva-role-java` в `e2e_install/conftest.py:310-315, 351-356` хардкодят 4 лейна против 6 в манифестах — `tacticum-lite-base` и `tacticum-research-base` не покрыты
- [факт] `usage_report.py:44` фильтрует `event_type = 'tool_call'`, исключая `feedback`; выделенного читателя фидбэка нет
- [факт] Golden отсутствуют у 30 из 48 профилей, включая устанавливаемый `iva-kmp-brownfield`
- [риск] `check_mirror_sync.py` по конструкции защищает устаревший контент от расхождения — для `calls-voip-fragile-zone` и `design-system-discovery` это ровно те пары, где контент протух
- [вопрос] Единственная сплошная валидация манифестов была удалена — сознательно или как побочный эффект? Коммит `b52c382` без объяснения

## Связи

- relates_to [[report-ds-lead-state]]
