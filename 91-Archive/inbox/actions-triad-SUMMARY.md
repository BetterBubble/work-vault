---
title: Actions-триада analyst MCP — nearest_spec / who_to_involve / effort_hint
tags:
- helm
- rag2
- mcp
- worker-report
date: 2026-07-16
permalink: tacticum/00-board/actions-triad-summary-1
status: archived
updated: 2026-07-18
---

# Actions-триада analyst MCP — отчёт воркера

## Итог (АКТУАЛЬНЫЙ SHA)
Тулы + фиксы живых багов в `src/helm/interface/mcp/analyst_server.py`. Тесты **43 passed**, ruff/mypy по моим файлам чисто. **Не запушено**.

- Ветка: `feat/mcp-actions-triad` (worktree `~/tacticum-worktrees/helm-actions-triad`, от origin/main 0c6c486)
- Финальный SHA: **c21d9154b91dba4741aa2f17f59a4b6cfb3168f0** (5 коммитов: c891b6c → 2fa4750 → bbc3656 → 6903327 → c21d915)

## ✅ Лид подтвердил фиксы на ЖИВОЙ БД (итерация 4)
- `arch_map level=C2` → **37 реальных узлов** (было 0). ✅
- `affected_systems.clients` → **['Мосбиржа']** (было пусто). ✅
- `effort_hint` джойн → **similar=20** (было 0), поиск по `EpicTask.summary` работает. ✅
- `fix_versions`/`open_risks` пустые у клиентских контейнеров — легитимно (нет версий/рисков), оставлено как есть.

## Итерация 5: честность effort_hint при пустом changelog (коммит c21d915)
Живой факт: найденные EpicTask имеют ПУСТОЙ `changelog` (напр. IVAONE-1450) → длительность недоступна, но показывался `active_days: 0.0` (обман — «сделано мгновенно»). ФИКС:
- `_status_durations`: нет валидных статус-переходов → `active_days = None` (НЕ 0.0). Законный 0.0 остаётся, только если переходы ЕСТЬ, но активного времени нет.
- `effort_hint`: в медианы (`_summary`) берём только задачи с реальными длительностями; `similar_tasks`/`samples` возвращаем всегда. Если ни у одной похожей задачи нет длительностей → медианы `null` + `note: "длительность недоступна: EpicTask-changelog без таймстемпов; полноценный расчёт требует query-time RAG#2-changelog"`.
- Тесты: пустой changelog→`None` (юнит `test_status_durations_empty`) + `test_effort_hint_empty_changelog_null_not_zero` (similar=1, медианы null, note). **44 passed**, ruff/mypy чисто.

⚠️ Живьём ЭТУ правку сам не гонял (docker exec в прод по-прежнему заблокирован песочницей — нужна авторизация пользователя). Логика чистая (только разбор changelog), покрыта юнитом на пустом changelog. Лид может подтвердить тем же сниппетом на задаче с пустым changelog → ждём `active_days: null` (не 0.0).

## ⚠️ ЖИВАЯ ВЕРИФИКАЦИЯ ЗАБЛОКИРОВАНА ПОЛИТИКОЙ
`docker exec` в прод-контейнер `helm-helm-1` запрещён песочницей: read живых данных требует явной авторизации САМОГО пользователя (не тиммейта), которой нет. Поэтому фиксы ниже сделаны по РЕАЛЬНОЙ схеме моделей (источник истины в репо) + точным диагнозам лида с прода, а НЕ проверены мной на живой БД. **Нужно:** либо пользователь авторизует мне доступ, либо кто-то с доступом прогонит тулы на живых `requirement_text`/компонентах и подтвердит непустые ответы.

## Итерация 4: фиксы живых багов (коммит 6903327)
Live-урок: юнит-фикстуры с совпадающими ключами скрыли разрывы джойнов. Тесты переписаны реалистично.

