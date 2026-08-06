---
title: Эталон — состав установки iva-role-analyst 0.3.0 (снято с прода 05.08)
type: note
status: current
created: 2026-08-05 15:45
updated: 2026-08-05 15:45
repo: tacticum-dev
project: iva-write
tags:
- board
- iva-write
- baseline
permalink: tacticum/00-board/baseline-etalon-analyst-030-2026-08-05
---

# Эталон переезда: чем именно будем сравнивать

Требование Президента дословно: *«у нас есть работающий вариант сейчас у роли
аналитика, надо ориентироваться на него»*. Значит правильность переезда доказывается
**сравнением состава с этой установкой**. Ниже — состав, снятый **с прод-каталога
запросом**, а не по памяти и не по манифестам в дереве.

Снято 05.08 ~15:45, сервер `tacticum_prod` (159.194.224.59), БД `tacticum_catalog`,
только SELECT.

## 0. Первая находка — работать надо не от локального дерева

Локальный `main` в `~/tacticum/tacticum-dev` **отстал от origin на 94 коммита**:
в нём `iva-role-analyst` версии **0.2.1 без канала записи вообще**, тогда как в
`origin/main` — **0.3.0 с каналом**. Сверка с локальным деревом дала бы неверный
эталон на первом же шаге.

- локальный `main` = `b612fd9`, `origin/main` = `d607513`;
- канал `helm-iva-write` в локальном main отсутствует (`git grep` пуст), в
  `origin/main` присутствует;
- PR #223 (`merge/iva-write-analyst`, коммит `2eb2307`) в origin/main **есть**, в
  локальном — **нет**.

Работаю в отдельном рабочем дереве `~/tacticum-worktrees/iva-write-lane` от
`origin/main`, ветка `feat/iva-write-base-lane`.

## 1. Живая установка, которую нельзя сломать

| Поле | Значение |
|---|---|
| профиль | `iva-role-analyst` |
| пин | **0.3.0** |
| фактически синхронизировано | **0.3.0** |
| проходов синка | 5 |
| последний синк | **05.08.2026** |
| архивная | нет |
| метка | `Codex work1 analyst role` |

Это та самая установка, на которой канал доказан живым человеком (17 проходов
канала). Пин и синк совпадают — расхождения «обновление есть, а у человека нет»
сейчас НЕТ, и после переезда его быть тоже не должно.

## 2. Состав установки — эталон для сравнения

### Роль `iva-role-analyst` 0.3.0 — собственные ингредиенты (4)

| ingredient_id | kind | tier |
|---|---|---|
| `claude-md-fragment` | instruction_pack | trial |
| `codex-agents-md` | instruction_pack | trial |
| `codex-config-toml` | instruction_pack | trial |
| `claude-settings` | repo_config | trial |

Три из четырёх — **instruction_pack, и именно в них сегодня лежат тексты про канал
записи**. Это и есть то, что переезжает.

### Композиция роли (`depends_on`, разрешённая в каталоге)

| Лейн | Версия |
|---|---|
| `tacticum-core-base` | 0.5.0 |
| `tacticum-analysis-core` | 0.3.0 |
| `iva-analysis-fr` | **0.2.0** |
| `tacticum-research-base` | 0.1.0 |

### Лейн `iva-analysis-fr` 0.2.0 — 15 ингредиентов

| ingredient_id | kind |
|---|---|
| `system-analyst` | agent_spec |
| `prepare-tz` | command_spec |
| `start-feature` | command_spec |
| `update-feature` | command_spec |
| **`helm-iva-write`** | **mcp_server_spec** ← переезжает |
| `helm-process` | mcp_server_spec |
| `iva-atlassian-write` | mcp_server_spec |
| `api-contracts-discovery` | skill_spec |
| `data-model-analyzer` | skill_spec |
| `design-system-discovery` | skill_spec |
| `events-analyzer` | skill_spec |
| `fr-authoring` | skill_spec |
| `mockup-authoring` | skill_spec |
| `process-analysis-stage` | skill_spec |
| `process-arch-signoff` | skill_spec |

**Итого в установке аналитика сегодня: 4 + 15 + (core-base + analysis-core +
research-base) ингредиентов.** После переезда сумма обязана остаться той же, изменится
только ОТКУДА приезжает `helm-iva-write` и тексты к нему.

## 3. Имя `iva-write-base` — чистое, не отравлено

Проверено запросом к прод-каталогу: профиля с `id ilike '%write%'` всего два, и это
`iva-techwriter-mcp` и `tacticum-role-techwriter`. **Профиля `iva-write-base` в
каталоге НЕТ** — ни активного, ни архивного, ни ретракнутого.

Это важно: в комментарии `iva-analysis-fr/manifest.yaml` есть след «удалённый лейн
iva-write-base», и была опасность коллизии `profile_id` при сиде. Коллизии не будет —
в БД имени нет. (История удаления разбирается отдельно, разведка идёт.)

## 4. Что из этого следует для проверки

Переезд считается верным, если после правки:

1. **набор ингредиентов установки аналитика совпал** с таблицами выше — тот же
   состав, ни одного лишнего, ни одного потерянного;
2. `helm-iva-write` приезжает из `iva-write-base`, а не из `iva-analysis-fr`;
3. адрес канала остался `https://helm.tacticum.ru/mcp/iva-write` (ловушка прежнего
   шлюза — в граблях, п. 9);
4. тексты про канал доехали до итоговых `CLAUDE.md` / `AGENTS.md` / `.codex/config.toml`
   — не «ингредиент объявлен», а **текст оказался в файле у человека** (грабли, п. 1:
   проверяем свойство, а не механику).

## Как повторить запросы

```sql
-- композиция роли
select bp.profile_id, bp.version from profile_version_dependencies d
join profile_versions dv on dv.id=d.dependent_version_id
join profile_versions bp on bp.id=d.base_version_id
where dv.profile_id='iva-role-analyst' and dv.version='0.3.0';

-- ингредиенты версии
select i.ingredient_id, i.kind, i.tier from ingredients i
join profile_versions pv on pv.id=i.profile_version_id
where pv.profile_id='iva-role-analyst' and pv.version='0.3.0';

-- живые установки
select profile_id, profile_version_pinned, last_synced_version, sync_count,
       last_synced_at, archived_at from installations
where profile_id='iva-role-analyst';
```

Доступ: `tacticum_prod` → `docker exec tacticum-postgres-1 psql -U catalog -d tacticum_catalog`.

## Ссылки

[[postanovka-dve-raboty-iva-write-2026-08-05]] ·
[[grabli-iva-write-na-chto-ne-nastupat-2026-08-05]] · [[report-iva-write]]
