---
title: Инвентарь write-возможностей в контур ИВА (Jira/Confluence) — объявлено vs
  реально вызывается
type: note
status: draft
role: explorer
tags:
- explorer
- iva-write
permalink: tacticum/00-board/explore-iva-write-inventory-2026-07-27
---

# Инвентарь write-канала ИВА: объявлено в манифесте vs реально вызывается скиллом

**Репо:** `/Users/bubblemac/tacticum/tacticum-dev`, ветка `main`, HEAD `119ccb0` («Add BPMS process skills to analysis profile»).
Разведка read-only. Все ссылки — файл:строка на этом HEAD.

## 0. Главный вывод одной строкой

Единственный write-канал в контуре — ингредиент `iva-atlassian-write` (`kind: mcp_server_spec`), объявлен в **5 манифестах**. **Реально дёргают write-тулы только 3 скилла аналитика** (`fr-authoring`, `process-analysis-stage`, `mockup-authoring`). Лейны архитектора, QA и техписа объявляют write-канал, но **не содержат ни одного скилла вообще** — там только `manifest.yaml`, `README.md`, `CHANGELOG.md`.

Плюс отдельная находка: **`allowed_tools` не доезжает до рендера** — ни claude-code, ни codex-рендерер его не пишут. Пер-ролевой скоуп существует только на бумаге.

---

## 1. Ингредиенты `kind: mcp_server_spec` с записью в Jira/Confluence

Единственный такой ингредиент — `iva-atlassian-write`. Всего 5 объявлений:

| # | Файл:строка | Профиль | transport | command/args | env_required | auth_type | allowed_tools | tier |
|---|---|---|---|---|---|---|---|---|
| 1 | `templates/iva-fr-analyst/manifest.yaml:187` | iva-fr-analyst (старый монопрофиль) | stdio | `uvx mcp-atlassian` | JIRA_URL, JIRA_PERSONAL_TOKEN, CONFLUENCE_URL, CONFLUENCE_PERSONAL_TOKEN | — (нет ключа) | **отсутствует** | trial |
| 2 | `templates/iva-analysis-base/manifest.yaml:381` | iva-analysis-base (лейн аналитика) | stdio | `uvx mcp-atlassian` | те же 4 | — | **отсутствует** | trial |
| 3 | `templates/iva-architect-mcp/manifest.yaml:85` | iva-architect-mcp | stdio | `uvx mcp-atlassian` | те же 4 | — | confluence_create_page, confluence_update_page, jira_create_issue, jira_update_issue, jira_add_comment, jira_get_transitions, jira_transition_issue | trial |
| 4 | `templates/iva-qa-mcp/manifest.yaml:101` | iva-qa-mcp | stdio | `uvx mcp-atlassian` | **только** JIRA_URL, JIRA_PERSONAL_TOKEN | — | jira_create_issue, jira_update_issue, jira_add_comment, jira_get_transitions, jira_transition_issue | trial |
| 5 | `templates/iva-techwriter-mcp/manifest.yaml:81` | iva-techwriter-mcp | stdio | `uvx mcp-atlassian` | CONFLUENCE_URL, CONFLUENCE_PERSONAL_TOKEN, JIRA_URL, JIRA_PERSONAL_TOKEN | — | confluence_create_page, confluence_update_page, jira_add_comment | trial |

`required_scopes` не задан **нигде** (схема его допускает: `templates/_schema/ingredient.v1.schema.json:102`). `auth_type` для stdio не выставляется — аутентификация целиком через env-PAT.

**Что это физически.** Обёртка над open-source `mcp-atlassian` (sooperset) — прямо названо в манифестах: `templates/iva-architect-mcp/manifest.yaml:24`, `templates/iva-qa-mcp/manifest.yaml:28`, `templates/iva-techwriter-mcp/manifest.yaml:26`, `templates/iva-fr-analyst/manifest.yaml:178-179`. Запуск — `uvx mcp-atlassian` (PyPI-пакет, не образ, не свой код). Режим — **Jira/Confluence Server/DC** против `jira.iva.ru` / `wiki.iva.ru` (переменная `*_PERSONAL_TOKEN` = PAT Server/DC, не Cloud с `*_API_TOKEN`). Аутентификация — **личный PAT конкретного человека**, из env процесса CLI.

