---
title: QA-профиль — онбординг команды ИВА (передача и запуск)
type: reference
permalink: tacticum/91-archive/qa/qa-profil-onbording-komandy-iva-peredacha-i-zapusk
status: archived
superseded-by: "[[iva-role-qa — установка и работа (для команды)]]"
created: 2026-07-23 14:27
updated: 2026-07-25 20:06
repo: tacticum-dev
project: tacticum-dev / профили (ADP) / QA-профиль
role: lead-qa
date: 2026-07-23
tags:
- qa
- onboarding
- runbook
- handoff
- iva-role-qa
- provisioning
- lead-qa
---

# QA-профиль `iva-role-qa` — передача QA-команде и запуск

> **АРХИВ (2026-07-25).** Это рабочий план передачи (наша сторона + сторона QA), первая версия от 2026-07-23. Содержательное слито в актуальный документ [[iva-role-qa — установка и работа (для команды)]] — он единственный актуальный по теме. Сюда перенесён без изменений содержания, чтобы не потерять исходник.
> ⚠️ **Устаревшие факты, не использовать:** здесь репозиторий назван `one-web` и стек «pytest + Selenium» — по факту это `one-web-kmp` и pytest + Playwright (canvas), исправлено 2026-07-24. Версия профиля `0.4.0` тоже устарела (в проде роль 0.5.1). Открытые уточнения по workspace и токенам закрыты — см. актуальный документ.

Пошаговый план: как отдать готовый профиль QA-команде ИВА и как они заводят его у себя и начинают проверять. QA-команда работает на **Codex** — ведущий путь Codex, Claude Code — как альтернатива.

Карта профиля: [[qa-profile-model-opis-multi-stek-model-qa-leinov]] · направление: [[Направление- Профили → QA-профиль (iva-role-qa) + AQA-toolkit ИВА]] · сценарий приёмки: [[verify-qa-kit-subagents]].

> **Порядок целиком.** Наша сторона: PR → merge → сид в каталог → провижн installation (Diaret) → выдать токены. Сторона QA: подключить tacticum-mcp → агент вытягивает профиль → прописать секреты → первый прогон.

---

## ЭТАП 0 — Наша сторона (до передачи)

1. **PR бандла** в `tacticum-dev` (жмёт президент) → **docker-CI зелёный** по DB-тестам → **merge в main**.
2. **Сид в прод-каталог** — на push в main CI сидит профиль в Postgres каталога (`iva-role-qa` + рёбра `depends_on` на лейны). Без сида профиль нельзя провижнить.
3. **Workspace QA-команды** в Tacticum существует (если нет — админ/Diaret заводит).  уточнить.

## ЭТАП 1 — Что должно быть у QA

| Что                                 | Зачем                                                                                                                        | Кто даёт                         |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| Репо `one-web` с рабочим окружением | лейн его НЕ провижнит — предполагает готовое: autocore-venv, `tools/testops`, `playwright-cli`/`allurectl`/`glab`, GitLab CI | у них есть                       |
| Codex CLI (или Claude Code)         | на нём гоняется профиль; субагенты на `gpt-5.4`                                                                              | у них                            |
| phk-токен `tacticum-mcp`            | вытянуть контент профиля из каталога                                                                                         | админ/Diaret (Этап 2)            |
| Allure TestOps apitoken             | `TESTOPS_ENDPOINT/TOKEN/PROJECT_ID` — читать ТС + публиковать автотесты                                                      | их TestOps                       |
| Личный Atlassian PAT (jira.iva.ru)  | заводить/обновлять дефекты (mcp-atlassian)                                                                                   | каждый инженер свой              |
| helm-analyst Bearer                 | read-срез факт-базы (requirement_tests и т.д.)                                                                               | админ/Diaret  уточнить env-форму |

## ЭТАП 2 — Провижн installation (админ/Diaret, сейчас через admin REST)

```bash
# создать installation для workspace QA-команды
curl -X POST https://dev.tacticum.dev/admin/installations \
  -H "Authorization: Bearer <admin_token>" -H "Content-Type: application/json" \
  -d '{"workspace_id":"<QA-workspace-UUID>","profile_id":"iva-role-qa",
       "profile_version_pinned":"0.4.0","label":"one-web QA / codex"}'

# выдать phk-токен этой installation
curl -X POST https://dev.tacticum.dev/admin/api-keys \
  -H "Authorization: Bearer <admin_token>" -H "Content-Type: application/json" \
  -d '{"installation_id":"<из ответа выше>","label":"QA laptop / codex"}'
# → phk_… отдаётся ОДИН РАЗ. Передать инженеру QA безопасным каналом.
```

DB-constraint `UNIQUE(workspace_id, profile_id)`: повторный POST с той же парой вернёт 409 — значит installation уже была. Self-service `provision_installation` одним MCP-вызовом приедет с US #434 — тогда админ не нужен.

## ЭТАП 3 — QA настраивает клиент, агент вытягивает профиль

