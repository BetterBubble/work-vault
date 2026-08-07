---
title: 'Разведка: как MCP iva-write доезжает до человека и откуда берётся ключ Tacticum'
type: note
status: draft
tags:
- board
- iva-write
- explorer
- profiles
repo:
- tacticum-dev
- helm
project: iva-write
permalink: tacticum/00-board/explore-profile-wiring-2026-07-30-1
archived-at: 2026-08-07 11:22
---

# Разведка: обвязка профиля под `iva-write` (2026-07-30)

Только чтение. Репозитории: `/Users/bubblemac/tacticum/tacticum-dev` (main, отстаёт на 105
коммитов от origin), `/Users/bubblemac/tacticum-worktrees/helm-iva-write-keystore`
(ветка `feat/iva-write-keystore`, HEAD `baa7895`) — код `iva_write` живёт ТОЛЬКО там,
в `/Users/bubblemac/tacticum/helm` (main) его нет.

## 1. Как MCP-сервер объявляется в профиле

Объявление — ингредиент `kind: mcp_server_spec` в `templates/<profile>/manifest.yaml`
каталога tacticum-dev. Все helm-серверы объявлены **одинаково: http + bearer**.

| Профиль/лейн | Ингредиент | Транспорт / URL | Файл:строка |
|---|---|---|---|
| `iva-analysis-base` | `helm-analyst` | http `https://helm.tacticum.ru/mcp/analyst` | `templates/iva-analysis-base/manifest.yaml:329-341` |
| `iva-analysis-base` | `iva-read` | http `https://mcp.tacticum.ru/iva-read/mcp` | `templates/iva-analysis-base/manifest.yaml:343-353` |
| `iva-analysis-base` | `helm-process` | http `https://helm.tacticum.ru/mcp/process/` (⚠️ trailing slash) | `templates/iva-analysis-base/manifest.yaml:355-379` |
| `iva-analysis-base` | `iva-atlassian-write` | **stdio** `uvx mcp-atlassian` | `templates/iva-analysis-base/manifest.yaml:381-390` |
| `iva-fr-analyst` | `helm-analyst` / `iva-mcp` / `iva-atlassian-write` | http / http / stdio | `templates/iva-fr-analyst/manifest.yaml:150,166,187` |
| `iva-architect-mcp` | `iva-atlassian-write` (скоуп по роли) | stdio | `templates/iva-architect-mcp/manifest.yaml:85-102` |
| `iva-qa-mcp` | `iva-atlassian-write` + `helm-analyst` | stdio / http | `templates/iva-qa-mcp/manifest.yaml:101`, `:130` |
| `iva-techwriter-mcp` | `iva-atlassian-write` | stdio | `templates/iva-techwriter-mcp/manifest.yaml:81` |
| `tacticum-dev-base` | `helm-analyst` | http | `templates/tacticum-dev-base/manifest.yaml:296` |
| `iva-go-backend-brownfield`, `iva-kmp-brownfield`, `iva-rn-brownfield`, `iva-web-brownfield`, `iva-brownfield-mail` | `helm-analyst` | http | `manifest.yaml:447 / :592 / :570 / :613 / :482` соответственно |

Форма http-объявления с авторизацией (эталон, `templates/iva-analysis-base/manifest.yaml:329`):

```yaml
  - ingredient_id: helm-analyst
    kind: mcp_server_spec
    tier: trial
    supports: [claude-code, codex]
    install_scope: repo
    body: ""
    metadata:
      transport: http
      url: "https://helm.tacticum.ru/mcp/analyst"
      env_required: [TACTICUM_TOKEN]
      auth_type: bearer
```

Манифесты попадают в БД сидером `apps/backend/scripts/seed_community.py:136`
(`templates/*/manifest.yaml` → upsert).

## 2. Откуда берётся токен при установке профиля

**Ответ: (в) + (а) — плейсхолдер в конфиге, значение вводит человек руками в env,
причём ДО установки профиля.** Установщик ключ не знает и нигде его не хранит.

Механизм — рендер `env_required` в плейсхолдер:

- `apps/backend/src/backend/catalog/infrastructure/renderers/claude_code.py:117-122` —
  remote-сервер (`url`, без `command`): при `auth_type == bearer` и непустом
  `env_required` пишется `config["headers"] = {"Authorization": f"Bearer ${{{md.env_required[0]}}}"}`,
  то есть в `.mcp.json` уезжает буквально `Bearer ${TACTICUM_TOKEN}`.
- `apps/backend/src/backend/catalog/domain/renderer.py:86-92` — тот же расклад для
  живого MCP-пути (`tacticum_init` / `pull_installation_content`); stdio-сервера
  получают `env: {VAR: "${env:VAR}"}`.
