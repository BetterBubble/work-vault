---
title: explore-iva-write-base
type: note
permalink: tacticum/00-board/explore-iva-write-base-1
status: draft
role: explorer
deliverable: 2A (дизайн-спека лейна iva-write-base)
tags:
- explorer
- iva-write
- helm
archived-at: 2026-08-03 11:16
---

# explore-iva-write-base

Разведка грунта под дизайн-спеку leaf-лейна `iva-write-base`. Read-only. Канон не пишу.

Связь: [[План- три деливеравла ИВА (iva-write - 3 роли - my_todo) — привязка к системе]]

> GUARDRAIL: analysis-сущности (`iva-analysis-base`, `iva-role-analyst`, их скиллы/манифесты) сегодня в проде у Diaret (обучение аналитиков) — **не трогать без эскалации**. Ниже читаю их только как образец.

## 0. Репо и якоря

- Репо: **`/Users/bubblemac/tacticum/tacticum-dev`** (private, monorepo `apps/`, ADR-0016). Не git-репо на верхнем уровне `~/tacticum`, но сам tacticum-dev — рабочее дерево.
- Схема манифеста: `templates/_schema/manifest.v2.schema.json` + `templates/_schema/ingredient.v1.schema.json`.
- Тест ролей-пресетов: `apps/backend/tests/catalog/test_iva_role_presets.py`.
- В tacticum-dev действует HARD RULE: Serena для навигации/правок кода (для .md/.yaml/.json — Read ок).

## 1. fr-authoring — что это и какие тулы зовёт (БАЗА, которую продакшним)

