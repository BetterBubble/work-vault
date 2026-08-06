---
title: 'Разведка: канал записи в Allure TestOps — тот же ли, что Jira/Confluence'
type: note
status: draft
role: explorer
tags:
- explorer
- iva-write
- allure
permalink: tacticum/00-board/explore-allure-write-channel-2026-07-27
archived-at: 2026-08-04 10:01
---

# Разведка: запись в Allure TestOps — один канал с Jira/Confluence или нет

Дата: 2026-07-27. Только чтение кода, на серверы не ходил, ничего не правил.

Срез репозиториев:
- `/Users/bubblemac/tacticum/helm` — `c8ddbf7 Add ownership assignment layer and API`
- `/Users/bubblemac/tacticum/tacticum-dev` — `119ccb0 Add BPMS process skills to analysis profile`

## 1. Чтение из Allure в helm — как сделано целиком

**Конфиг** — `/Users/bubblemac/tacticum/helm/src/helm/config.py:478-494`, префикс `HELM_`:
- `allure_base_url` (:482) — база API через SSH-туннель (systemd `iva-allure-tunnel`, порт 8791; в контейнере `allure.iva.ru` резолвится в docker-gateway).
- `allure_token` (:484) — комментарий дословно: «API-токен Allure (`Authorization: Api-Token <token>`). Секрет — только в `.env`».
- `allure_verify_tls` (:486) — по умолчанию `False` (внутренний cert контура ИВА).
- `allure_project_one: int = 5` (:488), `allure_project_kmp: int = 37` (:490).
- `allure_snapshot_dir: str = "data/real/allure"` (:494).

**Где секрет.** Имя переменной окружения — `HELM_ALLURE_TOKEN`, живёт в `/opt/helm/.env` на проде: спека `/Users/bubblemac/tacticum/helm/docs/superpowers/specs/2026-07-18-allure-testops-read-integration-design.md:52` и `:105`. В локальном `/Users/bubblemac/tacticum/helm/.env` строк с `ALLURE` нет (проверил грепом по имени, значения не выводил) — то есть локально интеграция не сконфигурирована. `docker-compose.prod.yml:101-104` добавляет `extra_hosts: allure.iva.ru:172.18.0.1` и монтирует `./data/real/allure` (:76-78).

**Клиент** — `/Users/bubblemac/tacticum/helm/src/helm/infrastructure/allure/client.py`:
- `AllureClient.__init__` (:28-36) — синхронный `httpx.Client`, заголовок задаётся один раз: `headers={"Authorization": f"Api-Token {token}"}` (:35), `verify=verify_tls`.
- Только `GET`: приватный `_get` (:48-54) и `_paged` (:56-65, Spring-пагинация `content[]` + `last`). Методов записи в классе нет.
- Эндпоинты (версия API — `/api/rs/*`):
  - `GET /api/rs/project` — `projects()` (:69);
  - `GET /api/rs/testcase/__search?projectId=&rql=issue = "<key>"` — `testcases_by_issue()` (:71-74);
  - `GET /api/rs/testcase/__search?projectId=&rql=cf["Feature"] = "<name>"` — `testcases_by_feature()` (:76-83; в комментарии :79-80 зафиксировано, что `feature = …` даёт `invalid.aql`, работает только `cf["Feature"]`);
  - `GET /api/rs/launch?projectId=&sort=createdDate,DESC` и `GET /api/rs/testresult?launchId=` — `recent_launch_statuses()` (:85-108).
- Докстринг модуля (:9): «Только чтение — ничего в TestOps не пишем (внешний контур заказчика)».

**Ингест** — `/Users/bubblemac/tacticum/helm/src/helm/ingest/allure_snapshot.py`: `build_snapshot()` (:55-83) собирает `by_jira` (track one, проект 5) и `by_feature` (track kmp, проект 37); `write_snapshot()` (:86-103) пишет `data/real/allure/allure_snapshot.json` + `manifest.json` (`as_of`, sha256, счётчики). Живой клиент создаётся только в `_main()` (:146-148), запуск — `python -m helm.ingest.allure_snapshot`.