- Codex-рендерер вместо заголовка пишет `bearer_token_env_var = "TACTICUM_TOKEN"`
  (`apps/backend/tests/catalog/infrastructure/renderers/test_codex_renderer.py:176`,
  эталон в `apps/backend/dev/e2e/install_scenario.py:140,151`).

**Раскрывает плейсхолдер сам CLI-клиент** (Claude Code / Codex) из переменных окружения
процесса. Ни backend, ни установщик подстановкой не занимаются.

Порядок для человека (`docs/user_manuals/iva-role-analyst-profile-quickstart.md`):
шаг **A.0/B.0 «подключить MCP (делает ПОЛЬЗОВАТЕЛЬ)»** — он руками правит
`.codex/config.toml` (`:64-81`) или зовёт `claude mcp add` (`:136-144`), затем
`export TACTICUM_TOKEN="phk_…"` (`:82`, `:148`) и перезапускает CLI. Только после этого
шаги A.1/B.1 (`whoami` → `installation_id`) и A.2/B.2 (применение профиля). То есть
MCP с ключом появляется РАНЬШЕ профиля, а не из профиля.

Оговорка из инструкции (`:146`): «401 при заданном env после перезапуска — впиши токен
литералом», и в `docs/user_manuals/tacticum-dev-base-profile-quickstart.md:116` прямо
показан вариант `http_headers = { "Authorization" = "Bearer phk_…" }` — секрет в файле.

## 3. Есть ли уже объявленный `iva-write` MCP в каталоге

**Нет. Это и есть недостающее звено.** Ни одного `mcp_server_spec` с `ingredient_id:
iva-write` и ни одного упоминания URL `/mcp/iva-write` в `templates/` не существует.

Более того, лейн под это когда-то был и **удалён**:

- `templates/iva-write-base/` создан коммитом `f69524d` — нёс ровно один ингредиент
  `iva-write`: `transport: http`, `url: "https://mcp.tacticum.ru/iva-write/mcp"`,
  `env_required: [TACTICUM_TOKEN]`, `auth_type: bearer`, `required_scopes: [iva-req-write]`,
  `allowed_tools: [confluence_create_page, confluence_update_page, jira_create_issue,
  jira_add_comment, jira_transition_issue]`.
- Снесён `git rm` в коммите `81fa9fd` («ретайр iva-write-base → own iva-atlassian-write»),
  см. `templates/iva-role-architect/CHANGELOG.md:31-36,51` и
  `templates/tacticum-role-techwriter/CHANGELOG.md:32-37,51`.

Важное расхождение адресов: удалённый лейн вёл на **gateway** `mcp.tacticum.ru/iva-write/mcp`,
а наша ветка поднимает поверхность в самом helm — `https://helm.tacticum.ru/mcp/iva-write`
(`src/helm/interface/mcp/iva_write_surface.py:68` `MOUNT_PATH`, монтаж
`src/helm/main.py:221`, работает и без завершающего слэша благодаря
`_McpMountTrailingSlash`, `src/helm/main.py:225-250`).

**Оценка объёма** (по образцу `helm-analyst`):

- Минимум, чтобы канал доехал до одной роли: **1 файл, ~14 строк** — блок ингредиента
  в `templates/iva-analysis-base/manifest.yaml` (или в per-role лейн) + бамп `version`.
- Паритет с текущим write-каналом (заменить/дополнить `iva-atlassian-write` во всех
  пяти носителях): **5 манифестов** (`iva-analysis-base:381`, `iva-fr-analyst:187`,
  `iva-architect-mcp:85`, `iva-qa-mcp:101`, `iva-techwriter-mcp:81`) ×~14 строк, плюс по
  конвенции репо README + CHANGELOG каждого шаблона (10 файлов, ~10-20 строк каждый),
  плюс тест `apps/backend/tests/catalog/test_iva_role_presets.py:229-237` (ожидаемый набор
  MCP роли architect). Golden-снапшоты `apps/backend/tests/e2e_install/golden/` MCP-конфиг
  не содержат — их это не задевает.
- Отдельно: квикстарты (`docs/user_manuals/*`) — там MCP-блок пользователь пишет руками,
  значит без правки инструкции сервер до человека НЕ доедет даже при объявлении в манифесте.

## 4. Как helm понимает, что запрос от конкретного человека

Путь (worktree `helm-iva-write-keystore`):

1. Запрос приходит на ASGI-поверхность `surface()`
   (`src/helm/interface/mcp/iva_write_surface.py:318`), она зовёт
   `require_write_identity(request)` (`:339`).
