---
title: ШАГ 0 — TACTICUM_TOKEN, один ключ или два
type: note
status: draft
tags:
- board
- iva-write
- tacticum-token
- explore
permalink: tacticum/00-board/step0-tacticum-token-2026-08-06
---

# ШАГ 0: TACTICUM_TOKEN — переменная одна, хранилищ ключей ДВА

**Короткий ответ.** Переменная одна (`TACTICUM_TOKEN` — и у каталожного
`mcp.tacticum.dev/mcp`, и у канала записи `helm.tacticum.ru/mcp/iva-write`), но
формат `phk_…` носят **два независимых хранилища ключей**: каталог `tacticum-dev`
(таблица `membership_api_keys`) и `project-hub` (таблица `api_keys`). helm
принимает ключи ОБОИХ. Каталог принимает **только свои**. Бот выпускает ключ
**в project-hub** — значит ключ от бота канал записи откроет, а каталожные
инструменты сломает. Ловушка реальна и она односторонняя.

## 1. Чем аутентифицируется каталожный MCP (`mcp.tacticum.dev/mcp`)

Заголовок `Authorization: Bearer <token>`, разбор — `scope_middleware`:
`/Users/bubblemac/tacticum/tacticum-dev/apps/backend/src/backend/identity/interface/middleware.py:44-84`.

- `phk_*` → `resolve_membership_scope_from_db` (sha256 по `membership_api_keys`
  каталожной БД): `.../identity/infrastructure/membership_key_db.py:33-63`.
  Не нашли / отозван → 401 `invalid_token` (`middleware.py:58-65`).
- **иначе** (в т.ч. `tac_live_*`) → legacy-путь `resolve_scope` →
  `resolve_scope_from_db` (sha256 по `api_tokens`):
  `.../identity/application/scope_resolver.py:49-66`,
  `.../identity/infrastructure/token_db.py:28-60`. Проверки префикса нет — годится
  любой `tac_*`, у которого есть живая строка. **Да, `tac_live_*` принимается.**
- **Без токена — нет.** `middleware.py:49-55`: пустой Bearer → 401 `missing_token`,
  и только при `settings.tacticum_dev_mode` запрос пропускается. На проде
  проверено: `TACTICUM_DEV_MODE=false` (`docker inspect tacticum-catalog-mcp-1`,
  сервер `tacticum_prod`, 06.08).
- **`installation_id` из `.tacticum/context.yaml` — НЕ аутентификатор.** Он только
  снимает неоднозначность «какая из установок» у org-scoped ключа
  (`scope_resolver.py:90-180`); без уже установленного scope — `AuthError
  missing_scope`.

**Важное следствие: `phk_`-ключ каталога — org-scoped, не person-scoped.**
`MembershipScope` = (organization_id, user_id, key_id), а доступ к установке
проверяется по совпадению ОРГАНИЗАЦИИ (`membership_installation_scope.build_authscope_from_membership`).
Любой живой `phk_` любого сотрудника орг-а «iva» открывает все её установки.

## 2. Чем аутентифицируется наш канал записи

`require_write_identity` — helm, `origin/main`:
`src/helm/interface/api/routers/iva_write.py:92-152` (в локальном чекауте
`/Users/bubblemac/tacticum/helm` файла НЕТ: ветка `main` там отстала, читать через
`git show origin/main:…`; свежий рабочий чекаут — `/Users/bubblemac/tacticum-worktrees/helm-observability`).
Тот же `require_write_identity` использует и MCP-поверхность канала:
`src/helm/interface/mcp/iva_write_surface.py:73` (импорт), решение 2 в докстринге.

Порядок:
1. Нет `HELM_RESOLVE_URL`/`HELM_RESOLVE_SERVICE_KEY` → 503.
2. Нет `Authorization: Bearer` → 401 (флаг `auth_required` здесь НЕ действует).
3. `resolve_token` → project-hub `POST /api/internal/resolve`
   (`infrastructure/auth/resolve_client.py:43-57`; любой не-200 → `ResolveError`).
   Хаб проверяет свой argon2-хеш: `project_hub/auth/api_keys.py:55+`, ручка —
   `project_hub/api/internal.py:281-311`.
