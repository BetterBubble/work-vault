---
title: Дизайн-спека iva-write-base (Вариант A) — на апрув
type: report
permalink: tacticum/20-architecture/dizain-speka-iva-write-base-variant-a-na-apruv
status: for-approval
role: lead (тимлид)
date: 2026-07-21
tags:
- spec
- iva-write
- 2a
- design
---

# Дизайн-спека: лейн `iva-write-base` (2A, Вариант A)

Синтез из [[explore-iva-write-base]] + [[explore-iva-write-multikey]] + решений [[Решения по 2A iva-write-base (2026-07-21)]]. Модель — Вариант A (одобрен пользователем).

## Ключевая идея
Клиент несёт **один** ключ `phk_` (`TACTICUM_TOKEN`). Gateway резолвит его → инжектит актора (`X-Auth-User-Id`) = **подпись актора**. Write-credential (PAT) и Allure-токен — **серверные** (env mcp-atlassian / helm), в манифест не попадают. → Манифест лейна **одинаков** при обеих auth-моделях ADR-грилла (техучётка vs личный PAT различаются только серверным env). Мульти-ключевость реализована архитектурой gateway, а не клиентским манифестом.

## Структура лейна (аддитивно, analysis НЕ трогаем)
`templates/iva-write-base/` (leaf, БЕЗ depends_on):
- `manifest.yaml` (schema v2), `README.md`, `CHANGELOG.md`. Каталог `ingredients/` не нужен (только server-spec).

## Ингредиент (единственный)
```yaml
- ingredient_id: iva-write            # НЕ iva-read/iva-mcp (иначе коллизия с analysis-base)
  kind: mcp_server_spec
  tier: trial
  supports: [claude-code, codex]
  install_scope: repo
  body: ""
  metadata:
    transport: http
    url: "https://mcp.tacticum.ru/iva-write/mcp"   # целевой; физически ждёт Ф1 ADR-0058
    env_required: [TACTICUM_TOKEN]                 # один клиентский ключ
    auth_type: bearer
    required_scopes: [iva-req-write]               # первое применение поля (см. риск)
    allowed_tools:                                 # сужение до write
      - confluence_create_page
      - confluence_update_page
      - jira_create_issue
      - jira_add_comment
      - jira_transition_issue
      # + Allure-статус-тул, если он на этом же сервере (уточнить в PoC)
```

## Композиция ролей (2B — только фиксируем)
- techwriter = core + documentation-base + **iva-write-base**.
- architect/qa = core + analysis-base + **iva-write-base** (write избыточен с analysis, но guardrail цел; дедуп — потом, при снятии guardrail).
- ⚠️ **fr-authoring остаётся в analysis** (guardrail). Открытый вопрос 2B: techwriter (без analysis) получит write-**тулы**, но не скилл-оркестратор fr-authoring. Решить в 2B — нужен ли techwriter'у свой write-скилл или облегчённый fr-authoring в documentation-base.

## PoC (песочница)
create page + create issue + transition на песочном Jira/Confluence; проверить **нативную атрибуцию актора**. Нужен: песочный endpoint + `phk_` со scope `iva-req-write`. ⚠️ Целевой `iva-write` endpoint физически ещё не поднят (Ф1 ADR-0058) → PoC либо через существующий `/iva-read/mcp` (те же write-тулы) как прокси-проверку атрибуции, либо ждёт песочный инстанс. Нужен доступ/ключ от пользователя.

## Риски / открытое
1. `required_scopes` — в схеме есть, dev-сидер его НЕ читает, но gateway-registry энфорсит. Проверить, что сид не падает на непустом поле.
2. Целевой write-endpoint не существует → PoC атрибуции делаем на доступном канале, прод ждёт Монахова.
3. Финал auth-модели (техучётка/личный PAT) — по гриллу ADR-0058 сегодня; на манифест НЕ влияет, только на серверный деплой.

## Связано
- [[explore-iva-write-multikey]] · [[Решения по 2A iva-write-base (2026-07-21)]] · [[plan-tri-deliveravla-iva-iva-write-3-roli-my-todo-priviazka-k-sisteme]]