**effort_hint — `similar_tasks: 0` на проде.** Корень: искал EpicTask по ключам jira-корпуса RAG#2 (не совпадают) и по product-компоненту (а `EpicTask.components` = командные дисциплины: Android/Backend/ФСТЭК/DevOps). ФИКС: похожие задачи ищем ПРЯМО в `EpicTask` по `summary` (заголовку) — грубый стем (усечение токенов до 5 символов, ловит морфологию «запись»/«записи»→«запис»), OR из ILIKE-термов; `system_id` сидируем заголовком узла C4 (`_resolve_node_title`). Добавлено поле `matched_terms` (прозрачность). Схема-корректно: `EpicTask.summary` — реальное поле, 'запис' в заголовках на проде есть (по словам лида).
  - ⚠️ Замечание по кириллице: в проде Postgres `lower()` регистронезависим по кириллице → работает. В юнит-тестах SQLite `lower()` кириллицу НЕ трогает → в тестах summary заданы строчными (иначе ложно-пусто). На прод-Postgres это не проблема.

**arch_map level=C2 → 0 узлов.** Корень: верхний уровень только C1, drill в C2 идёт через `parent`, мой пост-фильтр текущего среза C2 не давал. ФИКС: `level` без `parent`/`focus` = DRILL по уровню по ВСЕМУ дереву — собираю родителей уровня (`_parent_ids_for_level`: distinct `parent_id` узлов уровня target; C1→[None]) и обхожу их через `cio.arch_map`, склеивая узлы нужного уровня (агрегаты рисков/коммитеров сохранены, рёбра дедуплицированы). С явным `parent`/`focus` — прежний фильтр среза.

**affected_systems.open_risks/fix_versions/clients пусты на проде.** 
  - `open_risks`: точный матч `ArchRisk.node_id == container.id` терял риски, привязанные к узлам-ПОТОМКАМ. `cio.arch_map` их ПОДНИМАЕТ (`_lift`) к показанному узлу. ФИКС: беру готовый `risk_count` из `arch_map(focus=container)` — как в UI CIO. (Схема: `ArchRisk.node_id` → `ArchNode.id`, но гранулярность риска = дочерний компонент.)
  - `fix_versions`: `RequirementJira` (контейнерные `tasks`) fixVersion НЕ несёт (проверил: колонки `key` там нет — это `jira_key`; `_task_out` мапит его в `"key"`). Беру из `EpicTask.fix_versions` (CSV) по этим ключам. ОГРАНИЧЕНИЕ: резолвится только если `jira_key ∈ EpicTask.key` (могут быть разные проекты, напр. IVAONEHALF-* vs IVAONE-*) — где не пересекается, честно []. Другого query-time источника fixVersion для задачи нет.
  - `clients`: `detail.clients` (RequirementClient→Client). Расширил объединением с клиентами из рёбер юнит-экономики (`detail.client_values[].client`), т.к. часть требований линкует клиента только через ценность. Нет связи → [].

## Проверки итерации 4 (числами)
- `uv run pytest tests/interface/test_analyst_mcp.py -q` → **43 passed**.
- `ruff check` (мои 2 файла) → **All checks passed**.
- `mypy` (config-scoped) → в моих файлах **0 ошибок**.

## Новые/переписанные тесты (реалистичные — ловят именно эти баги)
- effort_hint: ключи EpicTask (`TASK-777/888/999`) НЕ совпадают с jira-корпусом; поиск по summary даёт similar=2; нерелевантный текст → 0; `system_id`→заголовок узла.
- arch_map: `c4_app` (C1→2×C2); `level=C2` возвращает оба C2-узла (раньше было 0); `level=C1`→система; несуществующий уровень→пусто.
- affected_systems.open_risks — риск на потомке учитывается через arch_map (в fixture риск на узле + closed не считается).

---
## (Исторический) Финальный SHA предыдущей итерации: c891b6c5f88bac5f2d1b930ada9781d18e4e3923