Дословный YAML (эталон, `templates/iva-architect-mcp/manifest.yaml:85-103`):

```yaml
  - ingredient_id: iva-atlassian-write
    kind: mcp_server_spec
    tier: trial
    supports: [claude-code, codex]
    install_scope: repo
    body: ""
    metadata:
      transport: stdio
      command: uvx
      args: [mcp-atlassian]
      env_required: [JIRA_URL, JIRA_PERSONAL_TOKEN, CONFLUENCE_URL, CONFLUENCE_PERSONAL_TOKEN]
      allowed_tools:
        - confluence_create_page
        - confluence_update_page
        - jira_create_issue
        - jira_update_issue
        - jira_add_comment
        - jira_get_transitions
        - jira_transition_issue
```

Вариант в лейне аналитика (`templates/iva-analysis-base/manifest.yaml:378-391`) — **без `allowed_tools` вообще**, то есть номинально полный набор тулов mcp-atlassian:

```yaml
  # Write-канал публикации FR (US #699, зеркало iva-fr-analyst 0.1.3+). INTERIM:
  # личный PAT (mcp-atlassian, Server/DC). Целевой iva-write (техучётка + подпись
  # актора, ADR-0058) заменит этот ингредиент — имена тулов совпадают, скиллы не трогаются.
  - ingredient_id: iva-atlassian-write
    kind: mcp_server_spec
    tier: trial
    supports: [claude-code, codex]
    install_scope: repo
    body: ""
    metadata:
      transport: stdio
      command: uvx
      args: [mcp-atlassian]
      env_required: [JIRA_URL, JIRA_PERSONAL_TOKEN, CONFLUENCE_URL, CONFLUENCE_PERSONAL_TOKEN]
```

### 1.1. `allowed_tools` — декларация без исполнения

`apps/backend/src/backend/catalog/infrastructure/renderers/claude_code.py:102-131` (`_render_mcp_server`) собирает entry для `.mcp.json` из `transport`/`command`/`args`/`url`/`env`/`headers`. **`allowed_tools` не читается и не пишется.** То же в codex-рендерере (`apps/backend/src/backend/catalog/infrastructure/renderers/codex.py:140+`) и в зеркале для live-пути (`apps/backend/src/backend/catalog/domain/renderer.py:61+` `_mcp_server_value`). Поле есть в доменной модели (`apps/backend/src/backend/catalog/domain/ingredients/mcp_server_spec.py:23`) и в схеме, но дальше БД не идёт.

Следствие: у архитектора/QA/техписа сервер поднимается с **полным** тулсетом mcp-atlassian. Единственное фактическое сужение — **env**: у QA не выставляются `CONFLUENCE_*`, поэтому Confluence-часть mcp-atlassian не поднимается (это отмечено и в комментарии `templates/iva-qa-mcp/manifest.yaml:99-100`). Jira-сужение («только дефекты/статусы») ничем не удерживается.

---

## 2. Кто несёт write-канал

Роли — pure-composition, канал приходит через `depends_on`:

| Роль | `depends_on` | Откуда write | Что объявлено |
|---|---|---|---|
| `iva-role-analyst` (`manifest.yaml:20-23`) | tacticum-core-base, **iva-analysis-base**, tacticum-research-base | iva-analysis-base | полный mcp-atlassian, без `allowed_tools` |
| `iva-role-architect` (`manifest.yaml:26-29`) | tacticum-core-base, iva-analysis-base, **iva-architect-mcp** | оба лейна (см. коллизию ниже) | Confluence page-authoring + полный Jira issue-ops |
| `iva-role-qa` (`manifest.yaml:29-32`) | tacticum-core-base, iva-qa-autotest-base, **iva-qa-mcp** | iva-qa-mcp | Jira defect/status (5 тулов), без Confluence |
| `tacticum-role-techwriter` (`manifest.yaml:27-30`) | tacticum-core-base, tacticum-documentation-base, **iva-techwriter-mcp** | iva-techwriter-mcp | Confluence create/update + `jira_add_comment` |
| `iva-role-go` (`manifest.yaml:18-24`) | core, development-core, iva-go-development-base, bugfix, lite, research | **нет** | — |
| `iva-role-web/java/ios/kmp/mail` | аналогично go + ui | **нет** | — |
| `iva-fr-analyst` | (монопрофиль, own-ингредиенты) | own | полный, без `allowed_tools` |

