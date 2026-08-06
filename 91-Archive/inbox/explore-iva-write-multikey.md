---
title: explore-iva-write-multikey
type: note
permalink: tacticum/00-board/explore-iva-write-multikey-1
status: draft
role: explorer
deliverable: 2A (мульти-ключевая аутентификация iva-write-base)
tags:
- explorer
- iva-write
- auth
archived-at: 2026-08-03 11:16
---

# explore-iva-write-multikey

Read-only разведка механики мульти-ключевой аутентификации под дизайн лейна `iva-write-base` (2A). Канон не пишу.

Связь: [[explore-iva-write-base]] · [[Решения по 2A iva-write-base (2026-07-21)]] · [[⚠️ Конфликт 2A iva-write ↔ ADR-0058 (личный PAT) — развести до дизайна]]

> GUARDRAIL: analysis-сущности читались только как образец, не трогались.

## ГЛАВНЫЙ ВЫВОД (сразу)

Архитектура gateway `mcp.tacticum.ru` **уже реализует 2-ключевую модель, но ключи разнесены по двум местам**, а не лежат вместе в манифесте:
1. **Клиентская сторона — РОВНО ОДИН ключ:** hub `phk_*` (`TACTICUM_TOKEN`) в Bearer. Gateway forward-auth резолвит его в project-hub и **сам добывает актора** (email) → это и есть «подпись актора».
2. **Серверная сторона (downstream):** gateway **срезает Bearer** (middleware `mcp-drop-auth`), а backend `mcp-atlassian` пишет по **сервисному PAT из своего env** (техучётка `iva` по ADR-0058). Этот PAT в манифест НЕ кладётся.

Плюс жёсткое ограничение рендерера (ниже): при `auth_type: bearer` для http-сервера в клиентский конфиг попадает **только `env_required[0]`**; остальные ключи массива **молча отбрасываются**. Т.е. «4 ключа на одном bearer-http mcp_server_spec» текущая схема+рендерер физически НЕ доставит.

---

## 1. ADR-0051 — как gateway извлекает актора (подпись)

Путь: `/Users/bubblemac/tacticum/tacticum-dev/docs/adr/0051-project-hub-as-single-idp-and-rbac-authority.md`

