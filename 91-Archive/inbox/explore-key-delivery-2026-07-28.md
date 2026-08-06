---
title: Доставка персонального Atlassian-ключа до MCP-сервера — разведка по коду
type: note
status: draft
role: explorer
tags:
- explorer
- iva-write
- keys
permalink: tacticum/00-board/explore-key-delivery-2026-07-28-1
created: 2026-07-28 11:39
updated: 2026-07-28 11:39
repo: tacticum-dev
project: iva-write
archived-at: 2026-08-05 15:19
---

# Доставка ключа: что есть в коде на 2026-07-28

Репозитории: `/Users/bubblemac/tacticum/tacticum-dev` (HEAD `26f5301`),
`/Users/bubblemac/tacticum/platform` (HEAD `c8648a5`). Только чтение, значения секретов не выводились.
Репозиторий `helm` НЕ смотрел — вне заданного скоупа (его ведёт отдельная разведка).

## 1. Как секрет доезжает до конфига сегодня: доезжает ССЫЛКА, не значение

Полная цепочка `env_required` → файл на машине человека:

1. **Манифест** объявляет только ИМЕНА переменных:
   - `templates/iva-analysis-base/manifest.yaml:391` — `env_required: [JIRA_URL, JIRA_PERSONAL_TOKEN, CONFLUENCE_URL, CONFLUENCE_PERSONAL_TOKEN]` (ингредиент `iva-atlassian-write`, `templates/iva-analysis-base/manifest.yaml:381`, `transport: stdio`, `command: uvx`, `args: [mcp-atlassian]`).
   - Тот же ингредиент — ещё в четырёх манифестах (`grep -rl "ingredient_id: iva-atlassian-write" templates/` → 5 файлов): `templates/iva-fr-analyst/manifest.yaml:187` (`env_required` :197 — те же четыре), `templates/iva-architect-mcp/manifest.yaml:85` (:95 — те же четыре), `templates/iva-techwriter-mcp/manifest.yaml:81` (:91 — те же четыре, порядок иной), `templates/iva-qa-mcp/manifest.yaml:101` — **у QA только две**: `env_required: [JIRA_URL, JIRA_PERSONAL_TOKEN]` (:111), Confluence-канал у QA не поднимается намеренно (`templates/iva-role-qa/README.md:67`).
2. **Схема ингредиента физически не имеет поля под значение**:
   `apps/backend/src/backend/catalog/domain/ingredients/mcp_server_spec.py:14-24` — `class McpServerMetadata(BaseModel)` с `model_config = ConfigDict(extra="forbid")`; поля `transport/command/url/args/env_required/auth_type/allowed_tools/required_scopes`. Поля `env` (значения) нет, и extra запрещены → положить значение в профиль сегодня невозможно без изменения схемы.
3. **Рендерер превращает имя в плейсхолдер**:
   `apps/backend/src/backend/catalog/domain/renderer.py:91-92`
   ```python
   elif env_required:
       value["env"] = {k: f"${{env:{k}}}" for k in env_required}
   ```
   Для remote+bearer — то же самое, только в заголовок: `renderer.py:89-90` → `value["headers"] = {"Authorization": f"Bearer ${{{env_required[0]}}}"}`.
   Канонические рендереры зеркалят: `infrastructure/renderers/claude_code.py:119-122`, `codex.py:152-160`, `copilot.py:147-148`, `gemini.py:90-91`, `opencode.py:83-84`.
4. **Действие клиенту** — merge_json в `.mcp.json`: `renderer.py:166-175` (`{"action": "merge_json", "path": ".mcp.json", "json_pointer": f"/mcpServers/{ingredient_id}", "value": _mcp_server_value(meta)}`).
5. **MCP-тулы установки** значения не добавляют: `catalog/interface/mcp/tacticum_init.py` — единственное упоминание env во всём файле это докстрока (`tacticum_init.py:49`); в `tacticum_init_manifest.py` слова `env` нет вовсе (grep пуст); `workspace/interface/mcp/pull_installation_content.py` — ни `env`, ни `secret`, ни `token` (grep пуст, есть только `def pull_installation_content` на строке 46).

**Вывод по п.1:** сервер значения PAT не знает и знать не может — оно живёт только в переменных окружения ОС на машине человека, поставленных руками (`docs/user_manuals/iva-fr-analyst-profile-quickstart.md:60-106`). Как именно CLI разворачивает `${env:VAR}` — не проверял (вне репозиториев).