**Как тул `requirement_tests` получает данные — ИЗ СНАПШОТА, не из живого API.** `/Users/bubblemac/tacticum/helm/src/helm/interface/mcp/analyst_server.py:481-524` берёт `cov = _allure_coverage(app)` (:500); хелпер `_allure_coverage` (:202-215) читает `allure_snapshot.json` через `AllureCoverage.load(directory)` с кэшем на `app.state` и инвалидацией по mtime файла — джоба (cron 12ч) перезаписывает файл, кэш перечитывается без рестарта. Реестр `AllureCoverage` — `/Users/bubblemac/tacticum/helm/src/helm/domain/allure.py:130-182`, докстринг модуля (:4): «БЕЗ сети (фетч — в `ingest/allure_snapshot.py`)». Нет снапшота → `available: false` и честное «нет данных» (`analyst_server.py:501-506`). В докстринге тула прямо: «НЕ запускает тесты и НЕ пишет в Allure (только чтение)» (:494-495).

Итого: HTTP к Allure идёт ТОЛЬКО из батч-джобы, MCP-тул — офлайн-читалка артефакта.

## 2. Модель аутентификации Allure TestOps

Из кода и спеки (`docs/superpowers/specs/2026-07-18-allure-testops-read-integration-design.md`):
- `:26` — «аутентификация `Authorization: Api-Token <token>` напрямую даёт 200 (обмен на JWT через `/api/uaa/oauth/token` тоже работает, но не обязателен)». То есть два рабочих режима: сырой api-token в заголовке (helm) и обмен на короткоживущий JWT (QA-лейн, см. §3).
- `:35` — «Доступ read-only через REST `/api/rs/*` (тот же токен, что у `allurectl`)».
- `:94` — «Allure — внешний контур заказчика ИВА: доступ строго read-only (`GET /api/rs/*`), ничего не пишем в TestOps».
- `:105` — контракт: `Auth: Authorization: Api-Token …`, значение только в `/opt/helm/.env`.

**Личный или сервисный токен — в коде/спеках репо НЕ сказано.** Формулировка «тот же токен, что у `allurectl`» (:35) и статус спеки «ждёт токен Allure» (:4) говорят о том, что токен получен от QA-стороны, но понятия «сервисная учётка» в разведанных файлах нет. Помечаю: **не проверял** (нужен ответ QA-lead ИВА, спека сама выносит вопросы к нему в §7). В Allure TestOps API-токен по устройству продукта выпускается в профиле конкретного пользователя — но это внешнее знание, а не факт из репо, поэтому как факт не подаю.

## 3. Запись в Allure — что уже есть в QA-лейнах tacticum-dev

**Канал записи сегодня — `allurectl` в CI, не MCP и не агент.**

- `templates/iva-qa-autotest-base/manifest.yaml:311` — «Внешние CLI в PATH: `pytest`, `playwright-cli` (бинарь), `allurectl`, `glab`, `git`. Лейн их НЕ устанавливает».
- `manifest.yaml:320` — «Публикация в Allure — через `allurectl`/`tools.testops`, НЕ через IVA QA Agent (сервис тест-дизайна)».
- `manifest.yaml:328` (non_goals) — «MCP-серверы — не нужны: Allure TestOps через `tools/testops` в репо, остальное внешние CLI».
- Чтение из TestOps в лейне — собственный read-only python-модуль репо-консьюмера `tools/testops`: `ingredients/skills/write-autotest/references/testops-api.md:8` — «MCP-сервер `testops` **не используется** — модуль точнее … работает на том же доступе к TestOps, что и `allurectl`»; `:109` — «Только чтение: модуль делает лишь GET (+ обмен api-token на JWT). Никаких create/update/delete — TC и launch'ами управляет человек».