- project-hub = единый IdP + RBAC. Sister-MCP резолвят идентичность через `POST /api/internal/resolve` (service-key в заголовке + **user phk_-токен в теле**) → `ResolveOut{user_id, email, tenant, memberships[{role, capabilities}], capabilities}` (стр. 16, 57).
- Двухслойная авторизация: сервис аутентифицируется своим service-key (scope `resolve`); user-токен валидируется в теле resolve; identity берётся ТОЛЬКО из ответа, никогда из client-supplied полей.
- Механизм `resolve-by-token` реализован (Phase 2, US#598, в проде): `require_hub_user` → `/api/internal/resolve` одним вызовом отдаёт identity+memberships+capabilities+plan (стр. 67-69).

**Реализация резолва в gateway** (platform):
- `platform/services/runtime/mcp_runtime/src/mcp_runtime/infrastructure/project_hub.py:64` `resolve_token()` — POST `/api/internal/resolve`, тело `{"token": phk_}`, заголовок `Bearer <service_key>`. Возвращает `ResolvedIdentity{user_id, email, tenant_id, memberships, capabilities}` (стр. 32-61).
- `.../infrastructure/resolver.py:15` `ProjectHubResolver.resolve()` → `Identity{user_id, tenant_id, project_id, scopes=capabilities}`.
- Fallback-цепочка `FallbackResolver` (resolver.py:38): hub и Dev-catalog делят префикс `phk_`, поэтому маршрутизация по префиксу невозможна — неизвестный хабу токен пробуется в Dev-каталоге (`DevCatalogResolver`, `dev_catalog.py:73`, ключи `tac_*`).

## 2. ADR-0058 — модель iva-write (требования к ключам)

Путь: `/Users/bubblemac/tacticum/tacticum-dev/docs/adr/0058-requirement-as-jira-us-ivareq-iva-write-role-profiles.md` (Proposed, грилл 2026-07-21).

**Решение 5 (стр. 53-68) — дословные требования:**
- Новый MCP-сервер `iva-write` = **инстанс mcp-atlassian на adp_emb, за gateway `mcp.tacticum.ru`**.
- Аутентификация записи = **PAT технической учётки `iva`**; права УЗ — только проект IVAREQ + новый Confluence-space; расширение — только новым решением.
- **Каждое действие обязано нести актора**: сервер требует контекст (email из hub-ключа, ADR-0051) и дописывает его в артефакт («через профиль <роль>, инициатор: <email>»); без подписи — **отказ**; лог тул-вызовов на gateway.
- Доступ по ролям через hub-ключи, **scope `iva-req-write`**: аналитик — постановки+тело US; разработчик — комменты/статусы; QA — статусы испытаний; техпис — доки.
- **PAT хранится в env сервиса (не в git, не в профилях).**
- Явно отвергнут «личный PAT каждому (Taiga #712)» — немасштабируемо (Альтернативы, стр. 136).
- Последствие/риск (стр. 152): «подпись актора требует **доработки mcp-atlassian (форк/обвязка)** — оценить в Ф1». Ф1 (создать IVAREQ+space+iva-write) ещё НЕ выполнена.

Итог по ключам ADR-0058: **клиенту — 1 hub-ключ (phk_, scope iva-req-write); write-PAT — серверный, в env сервиса iva-write; актор — из hub-ключа gateway'ем.**

## 3. Gateway `mcp.tacticum.ru` — где конфиг и как маршрутит

Репо: **`platform/services/runtime/mcp_runtime/`** (сервис `mcp-runtime`). НЕ в tacticum-dev, НЕ в helm.

- Traefik dynamic config: `deploy/traefik-mcp-gateway.example.yml` (на хосте = `~tacticum-deploy/gateway/dynamic/mcp.yml`). Прод живёт с 2026-06-26 (`deploy/PHASE1B.md`).
- **Маршрут iva-read** (traefik-...yml:45-49):
  ```
  rule: Host(`mcp.tacticum.ru`) && PathPrefix(`/iva-read`)
  service: svc-iva-read → http://172.20.0.1:19010  (обратный SSH-туннель к mcp-atlassian на ADP)
  middlewares: [mcp-forward-auth, mcp-iva-read-ratelimit, mcp-drop-auth, mcp-strip-iva-read]
  ```
- **Цепочка middleware — ядро 2-ключевой модели по факту:**
  1. `mcp-forward-auth` (стр. 10-13) → `http://mcp-runtime:8095/forward-auth`; на 200 копирует upstream identity-заголовки `authResponseHeaders: [X-Auth-User-Id, X-Tenant-Id, X-Project-Id, X-Scopes]`.
  2. `mcp-drop-auth` (стр. 18-19) `Authorization: ""` — **комментарий дословно: «после forward-auth снимаем Bearer: backend mcp-atlassian ходит по сервисному PAT из env»**.
  3. `mcp-strip-iva-read` — снимает префикс пути.
- **forward-auth логика:** `.../application/forward_auth.py:16`:
  - `registry.get(server)` → нет → 404;
  - `resolver.resolve(token)` → нет → 401;
  - `visible_to(entry, identity)` (tenant-гейт) → 403;
  - `missing_scopes(entry, identity)` → 403;
  - allow → инжектит заголовки `X-Auth-User-Id=identity.user_id (email)`, `X-Tenant-Id`, `X-Project-Id`, `X-Scopes` (стр. 42-51).
- **Downstream доверяет заголовкам, сам не резолвит** (docstring forward_auth.py:6-7; header-trust граница — сетевая, backend без портов наружу, `PHASE1B.md`). Для memory/knowledge флаг `*_TRUST_GATEWAY_HEADERS=true`; для iva-read/write — mcp-atlassian должен читать `X-Auth-User-Id` (это и есть точка «подписи актора» — требует форк/обвязку per ADR-0058).

Вывод §3: **схема «hub-ключ для identity + downstream-PAT для записи» УЖЕ работает** для iva-read. iva-write = тот же паттерн с другим сервисным PAT (техучётка iva) + обязательным чтением X-Auth-User-Id.

## 4. mcp-atlassian — конфигурация PAT

- Исходников mcp-atlassian в этих репозиториях НЕТ (сторонний сервер, живёт на ADP/adp_emb, доступен через SSH-туннель `172.20.0.1:19010`). Конфиг PAT — в env инстанса на хосте, вне git (по ADR-0058 стр. 65 и `PHASE1B.md`: секреты в host `.env`).
- Для iva-write ADR-0058 требует ОТДЕЛЬНЫЙ инстанс mcp-atlassian с PAT техучётки `iva` (права только IVAREQ + новый space). Downstream credential — один сервисный PAT на инстанс; поддержки «нескольких downstream-креды/актора» в самом mcp-atlassian по коду здесь не проверить (нет исходников) — актор приходит только заголовком от gateway, форк/обвязка обязателен.

## 5. Allure-PAT и «tactic» — что это и куда относятся

**Allure = серверный READ-токен в helm, НЕ клиентский ключ:**
- `helm/src/helm/config.py:354-355` `allure_token` (`Authorization: Api-Token <token>`, секрет только в `.env`).
- `helm/docs/superpowers/specs/2026-07-18-allure-testops-read-integration-design.md:52` — `HELM_ALLURE_BASE_URL/HELM_ALLURE_TOKEN/HELM_ALLURE_PROJECT_ID` в `/opt/helm/.env`.
- Используется ингестом `helm/src/helm/ingest/allure_snapshot.py:128` и тулом `requirement_tests` (helm-analyst, read). Клиент/агент Allure-токен НЕ держит — он на сервере helm.
- Сервер helm-analyst: `https://helm.tacticum.ru/mcp/analyst`, клиентский ключ = `TACTICUM_TOKEN` (phk_).

**«tactic» = тот же hub phk_ (`TACTICUM_TOKEN`)**, он же покрывает tacticum-mcp (KB, `mcp.tacticum.dev/mcp`), helm-analyst и iva-read (см. `templates/iva-go-backend-brownfield/manifest.yaml:421-461` — ТРИ сервера, у каждого `env_required: [TACTICUM_TOKEN]`, один и тот же phk_). Отдельная сущность `TACTICUM_MCP_TOKEN` — тот же phk_ под другим именем в `tacticum-platform-dev`.

Вывод §5: Allure и «tactic» — **на РАЗНЫХ серверах, не на iva-write mcp_server_spec**. Allure — серверный токен helm-analyst; tactic — тот же клиентский phk_. На iva-write они не нужны как отдельные клиентские ключи.

## 6. Как env_required реально потребляется (рендереры) — КЛЮЧЕВОЕ ОГРАНИЧЕНИЕ

Потребление НЕ в seed/provision (там валидация), а в **рендерерах** конфигов клиентов:
- `apps/backend/src/backend/catalog/domain/renderer.py:86-92`:
  ```python
  if auth_type == "bearer" and env_required:
      value["headers"] = {"Authorization": f"Bearer ${{{env_required[0]}}}"}  # ТОЛЬКО [0]
  elif env_required:
      value["env"] = {k: ... for k in env_required}   # весь массив — только НЕ-bearer / stdio
  ```
- То же в `renderers/claude_code.py:119-122`, `renderers/codex.py:159-160` (`bearer_token_env_var = env_required[0]`), `opencode.py:83-84`, `gemini.py:90-91`, `copilot.py:147-148`.
- **Следствие:** для http+bearer доставляется РОВНО ОДИН ключ (`env_required[0]` → Bearer). `env_required[1:]` в клиентский конфиг НЕ попадают. Несколько env-ключей рендерятся только для **stdio** серверов или когда `auth_type != bearer` (тогда `env`-словарь).
- `required_scopes`: поле есть в доменной модели `apps/backend/src/backend/catalog/domain/ingredients/mcp_server_spec.py:24`, но в рендерерах/сидере dev НЕ читается. Оно потребляется **на стороне gateway**: `mcp_runtime/domain/models.py:37` `RegistryEntry.required_scopes` (docstring: «schema lifted from Dev mcp_server_spec») → гейт `domain/scope.py:12 missing_scopes()` в forward-auth. Реестр gateway — JSON `registry.json`, монтируется в контейнер (`registry_store.py`, `config.py:19`), сид живёт на deploy-хосте (вне этих репо).

Вывод §6: объявить `required_scopes: [iva-req-write]` на ингредиенте iva-write — правильное место (значение течёт в registry gateway и там энфорсится), но требует, чтобы scope `iva-req-write` был выдан на phk_ пользователя в project-hub (per role, ADR-0058) и попал в сид registry.json gateway.

---

## Конструкции мульти-ключевой схемы (2-3 варианта)

Напоминание: клиентский bearer-http доставляет только 1 ключ; write-PAT и Allure — серверные.

### Вариант A — «Один клиентский ключ + gateway-подпись актора» (по ADR-0058) — РЕКОМЕНДУЮ
`iva-write` mcp_server_spec: `transport: http`, `url: https://mcp.tacticum.ru/iva-write/mcp`, `auth_type: bearer`, `env_required: [TACTICUM_TOKEN]` (один phk_), `required_scopes: [iva-req-write]`, `allowed_tools: [confluence_create_page, confluence_update_page, jira_create_issue, jira_add_comment, jira_transition_issue]`.
- Актор = `X-Auth-User-Id` от gateway. Jira/Confluence write-PAT (`iva`) — серверный env инстанса. Allure — отдельный сервер helm-analyst (read). tactic = тот же phk_.
- **+**: 1:1 с задеплоенным gateway; масштабируется (ADR-0058); PAT не раздаётся; рендерер уже поддерживает; полный аудит через актора; изменений схемы НЕ надо; `required_scopes` получает первое реальное применение.
- **−**: не кладёт «4 ключа в манифест» (расходится с буквой Решения #2 пользователя); предпосылки Ф1 ADR-0058 (создать iva-write-инстанс + PAT провижнит админ Jira) не выполнены; нужен форк/обвязка mcp-atlassian под чтение X-Auth-User-Id.
- **Риск**: URL/инстанса iva-write физически ещё нет; scope `iva-req-write` надо завести в project-hub и в сид registry.json.

### Вариант B — «Несколько серверов, по одному ключу на каждый» (расширение существующего паттерна)
Лейн композит НЕСКОЛЬКО mcp_server_spec, у каждого свой одиночный `env_required` и свой `required_scopes`: `iva-write` (phk_, iva-req-write) + при нужде отдельный write-сервер под Allure-статусы (`iva-allure-write`) и т.п. Зеркалит `iva-go-backend-brownfield` (3 сервера, каждый `[TACTICUM_TOKEN]`).
- **+**: 0 изменений схемы; проверенный паттерн; чистое разделение scope по серверам; не нарушает single-owner (разные ingredient_id).
- **−**: всё равно один клиентский ключ на сервер (лимит рендерера); если серверу реально нужен ВТОРОЙ клиентский секрет — не решает.

### Вариант C — «Расширение схемы под настоящий клиентский мульти-ключ» (только если сервер ОБЯЗАН получить 2+ клиентских секрета)
Новое поле в metadata mcp_server_spec, напр. `credentials: [{env: TACTICUM_TOKEN, target: bearer}, {env: JIRA_PAT, target: "header:X-Jira-PAT"}]`; научить ВСЕ 6 рендереров + `domain/renderer.py` эмитить несколько header/env; `auth_type` остаётся грубым типом.
- **+**: реально доставляет N клиентских ключей с разными назначениями.
- **−**: трогает публичную схему `ingredient.v1` + 6 рендереров + тесты; по ADR-0058 НЕ нужен (креды серверные); MCP-клиенты для remote-http обычно поддерживают лишь один Authorization — доп. секреты пойдут кастомными заголовками, которые gateway/backend должны интерпретировать; противоречит минимализму. Дорого и, вероятно, избыточно.

## Рекомендация
**Вариант A** (+ Вариант B, если Allure-write станет отдельным write-сервером). **Вариант C отклонить**, пока нет жёсткого требования «2+ клиентских секрета на ОДНОМ сервере» — и задеплоенная архитектура, и рендерер указывают прочь от него.

**Флаг тимлиду (важно для дизайна):** премиса задачи «4 ключа на адрес» и Решение #2 пользователя («hub + Jira/Confluence PAT + Allure-PAT + tactic на манифесте») расходятся с кодовой реальностью: write-PAT и Allure по архитектуре — **серверные** (ADR-0058, helm `.env`), а bearer-http рендерер физически доставляет клиенту лишь ОДИН ключ. Реально «мульти-ключевость» iva-write = **1 клиентский phk_ (identity+actor+scope) + downstream-креды на сервере**. Нужно решение: принимаем эту (уже существующую) 2-местную модель (Вариант A) или настаиваем на клиентском мульти-ключе (Вариант C, с расширением схемы и 6 рендереров).