**3.1. Подключить tacticum-mcp (Codex):**
```toml
# .codex/config.toml в корне one-web
[mcp_servers.tacticum-mcp]
url = "https://mcp.tacticum.dev/mcp"
transport = "http"
auth_type = "bearer"
env_required = ["TACTICUM_TOKEN"]
```
```bash
export TACTICUM_TOKEN="phk_…"   # phk из Этапа 2
```
Claude Code: `claude mcp add --transport http tacticum-mcp https://mcp.tacticum.dev/mcp --header "Authorization: Bearer phk_…"`

**3.2. Дать агенту в этом репо задачу «поставь профиль».** Он по канону:
- `whoami()` → видит installation с `profile_id == iva-role-qa`;
- (если репо не пустой от старого профиля) **clean install:** `rm -rf .agents/skills/` (codex) / `.claude/{agents,skills}/` (claude); из `AGENTS.md`/`CLAUDE.md` убрать блок с маркером старого профиля;
- `pull_installation_content_manifest(installation_id, target_cli="codex")` → список `actions_meta` (id+path, без тел);
- по каждому action — `tacticum_fetch_action(action_id)` → `write_file` по его path (каждый **отдельным** write; `~/.codex/config.toml` и `.claude/settings.json` — вне write-root, агент **эскалирует** на ручной merge);
- обновить `.tacticum/context.yaml`: `schema_version`, `mcp_endpoint`, `installation_id`, `organization_slug`, `workspace_slug`, `profile_id: iva-role-qa`;
- **рестарт CLI** — скиллы/субагенты попадают в сессию.

**3.3. Sanity:** `whoami()` → `profile_id == iva-role-qa`; в `.agents/` (codex) видны 3 субагента с `model: gpt-5.4`.

## ЭТАП 4 — Секреты профиля (токены подставляет инженер)

Профиль **рендерит конфиги MCP**, но токены/PAT инженер прописывает сам:
```bash
# Allure TestOps (автотест-скиллы) — env ИЛИ gitignored ./secrets.yaml (секция testops:)
export TESTOPS_ENDPOINT="https://<их-testops>"
export TESTOPS_TOKEN="<apitoken>"
export TESTOPS_PROJECT_ID="<numeric>"
# Atlassian PAT для mcp-atlassian (дефекты в Jira) — личный
export JIRA_URL="https://jira.iva.ru"
export JIRA_PERSONAL_TOKEN="<личный PAT>"
# helm-analyst Bearer — read-срез факт-базы (форма как у профиля аналитика; уточнить у Diaret)
```
`secrets.yaml` уже в `.gitignore` профиля; форма — `secrets.yaml.example`.

## ЭТАП 5 — Первый прогон (проверка, что всё работает)

По сценарию приёмки [[verify-qa-kit-subagents]]. В one-web с активным venv:

1. **Субагенты живые** — агент показывает 3 субагента на `gpt-5.4`.
2. **`write-autotest`** — дать URL тест-кейса TestOps → «напиши автотест». **OK:** появился `.tasks/work/tc-<id>/` (input/analysis/при пробелах locators), тест-функция в `tests/test_<feature>.py`, `pytest --collect-only` зелёный; в трейсе Task — спавн codebase-analyst/dom-explorer/code-writer.
3. **`batch-autotest`** — URL фильтра TestOps → веер по набору ТС.
4. **`fix-failed-test`** — имя упавшего теста / ланч-URL → диагноз (7 категорий) + правка + зелёный ре-прогон.
5. **Негативный тест секретов** — снять `TESTOPS_TOKEN` и убрать `secrets.yaml` → любой TestOps-запрос → **`TestOpsError` с инструкцией** (не молчаливый фейл, не хардкод). `grep allure.iva.ru` по установленной копии = 0.

## ЭТАП 6 — Что заранее сказать QA

- **Профиль = автоматизация** (генерация → прогон → починка → публикация → дефекты). **Тест-дизайн/авторинг ТС — не здесь** (это аналитик); QA получает готовые ТС и автоматизирует + ревьюит-дополняет.
- **Пока web-only** (one-web). iOS/KMP — отдельные профили позже.
- Один **phk = одна installation**; для обновления версии — `pull_installation_content_manifest(update=true)`, не новый провижн.
- Контакт по провижну/токенам — **Diaret**; по профилю — **мы (lead-qa)**.

## Открытые уточнения (до передачи)

1. **helm-analyst Bearer** — уточнить у Diaret точную env-форму/как выдаётся токен (у аналитика в проде работает).
2. **workspace QA-команды** в Tacticum — есть ли; если нет, Diaret заводит на Этапе 0.

## Связано

- [[qa-profile-model-opis-multi-stek-model-qa-leinov]] — карта профиля
- [[Направление- Профили → QA-профиль (iva-role-qa) + AQA-toolkit ИВА]] — направление
- [[verify-qa-kit-subagents]] — сценарий приёмки
- [[gate-qa-profile-bundle]] — вердикт гейта
- [[explore-qa-vs-analyst-final]] — отличия QA ↔ аналитик