**Переменные доступа — `TESTOPS_*`, не `ALLURE_*`:**
- `manifest.yaml:308-310` (post_install_notes): «СЕКРЕТЫ — по env-модели, НЕ зашиты в коде: env `TESTOPS_ENDPOINT`/`TESTOPS_TOKEN`/`TESTOPS_PROJECT_ID` → gitignored `secrets.yaml` (секция `testops:`) … apitoken → JWT (`grant_type=apitoken`)».
- `ingredients/config/secrets.yaml.example:11-17` — порядок резолва (env → `secrets.yaml` → явная ошибка), поле `token: "<your-api-token>"` c пометкой «apitoken (обменивается на JWT), НЕ пароль».
- `ingredients/skills/write-autotest/references/testops-api.md:30-38` — обмен `POST ${TESTOPS_ENDPOINT}/api/uaa/oauth/token`, `grant_type=apitoken`, дальше `Authorization: Bearer <jwt>`.
- История: `CHANGELOG.md:158-160` — из скиллов вычищен хардкод `allure.iva.ru` и нарратив «токен зашит, не секрет», заменён env-моделью.

**Где реально дёргается `allurectl`** — в CI-джобе проекта, агент лишь запускает пайплайн:
- `ingredients/skills/jira-issue-autotest/SKILL.md:69-74` — агент вызывает `glab ci run -b VCSAT-NNNN --variables … JIRA_ISSUE:<ISSUE>`, и `JIRA_ISSUE` «кладёт `<ISSUE>` в имя ланча и привязывает ланч к задаче (`allurectl launch add-issue --integration-id 5`)» — то есть сам `allurectl` исполняется внутри пайплайна.
- `:79` — «`allurectl` создаёт ланч в начале job».
- `ingredients/skills/batch-autotest/references/phases.md:247` — «Загрузку результатов делает CI-контур проекта … модуль tms — только чтение»; `:249` и `references/conventions.md:119` — статусы TC переключаются TMS автоматически после upload, руками в TMS ничего не менять.
- `ingredients/skills/batch-autotest/SKILL.md:68` — «Upload результатов в TMS — **только если пользователь явно попросит**».

**Оговорка про write-MCP — дословно:**
- `templates/iva-qa-autotest-base/manifest.yaml:16-18`:
  > `mcp_server_spec` НЕ требуется: обращение к Allure TestOps идёт НЕ через MCP, а через собственный read-only Python-модуль `tools/testops` в самом репо one-web-kmp; остальное — внешние CLI, установленные в окружении. Ни один скилл не дергает helm/iva-read/iva-atlassian-write MCP.
- `templates/iva-qa-autotest-base/README.md:56` — та же фраза: «Ни один скилл не дергает helm/iva-read/iva-atlassian-write MCP.»
- `README.md:102` — строка таблицы: «Allure / TestOps | бинари `allurectl` / `glab` / `playwright-cli` + модуль `tools/testops` в one-web-kmp | локально, **НЕ** MCP».
- `CHANGELOG.md:212-213` — «**mcp_server_spec — 0.** Allure TestOps через read-only модуль `tools/testops` в самом репо; остальное — внешние CLI. helm/iva-read/iva-write MCP не задействованы.»

**`iva-role-qa`** — сам MCP не несёт: `templates/iva-role-qa/README.md:11` («пак — это НЕ `mcp_server_spec`»), состав — `tacticum-core-base` + `iva-qa-autotest-base` + `iva-qa-mcp` (`README.md:21`). Allure упоминается только как назначение публикации (`manifest.yaml:60`, `:65`, `:67`).