## Изменённые файлы / символы
`src/helm/interface/mcp/analyst_server.py`:
- Новые тулы (`@mcp.tool()`, паттерны как у соседей — `ctx` первым, `await _require_principal(ctx)`, `_call_rest`/`_with_session`/`rag2_router`/`cio_router`, возврат `dict[str,Any]`):
  - `nearest_spec(ctx, requirement_text, k=3)` — rag2-поиск (pool=k*4) → фильтр `source=="confluence"` → топ-k `{key/page_id, title, url, space, score}` + `note`.
  - `who_to_involve(ctx, requirement_text=None, system_id=None)` — `owners` (system_id→owner_name узла arch_map; requirement_text→контейнеры→owner_name через путь affected_systems) + `recent_contributors` (assignee похожих jira, топ~8) + `note`.
  - `effort_hint(ctx, requirement_text=None, system_id=None)` — распределения `active_days`/`lead_time_days` (median/p25/p75) + `samples[:5]` + `disclaimer`.
- Новые хелперы: `_status_durations(ch)` (чистый разбор changelog), `_normalize_epic_changelog`, `_parse_ts`, `_quantile`, `_summary`, `_fetch_epic_tasks`, `_unique`.
- Обновлены `instructions=` FastMCP (упомянуты 3 тула).
- Новые импорты: `json, re, statistics, datetime, sqlalchemy.select, EpicTask, ingest.velocity.status_bucket, Iterable`.

`tests/interface/test_analyst_mcp.py`:
- `test_all_tools_registered` расширен (11 тулов).
- Секция 8: `nearest_spec` (2), `who_to_involve` (3), `effort_hint` — юниты `_status_durations` (3: active+closed / empty / never-closed) + интеграционные (by_text / by_component / requires_arg).

## Результаты проверок (числами)
- `uv run pytest tests/interface/test_analyst_mcp.py -q` → **32 passed** (0.73s). Смоук `test_velocity.py`+`test_task_mgmt.py` → **18 passed** (регрессий от новых импортов нет).
- `ruff check` (мои 2 файла) → **All checks passed**.
- `mypy` (config-scoped `files=["src","tests"]`) → в МОИХ файлах **0 ошибок**. В репозитории есть 31 предсуществующая ошибка в чужих файлах (`cio.py`, `req_matrix*`, `test_gateway.py`, `seed_req_matrix.py`) — не трогал.

## Ключевое решение (данные) — вариант A, реализован
🚩 Формат `ch {field,from,to,author,created}` + `cr`/`an`/`al` — это **ingest-формат** `rag2_extract`. В **query-time его НЕТ**: `rag2_router.search`→`JiraDoc` несёт только метаданные (key/title/url/space/score/status/assignee/source), БЕЗ changelog и БЕЗ created. Единственный query-time источник структурного changelog — таблица **`EpicTask`** (формат `{f,t,d}`, статусы, БЕЗ author).

Реализовал вариант A (озвучивал в плане):
- `effort_hint` читает changelog из `EpicTask` (по ключам из related_tasks-хитов, либо по компоненту `system_id`). Хелпер `_status_durations` держу на КАНОНИЧЕСКОМ формате `{field,from,to,author,created}`; строки EpicTask нормализую адаптером `{f,t,d}`→`{field:"status",...}`. `created` = `EpicTask.created`.
- `who_to_involve.recent_contributors` = `assignee` из JiraDoc-хитов. **Changelog-авторов НЕ добавляю** (в EpicTask их нет) — честно опущено. Если нужны авторы changelog — потребуется отдельный источник (live `live_mcp.py`, дорого per-task) — открытый вопрос.