## 2. Прецедент «платформа отдаёт клиенту сам секрет»: есть, но одноразовый и без хранения

- **Рождение `phk_*`**: `identity/interface/admin/memberships.py:302-308` (`_KEY_PREFIX = "phk_"`, `secrets.token_hex(32)`), выдача — `memberships.py:311-373`, в ответе `"key": raw_key` (`memberships.py:370`). Докстрока прямо: «Returns the plaintext key **once** — it is not stored» (`memberships.py:322`).
- **Хранится только хеш**: `identity/infrastructure/models.py:318-341` — `class MembershipApiKey`, `key_hash: LargeBinary unique`, `key_prefix: String(32)`; докстрока «Only the sha256 hash is stored; plaintext is shown to the admin once on creation and never again» (`models.py:321-322`). Повторной выдачи нет — эндпоинта чтения ключа не существует.
- **Проверка ключа** — только hash-lookup: `identity/infrastructure/membership_key_db.py:33-62` (`hash_token(raw_token)` → `select(MembershipApiKey).where(key_hash == h)`), с throttled `last_used_at`.
- **Аналогичные точки выдачи `tac_*`**: `identity/interface/api_router.py:151-156` и `platform/admin/router.py:355-360` — тоже `{"token": raw, ...}` один раз при выпуске.
- **Хранилища секретов «в открытом/расшифровываемом виде» в бэкенде НЕТ**: grep по `fernet|encrypt|cryptography|keyring|vault` в `apps/backend/src/backend` даёт только `SecretStr` в конфиге сервиса (`platform/config.py:6,47,62,74,77,88`) — то есть собственные креды сервиса из env, не чужие. В `platform/services` — ни одного совпадения.

**Вывод по п.2:** прецедент «отдать секрет в ответе API» есть (выпуск ключа), но прецедента «хранить чужой секрет и отдавать его повторно» нет ни одного. Централизованное хранилище PAT — новая сущность для этого кода: понадобится обратимое хранение (шифрование), которого сейчас в кодовой базе нет вообще.

## 3. Альтернатива «ключ не покидает сервер»: половина механики уже есть, недостаёт ровно одного звена

Что готово:

- **Гейтвей и цепочка middleware**: `platform/services/runtime/mcp_runtime/deploy/traefik-mcp-gateway.example.yml:45-50` — роутер `mcp-iva-read` на `mcp.tacticum.ru/iva-read` с цепочкой `["mcp-forward-auth", "mcp-iva-read-ratelimit", "mcp-drop-auth", "mcp-strip-iva-read"]`.
- **Снятие клиентского Bearer**: `traefik-mcp-gateway.example.yml:18-19` — `mcp-drop-auth: { headers: { customRequestHeaders: { Authorization: "" } } }`, комментарий: «после forward-auth снимаем Bearer: backend mcp-atlassian ходит по сервисному PAT из env». Upstream — `svc-iva-read: http://172.20.0.1:19010` (`:54`), тот же read-only mcp-atlassian на ADP через обратный SSH-туннель (`:43-44`).
- **forward-auth и инъекция идентичности**: `platform/.../application/forward_auth.py:42-51` — при allow отдаёт заголовки `X-Auth-User-Id`, `X-Tenant-Id`, `X-Project-Id`, `X-Scopes`; Traefik копирует их наверх (`authResponseHeaders`, yml:13).
- **⭐ Идентичность уже = E-MAIL человека**: `platform/.../infrastructure/dev_catalog.py:105-110` — `Identity(user_id=resolved.email, tenant_id=resolved.org_id, ...)`, то есть в `X-Auth-User-Id` уезжает e-mail. Подтверждено тестом `platform/services/runtime/mcp_runtime/tests/test_dev_catalog.py:49` (`X-Auth-User-Id == "dev@iva.ru"`).
- **Откуда e-mail берётся**: `tacticum-dev/apps/backend/src/backend/identity/interface/admin/resolve_key.py` — `POST /admin/identity/resolve-key`, тело `{key: phk_…}`, ответ `ResolveKeyResponse(email, org_id, active, key_id)`. Двухслойная auth: гейтвей аутентифицируется admin-токеном, сырой ключ едет только в теле — клиент `platform/.../infrastructure/dev_catalog.py:52-70`, серверная сторона `resolve_key.py:41-45` (`Depends(require_admin_either)`). Резолв кешируется TTL 300 c (`dev_catalog.py:81-88, 111`).
- **Прецедент downstream-сервиса, доверяющего заголовкам гейтвея**: `platform/services/memory/src/memory_service/gateway.py:22-37` — `identity_from_headers`, fail-closed без `X-Tenant-Id`/`X-Auth-User-Id`.

