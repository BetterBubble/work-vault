---
title: QA-профиль iva-role-qa — комплект передачи команде
type: runbook
status: current
date: 2026-07-24
tags:
- qa
- handoff
- iva-role-qa
- onboarding
permalink: tacticum/12-features/qa-profil-iva-role-qa-komplekt-peredachi-komande
---

# QA-профиль `iva-role-qa` — комплект передачи команде

Самодостаточный документ передачи: что это за профиль, что нужно подготовить, как установить, как проверить первый прогон и что важно знать команде. QA-команда работает на **Codex** — это ведущий путь; **Claude Code** поддерживается как альтернатива.

---

## 1. Что это за профиль

`iva-role-qa` — профиль автоматизации автотестов для веб-приложения (репозиторий `one-web`). Он покрывает конвейер работы с тест-кейсами: **генерация автотест-кода → прогон → починка падений → публикация → заведение дефектов**. На вход берутся готовые тест-кейсы из Allure TestOps, на выходе — автотесты (pytest + Playwright), опубликованные результаты и дефекты в Jira.

**Что профиль НЕ делает.** Авторинг тест-кейсов и тест-дизайн в профиль не входят — это зона аналитика. QA получает **готовые тест-кейсы**, автоматизирует их и **ревьюит-дополняет** на этапе сверки. Продуктовый код, постановку требований (BRD/FR/ADR), авторинг FR-страниц в Confluence и владение покрытием требований профиль тоже не делает. Покрытие требований профиль только **читает** (узкий read-срез факт-базы), но не владеет им.

**Пока web-only.** Профиль жёстко завязан на веб-тулинг (Playwright/Selenium/pytest/autocore/glab) и работает только в репозитории `one-web`. iOS и KMP — это отдельные профили в будущем; сегодня реальный скоуп — только web.

**Состав.** Профиль (версия `0.4.0`) собран из трёх лейнов: базовый (репо-биндинг, навигация по базе знаний, git-конвенции, базовый MCP), автотест-конвейер (9 скиллов + референс-стек + 3 субагента, привязка к `one-web`) и MCP-обвязка (write-дефекты в Jira + read-срез факт-базы через helm-analyst). Ведущий CLI — **Codex**; субагенты работают на модели **`gpt-5.4`**. Три субагента: `codebase-analyst` (read-only инвентарь кода), `dom-explorer` (разведка живого UI, единственный с браузером), `code-writer` (генерация кода тест-слоя).

---

## 2. Что должно быть у команды до старта

| Что | Зачем | Кто даёт |
| --- | --- | --- |
| Репо `one-web` с рабочим окружением | Профиль его НЕ провижнит — предполагает готовое: autocore-venv, `tools/testops`, `playwright-cli`/`allurectl`/`glab`, GitLab CI | у команды есть |
| Codex CLI (или Claude Code) | На нём гоняется профиль; субагенты на `gpt-5.4` | у команды |
| phk-токен `tacticum-mcp` | Вытянуть контент профиля из каталога | админ / Diaret (см. раздел 3) |
| Allure TestOps apitoken | `TESTOPS_ENDPOINT/TOKEN/PROJECT_ID` — читать тест-кейсы и публиковать автотесты | их TestOps |
| Личный Atlassian PAT (`jira.iva.ru`) | Заводить/обновлять дефекты (mcp-atlassian) | каждый инженер свой |
| helm-analyst Bearer | Read-срез факт-базы (`requirement_tests` и т.д.) | админ / Diaret · env-форму уточнить |

**Три ключа-секрета, которые подставляет инженер:**

- **`TESTOPS_*`** (Allure TestOps): `TESTOPS_ENDPOINT`, `TESTOPS_TOKEN`, `TESTOPS_PROJECT_ID` — для чтения тест-кейсов и публикации автотестов.
- **Atlassian PAT** (`jira.iva.ru`): личный токен инженера для заведения и обновления дефектов в Jira.
- **`TACTICUM_TOKEN`** (phk-токен установки): для подключения `tacticum-mcp` и вытягивания контента профиля.

Плюс **helm-analyst Bearer** для read-среза факт-базы (точную env-форму — уточнить, см. раздел 6).

---

## 3. Установка профиля

### 3.1. Провижн установки (админ / Diaret, сейчас через admin REST)

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

Ограничение БД `UNIQUE(workspace_id, profile_id)`: повторный POST с той же парой вернёт `409` — значит установка уже была.

### 3.2. Подключить `tacticum-mcp`

**Codex** — файл `.codex/config.toml` в корне `one-web`:

```toml
[mcp_servers.tacticum-mcp]
url = "https://mcp.tacticum.dev/mcp"
transport = "http"
auth_type = "bearer"
env_required = ["TACTICUM_TOKEN"]
```

```bash
export TACTICUM_TOKEN="phk_…"   # phk из шага 3.1
```

**Claude Code:**

```bash
claude mcp add --transport http tacticum-mcp https://mcp.tacticum.dev/mcp --header "Authorization: Bearer phk_…"
```

### 3.3. Агент вытягивает профиль

Дать агенту в репозитории `one-web` задачу «поставь профиль». По канону он:

- `whoami()` → видит установку с `profile_id == iva-role-qa`;
- (если репо не пустой от старого профиля) **clean install:** `rm -rf .agents/skills/` (Codex) / `.claude/{agents,skills}/` (Claude Code); из `AGENTS.md`/`CLAUDE.md` убрать блок с маркером старого профиля;
- `pull_installation_content_manifest(installation_id, target_cli="codex")` → список `actions_meta` (id + path, без тел);
- по каждому action — `tacticum_fetch_action(action_id)` → `write_file` по его path (каждый **отдельным** write; `~/.codex/config.toml` и `.claude/settings.json` — вне write-root, агент **эскалирует** на ручной merge);
- обновить `.tacticum/context.yaml`: `schema_version`, `mcp_endpoint`, `installation_id`, `organization_slug`, `workspace_slug`, `profile_id: iva-role-qa`;
- **рестарт CLI** — скиллы и субагенты попадают в сессию.

**Sanity-проверка:** `whoami()` → `profile_id == iva-role-qa`; в `.agents/` (Codex) видны 3 субагента с `model: gpt-5.4`.

### 3.4. Прописать секреты профиля

Профиль **рендерит конфиги MCP**, но токены и PAT инженер прописывает сам:

```bash
# Allure TestOps (автотест-скиллы) — env ИЛИ gitignored ./secrets.yaml (секция testops:)
export TESTOPS_ENDPOINT="https://<их-testops>"
export TESTOPS_TOKEN="<apitoken>"
export TESTOPS_PROJECT_ID="<numeric>"
# Atlassian PAT для mcp-atlassian (дефекты в Jira) — личный
export JIRA_URL="https://jira.iva.ru"
export JIRA_PERSONAL_TOKEN="<личный PAT>"
# helm-analyst Bearer — read-срез факт-базы (форма как у профиля аналитика; уточнить)
```

`secrets.yaml` уже в `.gitignore` профиля; форма — `secrets.yaml.example`.

---

## 4. Первый прогон — проверка, что всё работает

Прогон делается в `one-web` с активным venv. Порядок: сначала убеждаемся, что субагенты живые, затем прогоняем три автотест-скилла и негативный тест секретов.

**Пред-условия (env):**

```bash
export TESTOPS_ENDPOINT="https://<стенд-testops>"
export TESTOPS_TOKEN="<apitoken>"            # или ./secrets.yaml секция testops:
export TESTOPS_PROJECT_ID="<numeric>"
# venv one-web активен; в PATH: pytest, playwright-cli, allurectl, glab, git
# субагенты установлены: .agents/{codebase-analyst,dom-explorer,code-writer}
```

**Шаг 0 — субагенты живые.** Агент показывает три субагента, у каждого модель `gpt-5.4` (не дефолт сессии). **OK:** три субагента видны, `model: gpt-5.4` у каждого; в файле субагента один блок фронтматера с корректным `tools:`.

**Шаг 1 — `write-autotest` (1 тест-кейс).** Дать скиллу URL тест-кейса `${TESTOPS_ENDPOINT}/project/<pid>/test-cases/<id>` и попросить «напиши автотест».
**OK:** создан `.tasks/work/tc-<id>/input.md`; `analysis.md` записан оркестратором (в трейсе Task видно: `codebase-analyst` вернул текст, файл писал скилл); при пробелах адресации — `locators.md` от `dom-explorer`; тест-функция в `tests/test_<feature>.py`; `pytest --collect-only <файл>` зелёный. В логе Task — спавн `codebase-analyst`/`dom-explorer`/`code-writer`.

**Шаг 2 — `batch-autotest` (набор тест-кейсов).** Дать URL фильтра TestOps.
**OK:** создан `.tasks/work/wave-<кампания>/PROGRESS.md` + минимум один дом `.tasks/work/tc-<id>/`; веер research-агентов реально стартовал (по умолчанию 3); тесты пишет оркестратор, не `code-writer`.

**Шаг 3 — `fix-failed-test` (починка).** Дать имя упавшего теста или ланч-URL.
**OK:** диагноз (7 категорий) в `.tasks/work/batch-<ts>/fix-<test>-analysis.md` от `codebase-analyst`; при категории `DOM_CHANGED` — `fix-<test>-locators.md` от `dom-explorer`; правка тест-слоя от `code-writer` (с самопроверкой `py_compile`); верификационный ре-прогон зелёный; отчёт `fix-batch-<ts>-report.md`.

**Шаг 4 — негативный тест секретов.** Снять `TESTOPS_TOKEN` и убрать `secrets.yaml`, затем выполнить любой TestOps-запрос.
**OK:** возвращается `TestOpsError` с инструкцией (не молчаливый фейл, не хардкод-фолбэк). `grep allure.iva.ru` по установленной копии = 0 (полная проверка: `grep -rn "allure.iva.ru\|Bearer <literal>"` = 0).

---

## 5. Что важно знать команде

- **Профиль = автоматизация** (генерация → прогон → починка → публикация → дефекты). Тест-дизайн и авторинг тест-кейсов — не здесь (это аналитик); QA получает готовые тест-кейсы, автоматизирует их и ревьюит-дополняет.
- **Пока web-only** (`one-web`). iOS/KMP — отдельные профили позже.
- Один **phk-токен = одна установка**. Для обновления версии — `pull_installation_content_manifest(update=true)`, а не новый провижн.
- **Контакты.** По провижну и токенам (установка, phk, доступы) — **Diaret**. По самому профилю (состав, поведение скиллов, вопросы по работе) — **команда платформы Tacticum**.

---

## 6. Открытые уточнения

1. **helm-analyst Bearer** — уточнить точную env-форму и как выдаётся токен (у профиля аналитика в проде работает; QA использует тот же сервер, узкий read-срез).
2. **Workspace QA-команды** в Tacticum — есть ли он; если нет, заводится на этапе провижна перед установкой.