2. `require_write_identity` — `src/helm/interface/api/routers/iva_write.py:84-150`.
   Токен обязателен ВСЕГДА, `auth_required` игнорируется (`:84-96`). Из заголовка
   `Authorization: Bearer …` (`:104-111`).
3. Первый источник — project-hub: `resolve_token(token, resolve_url, service_key)`
   (`:115-119`) → POST на `resolve_url` с телом `{"token": …}` и service-ключом в
   заголовке (`src/helm/infrastructure/auth/resolve_client.py:47-57`). Возвращается
   `ResolvedUser(email, memberships, roles)` (`resolve_client.py:60-64`) — **почта, не user_id**.
4. Второй шанс, если hub токен не признал: `resolve_dev_token(token, mcp_url=settings.dev_mcp_url)`
   (`iva_write.py:121-136`) — зовёт тул `whoami` у dev-MCP и берёт `caller.user_email`,
   принимая только `auth_method == "phk"` (`src/helm/infrastructure/auth/dev_token.py:1-16`).
   Гейт по домену почты: `dev_token_email_domain_set` = `iva.ru,tacticum.ru`
   (`src/helm/config.py:659-670`). Членство синтезируется как `viewer`.
5. `principal_for_tenant(resolved.email, …, expected_tenant=settings.tenant or "iva")`
   (`iva_write.py:138-149`) → `Principal` с `.email` и `.tenant`. Нет членства → 403.
6. **`subject` = `normalize_subject(principal.email)`** — просто `strip().lower()`
   (`src/helm/infrastructure/db/credential_repo.py:41-43`). Поверхность передаёт
   `subject=principal.email` в `issue_credentials`
   (`iva_write_surface.py:259-267`), а там снова нормализует и ищет строку keystore:
   `get_credential_row(session, tenant, subject=key, system=…)`
   (`src/helm/application/iva_write.py:532-535`).

**Совпадает ли с почтой согласия.** `external_credential.subject` пишется тем же
`normalize_subject`, но источник почты зависит от того, каким путём человек давал согласие:

- через API `/api/iva/oauth/start` — почта из auth-принципала, тело её не принимает
  («иначе человек мог бы завести доступ на чужое имя»), `subject_source=SOURCE_AUTH`
  (`routers/iva_write.py:183-185, 248-259`). Здесь совпадение гарантировано.
- через бота — почту человек НАЗЫВАЕТ в чате, `subject_source=SOURCE_DECLARED`
  (`routers/bot_iva_write.py:192, 207-219`; разбор адреса `application/bot_iva_write.py:89`
  `extract_email`). Здесь совпадение **не гарантировано ничем**: если названный в чате
  адрес отличается от того, что project-hub/dev-MCP отдаёт по его Bearer (алиас,
  другой домен, личная vs корпоративная почта), запись в keystore ляжет на один subject,
  а поверхность будет искать по другому — и человек получит 403 `no_credentials`
  (`iva_write_surface.py:373-385`) при живом согласии. Сверка на колбэке
  (`application/iva_write.py:236-268`) проверяет ДРУГУЮ пару — согласие vs то, что вернул
  Atlassian `whoami`, — и этот разрыв не ловит.

**Риск №1 для приёмки: почта согласия из бота может не совпасть с почтой из токена.**

## 5. Где человек берёт свой ключ Tacticum

По коду helm про project-hub известно только это:

- `resolve_url` — приватный внутренний адрес, пример в комментарии
  `http://10.16.0.17:8770/api/internal/resolve` (`src/helm/config.py:61-63`,
  `src/helm/infrastructure/auth/resolve_client.py:8`). Не для людей.
- `oauth_issuer: str = "https://project.cifragen.ru"` (`src/helm/config.py:69`) — issuer
  OAuth2 PKCE. Публичная ручка `GET /api/auth/config`
  (`src/helm/interface/api/auth.py:164-176`) отдаёт SPA `authorize_url = {issuer}/authorize`,
  `token_url = {issuer}/token`, `client_id = "helm"`, `scope = "openid"`.
  То есть у project-hub есть веб-логин, но это вход в helm-дэш, **не выдача ключа**.

Про сам ключ:

- Пользовательская инструкция говорит «в чате поддержки»:
  `docs/user_manuals/iva-role-analyst-profile-quickstart.md:26` — phk-токен берётся
  «в [чате поддержки Tacticum](https://iva-uc.ru/chat/5e3c23f5-…)», один раз;
  то же в `docs/user_manuals/tacticum-dev-base-profile-quickstart.md:74`; более старая
  формулировка «выдаёт админ (Diaret) — приходит личным сообщением один раз»
  в `docs/agents/rn-profile-quickstart.md:10`.
- В коде выдача — **только админская ручка**:
  `POST /admin/memberships/{org_id}/{user_id}/api-keys`
  (`apps/backend/src/backend/identity/interface/admin/memberships.py:311-345`,
  `require_admin_either`), plaintext показывается ОДИН раз и не хранится (`:320-323`).
  UI — админский: `apps/admin_web/src/app/(authed)/orgs/[id]/members/[userId]/api-keys/page.tsx`.
  Процедура для админа — `docs/agents/installing-new-profile.md:64-72`.
- **Самообслуживания нет:** ручки «выпусти себе ключ» в коде не существует; страницы, где
  человек видит/создаёт свой ключ, тоже нет. Инструкция «где взять» — только квикстарты
  выше (иных источников в репозиториях не нашёл; если есть — это вики/Confluence, не код).

Отдельно: phk-ключ и hub-токен — **два разных типа**, helm принимает оба
(`config.py:651-657`; комментарий манифеста analysis-base — «личный TACTICUM_TOKEN, hub/phk»,
`templates/iva-analysis-base/manifest.yaml:358`).

## 6. Что происходит, если ключа Tacticum нет вообще

По коду — прямой отказ на входе, до всякой записи:

- Нет заголовка `Authorization: Bearer` → **401** «нужен bearer-токен», `WWW-Authenticate: Bearer`
  (`routers/iva_write.py:105-110`), поверхность оборачивает в JSON
  `{"error": "unauthorized", …}` (`iva_write_surface.py:340-348`).
- Токен есть, но ни hub, ни dev-MCP его не признали (либо почта не того домена) →
  **401** «токен не признан» (`routers/iva_write.py:127-134`).
- Токен признан, но нет членства в тенанте `iva` → **403** «нет доступа: требуется членство
  в тенанте «iva»» (`routers/iva_write.py:145-149`).
- Токен есть, согласия нет → **403** `no_credentials`, «нет личного доступа к Jira/Confluence:
  выдайте согласие (/api/iva/oauth/start) и повторите» (`iva_write_surface.py:373-385`);
  наверх, в контейнер `mcp-atlassian`, не уходит ничего (fail-closed).

Что скажет бот (`src/helm/application/tacticum_key.py`): решение о ключе принимается в
одном месте — `tacticum_key_status(subject, identity_source)` (`:39-52`): личность из auth
(`SOURCE_AUTH`) → `known_present` («ключ у него точно есть — он им только что
аутентифицировался»); личность названа в чате (`SOURCE_DECLARED`) → `unknown`. Текст даёт
`key_note(key_status)` (`:55-74`):

- всегда: «Работать нужно своим ключом Tacticum — тем же, которым вы уже пользуетесь в наших
  сервисах; ничего нового прописывать не нужно.»
- при `unknown` добавляется: «Если ключа Tacticum у вас ещё нет — сообщите об этом: без него
  доступ не заработает.»

Эта же фраза уходит и в чат (`application/bot_iva_write.py:206-207`), и на страницу колбэка
в браузере (`routers/iva_write.py:412-417`).

**Пути получить ключ, не выходя за пределы наших систем, НЕТ.** Это зафиксировано в коде
явно: helm ключи не выпускает и даже не умеет спросить project-hub «есть ли у человека ключ»
(`application/tacticum_key.py:14-24` — сейчас работает «вариант (а)», выпуск был бы обращением
наружу в project-hub и добавляется отдельной веткой в этот же модуль). Единственный живой путь —
человек пишет в поддержку/админу, тот выпускает phk руками админской ручкой (см. п. 5).

## Риски, которые видно из кода

1. **Расхождение subject** (см. п. 4): согласие через бота пишется на НАЗВАННУЮ почту,
   а канал ищет по почте ИЗ ТОКЕНА. Не совпали → 403 при живом согласии, причины снаружи
   неотличимы намеренно (`iva_write_surface.py:374-376`).
2. **Каталог не знает про канал** (п. 3): поверхность в helm есть, объявления в профиле нет —
   без него и без правки квикстартов сервер до человека не доедет.
3. **Разные адреса в истории**: снесённый лейн вёл на `mcp.tacticum.ru/iva-write/mcp`
   (gateway), новая поверхность живёт на `helm.tacticum.ru/mcp/iva-write` (helm напрямую).
   При объявлении легко скопировать старый URL и получить молчаливый 404.
4. **Ключ выдаётся вручную админом** (п. 5-6): любой новый человек в канале — это ручная
   операция вне систем, а инструкция ведёт в чат поддержки.