fr-authoring — это **skill_spec** (не роль, не лейн), присутствует в двух лейнах:
- `templates/iva-analysis-base/ingredients/skills/fr-authoring/SKILL.md` (образец под guardrail — читать, не трогать)
- `templates/iva-fr-analyst/ingredients/skills/fr-authoring/SKILL.md` (старый профиль iva-fr-analyst, US #683)

Точные **write-тулы iva-mcp**, которые зовёт скилл (SKILL.md стр. 59, 154-166):
- `confluence_create_page` / `confluence_update_page` — публикация полного FR (wiki)
- `jira_create_issue` / `jira_add_comment` — компактный ТЗ-блок в Jira (label `ready-for-dev`)

Read-тулы того же сервера (для контекста): `analyst_search`/`related_tasks` идут из helm-analyst, не iva-mcp. iva-mcp = Jira 10.3 / Confluence 9.2 через platform MCP gateway.

**mcp_server_spec, к которому привязан fr-authoring сейчас** (это то, что «продакшним»):

В `iva-fr-analyst/manifest.yaml` (стр. 114-124) ингредиент назван `iva-mcp`:
```yaml
- ingredient_id: iva-mcp
  kind: mcp_server_spec
  tier: trial
  supports: [claude-code, codex]
  install_scope: repo
  body: ""
  metadata:
    transport: http
    url: "https://mcp.tacticum.ru/iva-read/mcp"   # ВНИМАНИЕ: URL всё ещё /iva-read
    env_required: [TACTICUM_TOKEN]
    auth_type: bearer
```
Комментарий в манифесте прямо говорит: «Профилю нужны и read-, и write-тулы (confluence_create_page / jira_create_issue / jira_add_comment) — публикация FR. Тот же phk_* ключ.»

В `iva-analysis-base/manifest.yaml` (стр. 257-267) тот же сервер оформлен как ингредиент `iva-read` (тот же URL `https://mcp.tacticum.ru/iva-read/mcp`, тот же `TACTICUM_TOKEN`, bearer). Т.е. **сейчас write физически идёт через тот же endpoint iva-read под личным phk_*** — отдельного iva-write endpoint ещё нет.

## 2. Образец lane с mcp_server_spec (по нему делаем iva-write-base)

Минимальный образец ингредиента mcp_server_spec (из iva-analysis-base / iva-fr-analyst):
```yaml
- ingredient_id: <name>
  kind: mcp_server_spec
  tier: trial
  supports: [claude-code, codex]
  install_scope: repo
  body: ""
  metadata:
    transport: http            # enum: stdio|http|sse
    url: "https://..."
    env_required: [TACTICUM_TOKEN]   # массив имён env-переменных
    auth_type: bearer          # enum: none|bearer|basic|oauth
    # allowed_tools: [...]      # опц. — белый список тулов сервера (см. §5)
```
Пример с `allowed_tools` (ограничение набора тулов) — `templates/tacticum-platform-dev/manifest.yaml:78-82`:
```yaml
    allowed_tools:
      - get_applicable_standards
      - check_compliance
      - search_artifacts
```
Ещё примеры allowed_tools: `iva-kmp-brownfield/manifest.yaml:538` `["codegraph_explore"]` (и rn/mail-brownfield).

## 3. Образец leaf-лейна и роли (структура нового лейна)

**Leaf-лейн** (база, НЕ несёт depends_on) — образец `iva-analysis-base` (guardrail: читать):
- Структура каталога: `templates/<profile-id>/` = `manifest.yaml` + `README.md` + `CHANGELOG.md` + `ingredients/` (skills/agents/commands/agents-codex/repo-configs).
- В шапке manifest: `schema_version: "2"`, `profile_id`, `name`, `version`, `maintainer`, `license`, `deprecated: false`, `persona{role,scope}`, `target_tasks`, `stack{required:[],optional:[]}`, `ide_targets`, `profiles{trial,full}`, `ingredients: [...]`, `post_install_notes`, `non_goals`.
- Лейн БЕЗ `depends_on` (он сам база).

**Роль-пресет** (leaf, `ingredients: []`, depth-1 depends_on) — образцы `iva-role-analyst` (guardrail), `iva-role-go`:
- `iva-role-analyst/manifest.yaml`: `ingredients: []`, `depends_on: [tacticum-core-base, iva-analysis-base]`, `persona.role: analyst`.
- `iva-role-go`: `depends_on: [tacticum-core-base, iva-analysis-base, iva-go-development-base, tacticum-bugfix-base, tacticum-documentation-base]`.
- Каталог роли минимальный: только `manifest.yaml` + `README.md` + `CHANGELOG.md` (нет `ingredients/`, нет `scripts/`).

## 4. Схема mcp_server_spec (manifest.v2 / ingredient.v1)

`templates/_schema/ingredient.v1.schema.json` стр. 88-108, ветка `kind == mcp_server_spec`, `metadata` (required: `transport`):
- `transport`: enum `stdio|http|sse`
- `command`: string|null (для stdio)
- `url`: string|null
- `args`: array<string>
- `env_required`: **array<string>** — имена env-переменных (не значения)
- `auth_type`: enum `none|bearer|basic|oauth` — **ОДИН на сервер**
- `allowed_tools`: array<string> — белый список тулов
- `required_scopes`: array<string> — **есть в схеме, но НИГДЕ не используется** (0 вхождений в templates)

Вывод по мульти-ключам на уровне схемы: `env_required` — это плоский массив имён, поддерживает **несколько env-переменных** на один сервер. Но `auth_type` — скаляр (один тип аутентификации на сервер). Отдельной структуры «ключ→scope→auth per key» в схеме НЕТ.

## 5. Мульти-ключевая схема — прецедента НЕТ

Прогон по всем `env_required` в манифестах: **везде ровно ОДИН env-ключ** на сервер:
- `[TACTICUM_TOKEN]` — helm-analyst, iva-read/iva-mcp, tacticum-mcp (личный phk_*)
- `[GITHUB_TOKEN]` — iva-2-client-shell-dev
- `[TACTICUM_MCP_TOKEN]` — tacticum-platform-dev

Ни одного случая массива из 2+ ключей. Т.е. схема-то массив допускает (`env_required: [PHK_TOKEN, JIRA_PAT, ...]`), но **прецедента мульти-ключа нет** — это будет новая конструкция/паттерн, требующий решения: как несколько ключей разного назначения (hub + PAT) уживаются при одном `auth_type`. `required_scopes` в схеме есть, но не задействован — можно первым применить для декларации scope (напр. `iva-req-write` из ADR-0058).

## 6. Провижн и тесты

**Провижн:** лейны/роли (analysis-base, role-analyst, role-go) НЕ имеют `scripts/apply.ps1` — провижнятся через сид в Postgres:
- Сидер: `apps/backend/src/backend/catalog/application/seed_profile.py`; CLI-обёртка `apps/backend/scripts/seed_community.py`; провижн установки — `apps/backend/src/backend/workspace/application/provision_installation.py`.
- `apply.ps1` есть только у старых brownfield/dev-base профилей — для новых лейнов не нужен.
- Будущий `tacticum-cli apply` (epic E8) — не начат.

**Тест `test_iva_role_presets.py`** — файловый (без БД), проверяет для ролей из словаря `ROLE_LANES`:
- `test_manifest_validates_against_v2_schema` — валидация по схеме v2
- `test_role_is_pure_composition_leaf` — `ingredients == []`, persona.role
- `test_depends_on_is_the_declared_lanes_in_order` — depends_on == объявленный список, каждый лейн существует
- `test_lanes_are_depth1_bases` — ни один лейн не несёт своего depends_on (иначе depth-2 → reject at seed)
- `test_single_owner_lanes_are_pairwise_disjoint` — **лейны роли попарно НЕ пересекаются по ingredient_id**
- `test_golden_parity_union_equals_sum_of_lanes` — размер union == сумма размеров лейнов (строгая дизъюнктность)
- `test_ide_targets_claude_and_codex_full`

Что нужно добавить при вводе iva-write-base: если роль будет композить его через depends_on, обновить словарь `ROLE_LANES` (стр. 40-49) — и обеспечить single-owner disjointness (см. риск ниже). Смежные тесты: `test_seed_depends_on.py`, `test_composition.py`, `test_manifest_schemas.py`.

## Что нужно для manifest iva-write-base + мульти-ключевая схема

Минимальный набор для нового leaf-лейна `templates/iva-write-base/`:
1. `manifest.yaml` (schema v2, БЕЗ depends_on, `stack.required/optional: []`), `README.md`, `CHANGELOG.md`.
2. Ингредиент `kind: mcp_server_spec`, `ingredient_id: iva-write` (НЕ `iva-read`/`iva-mcp` — иначе коллизия с analysis-base, см. риск), `transport: http`, целевой `url` write-endpoint, `auth_type: bearer`, `env_required: [...]`, опц. `allowed_tools: [confluence_create_page, confluence_update_page, jira_create_issue, jira_add_comment, ...]` — сузить до write-тулов.
3. Продакшн fr-authoring: сейчас скилл лежит в iva-analysis-base — под guardrail. Решить с тимлидом, дублируется ли fr-authoring в write-lane или остаётся в analysis (single-owner запрещает один ingredient_id в двух лейнах одной роли!).
4. Мульти-ключ: если реально 3-4 ключа (hub phk_* + Jira PAT + Confluence PAT + Allure PAT) — оформить как `env_required: [PHK_TOKEN, JIRA_PAT, CONFLUENCE_PAT, ALLURE_PAT]`. Но: это новый паттерн (нет прецедента), и `auth_type` один — модель «hub-ключ для актора + PAT техучётки для записи» в текущую схему одним `auth_type` не ложится чисто. Возможно нужен либо новый field, либо семантика «gateway разбирает несколько env».

## Риски / открытые вопросы

1. **КРИТИЧНО — конфликт с ADR-0058.** `docs/adr/0058-requirement-as-jira-us-ivareq-iva-write-role-profiles.md` (статус **Proposed**, грилл-сессия с владельцем **2026-07-21 = сегодня**) Решение 5 прямо ПЕРЕСМАТРИВАЕТ Taiga #712: iva-write = **PAT ТЕХНИЧЕСКОЙ учётки `iva`** (инстанс mcp-atlassian на adp_emb за gateway `mcp.tacticum.ru`), с **принудительной подписью актора** (email из hub-ключа, ADR-0051), scope `iva-req-write`, права только IVAREQ + новый Confluence-space. Явно отвергнут «личный PAT каждому (Taiga #712 as-is)» как немасштабируемый (таблица альтернатив, стр. 136). **Это противоречит контексту задачи** («аутентификация — личный PAT сотрудника», «3-4 ключа на адрес»). Нужна эскалация тимлиду: спека 2A строится по #712 (личный PAT + мульти-ключ) или по ADR-0058 (техучётка + подпись актора)? От этого зависит вся конструкция mcp_server_spec и мульти-ключей.
2. **Single-owner коллизия fr-authoring.** fr-authoring уже принадлежит iva-analysis-base. Если роль (напр. iva-role-analyst) композит и analysis-base, и iva-write-base, а write-lane тоже несёт fr-authoring / иной общий ingredient_id → падёт `test_single_owner_lanes_are_pairwise_disjoint` + `golden_parity`. Аналогично ingredient_id `iva-read` уже занят в analysis-base — новый сервер обязан называться иначе (`iva-write`).
3. **Guardrail vs продакшн.** Продакшн fr-authoring подразумевает правку самого скилла/его привязки, но analysis-сущности под guardrail (Diaret обучает сегодня). Любое касание iva-analysis-base/iva-role-analyst — только через эскалацию. Возможно write-lane нужно сделать аддитивно, не трогая analysis.
4. **URL-долг.** Действующий write идёт через `https://mcp.tacticum.ru/iva-read/mcp` (имя endpoint = read, хотя зовутся write-тулы). Целевой write-endpoint (по ADR-0058 — отдельный инстанс mcp-atlassian) ещё не существует физически — Ф1 ADR-0058 («создать iva-write») не выполнена. Уточнить фактический URL write-сервера.
5. **required_scopes не обкатан.** Поле есть в схеме, но 0 использований — если закладываем scope `iva-req-write`, будем первыми; проверить, читает ли сидер/провижн это поле вообще (не проверял код seed_profile на предмет required_scopes).
6. **allowed_tools для write.** Стоит ли сузить write-lane до конкретных write-тулов (безопасность/минимальные права по ADR-0058) — решение дизайнера спеки.