**Коллизия у архитектора.** `iva-role-architect` композит и `iva-analysis-base` (широкий канал), и `iva-architect-mcp` (скоупленный) — один и тот же `ingredient_id`. Это единственное задокументированное исключение из single-owner: `apps/backend/tests/catalog/test_iva_role_presets.py:216-222` (`KNOWN_OVERRIDES = {"iva-role-architect": {"iva-atlassian-write"}}`), выигрывает более поздний в `depends_on`. Практически, поскольку `allowed_tools` не рендерится, разницы между двумя версиями ингредиента на диске нет вообще.

**Лейны `iva-architect-mcp` / `iva-qa-mcp` / `iva-techwriter-mcp`** — каталоги из ровно трёх файлов (`manifest.yaml`, `README.md`, `CHANGELOG.md`). Директории `ingredients/` нет. Ни скиллов, ни команд, ни агентов. `iva-qa-mcp` дополнительно несёт read-ингредиент `helm-analyst` (`manifest.yaml:122-140`, 7 read-тулов).

**INTERIM / US #699.** Помечен как временный только лейн аналитика:
- `templates/iva-analysis-base/manifest.yaml:35` — «iva-atlassian-write (mcp-atlassian, личный PAT — interim, US #699). Целевой iva-write…»
- `templates/iva-analysis-base/manifest.yaml:378-380` — «Write-канал публикации FR (US #699…). INTERIM: личный PAT…»
- `templates/iva-analysis-base/CHANGELOG.md:194` — «личный PAT — interim до iva-write ADR-0058»
- `templates/iva-fr-analyst/CHANGELOG.md:195` — «Write-канал публикации FR (US #699)» (первоисточник)
- `docs/adr/0057-capability-layers-and-role-presets.md:116` — «write-канал E-FRQA #699», целевка — `iva-write` с личным PAT, трекинг Taiga #712

Лейны архитектора/QA/техписа как INTERIM **не помечены** — там формулировка «личный Atlassian PAT» подана как штатная модель.

---

## 3. Реальное использование write-тулов

Поиск по `templates/**/SKILL.md`, `commands/**`, `agents/**` на все write-тулы mcp-atlassian (`confluence_create_page|update_page|add_comment|delete_page|upload_attachment`, `jira_create_issue|update_issue|add_comment|transition_issue|create_issue_link|delete_issue|add_worklog|batch_create|link_to_epic|upload_attachment`).

### 3.1. Все попадания вне манифестов/README/CHANGELOG

| Файл:строка | Лейн | Тул(ы) |
|---|---|---|
| `templates/iva-analysis-base/ingredients/skills/fr-authoring/SKILL.md:77` | iva-analysis-base | таблица инструментов: `confluence_create_page`, `confluence_update_page`, `jira_create_issue`, `jira_add_comment` |
| `.../fr-authoring/SKILL.md:285-286` | iva-analysis-base | те же 4, шаг 9 «Чем публиковать» |
| `.../fr-authoring/SKILL.md:291-292` | iva-analysis-base | `confluence_create_page` / `confluence_update_page` (публикация FR-страницы) |
| `.../fr-authoring/SKILL.md:302` | iva-analysis-base | `confluence_upload_attachment(content_id, file_path)` — drawio-вложения |
| `.../fr-authoring/SKILL.md:311` | iva-analysis-base | `jira_add_comment` (+ `jira_create_issue` условно) |
| `.../fr-authoring/SKILL.md:394-396` | iva-analysis-base | `confluence_update_page`, `jira_add_comment` (петля `/update-feature`) |
| `templates/iva-analysis-base/ingredients/skills/mockup-authoring/SKILL.md:59` | iva-analysis-base | `confluence_upload_attachment` |
| `templates/iva-analysis-base/ingredients/skills/process-analysis-stage/SKILL.md:41,89,91` | iva-analysis-base | `confluence_create_page` / `confluence_update_page` (публикация копии) |
| `templates/iva-fr-analyst/ingredients/skills/fr-authoring/SKILL.md:77,285-286,291-292,311,394-396` | iva-fr-analyst | зеркало analysis-base, те же тулы |

Больше **нигде**. В `templates/iva-architect-mcp`, `templates/iva-qa-mcp`, `templates/iva-techwriter-mcp`, `templates/tacticum-documentation-base`, `templates/iva-qa-autotest-base`, `templates/tacticum-bugfix-base` и во всех development-лейнах вызовов write-тулов нет. `tacticum-bugfix-base` работает с Jira, но только на чтение: `jira_get_issue`, `jira_get_issue_development_info`, `jira_get_issue_images` (`ingredients/skills/bug-fix/SKILL.md:65-68`).

### 3.2. Роль → какие write-тулы реально дёргаются

| Роль | Скиллы, дёргающие write | Тулы |
|---|---|---|
| `iva-role-analyst` | `fr-authoring` (шаг 9, 7б, `/update-feature`), `process-analysis-stage`, `mockup-authoring` | `confluence_create_page`, `confluence_update_page`, `confluence_upload_attachment`, `jira_add_comment`, `jira_create_issue` (условно) |
| `iva-fr-analyst` (монопрофиль) | `fr-authoring` | те же |
| `iva-role-architect` | `fr-authoring` и др. — **приходят из `iva-analysis-base`**, не из `iva-architect-mcp` | те же аналитические; собственных архитекторских write-скиллов нет |

### 3.3. Роль → объявлено, но ни один скилл не зовёт

| Роль | Объявлено | Скиллов, зовущих write | Комментарий |
|---|---|---|---|
| `iva-role-qa` | 5 Jira-тулов (`iva-qa-mcp`) | **0** | у лейна нет `ingredients/`; в 9 автотест-скиллах ни одного write-вызова |
| `tacticum-role-techwriter` | 3 тула (`iva-techwriter-mcp`) | **0** | единственный контентный скилл `doc-authoring` (`tacticum-documentation-base`) не упоминает Confluence/Jira вообще |
| `iva-role-architect` (в части `iva-architect-mcp`) | 7 тулов | **0** от лейна | write-вызовы приходят только из аналитических скиллов |

Обратная асимметрия: `confluence_upload_attachment` **вызывается** скиллами (`fr-authoring:302`, `mockup-authoring:59`), но **не объявлен** ни в одном `allowed_tools`. У аналитика это безвредно (`allowed_tools` пустой), у техписа — противоречие декларации, если бы скоуп исполнялся.

---

## 4. Проверка пяти утверждений

### (а) «iva-atlassian-write уже работает: обёртка над sooperset/mcp-atlassian, stdio, Jira Server/DC, личный PAT» — **ПОДТВЕРЖДЕНО**

- sooperset прямо назван: `templates/iva-architect-mcp/manifest.yaml:24`, `templates/iva-qa-mcp/manifest.yaml:28`, `templates/iva-techwriter-mcp/manifest.yaml:26`, `templates/iva-fr-analyst/manifest.yaml:178-179`, `templates/iva-fr-analyst/CHANGELOG.md:195-197`.
- `transport: stdio`, `command: uvx`, `args: [mcp-atlassian]` — все 5 объявлений.
- Server/DC против `jira.iva.ru` / `wiki.iva.ru`: `templates/iva-fr-analyst/CHANGELOG.md:196`, `templates/iva-architect-mcp/manifest.yaml:24-25`, `docs/user_manuals/iva-fr-analyst-profile-quickstart.md:35`.
- Личный PAT: `JIRA_PERSONAL_TOKEN`/`CONFLUENCE_PERSONAL_TOKEN` в env; `docs/user_manuals/iva-fr-analyst-profile-quickstart.md:35` — «выпускаешь сам: jira.iva.ru → Профиль → Personal Access Tokens».
- «Уже работает» — косвенное, но конкретное свидетельство: `templates/iva-fr-analyst/CHANGELOG.md:211-212` фиксирует боевой смоук публикации drawio-вложения на `wiki.iva.ru`, страница 211846230. Отдельного мониторинга/health-check в репо нет — **не проверял**, работает ли канал сегодня.

### (б) «подключён у аналитика (iva-analysis-base, INTERIM, US #699) и у архитектора (iva-architect-mcp: Confluence page-authoring + Jira issue-ops)» — **ПОДТВЕРЖДЕНО**

- Аналитик: `templates/iva-analysis-base/manifest.yaml:378-391` (INTERIM + US #699 в комментарии), роль — `templates/iva-role-analyst/manifest.yaml:20-23`.
- Архитектор: `templates/iva-architect-mcp/manifest.yaml:85-103` (ровно Confluence page-authoring + полный Jira issue-ops), роль — `templates/iva-role-architect/manifest.yaml:26-29`.
- **Уточнение:** «подключён» у архитектора = объявлен и отрендерится в `.mcp.json`. Скиллов, его дёргающих, в лейне нет (§3.3).

### (в) «разработчик, QA и техпис write-канала не имеют ВООБЩЕ» — **ЧАСТИЧНО ОПРОВЕРГНУТО**

- **Разработчик — подтверждено.** `iva-role-go/manifest.yaml:18-24` не тянет ни analysis, ни *-mcp лейнов. Инвариант закреплён тестом: `apps/backend/tests/e2e_install/test_install_flow.py:571-575` — `iva-atlassian-write` в списке `absent` для implement-only роли. Bugfix-лейн ходит в Jira только на чтение.
- **QA — опровергнуто.** Write объявлен: `templates/iva-qa-mcp/manifest.yaml:101-117` (5 Jira-тулов), роль тянет лейн (`iva-role-qa/manifest.yaml:29-32`), e2e ожидает его в бандле (`test_install_flow.py:846-850`). Сервер отрендерится и поднимется. Верно другое: **ни один скилл QA его не зовёт**.
- **Техпис — опровергнуто так же.** `templates/iva-techwriter-mcp/manifest.yaml:81-95`, роль — `tacticum-role-techwriter/manifest.yaml:27-30`. Объявлен, скиллами не используется.

Корректная формулировка: разработчик не имеет write-канала вообще; QA и техпис имеют объявленный и рендерящийся write-канал, но нулевое фактическое использование.

### (г) «QA-лейн iva-qa-autotest-base пишет в Allure через CLI allurectl + TESTOPS_TOKEN, и в манифесте прямо сказано, что скиллы не дёргают write-MCP» — **ПОДТВЕРЖДЕНО, с уточнением**

- `templates/iva-qa-autotest-base/manifest.yaml:16-18`: «mcp_server_spec НЕ требуется… **Ни один скилл не дергает helm/iva-read/iva-atlassian-write MCP**». Дословно.
- `manifest.yaml:309` — env `TESTOPS_ENDPOINT`/`TESTOPS_TOKEN`/`TESTOPS_PROJECT_ID` → gitignored `secrets.yaml`; `manifest.yaml:311` — внешние CLI `pytest`, `playwright-cli`, `allurectl`, `glab`; `manifest.yaml:320` — «Публикация в Allure — через `allurectl`/`tools.testops`, НЕ через IVA QA Agent».
- **Уточнение:** сам модуль `tools/testops` — read-only (`README.md:54`, `ingredients/skills/batch-autotest/references/phases.md:247` — «модуль tms — только чтение»). Заливку результатов делает CI-джоба через `allurectl` (`ingredients/skills/jira-issue-autotest/SKILL.md:70-74`: `allurectl launch add-issue --integration-id 5`), а не скилл напрямую. То есть «QA пишет в Allure через allurectl» верно, но пишет CI-контур репо `one-web-kmp`, скилл его лишь запускает.
- Также: скилл `jira-issue-autotest` явно декларирует «в саму Jira не ходим» (`SKILL.md:27,41`) — связь кейс↔задача резолвится через интеграцию TestOps↔Jira.

### (д) «контур IVAREQ (Jira-проект + Confluence-space) не создан» — **ПОДТВЕРЖДЕНО**

- `templates/iva-qa-autotest-base/ingredients/craft-stack/shared/pipeline-gate.md:79-82`: «Запись статусов в US IVAREQ (ADR-0060 §5/§7)… требует сервера `iva-write` **и** проекта IVAREQ — **их ещё НЕТ** (ADR-0058, будущая фаза)».
- `pipeline-gate.md:71`: «Запись статусов в US IVAREQ на этой фазе **НЕ делается**».
- `docs/adr/0058-...md:3`: статус Proposed, «к утверждению **до создания любых сущностей** — проект/space/MCP не создаются, пока ADR не принят».
- `docs/adr/0058-...md:119-120`: Ф1 (создать IVAREQ + space + iva-write) начинается только после утверждения ADR.
- `docs/adr/0058-...md:148`: «Воркфлоу IVAREQ и права техучётки настраивает админ Jira — зависимость от контура Монахова; без этого Ф1 стоит».
- Ни одного конфига/env/URL с IVAREQ в репо нет (grep по всему дереву — только упоминания в ADR и в TODO-текстах лейна QA). В Jira/Confluence не ходил, сущности не создавал.

---

## 5. ADR-0060 vs ADR-0058

**Статусы:** оба **Proposed**. ADR-0060 — `docs/adr/0060-...md:3` («Proposed; ревью Солонко в процессе; финализация после его OK»). ADR-0058 — `docs/adr/0058-...md:3`. В индексе `docs/adr/README.md` ADR-0058 (стр. 64) и ADR-0059 (стр. 65) есть, **ADR-0060 в индекс не внесён**.

**Что 0060 говорит про MCP-scoping (Решение 1, стр. 13-21):** сервер `helm-analyst` не делить, скоупить через `allowed_tools`, и `allowed_tools` живёт в **capability-лейне**, а не на роли-пресете; роль остаётся pure-composition. Критерий деления сервера — отдельный домен auth/секретов, независимый ЖЦ, иной бэкенд, изоляция blast-radius; для helm-analyst не выполнен ни один. Read-only даётся щедро, эскейп-хатч — расширение пресета, а не форк сервера. Стр. 17 отмечает: механизм сужения «смёржен и работает в проде (codegraph/arch)» — что расходится с найденным (§1.1): в рендерерах `allowed_tools` не обрабатывается ни для одного сервера.

**Что 0060 говорит про write (Решение 7, стр. 72-74):** write идёт через `iva-write` по модели ADR-0058 §5 — инстанс mcp-atlassian за `mcp.tacticum.ru`, PAT **технической учётки `iva`**, принудительная подпись актора из hub-ключа, scope `iva-req-write`. Локальный `iva-write-base` ретайрится; «наш `iva-atlassian-write` **привести к модели `iva-write`** (техучётка + подпись + scope), не «личный PAT»».

**Противоречие между ADR — нет; противоречие ADR ↔ реализация — есть.** 0060 не спорит с 0058, а явно принимает его модель write и уточняет только границу TC (Решение 4, стр. 39-44: TC пишет аналитик, QA дополняет на ревью — уточняет ADR-0058 §6). Сам 0060 фиксирует расхождение как открытый вопрос №1 (стр. 90): «Реконсиляция write-канала `iva-atlassian-write` ↔ `iva-write`: техучётка+подпись vs текущая реализация — сверить».

Фактическое расхождение с реализацией:

| Аспект | ADR-0058 §5 / ADR-0060 §7 (целевое) | Что в templates сейчас |
|---|---|---|
| Учётка | техучётка `iva` | личный PAT человека (все 5 объявлений) |
| Размещение | инстанс за `mcp.tacticum.ru` (remote) | локальный `uvx mcp-atlassian` (stdio) |
| Подпись актора | принудительная, из hub-ключа; без подписи — отказ | нет механизма |
| Scope | `iva-req-write` | `required_scopes` не задан нигде |
| Права | только IVAREQ + новый space | права личного PAT человека — всё, к чему он имеет доступ |
| Аудит | лог тул-вызовов на gateway | нет |

ADR-0058 отдельно называет «личный PAT каждому (Taiga #712 as-is)» **отклонённой альтернативой** — «неполный аудит, раздача PAT команде, ротация ×N» (таблица альтернатив). При этом ровно эта отклонённая модель — то, что стоит в манифестах, включая лейны архитектора/QA/техписа, созданные **позже** ADR-0058.

---

## 6. История

`git log --oneline -- templates/iva-analysis-base templates/iva-architect-mcp templates/iva-qa-mcp templates/iva-techwriter-mcp` — 19 коммитов; ключевые по write-каналу (`git log -S"iva-atlassian-write" --all`):

| sha | дата | сообщение | что произошло |
|---|---|---|---|
| `b6728a9` | 2026-07-20 | feat(templates): FR publication write-channel + fr_publish binding + label policy — iva-fr-analyst 0.1.3 (US #699-#701) | **рождение** `iva-atlassian-write` в `iva-fr-analyst` |
| `ecc9d5e` | 2026-07-21 | feat(templates): iva-analysis-base 0.1.3 — зеркальный перенос фич iva-fr-analyst 0.1.4–0.1.6 | зеркало канала в лейн аналитика |
| `c7103c4` | 2026-07-21 | feat(templates): паки уровня роли — iva-role-analyst/iva-role-go 0.1.1 | — |
| `b5fcdae` | 2026-07-22 | feat(templates)!: iva-role-go 0.2.0 — implement-only (ADR-0059 Решение 6) | dev отрезан от write |
| `86e6038` | 2026-07-23 | iva-role-qa: ретайр iva-write-base → own-ингредиент iva-atlassian-write (Jira-only) | QA получает канал |
| `81fa9fd` | 2026-07-23 | architect+techwriter: ретайр iva-write-base → own iva-atlassian-write (скоуп по роли) | архитектор+техпис получают канал |
| `1e0a8b3` | 2026-07-23 | iva-qa-autotest-base: переключить ссылки на write-канал iva-atlassian-write | — |
| `4532173` | 2026-07-23 | iva-role-qa: обновить нарратив/флаг под унификацию (лейн снесён, канал = own iva-atlassian-write) | лейн `iva-write-base` снесён |
| `6a3aa44` | 2026-07-23 | templates: три тонких per-role MCP-лейна (вариант B, ADR-0057) | **создание** `iva-architect-mcp`, `iva-qa-mcp`, `iva-techwriter-mcp` |
| `4bfbbdc` | 2026-07-23 | roles: вернуть QA/architect/techwriter в чистую композицию (вариант B) | роли → `ingredients: []` |
| `3010307` | 2026-07-23 | iva-role-qa: пак уровня роли (ADR-0060) вместо pure-composition | — |
| `20ff9b8` | 2026-07-24 | feat(profiles): QA-профиль iva-role-qa + автотест-лейн + per-role MCP-лейны (#133) | мерж в main |
| `119ccb0` | 2026-07-26 | Add BPMS process skills to analysis profile | текущий HEAD; `process-analysis-stage` подхватывает write |

Весь write-контур родился и разошёлся по ролям за 4 дня (20–24 июля). Профиль `iva-write-base` (упомянут в ADR-0060:74 как «ретайрится») в дереве отсутствует — снесён `4532173`.

## 7. Что не проверял

- Живую работоспособность канала на 2026-07-27 (в Jira/Confluence не ходил).
- Наличие проекта IVAREQ/space на стороне ИВА — только следы в репо.
- Поведение самого `mcp-atlassian` при частично выставленных env (вывод «Confluence не поднимется без `CONFLUENCE_*`» взят из комментария манифеста, в коде пакета не сверял).
- Ветки, кроме `main` (`git log -S --all` покрывает только локальные refs).