**`iva-qa-mcp`** — два сервера, Allure среди них нет (`templates/iva-qa-mcp/manifest.yaml:96-137`):
- `iva-atlassian-write`: `transport: stdio`, `command: uvx`, `args: [mcp-atlassian]`, `env_required: [JIRA_URL, JIRA_PERSONAL_TOKEN]`, `allowed_tools` сужён до `jira_create_issue`/`jira_update_issue`/`jira_add_comment`/`jira_get_transitions`/`jira_transition_issue`; `CONFLUENCE_*` не выставляется.
- `helm-analyst`: `transport: http`, `url: https://helm.tacticum.ru/mcp/analyst`, bearer `TACTICUM_TOKEN`, в `allowed_tools` — `requirement_tests` (то есть QA читает покрытие через helm, а не напрямую из Allure).

## 4. Сопоставление протоколов

**(а) Один ли это протокол — НЕТ.** Три независимых различия, каждое по факту из кода:
1. **API и схема.** Allure — свой REST `/api/rs/*` со Spring-пагинацией и языком запросов RQL/AQL (`client.py:69-108`). Jira/Confluence — Atlassian REST, который в MCP закрыт пакетом `mcp-atlassian` (sooperset) с тулами `jira_*`/`confluence_*` (`iva-qa-mcp/manifest.yaml:104-112`).
2. **Схема токена.** Allure: `Authorization: Api-Token <token>` (`client.py:35`) либо обмен apitoken→JWT и `Bearer <jwt>` (`testops-api.md:35-36`). Atlassian Server/DC: личный PAT в `JIRA_PERSONAL_TOKEN`/`CONFLUENCE_PERSONAL_TOKEN` (`iva-fr-analyst/manifest.yaml:197`).
3. **Хосты и транспорт.** Allure — `allure.iva.ru` через SSH-туннель (`config.py:479-480`, `docker-compose.prod.yml:101-104`). Atlassian-write — локальный `uvx mcp-atlassian` по stdio против `jira.iva.ru`/`wiki.iva.ru` (`iva-fr-analyst/manifest.yaml:178-197`).

**Уточнение к формулировке задачи:** write-канал Atlassian идёт НЕ через gateway. Через gateway (`https://mcp.tacticum.ru/iva-read/mcp`, bearer `TACTICUM_TOKEN`) — только ЧТЕНИЕ: `iva-fr-analyst/manifest.yaml:163-176`, комментарий дословно «Jira 10.3 / Confluence 9.2 через platform MCP gateway — ТОЛЬКО чтение (маршрут `iva-read` write-тулов не отдаёт)». Запись — отдельный локальный stdio-сервер под личным PAT (`:178-197`).

**(б) Может ли один MCP-сервер обслуживать оба — технически да, но общего у них будет только процесс.** Что потребуется:
- отдельный HTTP-клиент Allure (готовый образец уже есть — `helm/src/helm/infrastructure/allure/client.py`, но он GET-only, write-методов в нём нет);
- отдельный секрет (`Api-Token`/apitoken TestOps) рядом с Atlassian PAT — разные учётки, разные сроки жизни, разный владелец выпуска;
- отдельный сетевой путь (SSH-туннель до `allure.iva.ru` vs прямой доступ к `jira.iva.ru`);
- отдельный набор тулов (`allure_*`) — переиспользовать `jira_*` нельзя, схема сущностей другая (testcase/launch/testresult vs issue/page);
- если брать готовый `mcp-atlassian` (sooperset) — это форк или отдельный сервер: пакет ставится как `uvx mcp-atlassian`, апстрим.