## Допущения / константы
- **Активные статусы** (считаются в active_days) = корзина `"development"` из `velocity.status_bucket`: In Progress / В работе / Ready for Development / Merge request / To release. **Терминальные** (закрытие) = корзина `"closed"`: Закрыт / Готово / Resolved / Done. Единый источник — `ingest/velocity.py` (RU↔EN маппинг там же), чтобы не расходиться с расчётом скорости конвейера.
- `active_days` — честный нижний предел: время до первого перехода (бэклог) и «хвост» незакрытой задачи после последнего перехода не считаются активными.
- `lead_time_days` считается только для задач с терминальным переходом (иначе None и в распределение не входит).
- `EpicTask` по компоненту — фильтр `components ILIKE %system_id%` (подстрока).

## Честная семантика effort_hint (критично)
`disclaimer` дословно по смыслу: «Справка о том, как обычно ЕДУТ похожие задачи в этом контейнере (lead time / активное время), а НЕ оценка трудоёмкости. Задача могла долго лежать в бэклоге.» Поля называются `active_days`/`lead_time_days`, НИКОГДА не «оценка» (покрыто тестом `test_effort_hint_by_text`).

## Открытые вопросы к лиду (по триаде — ЗАКРЫТЫ лидом)
1. ~~`who_to_involve` без changelog-авторов~~ → ОК, оставляем assignee.
2. ~~Пороги effort_hint (20 by-text / все by-component)~~ → ОК.

---

# Итерация 2: gap_questions + C4-level (коммит 2fa4750)

## Итог
Добавлен тул `gap_questions` + параметр `level` в существующие `arch_map`/`arch_container`.
Тесты **38 passed**, ruff/mypy по моим файлам чисто. Коммит **2fa4750d71ce407b2d9ebbb01dfe0b4f7c16a599**, не пушено.

## A) gap_questions(ctx, requirement_text) -> dict
- Детерминированная эвристика (без сети/LLM). Сверяет текст со скелетом FR ИВА (6 секций: Начальная постановка · БФТ · ФТТ · Нефункциональные · Интерфейсные · Дизайн) по наличию маркеров-терминов (константа `_FR_SECTIONS`).
- Всегда возвращает типовые gap-вопросы (`_GAP_QUESTIONS`: актор/роли, поведение при отказе, нагрузка/лимиты, безопасность/права, миграция данных, влияние на релиз/клиентов) + адресные «Раздел «X» не раскрыт — добавить?» по missing-секциям.
- Возврат: `{covered_sections, missing_sections, questions, note}`; `note="черновой чеклист, не замена аналитику"`.
- Тесты: `test_gap_questions_missing_and_checklist`, `_covered_section`, `_requires_text`.

## B) arch_map / arch_container — параметр `level` ("C1"|"C2"|"C3")
- Хелпер `_c4_level_int`: метка→int (C1→1/C2→2/C3→3), fail-loud `ValueError` на неверной метке.
- **Ограничение (зафиксировано):** `cio.arch_map` НЕ умеет запрашивать по уровню — принимает только `parent`/`focus`/`window_days`. Поэтому `level` реализован как **пост-фильтр** по `ArchNodeOut.level` (в `arch_map` фильтрует узлы текущего parent-среза; может дать пусто, если срез ≠ уровню) и **эхо-метка** в поле `level`. В `arch_container` узел задан id → уровень не влияет на выборку, `level` возвращается эхом (фактический уровень — в `node.level`).
- `default level=None` → прежнее поведение обоих тулов не меняется (обратная совместимость; старые тесты зелёные).
- Тесты: `test_arch_map_level_filters_nodes`, `_level_invalid`, `test_arch_container_echoes_level`.

## Проверки итерации 2 (числами)
- `uv run pytest tests/interface/test_analyst_mcp.py -q` → **38 passed** (0.96s).
- `ruff check` (мои 2 файла) → **All checks passed**.
- `mypy` (config-scoped) → в моих файлах **0 ошибок** (репозиторные 31 — чужие, не трогал).