Чего НЕТ:

- **Механизма «по идентичности человека выбрать downstream-кред» не существует.** `RegistryEntry` (`platform/.../domain/models.py:30-40`) содержит `name/version/endpoint/transport/tool_surface/required_scopes/auth_mode/visibility/tenant_id` — поля под downstream-креды нет. Во всём `mcp_runtime/src` `Authorization` ставится только для собственных исходящих вызовов сервиса: к каталогу (`dev_catalog.py:62`), к хабу (`project_hub.py:75`), к телеметрии (`telemetry.py:47`). Downstream-кред сегодня — ОДИН на инстанс, лежит в env инстанса mcp-atlassian на ADP (комментарий `traefik-mcp-gateway.example.yml:18`, самого env-файла в репозиториях нет — не проверял).
- Целевая модель это подтверждает: ADR-0058 §5 (`tacticum-dev/docs/adr/0058-...md:53-65`) — `iva-write` = инстанс mcp-atlassian за гейтвеем на **PAT технической учётки `iva`**, «PAT выдаёт админ Jira, хранится в env сервиса» (`:65`), а персонализация делается не кредом, а **подписью актора**: «каждое действие обязано нести актора: сервер требует контекст (email из hub-ключа, ADR-0051) и дописывает его в артефакт» (`:59-62`).

**Вывод по п.3:** маршрут, снятие клиентского Bearer, резолв ключа в e-mail и инъекция e-mail в заголовок — уже работают в проде на `iva-read`. Строить придётся ровно одно: подстановку downstream-кред**а** по `X-Auth-User-Id` (email) внутри инстанса mcp-atlassian или в обвязке перед ним. Сегодня это место жёстко зашито на один сервисный PAT.

## 4. Умеет ли mcp-atlassian брать токен per-request

Пакет внешний (`uvx mcp-atlassian`, sooperset), исходников в наших репозиториях нет — сам пакет не оцениваю.
Что видно по нашим следам:

- Все наши конфигурации — процессные env: `JIRA_URL/JIRA_PERSONAL_TOKEN/CONFLUENCE_URL/CONFLUENCE_PERSONAL_TOKEN` (манифесты п.1), то есть один кред на процесс сервера.
- Прямое указание, что per-actor из коробки НЕТ: `docs/adr/0058-...md:152` — «⚠️ Подпись актора требует доработки mcp-atlassian (форк/обвязка) — оценить в Ф1».
- ADR-0060 (`docs/adr/0060-...md:74`) фиксирует то же направление: `iva-atlassian-write` привести к модели `iva-write` (техучётка + подпись + scope), «не личный PAT».
- Следов multi-user/OAuth/per-request-token режима в наших манифестах, доках и шаблонах не нашёл (grep по `multi-user|per-request|OAuth` в `docs/` и `templates/` вокруг mcp-atlassian — пусто).

**Вывод по п.4:** по нашим репозиториям per-request-кред не используется и не описан; наша же ADR планирует форк/обвязку для менее сложной задачи (подпись актора). Умеет ли upstream-пакет что-то ещё — не проверял.

## 5. Что сегодня человек настраивает руками и что сломается

Ручной контур (по `docs/user_manuals/iva-fr-analyst-profile-quickstart.md`):

- **Пять переменных окружения**, ставятся постоянно и до старта CLI: `TACTICUM_TOKEN` (phk) + `JIRA_URL`, `JIRA_PERSONAL_TOKEN`, `CONFLUENCE_URL`, `CONFLUENCE_PERSONAL_TOKEN` (`:60-64`; PowerShell-блок `:71-78`, bash/zsh-блоки `:84-101`; проверка `:106`).
- **Личные PAT человек выпускает сам** в двух системах: `jira.iva.ru` → Профиль → Personal Access Tokens → Create, то же на `wiki.iva.ru` (`:35`).
- **MCP-блоки в конфиге CLI человек правит руками** (шаг A.0 для Codex, `:119-135`), профиль доносит только `iva-atlassian-write` (`:119`).
- **Диагностика 401** сегодня формулируется как «перевыпусти PAT» (`:348`).