**(в) MCP-серверов для Allure нигде нет.**
- В конфиге пользователя: `~/.claude.json` и `~/.claude/settings.json` — 0 упоминаний `allure` (грепал по количеству, значения не выводил). `~/.claude/settings.local.json` и `~/.mcp.json` отсутствуют. Настроенные MCP: глобально `basic-memory`, `context7`, `serena`, `ssh-manager`; проектно для `/Users/bubblemac/tacticum/helm` — `helm-analyst`. Ни одного allure/testops.
- В `tacticum-dev`: `mcp_server_spec` с `testops`/`allure` нет ни в одном манифесте (грепал `ingredient_id: *testops` — пусто); `.mcp.json` в репозитории нет.
- Единственный след — прошлое: `testops-api.md:8` «MCP-сервер `testops` **не используется**» и `:106` (почему: терял shared steps, `attachments:[]`). То есть MCP к TestOps когда-то пробовали и осознанно ушли от него в пользу своего python-модуля.
- Поддержка Allure в open-source `mcp-atlassian` (sooperset): **исходников локально нет, не проверял** (искал `mcp_atlassian` в кэшах uv и в системе — не найден; пакет тянется через `uvx` в момент запуска).

## 5. Вывод: цена объединения vs allurectl

**Что даёт объединение записи Allure в один канал с Jira/Confluence:**
- один MCP-процесс и одно место конфигурации в профиле роли (`iva-qa-mcp` вместо «MCP + CI-переменные»);
- write в Allure становится доступен агенту напрямую, без пайплайна GitLab — сейчас это невозможно в принципе (`allurectl` живёт в CI-джобе, `jira-issue-autotest/SKILL.md:69-79`).

**Чего это стоит (конкретные работы):**
1. Новый HTTP-клиент записи в Allure: сейчас во всём helm нет НИ ОДНОГО write-вызова к Allure — `client.py` содержит только `_get`/`_paged`. Писать POST/PATCH-слой с нуля, включая контракт создания/обновления TC и launch (в разведанном контракте спеки §7a перечислены только GET-эндпоинты, `:106-112`).
2. Новый токен и новая учётка: apitoken TestOps с write-правами. Сейчас read-токен объявлен read-only на уровне нормы — `спека:94` «ничего не пишем в TestOps», `client.py:9` то же. Write-права придётся запрашивать у владельца контура ИВА отдельно; чей это будет аккаунт (личный QA или сервисный) — открытый вопрос, см. §2.
3. Новые тулы и разграничение прав: `allowed_tools` в `iva-qa-mcp` придётся расширять, а write в TMS сейчас явно отдан человеку/CI (`testops-api.md:109`, `conventions.md:119`).
4. Сетевой путь: до Allure из места запуска MCP нужен туннель (в helm он есть как systemd-юнит; на машине инженера — нет).
5. Ломается заявленная в манифесте граница: `manifest.yaml:18` «Ни один скилл не дергает … MCP» и `:328` «MCP-серверы — не нужны» перестают быть правдой — нормы надо переписывать, иначе документ будет описывать несуществующий порядок.

**Что даёт альтернатива «оставить Allure на allurectl»:**
- ноль новых секретов, клиентов и тулов — всё уже работает: upload делает CI, статусы TC переключает сам TMS (`phases.md:247-249`);
- запись в внешний контур заказчика остаётся за пайплайном и человеком, read-контур helm остаётся неизменно read-only (норма `спека:94` сохраняется);
- ограничение: агент не может писать в Allure вне пайплайна — только запустить пайплайн через `glab` и дождаться ланча (`jira-issue-autotest/SKILL.md:69-81`). Если задача в том, чтобы агент публиковал результаты сам, эта альтернатива её не решает.

**Ответ на исходный вопрос:** одним каналом с Jira/Confluence Allure не покрывается — это другой продукт, другой REST, другая схема токена и другой сетевой путь. «Объединить» означает написать второй клиент и завести второй секрет внутри одного процесса; экономия при этом — только на количестве MCP-серверов в конфиге.

## Что не проверял
- Личный vs сервисный характер токена Allure — в репо не зафиксировано.
- Исходники `mcp-atlassian` (sooperset) — локально отсутствуют.
- Репо-консьюмер `one-web-kmp` (модуль `tools/testops`, `.gitlab-ci.yml` с `allurectl`) — в рабочих директориях сессии его нет, судил по шаблонам лейна.
- На серверы не ходил, значений секретов не читал и не выводил.