## Реестр тулов (итого 12)
analyst_search · analyst_context · related_tasks · docs_ask · arch_map(+level) · arch_container(+level) · affected_systems · requirement_coverage · **nearest_spec** · **who_to_involve** · **effort_hint** · **gap_questions**

## Открытый вопрос по итерации 2
- `level` в arch_map — пост-фильтр по parent-срезу, а не сквозной drill по уровню (cio так не умеет). Если нужен настоящий «все узлы уровня C2 по всему дереву» — это правка `cio_router.arch_map` (вне текущей ветки/файла MCP), скажи — оформлю отдельной задачей.

---

# Итерация 3: обогащение affected_systems (коммит bbc36567)

## Итог
`affected_systems` — каждый контейнер обогащён 3 полями. Структура НЕ сломана (поля добавлены, ничего не убрано). Тесты **40 passed**, ruff/mypy по моим файлам чисто. Коммит **bbc36567b2ee11c2720732b79dc488ac088bcddf**, не пушено.

## Что добавлено (хелпер `_enrich_containers`, один проход + батч-запросы)
- `fix_versions` — релизы задач контейнера. **Откуда:** `containers[].tasks` строятся из `RequirementJira`, а он fixVersion НЕ несёт (проверено: `_task_out` отдаёт key/status/assignee/track/category). Поэтому беру из **`EpicTask.fix_versions`** (CSV-строка) по ключам задач контейнера — тот же приём, что в effort_hint. Нет задач/EpicTask → `[]`.
- `clients` — заказчики требования. **Откуда:** `detail.clients` (`RequirementDetailOut`, путь Requirement→`RequirementClient`→`Client.name`). Привязка на уровне ТРЕБОВАНИЯ, копируется на каждый его контейнер. Нет связи → `[]`.
- `open_risks` — число незакрытых архрисков узла. **Откуда:** прямой батч-запрос `ArchRisk` (`status != "closed"`, `node_id IN контейнеры`, group by node_id). Узлы arch_map несут `risk_count`, но requirement_detail-контейнеры его не отдают, поэтому считаю по `ArchRisk` напрямую (эквивалентно счётчику узла).

## Реализация
- Новый хелпер `_enrich_containers(session, *, containers, clients)` — мутирует dict'ы контейнеров, добавляя `fix_versions`/`clients`/`open_risks`. Батч-запросы (не N+1): один `EpicTask` по всем task-ключам, один `ArchRisk` по всем node_id.
- В `affected_systems` после `requirement_detail` — второй `_with_session(_enrich_containers, ...)`; `matched_by`/структура `requirements[rid]` без изменений.
- Импорты: `sqlalchemy.func`, `ArchRisk`.

## Тесты
- `test_affected_systems_containers_keep_base_fields` — обогащение не ломает id/components, новые поля есть даже при пустых данных (`[]`/`[]`/`0`).
- `test_affected_systems_enriched_fields` (богатая фикстура: RepoTrack+RequirementJira+EpicTask.fix_versions, 2 ArchRisk open/closed, Client+RequirementClient) → `fix_versions=["24.7","24.8"]`, `clients=["Акме"]`, `open_risks=1` (закрытый риск не считается).

## Ограничения (зафиксировано)
- `fix_versions` зависит от наличия задачи в `EpicTask`: если задача контейнера не попала в срез EpicTask (напр. закрыта до среза) — её релизы не увидим (`[]`). Полнота = полнота EpicTask-среза.
- `clients` — на уровне требования, не пер-контейнер (в схеме нет связи контейнер→клиент напрямую); одинаковы для всех контейнеров одного требования — это ожидаемо.

## Проверки итерации 3 (числами)
- `uv run pytest tests/interface/test_analyst_mcp.py -q` → **40 passed** (1.02s).
- `ruff check` (мои 2 файла) → **All checks passed**.
- `mypy` (config-scoped) → в моих файлах **0 ошибок**.

## Реестр тулов (без изменений, 12): affected_systems теперь с обогащёнными контейнерами.