Что централизация заменит и что придётся переnastроить:

- **Пять переменных → одна.** При гейтвейном варианте у человека остаётся `TACTICUM_TOKEN`, четыре PAT-переменные исчезают, а `iva-atlassian-write` из stdio-`uvx` превращается в http-спеку по образцу `iva-read` (`templates/iva-analysis-base/manifest.yaml:341-351`: `transport: http`, `url: https://mcp.tacticum.ru/iva-read/mcp`, `env_required: [TACTICUM_TOKEN]`, `auth_type: bearer`).
- **Переиздать придётся 5 манифестов** с ингредиентом (`iva-analysis-base`, `iva-fr-analyst`, `iva-architect-mcp`, `iva-qa-mcp`, `iva-techwriter-mcp`) — установленные у людей профили это НЕ обновляет само; человеку нужно переприменить профиль и перезапустить CLI.
- **Тексты про личные PAT разъедутся с реальностью в ~24 файлах**: `grep -rl JIRA_PERSONAL_TOKEN docs/ templates/` даёт 24 файла, в том числе скилл, который агент читает в рантайме — `templates/iva-analysis-base/ingredients/skills/fr-authoring/SKILL.md:24-27` (предусловие: «локальный mcp-atlassian на ЛИЧНЫХ Atlassian PAT аналитика»), закомментированные шаблоны конфигов `templates/iva-role-analyst/ingredients/repo-configs/codex/config.toml.template:22-25`, `templates/iva-role-qa/ingredients/repo-configs/{claude-code/CLAUDE.md.fragment:30, codex/AGENTS.md.fragment:35}`, README/CHANGELOG ролей.
- **Что у живой команды не сломается**: имена тулов при переезде на гейтвей сохраняются (заявлено в `templates/iva-analysis-base/manifest.yaml:379-381` и `docs/adr/0060-...md:74`), значит скиллы и промпты не трогаются — ломается только транспорт и настройка окружения.
- **Риск атрибуции**: при переходе на техучётку записи в Jira/Confluence перестают идти от личной УЗ человека и начинают идти от `iva` с подписью актора в теле (ADR-0058 `:59-62`). Для тех, кто «разобрался, работает» на личном PAT, это смена и автора правки, и прав: права УЗ сужены до IVAREQ + новый space (`:57-58`), то есть публикация в СТАРЫЕ спейсы, куда аналитики пишут сейчас, техучётке физически недоступна. Это самый серьёзный ломающий фактор из найденных по коду/докам.
- **Централизованная выдача PAT за человека** (вариант «система сама генерит токен») по коду не имеет опоры: API выпуска Atlassian-PAT от имени пользователя в наших репозиториях нет, и хранилища для обратимого хранения чужих секретов тоже нет (см. п.2). Не проверял: умеет ли Jira/Confluence Server/DC выпускать PAT за пользователя админским API — вне репозиториев.

## Итог: технически возможные пути

| Путь | Что уже есть | Что строить |
|---|---|---|
| **A. Ключ доезжает до машины человека через профиль** | ничего: схема запрещает значения (`mcp_server_spec.py:15`), рендерер пишет плейсхолдер (`renderer.py:92`) | поле значений в схеме + доставка секрета в `.mcp.json`/env, хранилище с обратимым шифрованием, ротация. Секрет оказывается на диске у каждого |
| **B. Ключ отдаётся один раз по API, человек кладёт в env сам** | прецедент выдачи «раз и навсегда» (`memberships.py:322,370`) | эндпоинт выдачи + хранилище; ручной шаг у человека остаётся — экономия только на походе в Jira за PAT |
| **C. Ключ не покидает сервер, гейтвей подставляет по e-mail** | маршрут, `mcp-drop-auth` (yml:19), forward-auth с `X-Auth-User-Id` = e-mail (`dev_catalog.py:105-110`), резолв `phk_`→email (`resolve_key.py`), прецедент header-trust (`memory_service/gateway.py:22-37`) | выбор downstream-креда по идентичности — этого нет нигде (`domain/models.py:30-40`); плюс хранилище PAT и обвязка/форк mcp-atlassian (ADR-0058:152) |
| **D. Техучётка + подпись актора (текущий план ADR-0058)** | тот же гейтвейный контур, что в C | обвязка подписи (ADR-0058:152), настройка прав УЗ админом Jira (ADR-0058:148); персональные ключи не нужны вовсе, но теряется авторство человека и доступ к старым спейсам |