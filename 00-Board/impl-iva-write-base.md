---
title: impl-iva-write-base
type: note
permalink: tacticum/00-board/impl-iva-write-base
status: draft
role: implementer
deliverable: 2A (скелет лейна iva-write-base)
tags:
- implementer
- iva-write
---

# impl-iva-write-base

Собран **скелет leaf-лейна `iva-write-base`** (2A, Вариант A) по спеке [[Дизайн-спека iva-write-base (Вариант A) — на апрув]]. Строго аддитивно, analysis не тронут. НЕ провижнил / НЕ сидил / НЕ пушил.

## Worktree / ветка
- Worktree: `/Users/bubblemac/tacticum/tacticum-dev-iva-write` (изолирован, вне основного дерева).
- Ветка: `feat/iva-write-base`, коммит `cf8dd61`.

## Созданные файлы (только новые, 173 строки)
- `templates/iva-write-base/manifest.yaml` — schema v2, leaf (БЕЗ `depends_on`), `stack.required/optional: []`. Шапка сверена с не-guardrail образцом `iva-go-development-base`: schema_version, profile_id=iva-write-base, name, version 0.1.0, maintainer mr.diaret@ya.ru, license MIT, deprecated:false, description, persona{role: requirements-author, scope}, target_tasks, ide_targets (claude/codex full), profiles{trial,full}, post_install_notes, non_goals.
- `templates/iva-write-base/README.md` — назначение (продакшн write-канал публикации FR в Confluence/Jira под подписью актора), модель auth Варианта A (один клиентский phk_/TACTICUM_TOKEN, актор из gateway X-Auth-User-Id, write-PAT серверный), scope iva-req-write, аддитивность (ingredient_id `iva-write`, НЕ `iva-read`).
- `templates/iva-write-base/CHANGELOG.md` — запись 0.1.0 (скелет).

Единственный ингредиент — точно по Варианту A:
```yaml
- ingredient_id: iva-write
  kind: mcp_server_spec
  tier: trial
  supports: [claude-code, codex]
  install_scope: repo
  body: ""
  metadata:
    transport: http
    url: "https://mcp.tacticum.ru/iva-write/mcp"
    env_required: [TACTICUM_TOKEN]
    auth_type: bearer
    required_scopes: [iva-req-write]
    allowed_tools: [confluence_create_page, confluence_update_page, jira_create_issue, jira_add_comment, jira_transition_issue]
```

## Валидация по схеме — PASS (реальный вывод)
Разовый jsonschema-скрипт (Draft7, повторяет логику `apps/backend/tests/catalog/test_manifest_schemas.py`): manifest против `manifest.v2.schema.json`, ингредиент против `ingredient.v1.schema.json`. Прогон `uv run --with jsonschema --with pyyaml`:
```
has depends_on: False
stack: {'required': [], 'optional': []}
ingredients: ['iva-write']
iva-write metadata keys: ['allowed_tools','auth_type','env_required','required_scopes','transport','url']
  required_scopes: ['iva-req-write']
VALIDATION: PASS (manifest.v2 + ingredient.v1, all ingredients)
```

## Флаги / расхождения
- **required_scopes — схемой ДОПУСКАЕТСЯ (флаг снят).** `ingredient.v1.schema.json` строка 103 явно объявляет `"required_scopes": {"type":"array","items":{"type":"string"}}` в ветке `mcp_server_spec`. Валидатор не strict на верхнем уровне (нет `additionalProperties:false`), но поле и так в схеме → непустой `[iva-req-write]` НЕ ломает валидацию. Схему НЕ трогал. Остаётся риск из спеки (не для схемы): dev-сидер поле не читает, энфорсит gateway-registry — проверить, что сид не падает на непустом поле (это уже provision-этап, вне 2A-скелета).
- **manifest.v2.schema.json** валидирует только верхний уровень (schema_version/depends_on/ide_targets); ингредиенты валидируются отдельно против ingredient.v1 (как в seed_profile). Обе схемы — PASS.
- Роль в `ROLE_LANES` теста НЕ добавлял (это 2B).
- Guardrail соблюдён: файлы `iva-analysis-base/*`, `iva-role-analyst/*` не изменялись (git показывает только 3 новых файла в `templates/iva-write-base/`).

## Связано
- [[Дизайн-спека iva-write-base (Вариант A) — на апрув]] · [[explore-iva-write-base]]