4. **Второй шанс** — `resolve_dev_token` (`infrastructure/auth/dev_token.py:84-114`):
   зовёт тул `whoami` каталожного MCP под токеном вызывающего, берёт
   `caller.user_email`, принимает ТОЛЬКО `auth_method == "phk"`
   (`dev_token.py:71-81`). Включается только при заданном `dev_mcp_url`.
5. Домен почты обязан быть в `dev_token_email_domain_set`, дальше
   `principal_for_tenant` с тенантом `iva`; иначе 401/403.

На проде helm (сервер `helm`, контейнер `helm-helm-1`) второй источник **включён**:
`HELM_DEV_MCP_URL=https://mcp.tacticum.dev/mcp`,
`HELM_DEV_TOKEN_EMAIL_DOMAINS=iva.ru,tacticum.ru,iva-te…`.

Каталожная сторона `whoami`: `apps/backend/src/backend/identity/interface/mcp/whoami.py:32-96`
(`auth_method` = `phk` для membership-ключа, `tac` для legacy; `user_email` только у `phk`).

## 3. Один ключ или два — ГЛАВНОЕ

| Ключ выпущен | каталог `mcp.tacticum.dev` | канал `helm.tacticum.ru/mcp/iva-write` |
|---|---|---|
| каталогом (`membership_api_keys`, админка `dev.tacticum.dev/admin`) | ✅ | ✅ через `dev_token` fallback |
| project-hub (`api_keys`, в т.ч. **выпуск ботом**) | ❌ 401 `invalid_token` | ✅ штатный `/resolve` |
| legacy `tac_live_*` | ✅ | ❌ (`whoami` даёт `auth_method: tac`, `user_email: None` → `dev_token.py:76-77` отвергает) |

Оба хранилища независимо чеканят один и тот же формат `phk_<32 hex>`:
- хаб — `project_hub/auth/api_keys.py:26,45-48` (argon2id, своя таблица `api_keys`);
- каталог — `membership_api_keys` (sha256), `membership_key_db.py:44-46`.

**Мостика между ними нет.** `project_hub/adapters/catalog.py` умеет заводить
пользователя, раздавать роли и **отзывать** каталожные `phk_`, но НЕ выпускать:
докстринг `catalog.py:1-16` + `activate_user` (`catalog.py:160-163`) — «no-op so hub
re-enable never silently mints a key». Ручка выпуска хаба пишет только к себе:
`project_hub/api/admin.py:812-838`.

**Про Острякова: противоречия нет, `key_unknown` — это не отказ в аутентификации.**
`KEY_UNKNOWN` (helm, `src/helm/application/tacticum_key.py:70`, решение —
`tacticum_key_status`, там же строка 104) означает «не знаем, есть ли у человека
ключ»: личность пришла НЕ через нашу аутентификацию (у бота это `platform`/`declared`),
а проверка в каталоге выключена или не ответила. helm ничей ключ в 06:59:47 не
отвергал — ему нечего было отвергать, к боту человек пришёл сообщением, а не Bearer'ом.
Отдельно от `KEY_ABSENT` («спросили, ключей нет») намеренно.

А каталожные инструменты у него работали потому, что каталожный `phk_` — org-scoped:
любой живой ключ орг-а «iva» открывает все её 24 установки, и кто именно им ходил,
по коду каталога неотличимо.

**Чего в логах каталога НЕТ и чем это добрать.** structlog каталога
(`/opt/tacticum/logs/catalog-mcp.log` на `tacticum_prod`) вызывающего не называет:
в строках нет ни почты, ни префикса ключа. За 06.08 там 410 `whoami` — **все**
`auth_method: "phk"`, `installation_count: 24`; ни одного `tac`. `design_list_systems`
в 07:37:55 есть; `kb_discover` — в 06:44/06:45/07:22/07:24/07:33/07:45; в 06:29:50
стоит `tacticum_init_called` (профиль `iva-web-brownfield`, codex). Событий
`check_updates` нет ни одного, но это ничего не доказывает: `check_updates.py`
структурного лога не пишет вовсе.

Привязка «кто ходил» есть только в БД каталога — `telemetry_events.membership_id`
(`apps/backend/src/backend/platform/telemetry_models.py:37-40`, пишется в
`platform/usage_telemetry.py:67-71`). SELECT у меня нет; запрос для лида:

```sql
SELECT te.occurred_at, te.tool_name, te.installation_id,
       mak.key_prefix, mak.created_at AS key_created, mak.revoked_at, u.email
FROM telemetry_events te
LEFT JOIN membership_api_keys mak ON mak.id = te.membership_id
LEFT JOIN users u ON u.id = mak.user_id
WHERE te.event_type = 'tool_call'
  AND te.occurred_at >= '2026-08-06 06:00:00+00'
  AND te.occurred_at <  '2026-08-06 08:00:00+00'
ORDER BY te.occurred_at;
```
`membership_id IS NULL` при непустом `installation_id` = ходили legacy `tac_*`.

## 4. Что будет, если человек перезапишет TACTICUM_TOKEN

- **Новый ключ выпущен КАТАЛОГОМ, та же организация** → каталожные инструменты
  работают как работали (проверка org-level), канал записи тоже. Ничего не ломается.
- **Новый ключ выпущен ботом (project-hub)** → канал записи заработает, а
  **каталожные инструменты немедленно отдадут 401 `invalid_token`**
  (`middleware.py:58-65`): в `membership_api_keys` такого ключа нет и не появится.
  Это ровно тот случай, который бот и создаёт.
- **Раньше в переменной лежал `tac_live_*`** → замена на любой `phk_` дополнительно
  меняет поведение каталога: `tac_` пинил одну установку, `phk_` покрывает много и
  требует явного `installation_id` (`scope_resolver.py:127-152`) — его даёт
  `.tacticum/context.yaml`; репозиторий без привязки словит `installation_id_required`.
  На практике сегодня `tac_` в проде не ходит никто (0 из 410 `whoami` за сутки).

## 5. Откуда человек штатно берёт TACTICUM_TOKEN

**Самообслуживания нет ни там, ни там.**

- Каталожный ключ: только админка `https://dev.tacticum.dev/admin` → организация →
  участник → API Keys (`apps/admin_web/src/app/(authed)/orgs/[id]/members/[userId]/api-keys/page.tsx`,
  API — `identity/interface/admin/memberships.py:312`). Это и есть путь из инструкций:
  `docs/agents/codex-init.md:51`, `docs/tacticum-mcp-install.md:37`. Пользовательской
  страницы выпуска в `apps/web` нет.
- Хабовский ключ: веб-админка project-hub (`web/routes.py:1691,1837`, скоупы
  `mcp`/`read`) либо админская API-ручка. Бот ходит именно туда:
  `helm/interface/api/key_issue.py:220` → `hub.issue_key_for` (scope по умолчанию
  `mcp`, `helm/config.py:767`).

**Путь бота и путь каталога — разные хранилища.** Совпадают только префикс и имя
переменной.

## Что писать в боте

Формулировка (не «положи ключ в переменную» и не «осторожно, там может быть другой»):

> Ключ выдан. Он открывает **канал записи** в Jira/Confluence.
> Если у тебя уже настроены инструменты Tacticum (`kb_*`, `design_*`) —
> **не заменяй ими существующий `TACTICUM_TOKEN`**: они ходят по другому ключу,
> и подмена их сломает. Пропиши новый ключ отдельной переменной для канала записи
> либо напиши нам — выдадим один ключ на всё.

Прямая строка для доски: **пока бот выпускает ключ в project-hub, обещание
quickstart «отдельный ключ не нужен» (`docs/user_manuals/iva-write-base-profile-quickstart.md:38-40`)
для получателей бота неверно.** Чинится либо выпуском ключа в каталоге, либо
отдельной переменной у ингредиента `helm-iva-write`
(`templates/iva-write-base/manifest.yaml:122-132`, сейчас `env_required: [TACTICUM_TOKEN]`,
как и у каталожного `tacticum-mcp` — `templates/tacticum-core-base/manifest.yaml:135-145`).
Выбор — за лидом, это не разведка.

## Замечания по постановке

- Лейн называется **`iva-write-base`**, ингредиент — `helm-iva-write`; прежний шлюз
  `mcp.tacticum.ru/iva-write/mcp` удалён.
- Локальный чекаут `/Users/bubblemac/tacticum/tacticum-dev` отстал от `origin/main`
  на 188 коммитов, а `tests/catalog/test_role_install_smoke.py` в нём изменён
  (чужая незакоммиченная правка). Всё, что выше, читалось из `origin/main`.
